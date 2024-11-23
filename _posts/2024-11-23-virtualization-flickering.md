---
title: React Native FlatList 깜빡임(Flickering) 현상, 왜 생기고 어떻게 해결할까
date: 2024-11-23 18:27:00 +/-TTTT
categories: [Frontend, React Native, Optimization]
tags: [flatlist, flickering, optimization, reconciliation, virtualization]
image:
  path: assets/img/writing/11/virtualized_list.png
sitemap:
  changefreq: daily
  priority: 0.5
---

## 1. 개요

React Native에서 대용량 데이터를 스크롤러블하게 렌더링하기 위해 [ScrollView](https://reactnative.dev/docs/scrollview)와 [FlatList](https://reactnative.dev/docs/flatlist)를 활용합니다. 이 중 `FlatList`는 **가상화(Virtualization)**를 통해 대량 데이터를 효율적으로 처리할 수 있도록 설계되었습니다.

> `ScrollView`는 모든 데이터를 한 번에 렌더링하는 반면, `FlatList`는 가상화 기술을 활용해 화면에 보이는 항목만 렌더링합니다.
{: .prompt-info }

하지만 `FlatList`를 사용할 때 몇 가지 문제가 발생할 수 있습니다. 그중 하나가 바로 **스크롤 도중 발생하는 깜빡임(Flickering) 현상**입니다. 이러한 깜빡임의 주요 원인은 주로 데이터를 렌더링할 때 고유 key 값이 적용되어 있지 않아 발생합니다.

이 글에서는 문제의 원인을 심층적으로 분석하고, 왜 고유 key 값을 적용하는 것이 깜빡임 현상을 해결할 수 있는지 알아보겠습니다.

+ 추가로 고유 key 값을 적용했음에도 깜빡임 현상이 발생하는 가상화의 한계까지 다뤄보겠습니다.

## 2. Flickering 원인 분석

우선 원활한 테스트를 위해 각 아이템에 [Lorem Picsum](https://picsum.photos/) 이미지와 [Lorem Ipsum](https://www.lipsum.com/)을 추가하고 Perf Monitor를 통해 성능을 확인해 보겠습니다.

> **Perf Monitor**
>
> UI Frame과 JS Frame을 실시간으로 모니터링하고 최적화 포인트를 파악하는 데 사용되는 도구로 주로 다음과 같은 상황에 사용될 수 있습니다.
> 1. UI/JS 프레임 속도 확인
> 2. Memory 사용량 분석
> 
> 참고: [https://reactnative.dev/docs/profiling#3-find-your-process](https://reactnative.dev/docs/profiling#3-find-your-process)

실제 프로덕션에서 사용될 수 있는 UI와 상황을 고려하여 다음과 같이 테스트 환경을 구성하였습니다.

- key 값에 중복이 발생할 수 있다.
- 빠른 속도로 스크롤링한다.

<div align="center">
  <video controls="" width="240" height="400" muted="" autoplay="">
    <source src="https://github.com/BangDori/bangdori.github.io/raw/main/assets/img/writing/11/flickering_issue.mp4" type="video/mp4">
  </video>
</div>

현재 위 영상에서는 스크롤 시 `Encountered two children with the same key` 경고가 표시되고 있으며 지속적으로 깜빡임 현상이 발생하고 있습니다. (+ JS Frame이 1까지 내려가는 프레임 저하는 덤)

위 이슈와 깜빡임 현상에는 어떤 관계가 있을까요? 이러한 현상이 발생하는 이유를 알기 위해서는 먼저 React가 가상 돔을 사용하여 실제 DOM과의 차이를 비교하고 업데이트하는 과정인 재조정(Reconciliation) 알고리즘에 대해 알아야 합니다.

### 2-1. Reconciliation

React 내부에서는 가상 DOM과 실제 DOM을 비교하는 알고리즘을 가지고 있는데, 기존에 하나의 트리를 가지고 다른 트리/로 변환하기 위한 일반적인 해결책이 있습니다. 하지만 [이러한 알고리즘](https://grfia.dlsi.ua.es/ml/algorithms/references/editsurvey_bille.pdf)도 n개의 엘리먼트가 있는 트리에 대해 O(N^3)의 복잡도를 가집니다.

만약 React에 이 알고리즘을 적용하게 된다면, 1000개의 엘리먼트를 그리기 위해 10억 번의 비교 연산을 수행해야 합니다. 너무나도 비싼 연산이고, 이를 적용했다면 React가 현재의 위상을 유지하지 못했을 겁니다.

그래서 React는 두 가지 가정을 기반하여 O(N) 복잡도의 [휴리스틱 알고리즘](https://en.wikipedia.org/wiki/Heuristic_(computer_science))을 구현했습니다.

1. 서로 다른 타입의 두 엘리먼트는 서로 다른 트리를 만들어낸다.
2. 개발자가 key prop을 통해, 여러 렌더링 사이에서 어떤 자식 엘리먼트가 변경되지 않아야 할지 표시해 줄 수 있다.

두 개의 트리(가상 DOM과 실제 DOM)를 비교할 때, React는 두 엘리먼트의 루트(root) 엘리먼트부터 비교합니다. 이후의 동작은 루트 엘리먼트의 타입에 따라 달라집니다.

#### 2-1-1. 엘리먼트의 타입이 다른 경우

두 루트 엘리먼트의 타입이 다르면, React는 이전 트리를 버리고 완전히 새로운 트리를 구축합니다.

![diffing_algorithm_difference_element](assets/img/writing/11/diffing_algorithm_difference_element.png)

위 간단한 예시를 가지고 생각해 보겠습니다. 비교 알고리즘에 따라 우선, Root를 기준으로 자식 엘리먼트로 내려가며 실제 DOM과 가상 DOM을 비교합니다. 이때 다른 엘리먼트(가상 DOM의 `span`과 실제 DOM의 `div`)가 있다면, 해당 엘리먼트의 자식들에 대해서는 비교를 진행하지 않고 하위 자식을 모두 재구축합니다.

> **React가 선언적(declarative)이라고 말할 수 있는 이유 중 하나가 바로 이러한 방식을 가지고 있기 때문**입니다. React의 선언적 특성은 UI를 구성하거나 업데이트할 때 "무엇을 보여줘야 하는지"만 명시하고, "어떻게 변경할지"는 React가 알아서 처리한다는 점에서 기인합니다.
> 
> - 자세히 알아보기: [https://react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative](https://react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)
{: .prompt-info }

#### 2-1-2. 자식에 대한 재귀적 처리

DOM 노드의 자식들을 재귀적으로 처리할 때, React는 기본적으로 동시에 두 리스트를 순회하고 차이점이 있으면 변경을 생성합니다.

예를 들어, 자식의 끝에 엘리먼트를 추가한다면, 두 트리 사이의 변경은 잘 작동할 것입니다.

![diffing_algorithm_recursion](assets/img/writing/11/diffing_algorithm_recursion.png)

React는 두 트리에서 `<li>first</li>`가 일치하는 것을 확인하고, `<li>second</li>`가 일치하는 것을 확인합니다. 그리고 마지막으로 `<li>third</li>`를 비교하는데 실제 DOM에는 해당 엘리먼트가 없으니, 트리에 추가합니다.

하지만 이렇게 단순하게 구현할 경우, 아래와 같이 리스트의 맨 앞에 엘리먼트를 추가하면 성능이 좋지 않습니다.

![diffing_algorithm_recursion_bad](assets/img/writing/11/diffing_algorithm_recursion_bad.png)

React는 두 트리에서 `third`와 `first`를 먼저 비교하고, `first`와 `second`를 비교하기 때문에, 종속 트리를 그대로 유지하는 대신 모든 자식을 변경합니다. 이러한 비효율적인 비교는 데이터의 수가 많아질수록 더 큰 문제를 낳게 됩니다. 이러한 문제를 해결하기 위해 React는 key 속성을 지원합니다. 자식들이 key를 가지고 있다면 React는 key를 통해 기존 트리와 이후 트리의 자식들이 일치하는지 확인합니다.

![diffing_algorithm_recursion_key](assets/img/writing/11/diffing_algorithm_recursion_key.png)

React의 key가 추가된 시점부터는 React는 key를 이용해서 비교를 진행하기에 `"3"` key를 가진 엘리먼트만 새로 추가되었다는 사실을 알 수 있습니다. 이렇듯이 React의 key 속성은 리스트 비교 과정에서 **비효율적인 DOM 업데이트를 방지하고 성능을 최적화하는 데 중요한 역할**을 합니다.

그렇다면 고유한 key 값을 적용하여 React가 효율적으로 DOM을 비교하고 업데이트할 수 있도록 서버에서 전달해 주는 데이터의 고유 id 값을 적용하여 다시 확인해 보겠습니다. (서버 역할을 수행하도록 하기 위해 [MSW](https://mswjs.io/)를 이용하여 네트워크 요청을 가로채도록 하였습니다.)

<div align="center">
  <video controls="" width="240" height="400" muted="" autoplay="">
    <source src="https://github.com/BangDori/bangdori.github.io/raw/main/assets/img/writing/11/flickering_resolve_key.mp4" type="video/mp4">
  </video>
</div>

그 결과 놀랍게도 이전과는 180도 다른 성능으로 JS 프레임이 50대 이상 유지되며, 깜빡임 현상도 사라진 것을 확인할 수 있습니다. (이것이 바로 React 휴리스틱 알고리즘의 힘....!)

## 3. 가상화 (Virtualization)의 한계

> **가상화**
>
> 컴퓨팅에서 가상화는 물리적 컴퓨팅 리소스를 일련의 가상 머신, 운영 체제, 프로세스 또는 컨테이너로 분할할 수 있게 해주는 일련의 기술입니다.
> - 참고: [https://en.wikipedia.org/wiki/Virtualization](https://en.wikipedia.org/wiki/Virtualization)

앞서 언급했듯이, React Native의 FlatList에서 가상화 리스트는 대량의 데이터를 효율적으로 렌더링하기 위해 화면에 필요한 데이터만 렌더링하는 기술을 의미합니다. 이러한 방식은 컴퓨팅에서 일반적으로 말하는 가상화와는 다른 맥락이지만, 효율성을 높이기 위한 "가상 처리"라는 점에서는 유사한 원리가 적용됩니다.

<details>
  <summary><strong>React Native에서 시각적으로 확인하는 가상화</strong></summary>
  <div align="center">
    <video controls="" width="240" height="400" muted="" autoplay="">
      <source src="https://github.com/BangDori/bangdori.github.io/raw/main/assets/img/writing/11/virtualized_mount.mp4" type="video/mp4">
    </video>
    <img src="assets/img/writing/11/virtualized_list.png" width="480" alt='virtualized list image' />
  </div>
  <p>실제로 현재 화면에 보이는 아이템들은 mount되며, 화면에 보여지지 않는 이전 데이터들은 unmounted되는 것을 확인할 수 있습니다</p>
</details>

<br />

앞서 고유 key 값을 적용하여 깜빡임 현상이 사라진 것을 확인하였습니다. 하지만 과연 더 이상 문제가 없을까요?

<div align="center">
  <video controls="" width="240" height="400" muted="" autoplay="">
    <source src="https://github.com/BangDori/bangdori.github.io/raw/main/assets/img/writing/11/flickering_issue_on_scroll_up.mp4" type="video/mp4">
  </video>
</div>

위 영상에서 확인할 수 있듯, 위로 스크롤하는 경우에는 여전히 깜빡임 현상이 나타나고 있습니다. 이러한 문제는 **가상화 기술의 구조적 한계**와 관련이 있습니다.

이 상황에서 깜빡임 현상이 발생하는 주된 원인은 React가 **데이터 렌더링이 사용자의 스크롤 속도를 따라가지 못하기 때문**입니다. 아래로 스크롤하는 경우 다음과 같이 리스트의 끝부분에 도달했을 때, 새로운 데이터를 요청하고 응답받은 데이터를 렌더링하는 방식을 취하기에 응답받은 데이터의 수가 제한되어 있습니다.

![virtualized_list_on_scroll](assets/img/writing/11/virtualized_list_on_scroll.png)

하지만 위로 스크롤하는 경우 서버로 데이터를 요청하는 것이 아닌 이전에 받아온 `unmounted`된 데이터를 렌더링합니다. 그렇기 때문에 이 과정에서 렌더링 속도가 사용자의 스크롤 속도를 따라가지 못해 빈 화면이 나타나거나 깜빡이는 것처럼 보일 수 있는 것입니다.

## 4. Conclusion

React Native의 FlatList는 대용량 데이터를 효율적으로 처리하기 위한 강력한 도구이며, key 속성을 통해 React의 가상 DOM 비교를 최적화함으로써 깜빡임 현상을 해결할 수 있었습니다. 그러나 이러한 최적화에도 불구하고, 가상화 기술 자체의 구조적 한계로 인해 스크롤 속도가 렌더링 속도를 초과하는 경우 문제가 발생할 수 있음을 확인하였습니다.

그러나 굉장히 빠른 속도로 위로 스크롤하는 경우는 일반적인 사용자 시나리오와는 다소 거리가 있을 수 있습니다. 대부분의 사용자는 아래로 스크롤하며 콘텐츠를 탐색하고, 위로 스크롤하더라도 일반적인 속도로 탐색하므로 **이러한 문제가 모든 상황에서 치명적으로 작용하는 것은 아닙니다.**

그렇기에 각 애플리케이션에서 사용되는 리스트의 시나리오를 고려하면 좋을 것 같습니다.

> 추가적인 관점
> 
> 앞서 언급하지 않은 내용이지만 서버의 응답이 너무 빠른 경우, 거의 즉시 데이터들을 받아오기 때문에 위로 스크롤 시 발생하는 깜빡임과 동일한 결과를 초래할 수 있습니다. 따라서 API의 요청을 제한하기 위한 쓰로틀링(throttling)이나 디바운싱(debouncing) 등의 기법을 적극적으로 활용하는 것이 중요합니다.
{: .prompt-tip }

## 참고

- [재조정 (Reconciliation)](https://ko.legacy.reactjs.org/docs/reconciliation.html)
- [Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)
- [Reacting to Input with State](https://react.dev/learn/reacting-to-input-with-state)
- [Virtual Scroll - 가상 스크롤 구현](https://pepperminttt.tistory.com/56)