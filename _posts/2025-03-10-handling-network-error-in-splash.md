---
title: 네트워크 에러에도 끊김 없는 스플래시 화면 설계하기
date: 2025-03-10 01:36:00 +/-TTTT
categories: [Frontend, React Native]
tags: [react native, splash screen, network error, error handling, retry mechanism, error boundary, tanstack query, ux improvement]
image:
  path: assets/img/writing/15/sentry_issue.png
sitemap:
  changefreq: daily
  priority: 0.5
---

> 목차
> 1. 개요
> 2. 문제 인식
> 3. 문제 접근
> 4. 테스트를 통한 안정성 검증
> 5. 결론
{: .prompt-tip }

## 1. 개요

> **스플래시(Splash)**
>
> 모바일 앱 실행 시 가장 처음 만나게 되는 화면으로, 보통 1초에서 3초정도 이어집니다. 대부분의 앱은 일반적으로 스플래시 UI를 가지고 있습니다. - [1초의 디테일, 스플래시 시각보정.](https://brunch.co.kr/@shaun/60)
{: .prompt-info }

현재 서비스는 앱을 시작하면 스플래시가 시작되고, 스플래시가 시작되는 동안 인증 정보와 사용자 정보를 받아오도록 플로우가 구성되어 있습니다. staging에서 QA를 진행할 때마다 매번 문제가 없었는 것 같은데..

![Sentry Issue](assets/img/writing/15/sentry_issue.png){: width="720" }

어느날 이런 이슈가 발생했습니다..! 아니 Users를 확인해보니, 이미지에는 나와있지 않지만 이미 <span style="color: #FF6262;">3명의 사용자가 강제로 로그아웃</span>을 당했었습니다. 😭

분명 QA 과정에서는 별다른 문제가 없었는데, 갑자기 이런 이슈가 터지다니.. 심지어 이미 3명의 사용자가 강제로 로그아웃된 걸 확인하고 나니, 그냥 넘길 수가 없었습니다. 😭

그래서 이번 글에서는 어떤 문제가 있었는지 분석해 보고, 다양한 해결책을 고민한 끝에 제가 이 문제를 해결해나간 과정을 공유해 보려고 합니다!

## 2. 문제 인식

![App flow](assets/img/writing/15/app_flow.png){: width="720" }
_기존 플로우_

현재 서비스는 앱을 시작하면 스플래시가 시작되고, 스플래시가 시작되는 동안 인증 정보와 사용자 정보를 받아오도록 플로우가 구성되어 있습니다. 이러한 플로우에는 한 가지 큰 문제점이 있습니다.

혹시 이 플로우에서 문제가 보이시나요?

바로 <span style="color: #FF6262;">클라이언트 에러, 네트워크 장애나 서버 오류(Network/Server Error, 이하 N/S)에 대한 에러 핸들링이 분리가 되어 있지 않은 것</span>입니다. 그래서 네트워크가 원활하지 않은 환경에서 앱을 시작하면 데이터를 읽을 수 없게 되어 로그아웃되었던 것입니다.

**이게 큰 문제인가?** 라고 생각할 수 있는데, 네트워크 장애나 서버 오류(Network/Server Error, 이하 N/S)는 클라이언트 측에서 발생하는 문제가 아닌, 외부 환경적인 요소 영향을 받는 에러이기 때문에 앱 내 사용자에게는 영향을 주지 않아야 합니다.

즉, 이미 로그인 한 사용자는 잘못된 접근이 아닌 이상은 로그아웃 되지 않아야 한다는 말이죠!

만약 이 이슈를 빠르게 해결하지 않는다면, 다른 사용자들에게 전파되어 사용자가 다시 로그인을 시도하게 만듦으로써 사용자 경험을 크게 저하시키는 원인이 될 수 있습니다. 특히나 로그인이 필수적인 서비스에게는 치명적인 문제로 다가올 수 있기 때문에, 로그인 상태를 최대한 유지할 수 있도록 플로우를 개선해야 합니다.

## 3. 문제 접근

문제 해결을 하기 앞서, 해결하고자 하는 바를 다음과 같이 명확히 정의했습니다.

> <span style="color: #00CA5B;">N/S로 인해 사용자 정보(`refreshToken`)가 손실되지 않도록 하자</span>

문제를 해결하기 위해 고려해본 접근 방식은 다음과 같습니다.

1. API 요청 전 네트워크 상태 확인
2. Queue를 활용한 요청 재시도(Retry)
3. Retry와 ErrorBoundary를 조합하여 유연하게 처리하기
4. 점진적 Retry 후 네트워크 상태 확인

이제 각 방법을 하나씩 자세히 살펴보겠습니다.

### 3-1. API 요청 전 네트워크 상태 확인

첫 번째 대안은 <span style="color: #00CA5B;">"Refresh API Call 전에 네트워크 상태를 확인"</span>하는 방식입니다.

![App flow 대안 1](assets/img/writing/15/app_flow_alternative1.png){: width="720" }
_첫 번째 대안_

앱 실행 시 네트워크 상태를 확인하고, 연결이 원활하지 않다면 ConnectionError 화면을 띄웁니다. 이를 통해 네트워크 문제를 사용자에게 알리고, Refresh API Call 요청 시 네트워크가 연결되어 있음을 보장할 수 있습니다.

그러나 이 방법은 API 요청 후 발생하는 N/S 문제를 해결하지 못한다는 한계가 있습니다. User API Call에서 에러가 발생하면 여전히 로그아웃이 발생할 수 있어 적합하지 않습니다.

### 3-2. Queue를 활용한 요청 재시도(Retry)

두 번째 대안은 <span style="color: #00CA5B;">"N/S 발생 시 요청을 Queue에 저장하고, 네트워크 복구 후 자동 재처리하는 방식"</span>입니다.

![App flow 대안 2](assets/img/writing/15/app_flow_alternative2.png){: width="720" }
_두 번째 대안_

이 방식의 장점은 클라이언트(`4xx`)와 서버 오류(`5xx`) 처리를 분리하여 기존 로그인 상태를 유지하면서 네트워크 불안정성을 보완할 수 있다는 점입니다.

하지만 해당 대안을 구현하기 위해서는 너무 많은 예외 상황을 고려해야 합니다.

1. 홈페이지로 이동하는 조건이 두 개라 플로우가 복잡해짐
2. 네트워크 복구 후 Queue에서 요청을 재처리하는 로직이 필요함
3. 서버 장애(`5xx`) 발생 시 복구 여부를 확인하기 어려움
4. 홈페이지 이동 후에 발생하는 API 에러에 대한 추가적인 처리 필요

플로우가 너무 복잡하고 예외 처리가 많아야 한다는 점에서 적합하지 않다고 판단했습니다.

### 3-3. Retry와 ErrorBoundary를 조합하여 유연하게 처리하기

세 번째 대안은 <span style="color: #00CA5B;">"Retry와 `ErrorBoundary`를 활용하여 유연하게 처리하는 방식"</span> 입니다.

![App flow 대안 3](assets/img/writing/15/app_flow_alternative3.png){: width="720" }
_세 번째 대안_

기존의 플로우를 유지하되 N/S에 대해서는 `Retry`를 1초의 딜레이를 가지고 진행합니다. 그리고, 재시도 후에도 실패하면 Splash가 종료된 이후 `ErrorBoundary`를 활용해 사용자에게 오류 메시지를 제공하고 재시도를 유도합니다.

이 방식은 복잡도를 최소화하면서도 유연한 대응이 가능하다는 장점이 있으나, <span style="color: #FF6262;">무한 루프의 발생 가능성</span>이라는 치명적인 단점이 있습니다. 물론 이런 상황이 발생할 가능성은 매우 낮겠지만, 예를 들어 네트워크 장애가 장시간 지속되거나, 서버가 응답을 줄 수 없는 상태라면 어떻게 될까요?

사용자는 계속해서 N/S에 마주하게 되고, 결국 `ErrorBoundary` 화면에서 벗어나지 못하는 문제가 발생합니다.

이는 로그인 플로우에서 치명적일 뿐만 아니라, 추후 서비스가 확장되어 로그인 없이도 이용할 수 있는 구조가 되었을 때도 심각한 문제로 작용할 수 있습니다. 이러한 설계를 유지한다면, <span style="color: #FF6262;">인증 여부와 관계없이 서비스 전반에서 사용자가 정상적인 화면을 볼 수 없게 되는 리스크</span>가 발생할 수 있습니다. 이러한 이유로, 로그인 플로우에서 안정성을 보장하기 어렵다고 판단했습니다.

### 3-4. 점진적 Retry 후 네트워크 상태 확인

마지막 대안은 1번 대안과 3번 대안의 아이디어를 합친 방식입니다.

![App flow 대안 4](assets/img/writing/15/app_flow_alternative4.png){: width="720" }
_네 번째 대안_

해당 방법에서는 N/S가 발생하면 `Retry`를 진행합니다. 여기서 중요한 점은 단순히 일정 간격으로 `Retry` 하는 것이 아닌, 재시도 간격을 점진적으로 늘리는 전략을 취해 연결 에러 화면을 최대한 보지 않도록, 성공 확률을 높이는 것입니다. 추가로 `Retry` 이후에도 네트워크가 연결되지 않는다면, RTK 토큰 정보는 유지한 채 연결 에러 화면을 렌더링하여 사용자가 네트워크 확인 후 재시도할 수 있도록 유도합니다.

즉, 마지막 대안은 ✅ <span style="color: #00CA5B;">1초 → 2초 → 4초로 증가하는 방식의 Exponential Backoff Retry 전략과 최후의 수단으로 네트워크 상태를 확인</span>하는 방법입니다.

이 방식은 기존 세 번째 대안과 유사해보이지만 `ErrorBoundary`를 제거하고 Retry 및 네트워크 연결 상태 확인을 통해 에러 핸들링을 더욱 단순화하였습니다.

> **🧐 네트워크 연결이 되어있지 않은 상황이라면 굳이 Retry 할 필요가 없지 않나요?**
>
> 네트워크가 연결되어 있지 않은 환경이라면 굳이 Retry를 시도할 필요가 없겠지만, 생각보다 네트워크가 불안정한 환경이 많기 때문에 Retry Option을 추가하였습니다. (ex. 와이파이 1칸, 엘리베이터, 지하와 같이 갇힌 공간 등)
{: .prompt-info }

### 🚀 3-5. 최종 결론

1번 대안은 앱 시작 시에만 연결 상태를 확인하기 때문에, 근본적인 해결책이 될 수 없다고 판단하여 제외하였습니다.

| \            | 2️⃣ Queue 방식 | 3️⃣ Retry & ErrorBoundary | 4️⃣ Network Check and Retry |
|---------------------------|---------------|---------------------------|------------------------------|
| **N/S 대응**  | ❌ Queue의 복잡한 예외 처리 문제 | ➖ 재시도 및 UI를 제공하지만 무한 루프 위험 | ✅ 점진적 Retry + 네트워크 확인 |
| **안정성**     | ➖ `5xx`에 대한 대응이 어려움  | ❌ 무한 루프 위험 | ✅ Retry 초과 시 네트워크 확인 후 재시도 |
| **확장성**     | ❌ Queue 관리의 부담 | ➖ API별로 ErrorBoundary 처리가 필요하지만 확장 가능 | ✅ Queue/ErrorBoundary 없이 N/S 대응이 가능하기에 유지보수가 용이하며 확장성 고려 가능 |

비교 결과, N/S 대응이 가능하며 안정성과 확장성 측면에서 강점을 가진 대안 4가 가장 적합하다고 판단하였습니다.

## 4. 테스트를 통한 안정성 검증

이번 프로세스 개선 작업을 진행하면서, 문제 해결을 위한 코드 수정뿐만 아니라 **실제로 수정된 코드가 의도한 대로 동작하는지 검증하는 과정**도 매우 중요했습니다.

특히, 네트워크 장애 상황이나 예외 처리가 핵심이기 때문에, 단순한 단위 테스트를 넘어 다양한 시나리오를 모킹(Mock)하여 검증하는 과정이 필요했습니다.  

테스트는 [Jest](https://jestjs.io/) + [React Native Testing Library](https://callstack.github.io/react-native-testing-library/)을 활용하여 진행하였으며,  API 요청과 네트워크 상태를 제어하기 위해 `mock`을 적극적으로 활용했습니다.  

> 해당 섹션에서는 많은 테스트 코드가 포함되어 있습니다.
> 코드 예시가 많으므로, 필요한 경우에만 확인하는 것을 권장합니다.  
{: .prompt-warning }

React Native 환경에서는 외부 모듈에 대한 모킹 부담이 크기 때문에 에러 유형별 테스트가 아닌, API 단위로 테스트를 진행하겠습니다.

### 4-1. Refresh API Call 테스트

Refresh API Call과 User API Call은 `useAuth` 훅을 통해 관리됩니다.  
따라서, **`useAuth`를 중심으로 인증 흐름을 검증하는 방식으로 테스트를 진행합니다.**

1. Refresh API Call이 성공하면 User API Call이 호출된다.
2. 네트워크 장애(offline) 발생 시, N/S 에러가 발생한다.
3. 서버 오류(5xx) 발생 시, N/S 에러가 발생한다.
4. 클라이언트 오류(4xx) 발생 시, N/S 에러가 발생하지 않는다.

```tsx
describe('Refresh API Call 테스트', () => {
  beforeEach(() => {
    // 모든 mock 함수와 queryClient를 초기화합니다.
    jest.clearAllMocks();
    queryClient.clear();
  });

  it('Refresh API Call이 성공하면 User API Call이 호출된다.', async () => {
    (useRefreshTokenQuery as jest.Mock).mockReturnValue({
      isSuccess: true,
      isError: false,
      error: undefined,
    });

    renderHook(() => useAuth(), { wrapper });

    await waitFor(() => {
      expect(getUserDetail).toHaveBeenCalledTimes(1);
    });
  });

  it('네트워크 장애(offline) 발생 시, N/S 에러가 발생한다.', async () => {
    (useRefreshTokenQuery as jest.Mock).mockReturnValue({
      isSuccess: false,
      isError: true,
      error: new ApiError(new Error('Network Error') as any),
    });

    const { result } = renderHook(() => useAuth(), { wrapper });

    await waitFor(() => {
      expect(result.current.isAuthFinished).toBe(true);
      expect(result.current.isNSError).toBe(true);
      
      expect(result.current.isLoggedIn).toBe(false);
    });
  });

  it('서버 오류(5xx) 발생 시, N/S 에러가 발생한다.', async () => {
    const statusArray = [500, 502, 503];

    for (const status of statusArray) {
      (useRefreshTokenQuery as jest.Mock).mockReturnValue({
        isSuccess: false,
        isError: true,
        error: new ApiError({ response: { status } } as any),
      });

      const { result } = renderHook(() => useAuth(), { wrapper });

      await waitFor(() => {
        expect(result.current.isAuthFinished).toBe(true);
        expect(result.current.isNSError).toBe(true);

        expect(result.current.isLoggedIn).toBe(false);
      });
    }
  });

  it('클라이언트 오류(4xx) 발생 시, N/S 에러가 발생하지 않는다.', async () => {
    const statusArray = [400, 401, 403, 404];

    for (const status of statusArray) {
      // 클라이언트 에러의 경우 interceptor에서 error.response.data를 던짐
      (useRefreshTokenQuery as jest.Mock).mockReturnValue({
        isSuccess: false,
        isError: true,
        error: { code: status, message: 'Client Error' },
      });

      const { result } = renderHook(() => useAuth(), { wrapper: testWrapper });

      await waitFor(() => {
        expect(result.current.isAuthFinished).toBe(true);

        expect(result.current.isLoggedIn).toBe(false);
        expect(result.current.isNSError).toBe(false);
      });
    }
  });
});
```

MSW를 이용해 API 요청을 가로챌 수 있다면, 모킹 없이 더욱 실상황의 테스트를 진행할 수 있지만 현재 MSW가 없기 때문에 토큰을 가져오는 훅스인 `useRefreshTokenQuery`에 대한 응답을 다음과 같이 모킹하여 확인하였습니다.

### 4-2. User API Call 테스트

User API Call에 대한 테스트도 Refresh API Call과 동일하게 다음의 테스트를 작성하였습니다.

1. 네트워크 장애(offline) 발생 시, N/S 에러가 발생한다.
2. 서버 오류(5xx) 발생 시, N/S 에러가 발생한다.
3. 클라이언트 오류(4xx) 발생 시, N/S 에러가 발생하지 않는다.

테스트 코드는 Refresh API Call과 유사하므로 생략하였습니다.

### 4-3. 케이스별 화면 이동 테스트

이제 에러가 발생하는 상황에 대한 테스트를 진행하였으니, 각 성공 & 에러별로 화면이 잘 렌더링되는지를 테스트하겠습니다. 테스트할 항목은 다음과 같습니다.

1. 앱을 시작하면 스플래시 화면이 렌더링된다.
2. N/S 발생 시 네트워크 에러 화면이 렌더링된다.
3. 클라이언트 오류(4xx) 발생 시 로그인 화면이 렌더링된다.
4. 인증 성공 시 홈 화면이 렌더링된다.

```tsx
import { render, screen } from '@testing-library/react-native';
import App from "../App";

describe('RootNavigator 화면 이동 테스트', () => {
  it('Splash 화면이 렌더링된다.', () => {
    render(<RootNavigator />);

    expect(screen.getByTestId('splash-lottie')).toBeVisible();
  });

  it('N/S 발생 시 네트워크 에러 화면이 렌더링된다.', () => {
    (useAuth as jest.Mock).mockReturnValue({
      isLoggedIn: false,
      isAuthFinished: true,
      isNSError: true,
      refetch: jest.fn(),
    });

    render(<RootNavigator />);

    const errorMessage = screen.getByText('인터넷이 연결되어 있지 않아요');
    const retryButton = screen.getByText('다시 시도하기');

    expect(errorMessage).toBeVisible();
    expect(retryButton).toBeEnabled();
  });

  it('클라이언트 오류(4xx) 발생 시 로그인 화면이 렌더링된다.', async () => {
    (useAuth as jest.Mock).mockReturnValue({
      isLoggedIn: false,
      isAuthFinished: true,
      isNSError: false,
      refetch: jest.fn(),
    });

    render(<RootNavigator />);

    await waitFor(() => {
      expect(screen.getByText('로그인')).toBeVisible();
    })
  });

  it('인증 성공 시 홈 화면이 렌더링된다.', () => {
    (useAuth as jest.Mock).mockReturnValue({
      isLoggedIn: true,
      isAuthFinished: true,
      isNSError: false,
      refetch: jest.fn(),
    });

    render(<RootNavigator />);

    expect(screen.getByText('홈')).toBeVisible();
  });
});
```

위와 같이 모든 테스트를 작성하고 완료하면 짜잔! 테스트가 모두 통과하게 됩니다.

![Jest Verbose](assets/img/writing/15/test_verbose.png){: width="480" }

물론 테스트만으로는 완벽하지 않기 때문에 팀원들과 함께 QA도 진행해야 합니다.

## 5. 결론

이번 글에서는 네트워크 장애나 서버 오류로 인해 사용자가 강제로 로그아웃되는 문제를 해결하기 위한 과정을 공유했습니다. 

이슈를 분석한 결과, 기존 인증 플로우에서는 네트워크 장애와 클라이언트 오류(`4xx`), 서버 오류(`5xx`)에 대한 처리가 분리되지 않아 N/S 발생 시 사용자 인증이 실패하고 로그아웃되는 문제가 발생하고 있었습니다.

이를 해결하기 위해 여러 가지 대안을 고려하였으며, 최종적으로 Exponential Backoff Retry 후 네트워크 상태를 확인하는 방식을 적용하여 보다 안정적인 로그인 유지 전략을 설계할 수 있었습니다.

또한, 단순히 코드 수정에서 끝나는 것이 아니라, 실제 환경에서 어떻게 동작하는지 검증하기 위해 Jest를 활용한 테스트 코드도 작성하였습니다. 특히, 다양한 네트워크 환경과 예외 상황을 고려하여 테스트를 구성하고 API 응답을 모킹(Mock)하여 실전과 유사한 시뮬레이션을 진행하였습니다.

이번 블로깅을 통해 정말 많은 것들을 얻을 수 있었습니다.

> 이번 개선을 통해 얻은 인사이트
>
> 1. 네트워크 장애와 클라이언트 오류(`4xx`), 서버 오류(`5xx`)를 상황에 따라 분리하여 처리해야 한다.
> 2. 일정 시간 간격이 아닌 Exponential Backoff Retry를 활용하면 N/S로 인한 불필요한 API 요청을 줄일 수 있다.
> 3. 테스트 코드는 예쌍치 못한 버그를 방지하고 코드의 신뢰성을 크게 높여줄 수 있다.
> 4. [MSW(Mock Service Worker)](https://mswjs.io/)를 활용했더라면, 단순 모킹이 아닌 실제 환경을 기반으로 더욱 정교한 통합 테스트를 진행할 수 있다.
{: .prompt-info }

<br />

사실 이 블로그를 쓰기 위해 플로우를 고민하는 것부터 테스트 코드를 통해 안정화하는 작업까지 정말 오래 걸렸습니다. 테스트 코드 작성에 익숙하지 않는 탓도 물론 있겠지만요.. 😅

하지만 이 과정을 통해 플로우의 설계부터, 테스트 코드 작성의 중요성을 몸소 깨닫는 계기가 되었습니다. 특히나 테스트 코드 작성을 통해 단순히 코드가 동작하는 것만 확인하는 것이 아니라, 예상치 못한 버그를 방지하고 신뢰성을 높이는 과정이 필수적이라는 것을 배우게 되었습니다.

이 글이 비슷한 문제를 고민하는 분들께 도움이 되었길 바라며, 앱에서 네트워크 장애 대응을 고민하는 분들께 유용한 참고 자료가 되길 바랍니다. 긴 글 읽어주셔서 감사합니다! 🚀

> 이 글의 추가적인 개선점이 궁금하시다면, [네트워크 / 서버 에러 대응을 위한 추가적인 방안 적용](https://bangdori.kr/posts/handling-network-error-in-splash2)를 확인해보세요!