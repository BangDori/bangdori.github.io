---
title: React Native 다른 지도 앱으로 열기
description: React Native Linking을 이용하여 외부 지도를 열어보자
date: 2024-10-23 00:35:00 +/-TTTT
categories: [Frontend, React Native]
tags: [react native, linking, query, naver map]
image:
  path: assets/img/thumbnail/react_native_linking.png
sitemap:
  changefreq: daily
  priority: 0.5
---

## 1. 개요

앱 내 지도에 표시된 마커까지의 거리나 내비게이션 기능에 대한 구현 방법을 고민하던 중 이를 직접 구현하게 되면 많은 개발 비용이 들어가게 될 것이라 생각했다. 그래서 이러한 기능을 직접 구현하지 않고 Linking을 이용해서 외부 지도 앱으로 연동시켜 주어 해결하였는데 오늘 이 방법에 대해 공유하고자 한다.

## 2. Linking

외부 지도 앱으로 연동하기 위해서는 React Native에서 제공해 주는 [Linking](https://reactnative.dev/docs/linking) 기능을 적용해야 한다.

> **Linking**
> 
> Linking gives you a general interface to interact with both incoming and outgoing app links.
{: .prompt-tip }

Linking을 이용하면 다른 앱들과 상호작용을 할 수 있다. 다른 앱들과 상호작용 하기 위해서는 기존의 웹에서 사용하던 `https://` 혹은 `http://`로 시작하는 링크가 아닌 각각의 앱이 가지고 있는 URL Scheme을 이용해야 한다.

우선 Linking을 통해 외부 앱을 열기 이전에 연결하려는 지도의 scheme 들을 iOS(info.plist)와 안드로이드(AndroidManifest.xml)에 각각 추가해 주자.

난 네이버 지도, 카카오맵, 티맵을 사용할 예정이기 때문에 아래와 같이 3개만 등록하였다.

```html
<dict>
  <key>LSApplicationQueriesSchemes</key>
  <array>
    <string>tmap</string>
    <string>nmap</string>
    <string>kakaomap</string>
  </array>
</dict>
```

```xml
// AndroidManifest.xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.example.myapp">
    <application>
      // ...
    </application>

    <queries>
      <package android:name="com.skt.tmap.ku" /> // 추가 안하면 티맵 작동 X
      <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="tmap" />
      </intent>
      <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="nmap" />
      </intent>
      <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="kakaomap" />
      </intent>
    </queries>
</manifest>
```

여기서 주의할 점이 있는데 tmap은 위 주석이 작성된 package를 추가해 주지 않으면 정상적으로 Linking이 동작하지 않으니 조심하자. 이제 설정이 끝났으니, 이제 본격적으로 네이버 지도를 실행하기 위한 URL Scheme부터 하나씩 알아보자.

### 2-1. 네이버 지도 URL Scheme

네이버 지도의 App Scheme은 [지도 앱 연동 URL Scheme](https://guide.ncloud-docs.com/docs/maps-url-scheme)을 참고하면 쉽게 얻을 수 있다. 나는 현재 필요한 액션이 자동차로 길 찾기이기 때문에 `route/car` 액션을 사용하겠다. 근데 해당 예제를 보면 수많은 쿼리를 확인할 수 있는데, 각 쿼리 파라미터가 의미하는 바는 다음과 같다.

- `slat`: 출발지 위도 
- `slng`: 출발지 경도
- `sname`: 출발지 이름
- `dlat`: 도착지 위도
- `dlng`: 도착지 경도
- `dname`: 도착지 이름
- `appname`: 패키지명

해당 앱 스킴으로 연결할 때 네이버 지도 앱이 설치되어 있고, 네이버 지도에서 위치 권한이 허용되어 있는 상태라면 slat, slng, sname이 자동으로 지정되기 때문에 도착지의 정보만 제공해 주면 된다. 근데 만약 출발지의 위치까지 지정하기를 원한다면 모든 쿼리를 사용하면 된다.

```ts
// 네이버 지도 길찾기 앱 스킴
const NAVER_MAP_APP_SCHEME = ({ latitude, longitude, destination }) => `nmap://route/car?dlat=${latitude}&dlng=${longitude}&dname=${destination}&appname=com.example.myapp`;
```

그리고 사용자가 네이버 지도가 추가되어 있지 않은 경우도 있기 때문에 iOS와 안드로이드 플랫폼별로 이에 대해서도 대응해 줘야 한다. 추가되어 있지 않은 경우에는 각 플랫폼 스토어로 연결해 주면 된다.

```ts
// 네이버 지도 길찾기 앱 스킴
const NAVER_MAP_APP_SCHEME = ({ latitude, longitude, destination }) => `nmap://route/car?dlat=${latitude}&dlng=${longitude}&dname=${destination}&appname=com.example.myapp`;

const NAVER_MAP_ANDROID_STORE = 'market://details?id=com.nhn.android.nmap'; // 네이버 지도 안드로이드 구글 플레이 링크
const NAVER_MAP_IOS_STORE = 'https://itunes.apple.com/app/id311867728?mt=8'; // 네이버 지도 iOS 앱 스토어 링크
```

이렇게 하면 네이버는 끝났다. 이제 다음으로, 카카오맵으로 이동해보자.

### 2-2. 카카오맵 URL Scheme

카카오맵도 동일하게 공식 문서가 존재하는데 [카카오맵 URL Scheme](https://apis.map.kakao.com/ios_v2/docs/getting-started/urlscheme/)을 참고하면 쉽게 적용할 수 있다. 네이버와 카카오는 공식 문서가 잘 정리되어 있어서 편하다.

네이버 지도와 마찬가지로 앱 스킴과 플랫폼별 스토어를 추가해 주자.

```ts
// 카카오맵 앱 스킴
const KAKAO_MAP_APP_SCHEME = ({ latitude, longitude }) => `kakaomap://route?ep=${latitude},${longitude}&by=CAR`;

const KAKAO_MAP_ANDROID_STORE = 'market://details?id=net.daum.android.map'; // 카카오맵 안드로이드 구글 플레이 링크
const KAKAO_MAP_IOS_STORE = 'https://itunes.apple.com/app/id304608425?mt=8'; // 카카오맵 IOS 앱 스토어 링크
```

### 2-3. TMAP URL Scheme

티맵은 공식 문서가 따로 존재하지 않아서 구글링을 통해 다른 사람들이 한 링크들을 참고하였다.

```ts
// T맵 앱 스킴
const TMAP_MAP_APP_SCHEME = ({ latitude, longitude, destination }) => `tmap://route?rGoName=${destination}&rGoX=${longitude}&rGoY=${latitude}`;

const TMAP_MAP_ANDROID_STORE = 'market://details?id=com.skt.tmap.ku'; // T맵 안드로이드 구글 플레이 링크
const TMAP_MAP_IOS_STORE = 'https://itunes.apple.com/app/id431589174?mt=8'; // T맵 IOS 앱 스토어 링크
```

실제 확인을 해보면 iOS에서는 위 앱 스킴이 잘 동작하는데 안드로이드에서는 정상 동작하지 않는다. 그래서 [inflearn에 올라온 질문](https://www.inflearn.com/community/questions/509596/%ED%8B%B0%EB%A7%B5%EC%9D%84-linking-openurl-%EB%A1%9C-%EC%97%AC%EB%8A%94-%EA%B2%83%EC%9D%80-%EC%96%B4%EB%96%A4%EA%B0%80%EC%9A%94?srsltid=AfmBOoqTT2XqQo5mZFON-vmlvKlM7zf4fwGZrsfEueKcE5rkXnsOviku)을 참고하여 앱 스킴을 다음과 같이 수정해주었다.

```ts
// 변경된 T맵 앱 스킴
const TMAP_MAP_APP_SCHEME = ({ latitude, longitude, destination }) => `tmap://route?goalname=${destination}&goalx=${longitude}&goaly=${latitude}`;

const TMAP_MAP_ANDROID_STORE = 'market://details?id=com.skt.tmap.ku'; // T맵 안드로이드 구글 플레이 링크
const TMAP_MAP_IOS_STORE = 'https://itunes.apple.com/app/id431589174?mt=8'; // T맵 IOS 앱 스토어 링크
```

근데 만약 iOS Platform을 타겟으로 한다면 굳이 안드로이드에서도 동작하도록 앱 링크를 수정해 줄 필요는 없을 것 같다.

## 3. Linking 연동하기

Linking을 이용해서 외부 앱을 연결하는 방법은 정말 쉽다. 2가지의 기능만 숙지하고 들어가자.

1. `canOpenURL`: 앱이 매개변수로 주어진 URL을 처리할 수 있는지를 확인하는 메서드
2. `openURL`: 매개변수로 주어진 URL을 여는 메서드

우리는 이제 Linking의 중요 기능에 대해 배웠으니 이제 구현해 보자. 코드는 아래와 같다.

```ts
type MapProvider = 'tmap' | 'nmap' | 'kakaomap';
    
const handlePressOpenMap = async (provider: MapProvider, coordinate: LatLng, destination: string) => {
  // 1. App Scheme Link 가져오기
  const appLink = links[provider].app({ ...coordinate, destination });

  // 2. 내 휴대폰에 App이 설치되어 있는지 확인하기
  const supported = await Linking.canOpenURL(appLink);

  // 3. 만약 설치되어 있다면,
  if (supported) {
    // 설치된 앱 내부로 이동
    await Linking.openURL(appLink);
    return;
  }

  // 4. 설치가 되어있지 않다면, 현재 앱을 실행하고 있는 플랫폼을 가져오기
  const platform = Platform.OS === 'ios' ? 'ios' : 'android';

  // 5. App 설치를 위한 플랫폼의 스토어로 이동하기 
  await Linking.openURL(links[provider].store[platform]);
};
```

이제 위 메서드를 원하는 대로 커스텀해서 내 프로젝트에 적용하면 외부 앱으로 연동이 된다.

아참! 안드로이드 에뮬레이터에서는 구글 플레이가 지원돼서 정상적으로 동작함을 확인할 수 있는데, iOS에서는 앱 스토어가 실행이 안 되기 때문에 실제 Device를 맥북과 연결해서 테스트해 봐야 한다.

## 참고

- [React Native - Linking](https://reactnative.dev/docs/linking)
- [[React Native] 다른 앱(play store, instagram 등) 열기](https://surprisecomputer.tistory.com/125)
- [티맵을 Linking.openURL()로 여는 것은 어떤가요?](https://www.inflearn.com/community/questions/509596/%ED%8B%B0%EB%A7%B5%EC%9D%84-linking-openurl-%EB%A1%9C-%EC%97%AC%EB%8A%94-%EA%B2%83%EC%9D%80-%EC%96%B4%EB%96%A4%EA%B0%80%EC%9A%94?srsltid=AfmBOoqTT2XqQo5mZFON-vmlvKlM7zf4fwGZrsfEueKcE5rkXnsOviku)
- [SwiftUI kakao map, Naver map, Tmap, apple map 길찾기 구현하기](https://develop-const.tistory.com/33)