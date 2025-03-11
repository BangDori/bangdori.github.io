---
title: 네트워크 / 서버 에러 대응을 위한 추가적인 방안 적용
date: 2025-03-10 01:36:00 +/-TTTT
categories: [Frontend, React Native]
tags: [react native, error handling, network error, server error, retry mechanism, exponential backoff, ux improvement, splash screen]
image:
  path: assets/img/writing/16/network_handling_sentry_result.png
sitemap:
  changefreq: daily
  priority: 0.5
---

> 목차
> 1. 네트워크 에러 대응 후의 이야기
> 2. 네트워크 장애 상황에서 retry가 필요할까
> 3. N/S 분리를 통해 에러 메시지 명확하게 전달하기
> 4. RTK가 만료된 사용자의 UX 개선하기
{: .prompt-tip }

## 1. 네트워크 에러 대응 후의 이야기

![Sentry Result](assets/img/writing/16/network_handling_sentry_result.png){: width="360" }

이전 [네트워크 에러에도 끊김 없는 스플래시 화면 설계하기](https://bangdori.kr/posts/handling-network-error-in-splash/) 글에서 작성한 프로세스를 적용한 이후 **3월 11일 기준 10일 이내로 단 한건의 에러도 발생하지 않았습니다! 🎉**

새로운 프로세스 적용한 덕분에 **네트워크 및 서버 에러(Network/Server Error, 이하 N/S) 대응이 한층 더 안정적으로 이루어졌습니다.** 하지만, 여전히 보완할 수 있는 부분들이 눈에 띄었고, 팀원들과 논의하며 추가적인 피드백도 받게 되었습니다.

#### 개선안 목록

1. 네트워크 장애 상황에서 retry가 필요할까?
2. 네트워크와 서버 에러에 대한 명확한 에러 메시지를 전달해주자.
3. RTK가 만료되어서 로그아웃 되는 사용자를 위한 UX 개선하기

그래서 이번 글에서는 기존 개선안에서 부족했던 부분을 어떻게 보완했으며, 예외 상황을 좀 더 견고하게 처리했는지 간단한 코드와 함께 경험을 공유하려고 합니다!

## 2. 네트워크 장애 상황에서 retry가 필요할까

기존 N/S에 대한 `retry` 대응 전략은 다음과 같이 구현되어 있었습니다.

```ts
export async function onError(error: AxiosError<ResponseError>) {
  const apiError = new ApiError(error);

  /** 🌐 네트워크 에러 (서버에 도달하지 못함) */
  if (
    !apiError.response || // 응답 자체가 없는 경우
    apiError.code === 'ERR_NETWORK' || // Axios가 명시적으로 네트워크 장애로 판별한 경우
    apiError.code === 'ECONNABORTED' // 요청이 타임아웃된 경우
  ) {
    return Promise.reject(apiError);
  }

  /** 🔥 서버 에러 (5XX) */
  if (apiError.response.status >= 500) {
    return Promise.reject(apiError);
  }

  // ...

  // 🚨 클라이언트 에러 핸들링 (4XX)
  return Promise.reject(apiError.response.data);
}
```

API 요청 도중 에러가 발생하면, `onError` 인터셉터에서 에러 유형을 분류한 뒤 `reject`합니다. 이렇게 `reject`된 에러는 `useQuery`의 `retry` 메서드에서 감지되어 재시도 로직이 Exponential하게 실행됩니다.

```ts
export function useGetRefreshTokenQuery() {
  return useQuery({
    queryFn: refreshToken,
    retry: (failureCount, error) => {
      // 🌐 N/S 재시도 (2회)
      if (error instanceof ApiError) {
        if (!error.response || error.response.status >= 500) {
          return failureCount < 2;
        }
      }

      // 🚨 Client Error
      return false;
    },
    retryDelay: (attemptIndex) => 1000 * 2 ** attemptIndex,
    // ...
  });
}
```

**하지만 과연 이 방식이 최신일까요?**

네트워크 장애 상황에도 꼭 `retry`가 필요할까요?

우선 네트워크 장애가 발생할 수 있는 장소(와이파이 신호가 약한 곳, 터널 등)로 가정해보고 생각해보겠습니다. 

네트워크가 불안정한 상황 속에서 사용자가 앱의 네트워크 에러 페이지를 마주하게 된다면, 사용자는 불편함을 느낄 수 있지만 최소한 "네트워크가 끊겼다"라는 사실을 명확하게 인지할 수 있습니다. 하지만 <span style="color: #FF6262;">반대로 네트워크가 불안정한 상태에서 로그인에 성공한다면?</span> Exponential Backoff Retry 전략으로 인해, 오히려 사용자가 스플래시 화면에 갇힌 것처럼 느낄 수 있어 큰 불편함을 느낄 수 있을 것입니다. ("앱 망했나?"와 같은 생각을 할 수도......)

뿐만 아니라 로그인에 성공하더라도 서비스 내부에서 지속적으로 에러가 발생할 가능성이 높고, 네트워크 지연으로 인해 UX/UI가 크게 저하될 것입니다.

그렇기 때문에 네트워크 장애 상황에서는 `retry`를 하는 것이 아닌 즉시 네트워크가 끊겼다를 명확하게 인지하도록 하는 것이 더 낫다고 판단하였습니다.

### 그렇다면 서버 에러가 발생했을 때는 `retry` 왜 필요할까요?

네트워크 장애 상황에서는 `retry`를 하지 않는 것이 더 나은 선택이었지만, **서버 에러(5xx)**의 경우에는 이야기가 다릅니다.

> **블루/그린 배포(Blue-Green Deployment)**
>
> 무중단 배포(Zero Downtime Deployment)를 위한 전략 중 하나로, 구버전(Blue)과 신버전(Green)을 지칭하여 붙어진 이름으로 운영 환경에 구버전과 동일하게 신버전의 인스턴스를 구성한 후, 로드밸런스를 통해 신버전으로 모든 트래픽을 전환하는 배포 방식
{: .prompt-info }

현재 저희 서버에는 **블루/그린 배포(Blue-Green Deployment)** 전략이 적용되어 있는데, 트래픽이 전환되는 과정에서 일시적으로 `503 Bad Gateway` 에러가 발생할 수 있는데, 이는 서버가 완전히 교체되기 전에 일부 요청이 유효하지 않은 라우트로 전달되면서 발생하는 현상입니다.

이런 경우에는 일정 시간 후 다시 요청하면 정상적으로 응답을 받을 수 있는데, `retry`를 적용하는 것이 유효하다고 판단하였습니다.

```ts
export function useGetRefreshTokenQuery() {
  return useQuery({
    queryFn: refreshToken,
    retry: (failureCount, error) => {
      // 🌐 Server Error 재시도 (2회)
      if (error instanceof ApiError) {
        if (error.response && error.response.status >= 500) {
          return failureCount < 2;
        }
      }

      // 🚨 Network / Client Error
      return false;
    },
    retryDelay: (attemptIndex) => 1000 * 2 ** attemptIndex,
    // ...
  });
}
```

Network와 Client Error가 발생한 경우에는 재시도를 하지 않고, Server 에러가 발생하는 경우에만 `retry`를 하도록 개선하였습니다.

## 3. N/S 분리를 통해 에러 메시지 명확하게 전달하기

## 4. RTK가 만료된 사용자의 UX 개선하기