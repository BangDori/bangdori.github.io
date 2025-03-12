---
title: 네트워크 / 서버 에러 대응을 위한 추가적인 방안 적용
date: 2025-03-10 01:36:00 +/-TTTT
categories: [Frontend, React Native]
tags: [react native, error handling, network error, server error, retry mechanism, exponential backoff, ux improvement, splash screen]
image:
  path: assets/img/thumbnail/network_error.png
sitemap:
  changefreq: daily
  priority: 0.5
---

> 목차
> 1. 네트워크 에러 대응 후
> 2. 네트워크 장애 상황에서의 retry
> 3. N/S 분리를 통해 에러 메시지 명확하게 전달하기
> 4. RTK가 만료된 사용자의 UX 개선하기
{: .prompt-tip }

## 1. 네트워크 에러 대응 후

![Sentry Result](assets/img/writing/16/network_handling_sentry_result.png){: width="360" }

이전 [네트워크 에러에도 끊김 없는 스플래시 화면 설계하기](https://bangdori.kr/posts/handling-network-error-in-splash/) 글에서 작성한 프로세스를 적용한 이후 **3월 11일 기준 10일 이내로 단 한건의 에러도 발생하지 않았습니다! 🎉**

새로운 프로세스 적용한 덕분에 **네트워크 및 서버 에러(Network/Server Error, 이하 N/S) 대응이 한층 더 안정적으로 이루어졌습니다.** 하지만, 여전히 보완할 수 있는 부분들이 눈에 띄었고, 팀원들과 논의하며 추가적인 피드백도 받게 되었습니다.

#### 개선안 목록

1. 네트워크 장애 상황에서 retry가 필요할까?
2. 네트워크와 서버 에러에 대한 명확한 에러 메시지를 전달해주자.
3. RefreshToken(이하 RTK)가 만료되어서 로그아웃 되는 사용자를 위한 UX 개선하기

그래서 이번 글에서는 기존 개선안에서 부족했던 부분을 어떻게 보완했으며, 예외 상황을 좀 더 견고하게 처리했는지 간단한 코드와 함께 경험을 공유하려고 합니다!

## 2. 네트워크 장애 상황에서의 retry

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

**하지만 과연 이 방식이 최선일까요?**

네트워크 장애 상황에도 꼭 `retry`가 필요할까요?

우선 네트워크 장애가 발생할 수 있는 장소(와이파이 신호가 약한 곳, 터널 등)로 가정해보고 생각해보겠습니다. 

네트워크가 불안정한 상황 속에서 사용자가 앱의 네트워크 에러 페이지를 마주하게 된다면, 사용자는 불편함을 느낄 수 있지만 최소한 "네트워크가 끊겼다"라는 사실을 명확하게 인지할 수 있습니다. 하지만 <span style="color: #FF6262;">반대로 네트워크가 불안정한 상태에서 로그인에 성공한다면?</span> Exponential Backoff Retry 전략으로 인해, 오히려 사용자가 스플래시 화면에 갇힌 것처럼 느낄 수 있어 큰 불편함을 느낄 수 있을 것입니다. ("앱 망했나?"와 같은 생각을 할 수도......)

뿐만 아니라 로그인에 성공하더라도 서비스 내부에서 지속적으로 에러가 발생할 가능성이 높고, 네트워크 지연으로 인해 UX/UI가 크게 저하될 것입니다.

그렇기 때문에 네트워크 장애 상황에서는 `retry`를 하는 것이 아닌 즉시 네트워크가 끊겼다를 명확하게 인지하도록 하는 것이 더 낫다고 판단하였습니다.

### 그렇다면 서버 에러가 발생했을 때도 `retry`가 필요없지 않을까요?

네트워크 장애 상황에서는 `retry`를 하지 않는 것이 더 나은 선택이었지만, **서버 에러(5xx)**의 경우에는 이야기가 다릅니다.

> **블루/그린 배포(Blue-Green Deployment)**
>
> 무중단 배포(Zero Downtime Deployment)를 위한 전략 중 하나로, 구버전(Blue)과 신버전(Green)을 지칭하여 붙어진 이름으로 운영 환경에 구버전과 동일하게 신버전의 인스턴스를 구성한 후, 로드밸런스를 통해 신버전으로 모든 트래픽을 전환하는 배포 방식
{: .prompt-info }

현재 저희 서버에는 **블루/그린 배포(Blue-Green Deployment)** 전략이 적용되어 있는데, 구버전에서 신버전으로 트래픽이 전환되는 과정에서 일시적으로 `503 Bad Gateway` 에러가 발생할 수 있습니다. 이는 서버가 완전히 교체되기 전에 일부 요청이 유효하지 않은 라우트로 전달되면서 발생하는 현상입니다.

이 경우에는 일정 시간 후 다시 요청하면 정상적으로 응답을 받을 수 있어 `retry`를 적용하는 것이 유효하다고 판단하였습니다.

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

![NS Error UI](assets/img/writing/16/ns_error_ui.png){: width="240" }
_N/S 에러_

기존에는 Network 에러가 발생하게 되면 아래 컴포넌트가 렌더링되고 있었습니다.

```tsx
export function NSErrorScreen({ refetch }: NSErrorScreenProps) {
  return (
    <SafeAreaView>
      <Icon name="alert" />
      <View>
        <Text>인터넷이 연결되어 있지 않아요</Text>
        <Text>연결 후 다시 시도해주세요.</Text>
      </View>
      <Button onPress={refetch}>다시 시도하기</Button>
    </SafeAreaView>
  );
}
```

이 방식은 단순히 N/S 에러에 대해 "인터넷이 연결되지 않았다"는 메시지를 출력하지만, 이는 <span style="color: #FF6262;">사용자가 네트워크 문제인지, 서버 문제인지 정확히 구분하기 어렵다</span>는 문제가 있습니다.

이를 분리하기 위해서 에러 객체의 response 정보를 이용하였습니다.

```tsx
/**
 * @description 
 * 네트워크 또는 서버 오류가 발생했을 때 사용자에게 적절한 메시지를 표시하는 컴포넌트입니다.
 * 네트워크 연결이 끊긴 경우와 서버 오류가 발생한 경우를 구분하여 각각 다른 메시지를 제공합니다.
 */
export function NSErrorScreen({ error, refetch }: ConnectionErrorScreenProps) {
  const isNetworkError = !error?.response;

  const title = isNetworkError
    ? '인터넷이 연결되어 있지 않아요'
    : '서버에 문제가 발생했어요';
  const description = isNetworkError
    ? '연결 후 다시 시도해주세요.'
    : '잠시 후 다시 시도해주세요.';

  return (
    <SafeAreaView style={styles.container}>
      <Icon name={isNetworkError ? "wifi" : "alert"} />
      <View>
        <Text>{title}</Text>
        <Text>{description}</Text>
      </View>
      <Button onPress={refetch}>다시 시도하기</Button>
    </SafeAreaView>
  );
}
```

그 결과 네트워크 에러가 발생한 경우와 서버 에러가 발생한 경우의 메시지를 명확하게 사용자에게 전달해줄 수 있게 되었습니다. (와이파이 아이콘 정말 예쁜 거 같아요)

![Network and Server Error UI](assets/img/writing/16/devied_ns_error_ui.png){: width="480" }
_(좌) 네트워크 장애 - 인터넷 연결 필요 / (우) 서버 에러 - 잠시 후 다시 시도_

## 4. RTK가 만료된 사용자의 UX 개선하기

사용자가 로그아웃되지 않도록 프로세스를 개선해, 사용자 경험을 해치지 않도록 노력해왔습니다. 하지만, RTK가 만료되는 상황은 완전히 피할 수 없습니다. 예를 들어, 1~2달 주기로 서비스를 이용하는 사용자라면 어떻게 대응하는 것이 좋을까요? 이들을 위해 RTK의 주기를 길게 설정하는 것이 최선의 방법일까요?

RTK의 주기를 길게 설정하는 것은 보안의 위험에 노출시킬 수 있기 때문에, 로그아웃이 된 사용자가 더 빨리 서비스 내부로 접근할 수 있도록 고려하였습니다. 현재 서비스에서는 애플/카카오/네이버/로컬 로그인으로 총 4개의 수단을 제공해주고 있습니다.

문뜩 그런 생각이 들었습니다.

> 🧐 과연 로그인 후 한달이 지난 시점에 내가 이전 로그인 수단을 기억할 수 있을까?

그래서 저희 팀에서는 RTK가 만료된 사용자의 재로그인을 빠르게 돕기 위해, <span style="color: #00CA5B;">이전 로그인 수단을 안내</span>하는 작은 말풍선을 추가하였습니다.

![Onboarding Screen](assets/img/writing/16/onboarding.png){: width="240" }
_출처: [스폰지](https://app.sponjy.com/mYCR/invite)_

작은 변화지만, 사용자 입장에서 고민을 덜어주는 큰 차이를 만들어냈습니다.

## 5. 결론

