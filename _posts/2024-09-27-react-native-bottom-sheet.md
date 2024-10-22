---
title: React Native에서 사용되는 바텀 시트의 높이를 고정해 보자
description: 못말리는 짱구도 아니고 못말리는 바텀 시트
date: 2024-09-27 01:49:00 +/-TTTT
categories: [Frontend, React Native]
tags: [app, react native, react-native-bottom-sheet]
image:
  path: assets/img/thumbnail/bottom_sheet.png
sitemap: 
    changefreq : daily
    priority : 0.5
---

> 세 줄 요약
> 1. `@gorhom/bottom-sheet` iOS, Android에서 bottom-sheet의 UI가 동일하게 구성되지 않음
> 2. 높이가 동적으로 적용되어 있기 때문에 발생한 문제였음!
> 3. `enableDynamicSizing`을 `false`로 설정하기!
{: .prompt-tip }

## 1. 개요

바텀 시트를 구현하기 위해 `@gorhom/bottom-sheet` 라이브러리를 사용하고 있는데 하단에 고정해 둔 버튼이 예상대로 렌더링 되지 않는 문제가 발생하였다.

## 2. 문제 원인

![React Native Bottom Sheet](assets/img/writing/2/bottom_sheet_before.png)
_좌 iOS, 우 Android_

동일한 높이의 `snapPoints`를 설정해 두고 렌더링 한 결과 위 이미지와 같이 iOS에서는 예상대로 버튼이 렌더링 되지만 Android에서는 예상과는 다르게 렌더링 되는 문제가 발생하였다.

> 포스팅을 다 한 시점에서 알게 되었는데 iOS에서도 10번 중의 1번 정도 위 문제가 발생하고 있었다.

```tsx
function BottomSheet() {
  const bottomSheetModalRef = useRef<BottomSheetModal>(null);
  const snapPoints = useMemo(() => ['50%'], []);

  const handlePresentModalPress = () => bottomSheetModalRef.current?.present();

  return (
    <View style={styles.container}>
      <Pressable style={styles.presentButton} onPress={handlePresentModalPress}>
        <Text style={styles.presentText}>Present Modal</Text>
      </Pressable>
      <BottomSheetModal
        ref={bottomSheetModalRef}
        index={1}
        snapPoints={snapPoints}
      >
        <BottomSheetView style={styles.contentContainer}>
          <Text style={styles.text}>Awesome 🎉</Text>

          <Pressable style={styles.buttonContainer}>
            <View style={styles.button}>
              <Text style={styles.buttonLabel}>버튼</Text>
            </View>
          </Pressable>
        </BottomSheetView>
      </BottomSheetModal>
    </View>
  );
}

const styles = StyleSheet.create({
  // ...

  buttonContainer: {
    position: 'absolute',
    bottom: 44
  }
})
```

코드 자체에서 문제가 있는 것인지 확인해 보았지만, Android 기기에 따라 다르게 렌더링 되도록 작성한 코드가 없었기에 코드상에 문제는 없다고 생각했다. 그렇다면 `@gorhom/bottom-sheet`에서 제공해 주는 바텀 시트가 **동적으로 위치를 설정할 수 있기 때문에 그런 것은 아닐까**하여 확인해 본 결과 아래 이미지와 같이 버튼이 하단에 정상적으로 렌더링 되고 있는 것을 확인할 수 있었다.

![React Native Bottom Sheet](assets/img/writing/2/android_bottom_sheet.png){: width="360" }
_Android Bottom Sheet_

### 2-1. snapPoints 분석

잠깐 `snapPoints`의 의미부터 짚고 넘어가면 좋을 것 같다. [React Native Bottom Sheet](https://gorhom.dev/react-native-bottom-sheet/props#snappoints)에서 `snapPoinst` 속성을 다음과 같이 설명하고 있다.

> `snapPoints`: 바텀 시트가 스냅 될 지점입니다. 지점은 아래에서 위로 정렬되어야 합니다. 숫자, 문자열 또는 혼합 배열을 허용합니다.

바텀 시트가 스냅 될 지점이라는 것은 풀어서 말하면 바텀 시트가 특정 높이로 스냅(고정) 될 위치를 의미한다. 그렇다면 `snapPoints`를 하나의 값만으로 지정한 상황에서는 반드시 하나의 고정 위치만을 가져야 할 텐데 **바텀 시트가 렌더링 될 때 Android에서는 `snapPoints`에 100%가 삽입**되고 있기 때문에 100%로도 스크롤링을 할 수 있는 게 아닐까?

```typescript
// iOS
const snapPoints = useMemo(() => ['50%'], []);

// Android
const snapPoints = useMemo(() => ['50%', '100%'], []); // 🧐 Android에서는 렌더링 시점에서 100% 삽입?
```

`snapPoints` props를 다루고 있는 [BottomSheet.tsx](https://github.com/gorhom/react-native-bottom-sheet/blob/master/src/components/bottomSheet/BottomSheet.tsx) 컴포넌트를 확인해 본 결과 기기 별로 다르게 처리하고 있는 것으로 보이진 않는다.

그렇다면 `snapPoints`의 문제는 아닌 것 같다.

### 2-2. Dynamic Resizing Off

```typescript
const DEFAULT_DYNAMIC_SIZING = true;
```

[BottomSheet Constants](https://github.com/gorhom/react-native-bottom-sheet/blob/master/src/components/bottomSheet/constants.ts#L16)에서 위와 같이 `DEFAULT_DYNAMIC_SIZING` 변수의 Default 값이 `true`로 설정된 것을 확인할 수 있는데 이로 인해 자동으로 높이 조절이 되고 Android에서는 `100%`까지 스크롤링이 되게 된 것 같다.

그렇다면 `snapPoints`를 설정해 주고 자동으로 높이 조절이 불가능하게 하면 되지 않을까? 이 아이디어를 가지고 `snapPoints`를 설정해 주고 `enableDynamicSizing` props를 `false`로 설정하였다.

```tsx
function BottomSheet() {
  // ...

  return (
    <View style={styles.container}>
      <Pressable style={styles.presentButton} onPress={handlePresentModalPress}>
        <Text style={styles.presentText}>Present Modal</Text>
      </Pressable>
      <BottomSheetModal
        ref={bottomSheetModalRef}
        index={1}
        snapPoints={snapPoints}
        enableDynamicSizing={false} // ✅ 동적 높이 조절 기능 Off
      >
        <BottomSheetView style={styles.contentContainer}>
          <Text style={styles.text}>Awesome 🎉</Text>

          <Pressable style={styles.buttonContainer}>
            <View style={styles.button}>
              <Text style={styles.buttonLabel}>버튼</Text>
            </View>
          </Pressable>
        </BottomSheetView>
      </BottomSheetModal>
    </View>
  );
}
```

그 결과 짜잔 🎉

정상적으로 내가 예상하는 바텀 시트가 렌더링이 되는 모습을 확인할 수 있다.

![React Native Bottom Sheet](assets/img/writing/2/bottom_sheet_after.png)
_좌 iOS, 우 Android_

## 참고

- [React Native Bottom Sheet Props](https://gorhom.dev/react-native-bottom-sheet/props#snappoints)