---
title: React Native Fastlane Discord 알림 연동하기
date: 2024-11-30 15:43:00 +/-TTTT
categories: [DevOps]
tags: [app, react native, deployment, fastlane, notifier, discord]
image:
  path: assets/img/thumbnail/fastlane_with_discord_notifier.png
sitemap: 
    changefreq : daily
    priority : 0.5
---

## 1. 개요

앱 개발자로 일하면서 느낀 점은 개발하는 시간만큼이나 **배포에 많은 시간이 투자된다는 점**이었다.

![iOS Deploy Process](assets/img/writing/1/ios_deploy_process.png){: width="480" }
_iOS Internal Testing Deploy Process_

앱은 웹과 달리 상당히 복잡한 프로세스([Android & iOS Deploy Process](https://bangdori.kr/posts/internal-testing/#2-2-android-deploy-process))를 거쳐 배포되기에, 앱 수동 배포를 몇 번만 해봐도 진절머리가 난다. 심지어 자동화 배포를 구축하는 과정도 빌드 넘버 관리부터 시작해서 인증서 관리까지 생각보다 복잡하다. 다행히도 관련 자료가 많아 fastlane을 이용해 비교적 빠르게 해결할 수 있었다.

근데 문제는 배포 자동화를 적용하여 복잡한 프로세스를 간편하게 처리할 수 있게 되었는데, 배포가 끝났다는 공지를 위해 15~20분가량을 매번 기다리고 있었다는 것이었다. (진짜.... 바본가? 🥲)

그래서 오늘은 **알림 자동화 기능**을 추가하여 잃어버린 시간을 되찾아 보려고 한다.