---
title: "React는 어떻게 동작할까"
description: "React의 렌더와 커밋, 재조정, 컴포넌트 라이프사이클, 상태와 훅, 리렌더링이 동작하는 방식을 정리한 내용."
date: 2026-10-13
categories: [학습정리]
tags: [Frontend, Dev-Basics, React]
---

## 렌더와 커밋

React 공식 문서는 화면이 바뀌는 과정을 렌더 트리거, 렌더, 커밋 세 단계로 설명한다. 이 중 렌더와 커밋은 React 내부에서도 나뉘는 단계이고, 트리거는 렌더가 시작되는 계기를 가리킨다.

1. **트리거**: 렌더가 필요한 상황이 생긴다. 앱이 처음 뜰 때(`createRoot(...).render(...)`)와 어떤 컴포넌트의 상태가 바뀔 때(`setState`)다.
2. **렌더**: React가 컴포넌트 함수를 호출한다. 처음에는 루트 컴포넌트부터, 이후에는 상태가 바뀐 컴포넌트부터 그 아래로 내려가며 호출해서 새 React 엘리먼트 트리를 만들고, 이전 트리와 비교해서 바뀐 부분을 계산한다.
3. **커밋**: 렌더 단계에서 계산한 바뀐 부분만 실제 DOM에 반영한다. 처음에는 만든 DOM 노드를 전부 붙인다.

커밋이 끝나면 브라우저가 바뀐 DOM으로 화면을 다시 그린다(페인트).

React에서 말하는 렌더링은 컴포넌트 함수를 호출하는 일이다. 브라우저의 페인트와는 다르다. 컴포넌트가 다시 렌더링돼도 결과가 이전과 같으면 커밋 단계에서 바뀌는 DOM은 없다.

렌더 단계는 **순수**해야 한다. 같은 props와 state가 들어오면 같은 JSX를 반환해야 하고, 렌더 중에 컴포넌트 밖의 변수나 DOM을 바꾸면 안 된다. React는 렌더를 여러 번 호출하거나, 렌더하던 결과를 커밋하지 않고 버릴 수도 있기 때문이다. 데이터 요청이나 구독 같은 부수 효과는 이벤트 핸들러나 Effect(`useEffect`)에 둔다.

## 재조정

렌더 단계에서 만든 새 트리와 이전 트리를 비교해서 무엇을 바꿀지 정하는 과정을 **재조정**(reconciliation)이라고 한다. 이 엘리먼트 트리를 흔히 **가상 DOM**이라고 부른다.

트리 두 개의 차이를 정확하게 구하는 일반적인 알고리즘은 O(n³)이 걸린다. 노드가 1,000개면 10억 번 비교해야 해서 화면을 바꿀 때마다 쓸 수 없다. React는 두 가지 가정을 두고 O(n)으로 비교한다.

**타입이 다르면 다른 트리로 본다.** 같은 위치의 엘리먼트 타입이 바뀌면 React는 이전 노드와 그 아래를 전부 버리고 새로 만든다.

```jsx
// 이전
<div><Counter /></div>

// 이후
<section><Counter /></section>
```

`Counter` 자체는 그대로지만 부모가 `div`에서 `section`으로 바뀌었기 때문에 `Counter`도 새로 만들어지고, 안에 있던 상태도 초기화된다.

**형제 사이에서는 key로 같은 항목을 찾는다.** 목록을 비교할 때 key가 없으면 React는 순서대로 짝을 맞춘다. 맨 앞에 항목을 하나 추가하면 모든 항목이 한 칸씩 밀려서 전부 바뀐 것으로 판단한다. key가 있으면 key가 같은 항목끼리 짝을 맞춰서 새 항목만 추가한다.

배열 인덱스를 key로 쓰면 key가 없을 때와 같은 문제가 생긴다.

```jsx
items.map((item, i) => <Row key={i} item={item} />)

// 이전: [A, B]     → key 0 = A, key 1 = B
// 이후: [C, A, B]  → key 0 = C, key 1 = A, key 2 = B
```

React는 key 0을 이전의 key 0과 같은 컴포넌트로 본다. 그래서 A를 그리던 컴포넌트를 그대로 두고 props만 C로 바꾼다. key 1도 B를 그리던 컴포넌트에 A를 넘기고, key 2만 새로 만든다. 항목은 하나 추가됐지만 기존 행의 props가 전부 바뀌어서 모든 행의 DOM이 갱신된다. A 행 컴포넌트가 가진 상태(`useState` 값, 비제어 `<input>`의 입력값, 체크 여부)는 key 0 컴포넌트에 남아 있어서 C 행에 표시된다.

`key={item.id}`로 쓰면 A와 B는 이전 컴포넌트와 짝이 맞고 C만 새로 만들어진다. 순서가 바뀌어도 같은 key의 컴포넌트는 상태와 DOM 노드를 유지한 채 위치만 옮겨진다. 이때 컴포넌트 함수 호출까지 건너뛰는 건 아니다. key가 같아도 부모가 다시 렌더링되면 자식도 다시 호출된다.

key는 데이터의 id처럼 항목이 바뀌어도 유지되는 값을 쓴다. 목록이 정렬·삽입·삭제 없이 고정이거나 끝에만 항목을 추가하는 경우에는 인덱스와 항목의 대응이 바뀌지 않아서 인덱스를 key로 써도 된다.

key는 상태를 일부러 초기화할 때도 쓴다. `<Profile key={userId} />`처럼 쓰면 `userId`가 바뀔 때 `Profile`이 새로 만들어진다.

React 16(2017년)부터는 재조정을 **Fiber**라는 구조로 한다. 이전 방식은 트리를 재귀로 끝까지 비교해서, 한 번 시작하면 중간에 멈출 수 없었다. 트리가 크면 그동안 메인 스레드가 막혀 입력이나 애니메이션이 끊겼다.

Fiber에서는 컴포넌트 하나하나가 fiber라는 작업 단위가 된다. React는 작업을 단위별로 처리하다가 중간에 멈추고 브라우저에 제어권을 넘길 수 있고, 사용자 입력처럼 급한 업데이트를 먼저 처리할 수 있다. React 18의 `startTransition`, `useDeferredValue`가 이 구조 위에서 동작한다. 멈출 수 있는 건 렌더 단계뿐이고, 커밋 단계는 한 번에 끝난다. 렌더 단계가 순수해야 하는 이유도 여기서 나온다. 중간에 멈췄다가 버리거나 다시 시작할 수 있는 단계라서 부수 효과가 있으면 안 된다.

## 컴포넌트 라이프사이클

컴포넌트는 처음 화면에 추가되는 **마운트**, props나 state가 바뀌어 다시 렌더링되는 **업데이트**, 화면에서 제거되는 **언마운트**를 거친다.

React 16.8(2019년)에 훅이 나오기 전에는 클래스 컴포넌트의 `componentDidMount`, `componentDidUpdate`, `componentWillUnmount` 메서드로 시점마다 코드를 실행했다. 함수 컴포넌트에서는 `useEffect`를 쓴다.

`useEffect`는 보통 의존성 배열로 실행 시점을 정하는 훅으로 설명된다. 배열이 비어 있으면 마운트 때만, 배열이 없으면 렌더링될 때마다, 값을 넣으면 그 값이 바뀔 때 실행된다.

```jsx
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  return <h1>{roomId} 채팅방</h1>;
}
```

`roomId`가 `'a'`인 상태로 마운트되면 `a`에 연결한다. `roomId`가 `'b'`로 바뀌면 이전 Effect의 cleanup으로 `a` 연결을 끊고, 새 Effect로 `b`에 연결한다. 언마운트되면 cleanup으로 `b` 연결을 끊는다. 의존성 배열의 값이 바뀔 때마다 "정리하고 다시 연결"을 반복하는 구조라서, 클래스 컴포넌트처럼 마운트와 업데이트를 나눠 쓸 필요가 없다.

값이 바뀔 때 다른 state를 갱신하는 데 `useEffect`를 쓰는 경우도 많다. 공식 문서의 "You Might Not Need an Effect"는 이런 경우 렌더 중에 값을 계산하거나 이벤트 핸들러에서 처리하라고 안내한다.

Effect가 실행되는 시점은 훅마다 다르다.

1. 렌더
2. 커밋 (DOM 변경)
3. `useLayoutEffect` 실행
4. 브라우저 레이아웃·페인트
5. `useEffect` 실행

`useEffect`는 보통 화면이 그려진 뒤에 실행돼서 페인트를 막지 않는다. `useLayoutEffect`는 커밋 직후, 브라우저가 화면을 그리기 전에 동기로 실행된다. 브라우저는 자바스크립트가 실행되는 동안 화면을 그리지 못해서, 이 안에서 DOM 크기를 재고 state를 바꾸면 그 결과까지 반영된 화면이 처음 그려진다.

툴팁 위치처럼 DOM 크기를 재서 바로 반영해야 하는 경우에 `useLayoutEffect`를 쓴다. `useEffect`로 하면 잘못된 위치로 한 번 그려졌다가 옮겨지는 깜빡임이 생긴다. 다만 `useLayoutEffect`는 끝날 때까지 페인트를 막기 때문에, 이런 경우가 아니면 `useEffect`를 쓴다.

개발 모드에서 `StrictMode`를 켜면 컴포넌트가 마운트된 직후 Effect를 setup → cleanup → setup 순서로 한 번 더 실행한다. 언마운트됐다가 다시 마운트되는 상황을 흉내 내는 것이고, state와 DOM은 그대로 유지된다. cleanup이 제대로 있으면 결과는 한 번 실행했을 때와 같고, cleanup을 빠뜨렸다면 연결이 두 개 생기거나 이벤트 핸들러가 두 번 실행되는 식으로 문제가 드러난다. 같은 이유로 컴포넌트 함수도 두 번 호출해서 렌더가 순수하지 않은 코드를 찾는다. 프로덕션 빌드에서는 한 번만 실행된다.

## 상태와 훅

`useState`는 현재 상태와 상태를 바꾸는 함수를 반환한다. 상태를 바꾸는 함수는 값을 바로 바꾸지 않고 다음 렌더를 예약한다. 한 번의 렌더 안에서 state는 고정된 값이다.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
  };

  return <button onClick={handleClick}>{count}</button>;
}
```

버튼을 한 번 누르면 3이 아니라 1이 된다. 이 렌더에서 `count`는 0이라서 `setCount(0 + 1)`을 세 번 호출한 것과 같다. 이전 값을 기준으로 바꾸려면 `setCount((c) => c + 1)`처럼 함수를 넘긴다. React는 넘긴 함수들을 순서대로 실행해서 3을 만든다.

`setCount`를 세 번 호출해도 렌더는 한 번만 일어난다. React는 이벤트 핸들러 안의 상태 업데이트를 모았다가 한 번에 처리하는데, 이걸 **배치**라고 한다. React 17까지는 이벤트 핸들러 안에서만 배치가 됐고, React 18부터는 `setTimeout`, Promise 안의 업데이트도 자동으로 배치된다.

함수 컴포넌트는 렌더할 때마다 새로 호출되는 함수라서 지역 변수로는 값을 유지할 수 없다. 상태는 React가 컴포넌트의 fiber에 저장해둔다. 이때 훅마다 이름을 붙이지 않고 **호출 순서**대로 저장한다. 컴포넌트에서 첫 번째로 호출된 `useState`는 첫 번째 칸, 두 번째 `useState`는 두 번째 칸을 읽는다.

```jsx
function Form() {
  const [name, setName] = useState('');

  if (name) {
    useEffect(() => {
      localStorage.setItem('name', name);
    });
  }

  const [age, setAge] = useState(0);
  // ...
}
```

`name`이 빈 문자열인 첫 렌더에서는 훅이 두 개 호출된다. `name`이 채워진 다음 렌더에서는 `useEffect`가 끼어들어 세 개가 된다. 이전 렌더에서 `age`가 쓰던 두 번째 칸을 `useEffect`가 읽고, `age`는 세 번째 칸을 찾는다. React는 이 경우 이전 렌더보다 훅이 많이 호출됐다는 에러를 낸다.

그래서 훅에는 규칙이 있다.

- 컴포넌트나 커스텀 훅의 최상위에서만 호출한다. 조건문, 반복문, 중첩 함수 안에서 호출하지 않는다.
- React 함수 컴포넌트나 커스텀 훅 안에서만 호출한다.

`eslint-plugin-react-hooks`가 이 규칙을 검사한다. React 19에 추가된 `use`는 예외로 조건문 안에서 호출할 수 있다.

## 리렌더링

컴포넌트가 다시 렌더링되는 경우는 세 가지다.

- 자기 state가 바뀌었을 때
- 부모 컴포넌트가 다시 렌더링됐을 때
- 구독한 Context의 값이 바뀌었을 때

props가 바뀌어서 다시 렌더링된다고 설명하는 경우가 많은데, props는 부모가 다시 렌더링돼야 바뀐다. 부모가 다시 렌더링되면 자식은 props가 그대로여도 다시 렌더링된다. state를 이전과 같은 값(`Object.is`로 비교)으로 바꾸면 React는 그 컴포넌트와 자식의 렌더를 건너뛴다. 다만 공식 문서는 경우에 따라 컴포넌트 함수를 한 번 호출한 뒤에 자식을 건너뛸 수도 있다고 적어두고 있다.

### 리렌더링이 문제가 되는 경우

다시 렌더링되는 것 자체는 React가 화면을 바꾸는 방식이라 문제가 아니다. 문제는 다시 렌더링될 때 함께 일어나는 일에서 생긴다.

**느려진다.** 렌더는 메인 스레드에서 컴포넌트 함수를 호출하는 일이라, DOM이 바뀌지 않아도 비용이 든다. 렌더 비용이 큰 컴포넌트가 자주 다시 렌더링되면 입력이 끊긴다.

```jsx
function Page({ rows }) {
  const [query, setQuery] = useState('');
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <HugeTable rows={rows} />
    </>
  );
}
```

글자를 하나 입력할 때마다 `Page`가 다시 렌더링되고, `query`와 관계없는 `HugeTable`도 함께 다시 렌더링된다.

**Effect가 다시 실행된다.** 렌더할 때마다 새로 만들어지는 객체나 함수를 의존성 배열에 넣으면, 다시 렌더링될 때마다 Effect가 다시 실행된다.

```jsx
function ChatRoom({ roomId }) {
  const options = { roomId };
  useEffect(() => {
    const connection = createConnection(options);
    connection.connect();
    return () => connection.disconnect();
  }, [options]);
  // ...
}
```

`options`는 렌더마다 새 객체라서 `roomId`가 그대로여도 연결을 끊었다가 다시 맺는다. `options`를 Effect 안에서 만들고 의존성 배열에는 `roomId`를 넣으면 `roomId`가 바뀔 때만 다시 연결한다.

**렌더가 끝없이 반복된다.** 렌더 중에 조건 없이 `setState`를 호출하면 그 호출이 다시 렌더를 예약해서 반복되고, React는 "Too many re-renders" 에러를 낸다. Effect 안에서 `setState`를 하는데 의존성 배열이 없거나 그 state가 의존성 배열에 들어 있으면, Effect 실행 → state 변경 → 렌더 → Effect 실행이 반복된다.

**상태가 초기화된다.** 컴포넌트 안에서 다른 컴포넌트를 정의하면, 바깥 컴포넌트가 다시 렌더링될 때마다 안쪽 컴포넌트는 새 함수가 된다.

```jsx
function Form() {
  const [name, setName] = useState('');

  function Field() {
    return <input />;
  }

  return <Field />;
}
```

재조정은 타입이 다르면 다른 트리로 보기 때문에, React는 `Field`를 매번 버리고 새로 만든다. `Field` 안의 입력값과 포커스가 렌더마다 사라진다. 컴포넌트는 다른 컴포넌트 밖에 정의한다.

첫 번째는 성능 문제고, 나머지 셋은 동작이 틀려지는 문제다. 어떤 컴포넌트가 왜 다시 렌더링됐는지는 React DevTools에서 확인한다. Components 탭 설정의 "Highlight updates when components render"를 켜면 다시 렌더링된 컴포넌트가 화면에 표시되고, Profiler는 렌더마다 각 컴포넌트가 걸린 시간과 다시 렌더링된 이유를 보여준다.

### 리렌더링 범위 줄이기

메모이제이션보다 먼저, 다시 렌더링되는 범위가 좁아지도록 컴포넌트를 나누는 방법이 있다.

**state를 쓰는 곳 가까이에 둔다.** state가 바뀌면 그 state를 가진 컴포넌트부터 아래로 다시 렌더링된다. 입력 예시의 `Page`에서 `query`와 `input`을 `SearchInput` 컴포넌트로 옮기면 입력할 때 `SearchInput`만 다시 렌더링되고 `HugeTable`은 다시 렌더링되지 않는다.

**바뀌지 않는 부분을 `children`으로 받는다.** state를 위에 둬야 하는 경우에는 그 아래 내용을 `children`으로 받는다.

```jsx
function ScrollTracker({ children }) {
  const [y, setY] = useState(0);
  // 스크롤할 때마다 setY
  return <div data-y={y}>{children}</div>;
}

<ScrollTracker>
  <HugeTable rows={rows} />
</ScrollTracker>
```

`<HugeTable />` 엘리먼트는 `ScrollTracker`를 사용하는 바깥 컴포넌트가 만들어서 넘긴 객체다. `ScrollTracker`가 다시 렌더링돼도 `children`은 이전과 같은 객체라서, React는 `HugeTable`의 렌더를 건너뛴다.

**Context를 나눈다.** Context 값이 바뀌면 그 Context를 쓰는 컴포넌트는 전부 다시 렌더링되고, `memo`로도 막을 수 없다. Provider의 `value`에 렌더마다 새 객체를 넘기면 값이 그대로여도 매번 바뀐 것으로 처리된다. 자주 바뀌는 값과 거의 바뀌지 않는 값은 Context를 나눠서 담는다.

### 메모이제이션

`memo`로 감싼 컴포넌트는 부모가 다시 렌더링돼도 props가 이전과 같으면 렌더를 건너뛴다. props 비교는 각 prop을 `Object.is`로 비교하는 얕은 비교다. 객체나 함수는 렌더할 때마다 새로 만들어져서 내용이 같아도 다른 값으로 판단되기 때문에, `memo`만 쓰면 효과가 없는 경우가 많다. 이때 `useMemo`와 `useCallback`으로 참조를 유지한다.

```jsx
const List = memo(function List({ items, onSelect }) {
  // ...
});

function Page({ data }) {
  const items = useMemo(() => data.filter((d) => d.visible), [data]);
  const onSelect = useCallback((id) => {
    // ...
  }, []);

  return <List items={items} onSelect={onSelect} />;
}
```

공식 문서는 렌더링이 눈에 띄게 느릴 때만 이런 최적화를 하라고 안내한다. 다시 렌더링돼도 결과가 같으면 DOM은 바뀌지 않기 때문에, 대부분의 리렌더링은 성능 문제가 되지 않는다.

### React Compiler

**React Compiler**는 `memo`, `useMemo`, `useCallback`으로 하던 메모이제이션을 빌드 단계에서 자동으로 넣어주는 도구다. 2025년 10월 7일에 1.0이 나왔다. Babel 플러그인(`babel-plugin-react-compiler`)으로 동작하고, React 17 이상에서 쓸 수 있다. React 17·18에서는 `react-compiler-runtime` 패키지를 따로 설치해야 한다. Expo는 SDK 54부터 기본으로 켜져 있고, Vite와 Next.js는 프로젝트를 만들 때 템플릿으로 고를 수 있다.

컴파일러는 먼저 어떤 함수를 컴파일할지 정한다. 기본 설정에서는 이름이 대문자로 시작하거나(컴포넌트) `use`로 시작하는(훅) 함수 중에서, JSX를 만들거나 다른 훅을 호출하는 함수만 컴파일한다. 이름이 `useVisible`이어도 안에서 훅을 호출하지 않고 JSX도 만들지 않으면 컴파일하지 않는다.

컴파일할 함수는 Babel이 만든 구문 트리를 컴파일러 자체의 중간 표현(HIR)으로 바꿔서 분석한다. 각 값이 어떤 값에서 계산되는지(데이터 흐름)와 어디서 바뀌는지(변경 가능성)를 추적해서, 함께 계산되는 값들을 묶고 그 묶음이 어떤 입력에 의존하는지 정한다. 그다음 묶음마다 입력이 이전 렌더와 같은지 확인하는 코드를 넣는다.

```jsx
const $ = _c(7); // 이 컴포넌트가 쓸 캐시 칸 7개

let t1;
if ($[0] !== data) {
  t1 = data.filter(_temp); // data가 바뀌었으면 다시 계산하고 저장
  $[0] = data;
  $[1] = t1;
} else {
  t1 = $[1]; // 같으면 저장해둔 결과를 그대로 쓴다
}
```

캐시 배열은 컴포넌트 인스턴스마다 하나씩 있고 다시 렌더링돼도 유지된다. 입력 비교는 `!==`로 하는 참조 비교다. 이 구조로 컴파일러는 세 가지를 대신한다.

- 계산 결과를 입력이 같을 때 재사용한다. `useMemo`가 하던 일이다.
- props나 state를 참조하지 않는 함수는 컴포넌트 밖으로 빼내고, 참조하는 함수는 입력이 같을 때 이전 함수를 재사용한다. `useCallback`이 하던 일이다.
- JSX 엘리먼트도 입력이 같으면 이전 렌더에서 만든 객체를 재사용한다. React는 자식 엘리먼트가 이전 렌더와 같은 객체면 그 자식의 렌더를 건너뛴다. 그래서 자식을 `memo`로 감싸지 않아도, 자식에게 넘기는 값이 그대로면 자식은 다시 렌더링되지 않는다.

훅은 조건문이나 이른 `return` 뒤에서 호출할 수 없어서, `useMemo`로는 `if (!user) return null;` 뒤에 있는 계산을 메모이제이션할 수 없다. 컴파일러는 훅을 호출하는 게 아니라 캐시를 읽고 쓰는 코드를 넣기 때문에 이런 위치의 계산도 메모이제이션한다.

#### React Compiler가 처리하지 않는 경우

- **컴포넌트와 훅이 아닌 함수, 클래스 컴포넌트**는 컴파일하지 않는다. 컴포넌트 안에서 호출하는 유틸 함수는 호출 결과만 메모이제이션되고, 함수 자체는 바뀌지 않는다.
- **캐시는 컴포넌트 인스턴스마다 따로 있다.** 여러 컴포넌트가 같은 데이터로 같은 비싼 계산을 하면 각자 한 번씩 계산한다. 공식 문서는 이런 함수는 React 밖에서 따로 메모이제이션하라고 안내한다.
- **React 규칙을 어긴 컴포넌트는 건너뛴다.** 컴파일러는 렌더 단계가 순수하고 훅 규칙을 지킨다고 가정하고 코드를 바꾼다. 훅을 조건문 안에서 호출하는 것처럼 규칙을 어긴 코드를 찾으면 그 컴포넌트는 컴파일하지 않고 원래 코드 그대로 둔다. 빌드는 실패하지 않아서 어떤 컴포넌트가 빠졌는지 드러나지 않는다. `eslint-plugin-react-hooks`의 `recommended` 설정에 들어간 컴파일러 기반 규칙으로 이런 코드를 찾는다.
- **규칙 위반을 전부 찾지는 못한다.** 렌더 중에 props로 받은 배열에 `push`하는 코드는 그대로 컴파일됐다. 이런 코드는 메모이제이션된 결과가 실제 값과 어긋날 수 있다.
- **참조가 매번 바뀌는 입력은 캐시를 쓰지 못한다.** 입력 비교가 참조 비교라서, 컴파일되지 않은 부모나 라이브러리가 렌더할 때마다 새 객체나 함수를 넘기면 그 값에 의존하는 묶음은 매번 다시 계산된다.

특정 컴포넌트를 컴파일에서 빼려면 함수 본문 맨 앞에 `"use no memo"`를 쓴다. 공식 문서는 이 지시어를 문제를 고칠 때까지만 쓰는 임시 수단으로 안내한다.

공식 블로그는 새 코드에서는 메모이제이션을 컴파일러에 맡기고, 메모이제이션할 값을 직접 정해야 할 때만 `useMemo`, `useCallback`을 쓰라고 안내한다. 이미 있는 `useMemo`, `useCallback`은 지우면 컴파일 결과가 달라질 수 있어서, 그대로 두거나 테스트한 뒤에 지우라고 한다. Meta Quest Store에 적용했을 때 첫 로드와 페이지 이동이 최대 12% 빨라졌고, 일부 상호작용은 2.5배 이상 빨라졌다고 한다.
