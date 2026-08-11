---
title: "AI가 만든 컴포넌트는 왜 전부 function일까"
description: "컴포넌트를 화살표 함수로 쓰는 걸 관행처럼 따라 쓰고 있었다. 그게 어디서 온 건지 확인해본 내용."
date: 2026-07-31
categories: [학습정리]
tags: [Frontend, Convention, AI, ESLint, React]
---

[지난 글]({% post_url 2026-07-30-fsd-vibe-coding %})에서 FSD를 지키게 만드는 방법을 정리했다. 파일이 어디에 놓이는지는 그렇게 어느 정도 정리가 됐다.

다음은 파일 안이었다. AI로 빠르게 만들다 보니 컴포넌트 대부분이 `function Component() {}` 형태로 쌓여 있었다. 나는 `const Component = () => {}`를 훨씬 많이 썼고, 예전 팀에서도 그게 자연스러웠다.

그런데 시키지도 않았는데 계속 `function`으로 나온다. 고치라고 하기 전에, 내가 화살표를 쓰던 이유부터 다시 확인해보기로 했다.

## 화살표 함수가 나온 이유

ES5에서는 콜백 안에서 `this`가 사라졌다.

```js
function Timer() {
  var self = this;          // 또는 .bind(this)
  setInterval(function () {
    self.tick++;
  }, 1000);
}
```

`var self = this`와 `.bind(this)`를 반복해서 써야 했다. 이 패턴이 너무 흔해서, 언어 차원에서 "자기 `this`를 만들지 않는 함수"를 추가한 게 화살표 함수다.

`function`의 범용 대체품으로 나온 문법이 아니다. 콜백에서 반복되던 `this` 문제를 없애려고 추가된 것이고, 콜백이 많으니 쓰이는 빈도가 높아졌다.

React 클래스 컴포넌트에서도 마찬가지였다. `constructor`에서 `this.handleClick = this.handleClick.bind(this)`를 반복하는 대신 `handleClick = () => {}`로 쓰면 됐다.

## 할 수 있는 게 적다는 것

화살표 함수는 `function`보다 할 수 있는 게 적다. 자기 `this`가 없고, `arguments`가 없고, `prototype`이 없고, `new`로 호출할 수도 없다.

보통 기능이 적은 건 단점인데 여기서는 반대다. `n => n * 2`를 보면 본문을 안 읽어도 이게 생성자가 아니고 `this`를 건드리지 않는다는 걸 안다. `function`은 그 가능성이 전부 열려 있어서 안을 봐야 안다.

`var`를 `const`로 바꾸는 것과 같은 논리다. 능력이 제일 적은 걸 기본값으로 두고, 필요할 때만 강한 쪽을 꺼낸다.

## 함수를 값으로 다루기

`function Component() {}`는 선언문이고, `const Component = () => {}`는 함수를 값으로 만들어 변수에 담는 표현식이다. 나머지 차이는 대부분 여기서 나온다.

표현식은 값이라 그대로 다른 함수에 넘길 수 있다. `const Component = memo((props) => {...})`처럼 통째로 감싸면 끝이다. 선언문은 이름 붙여 정의한 뒤 별도 줄에서 감싸야 해서 한 단계가 더 늘어난다.

`const`에 담기니까 재할당도 막히고, 변수든 함수든 `const 이름 = 값` 하나의 형태로 통일된다.

## 컴포넌트에는 해당이 없다

여기까지가 화살표를 쓰는 이유다. 컴포넌트에 하나씩 대보면 남는 게 없다.

함수 컴포넌트에는 `this`가 없어서, 화살표가 없애주는 문제 자체가 생기지 않는다.

간결함도 아니다. 콜백이라면 `function (x) { return x * 2; }`가 `x => x * 2`로 줄지만, 최상위 선언에서는 `const Component = () => {}`가 `function Component() {}`보다 길다.

`this`나 `arguments`를 쓸 일이 없으니 최소 권한도 마찬가지다. 쓰지 않는 기능이 막히는 것뿐이다.

TypeScript를 쓰면서는 `React.FC`가 이유가 되기도 했다. `const Component: React.FC<Props> = () => {}`처럼 함수 전체에 타입을 붙이려면 값으로 만들어 변수에 담아야 한다. 다만 2020년에 create-react-app이 TypeScript 템플릿에서 이걸 걷어냈다. props에 `children`이 자동으로 붙는 게 문제였고, 그 뒤로는 타입을 인자에 바로 붙이는 쪽이 자연스러워졌다. 그건 함수 선언문에서도 똑같이 된다.

남는 건 `memo`나 `forwardRef`로 감쌀 때 한 줄 덜 쓰는 것 정도였다.

## `function` 쪽에도 근거가 있다

반대쪽도 확인해봤다.

선언문은 호이스팅된다. 파일 아래쪽에 정의해도 위에서 먼저 부를 수 있다. 컴포넌트에서는 렌더 시점에 본문이 실행되니 동작에는 차이가 없지만, 파일을 어떻게 배치하느냐가 달라진다.

`function`이면 `export default function Page()`를 파일 맨 위에 두고 서브 컴포넌트와 헬퍼를 아래로 내릴 수 있다. 파일을 위에서 아래로 읽으면 추상화 레벨이 내려간다. 화살표는 사용 전에 정의해야 해서 반대가 된다. 파일 첫 화면이 세부 구현부터 시작한다.

공식 문서도 이 형태다. react.dev의 컴포넌트 예제는 `function` 선언이고, Next.js 문서도 `export default function Page()`다. `react/function-component-definition` 룰의 기본값마저 `function-declaration`이다. 내가 화살표를 강제하려고 켠 그 룰이, 아무 설정 없이 켜면 `function`을 강제한다.

다만 어느 쪽도 이유는 안 적어뒀다. react.dev에는 선언 스타일에 대한 가이드 자체가 없고, 룰 문서에도 왜 그 기본값인지는 없다. 위 배치 얘기는 일반적인 근거일 뿐이지 React가 그걸 근거로 골랐다는 뜻은 아니다.

그런데 실제 코드는 또 다르다. React와 TypeScript를 같이 검색했을 때 닿는 레퍼런스들은 화살표를 쓰고, 화살표로 통일해둔 오픈소스 프로젝트도 많다. 다만 그런 곳들의 컨벤션 문서를 열어봐도 "화살표를 쓴다"고만 적혀 있다.

ESLint의 `func-style` 문서는 아예 이렇게 적어뒀다.

> 여기에 옳고 그른 선택은 없다. 그냥 취향이다.

## 그래서 화살표로 맞췄다

정리하면 이렇다. 콜백과 중첩 함수에서는 화살표를 쓸 이유가 분명하다. `this`나 `new`, `arguments`가 필요하면 `function`을 써야 한다. 모듈 최상위의 이름 붙은 함수는 어느 쪽도 결정적이지 않고, 컴포넌트가 여기에 해당한다.

그럼 남는 기준은 우리 코드베이스 안쪽뿐이었다.

커스텀 훅, 이벤트 핸들러, 유틸 함수는 이미 다 화살표 함수로 쓰고 있었다. 컴포넌트만 `function`이면 같은 파일 안에서 선언 스타일이 두 가지가 된다.

파일 전체가 "값으로 다루는 함수들"이라는 하나의 모델을 갖는 게, 내가 읽을 때도 AI가 옆 코드를 참조할 때도 더 일관됐다.

대신 감수하는 것도 있다. 파일 배치는 세부 구현이 위로 올라오는 쪽을 받아들였고, 제네릭 컴포넌트를 쓸 때 `<T,>`처럼 콤마를 넣어야 JSX 태그로 안 읽히는 것도 그냥 두기로 했다. 둘 다 파일 안에서 스타일을 둘로 나눌 만큼은 아니었다.

다만 이건 "화살표가 옳다"는 결론은 아니다. "우리는 화살표로 맞춘다"는 결정이다.

---

### 참고

- [react/function-component-definition — eslint-plugin-react](https://github.com/jsx-eslint/eslint-plugin-react/blob/master/docs/rules/function-component-definition.md)
- [func-style — ESLint](https://eslint.org/docs/latest/rules/func-style)
- [8.1 Arrow Functions — Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript#arrows--use-them)
- [Arrow function expressions — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)
- [Remove React.FC from Typescript template — create-react-app #8177](https://github.com/facebook/create-react-app/pull/8177)
- [Your First Component — react.dev](https://react.dev/learn/your-first-component)
- [Layouts and Pages — Next.js](https://nextjs.org/docs/app/getting-started/layouts-and-pages)

