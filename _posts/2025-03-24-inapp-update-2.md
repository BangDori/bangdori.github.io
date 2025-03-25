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

글을 시작하기 앞서, 이전에 작성했던 [React Native In-App Updates](https://bangdori.kr/posts/inapp-update/)에서는 아래의 문제들이 발생하고 있었습니다.

1. 신규 사용자의 번들 초기화 문제
2. 버전별 Hotfix 처리의 어려움
3. 점진적 업데이트의 부재
4. (추가) 불필요한 워크플로우 동작

생각보다 많은 문제가 한꺼번에 드러나다 보니(하하), 이번에는 우선 인앱 업데이트를 왜 도입했는지를 명확히 정리한 뒤, 그 목적에 맞춰 문제들을 어떻게 해결해 나갔는지 공유드리려 합니다.

이번 글에서는 다음 내용을 중심으로 다루고자 합니다:

- 인앱 업데이트를 도입한 목적
- 실제 서비스 흐름에 맞춘 개선 방향
- 버전 관리 플로우 개선

해당 내용은 저희 서비스 상황에 맞게 커스터마이징한 방식이기 때문에, 인앱 업데이트를 도입하실 분들은 각자의 서비스 환경에 맞게 참고하시면 좋겠습니다.

## 2. 인앱 업데이트 서비스에 맞게 적용하기

문제를 다루기 이전까지는 서비스에 인앱 업데이트를 적용하는 목적이 다소 뚜렷하지 않았습니다. 그래서 우선 서비스에서 인앱 업데이트를 도입하는 이유를 다음과 같이 명확히 하고자 했습니다.

### 2-1. 인앱 업데이트의 목적 구체화

> 인앱 업데이트를 통해 신규 기능을 빠르게 제공할 수 있어야 하며, 에러 발생 시 빠른 대응이 가능해야 한다.

위와 같은 목적을 설정한 이유는 현재 서비스가 MVP 단계에서 운영되고 있고, 기능이 1~2주 간격으로 계속 추가되고 있기 때문입니다. 이러한 상황에서는 버전을 나눠서 에러에 대응하기보다는, <span style="color: #00CA5B;">사용자가 최대한 빠르게 새로운 기능을 접할 수 있도록 최신 버전을 유지하는 것</span>이 더 효과적이라고 판단했습니다.

이처럼 현재의 서비스 상황에 맞춰 인앱 업데이트의 목적을 명확히 설정하고 나니, 과거에 문제라고 여겼던 일부 항목들이 실제로는 당장 해결이 필요하지 않은 사안으로 판단되었습니다:

2. 버전별 Hotfix 처리의 어려움
3. 점진적 업데이트의 부재

예전에는 이 두 가지가 인앱 업데이트의 핵심 과제로 느껴졌지만, **"최신 버전에 가깝게 유지하는 전략"**을 택하면서, 특정 버전에 맞춰 핫픽스를 배포하거나 점진적으로 나눠 배포할 필요성이 자연스럽게 줄었습니다.

물론 서비스 규모가 커지고 사용자 수가 많아진다면, 버전별 제어, 점진적 배포는 다시 중요한 이슈로 떠오를 수 밖에 없습니다. 다만 현재는 사용자가 원하는 기능을 빠르게 전달하는 것이 더 효과적이라고 판단했습니다.

이에 따라, 이번 개선에서는 "<span style="color: #00CA5B;">신규 사용자의 번들 초기화 문제</span>"에 먼저 집중하게 되었습니다.

### 2-1. 외부 버전 정보와 설치된 앱 버전의 정합성 맞추기

기존에는 외부 스토리지에 버전 정보가 업데이트되는 시점과 앱 제출이 완료되는 시점 사이에, 신규 사용자의 앱 버전이 올바르게 초기화되지 않는 문제가 발생했었습니다. 저는 이 문제를 **"신규 사용자의 번들 초기화 문제"**라고 정의했는데요, 구체적인 상황은 아래와 같습니다.

![Bundle Issue](assets/img/writing/17/bundle_issue.png){: width="720" }
_번들 초기화 문제_

위 이미지는 앞서 언급한 번들 초기화 문제를 시각화한 내용이며, 상황을 요약하면 다음과 같습니다.

1. 앱 스토어 심사를 받고 있는 동안, 외부 스토리지(예: S3)에 번들 버전 정보가 먼저 업데이트됨.
2. 기존 사용자 1
    - 기존에 번들 버전이 32(구버전)였는데, 인앱 업데이트 통해 최신 번들 버전(33)으로 업데이트 -> 문제 없음
3. 신규 사용자 1 (문제 발생)
    - 앱 스토어에서 32 버전을 다운로드
    - <span style="color: #FF6262;">최초 앱 시작 시 외부 스토리지의 번들 버전으로 버전을 초기화시켜, 번들 버전 33으로 초기화됨</span>
    - 앱 내부 로직과 번들 내용이 불일치하며, 예상치 못한 버그 발생
    - 추가로 최신 버전(33)을 사용하지 못함
4. 신규 사용자 2
    - 앱 스토어에 1.6.1이 올라간 이후 다운로드 -> 문제 없음

#### 📌 여기서 문제점은?

"최신 버전 여부를 `bundleVersion`으로 비교하여 발생한 문제"

돌이켜보면 정말 사소하지만 치명적인 착각이었습니다. 개발 서버에서 동일한 `appVersion`으로 여러 번들 버전을 배포하다 보니(ex. `1.0.0(20)`, `1.0.0(21)`, `1.0.0(22)`), 앱 스토어에 올라가는 버전도 Unique하지 않다고 오해했고, 그로 인해 발생한 문제였습니다.

해당 문제를 해결하기 위해, 스토리지에서 보관하고 있는 json 정보를 다음과 같이 정리하였습니다.

```js
{
  "updatedAt": "2025-02-24T17:09:26Z",
  "appVersion": "1.6.0",
  "bundleVersion": 59, // @deprecated 더 이상 사용되지 않는 필드
  "marketUpdateVersion": "1.6.0",
  "downloadAndroidUrl": "https://storage.com/production/index.android.bundle.zip",
  "downloadIosUrl": "https://storage.com/production/main.jsbundle.zip",
}
```

1. `updatedAt`: 업데이트된 날짜
2. `appVersion`: 현재 앱 버전
3. `marketUpdateVersion`: 마켓 업데이트가 필요한 버전 (네이티브가 변경된 경우 버전을 최신화)
4. `downloadAndroidUrl`, `downloadIosUrl`: Android/iOS 번들 주소

위 버저닝을 적용한 후, `marketUpdateVersion`과 `appVersion`을 현재 사용자의 기기에 설치된 앱 버전을 비교하여 최신 버전인지를 비교하도록 다음과 같이 플로우를 구성하였습니다.

![최종 인앱 업데이트 플로우](assets/img/writing/17/final_inapp_update_process.png){: width="640" }
_최종 인앱 업데이트 플로우_

그 결과, 앱 최초 실행 시 사용자 기기에 설치된 버전을 기준으로 업데이트 여부를 판단할 수 있게 되었고, 별도로 초기 번들 버전을 관리하지 않아도 **정확한 버전 정합성을 유지**할 수 있게 되었습니다. 이를 통해 <span style="color: #00CA5B;">신규 사용자의 번들 초기화 문제도 자연스럽게 해결</span>되었으며, 추가적으로 인앱 업데이트가 **사용자 기기의 상태를 보다 정확하게 추적할 수 있는 구조**로 개선되었습니다.

## 3. 버전 관리 플로우 효율적으로 개선하기

이제 번들 버전이 불필요하게 되었으니 전체 버전 관리 플로우를 점검하고, 워크플로우를 좀 더 효율적으로 개선해보겠습니다.

![App Workflow](assets/img/writing/14/app_workflow.png){: width="640" }
_버전 관리를 위한 자동화 파이프라인 워크플로우 (구버전)_

기존 워크플로우에는 다음과 같은 두 가지 문제가 있었습니다.

1. PR을 올리기만 해도 불필요한 배포 플로우가 실행됨
2. 개발 버전에서도 인앱 업데이트가 동작됨 (즉, 외부 버전에 항상 동기화됨)

1번도 충분히 번거로운 문제였지만, 2번의 경우 개발 중인 앱이 항상 최신 버전의 인앱 업데이트를 강제 받기 때문에 <span style="color: #FF6262;">특정 버전에서의 디버깅이 필요한 시점에 큰 장애 요소</span>가 될 수 있었습니다.

이를 해결하기 위해 다음과 같은 기준으로 플로우를 재설계했습니다.

- 개발 버전은 release 브랜치에 새로운 기능이 올라올 때만 배포
- 개발 버전은 외부 스토리지의 버전 정보와는 별도로 동작

### 3-1. 워크플로우 1차 개선안

![App Workflow](assets/img/writing/17/updated_workflow.png){: width="640" }
_1차 개선안_

개발 버전의 외부 버전 업데이트는 제거하고, `release` 브랜치에 `push` 되는 시점에 새로운 빌드 번호의 앱을 업로드하도록 하였습니다. 하지만 이러한 문제를 해결하고 보니, 또 다른 문제가 등장했습니다.

![버전 차이 비교](assets/img/writing/17/diff_app_version.png){: width="640" }
_개발 버전과 운영 버전의 버전 정보 추적 문제_

기존에는 정식 출시될 버전의 스토리지 정보(`update.production.json`)를 개발 버전에서 사용하고 있던 스토리지 정보(`update.staging.json`)를 통해,

- 다음 배포에서 앱 버전이 몇으로 될 지
- 다음 배포에 네이티브 변경 여부가 있을지

등을 미리 확인할 수 있었지만, 개발 버전의 스토리지 정보를 더 이상 업데이트하지 않게 되면서 <span style="color: #FF6262;">향후 배포될 버전에 대한 정보를 사전에 파악하기 어려운 문제</span>가 생겼습니다.

물론 이것도 해결 방법이 있었습니다! 메인 브랜치에 PR을 올리는 시점(최신 버전이 앱 스토어에 제출되는 시점)에 해당 PR이 배포하게 될 버전 정보를 자동 코멘트로 표시하도록 개선하였습니다.

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

1차적으로는 네이티브 버전 변경에 단순하게 `android` 폴더와 `ios` 폴더의 해시 값을 확인하여, 변경된 경우 네이티브 버전(마켓 업데이트 필요)이 변경되는 것으로 하였습니다.

![댓글로 버전 정보 추적하기](assets/img/writing/17/comment_ota_update_version.png){: width="640" }

짜잔! 위와 같이 PR 코멘트를 통해 배포 예정 버전 정보를 자동으로 확인할 수 있게 되었습니다. 덕분에 네이티브의 변경이 발생했는지를 직접 추적하지 않고도, **배포될 앱 버전과 마켓 업데이트 여부를 사전에 명확히 파악할 수 있는 구조**를 만들 수 있었습니다.

이제 다음 단계는, **"마켓 업데이트가 필요한 네이티브 변경이 발생했는가?"**를 좀 더 명확하게 추적하는 것입니다.

### 3-2. 네이티브 변경 지점을 추적하자

기존에는 네이티브 변경 여부를 감지할 때 단순히 `android/` 혹은 `ios/` 폴더의 변경 여부만 체크하고 있었습니다.

하지만 이 방식은 너무 포괄적이기 때문에, 실제로 앱 기능에는 영향이 없는 파일 변경(예: 버전 번호만 바뀐 `info.plist`, `build.gradle`)까지도 "네이티브 변경"으로 잘못 판단하는 문제가 있었습니다.

그래서 다음과 같이, 실제로 앱의 기능이나 런타임 동작에 영향을 줄 수 있는 파일/폴더만을 선별하여 추적 기준으로 삼는 정책을 수립했습니다.

#### 📌 Android 관련 변경 감지 대상

| 파일/폴더 | 설명 |
| ------  | --- |
| `android/app/src/main/` | 네이티브 코드 (Java/Kotlin) 및 리소스 |
| `android/app/proguard-rules.pro` | 코드 난독화 및 최적화 규칙 |
| `android/build.gradle` | 빌드 설정, 네이티브 라이브러리 추가 등 |
| `android/gradle.properties` | Gradle 환경 변수 |
| `android/gradlew` | Gradle 실행 파일 |
| `android/link-assets-manifest.json` | 네이티브 자산 매니페스트 |
| `android/settings.gradle` | 모듈 설정 |
| `android/app/production/` | 프로덕션 빌드 관련 설정 |

#### 📌 iOS 관련 변경 감지 대상

| 파일/폴더 | 설명 |
| ------  | --- |
| `ios/firebaseConfigGoogleService-Info.plist` | Firebase 설정 파일 |
| `ios/(project)/Images.xcassets` | 앱 아이콘 및 리소스 |
| `ios/(project)/AppDelegate` | iOS 앱 진입점 |
| `ios/(project)/LaunchScreen` | 스플래시 화면 |
| `ios/(project)/main` | 앱 실행 진입점 |
| `ios/(project)/(project).entitlements` | 앱 권한 관련 설정 |
| `ios/(project).xcodeproj/xcshareddata/xcschemes/(project).xcscheme` | Xcode 빌드 스킴 |
| `ios/Podfile`, `ios/Podfile.lock` | iOS 의존성 관리 |

#### 📌 제외한 파일

| 파일/폴더 | 설명 |
| ------  | --- |
| `android/app/build.gradle`, `ios/(project)/info.plist` | 단순 버전/빌드 넘버 변경은 무조건 발생하므로, 감지 대상에서 제외 |
| `ios/(project).xcodeproj/project.pbxproj` | Xcode의 자동 수정이 잦으며, 앱 기능과 직접적인 관련이 없음 |

위 정책을 기반으로 네이티브 변경 지점을 더욱 상세히 추적하기 위해 워크플로우에서 `git diff`로 파일 목록을 추출하고, 위 파일들에 해당하는 변경 사항이 있을 경우 `NATIVE_CHANGED=true`로 설정하도록 개선했습니다.

{% raw %}
```yml
# 🚨 기존 버전 정보 추출 (포괄적인 범위)
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

# ✅ 변경된 버전 정보 추출 (네이티브 변경 지점 추적)
- name: Check for Native Changes
  id: check_native
  run: |
    CHANGED_FILES=$(git diff --name-only HEAD~1)

    if echo "$CHANGED_FILES" | grep -E \
    '^(android/app/src/main/|android/app/proguard-rules.pro|android/build.gradle|android/gradle.properties|android/gradlew|android/link-assets-manifest.json|android/settings.gradle|android/app/production/|ios/firebaseConfigGoogleService-Info.plist|ios/(project)/Images.xcassets|ios/(project)/AppDelegate|ios/(project)/LaunchScreen|ios/(project)/main|ios/(project)/(project).entitlements|ios/(project).xcodeproj/xcshareddata/xcschemes/(project)Prod.xcscheme|ios/Podfile|ios/Podfile.lock)'; then
      echo "NATIVE_CHANGED=true" >> $GITHUB_ENV
    else
      echo "NATIVE_CHANGED=false" >> $GITHUB_ENV
    fi
```
{% endraw %}

이를 통해, 불필요한 마켓 업데이트 버전의 증가를 줄일 수 있었고 실제로 네이티브 변경이 필요한 상황에서만 정확하게 감지하고 대응할 수 있는 구조를 갖추게 되었습니다.

