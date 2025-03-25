---
title: React Native In-App Updates (CodePush의 대안을 찾아서)
date: 2025-02-25 19:55:00 +/-TTTT
categories: [Frontend, React Native]
tags: [code push, aws s3, In-app updates, hot update]
image:
  path: assets/img/thumbnail/inapp_update_thumbnail.png
sitemap:
  changefreq: daily
  priority: 0.5
---

> 목차
> 1. 개요
> 2. CodePush의 대안
> 3. React Native HotUpdate가 인앱 업데이트를 하는 방법
> 4. 버전 관리 전략과 자동화 파이프라인 구축
> 5. 결론
{: .prompt-tip }

## 1. 개요

모바일 앱을 개발하다 보면 클라이언트 사이드(Production)에서 발생하는 수 많은 문제에 직면하게 됩니다. TypeError부터 시작해서 ApiError 심지어는 발생해서는 안되는 Critical한 문제로 인해 앱 충돌이 발생하는 정말 다양한 상황이 연출되죠.

이렇게 발생하는 문제는 빠르게 해결하지 않는다면, 서비스의 신뢰도를 떨어트리고 다른 사용자들에게도 전파되어 사용자 이탈로 이어지기 때문에 반드시 빠르게 해결해야만 합니다.

이전까지의 서비스들은 CodePush를 이용하여 실시간 업데이트를 통해 이슈에 대해 빠르게 대응하였지만, CodePush가 다음과 같이 서비스 종료됨에 따라 대안이 필요하게 되었습니다.

![Retirment App Center](assets/img/writing/14/retirment_app_center.png){: width="480" }
_출처: [https://appcenter.ms/](https://appcenter.ms/)_

이 글에서는 CodePush의 종료에 따른 그 대안과 제가 선택한 대안을 프로젝트에 어떻게 적용했는지 소개해드리려고 합니다.

## 2. CodePush의 대안

CodePush 대안에 알아보기 이전에 CodePush의 역할에 대해 구체적으로 짚고 넘어가보겠습니다.

### 2-1. CodePush

모바일 앱에서는 일반적으로 업데이트된 버전을 올리기 위해서는 AppStore/PlayStore에 제출한 후 승인을 기다려야만 합니다. 하지만 이 경우에는 최대 하루가 걸리기 때문에 버그 수정이 발생하는 경우에는 빠른 대응이 어렵게 됩니다.

이러한 문제를 해결하고자 만들어진 것이 CodePush입니다.

![CodePush Approach](assets/img/writing/14/codepush_approach.png){: width="480" }
_출처: [https://medium.com/@katramesh91/what-is-codepush-3f4639d54be1](https://medium.com/@katramesh91/what-is-codepush-3f4639d54be1)_

CodePush는 전통적인 접근 방법과 달리 앱 검토 프로세스를 기다리지 않고 즉시 앱을 업데이트할 수 있도록 하는 솔루션으로 Microsoft에서 제공해주는 **AppCenter를 이용하여 즉시 배포**할 수 있습니다.

AppCenter는 OTA 업데이트 뿐만 아니라 환경에 따른 배포 트랙, 롤백 그리고 모니터링 기능을 지원하는 굉장한 무료 툴이였습니다.

BUT... CodePush를 적용할 수 있었으면 좋았겠지만... CodePush가 앞서 개요에서 나온 이미지처럼 3월 31일 종료하게 되어 새로운 대안을 찾게 되었습니다.

### 2-2. CodePush의 대안

CodePush의 대안으로 무료 툴부터 무료 툴까지 많은 도구가 있었지만, 비용 부담 문제로 인해 무료 툴에 대해서만 알아보았으며 이 중 저희 팀에서 선택한 대안에 대해 소개해드리겠습니다.

#### Case 1. CodePush Standalone

CodePush Sever는 사용자가 자체 호스팅 환경에서 React Native (이하 RN) Application에 대한 OTA 업데이트 배포 및 관리를 할 수 있도록 제공해주는 서버입니다.

> OTA Update
> 
> OTA는 Over the air의 약자로서 새로운 소프트웨어, 펌웨어, 설정 등의 임베디드 시스템을 무선으로 업데이트 하기 위한 방식으로 모바일의 관점에서는 스토어를 통하지 않고 업데이트 하는 것을 의미 (참고: [https://en.wikipedia.org/wiki/Over-the-air_update](https://en.wikipedia.org/wiki/Over-the-air_update))

CodePush를 기반으로 하기 때문에 기존에 신뢰도가 높다는 장점이 있지만, [공식 문서](https://github.com/microsoft/code-push-server/blob/main/api/README.md)에서 언급되어 있듯 CodePush Server를 적용하기 위해서는 HTTPS 설정부터 Azure Blob Storage까지 추가적인 인프라 리소스가 발생하기 때문에 소규모의 팀 보다는 배포 및 관리를 담당하는 팀이 있는 경우에 적합한 대안입니다.

#### Case 2. EAS Update

Expo 팀에서 제공하는 호스팅 서비스로 EAS Update는 `expo-updates` 라이브러리를 사용한 업데이트를 제공합니다.

현재 RN 공식 문서에서도 Expo를 권장하고 있듯이, Expo는 강력한 생태계를 가지고 있으며 추가적인 서버 구축 없이 관리현 서비스를 제공하고 있기 때문에 Expo Framework를 사용하고 있다면 더할 나위 없이 좋은 대안입니다.

#### Case 3. 외부 저장소를 사용한 React Native HotUpdate

해당 방법은 기존 대안들과 달리, 앱 스토어를 통해 직접 업데이트를 진행하는 방식이 아니라, 아래 이미지처럼 외부 저장소에서 버전과 JS 번들을 관리 및 호스팅하는 방식을 채택하고 있습니다.

![스토리지를 사용한 인앱 업데이트 플로우](assets/img/writing/14/in_app_update_using_storage.png){: width="480" }
_외부 저장소를 사용한 인앱 업데이트 플로우_

이 방법은 다른 OTA 업데이트 도구에 비해 코어 기능만을 제공하기 때문에 다른 외부 요소(저장소, 버전 관리 정책 등)를 고려해야 합니다. 하지만 저희 팀은 크게 2가지의 이류를 가지고 대안을 선택하였습니다.

1. 서버 구축 없이 외부 스토리지만 있으면 코어 기능을 사용할 수 있다.
2. bare rn에서 expo 프레임워크로의 전환과 같은 큰 개발 비용을 필요로 하지 않는다.

## 3. React Native HotUpdate가 인앱 업데이트를 하는 방법

RN HotUpdate는 기존의 앱 스토어 업데이트 방식과 달리, 외부 저장소에서 최신 버전의 JS 번들을 관리하고, 앱 내부에서 이를 다운로드 받아 즉시 업데이트를 적용하는 인앱 업데이트 메커니즘을 제공합니다.

![Inapp Update diagram](assets/img/writing/14/inapp_update_diagram.png){: width="480" }
_인앱 업데이트 플로우차트_

RN HotUpdate의 업데이트 과정은 크게 다음과 같습니다.

1. **버전 확인**: 앱 실행 시, 외부 저장소에 등록된 최신 버전 정보를 가져와 현재 앱 버전과 비교
2. **번들 다운로드**: 최신 버전이 존재할 경우, 해당 버전의 JS 번들과 Assets 파일들을 다운로드
3. **번들 설치 및 적용**: 다운로드가 완료되면, RN의 NativeModules 내의 OtaHotUpdate 메서드를 통해 새 번들을 설정하고, 앱을 재시작하여 업데이트를 적용

RN HotUpdate를 사용하여 빠르게 인앱 업데이트의 초기 구조를 잡을 수 있었습니다. 하지만 간편해진만큼 더 많은 것들을 고려해야 했습니다.

1. Native가 변경되는 경우에 대한 대응
2. 버전 관리 전략

이러한 문제점들을 보안하기 위해 저희 팀은 두 가지의 주요 개선 방안을 도입했습니다.

## 4. 버전 관리 전략과 자동화 파이프라인 구축

인앱 업데이트를 안정적으로 운영하기 위해, 먼저 버전 관리 전략을 명확히 정의할 필요가 있었습니다. 특히, 네이티브 코드 업데이트와 자바스크립트 번들 업데이트를 구분하여 관리해야 했습니다.

### 4-1. 버전 관리 전략

만약 네이티브 코드가 변경된 경우, 기존 JS 번들과의 호환성이 깨질 위험이 있기 때문에, 이를 구분하는 명확한 버전 관리 체계가 필요했습니다. 그래서 다음과 같은 버전 관리 규칙을 도입했습니다.

1. 네이티브 버전과 JS 번들 버전을 분리하여 관리
  - `bundleVersion`: JS 번들이 변경되는 경우 증가 (ex. 100)
  - `nativeVersion`: Android/iOS 내부 모듈이 변경되는 경우 현재 작업중인 버전으로 업데이트 (ex. 1.1.2)
2. 서버에서 제공하는 버저닝 정보 파일을 통해 업데이트 가능 여부를 판단
  - 앱이 실행될 때, 외부 저장소에서 읽어 온 JSON의 `nativeVersion`과 `bundleVersion`을 순차적으로 조회하여 최신 버전인지 확인

- 그 외 updatedAt과 appVersion은 업데이트 일시 및 업데이트 당시 버전을 기록하기 위해 추가

```json
{
  "updatedAt": "2025-02-24T17:09:26Z",
  "appVersion": "1.6.0",
  "bundleVersion": 59,
  "nativeVersion": "1.6.0",
  "downloadAndroidUrl": "https://storage.com/production/index.android.bundle.zip",
  "downloadIosUrl": "https://storage.com/production/main.jsbundle.zip",
}
```

이렇게 적용된 전략은 다음과 같은 프로세스로 앱이 진행됩니다.

![Inapp Update process](assets/img/writing/14/inapp_update_process.png){: width="640" }
_인앱 업데이트 프로세스_

### 4-2. 버전 관리 자동화

이러한 버전은 외부 저장소를 통해 앱이 업데이트될 때마다 변경되는데, 버전 정보를 수동으로 업데이트하는 과정은 휴먼 에러를 유발할 수 있게 되고 이는 큰 리스크를 가지게 됩니다.

리스크를 사용자에게 부담할 수는 없기 때문에 해당 과정을 자동으로 처리하도록 다음과 같은 워크플로우를 구성하였습니다.

![App Workflow](assets/img/writing/14/app_workflow.png){: width="640" }
_버전 관리를 위한 자동화 파이프라인 워크플로우: [https://gist.github.com/BangDori/d8ce7347e3b1e5d25ffade8ff51f4b62](https://gist.github.com/BangDori/d8ce7347e3b1e5d25ffade8ff51f4b62)_

워크플로우는 prod/staging 환경에 따른 버전 관리 워크플로우를 나타내고 있으며, 기본적인 흐름은 다음과 같습니다.

1. 테스트 단계
2. (`on: pull_request: release-**`) 코드 변경 감지
  - 네이티브 코드 변경 시: Staging 환경에 Android/iOS 배포
  - JS 코드 변경 시: 외부 저장소에 Staging 환경의 버전 정보 및 번들 업데이트
3. (`on: pull_request: main`) 프로덕션 배포
  - 최신 버전 유지를 위해 Android/iOS를 앱 스토어에 배포
4. (`on: push: main`) 프로덕션 버전 정보 및 번들 업데이트

이러한 자동화를 통해 수동 업데이트 없이, 자동으로 안전한 버전 관리 체계를 기반으로 업데이트할 수 있었습니다.

자동화를 추가하는 것으로 끝나는 것이 아닌, 한 가지 고려해야 할 점이 있었습니다. 바로 스토어 제출과 외부 저장소의 버전 업데이트 사이에 앱을 다운로드 받는 사용자가 최신 버전을 보장할 수 있도록 하는 것이였습니다.

![App Timeline Problem](assets/img/writing/14/timeline_problem.png){: width="640" }
_타임라인에 따른 앱 설치_

위 타임라인을 살펴보면 기존 사용자와, 신규 사용자 2의 경우에는 최신 버전을 보장할 수 있지만, 문제는 1번과 2번 시간 사이에 앱을 다운로드 받는 경우에 발생합니다.

신규 사용자 1은 `bundleVersion`이 32인 앱을 다운받지만, 앱 최초 시작시 번들 버전이 33으로 초기화되기 때문에 최신 버전의 앱이 추후 배포됨에도 불구하고 최신 버전을 다운받지 못하게 됩니다. (앱 최초 시작 시 외부 저장소의 `bundleVersion`으로 초기화)

이렇게 최신 버전을 받지 못하는 사용자를 줄이기 위해, 저희 팀은 앱을 업데이트가 완료된 후 main 브랜치로의 PR을 push(프로덕션 버전 정보 및 번들 업데이트 워크플로우 실행) 하도록 임시 방편을 마련하였습니다.

## 5. 결론

외부 저장소를 활용한 React Native HotUpdate 방식은 빠른 업데이트 적용, 서버 구축 비용 절감, 스토어 검수를 기다릴 필요 없는 배포 방식 등의 장점이 있었지만, 버전 관리의 복잡성과 스토어 배포 시점과의 동기화 문제와 같은 새로운 과제가 발생했습니다.

이를 해결하기 위해,

- 네이티브 코드와 JS 번들 버전을 분리하여 관리하는 **버전 관리 전략을 수립**하고,
- **자동화된 CI/CD 파이프라인을 구축하여 휴먼 에러를 줄이고 배포 일관성을 유지**하는 방식을 적용했습니다.
 
그러나 아직 해결해야 할 문제점들이 남아 있습니다.

1. 신규 사용자의 번들 초기화 문제 (현재 임시방편으로 처리)
  - 신규 사용자가 앱을 설치할 때, 스토어 배포 시 포함된 번들 버전과 외부 저장소에서 관리하는 최신 번들 버전 간의 불일치 발생
2. 버전별 Hotfix 처리의 어려움
  - 특정 버전에서 치명적인 버그가 발생했을 경우, 해당 버전 사용자만을 위한 Hotfix 업데이트를 적용하는 것이 어려움
3. 점진적 업데이트의 부재
  - 현재 방식에서는 모든 사용자가 항상 최신 버전을 강제 적용받는 구조
  - 하지만 최신 버전에 예기치 못한 치명적인 에러가 발생할 경우, 전체 사용자에게 영향을 미치는 심각한 문제가 발생할 수 있음

긴 글을 읽어주셔서 감사합니다. (잘못된 내용이 있을 수 있습니다..!)

> Next!
> 1. 신규 사용자가 항상 최신 번들을 받을 수 있도록 보장하기
> 2. 특정 버전 사용자에게만 핫픽스를 제공하는 방법
> 3. 점진적 업데이트(Gradual Rollout) 기법을 통해 앱 이슈 최소화하기
{: .prompt-info }

> 이 글의 다음 내용이 궁금하시다면, [체크박스 하나로 인앱 업데이트 제어하기 (with 이전 문제를 해결하며)](https://bangdori.kr/posts/inapp-update-2)를 확인해보세요!

## 참고

- [Over-The-Air updates with CodePush - The Warm Up (React Native Heroes 2023)](https://www.youtube.com/watch?v=XXe_o3wu-n4)
- [앱 버전 관리, 어떻게 하는게 좋을까?](https://honeystorage.tistory.com/310)
- [react-native-ota-hot-update](https://github.com/vantuan88291/react-native-ota-hot-update)