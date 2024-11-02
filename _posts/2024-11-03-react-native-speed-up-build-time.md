---
title: React Native 빌드 시간 개선하기
description: 불필요한 빌드 시간을 줄이고 개발 생산성을 향상하자
date: 2024-11-03 02:46:00 +/-TTTT
categories: [Frontend, React Native]
tags: [react native, abi, build]
image:
  path: assets/img/thumbnail/react-native-speed-up-build-time.png
sitemap:
  changefreq: daily
  priority: 0.5
---

## 1. 개요

어느 순간부터 React Native 앱을 빌드하는데 1분 이상의 시간이 소요되고 있어 빌드하는 시간이 지루하게 느껴지게 되었고, 개발 생산성을 저하시킨다고 판단하게 되었습니다. 이게 1분이 말이 1분이지, 네이티브 소스를 수정할 때마다 그리고 앱을 다시 실행하려고 할 때 마다 다시 빌드 시간을 거쳐야 했기 때문에 n배의 시간이 소요되고 있었습니다.

그래서 오늘은 React Native에서 제공해주는 공식 문서를 참고하여 이러한 불필요한 빌드 시간을 개선해보도록 하겠습니다.

## 2. Build only one ABI during development (Android-only)

로컬에서 안드로이드 앱을 빌드하는 경우 기본적으로 4가지의 Application Binary Interfaces (ABIs)를 모두 빌드하도록 되어 있습니다. 그러나 개발 중에는 특정 ABI만 빌드해도 충분하기에 모든 ABI를 빌드할 필요는 없습니다. 

React Native CLI에서는 아래의 명령어와 같이 `run android` 명령어에 `--active-arch-only` 플래그를 추가하여 단 하나의 ABI만 빌드되도록 할 수 있습니다.

```shell
yarn react-native run-android --active-arch-only
```

위 명령어를 실행하게 되면 특정 ABI를 타겟으로 빌드하기 때문에, 아래 이미지와 같이 터미널에서 info Detected architectures arm64-v8a가 표시되는 것을 확인할 수 있습니다.

![detected_architectures](assets/img/writing/9/detected_architectures.png){: width="480" }

이러한 플래그 옵션을 통해 하나의 ABI를 빌드하게 되면 네이티브 빌드 시간이 이론적으로 약 75% 정도 단축된다고 공식 문서에 적혀있지만, 저의 경우에는 아직까지 네이티브 빌드 시간이 크게 길지 않아 다음과 같이 개선되었습니다.

![android build time](assets/img/writing/9/android_build_time.png){: width="480" }
_위: 플래그 적용 전, 아래: 플래그 적용 후_

- App Build Time: 55s -> 30s
- App Start Time: 68.16s -> 38.45s

## 3. Use a compiler cache

React Native의 고질적인 문제점 중 하나라면 긴 빌드 시간이라고 할 수 있는데, 이렇게 긴 빌드 시간이 걸리는 주요 원인은 바로 JS와 Native 코드 간의 번들링 과정에 있습니다. React Native에서는 앱을 끄고 재실행하게 되면 처음부터 다시 모든 파일을 번들링하게 되는데 이는 캐싱이 되어있지 않기 때문에 매번 동일한 작업을 진행하게 됩니다. 이번 챕터에서는 기존 빌드의 컴파일을 캐시하고 중간 컴파일 결과가 저장된 경우 컴파일을 건너뛰는 방식으로 빌드 속도를 개선해보겠습니다.

### 3-1. Enable the Build Cache on Gradle

Gradle의 [Enable the Build Cache](https://docs.gradle.org/current/userguide/build_cache.html#sec:build_cache_enable)를 살펴보면 Gradle은 기본적으로 빌드 캐시를 활성화하지 않습니다. 빌드 캐시는 두 가지의 방법으로 활성화할 수 있습니다.

1. 안드로이드 앱을 실행할 때 `--build-cache` 플래그 추가하기
2. `gradle.properties`에 `org.gradle.caching=true` 추가하기

둘 중 편한 방법을 선택하시면 됩니다. 저는 2번 방법을 적용하였습니다. 그 결과 앞서 30초의 빌드 시간이 14초로 개선되었습니다.

![android_cache_build_time](assets/img/writing/9/android_cache_build_time.png){: width="480" }

### 3-2. Local caches

iOS에서는 [ccache](https://ccache.dev/)를 이용하여 빌드 캐시를 저장하는 방식으로 개선해보겠습니다.

우선 macOS의 경우에는 `brew install ccache`를 이용하여 설치합니다. 그리고 NDK 컴파일 결과를 캐싱하기 위해 다음과 같이 심볼릭 링크를 구성합니다.

> NDK: Native Development Kit의 약자로, Android 앱에서 네이티브 코드를 작성하고 실행할 수 있도록 지원하는 개발 도구입니다. 주로 성능이 중요한 부분에서 자바스크립트 또는 Java만으로는 성능이 부족할 때 C나 C++와 같은 네이티브 언어로 코드를 작성하여 성능을 향상시킬 수 있도록 도와줍니다.

```shell
ln -s $(which ccache) /usr/local/bin/gcc
ln -s $(which ccache) /usr/local/bin/g++
ln -s $(which ccache) /usr/local/bin/cc
ln -s $(which ccache) /usr/local/bin/c++
ln -s $(which ccache) /usr/local/bin/clang
ln -s $(which ccache) /usr/local/bin/clang++
```

심볼릭 링크 생성이 완료되었다면, which 커맨드를 사용하여 정상적으로 동작하는지 확인합니다.

```bash
$ which gcc
/usr/local/bin/gcc
```

이때 만약 output이 `/usr/local/bin/gcc`가 아닌 `/usr/bin/gcc`로 나온다면 `/usr/bin`이 `/usr/local/bin`보다 앞에 위치해있다는 것이므로 아래 명령어를 통해 `/usr/local/bin`이 앞에 위치하도록 설정합니다.

```shell
export PATH="/usr/local/bin:$PATH"

source ~/.bashrc
or
source ~/.zshrc
```

안드로이드에서는 `android/app/build`를 제거한 이후 다시 앱을 빌드하면 `ccache -s` 옵션을 통해 cache hit/miss를 확인할 수 있습니다.

iOS에서는 추가 설정이 필요합니다.

1. Xcode와 xcodebuild가 컴파일러 명령을 호출하는 방식을 변경해야 합니다. 기본적으로 컴파일러 바이너리에 저장된 지정 경로를 사용하기 때문에 상대 경로를 구성해야 합니다.

```ruby
post_install do |installer|
  react_native_post_install(installer)

  # ...possibly other post_install items here

  # ✨ 추가된 부분
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings["CC"] = "clang"
      config.build_settings["LD"] = "clang"
      config.build_settings["CXX"] = "clang++"
      config.build_settings["LDPLUSPLUS"] = "clang++"
    end
  end

  __apply_Xcode_12_5_M1_post_install_workaround(installer)
end
```

2. Xcode 컴파일 중에 ccache가 캐시 적중을 할 수 있도록 캐시 동작을 허용하는 ccache 구성이 필요합니다.

```shell
export CCACHE_SLOPPINESS=clang_index_store,file_stat_matches,include_file_ctime,include_file_mtime,ivfsoverlay,pch_defines,modules,system_headers,time_macros
export CCACHE_FILECLONE=true
export CCACHE_DEPEND=true
export CCACHE_INODECACHE=true
```

![ios_build_time](assets/img/writing/9/ios_build_time.png){: width="480" }

위와 같이 적용한 결과 iOS 빌드 시간이 85.55s -> 29.80s로 개선되었습니다.

## 4. 마치며

포스팅 내용은 길지 않지만, 빌드 타임 개선을 빌드 타임 개선을 이루어나가는 과정에서 수치상에 개선뿐만 아니라 Android에서 사용되는 ABI, gradle 등에 대한 추가적인 이해를 할 수 있어서 즐거운 시간이었습니다.

## 참고

- [Speeding up your Build phase](https://reactnative.dev/docs/build-speed)
- [[React Native] 빌드 속도 향상시키기 (캐쉬화)](https://cotist.tistory.com/26)
- [Revolutionise Your Workflow: Cut React Native Build Time by 67%!](https://medium.com/engineering-at-bajaj-health/revolutionise-your-workflow-cut-react-native-build-time-by-67-68a47dbf993b)
- [Reduce React Native XCode build time](https://zanechua.com/blog/reduce-react-native-xcode-build-time/)