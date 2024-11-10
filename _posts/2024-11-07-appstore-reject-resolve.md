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

Appe의 App Store 심사 가이드라인 4.8은 주로 앱이 타사 로그인 서비스(Facebook, Google, Twitter, Kakaotalk 등)을 사용하는데, 위와 같은 동등한 로그인 옵션을 제공하지 않기에 발생하는 리젝 사유입니다. 사실 위 리젝 사유는 조금만 자세히 살펴보면 리뷰를 살펴보면 하단에 아래와 같은 문장을 확인할 수 있습니다.

- Note that Sign in with Apple meets the requirements specified in [guideline 4.8](https://developer.apple.com/app-store/review/guidelines/#login-services).

즉 애플 로그인만 추가하면 충족할 수 있는 리젝 사유입니다. 

위 리젝 사유는 앱 심사에서 흔히 리젝되는 사유 중 하나인데요. 항상 모든 앱 심사에서 요구되는 가이드라인은 아닙니다. 그렇다면 위 리젝 사유가 발생하지 않는 경우에는 어떤 케이스가 있을까요?

### 3-1. 리젝 사유가 발생하지 않는 경우

1. "Sign in with Apple"을 이미 제공하는 경우
2. 타사 로그인 서비스를 사용하지 않는 경우
3. 로그인 기능이 필요 없는 경우

### 3-2. 리젝 사유가 발생하는 경우

1. 타사 로그인 서비스만 제공하고 "Sign in with Apple"을 제공하지 않는 경우
2. "Sign in with Apple"을 동등한 수준으로 제공하지 않는 경우
  - 로그인 화면에서 "Sign in with Apple" 버튼이 다른 로그인 옵션보다 눈에 띄지 않게 배치되거나, 크기나 디자인이 일관되지 않은 경우입니다.
3. 사용자 데이터 수집 방침이 가이드라인을 준수하지 않는 경우
  - 로그인 과정에서 사용자의 이름과 이메일 주소 외에 추가적인 개인정보를 수집하거나, 이메일 주소를 비공개로 유지할 수 있는 옵션을 제공하지 않을 때입니다.
  - 하지만 이 케이스의 경우에는 **추가적인 개인정보 수집에 대한 사유를 명확하게 작성해주면(설득하면) 통과되는 것 같습니다.**
4. 로그인 없이 앱의 핵심 기능을 사용할 수 없는 경우
  - Apple은 사용자가 로그인 없이도 앱의 기본 기능을 사용할 수 있도록 권장하며, 로그인 필수인 경우 가이드라인에 따라야 합니다.

## 4. 마치며

앱스토어에 앱을 제출할 때 만약 소셜 로그인 서비스가 포함되어 있다면 반드시 애플 로그인을 추가하자!

만약 소셜 로그인을 추가하는데 애플 로그인을 추가하지 않는다면 앱 스토어 심사에서 리젝될 것이고, 기존에 만들어두었던 회원가입 플로우가 꼬여서 애플 로그인을 추가하는 과정이 굉장히 고통스러워질 수 있다...

## 참고

- [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
