---
title: React Native에서 고유 식별자를 안전하게 서버로 전송하기
description: React Native 앱에서 고유 식별자를 SHA-256으로 암호화하여 안전하게 전송하는 방법을 알아보자!
date: 2024-11-12 22:46:00 +/-TTTT
categories: [Frontend, React Native]
tags: [api, react native, security, SHA-256, unique identifier]
image:
  path: assets/img/writing/10/client_server_userId.png
sitemap:
  changefreq: daily
  priority: 0.5
---

> **세 줄 요약**
> 1. 서버에서 API 요청 제한을 위해 고유 식별자가 필요하며, 클라이언트에서는 안전하게 고유 식별자를 전달해 주어야 한다.
> 2. 기기의 `uniqueId`를 활용해 사용자 요청을 추적할 수 있지만, 이를 암호화하지 않으면 보안 위험이 발생할 수 있다.
> 3. SHA-256 해싱 알고리즘을 사용해 `uniqueId`를 안전하게 전송함으로써 보안성을 강화하고, 서버에서 효과적으로 요청을 제한할 수 있다.
{: .prompt-tip }

## 1. 개요

애플리케이션이 서버와 통신할 때, 지나치게 많은 API 요청이 발생하면 서버 응답 지연, 성능 저하, 심각한 경우 서버 다운과 같은 문제를 유발하게 됩니다. 이를 방지하기 위해서는, 서버에서는 각 사용자의 고유 식별자를 통해 요청을 추적하고 제한하는 방안을 마련해야 하며, 클라이언트에서는 서버가 사용자를 식별할 수 있도록 고유 식별자를 안전하게 전달해야 합니다.

이 글에서는 클라이언트(React Native)에서 고유 식별자를 안전하게 다루기 위해 제가 접근한 방법과 실제 채택한 방법을 알아보겠습니다.

## 2. 접근 방법

### 2-1. userId를 활용한 API 요청 제한

![client_server_userId](assets/img/writing/10/client_server_userId.png)

처음 생각한 방법은 위 이미지와 같이 데이터베이스 기록된 `userId`를 활용하는 방법이었습니다. 모든 사용자는 고유 아이디가 데이터베이스에 등록되어 있으며, API 요청 시 헤더에 포함되는 Authorization을 디코딩하여 user 정보를 획득할 수 있으니 나쁘지 않은 방법처럼 보였습니다. 또한 서버에서만 작업이 이루어지니 클라이언트에서 사용자의 고유 아이디를 무리해서 획득할 필요도 없습니다.

하지만, 이 방법의 경우에는 치명적인 문제점이 있었습니다. 로그인, 회원가입 등과 같이 Authorization 정보 없이 API를 요청할 수 있는 경우에는 사용자를 특정할 수 없다는 점입니다.

![client_server_userId_anonymous](assets/img/writing/10/client_server_userId_anonymous.png)

실제로 위 이미지처럼 Authorization이 없는 경우에는 서버에서는 API 요청에 대한 사용자를 특정할 수 없게 되고, 익명의 사용자에게 모든 API 요청을 반환해 주게 됩니다. 그렇게 되면 결국 수백, 수천만 건의 요청에 대한 응답을 반환해 주게 되고 서버는 결국 이를 견디지 못해 터지게 됩니다. 그렇다면 다른 방법에는 어떤 것이 있을까요?

### 2-2. uuid를 활용한 API 요청 제한

두 번째로 생각한 방법은 `uuid`를 생성하여 서버로 전송하고, 서버에서는 해당 `uuid`를 기반으로 API 요청을 추적하는 방법이었습니다.

> **UUID (Universally Unique Identifier)**
>
> Universally Unique Identifier는 컴퓨터 시스템에서 객체를 고유하게 식별하는 데 사용되는 128비트 레이블입니다.
> 
> [https://en.wikipedia.org/wiki/Universally_unique_identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier)
{: .prompt-info }

`uuid`는 다음과 같이 앱 내에서 쉽게 생성할 수 있고, 충돌 가능성이 매우 낮은 고유 식별자이기 때문에 사용자나 기기를 식별하는 데 적합하다고 생각했습니다.

```js
import axios from 'axios';
import { v4 as uuidv4 } from "uuid";

const axiosInstance = axios.create({ ... });

axiosInstance.interceptors.request.use((config) => {
  const deviceId = getHeader(headerKeys.DEVICE_ID);
  if (!deviceId) {
    const uuid = uuidv4();
    setHeader(headerKeys.DEVICE_ID, uuid);
  }

  return config;
});
```

그러나 이 방법에도 몇 가지 문제점이 있었습니다.

1. `uuid`는 앱에서 자체적으로 생성되기 때문에 사용자가 앱을 재실행, 재설치 혹은 캐시를 초기화하는 경우 새로운 UUID가 생성되어 다른 사용자로 인식됨
2. `uuid`는 128비트의 고유 식별자이지만 정말 낮은 확률로 중복이 발생할 수 있음

2번의 경우에는 정말 낮은 확률이기에 무시가 가능한 수치였지만 악의적으로 앱에 수백 개의 요청을 보낸 후 껐다가 다시 요청을 보내는 경우를 대비할 수 없었습니다. 그래서 새로운 방법을 모색하게 되었습니다.

### 2-3. uniqueId를 활용한 API 요청 제한

[react-native-device-info](https://github.com/react-native-device-info/react-native-device-info) 라이브러리는 기기의 고유 식별자를 제공하는 다양한 메서드를 지원합니다.

이 중 `getUniqueId` 메서드는 iOS에서는 [IDFV](https://developer.apple.com/documentation/uikit/uidevice/1620059-identifierforvendor)값을 Android에서는 [ANDROID_ID](https://developer.android.com/reference/android/provider/Settings.Secure.html#ANDROID_ID)값을 반환하는데, 이 값들은 기기에 등록된 고유 정보로 **앱의 재설치 유무와 관계없이 항상 동일한 값을 제공**합니다.

```js
import axios from 'axios';
import { getUniqueId } from 'react-native-device-info';

const axiosInstance = axios.create({ ... });

axiosInstance.interceptors.request.use((config) => {
  const deviceId = getHeader(headerKeys.DEVICE_ID);
  if (!deviceId) {
    const uniqueId = await getUniqueId();
    setHeader(headerKeys.DEVICE_ID, uniqueId);
  }

  return config;
});
```

![client_server_uniqueId](assets/img/writing/10/client_server_uniqueId.png)

이렇게 하면 서버에서는 모든 API 요청에 사용자를 식별할 수 있는 `uniqueId`가 포함되고 기기별로 요청을 추적 및 제한할 수 있습니다.

## 3. uniqueId를 그대로 전송하면 발생하는 문제점

앞서 설명한 대로 `uniqueId`를 API 요청 헤더에 추가하면 서버에서 사용자별로 요청을 추적할 수 있습니다. 그러나 `uniqueId`를 그대로 전송하는 것은 사용자 기기 정보를 포장지 없이 원본 그 자체를 서버로 전송하는 것이기 때문에 아래 그림처럼 해커에게 탈취당할 경우 보안 및 개인정보 보호 측면에서 문제가 될 수 있습니다.

![client_server_uniqueId_hacking](assets/img/writing/10/client_server_uniqueId_hacking.png)

물론 사용자의 개인 정보나 민감한 데이터를 해커가 직접적으로 수집할 수는 없겠지만, 탈취한 `uniqueId`를 통해 사용자의 행동 패턴을 분석하거나 사용자의 다른 정보도 탈취하게 된다면 개인 정보가 노출될 수 있습니다. 그렇기에 `uniqueId`를 사용할 때는 반드시 암호화 알고리즘을 적용하여 포장된 상태로 서버로 전송해 주어야 합니다.

※ `uniqueId`를 그대로 사용하는 것은 앱 심사 리젝 사유가 될 수 있으니 반드시 유의해야 합니다.

### 3-1. SHA-256으로 안전하게 전달하기

서버에서는 클라이언트로부터 전달받은 `uniqueId`를 복호화할 필요 없이 고유 식별자로 사용하면 되기 때문에, 단방향 해싱 알고리즘인 SHA-256을 적용하였습니다.

> **SHA-256**
>
> SHA-256은 SHA(Secure Hash Algorithm) 단방향 알고리즘의 한 종류로, 해시값을 이용한 암호화 방식 중 하나이다. 256비트로 구성되며 64자리 문자열을 반환한다.
{: .prompt-info }

React Native에서 [react-native-sha256](https://github.com/itinance/react-native-sha256)를 적용하여 `uniqueId`를 해싱한 방법은 다음과 같습니다:

```js
import axios from 'axios';
import { getUniqueId } from 'react-native-device-info';
import { sha256 } from 'react-native-sha256';
import Config from 'react-native-config';

const axiosInstance = axios.create({ ... });

axiosInstance.interceptors.request.use((config) => {
  const deviceId = getHeader(headerKeys.DEVICE_ID);
  if (!deviceId) {
    const uniqueId = await getUniqueId().then((id) => sha256(id + Config.UNIQUE_ID_SALT)); // 추가적인 보안 강화를 위해 SALT 추가
    setHeader(headerKeys.DEVICE_ID, uniqueId);
  }

  return config;
});
```

위와 같이 적용함으로써 사용자의 `uniqueId`는 안전하게 전송하고 서버에서는 효율적으로 API 요청 제한을 구현할 수 있게 되었습니다.

## 4. 마치며

이번 포스팅은 아주 깊이 있는 주제는 아니며, 다른 개발자들에게 큰 도움이 될 내용도 아닐 수 있습니다. 그렇지만, 직면한 상황에 맞춰 해결 방법을 찾아가는 과정을 스스로 점검해 보고자 문제를 해결하는 과정을 기록하게 되었습니다.

글을 마무리하면서, 단순히 문제 해결 과정을 넘어 데이터 보안과 개인정보 보호의 중요성을 다시 한번 깨달을 수 있었습니다. 특히 네트워크를 통한 데이터 전송 시 보안에 취약한 부분이 없는지 항상 확인하고, 필요한 경우 암호화나 해싱 같은 방법을 적용해 안전성을 높이는 것이 필수적임을 느꼈습니다.

## 5. 참고

- [Payments 개발자센터 UUID(Universally Unique Identifier)](https://docs.tosspayments.com/resources/glossary/uuid)
- [How to generate unique id in react native](https://stackoverflow.com/questions/67682503/how-to-generate-unique-id-in-react-native)
- [identifierForVendor를 이용한 기기 식별하기](https://green1229.tistory.com/364)
- [[단방향 암호화]해시 함수 / Salt](https://cjw-awdsd.tistory.com/18)
- [[Rate Limit - step 1] Rate Limit이란? (소개, Throttling, 구현시 주의사항 등)](https://etloveguitar.tistory.com/126)