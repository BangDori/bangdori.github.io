---
title: 프론트엔드에서 개발 생산성 향상하기
description: pull request에서 변경된 UI를 실시간으로 볼 수 있다면..
date: 2024-10-14 20:26:00 +/-TTTT
categories: [DevOps]
tags: [frontend, 개발 생산성, auto assign, amplify, pull request]
image:
  path: assets/img/thumbnail/nextjs_and_amplify.png
---

## 1. 개요

사이드 프로젝트를 진행하게 되었는데 프로젝트 설정 초기 단계에서 매번 담당자와 리뷰어를 수동으로 설정해 줘야 하는 불편함이 생겨 이를 해결한 경험을 공유하고자 합니다. 추가로 pull request에서 변경된 UI를 실시간으로 볼 수 있도록 AWS amplify 설정까지 한 번 알아보겠습니다.

## 2. Auto Assign

![no_reviewers_and_assignees](assets/img/writing/6/no_reviewers_and_assignees.png){: width="360" }

Github에서 pull request를 작성하면 위 이미지와 같이 assignees와 reviewers가 아무도 등록되어 있지 않은 상태로 생성이 됩니다. 리뷰어가 한 명이거나 없는 경우에는 크게 불편하지 않을 수 있지만 만약 팀의 규모가 크다면 모든 팀원을 리뷰어로 등록하는 과정이 지루해지게 느껴질 수 있습니다. 오늘은 이러한 문제를 Auto Assign Action을 이용하여 해결한 경험을 작성해 보겠습니다.

우선 [Auto Assign Action](https://github.com/marketplace/actions/auto-assign-action)은 Github Marketplace에서 손쉽게 찾을 수 있는데, 이 도구를 사용하면 정말 쉽게 담당자와 리뷰어를 자동으로 할당할 수 있게 도와줍니다. 그리고 바로 해당 Action에 대한 사용법으로 넘어가 보겠습니다.

### 2-1. pr-auto-assign.yml

.github/workflow 디렉터리 내부에서 pr-auto-assign.yml을 생성합니다.

{% raw %}
```yml
# .github/workflow/pr-auto-assign.yml

name: auto-assign
on:
  pull_request:
    types: [opened, ready_for_review]

permissions:
  contents: read
  pull-requests: write

jobs:
  add-reviewers-and-assignee:
    runs-on: ubuntu-latest
    steps:
      - uses: kentaro-m/auto-assign-action@v2.0.0
        with:
          repo-token: '${{ secrets.GITHUB_TOKEN }}'
          configuration-path: '.github/auto_assign_config.yml'
```
{% endraw %}

위 작성된 pr-auto-assign.yml 워크플로우는 pr가 생성되었을 때와 draft 상태에서 pr로 변경될 때 실행됩니다. 그리고 한 가지 중요한 점이 있는데 permissions에서 pull-requests 옵션을 write로 설정해야 합니다. pull-requests 옵션을 write로 설정해 주지 않으면 해당 워크플로우가 PR 내용에 대한 수정 작업을 진행할 수 없어서 리뷰어와 담당자가 정상적으로 추가되지 않을 수 있습니다.

### 2-2. auto_assign_config.yml 파일 생성

./github 디렉터리 내부에서 앞서 작성한 워크플로우의 configuration-path의 파일명을 작성합니다. 그리고 해당 설정 파일에서는 리뷰어와 담당자를 어떻게 할당할 것인지 설정하는 파일인데 간단하게 `addReviewers`와 `addAssignees` 설정만 해주면 됩니다.

```yml
# .github/auto_assign_config.yml

addAssignees: author
addReviewers: true

numberOfReviewers: 0 # 해당 라인을 추가해주지 않으면 리뷰어가 1명만 등록되는 문제가 있어 0으로 명시해야 합니다.
```

- `addAssignees: author`: PR 작성자를 담당자로 설정합니다.
- `addReviewers: true`: 프로젝트에 참여하는 팀원들을 리뷰어로 등록합니다.

위와 같이 설정을 완료하고 PR을 등록해 주면 담당자는 PR을 작성한 나, 그리고 팀원들은 자동으로, 리뷰어로 등록이 되는 것을 확인할 수 있습니다.

![reviewers_and_assignees](assets/img/writing/6/reviewers_and_assignees.png){: width="360" }

## 3. AWS Amplify

전통적으로 애플리케이션 개발에는 백엔드 인프라 프로비저닝, 데이터베이스 관리, 사용자 인증, 프론트엔드 디자인 등의 많은 작업이 포함됩니다. 이렇게나 많은 작업으로 인해 개발 주기가 길어지고, 유지 관리 비용이 증가하며 개발자의 학습 곡선이 가파른 경우가 많았습니다.

하지만 현시점에서 클라우드 서비스의 기하급수적인 성장으로 개발자들은 인프라 관리의 복잡성을 추상화하여 사용자 경험과 혁신적인 기능을 만드는 데만 집중할 수 있는 플랫폼이 필요해지게 되었습니다. AWS Amplify는 이러한 복잡한 프로세스를 단순화시키기 위해 탄생한 솔루션입니다. 그렇다면 AWS Amplify에서는 어떤 기능을 제공해 줄까요?

aws amplify hosting 랜딩 페이지를 살펴보면 Key features에서 5가지의 기능을 소개하고 있습니다.

1. Feature branch deployments
2. Pull request previews
3. Easy custom domains with SSL
4. Redirects and custom headers
5. Monitoring

오늘은 이 5가지의 기능 중 2번 Pull request previews에 대해 알아보겠습니다.

### 3-1. Pull request previews

[Web previews for pull requests](https://docs.aws.amazon.com/amplify/latest/userguide/pr-previews.html)의 공식 문서를 살펴보면 다음과 같이 소개하고 있습니다.

> 웹 미리보기는 개발 및 품질 보증(QA) 팀이 프로덕션 또는 통합 브랜치에 코드를 병합하기 전에 PR의 변경 사항을 미리 볼 수 있는 방법을 제공합니다. PR을 사용하면 저장소의 브랜치에 푸시한 변경 사항을 다른 사람에게 알릴 수 있습니다.
{: .prompt-info }

PR의 변경 사항을 미리 볼 수 있다니 정말 생각만 해도 두근두근하군요. 그렇다면 바로 사이드 프로젝트에 적용을 해보겠습니다. 우선 AWS Amplify 콘솔로 이동하여 Deploy an app을 클릭합니다.

![amplify_step_1](assets/img/writing/6/amplify-1.png)

- Step 1. 연동하려는 저장소를 선택합니다. 저는 Github를 사용하고 있기 때문에 Github를 선택하겠습니다. 그리고 수동 배포를 하지 않고 Amplify Amplify에서 제공해 주는 Preview server를 이용할 것이기 때문에 다음으로 이동합니다.

![amplify_step_2](assets/img/writing/6/amplify-2.png)

- Step 2. 리포지토리를 연동합니다. 만약 리포지토리가 존재하지 않는다면 Github 권한 업데이트를 통해 AWS Amplify의 권한을 설정해 주어야 합니다. 리포지토리를 연동했다면 하단에서 어떤 브랜치에 대해 적용할 것인지를 선택합니다. 저는 main 브랜치로 pull request 할 경우에 웹 미리보기를 하기 위해 main 브랜치로 설정하였습니다.

![amplify_step_3](assets/img/writing/6/amplify-3.png)

- Step 3. 앱 설정을 진행하는데 빌드 설정에서 빌드 명령어와 빌드가 출력되는 디렉터리를 입력합니다. (AWS Amplify에서 자동으로 감지해서 입력해 주는 경우도 있습니다.) 환경 변수가 포함되어야 한다면 고급 설정에서 환경 변수를 추가해 줍니다. 그리고 추가로 내 사이트를 외부에서 접속하지 못하게 하고 싶다면 내 사이트를 비밀번호로 보호를 클릭하고 사용자 이름과 암호를 입력해 주고 유출되지 않도록 잘 보관해 줍니다.

![amplify_step_4](assets/img/writing/6/amplify-4.png)

그리고 마지막 단계로 생성한 앱의 호스팅 > 미리 보기로 이동하여 main 브랜치를 체크하고 설정 편집하여 pull 요청 미리 보기를 활성화해 줍니다. 

![success_aws_amplify](assets/img/writing/6/success_aws_amplify.png)

그리고 PR을 작성하면 다른 github actions가 좀 많긴 한데, 무시해 주세요..! AWS Amplify Console Web Preview만 확인해 보시면 정상적으로 Pass가 되고 댓글로 미리보기 preview url이 추가되는 것을 확인할 수 있습니다.

### 3-2. Amplify가 동작하지 않는다면

만약 위 이미지와 달리 AWS Amplify Console Web Preview가 표시되지 않는다면 AWS Amplify 빌드 에러일 문제가 있습니다. 문제의 원인을 파악하는 가장 빠른 방법은 AWS Amplify의 자신이 추가한 앱으로 이동하여 빌드 로그를 확인하면 됩니다!

저는 pnpm command를 찾을 수 없다는 에러가 발생하여 호스팅 > 빌드 설정으로 이동하여 아래와 같이 amplify.yml을 수정하였습니다.

```yml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - corepack enable # ✅ pnpm을 관리하는 corepack 활성화
        - corepack prepare pnpm@latest --activate # ✅ 최신 pnpm 설치 및 활성화
        - pnpm install --frozen-lockfile
    build:
      commands:
        - pnpm run build
  artifacts:
    baseDirectory: .next
    files:
      - '**/*'
  cache:
    paths:
      - .next/cache/**/*
      - node_modules/**/*
```

아 참! 현재 진행중인 사이드 프로젝트는 아래 링크에서 확인할 수 있습니다.

- [https://github.com/CollaBu/pennyway-client-webview-next](https://github.com/CollaBu/pennyway-client-webview-next)

## 참고

- [Github actions를 통한 PR Assignee & Reviewers 자동 할당](https://devjem.tistory.com/85)
- [What is AWS Amplify? A Complete Guide](https://www.theknowledgeacademy.com/blog/what-is-aws-amplify/)
- [AWS Amplify Hosting](https://aws.amazon.com/ko/amplify/hosting/?nc=sn&loc=2&dn=2)
- [Web previews for pull requests](https://docs.aws.amazon.com/amplify/latest/userguide/pr-previews.html)