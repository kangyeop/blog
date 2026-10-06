---
title: "React는 무엇일까"
description: "React가 무엇인지, HTML·CSS·JS만 쓸 때와 무엇이 다른지, Vue·Angular와는 무엇이 다른지 정리한 내용."
date: 2026-10-06
categories: [학습정리]
tags: [Frontend, Dev-Basics, React]
---

## React

React는 Meta(당시 Facebook)가 만든 자바스크립트 라이브러리다. Facebook 내부에서 쓰이다가 2013년에 오픈소스로 공개됐다. 공식 사이트는 React를 "The library for web and native user interfaces"라고 소개한다.

"UI 라이브러리"라고 하면 MUI나 shadcn/ui처럼 버튼, 모달 같은 완성된 컴포넌트를 모아둔 라이브러리가 먼저 떠오른다. React는 그런 컴포넌트를 주지 않는다. 컴포넌트를 만들고, 상태가 바뀌면 화면을 다시 그리는 구조를 준다. MUI나 shadcn/ui도 React로 만든 컴포넌트 모음이다.

라이브러리와 프레임워크를 나누는 기준으로 **제어의 역전**(inversion of control)이 자주 쓰인다. 내 코드가 라이브러리를 호출하면 라이브러리, 프레임워크가 내 코드를 호출하면 프레임워크다. lodash의 `debounce`는 내가 필요할 때 호출하니 라이브러리다. 이 기준으로 보면 React는 프레임워크에 가깝다. 나는 컴포넌트 함수를 정의만 하고, 그 함수를 언제 호출할지, Effect를 언제 실행할지는 React가 정한다. 훅 규칙처럼 React가 정한 규칙도 지켜야 한다.

React가 스스로를 라이브러리라고 부르는 건 맡는 범위를 기준으로 했을 때다. 라우팅, 데이터 요청, 빌드 설정, 폴더 구조는 React가 정해두지 않는다. 필요하면 React Router, TanStack Query, Vite 같은 도구를 조합하거나, 이런 기능을 묶어둔 Next.js 같은 프레임워크를 쓴다. 공식 문서도 새 앱을 만들 때는 프레임워크로 시작하라고 안내한다.

다만 React가 맡는 범위도 계속 넓어지고 있다. React 18의 `startTransition`처럼 렌더링 작업의 우선순위를 정하는 스케줄러가 들어갔고, React 19에는 `<form action>`, `useActionState` 같은 폼 처리와 서버에서 실행되는 Server Components, Server Functions가 들어갔다. Server Components는 번들러와 서버가 함께 지원해야 해서 React만으로는 쓸 수 없고, Next.js 같은 프레임워크를 통해 쓴다. 화면만 그리는 작은 라이브러리로 보기에는 React가 하는 일이 많다.

React로 만드는 UI의 단위는 **컴포넌트**다. 컴포넌트는 화면 조각을 반환하는 함수이고, 반환값은 **JSX**로 쓴다.

```jsx
function Greeting({ name }) {
  return <h1 className="title">안녕, {name}</h1>;
}
```

JSX는 자바스크립트 안에 HTML과 비슷한 문법을 쓰게 해주는 확장 문법이다. 브라우저는 JSX를 모르기 때문에 빌드 단계에서 함수 호출로 바뀐다.

```js
import { jsxs as _jsxs } from 'react/jsx-runtime';

function Greeting({ name }) {
  return _jsxs('h1', { className: 'title', children: ['안녕, ', name] });
}
```

이 함수가 반환하는 건 DOM 노드가 아니다. `{ type: 'h1', props: { className: 'title', children: [...] } }` 형태의 일반 객체이고, 이걸 **React 엘리먼트**라고 부른다. React는 이 객체를 보고 실제 DOM을 만들거나 고친다.

## HTML, CSS, JS만 쓸 때와의 차이

React를 써도 브라우저가 받는 건 여전히 HTML, CSS, JS다. React는 DOM API로 화면을 고치고, CSS에는 관여하지 않는다. 차이는 DOM을 누가, 어떤 방식으로 바꾸느냐에 있다.

할 일을 추가하면 목록과 개수가 같이 바뀌는 화면을 자바스크립트만으로 만들면 이렇다.

```html
<input id="input" />
<button id="add">추가</button>
<ul id="list"></ul>
<p id="count">0개</p>
```

```js
const todos = [];
const input = document.querySelector('#input');
const list = document.querySelector('#list');
const count = document.querySelector('#count');

document.querySelector('#add').addEventListener('click', () => {
  todos.push(input.value);

  const li = document.createElement('li');
  li.textContent = input.value;
  list.appendChild(li);

  count.textContent = `${todos.length}개`;
  input.value = '';
});
```

데이터(`todos`)를 바꾸는 코드와 화면을 바꾸는 코드를 둘 다 직접 쓴다. 이렇게 "무엇을 어떤 순서로 바꿀지"를 하나하나 지시하는 방식을 **명령형**이라고 한다.

기능이 삭제, 완료 체크, 필터로 늘어나면 데이터가 바뀌는 곳마다 목록과 개수를 모두 갱신해야 한다. 한 곳이라도 빠뜨리면 데이터와 화면이 어긋난다. 데이터가 바뀔 때마다 `innerHTML`로 목록을 통째로 다시 만들면 코드는 단순해지지만, 매번 DOM을 새로 만들어서 입력 중이던 포커스나 스크롤 위치가 사라지고, 사용자 입력을 `innerHTML`에 넣으면 XSS 위험도 생긴다.

같은 화면을 React로 만들면 이렇다.

```jsx
function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [text, setText] = useState('');

  const add = () => {
    setTodos([...todos, { id: crypto.randomUUID(), text }]);
    setText('');
  };

  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={add}>추가</button>
      <ul>
        {todos.map((todo) => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
      <p>{todos.length}개</p>
    </>
  );
}
```

`add`는 상태만 바꾼다. 목록에 `<li>`를 붙이거나 개수를 고치는 코드는 없다. 화면은 "상태가 이러면 이렇게 보인다"로 한 번만 적어두고, 상태가 바뀌면 React가 컴포넌트를 다시 호출해서 이전 결과와 비교한 뒤 바뀐 DOM만 고친다. 이 방식을 **선언형**이라고 하고, 흔히 `UI = f(state)`로 표현한다.

화면이 커지면 컴포넌트를 나눈다.

```jsx
function TodoItem({ todo, onRemove }) {
  return (
    <li>
      {todo.text}
      <button onClick={() => onRemove(todo.id)}>삭제</button>
    </li>
  );
}
```

부모는 데이터를 props로 내려주고, 자식은 데이터를 직접 바꾸지 않고 `onRemove` 같은 콜백으로 부모에게 알린다. 데이터가 위에서 아래로만 흐르기 때문에 **단방향 데이터 흐름**이라고 부른다. 상태를 바꾸는 곳이 상태를 가진 컴포넌트 한 곳으로 모여서, 화면이 이상할 때 어디서 상태가 바뀌었는지 찾기 쉽다.

스타일은 React가 정하지 않는다. `className`에 일반 CSS 클래스를 쓰거나 CSS Modules, Tailwind CSS 같은 방식을 프로젝트마다 고른다.

## Vue, Angular와의 차이

Vue와 Angular도 선언형 UI와 컴포넌트를 쓴다. 셋은 프레임워크가 맡는 범위, UI를 쓰는 방식, 상태 변화를 감지하는 방식이 다르다.

| | React | Vue | Angular |
|---|---|---|---|
| 공개 | 2013, Meta | 2014, Evan You | 2016, Google (전신 AngularJS는 2010) |
| 범위 | 컴포넌트와 렌더링 | 코어 + 공식 라이브러리 | 라우터, 폼, HTTP, DI까지 포함 |
| UI 작성 | JSX | HTML 기반 템플릿 | HTML 기반 템플릿 + TypeScript 클래스 |
| 상태 변화 감지 | 컴포넌트 함수 재호출 | 반응형 값이 읽힌 곳을 추적 | Zone.js 기반 변경 감지, 최근 버전은 Signals |
| 가상 DOM | 사용 | 사용, 컴파일러가 비교할 범위를 줄임 | 사용하지 않음 |

### 프레임워크가 맡는 범위

Angular는 라우터, 폼, HTTP 클라이언트, 의존성 주입(DI), 테스트 도구, CLI까지 프레임워크 안에 들어 있고, 폴더 구조와 코드 작성 방식도 정해져 있다. 팀이 따로 고를 게 적어서 개발자가 많은 조직에서도 코드가 비슷한 모양으로 유지된다. 그래서 대기업 사내 시스템이나 금융권처럼 규모가 크고 오래 유지보수하는 업무용 앱에서 많이 쓰인다. TypeScript 클래스에 데코레이터와 DI를 쓰는 방식은 Spring, .NET과 비슷하다.

Vue는 코어 위에 Vue Router, Pinia 같은 공식 라이브러리를 필요한 만큼 붙이는 구조라서 스스로를 점진적 프레임워크라고 부른다. `<script>` 태그 하나로 기존 페이지의 일부에만 넣을 수도 있어서, Laravel처럼 서버가 HTML을 만드는 프로젝트에 붙여 쓰기 쉽다. Laravel은 한동안 프론트엔드 기본 구성으로 Vue를 넣어줬다. 중국에서도 많이 쓰이는데, Evan You가 중국계라서 중국어 문서와 커뮤니티가 일찍 갖춰졌다.

React는 컴포넌트와 렌더링만 맡고 나머지는 외부 라이브러리로 채운다. 프로젝트마다 고를 게 많지만, 같은 코어 위에 Next.js 같은 웹 프레임워크와 React Native 같은 모바일 런타임이 따로 만들어졌다.

### UI 작성 방식

Vue와 Angular는 HTML에 디렉티브를 더한 템플릿을 쓴다. 할 일 목록을 그리는 코드는 각각 이렇다.

```vue
<li v-for="todo in todos" :key="todo.id">{{ todo.text }}</li>
```

```html
@for (todo of todos; track todo.id) {
  <li>{{ todo.text }}</li>
}
```

HTML을 아는 사람은 바로 읽을 수 있지만 `v-for`, `@for` 같은 템플릿 문법을 따로 익혀야 한다. 템플릿은 TypeScript 파일이 아니라서, 템플릿 안에서 타입 검사를 받으려면 Vue는 Vue - Official 확장(Volar), Angular는 Angular Language Service 같은 별도 도구가 필요하다.

React의 JSX는 자바스크립트 표현식이다. 조건은 `&&`나 삼항 연산자로, 반복은 앞의 `TodoApp`처럼 `map`으로 쓴다. 컴포넌트와 커스텀 훅이 모두 함수라서 함수를 나누고 합치는 방식으로 UI를 재사용하고, TypeScript도 일반 코드와 같은 방식으로 적용된다.

### 상태 변화 감지

Vue의 반응형 값은 어디서 읽혔는지 기록해둔다. 값을 바꾸면 그 값을 읽은 컴포넌트만 다시 그리기 때문에 최적화를 따로 할 일이 적다. 다시 그릴 때는 React처럼 가상 DOM을 비교하지만, 템플릿 컴파일러가 바뀔 수 없는 노드를 비교 대상에서 빼고 동적인 노드에는 어떤 속성이 바뀔 수 있는지 표시해둬서 비교할 양이 적다.

Angular는 Zone.js가 이벤트나 비동기 작업이 끝나는 시점을 잡아서 컴포넌트 트리의 변경을 검사하는 방식이었다. 최근 버전은 Vue처럼 값 단위로 추적하는 Signals로 옮겨가고 있다. Angular는 가상 DOM을 쓰지 않는다. 컴파일러가 템플릿을 DOM을 만들고 갱신하는 코드로 미리 바꿔두고, 변경을 검사할 때는 트리를 비교하지 않고 바인딩된 값이 이전 값과 같은지만 확인해서 다르면 그 자리의 DOM을 고친다.

React는 값이 어디서 쓰였는지 추적하지 않는다. `setState`가 호출되면 컴포넌트 함수를 통째로 다시 호출하고, 기본적으로 그 아래 자식 컴포넌트도 다시 호출한다. 그렇게 만든 새 React 엘리먼트 트리를 이전 트리와 비교해서 바뀐 DOM을 찾는데, 이 엘리먼트 트리를 흔히 **가상 DOM**이라고 부른다. `UI = f(state)` 하나로 동작이 설명돼서 예측하기 쉽지만, 불필요한 재호출을 줄이려면 `memo`, `useMemo` 같은 도구를 써야 한다.

### 사용량

셋 중에서는 React가 가장 많이 쓰인다. 2026년 9월 20일부터 26일까지 npm 주간 다운로드 수는 이랬다.

| 패키지 | 주간 다운로드 |
|---|---|
| `react` | 약 2억 350만 |
| `vue` | 약 1,846만 |
| `@angular/core` | 약 729만 |

다운로드 수에는 CI의 반복 설치나 다른 패키지의 의존성으로 설치된 경우도 섞여 있어서 사용자 수와 같지는 않다.

React가 많이 쓰이는 이유로는 이런 것들이 꼽힌다.

- Facebook, Instagram 같은 대규모 서비스에서 먼저 쓰였다.
- 맡는 범위가 작아서 다른 도구와 조합하기 쉽고, 그 위에 Next.js와 React Native가 만들어져 웹과 모바일에 같은 방식을 쓸 수 있다.
- JSX가 자바스크립트라서 템플릿 문법을 따로 배우지 않아도 되고, TypeScript와 잘 맞는다.
- 사용자가 많아서 라이브러리, 자료, 채용 공고가 많고, 이게 다시 React를 고르는 이유가 된다.
