---
title: 자바스크립트 배열에 대해 알아보자
date: 2025-05-06 01:51:00 +/-TTTT
categories: [Frontend, JavaScript]
tags: [JavaScript Array, Sparse Array, ECMAScript, Frontend Performance, Browser Internals]
image:
  path: assets/img/thumbnail/js_array_empty.png
sitemap:
  changefreq: daily
  priority: 0.5
---

> 목차
> 1. 개요
> 2. 자바스크립트의 배열
> 3. 결론
{: .prompt-tip }

## 1. 개요

자바스크립트를 이용해 알고리즘 문제를 해결하는데 문득 궁금한 점이 생겼다.

```js
const arr = [];
arr[5] = 5;
```

나는 배열의 크기를 할당해주지 않았는데 어떻게 5번 인덱스에 접근이 가능한걸까? 그리고 또, `array`를 출력하면 다음과 같은 결과물이 출력된다.

```js
console.log(arr); // [empty x 3, 5]
```

일반적으로 알고 있는 C언어의 경우에는 정적으로 배열의 크기를 선언해주어야 하고, 자바에서 `ArrayList` 같은 컬렉션 클래스는 내부적으로 배열을 쓰면서, 일정 크기가 차면 자동으로 배열 크기를 늘리도록 동작한다.

그렇다면 자바스크립트는 어떻게 동작할까?

## 2. 자바스크립트의 배열

[ECMAScript® 2026 Language Specification의 LengthOfArrayLike](https://tc39.es/ecma262/#sec-lengthofarraylike)를 확인해보면 자바스크립트에서 배열은 객체와 같이 동작한다. 즉, 배열에서 인덱스는 프로퍼티 키처럼 동작하며 프로퍼티는 "가장 큰 인덱스 + 1"로 자동 관리된다.

실제로 확인해보자.

```js
const arr = [];
console.log(arr.length); // 0
arr[3] = 5;
console.log(arr.length); // 4 → 0 ~ 3 인덱스 기준
```

분명 나는 배열의 크기를 할당해주지 않았는데, 3번째 인덱스에만 값이 정상적으로 할당되었고 배열의 길이는 4로 변경되었다. 이는 자바스크립트가 다른 언어처럼 고정된 메모리 공간을 미리 할당하는 방식이 아닌 자동으로 확장되는 방식을 취하고 있기 때문이다.

### 2-1. 배열의 최대 크기

자바스크립트 배열의 최대 길이는 플랫폼, 브라우저에따라 다를 수 있지만, MDN 문서에 따르면 배열의 최대 길이는 2^32 -1을 따른다고 한다.

![Array Maximum Length](assets/img/writing/19/array_maximum_length.png){: width="640" }

에러 메시지는 브라우저마다 다르게 나타날 수 있는데 모두 `RangeError` 객체를 반환하며 에러 메시지는 다음과 같다.

```
RangeError: invalid array length (V8-based & Firefox)
RangeError: Array size is not a small enough positive integer. (Safari)

RangeError: Invalid array buffer length (V8-based)
RangeError: length too large (Safari)
```

### 2-2. Empty

```js
const arr = [];
arr[3] = 5;
console.log(arr); // [empty x 3, 5]
```

우리가 일반적으로 알고 있는 빈 값은 `null`과 `undefined`가 있는데, empty는 뭘까?

자바스크립트 배열에서 empty는 해당 인덱스가 명시적으로 값이 할당되지 않은 상태를 의미한다. 위 `arr`에서 0 ~ 2 인덱스는 값이 아예 존재하지 않는 empty slot인데 표로 비교해보자.

| 구분 | 설명 |
| --- | --- |
| **null** | **명시적으로 “없음”을 표시하는 값** → 의도적으로 비워둠 |
| **undefined** | **정의되지 않음** → 선언만 했거나, 명시적으로 `undefined` 할당 |
| **empty (slot)** | 배열에서 **값 자체가 존재하지 않는 자리** (구멍) |

아래 코드를 실행해보면 empty가 더욱 와닿을 수 있다.

```js
const array = [];
array[3] = 10;

console.log(Object.keys(array)); // ['3']
console.log(0 in array); // false
console.log(array.hasOwnProperty(0)); // false
console.log(array[0]); // undefined (접근하면 undefined처럼 보이지만 실제 값은 없음)
```

empty는 메모리상 슬롯 공간을 할당받지 않았으며, 어떠한 값도 존재하지 않기 때문에 empty를 나타내는 키가 존재하지 않고, 0번째 인덱스, 프로퍼티가 존재하지 않는 것을 확인할 수 있다.

여기서 재미있는 점은 순회에서 차이가 발생하는 것인데, 반복문과 배열 메서드는 empty slot을 다루는 방식에서 큰 차이가 있다.

ECMAScript를 기준으로 [for … in](https://tc39.es/ecma262/#sec-for-in-iterator-objects)은 Key를 기반으로 순회하고, `forEach`, `map`, `filter`은 hasProperty를 통해 property가 존재하는 영역에 대해서만 순회합니다.

![forEach Operation Process](assets/img/writing/19/forEach_operation_code.png){: width="640" }
_출처: [https://tc39.es/ecma262/#sec-array.prototype.foreach](https://tc39.es/ecma262/#sec-array.prototype.foreach)_

위 이미지는 `array.prototype.forEach`의 동작 과정을 나타낸 수도 코드입니다. 이 내부에서는 `O` 객체에 `P`라는 프로퍼티 키가 존재하는지를 확인하는 `HasProperty(O, Pk)` 구문이 실행됩니다. 이 과정 덕분에 `forEach`, `map`, `filter` 같은 메서드는 배열을 순회할 때 **값이 할당된 인덱스만 대상으로 하고, 빈 슬롯(empty slot)은 건너뛰게 됩니다**.

```js
const array = [];
array[2] = 2;
array[5] = 5;

for (const number in array) {
  console.log(number);
}
// 2, 5

array.forEach((number) => console.info(number)); // 2, 5
```

다만 length를 기준으로 하는 `for`문이나 `for ... of`문은 배열 내부의 길이만큼 반복하기 때문에 다음과 같은 값이 출력됩니다.

```js
const arr = [];
arr[3] = 5;

for (let i = 0; i < arr.length; i++) {
  console.log(arr[i]);
}
// undefined, undefined, undefined, 5

for (const num of arr) {
  console.log(num);
}
// undefined, undefined, undefined, 5
```

## 3. 결론

자바스크립트의 배열은 고정 크기가 아닌, 객체처럼 동작하는 유연한 구조로 설계되어 있습니다. 배열의 인덱스는 프로퍼티 키로 관리되며, 값이 없는 인덱스는 `empty slot`으로 처리됩니다. 이로 인해 다른 언어에서는 보기 힘든 독특한 동작 방식이 나타나죠.

특히 반복문과 배열 메서드에서 empty slot을 다루는 방식이 다르다는 점은 실무에서 놓치기 쉽습니다. 배열 내부를 초기화하지 않고 특정 인덱스에만 값을 저장하는 **희소 배열(sparse array)** 방식으로 사용하면, 데이터가 의도치 않게 누락되거나 메모리 사용량이 비효율적으로 증가할 수 있습니다.

따라서 실무에서는 `Array(n).fill()`이나 `Array.from()` 등을 사용해 배열을 초기화한 후 사용하는 것을 권장합니다.

- `for...of` → 배열 길이(length)만큼 순회, 빈 슬롯은 `undefined`로 처리
- `for...in`, `forEach`, `map`, `filter` → 값이 존재하는 인덱스만 순회

이 동작 차이를 잘 이해하면, 더 안전하고 예측 가능한 코드를 작성할 수 있을 것입니다.
