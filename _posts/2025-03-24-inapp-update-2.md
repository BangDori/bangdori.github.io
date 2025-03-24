---
title: In-App Updates의 버전을 관리해보자
date: 2025-03-24 12:58:00 +/-TTTT
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
> 2. 인앱 업데이트 서비스에 맞게 적용하기
> 3. 버전 관리 플로우 효율적으로 개선하기
{: .prompt-tip }

## 1. 개요

이전에 작성했던 [React Native In-App Updates](https://bangdori.kr/posts/inapp-update/)에서는 인앱 업데이트 적용 시, 아래의 문제가 발생하고 있었습니다.

1. 신규 사용자의 번들 초기화 문제
2. 버전별 Hotfix 처리의 어려움
3. 점진적 업데이트의 부재

이에 더해 워크플로우도 불필요하게 동작하고 있었습니다. 그래서 이번 게시물에서는 이러한 문제를 어떻게 개선했는지에 대해 소개해보려고 합니다. 이번 게시물에서 다룰 내용은 다음과 같습니다.

1. 인앱 업데이트 서비스에 맞게 적용하기
2. 버전 관리 플로우 효율적으로 개선하기

오늘 제가 작성하는 게시물 내용은 저희 서비스에 맞게 제가 커스텀한 내용이므로, 인앱 업데이트를 도입할 예정이신 분들은 각자 서비스에 맞게 도입하시면 좋을 것 같습니다.

## 2. 인앱 업데이트 서비스에 맞게 적용하기

우선 이전에 발생했던 문제점들을 개선하기에 앞서, 저희 서비스에서 인앱 업데이트를 도입하는 이유를 명확히 하고자 했습니다. 저희 서비스에서는 인앱 업데이트를 다음과 같은 목적으로 사용하고 있습니다.

> **목적**
> 
> 1. 에러 발생 시 빠른 대응
> 2. 신규 기능에 추가 시, 사용자의 간편한 업데이트 (네이티브 변경이 없는 경우)

저희 서비스는 아직 MVP 단계로 출시된 상태이며, 기능이 1~2주 간격으로 계속해서 추가되고 있는 상황입니다. 그렇다 보니, 버전을 세세하게 분기해서 버전별 에러에 대응하는 것에 초점을 맞추기 보다는 <span style="color: #00CA5B;">사용자가 신규 기능을 빠르게 접할 수 있도록 최신 버전에 가깝게 유지</span>하는 게 핵심이었습니다.

이러한 목적을 가지고 인앱 업데이트를 바라보게 되니, 이전에 문제라고 여겼던 항목들 중 일부는 **현재 해결하지 않아도 되는 사안**으로 바뀌게 되었습니다.

2. 버전별 Hotfix 처리의 어려움
3. 점진적 업데이트의 부재

과거에는 이 두 가지 이슈가 인앱 업데이트의 가장 큰 과제로 느껴졌지만, 최신 버전에 가깝게 유지하는 전략을 택하면서, 특정 버전에 맞춰 핫픽스를 배포하거나 점진적으로 나눠 배포할 필요성이 자연스럽게 줄었습니다. 

물론 서비스 규모가 커지고 사용자 수가 많아진다면, 버전별 제어, 점진적 배포는 다시 중요한 이슈로 떠오를 수 밖에 없습니다. 다만 <span style="color: #00CA5B;">현재는 사용자들이 원하는 기능을 만드는 데 초점을 맞추고, 빠르게 서빙하는 것이 더 효과적이라고 판단했습니다.</span>

그래서 우선은 신규 사용자의 번들 초기화 문제에 집중했습니다.

### 2-1. 외부 버전 정보와 설치된 앱 버전의 정합성 맞추기

초기에는 외부 스토리지에 업데이트하는 시점과 앱 제출이 완료되는 시점 사이에, 신규 사용자의 번들 버전이 올바르게 초기화되지 않는 문제가 있었습니다.

![Bundle Issue](assets/img/writing/17/bundle_issue.png){: width="720" }
_번들 초기화 문제_

위 이미지는 앞서 언급한 번들 초기화 문제를 시각화한 내용이며, 상황을 요약하면 다음과 같습니다.

1. 앱 스토어 심사를 받고 있는 동안, 외부 스토리지(예: S3)에 번들 버전 정보가 먼저 업데이트됨.
2. 기존 사용자 1
    - 기존에 번들 버전이 32(구버전)였는데, 인앱 업데이트 통해 최신 번들 버전(33)으로 업데이트 -> 문제 없음
3. 신규 사용자 1 (문제 발생)
    - 앱 스토어에어 32 버전을 다운로드
    - <span style="color: #FF6262;">하지만 외부 번들 버전 정보는 33이므로, 버전을 33으로 초기화</span>
    - 버그 발생 및 최신 버전(33)을 사용하지 못함
4. 신규 사용자 2
    - 앱 스토어에 1.6.1이 올라간 이후 다운로드 -> 문제 없음

#### 📌 여기서 문제점은?

"최신 버전인지를 판별하는 여부를 `bundleVersion`으로 비교하여 발생한 문제"

돌이켜보면 정말 사소하지만 치명적인 착각이었습니다. 개발 서버에서 동일한 appVersion으로 여러 번들 버전을 배포하다 보니, 앱 스토어에 올라가는 버전도 Unique하지 않다고 오해했고, 그로 인해 발생한 문제였습니다.

해당 문제를 해결하기 위해 우선 스토리지에서 보관하고 있는 json 정보를 다음과 같이 업데이트하였습니다.

```js
{
  "updatedAt": "2025-02-24T17:09:26Z",
  "appVersion": "1.6.0",
  "bundleVersion": 59, // deprecated
  "marketUpdateVersion": "1.6.0",
  "downloadAndroidUrl": "https://storage.com/production/index.android.bundle.zip",
  "downloadIosUrl": "https://storage.com/production/main.jsbundle.zip",
}
```

위 json 파일에 기록된 필드의 정보는 다음과 같습니다.

1. `updatedAt`: 업데이트된 날짜
2. `appVersion`: 현재 앱 버전
3. `marketUpdateVersion`: 마켓 업데이트가 필요한 버전 (네이티브가 변경된 경우 버전을 최신화)
4. `downloadAndroidUrl`, `downloadIosUrl`: Android/iOS 번들 주소

위 버저닝을 적용한 후, `appVersion`과 현재 사용자의 기기에 설치된 앱 버전을 비교하여 최신 버전인지를 비교하도록 하였습니다. 이렇게 함으로써 앱을 최초 시작한 사용자의 초기 번들 버전을 따로 설정할 필요가 없게 되었고, 그 결과 자연스럽게 인앱 업데이트가 사용자의 번들을 정확히 추적할 수 있게 되었습니다.

## 3. 버전 관리 플로우 효율적으로 개선하기

![App Workflow](assets/img/writing/14/app_workflow.png){: width="640" }
_버전 관리를 위한 자동화 파이프라인 워크플로우 (구버전)_

이전에는 위와 같이 워크플로우가 구성되어 있었는데, 크게 다음과 같은 문제가 있었습니다.

1. 불필요한 배포 플로우가 동작한다.
2. 개발 버전에서도 항상 최신 버전을 따라야 한다.

1번 문제는 큰 문제로 떠오르지 않을 수 있지만, 2번 문제의 경우 개발 버전이 항상 최신 버전을 따르기 때문에 <span style="color: #FF6262;">특정 버전에 대한 디버깅이 필요한 시점에 큰 문제</span>가 될 수 있었습니다. 그래서 이러한 플로우를 다음과 같이 개선하고자 했습니다.

1. 개발 버전은 새로운 기능이 release 브랜치에 올라올 때마다 배포되어야 한다.
2. 개발 버전이 업데이트될 때에는 외부 스토리지의 버전 정보를 업데이트하지 않는다.

1차적으로 개선된 플로우는 다음과 같습니다.

![App Workflow](assets/img/writing/17/updated_workflow.png){: width="640" }
_1차 개선안_

개발 과정에서의 버전 업데이트를 제거하고, release 브랜치에 push 되는 시점에 업데이트를 하도록 하였습니다. 하지만 문제 아닌 문제가 있었습니다.

![버전 차이 비교](assets/img/writing/17/diff_app_version.png){: width="640" }

이전까지는 개발 버전에 업데이트된 버전 정보를 통해 다음에 업데이트 될 인앱 업데이트를 위한 json 정보를 파악할 수 있었습니다. 자바스크립트가 변경되어 다음 버전에는 appVersion만 업데이트 될 예정이라거나, 네이티브가 변경되어 다음 버전은 마켓 업데이트가 버전이 최신 버전으로 업데이트 될 것이다 등

하지만 개발 버전에서는 더 이상 업데이트하지 않게 되면서, 더 이상 추적이 어려워지는 문제가 발생하게 되었습니다. 이를 해결하기 위해, 메인 브랜치로 PR을 올리는 시점(최신 버전이 앱 스토어에 제출되는 시점)에 업데이트 될 버전 정보를 다음과 같이 코멘트로 확인할 수 있도록 하였습니다.

#### 🚨 workflows 코드가 길 수 있으니 생략하셔도 좋습니다.

{% raw %}
```yml
name: ota-update-comment

on:
  pull_request:
    branches:
      - main

permissions:
  id-token: write
  contents: read
  actions: read
  pull-requests: write

jobs:
  ota-update-comment:
    runs-on: [self-hosted, macOS, x64]
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # 모든 Git 히스토리를 가져옴 (HEAD~1 사용 가능)

      # ✅ AWS Credentials 설정
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.ROLE_TO_ASSUME }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Download update.production.json from S3
        run: |
          aws s3 cp s3://storage/update.production.json ./update.production.json

      # 버전 정보 추출
      - name: Extract Version from Base Branch
        shell: bash
        run: |
          BASE_BRANCH="${{ github.head_ref }}"

          if [[ "$BASE_BRANCH" =~ ^(release|hotfix)-([0-9]+\.[0-9]+\.[0-9]+)$ ]]; then
            RELEASE_APP_VERSION="${BASH_REMATCH[2]}"
            echo "RELEASE_APP_VERSION=$RELEASE_APP_VERSION" >> $GITHUB_ENV
          fi

      # ✅ 네이티브 변경 감지
      - name: Check for Native Changes
        id: check_native
        run: |
          CHANGED_FILES=$(git diff --name-only HEAD~1)

          if echo "$CHANGED_FILES" | grep -E \
          '^(android/|ios/)'; then
            echo "NATIVE_CHANGED=true" >> $GITHUB_ENV
          else
            echo "NATIVE_CHANGED=false" >> $GITHUB_ENV
          fi

      # update.json 파일 읽기 (date, appVersin, marketUpdateVersion)
      - name: Read `update.production.json`
        run: |
          CURRENT_DATE=$(TZ=Asia/Seoul date +"%Y-%m-%dT%H:%M:%SZ") # 현재 날짜를 Seoul (KST, UTC+9) 기준으로 가져오기
          APP_VERSION=$(jq -r '.appVersion' update.production.json)
          MARKET_UPDATE_VERSION=$(jq -r '.marketUpdateVersion' update.production.json)

          if [[ "$NATIVE_CHANGED" == "true" ]]; then
            echo "NEW_MARKET_UPDATE_VERSION=$RELEASE_APP_VERSION" >> $GITHUB_ENV
          fi

          echo "CURRENT_DATE=$CURRENT_DATE" >> $GITHUB_ENV
          echo "OLD_APP_VERSION=$APP_VERSION" >> $GITHUB_ENV
          echo "OLD_MARKET_UPDATE_VERSION=$MARKET_UPDATE_VERSION" >> $GITHUB_ENV

      - name: Post PR Comment for OTA Update
        run: |
          MESSAGE="### 📢 OTA Update 예정 알림

          | 항목 | 기존 | 변경 |
          | :-: | :-: | :-: |
          | **앱스토어 업로드 예정 버전** | ${{ env.OLD_APP_VERSION }} | ${{ env.RELEASE_APP_VERSION }} |
          | **변경될 번들 버전** | ${{ env.OLD_APP_VERSION }} | ${{ env.RELEASE_APP_VERSION }} |
          | **변경될 네이티브 버전** | ${{ env.OLD_MARKET_UPDATE_VERSION }} | ${{ env.NEW_MARKET_UPDATE_VERSION || '기존 버전 유지' }} |
          "

          if [[ "$NATIVE_CHANGED" == "true" ]]; then
              MESSAGE="$MESSAGE

          ### 🚨 **네이티브 변경이 감지되었어요** 🚨
          - 이번 업데이트는 스토어에서 앱 업데이트가 필요할 수도 있어요."
          fi

          gh pr comment ${{ github.event.pull_request.number }} --body "$MESSAGE"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
{% endraw %}

네이티브 버전 변경에 단순하게 android 폴더와 ios 폴더의 해시 값을 확인하여, 변경된 경우 네이티브 버전(마켓 업데이트 필요)이 변경되는 것으로 하였습니다.

![댓글로 버전 정보 추적하기](assets/img/writing/17/comment_ota_update_version.png){: width="640" }

짜잔! 우리 애기 예쁘죠? 하지만 아직 개선할 부분이 많으니 쭉쭉 나가보겠습니다.

### 3-1. 외부 스토리지 업데이트 시점 정하기

현재 플로우는 main 브랜치에 push가 되어야만 외부 스토리지 버전이 업데이트되고 있었습니다. 그러다보니, 이전에 서비스를 사용하고 있는 사용자들에게 더 빠르게 서빙해줄 방법이 없었고 이슈가 발생하는 경우에도 대기를 해야만 했습니다. 빠르게 하더라도, 결국 수동으로 배포해야만 한다는 문제가 있었죠.

그래서 저는 이러한 자동화 과정을 조금은 다르게 접근했습니다.

"업데이트는 자동으로 하되, 시점은 내가 정할 수 없을까?"

이를 위해 

### 3-2. 네이티브의 변경 지점을 추적하자