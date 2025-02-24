---
title: React Native 인앱 업데이트 적용기
date: 2025-02-22 23:22:00 +/-TTTT
categories: [Frontend, React Native]
tags: [code push, aws s3, In-app updates, hot update]
image:
  path: assets/img/thumbnail/
sitemap:
  changefreq: daily
  priority: 0.5
---

> 목차
> 1. 개요
> 2. CodePush의 대안
> 3. React Native HotUpdate가 인앱 업데이트를 하는 방법
> 4. 버전 관리 전략과 자동화 파이프라인 구축
{: .prompt-info }

## 1. 개요

모바일 앱을 개발하다 보면 클라이언트 사이드(Production)에서 발생하는 수 많은 문제에 직면하게 됩니다. 잘못된 로직으로 인해 발생하는 이슈부터 특정 기기에서만 발생하는 이슈까지 정말 다양한 상황이 연출되죠.

이렇게 발생하는 문제는 빠르게 해결하지 않는다면, 서비스의 신뢰도를 떨어트리고 다른 사용자들에게도 전파되어 사용자 이탈로 이어지기 때문에 반드시! 빠르게 해결해야만 합니다.

이전까지는 CodePush를 이용하여 실시간 업데이트를 통해 이슈에 대해 빠르게 대응하였지만, CodePush가 다음과 같이 서비스 종료됨에 따라 대안이 필요하게 되었습니다.

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

CodePush는 전통적인 접근 방법과 달리 앱 검토 프로세스를 기다리지 않고 즉시 앱을 업데이트할 수 있도록 하는 솔루션으로 Microsoft에서 제공해주는 AppCenter를 이용하여 즉시 배포할 수 있습니다.

물론 네이티브의 변경이 아닌 자바스크립트의 변경에 한해서만 가능합니다. 

CodePush를 적용할 수 있었으면 좋았겠지만... CodePush가 앞서 개요에서 나온 이미지처럼 3월 31일 종료하게 됨에 대안을 찾게 되었습니다.

### 2-2. CodePush의 대안

CodePush의 대안으로 무료 튤부터 무료 툴까지 많은 도구가 있었지만, 비용 부담 문제로 인해 무료 툴에 대해서만 알아보았으며 이 중 저희 팀에서 선택한 대안에 대해 소개해드리겠습니다.

#### Case 1. CodePush Standalone

CodePush Sever는 사용자가 자체 호스팅 환경에서 React Native (이하 RN) Application에 대한 OTA 업데이트 배포 및 관리를 할 수 있도록 제공해주는 서버입니다.

> OTA Update
> 
> OTA는 Over the air의 약자로서 새로운 소프트웨어, 펌웨어, 설정 등의 임베디드 시스템을 무선으로 업데이트 하기 위한 방식으로 모바일의 관점에서는 스토어를 통하지 않고 업데이트 하는 것을 의미 (참고: [https://en.wikipedia.org/wiki/Over-the-air_update](https://en.wikipedia.org/wiki/Over-the-air_update))

CodePush를 기반으로 하기 때문에 기존에 신뢰도가 높다는 장점이 있지만, [공식 문서](https://github.com/microsoft/code-push-server/blob/main/api/README.md)에서 언급되어 있듯 CodePush Server를 적용하기 위해서는 HTTPS 설정부터 Azure Blob Storage까지 추가적인 인프라 리소스가 발생하기 때문에 소규모의 팀 보다는 배포 및 관리를 담당하는 팀이 있는 경우에 적합하다고 생각되어 저희 팀에서는 보류하였습니다.

#### Case 2. EAS Update

Expo 팀에서 호스팅 서비스로 EAS Update는 `expo-updates` 라이브러리를 사용하여 프로젝트에 대한 업데이트를 제공합니다.

현재 RN 공식 문서에서도 Expo를 권장하고 있듯이, Expo는 강력한 생태계를 가지고 있습니다. 또한 추가적인 서버 구축 없이 관리현 서비스를 제공하고 있기 때문에 Expo Framework를 사용하고 있다면 더할 나위 없이 좋은 대안이라고 생각합니다.

하지만, 저희 팀은 Bare RN로 프로젝트를 구성하였기 때문에 Expo로의 전환 과정에서 발생하는 개발 비용에 대한 부담을 느껴 보류하였습니다.

#### Case 3. React Native HotUpdate

[Levi](https://github.com/vantuan88291)가 만든 라이브러리로 NativeModules의 OtaHotUpdate를 기반으로 동작하는 라이브러리입니다.

이 라이브러리는 기존 대안들과 달리, 앱 스토어를 통해 직접 업데이트를 진행하는 방식이 아니라, 아래 이미지처럼 외부 저장소에서 버전과 JS 번들을 관리 및 호스팅하는 방식을 채택하고 있습니다.

![스토리지를 사용한 인앱 업데이트 플로우](assets/img/writing/14/in_app_update_using_storage.png){: width="480" }
_외부 저장소를 사용한 인앱 업데이트 플로우_

해당 라이브러리는 RN에서 제공하는 NativeModules 기반의 업데이트 방식으로 높은 신뢰성을 제공하며, 추가적인 인프라 리소스가 발생하지 않는 장점이 있습니다. 특히, 앱 프론트엔드 개발을 혼자서 진행하는 상황에서는 간편한 설정과 손쉬운 통합 덕분에 초기 환경 구성에 소요되는 시간과 노력을 크게 절감할 수 있어, 매우 매력적인 선택지라 느껴 저희 팀에서는 해당 라이브러리를 채택하게 되었습니다.

## 3. React Native HotUpdate가 인앱 업데이트를 하는 방법

RN HotUpdate는 기존의 앱 스토어 업데이트 방식과 달리, 외부 저장소에서 최신 버전의 JS 번들을 관리하고, 앱 내부에서 이를 다운로드 받아 즉시 업데이트를 적용하는 인앱 업데이트 메커니즘을 제공합니다.

![Inapp Update diagram](assets/img/writing/14/inapp_update_diagram.png){: width="480" }
_인앱 업데이트 플로우차트_

RN HotUpdate의 업데이트 과정은 다음과 같습니다.

1. **버전 확인**: 앱 실행 시, 외부 저장소에 등록된 최신 버전 정보를 가져와 현재 앱 버전과 비교
2. **번들 다운로드**: 최신 버전이 존재할 경우, 해당 버전의 JS 번들과 Assets 파일들을 다운로드
3. **번들 설치 및 적용**: 다운로드가 완료되면, RN의 NativeModules 내의 OtaHotUpdate 메서드를 통해 새 번들을 설정하고, 앱을 재시작하여 업데이트를 적용

RN HotUpdate를 사용하면서 간편한 설정 및 인앱 업데이트의 초기 구조를 잡을 수 있었습니다. 하지만 간편해진 만큼 더 많은 것들을 고려해야 했습니다.

1. Native가 변경되는 경우
2. 버전 관리의 복잡성
3. 스토어의 앱을 업데이트하는 방식이 아니기에 항상 최신 버전을 보장하지 못함

이러한 문제점들을 보안하기 위해 저희 팀은 두 가지의 주요 개선 방안을 도입했습니다.

### 4. 버전 관리 전략과 자동화 파이프라인 구축

인앱 업데이트를 안정적으로 운영하기 위해, 먼저 버전 관리 전략을 명확히 정의할 필요가 있었습니다. 특히, 네이티브 코드 업데이트와 자바스크립트 번들 업데이트를 구분하여 관리해야 했습니다.

### 4-1. 버전 관리 전략

RN에서는 네이티브 코드(iOS, Android)와 자바스크립트 번들(JS Bundle)의 버전을 다르게 관리합니다. 만약 네이티브 코드가 변경된 경우, 기존 JS 번들과의 호환성이 깨질 위험이 있기 때문에, 이를 구분하는 명확한 버전 관리 체계가 필요했습니다.

이를 해결하기 위해, 저희 팀에서는 다음과 같은 버전 관리 규칙을 도입했습니다.

1. 네이티브 버전과 JS 번들 버전을 분리하여 관리
  - `bundleVersion`은 JS 번들이 변경되는 경우 증가 (ex. 100)
  - `nativeVersion`은 네이티브 android/ios 내부 모듈이 변경되는 경우 현재 작업중인 버전으로 설정 (ex. 1.1.2)
2. 서버에서 제공하는 버저닝 정보 파일을 통해 업데이트 가능 여부를 판단
  - 앱이 실행될 때, 외부 저장소에서 읽어 온 JSON의 `nativeVersion`과 `bundleVersion`을 조회하여 최신 버전인지 확인

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

이러한 버전은 외부 저장소를 통해 관리되는데, 외부 저장소에 버전 정보를 수동으로 업데이트하는 과정은 휴먼 에러를 유발할 수 있어 다음과 같이 워크플로우를 구성하였습니다.

![App Workflow](assets/img/writing/14/app_workflow.png){: width="640" }
_버전 관리를 위한 자동화 파이프라인 워크플로우_

워크플로우는 prod/staging 환경에 따른 버전 관리 워크플로우를 나타내고 있으며, 기본적인 흐름은 다음과 같습니다.

1. 테스트 단계
2. (`on: pull_request: release-**`) 코드 변경 감지
  - 네이티브 코드 변경 시: Staging 환경에 Android/iOS 배포
  - JS 코드 변경 시: 외부 저장소에 Staging 환경의 버전 정보 및 번들 업데이트
3. (`on: pull_request: main`) 프로덕션 배포
  - 최신 버전 유지를 위해 Android/iOS를 앱 스토어에 배포
4. (`on: push: main`) 프로덕션 버전 정보 및 번들 업데이트

> 버전 관리 워크플로우는 아래 Gist를 참고해주세요!
>
> [https://gist.github.com/BangDori/d8ce7347e3b1e5d25ffade8ff51f4b62](https://gist.github.com/BangDori/d8ce7347e3b1e5d25ffade8ff51f4b62)

이러한 워크플로우를 통해 외부 저장소에 기록된 버전 정보를 수동으로 업데이트하는 과정에서 발생할 수 있는 실수를 방지하여 안정성을 높일 수 있게 되었습니다.