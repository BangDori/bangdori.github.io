---
title: React Native 페이지 이동시 잔상이 남는 현상 해결
description: 페이지 이동을 더욱 쾌적하게
date: 2024-10-26 12:48:00 +/-TTTT
categories: [Frontend, React Native]
tags: [react native, stack navigator, margin, padding]
image:
  path: assets/img/thumbnail/margin_and_padding.png
sitemap:
  changefreq: daily
  priority: 0.5
---

> 세 줄 요약
> 1. React Native에서 페이지 전환 시 이전 페이지의 잔상이 남는 현상 발생
> 2. 이로 인해 페이지 전환 시 성능 저하가 발생하고 사용자 경험에 부정적 영향을 미침
> 3. Screen에 적용된 cardStyle의 margin을 padding으로 변경하여 문제 해결
{: .prompt-tip }

## 1. 개요

React Native에서 화면 전환을 위해 Stack Navigator를 이용하던 중 이전 페이지의 잔상이 남는 문제가 발생했습니다. 이 문제는 페이지 전환 시 이전 페이지가 완전히 사라지지 않고 옆에 남는 현상으로, 성능 저하와 사용자 경험에 영향을 미치게 되었습니다. 이 글에서는 이러한 문제를 해결하여 보다 쾌적한 페이지 전환을 구현하는 방법을 소개합니다.

## 2. 문제 원인

{% raw %}
```tsx
const Stack = createStackNavigator<AuthStackParamList>();

export function AuthStackNavigator() {
  return (
    <Stack.Navigator
      initialRouteName={authNavigations.AUTH_HOME}
      screenOptions={{ headerShown: false }}
    >
      <Stack.Screen
        name={authNavigations.AUTH_HOME}
        component={AuthHomeScreen}
        options={{
          headerShown: false,
          cardStyle: { marginHorizontal: 20 }, // 🚨 문제가 발생하는 지점
        }}
      />
      <Stack.Screen
        name={authNavigations.LOGIN}
        component={LoginScreen}
        options={{
          headerTitle: '로그인',
          cardStyle: { marginHorizontal: 20 }, // 🚨 문제가 발생하는 지점
        }}
      />
    </Stack.Navigator>
  );
}
```
{% endraw %}

잔상이 남는 이유는 StackNavigator에 포함된 Screen 컴포넌트의 cardStyle에 margin이 적용되어 있기 때문입니다. margin은 외부 간격을 설정해 페이지 전환 시 애니메이션이 진행되면서 이전 페이지의 흔적이 남게 만듭니다. 이전 페이지의 흔적이 남는 이유는 각 페이지(AuthHomeScreen, LoginScreen)의 너비가 기기의 전체 너비(screenWidth)를 가지는 것이 아닌 margin의 특성으로 인해 screenWidth - 40만큼 가지게 되어 발생합니다.

![react native afterimage](assets/img/writing/8/react_native_afterimage.png){: width="360" }
_좌: 현재 페이지, 우: 페이지 이동 중_

이에 따라 위 이미지처럼 페이지 이동 시 페이지 이동 중 좌측 여백을 확보하는 대신 이전 페이지가 임시로 옆에 떠 있는 것처럼 보이게 되며, 애니메이션이 완료된 시점에서도 일시적으로 이전 페이지의 잔상이 남아있게 됩니다.

## 3. 해결

문제 해결 방법은 정말 간단합니다.

margin 대신 padding을 사용하여 내부 여백을 설정하면, 페이지들의 너비 자체는 screenWidth를 그대로 유지하면서, 페이지 내부의 요소들에 대해 간격이 적용되기 때문에 페이지 이동 시 간격을 유지하면서도 잔상 문제를 방지할 수 있습니다. 아래와 같이 cardStyle의 margin을 padding으로 수정하면 끝입니다.

{% raw %}
```tsx
export function AuthStackNavigator() {
  return (
    <Stack.Navigator
      initialRouteName={authNavigations.AUTH_HOME}
      screenOptions={{ headerShown: false }}
    >
      <Stack.Screen
        name={authNavigations.AUTH_HOME}
        component={AuthHomeScreen}
        options={{
          headerShown: false,
          cardStyle: { paddingHorizontal: 20 }, // ✅ 해결된 지점
        }}
      />
      <Stack.Screen
        name={authNavigations.LOGIN}
        component={LoginScreen}
        options={{
          headerTitle: '로그인',
          cardStyle: { paddingHorizontal: 20 }, // ✅ 해결된 지점
        }}
      />
    </Stack.Navigator>
  );
}
```
{% endraw %}

## 4. 마치며

사실 오늘 포스팅의 내용은 새로운 기술, 성능 개선과 관련한 내용을 다루지는 않았습니다. 어떻게 보면 프론트엔드 개발자로서 margin과 padding의 사용을 헷갈릴 수 있냐고 생각할 수 있지만, 이러한 스타일 속성 하나를 실수하여 성능에 영향을 미칠 수 있다는 점에 신선한 충격을 받아 이 포스트를 작성하게 되었습니다.

스타일 속성 하나하나, 코드 한 줄을 작성하더라도 어떤 의미로 작용하게 될지, 어떤 영향을 미치게 될지를 다시 한번 생각해 볼 수 있는 계기가 된 글이었습니다.