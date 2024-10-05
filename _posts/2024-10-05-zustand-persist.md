---
title: Zustand persist로 지속적인 사용자 경험 제공하기
description: zustand랑 조금 더 친해져보자
date: 2024-10-05 21:24:00 +/-TTTT
categories: [Zustand]
tags: []
image:
  path: assets/img/thumbnail/
---

## 1. 개요

지금까지 전역 상태 관리를 위해 Zustand를 사용해 왔지만 단순 상태 관리 외에는 활용하지 못하고 있다는 느낌이 들었다. 이러한 와중에 Zustand persist 기능을 사용하게 되었는데 이 기능에 감동하여서 zustand와 조금 더 친해지고자 이 게시물을 작성하게 되었다.

## 2. Zustand

Zustand는 간소화된 Flux 원리를 사용한 작고, 빠르고, 확장 가능한 상태 관리 라이브러리로 최근 인기를 끌고 있는 라이브러리이다.

![zustand_bundle_phobia](assets/img/writing/5/zustand_bundle_phobia.png){: width="640" }
_https://bundlephobia.com/package/zustand@5.0.0-rc.2_

라이브러리의 사이즈가 얼마나 작냐면 실제로 MINIFIED 된 번들 사이즈가 1.2 kB밖에 되지 않는다. 이 수치는 다른 전역 상태 관리 라이브러리들과 비교하였을 때 현저히 적은 수치이다. 그렇다면 라이브러리의 번들 사이즈가 작은 만큼 기능도 다른 전역 상태 관리 라이브러리에 비해 적지 않을까? 라는 생각을 할 수 있는데 Zustand는 활발하게 개발되는 오픈 소스로 당당하게 지난 1년간의 다운로드 수 순위 중 2위를 차지하고 있을 만큼 인기가 있으며 다양한 기능들을 제공해 주고 있다.

![state_management_downloads](assets/img/writing/5/state_management_downloads.png){: width="640" }
_https://npmtrends.com/mobx-vs-recoil-vs-redux-vs-zustand_

그렇다면 이제 이렇게 인기를 끌고 있는 Zustand의 많은 기능 중 persist에 대해 알아보자.

### 2-1. 지속적인 사용자 경험 제공하기

Zustand persist를 사용하면 상태를 localStorage, AsyncStorage 등에 저장하여 데이터를 영구적으로 유지할 수 있게 해준다. 그렇기 때문에 사용자가 웹 브라우저 혹은 앱을 종료하고 다시 실행하더라도 이전 상태를 복원하여 **지속적인 사용자 경험을 제공**해줄 수 있다.

> **AsyncStorage**
> 
> AsyncStorage는 웹 브라우저에서의 localStorage와 유사하게 React Native에서 제공하는 간단한, 비동기적인 영구적인 키-값을 저장하는 저장소로, 주로 사용자별로 다른 `theme` 또는 `language`를 적용할 때 사용될 수 있다.
{: .prompt-tip }

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

type Theme = 'light' | 'dark' | 'auto';
type Language = 'en' | 'ko';

interface AppState {
  theme: Theme;
  language: Language;
  setTheme: (theme: Theme) => void;
  setLanguage: (language: Language) => void;
}

export const useAppStore = create<AppState>((set) => ({
  theme: 'auto', // 기본값 설정
  language: 'en', // 기본값 설정
  setTheme: (theme) => set(() => ({ theme })),
  setLanguage: (language) => set(() => ({ language })),
}));
```

위 저장소를 예시로 생각해 보자. 위 저장소에서는 현재 `theme`과 `language`에 대한 값들을 저장하고 있다. 하지만 위 상태는 값이 지속적으로 유지되지 않는다. 왜냐하면 단순히 사용자의 데이터를 `useAppStore`에서만 관리하고 있기 때문이다. 그렇다면 이를 localStorage와 연동하여 전역 저장소에 저장해보자.

```typescript
// ...
import { persist } from 'zustand/middleware';

export const useAppStore = create<AppState>()(
  persist(
    (set) => ({
      theme: 'auto', // 기본값 설정
      language: 'en', // 기본값 설정
      setTheme: (theme) => set(() => ({ theme })),
      setLanguage: (language) => set(() => ({ language })),
    }),
    {
      name: 'app-settings', // 로컬 스토리지에 저장될 키 이름
      // getStorage: () => localStorage, // (optional) by default, 'localStorage' is used
    }
  )
);
```

스토리지를 작성해 주지 않으면 기본적으로 localStorage가 사용된다.

![zustand_persist_store](assets/img/writing/5/zustand_persist_store.png)

> 그런데 여기서 한 가지 의문이 생겼다. `useAppStore`를 가져와서 상태를 변경하면 위 이미지와 같이 localStorage에 저장되지만, 상태를 변경하지 않는다면 localStorage에는 어떠한 값도 저장되지 않는다. 🧐 왜 그럴까? 

내 생각으로는 `useAppStore`의 상태를 변경시키지 않았다면 해당 상태가 초기 상태랑 동일하기 때문에 저장할 필요가 없어서 그런 것 같다. 하지만 이러한 추측은 언제나 사이드 이펙트를 낳는 법이니 Zustand 오픈 소스로 이동하여 persist가 어떻게 스토리지에 상태들을 저장하는지 확인해 보자.

```typescript
// https://github.com/pmndrs/zustand/blob/main/src/middleware/persist.ts

const setItem = () => {
    const state = options.partialize({ ...get() })
    return (storage as PersistStorage<S>).setItem(options.name, {
        state,
        version: options.version,
    })
}

api.setState = (state, replace) => {
    savedSetState(state, replace as any)
    void setItem()
}

// ...
```

> Zustand 오픈 소스에서 코드를 확인해 본 결과 "초기 상태와 동일하기 때문에 저장하지 않는다"라는 틀린 말이었다. 초기 상태와 동일하더라도 `setState` 메서드가 호출되면 스토어의 아이템들을 storage에 저장하는 방식을 가지고 있었다.
{: .prompt-warning }

![zustand_persist_store_2](assets/img/writing/5/zustand_persist_store_2.png)

localStorage를 확인해 본 결과 실제로 초기 상태와 동일하지만, 스토어의 상태가 업데이트될 때, 즉 `setState`가 호출될 때 localStorage에 저장된 것을 확인할 수 있었다.

### 2-2. 일부 상태 필드만 저장하기

> 늘 그렇듯이 변동 사항은 생기기 마련이다.

만약에 서비스의 타겟이 글로벌 서비스가 아닌 대한민국에 먼저 출시하게 되어 `language`의 값을 변경 불가능하게 고정해야 한다고 생각해 보자. 그렇다면 language의 초기 상태는 유지하되 더 이상 localStorage에 `language` 값을 저장할 필요가 없게 된다. 이럴 때 사용할 수 있는 것이 바로 `partialize`다. `partialize`를 사용하면 localStorage에 저장할 상태 필드를 선택할 수 있다.

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

type Theme = 'light' | 'dark' | 'auto';
type Language = 'en' | 'ko';

interface AppState {
  theme: Theme;
  language: Language;
  setTheme: (theme: Theme) => void;
  setLanguage: (language: Language) => void;
}

export const useAppStore = create<AppState>()(
  persist(
    (set) => ({
      theme: 'auto', // 기본값 설정
      language: 'ko', // 기본값 설정 (v1.0.0 고정)
      setTheme: (theme) => set(() => ({ theme })),
      setLanguage: (language) => set(() => ({ language })),
    }),
    {
      name: 'app-settings', // 로컬 스토리지에 저장될 키 이름
      // getStorage: () => localStorage, // (optional) by default, 'localStorage' is used
      partialize: (state) => ({ theme: state.theme }), // theme 상태만 저장
    }
  )
);
```

위와 같이 `partialize`를 적용하고 저장소를 업데이트한 후 localStorage를 확인해 보면 `theme`의 상태만 정상적으로 저장되는 것을 확인할 수 있다.

![zustand_persist_store_3](assets/img/writing/5/zustand_persist_store_3.png)

## 참고

- [Flux Overview](https://haruair.github.io/flux/docs/overview.html)
- [Persisting store data](https://zustand.docs.pmnd.rs/integrations/persisting-store-data)