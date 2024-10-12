---
title: 프론트엔드에서 개발 생산성 향상하기
description: pull request에서 변경된 UI를 실시간으로 볼 수 있다면..
date: 2024-10-13 00:00:00 +/-TTTT
categories: [DevOps]
tags: [frontend, 개발 생산성, auto assign, amplify, pull request]
image:
  path: assets/img/thumbnail/
---

## 1. 개요

사이드 프로젝트를 진행하게 되었는데 프로젝트 설정 초기 단계에서 매번 assignees, reviewers를 수동으로 설정해줘야 하는 불편함이 생겨 이를 해결해보고자 한다. 추가로 pull request에서 변경된 UI를 실시간으로 볼 수 있도록 AWS amplify 설정까지 진행해보자.

## 2. Auto Assign

![no_reviewers_and_assignees](assets/img/writing/6/no_reviewers_and_assignees.png){: width="360" }

Github에서 pull request를 작성하면 아래와 같이 아무것도 등록되어 있지 않은 상태로 표시가 된다. 여기서 Reviewers는 pull request를 확인해줄 리뷰어를 의미하며, assignees는 작업의 담당자를 의미한다. 리뷰어가 한 명이거나 없는 경우에는 크게 불편하지 않을 수 있지만 만약 팀의 규모가 크다면 모든 팀원들을 리뷰어로 등록하는 과정이 지루해지게 느껴질 수 있으니 이러한 과정을 auto-assign-action을 이용하여 자동화해보자.

Github Marketplace를 살펴보면 [Auto Assign Action](https://github.com/marketplace/actions/auto-assign-action)을 찾을 수 있는데 이를 적용하면 손쉽게 설정할 수 있다.

### 1. pr-auto-assign.yml

.github/workflow 디렉토리 내부에서 pr-auto-assign.yml을 생성합니다.

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
      - uses: kentaro-m/auto-assign-action@v1.2.5
        with:
          repo-token: '${{ secrets.GITHUB_TOKEN }}'
          configuration-path: '.github/AUTO_ASSIGN_CONFIG.yml'
```
{% endraw %}

위 작성된 pr-auto-assign.yml 워크플로우는 pr이 생성되었을 때와 draft 상태에서 pr로 변경될 때 실행됩니다. 그리고 한 가지 중요한 점이 있는데 permissions에서 pull-requests 옵션을 write로 설정해야 합니다. pull-requests 옵션을 write로 설정해주지 않으면 해당 워크플로우가 PR 내용에 대한 수정 작업을 진행할 수 없어서 리뷰어와 담당자가 정상적으로 추가되지 않을 수 있습니다.

### 2. AUTO_ASSIGN_CONFIG.yml 파일 생성

./github 디렉토리 내부에서 앞서 작성한 워크플로우의 configuration-path의 파일명을 작성합니다. 그리고 해당 설정 파일에서는 리뷰어와 담당자를 어떻게 할당할 것인지 설정하는 파일인데 간단하게 `addReviewers`와 `addAssignees` 설정만 해주면 됩니다.

```yml
# .github/AUTO_ASSIGN_CONFIG.yml

addAssignees: author
addReviewers: true
```

- `addAssignees: author`: PR 작성자를 담당자로 설정합니다.
- `addReviewers: true`: 프로젝트에 참여하는 팀원들을 리뷰어로 등록합니다.

위와 같이 설정을 완료하고 PR을 등록해주면 담당자는 PR을 작성한 나, 그리고 팀원들은 자동으로 리뷰어로 등록이 되는 것을 확인할 수 있습니다.

이미지!

## 2. AWS Amplify

## 참고

- [About code owners](https://docs.github.com/ko/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)