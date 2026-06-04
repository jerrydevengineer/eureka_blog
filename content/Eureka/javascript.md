---
title: "자바스크립트 기본 문법: var, let, const 그리고 화살표 함수"
tags:
  - 멀티캠퍼스부트캠프
  - 부트캠프
  - 유레카4기
description: 자바스크립트의 변수 선언 방식(var, let, const)의 차이점과 모던 JS의 핵심인 화살표 함수(Arrow Function)에 대해 알아봅니다.
date: 2026-06-04
---
자바스크립트(JavaScript)는 과거 웹 브라우저에 생동감을 불어넣는 스크립트 언어로 시작했지만, 현재는 Node.js 등을 통해 백엔드까지 영역을 확장한 필수 언어입니다. 오늘은 수업에서 배운 기초 문법인 **변수 선언 방식**과 **화살표 함수**에 대해 정리해 보겠습니다.

---
## 1. 변수 선언: var, let, const의 차이

과거 자바스크립트에서는 변수를 선언할 때 오직 `var`만을 사용했습니다. 하지만 여러 가지 구조적 문제(호이스팅, 스코프 등)로 인해 ES6(2015년)부터 `let`과 `const`가 등장했습니다.
### ① var: 과거의 유물 (사용을 지양하자)
`var`는 함수 레벨 스코프(Function-scoped)를 가집니다. 즉, 함수 안에서 선언된 변수만 지역 변수로 인정하고, `if`문이나 `for`문 같은 블록 안에서 선언해도 전역 변수처럼 취급되는 치명적인 단점이 있습니다.

```javascript
if (true) { var a = 10; } console.log(a); // 10 (블록 밖에서도 접근 가능. 버그의 주범이 됩니다.)
```

### ② let: 재할당이 가능한 변수

자바의 일반적인 변수 선언과 가장 비슷합니다. 블록 레벨 스코프(Block-scoped)를 가지며, 선언한 변수의 값을 나중에 변경할 수 있습니다.

``` JavaScript
let count = 1;
count = 2; // 재할당 가능
console.log(count); // 2
```

### ③ const: 재할당이 불가능한 상수

한 번 선언하고 값을 할당하면, 다른 값으로 **재할당할 수 없는 상수**입니다. `let`과 마찬가지로 블록 레벨 스코프를 가집니다.

``` JavaScript
const pi = 3.14;
pi = 3.14159; // TypeError: Assignment to constant variable.
```


**💡 실무 적용 꿀팁**
기본적으로 모든 변수는 **`const`** 로 선언하세요. 코드를 짜다가 "아, 이 값은 반복문 등에서 계속 변해야 하네?"라는 생각이 들 때만 **`let`** 으로 바꾸는 것이 가장 안전한 코딩 습관입니다. (`var`는 이제 쓰지 않습니다!)

---

## 2. 화살표 함수 (Arrow Function)

자바(Java)에 람다식(Lambda)이 있다면, 자바스크립트에는 화살표 함수(Arrow Function)가 있습니다. 기존의 `function` 키워드를 생략하고 `=>` (화살표)를 사용해 함수를 매우 간결하게 작성할 수 있습니다.

### 기존 함수형태 vs 화살표 함수

**기존 방식 (함수 선언문/표현식)**

``` JavaScript
const add = function(a, b) {
  return a + b;
};
```

**화살표 함수 방식**

``` JavaScript
const add = (a, b) => {
  return a + b;
};
```

### 화살표 함수의 강력한 생략 기능

화살표 함수는 코드가 한 줄이고, 그 한 줄이 곧 반환(return) 값이라면 중괄호 `{}`와 `return` 키워드마저 생략할 수 있습니다.

``` JavaScript
// 중괄호와 return 생략 (가장 많이 쓰이는 형태)
const multiply = (a, b) => a * b;

// 매개변수가 딱 1개라면 괄호()도 생략 가능합니다.
const square = x => x * x;
```

### 배열 메서드와의 찰떡궁합

화살표 함수는 `map`, `filter` 같은 배열의 고차 함수와 함께 쓸 때 진가를 발휘합니다.

``` JavaScript
const numbers = [1, 2, 3, 4, 5];

// 짝수만 걸러내서 2배로 만들기 (화살표 함수 적용)
const evenDoubled = numbers
  .filter(n => n % 2 === 0)
  .map(n => n * 2);

console.log(evenDoubled); // [4, 8]
```