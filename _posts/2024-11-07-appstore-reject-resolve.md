---
title: App Store 심사 Reject 사유 해결하기
description: 앱 스토어에 제출할 땐 애플 로그인을 반드시 추가하자
date: 2024-11-07 00:19:00 +/-TTTT
categories: [React Native, Distribution]
tags: [app store, distribution, reject]
sitemap:
  changefreq: daily
  priority: 0.5
---

## 1. 개요

앱 스토어에 앱을 등록하기 위해 앱을 제출했는데 2가지의 사유로 심사를 통과하지 못했습니다. 그래서 오늘은 심사를 통과하지 못한 사유와 이를 어떻게 해결하였는지에 대해 공유해보겠습니다.

## 2. Performance: App Completeness

> **Guideline 2.1 - Information Needed**
> 
> We have started the review of your app, but in addition to the demo account username and password you provided, we need an authentication code to complete the registration process.
> 
> App Review conducts a full review of all apps submitted to the App Store, including content and services that require authentication codes to access.
{: .prompt-warning }

해당 리젝 사유는 앱 심사를 진행할 때 모든 기능을 테스트할 수 있어야 하는데, 앱 데모를 위한 충분한 정보를 제공해 주지 않아서 발생한 문제입니다.

애플에서는 리젝 사유를 생각보다 상세하게 알려주는데 위 리젝 사유가 발생한다면 이미지가 첨부되어 있을 확률이 높습니다. 저희 앱의 경우 SMS 인증 화면이 이미지로 첨부되어 있어 이를 해결하기 위해 서버에서 데모용 SMS 인증을 위해 테스트 번호(000-0000-0000)와 인증 번호(000000)를 각각 저장해두고 다음과 같이 메모에 정보를 작성하여 제출하였습니다.

- 아이디: test1234
- 비밀번호: test1234!
- SMS 인증을 위한 데모 휴대폰 번호: 000-0000-0000
- SMS 인증을 위한 데모 인증번호: 000000

## 3. Design: Login Services

> **Guideline 4.8 - Design - Login Services**
>
> The app uses a third-party login service, but does not appear to offer an equivalent login option with the following features:
> 
> - The login option limits data collection to the user’s name and email address.
> - The login option allows users to keep their email address private as part of setting up their account.
> - The login option does not collect interactions with the app for advertising purposes without consent. 
{: .prompt-warning }

사실 해당 리젝을 해결하기 위해 많은 시간을 소요했는데 해당 리젝은 **다른 소셜 로그인 서비스를 사용한다면 계정에서 이메일을 가릴 수 있는 소셜 로그인을 추가**하라는 의미인데 조금 내려서 내용을 더 살펴보면 이런 문장을 확인할 수 있습니다.

- Note that Sign in with Apple meets the requirements specified in [guideline 4.8](https://developer.apple.com/app-store/review/guidelines/#login-services).

Apple로 로그인 기능은 가이드라인 4.8에 명시된 요구 사항을 충족하기에 Apple 로그인을 적용하면 해결할 수 있습니다. 저는 React Native를 사용하고 있어, [react-native-apple-authentication](https://github.com/invertase/react-native-apple-authentication) 라이브러리를 이용하여 Apple 로그인을 적용하였습니다. 해당 라이브러리를 적용할 때는 아래의 요구 사항을 충족해야 합니다.

- React Native version 0.60 or higher.
- Xcode version 11 or higher.
- iOS version 13 or higher.

## 참고

- [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
