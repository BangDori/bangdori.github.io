---
title: self-hosted runner로 Github Actions 실행하기
description: 생각보다 너무 간단한 self hosted runner 설정
date: 2024-10-03 12:35:00 +/-TTTT
categories: [DevOps]
tags: [github actions, self-hosted]
image:
  path: assets/img/writing/github_actions_hosting_flow.png
---

### 1. 개요

Github Actions를 사용하다가 빌드 타임이 너무 길어지게 되었고 Github Server에서 제공해 주는 호스팅 실행기를 사용하다가는 월간 무료 제공 시간을 다 사용해 버릴 것 같아서 self-hosted runner 방식으로 변경하게 되었는데 이 과정에서 알게 된 지식을 공유하고자 글을 작성하게 되었다.

### 2. About Github Actions

self-hosted runner는 Github에서 Github Actions의 작업을 실행하기 위해 배포하고 관리하는 시스템이다. 우선 self-hosted runner에 대해 알아보기 이전에 간단하게 Github Actions의 flow에 대해 알아보자.

![github_actions_flow](assets/img/writing/github_actions_flow.png)

Github Actions가 trigger 되기까지의 과정은 생각보다 단순하다.

1. 개발자가 작성한 코드를 Github에 develop 브랜치에 `push` 한다.
2. develop 브랜치에서 main 브랜치로 `pull_request` 한다.
3. 이때 `pull_request` 요청을 한 브랜치의 `.github/workflows` 폴더에 있는 workflow를 확인한다.
4. Github Actions가 workflow를 실행한다.

> 그렇다면 여기서 궁금증이 생길 수 있는데, 도대체 이 Github Actions는 누가? 어디에서 실행해 주는 것일까?

[About GitHub-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/using-github-hosted-runners/about-github-hosted-runners)에서 답을 찾을 수 있었다. 우리가 워크플로우에서 실행 환경을 `ubuntu-latest`와 같이 작성해 주면 Github Actions에서는 해당 운영체제에 맞는 호스팅 실행기를 제공해 준다. 그림으로 알아보자.

![github_actions_hosting_flow](assets/img/writing/github_actions_hosting_flow.png)

1. workflow에 설정한 이벤트(push, pull_request 등)에 의해 Github Actions가 trigger 된다.
2. 트리거가 실행되면 Github는 사용자가 정의한 `runs-on` 키에 따라 워크플로우가 실행될 가상 머신을 선택한다.
  - 2-1. `runs-on`이 `ubuntu-latest`, `windows-latest`, `macos-latest` 등과 같이 정의되어 있다면 Github는 해당 OS가 설치된 호스팅 실행기를 할당한다.
  - 2-2. `runs-on`이 `self-hosted`로 정의되어 있다면 Github는 해당 리포지토리의 Runners에 등록된 호스팅 실행기를 할당한다.
3. 할당된 호스팅 실행기에서 workflow를 실행한다.

<br />

> **runs-on 조건문이 Github Server 내부에 위치한 이유**
> 
> Github Server에서는 workflow에 작성된 실행 환경(`runs-on`)을 확인하고 어떤 호스팅 실행기를 할당할지를 결정한다. 그렇기 때문에 실제로 `self-hosted` 방식으로 하기 위해서는 `self-hosted` 호스팅 실행기를 할당하기 위해 Github Server에 등록하는 과정을 거쳐야 한다.
{: .prompt-info }

이제 이 그림에서 우리가 주목할 부분은 self-hosted runner 방식으로 환경을 설정하여 나의 로컬 서버에서 workflow를 실행하는 것이다.

### 3. Setup self-hosted runner

self-hosted runner 방식은 어렵지 않다.

![self_hosted_runner_setup_1](assets/img/writing/self_hosted_runner_setup_1.png)

1. self-hosted runner를 등록하려는 리포지토리의 Settings 탭으로 이동한다.
2. 좌측 Aside에서 Actions 탭에 Runners로 이동한다.
3. New self-hosted runner를 클릭한다.

![self_hosted_runner_setup_2](assets/img/writing/self_hosted_runner_setup_2.png)

이 창에서 자신의 운영체제를 선택하고 아래 Download, Configure에 나온 명령어를 순서대로 자신의 터미널에서 입력한다.

- `./config.sh`를 하면 입력창이 나오는데 여기서 별다른 입력 없이 Enter를 입력하자.

![self_hosted_runner_setup_3](assets/img/writing/self_hosted_runner_setup_3.png)

그럼 이렇게 self-hosted runner가 정상적으로 등록된 것을 확인할 수 있다. 여기서 주의할 점이 있다.

1. workflow 실행 환경 변경 
  - 위 이미지에 나온 실행 환경을 workflow에 추가해 주면 된다. `runs-on: [self-hosted, macOS, x64]`
2. run.sh 실행
  - 현재 상태가 Offline이라면 Github Actions가 trigger 되더라도 Github에서 로컬 호스팅 실행기를 찾지 못하기 때문에 정상적으로 workflow가 실행되지 않는다. 앞서 설치한 actions-runner 폴더 내부에서 run.sh를 실행해 주자.

```shell
./run.sh
```

터미널에서 실행하게 되면 상태가 Offline에서 Idle로 변환된 것을 확인할 수 있다. 그리고 이제 pull_request를 작성하여 확인해 보자.

![self_hosted_runner_setup_4](assets/img/writing/self_hosted_runner_setup_4.png)

터미널에서는 job의 실행 과정이 표기되며 Github Actions 탭에서는 워크플로우의 결과를 확인할 수 있다. 워크플로우 파일은 아래와 같다.

```yml
name: test

on:
  pull_request:
    branches:
      - "main"

jobs:
  hello_world_job:
    runs-on: [self-hosted, macOS, x64]

    steps:
      - uses: actions/checkout@master

      - name: Install correct Bundler version
        run: echo "Hello, self hosted runner!"
```

self-hosted를 테스트한 리포지토리는 아래에 링크로 첨부해 둘 테니 이 글을 보는 사람들에게 도움이 되었으면 좋겟다.
- [https://github.com/BangDori/self-hosted-test-repo/pull/1](https://github.com/BangDori/self-hosted-test-repo/pull/1)

## 참고

- [Understanding Github Actions](https://docs.github.com/en/actions/about-github-actions/understanding-github-actions)
- [About self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners)