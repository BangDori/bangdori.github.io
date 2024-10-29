---
title: React Native nestedScrollView issue - Android
description: 중첩 스크롤을 이슈를 해결해보자!
date: 2024-10-29 20:32:00 +/-TTTT
categories: [Frontend, React Native]
tags: [react native, scrollview, flatlist, nestedScrollView, bubbling, capturing]
sitemap:
  changefreq: daily
  priority: 0.5
---

> 세 줄 요약
> 1. 중첩 ScrollView 사용 시 안드로이드에서는 스크롤이 정상적으로 작동하지 않음
> 2. Android에서는 상위 뷰가 모든 터치 이벤트를 가로채서 처리하는 방식으로 이벤트 전파 방식을 가짐
> 3. 중첩 ScrollViw를 react-native-gesture-handler의 ScrollView로 변경하여 해결
{: .prompt-tip }

## 1. 개요

React Native(이하 'RN')를 이용하여 앱을 제작하고 있는 도중 `ScrollView` 내부에 `ScrollView`가 중첩으로 존재하는 경우 안드로이드에서는 정상적으로 스크롤이 되지 않는 문제가 생겼습니다. 오늘은 이러한 문제의 원인은 무엇이며 왜 Android에서만 발생하는지, 그리고 어떻게 해결하는지 알아보겠습니다.

## 2. 문제의 원인

RN에서 `ScrollView`가 중첩될 경우, Android에서 스크롤이 작동하지 않는 것은 Android와 iOS의 이벤트 전파 방식 차이에서 비롯됩니다.

iOS에서는 하위에서 상위로 이벤트가 전파되는 방식을 가지고 있기 때문에 하위 요소의 Scroll이 정상적으로 작동하면 반면, 안드로이드에서는 기본적으로 상위 뷰가 모든 터치 이벤트를 가로채서 처리하며, 터치 이벤트가 하위 뷰로 내려가는 방식을 가지고 있습니다. 즉, 상위 `ScrollView`가 터치 이벤트를 우선적으로 받기 때문에, 하위에 중첩된 `ScrollView`나 `FlatList`는 스크롤 이벤트를 감지하지 못하고 상위 `ScrollView`가 우선권을 차지해버리는 문제가 발생합니다.

그렇다면 Android에서 자식의 ScrollEvent에 대응하기 위해 어떻게 해야 할까요?

## 3. 해결 방법

중첩된 `ScrollView`에서 안드로이드 스크롤 문제를 해결하는 몇 가지 방법이 있습니다.

### 3-1. react-native-gesture-handler의 ScrollView 사용하기

**react-native-gesture-handler**(이하 'GH')는 RN의 기본 터치 이벤트를 보완하기 위해 제공되는 라이브러리로, 이 라이브러리의 `ScrollView` 컴포넌트를 사용하면 터치 이벤트 전파와 스크롤 동작을 더 원활하게 처리할 수 있습니다. 이를 통해 Android에서 터치 이벤트가 상위와 하위 `ScrollView` 모두에 고르게 분배되도록 할 수 있습니다.

```tsx
import { ScrollView } from 'react-native';
import { ScrollView as GestureScrollView } from 'react-native-gesture-handler';

<ScrollView>
  <GestureScrollView>
    {/* 중첩된 콘텐츠 */}
  </GestureScrollView>
</ScrollView>
```

> **🧐 ScrollView를 사용할 때 RN의 `ScrollView`가 아닌 GH의 `ScrollView`만 사용하면 되지 않나요?**

[What is the difference between the ScrollView here and the one shipping with React Native?](https://github.com/software-mansion/react-native-gesture-handler/discussions/2152) 해당 discussion에서 Software Mansion 팀에서 일하고 있는 팀원의 언급에 따르면 일반적으로 `ScrollView` 안에 터치 가능 또는 제스처가 있는 경우 GH 라이브러리의 `ScrollView`를 사용해야 제스처/구성 요소가 스크롤링과 잘 어울린다고 설명합니다.

그렇기 때문에 첫 스크롤 영역에 대해서는 RN에서 제공해 주는 `ScrollView`를 사용하고, ScrollView 내부에 스크롤링이 다시 필요한 경우, 즉 중첩 스크롤이 필요한 경우 GH 라이브러리의 `ScrollView`를 사용하면 될 것 같습니다.

### 3-2. nestedScrollEnabled 옵션 사용

[React Native ScrollView](https://reactnative.dev/docs/scrollview) 공식 문서에 nestedScrollEnabled props가 있는데 해당 props를 통해서도 Android에서 중첩 스크롤을 가능하게 할 수 있습니다.

```tsx
import { ScrollView } from 'react-native';

<ScrollView>
  <ScrollView nestedScrollEnabled>
    {/* 중첩된 콘텐츠 */}
  </ScrollView>
</ScrollView>
```

하지만 공식 문서에서도 언급되어 있듯이 Android API 레벨 21+ 에서 중첩 스크롤이 활성화되기 때문에 API 레벨이 20보다 낮을 경우에는 정상적으로 동작하지 않을 수 있습니다.

## 4. 결론

과거 웹을 진행할 당시 버블링과 캡처링의 개념에 대해 단순하게 알고 있었는데, 중첩 스크롤 이슈를 처리하며 버블링과 캡처링의 개념에 대해서도 다시 되돌아볼 수 있었습니다.

- Android 이벤트 전파 방식: 상위 뷰가 모든 하위 뷰의 터치 이벤트를 가로채서 처리하며, 터치 이벤트가 하위 뷰로 내려가는 방식
- iOS 이벤트 전파 방식: 하위에서 상위로 이벤트가 전파되는 방식으로 버블링과 유사
- 캡처링: 상위에서 하위로 이벤트가 전달되는 방식입니다.
- 버블링: 하위에서 상위로 이벤트가 전달되는 방식입니다.

만약 이벤트 전파에 대한 개념 자체를 모르고 있었더라면 문제의 근본적인 원인과 해결 방법을 찾는데 많은 시간이 소요되었겠지만, 웹을 공부하며 이벤트 전파에 대한 개념을 학습한 경험이 있어 생각보다 쉽게 플랫폼별 차이로 인한 문제를 해결할 수 있었습니다.

## 참고

- [React Native ScrollView](https://reactnative.dev/docs/scrollview)
- [NestedScrollView](https://developer.android.com/reference/androidx/core/widget/NestedScrollView)
- [React Native) 모달 안쪽에ScrollView / FlatList를 넣고 싶다면](https://velog.io/@2ast/React-Native-%EB%AA%A8%EB%8B%AC-%EC%95%88%EC%AA%BD%EC%97%90ScrollView-FlatList%EB%A5%BC-%EB%84%A3%EA%B3%A0-%EC%8B%B6%EB%8B%A4%EB%A9%B4)
- [React-native nested scroll view](https://velog.io/@loopback_log/React-native-nested-scroll-view)
- [nestedScrollEnabled prop not working in ScrollView (Android) with horizontal direction](https://github.com/facebook/react-native/issues/21436)
- [[Android] NestedScrollView에 대해 알아보자!](https://velog.io/@kimbsu00/Android-7#scrollview%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%96%88%EC%9D%84-%EB%95%8C%EC%9D%98-%EB%AC%B8%EC%A0%9C%EC%A0%90)