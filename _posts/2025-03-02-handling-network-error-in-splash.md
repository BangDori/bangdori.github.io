---
title: 네트워크 에러에도 끊김 없는 스플래시 화면 설계하기
date: 2025-03-06 09:18:00 +/-TTTT
categories: [Frontend, React Native]
tags: [react native, splash screen, network error, error handling, retry mechanism, error boundary, tanstack query, ux improvement]
sitemap:
  changefreq: daily
  priority: 0.5
---

> 목차
> 1. 개요
> 2. 문제 인식
> 3. 문제 접근
> 4. 테스트 시나리오 설계
{: .prompt-tip }

## 1. 개요

> **스플래시(Splash)**
>
> 모바일 앱 실행 시 가장 처음 만나게 되는 화면으로, 보통 1초에서 3초정도 이어집니다. 대부분의 앱은 일반적으로 스플래시 UI를 가지고 있습니다. - [1초의 디테일, 스플래시 시각보정.](https://brunch.co.kr/@shaun/60)
{: .prompt-info }

현재 서비스는 앱을 시작하면 스플래시가 시작되고, 스플래시가 시작되는 동안 인증 정보와 사용자 정보를 받아오도록 플로우가 구성되어 있습니다. staging에서 QA를 진행할 때마다 매번 문제가 없었는 것 같은데..

![Sentry Issue](assets/img/writing/15/sentry_issue.png){: width="720" }

어느날 이런 이슈가 발생했습니다..! 아니 Users를 확인해보니, 이미지에는 나와있지 않지만 이미 3명의 사용자가 강제로 로그아웃을 당했었습니다. 😭

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
> 네트워크가 연결되어 있지 않은 환경이라면 굳이 Retry를 시도할 필요가 없겠지만, 생각보다 네트워크가 불안정한 환경이 많기 때문에 Retry Option을 추가하였습니다. (ex. 엘리베이터, 지하 등)
{: .prompt-info }

### 🚀 3-5. 최종 결론

1번 대안은 앱 시작 시에만 연결 상태를 확인하기 때문에, 근본적인 해결책이 될 수 없다고 판단하여 제외하였습니다.

| \            | 2️⃣ Queue 방식 | 3️⃣ Retry & ErrorBoundary | 4️⃣ Network Check and Retry |
|---------------------------|---------------|---------------------------|------------------------------|
| **N/S 대응**  | ❌ Queue의 복잡한 예외 처리 문제 | ➖ 재시도 및 UI를 제공하지만 무한 루프 위험 | ✅ 점진적 Retry + 네트워크 확인 |
| **안정성**     | ➖ `5xx`에 대한 대응이 어려움  | ❌ 무한 루프 위험 | ✅ Retry 초과 시 네트워크 확인 후 재시도 |
| **확장성**     | ❌ Queue 관리의 부담 | ➖ API별로 ErrorBoundary 처리가 필요하지만 확장 가능 | ✅ Queue/ErrorBoundary 없이 N/S 대응이 가능하기에 유지보수가 용이하며 확장성 고려 가능 |

비교 결과, N/S 대응이 가능하며 안정성과 확장성 측면에서 강점을 가진 대안 4가 가장 적합하다고 판단하였습니다.

## 4. 테스트 시나리오 설계

TDD(Test-Driven Development) 방식이 항상 최선의 선택은 아니지만, 스플래시 화면과 같이 **잘못된 시나리오가 적용되면 치명적인 문제가 발생할 수 있는 경우**에는 테스트 시나리오를 먼저 설계하는 것이 중요합니다.

이 섹션에서는 앞서 개선된 플로우를 안전하게 적용하기 위해 **테스트 케이스를 설계하는 과정**과 주요 고려 사항을 다룹니다.

> 해당 섹션에서는 많은 테스트 코드가 포함되어 있습니다.
> 코드 예시가 많으므로, 가독성을 위해 필요한 경우에만 코드를 확인하는 것을 권장합니다.
{: .prompt-warning }

<br />

테스트는 [Jest](https://jestjs.io/) + [React Testing Library](https://testing-library.com/docs/)를 이용하여 진행하고, 필요한 api는 모킹하여 진행합니다.

### 4-1. 초기 로딩 테스트

1. 앱 실행 후 Splash 화면이 렌더링된다.

```tsx
import { render, screen } from "@testing-library/react";
import App from "../App"; // 실제 컴포넌트 경로

describe("초기 로딩", () => {
  it("Splash 화면이 렌더링된다.", () => {
    render(<App />);
    expect(screen.getByText("Splash")).toBeInTheDocument();
  });
});
```

### 4-2. Refresh API Call 테스트

1. 성공하면 User API Call이 정상적으로 호출된다.
2. 네트워크 장애(offline)일 경우 3회 Retry 후 Connection Error 페이지로 이동한다.
3. 서버 에러(5xx)일 경우 3회 Retry 후 Connection Error 페이지로 이동한다.
4. 클라이언트 에러(4xx)일 경우 Retry 없이 로그인 페이지로 이동한다.

```tsx
describe("Refresh API Call", () => {
  beforeEach(() => {
    jest.clearAllMocks();
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  const triggerAllTimers = async (times) => {
    for (let i = 0; i < times; i++) {
      jest.advanceTimersByTime(Math.pow(2, i) * 1000); // 1 → 2 → 4
      await waitFor(() => {});
    }
  };

  it("성공하면 User API Call이 정상적으로 호출된다.", async () => {
    api.refreshToken.mockResolvedValue({ success: true });
    api.getUserInfo.mockResolvedValue({ success: true });

    render(<App />);

    await waitFor(() => expect(api.refreshToken).toHaveBeenCalledTimes(1));
    await waitFor(() => expect(api.getUserInfo).toHaveBeenCalledTimes(1));
  });

  it("네트워크 장애(offline)일 경우 3회 Retry 후 Connection Error 페이지로 이동한다.", async () => {
    api.refreshToken.mockRejectedValue(new Error("Network Error"));

    render(<App />);

    await triggerAllTimers(3);

    await waitFor(() => expect(screen.getByText("Connection Error 페이지")).toBeInTheDocument());
    expect(api.refreshToken).toHaveBeenCalledTimes(3);
  });

  it("서버 에러(5xx)일 경우 3회 Retry 후 Connection Error 페이지로 이동한다.", async () => {
    api.refreshToken.mockRejectedValue({ response: { status: 500 } });

    render(<App />);

    await triggerAllTimers(3);

    await waitFor(() => expect(screen.getByText("Connection Error 페이지")).toBeInTheDocument());
    expect(api.refreshToken).toHaveBeenCalledTimes(3);
  });

  it("클라이언트 에러(4xx)일 경우 Retry 없이 로그인 페이지로 이동한다.", async () => {
    api.refreshToken.mockRejectedValue({ response: { status: 401 } });

    render(<App />);

    await waitFor(() => expect(screen.getByText("로그인 페이지")).toBeInTheDocument());
    expect(api.refreshToken).toHaveBeenCalledTimes(1);
  });
});
```

### 4-3. User API Call 테스트

1. 성공한 후, Splash가 종료되면 홈 화면이 렌더링된다.
2. 네트워크 장애(offline)일 경우 3회 Retry 후 Connection Error 페이지로 이동한다.
3. 서버 에러(5xx)일 경우 3회 Retry 후 Connection Error 페이지로 이동한다.
4. 클라이언트 에러(4xx)일 경우 Retry 없이 로그인 페이지로 이동한다.

```tsx
describe("User API Call", () => {
  it("성공한 후, Splash가 종료되면 홈 화면이 렌더링된다.", async () => {
    api.refreshToken.mockResolvedValue({ success: true });
    api.getUserInfo.mockResolvedValue({ success: true });

    render(<App />);

    await waitFor(() => expect(screen.getByText("Home Page")).toBeInTheDocument());
  });

  it("네트워크 장애(offline)일 경우 3회 Retry 후 Connection Error 페이지로 이동한다.", async () => {
    api.getUserInfo.mockRejectedValue(new Error("Network Error"));

    render(<App />);

    await triggerAllTimers(3);

    await waitFor(() => expect(screen.getByText("Connection Error 페이지")).toBeInTheDocument());
    expect(api.getUserInfo).toHaveBeenCalledTimes(3);
  });

  it("서버 에러(5xx)일 경우 3회 Retry 후 Connection Error 페이지로 이동한다.", async () => {
    api.getUserInfo.mockRejectedValue({ response: { status: 500 } });

    render(<App />);

    await triggerAllTimers(3);

    await waitFor(() => expect(screen.getByText("Connection Error 페이지")).toBeInTheDocument());
    expect(api.getUserInfo).toHaveBeenCalledTimes(3);
  });

  it("클라이언트 에러(4xx)일 경우 Retry 없이 로그인 페이지로 이동한다.", async () => {
    api.getUserInfo.mockRejectedValue({ response: { status: 403 } });

    render(<App />);

    await waitFor(() => expect(screen.getByText("로그인 페이지")).toBeInTheDocument());
    expect(api.getUserInfo).toHaveBeenCalledTimes(1);
  });
});
```

### 4-4. 최종 페이지 이동 테스트

1. 네트워크가 원활하지 않을 경우, Connection Error 페이지로 이동한다.
2. Refresh API Call과 User API Call이 모두 성공하면, 홈 페이지로 이동한다.
3. RTK가 존재하지 않을 경우 로그인 페이지로 이동한다.

```tsx
describe("최종 페이지 이동", () => {
  it("네트워크가 원활하지 않을 경우, Connection Error 페이지로 이동한다.", async () => {
    api.isConnectedNetwork.mockResolvedValue(false);

    render(<App />);

    await waitFor(() => expect(screen.getByText("Connection Error 페이지")).toBeInTheDocument());
  });

  it("Refresh API Call과 User API Call이 모두 성공하면 홈 페이지로 이동한다.", async () => {
    api.refreshToken.mockResolvedValue({ success: true });
    api.getUserInfo.mockResolvedValue({ success: true });

    render(<App />);

    await waitFor(() => expect(screen.getByText("Home Page")).toBeInTheDocument());
  });

  it("RTK가 존재하지 않을 경우 로그인 페이지로 이동한다.", async () => {
    api.getUserInfo.mockResolvedValue(null);

    render(<App />);

    await waitFor(() => expect(screen.getByText("로그인 페이지")).toBeInTheDocument());
  });
});
```

### 4-5. 고려해야 할 점

단위 테스트만으로는 모든 시나리오를 완벽하게 점검하기 어려운 한계가 있습니다.
특히, 아래와 같은 상황에서는 추가적인 테스트 방법이 필요할 수 있습니다.

#### 📌 네트워크 환경의 다양한 변동성 테스트

단위 테스트에서는 `mock`을 사용하여 네트워크 장애 및 응답에 대한 최소한의 시뮬레이션을 진행하지만, 실제 네트워크 환경에서 발생할 수 있는 변동성을 완전히 반영하기는 어렵습니다.

E2E 테스트를 수행하는 것 또한 하나의 방법이 될 수 있지만, 너무 많은 자동화 테스트는 오히려 개발 속도를 저하시킬 수 있으며, 실전에서의 예외적인 상황을 모두 반영하기 어렵다고 판단하였습니다.

따라서, 아래와 같이 중요한 핵심 흐름은 팀원들 간의 실사용 테스트를 통해 진행하도록 결정하였습니다.

1. 실제 기기에서 네트워크가 변동되는 상황(예: Wi-Fi ↔ LTE 전환, VPN 사용)을 테스트
2. 앱을 실행한 상태에서 백그라운드 전환 및 복귀 시 정상 동작 여부 확인
3. 예상치 못한 시나리오(예: 화면을 빠르게 여러 번 전환하는 경우)
4. 네트워크를 연결하지 않고 앱 실행 시 Connection Error가 나타나는지
5. Connection Error 페이지에서 네트워크 연결 후 재시도하면 정상적으로 로그인되는지

## 5. 성공하는 테스트를 작성하기 위해..