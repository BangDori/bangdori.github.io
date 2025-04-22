---
title: 크롬 브라우저 구조 위에서, 사용자에게 감동을
date: 2025-04-22 22:53:00 +/-TTTT
categories: [Frontend Insight, Browser]
tags: [Chrome Architecture, Browser Internals, IPC, Multi-Process, Frontend UX, Tab Sync, Storage Event, Renderer Process, Browser Process]
image:
  path: assets/img/writing/18/multi_process_architecture_chromium.png
sitemap:
  changefreq: daily
  priority: 0.5
---

> 목차
> 1. 개요
> 2. The Chromium Projects
> 3. 크롬의 IPC 구조
> 4. 멀티탭 환경에서 사용자에게 감동을
{: .prompt-tip }

## 1. 개요

평소 다른 서비스를 구경하면서 UI/UX 측면의 영감을 얻거나, 구현 방식에 대해 살펴보는 편인데요. 최근에 한 서비스를 살펴보다가 궁금증이 생기게 되었어요.

> “프론트엔드 개발을 하면서 멀티 프로세스를 고려해본 적이 있었던가?”

음 자바스크립트에서 웹 워커를 사용해서 멀티 쓰레드처럼 처리해본 경험도 없는데, 멀티 프로세스라뇨??

그나저나 이런 궁금증은 왜 생겼냐고요? 주변 카페 찾으려고 탭 여러 개 켜두고, 한 탭에서 로그인했을 때 <span style="color: #00CA5B;">나머지도 다 로그인 상태일 줄 알았는데, 아니더라고요.</span> 물론 새로고침하면 로그인되긴 하지만… n-1개의 탭을 일일이 새로고침? 귀찮잖아요.

> “그냥 탭들끼리 실시간으로 연동해주면 안되나?”

이 생각이 계기가 되어 크롬 브라우저의 내부 구조를 조금씩 들여다보게 되었고, 생각보다 흥미로운 구조들을 많이 발견하게 되었어요.

이 글에서는 멀티 프로세스, 멀티 스레드 같은 개념 자체를 깊이 다루기보다는, **크롬 브라우저 내부에서 프로세스들이 어떻게 통신하는지**, 그리고 **각 탭들 간 데이터를 서로 공유**하려면 어떻게 해야하는지를 공유해려고 합니다.

## 2. The Chromium Projects

크롬 브라우저의 내부 구성과 프로세스 간 통신 구조를 이해하기 위해, 먼저 Chromium 프로젝트에 대해 간단히 살펴보겠습니다.

[Chromium](https://github.com/chromium/chromium)은 크롬, 오페라, 마이크로소프트 엣지 등 여러 인기 브라우저의 기반이 되는 오픈소스 프로젝트로, 기존 브라우저들이 가지고 있던 여러 문제점을 개선하기 위해 시작되었습니다.

대표적인 문제는 다음과 같습니다:

1. 단일 프로세스 구조
    - IE와 초기 Fiexfox는 하나의 프로세스 내에서 모든 탭과 기능을 처리했습니다. 이로 인해 **한 탭에서 크래시가 발생하면 전체 브라우저가 종료**되는 구조적 한계가 있었습니다.
2. 느린 JavaScript 엔진
    - 당시 브라우저는 복잡한 웹 애플리케이션을 돌리기엔 JS 실행 속도가 충분히 빠르지 않았습니다.

Chromium은 이런 문제를 해결하기 위해 **멀티 프로세스 아키텍처**와 고성능 JavaScript 엔진(V8) 등의 기술을 도입하며 지금의 크롬 아키텍처의 기반을 마련하게 됩니다.

### 2-1. Multi Process Architecture

![App Workflow](assets/img/writing/18/multi_process_architecture_chromium.png){: width="640" }
_출처: [https://developer.chrome.com/blog/inside-browser-part1](https://developer.chrome.com/blog/inside-browser-part1?hl=ko)_

크롬은 하나의 프로세스에서 모든 작업을 수행하는 방식을 탈피하고자, 브라우저의 각 구성 요소를 별도의 프로세스로 분리하는 **멀티 프로세스 아키텍처**를 도입했습니다.

대표적으로 다음과 같은 프로세스들로 분리되었습니다:

|      프로세스      | 역할 |
|------------------|-----|
| Browser Process  | 브라우저의 메인 컨트롤 센터. 상단 브라우저 UI, 하단 UI, 네트워크, 저장소 등 |
| Renderer Process | 웹페이지 렌더링, JS 실행 등 실제 콘텐츠 처리 |
| Utility Process  | 이미지 디코딩, PDF 렌더링 등 특정 기능 전담 처리 |
| GPU Process      | 그래픽 가속 처리 담당 (WebGL, CSS transform 등) |
| Plugin Process   | 외부 플러그인 처리 |

각 프로세스들은 서로 독립적으로 실행되며, 문제가 생겨도 다른 프로세스에 영향을 주지 않도록 [샌드박스](https://ko.wikipedia.org/wiki/%EC%83%8C%EB%93%9C%EB%B0%95%EC%8A%A4#:~:text=%EC%83%8C%EB%93%9C%EB%B0%95%EC%8A%A4(Sandbox)%EB%8A%94%20%EC%99%B8%EB%B6%80,%EC%9D%84%20%EC%9C%84%ED%95%9C%20%ED%85%8C%EC%8A%A4%ED%8A%B8%20%ED%99%98%EA%B2%BD%EC%9D%B4%EB%8B%A4.) 환경에서 격리되어 있습니다. 

이처럼 크롬은 멀티 프로세스 구조를 통해 안정성과 보안성을 확보했으며, 사용자 입장에서는 하나의 탭이 멈추더라도 다른 탭은 정상적으로 작동하는 더 쾌적한 환경을 경험할 수 있게 된 것입니다.

### 2-2. 멀티 프로세스인데 왜 탭끼리 영향을 줄까?

이론상으로 크롬은 명확한 멀티 프로세스 아키텍처를 가지고 있지만, 실제로 브라우저를 사용하다 보면 **한 탭의 동작이 다른 탭에 영향을 주는** 경우를 종종 경험하게 됩니다. 저도 처음에는 이런 상황을 보면서 헷갈렸던 기억이 있어요. <span style="color: #00CA5B;">멀티 프로세스 구조라면 탭끼리 완전히 독립적으로 동작해야 하지 않나?</span> 라는 생각이 자연스럽게 들더라고요.

그러나 여기서 말하는 **“프로세스가 분리되어 있다”**는 건 안정성과 보안을 위한 논리적 격리를 의미할 뿐, 물리적인 자원(CPU, 메모리, GPU 등)을 완전히 분리한다는 의미는 아닙니다.

즉, 각 프로세스는 서로 다른 영역에서 실행되긴 하지만, 결국 같은 컴퓨터의 하드웨어 자원을 공유하기 때문에 **한 탭에서 CPU를 과도하게 점유하거나, 메모리를 많이 할당하면** → **다른 탭에도 간접적으로 영향을 줄 수 있습니다.**

그래서 탭이 느려지거나 렌더링이 멈추는 상황을 보고 “어? 이거 멀티 프로세스가 아니라 멀티 쓰레드처럼 작동하는 거 아냐?” 라고 오해하기 쉬운 거죠.

하지만 그런 상황도 **멀티 프로세스 구조 내부에서 발생한 자원 충돌 문제**일 뿐이고, 브라우저는 여전히 프로세스 간 격리를 유지하며 동작하고 있는 것입니다.

## 3. 크롬의 IPC 구조

> **Origin**
>
> 출처(Origin)란 URL 구조에서 Protocl, Host, Port를 합친 것을 의미합니다.
{: .prompt-info }

크롬에서 동일한 origin으로 여러 탭을 켜두고, 로컬스토리지에 데이터를 추가하면 다른 탭에도 데이터가 실시간으로 동기화되는 것을 확인할 수 있습니다.

하지만 브라우저의 각 탭은 서로 다른 렌더러 프로세스로 실행되는데, 어떻게 실시간으로 정보를 공유할 수 있을까요?

### 3-1. 크롬 탭은 어떻게 통신할까

크롬은 각 탭을 별도의 Renderer Process로 실행하여 안정성과 보안을 확보하고 있습니다. 여기서 각 탭이 서로 통신을 하기 위해서는 프로세스 간 통신(IPC, Inter-Process Communication) 과정을 거쳐야 합니다. <span style="color: #00CA5B;">크롬은 IPC를 탭 간 직접적인 연결이 아닌 브라우저 프로세스를 이용하는 방식</span>으로 처리하고 있습니다.

![IPC Process Architecture](assets/img/writing/18/chrome_ipc_process_architecture.png){: width="640" }
_출처: [Multi-process Architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture/)_

즉, 위 그림과 같이 Renderer <-> Renderer 직접 통신은 허용하지 않고, 항상 Renderer <-> Browser <-> Renderer의 구조로 Channel을 통해 통신이 이루어집니다.

> 🧩 그렇다면, 왜 탭들끼리 직접 연결하지 않고 브라우저 프로세스를 거칠까요?

##### 📌 이유 1: 보안

- 탭 간 직접 통신을 허용하면 XSS나 Origin 위조 등의 보안 이슈 발생 가능성이 증가
- 브라우저 프로세스가 중계자 역할을 함으로써 보안성과 검증 과정을 거칠 수 있음

##### 📌 이유 2: 중앙 관리

- 네트워크, 파일 시스템, 쿠키, localStorage 등은 공통 자원이므로 중앙에서 관리되어야 함
- 모든 Renderer가 직접 자원에 접근하면 충돌, 데이터 불일치 문제 발생 가능

##### 📌 이유 3: 확장성과 유지 보수

- 브라우저 프로세스가 중간자 역할을 하면 구조가 명확해지고, 확장성이 좋아짐

이제 크롬 탭 간 IPC가 왜 브라우저 프로세스를 통해 중계되는 구조인지 이해했으니, 실제로 크롬은 어떤 방식으로 이 통신을 구현했을지 알아보겠습니다.

### 3-2. 초창기 IPC 방식: Named Pipe

> **Pipe vs Named Pipe**
> 
> - Pipe: 부모 자식 간 단 방향 통신을 할 때 사용하는 IPC 방식
> - Named Pipe: 다른 프로세스와 단 방향 통신을 할 때 사용하는 IPC 방식
{: .prompt-info }

크롬의 초기 버전은 Windows의 Named Pipe, Linux의 Unix Domain Socket을 기반으로 IPC를 구현했습니다.

실제 Chromium의 [IPC 관련 코드](https://github.com/adobe/chromium/blob/master/ipc/ipc_channel_win.cc)를 살펴보면 `CreateNamedPipe`, `ConnectNamedPipe`와 같은 시스템 콜과 `CreatePipe`를 이용해 Named Pipe 방식으로 동작한다는 것을 확인할 수 있습니다. - [Named Pipe 코드 확인하기](https://github.com/CorTorE/js-lab/blob/main/issue-01/named_pipe.c)

하지만 이러한 Named Pipe 기반 IPC 시스템에는 분명 한계가 존재했습니다.

1. 플랫폼에 종속적이다.
    - Named Pipe는 Windows 중심, Unix Domain Socket은 Linux 중심
    - Chromium은 크로스 플랫폼 브라우저이기 때문에, 각 플랫폼마다 다른 IPC 구현을 유지 -> 코드 중복 & 관리 복잡성 증가
2. 유연하지 않은 메시지 구조
    - 기존 IPC 시스템에서는 [메시지를 정의할 때 C++ 매크로](https://github.com/adobe/chromium/blob/master/ipc/ipc_message_macros.h#L215-L426) 사용
    - 이로 인해 타입 안정성 부족, 확장성 제한이라는 한계점
3. 보안 취약성
    - 기존 IPC 시스템은 플랫폼에 종속적인 구현으로 인해 메시지 포맷이 정형화되어 있지 않음
    - 입력 검증 부족 및 플랫폼별 차이로 인해 일관된 보안 정책 적용 어려움

이러한 문제를 개선하고자 새로운 IPC 시스템인 Mojo를 도입하게 됩니다.

> Mojo에 대해 더 깊이있는 정보를 원하신다면 아래 블로그와 영상을 참고해보세요!
> - [How Chromium Got its Mojo?](https://blogs.igalia.com/gyuyoung/2020/05/11/how-chromium-got-its-mojo/)
> - [Mojo - Chrome’s inter-process communication system (Chrome University 2019)](https://www.youtube.com/watch?v=o-nR7enXzII)

## 4. 멀티탭 환경에서 사용자에게 감동을

이 글은 <span style="color: #00CA5B;">"크롬 브라우저에서 탭 간 통신을 통해 UX를 개선할 수 있는 방법이 없을까?"</span>라는 질문에서 출발했습니다.

다만 이 질문을 탐구하다 보니, 멀티 프로세스 구조나 프로세스 간 통신 같은 내용을 먼저 이해할 필요가 있더라고요. 그래서 본론에 들어가기 전, **게시물의 절반 이상이 크롬의 구조에 대한 배경 설명**으로 채워졌습니다.

조금 길게 느껴지셨을 수도 있지만, 이후 내용을 이해하는 데 꼭 필요한 기반으로 생각해주시면 감사하겠습니다.

### 4-1. 어떤 부분에서 필요할 수 있을까?

대부분의 사용자는 한 탭에서만 웹사이트를 이용하지만, **여러 탭을 동시에 띄워두고 사용하는 경우**도 꽤 자주 있습니다. 이때, 탭 간 동기화가 되지 않는다면 어떤 일이 생길까요?

- 언어 설정 / 테마 변경 등의 정보가 즉시 반영되지 않음
- 로그인 후 다른 탭은 여전히 로그아웃 상태

![동기화가 되지 않는 경우](assets/img/writing/18/desynchronization.png){: width="640" }
_동기화가 정상적으로 이루어지지 않는 경우_

한 탭에서 다크 테마로 변경했지만, 다른 탭에서는 여전히 라이트 테마로 보이고 있습니다. 물론 새로고침하면 브라우저 프로세스의 저장소에 저장된 테마 값을 반영하겠지만, n개의 탭을 켜뒀다면 n-1번 새로고침해야 한다는 이야기입니다.

굉장히 번거롭고 비효율적이죠.

> 🧐 ...와닿지 않으신가요?

그렇다면, 우리가 흔히 알고 있는 시간복잡도 개념을 여기에 한번 적용해볼까요? 사용자는 <span style="color: #FF6262;">다른 탭과의 동기화를 위해 총 O(N) 번의 새로고침이라는 작업을 수행</span>해야 합니다.

이 정도면 UX 관점에서도 충분히 최적화 대상이겠죠.

### 4-2. 사용자에게 감동 포인트 한 스푼 더하기

사용자에게 감동을 주는 건 생각보다 어렵지 않습니다. 크롬은 이미 내부적으로 탄탄한 구조와 IPC 시스템을 갖추고 있고, 우리는 그걸 **잘 활용해주기만** 하면 되니까요.

```js
// 로컬스토리지 변경 이벤트 리스너 등록
window.addEventListener("storage", (event) => {
  if (event.key === "theme") {
    // 테마 변경 감지
    const newTheme = event.newValue;
    document.documentElement.setAttribute("data-theme", newTheme);
    this.updateThemeButton(newTheme);
  }  else {
    // 기타 로컬스토리지 변경 감지
    view.logStorageEvent(event);
  }
});
```

브라우저에 storage 이벤트를 추가해주면 우리는 다음과 같이 탭 간 동기화가 이루어지는 것을 확인할 수 있습니다.

<div align="center">
  <video controls="" width="640" height="400" muted="" autoplay="">
    <source src="https://github.com/BangDori/bangdori.github.io/raw/main/assets/img/writing/18/browser_ipc.mp4" type="video/mp4">
  </video>
</div>
<br />

이렇게 엄청난 구조 위에, 단 몇 줄의 코드만 얹어도 사용자에게는 확연한 차이로 느껴지는 감동을 줄 수 있게 됩니다. 대단하지 않나요?

지금은 테마 변경이었지만, 여기서 우리의 상상력이 발휘된다면? 로그인, 알림, 심지어는 공동 편집까지 더 많은 감동을 줄 수 있지 않을까요?

### 4-3. 감동이 이루어지는 내부 과정

스토리지 이벤트를 통해 변경사항을 감지하고 자동으로 동기화해주는 과정을 되게 단순하게 말해줬지만 실제로는 아래와 같은 과정이 이루어지고 있습니다.

![스토리지 동기화 플로우](assets/img/writing/18/localstorage_sync_flow.png){: width="640" }
_스토리지 동기화 플로우_

1. Origin A의 탭 A에서 `localStorage.setItem()`을 호출합니다.
    - 이 명령은 렌더러 프로세스에서 시작되지만, **브라우저 프로세스로 전달**됩니다.
2. 브라우저 프로세스는 해당 origin의 localStorage에 데이터를 저장합니다.
    - 동시에 **디스크에도 반영해 영구적으로 보관**합니다.
3. 저장이 완료되면, 같은 origin의 다른 탭에 **storage 이벤트를 전파**합니다.
    - 단, setItem을 호출한 탭에는 이벤트가 발생하지 않습니다.
4. 렌더러 프로세스가 스토리지 이벤트를 수신합니다.
    - 등록된 `window.addEventListener("storage", ...)` 핸들러가 실행되어 새로운 값으로 UI나 상태를 업데이트합니다.

사실 이 과정 자체도 보기 좋게 단순화한 흐름입니다. 내부적으로는 IPC, 보안 검증, 큐 처리 등 수많은 메커니즘이 조용히 작동하고 있습니다.

## 결론

이 글에서는 크롬 브라우저의 내부 구조와 통신 원리에 대해 다루어 보았습니다. 이 글을 작성하면서 느낀점이 한 가지 있습니다.

> **"기술은 결국 사용자의 불편함을 줄이고, 더 나은 경험을 만들어주기 위한 도구"**
{: .prompt-info }

시대가 변화하고 개발 환경이 더 편리해지는 지금, 프론트엔드 개발자는 단순히 화면을 구현하는 역할을 넘어, 사용자에게 더 나은 경험을 설계하고 전달하는 역할까지 함께 고민해야 한다고 생각합니다.

복잡한 내부 구조를 이해하고, 그 위에 단순한 감동을 얹는 것.

그게 바로 프론트엔드 개발자가 지향해야 할 방향 아닐까요?

긴 글 읽어주셔서 감사합니다.

<br />

> **feat. 함께 깊고 깊은 심해 영역을 파헤치실 분을 구해요 👀**
>
> 프론트엔드의 깊은 영역을 분석하고, 이를 바탕으로 새로운 경험을 만들어가는 여정을 즐기시는 분들은 편하게 [LinkedIn](https://www.linkedin.com/in/bangdori/)으로 연락 주세요!
>
> 관련 예제 및 스터디 기록은 아래 저장소에서 확인하실 수 있습니다:
> - [🔗 스터디 저장소 - js-lab](https://github.com/CorTorE/js-lab/tree/main)

## 참고

- [최신 웹브라우저 들여다보기 (1부)](https://developer.chrome.com/blog/inside-browser-part1?hl=ko)
- [The Chromium Projects Multi-process Architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture/)
- [The Chromium Projects Inter-process Communication (IPC)](https://www.chromium.org/developers/design-documents/inter-process-communication/)
- [Difference between pipes and named pipes](https://atrystwithprogramming.wordpress.com/2015/10/15/difference-between-pipes-and-named-pipes/)
- [[리눅스 IPC 프로그래밍-이론과 실습] Live Programming(named pipe)](https://www.youtube.com/watch?v=CbbzmB8TPXk)
- [How Chromium Got its Mojo?](https://blogs.igalia.com/gyuyoung/2020/05/11/how-chromium-got-its-mojo/)
- [Window: storage event](https://developer.mozilla.org/en-US/docs/Web/API/Window/storage_event)