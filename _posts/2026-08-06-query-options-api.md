---
title: "예전엔 query-key-factory를 썼는데 지금은 왜 queryOptions일까"
description: "쿼리 키를 라이브러리로 관리하던 이유와, 그 일이 TanStack Query 코어의 queryOptions로 넘어온 과정을 정리한 내용."
date: 2026-08-06
categories: [학습정리]
tags: [TanStack-Query, React-Query, TypeScript, Frontend]
---
요즘 하는 프로젝트 코드를 보니 `queryOptions`를 쓰고 있었다. 예전에 React Query를 쓰던 프로젝트에서는 `@lukemorales/query-key-factory`로 쿼리 키를 관리했는데, 이번 프로젝트에는 그 라이브러리가 없었다. 대신 공식 문서에 있는 Query Options 가이드의 헬퍼만 쓰고 있었다. 왜 라이브러리를 걷어냈는지 궁금해서 찾아봤다.

## 쿼리 키 관리

`queryKey`는 React Query가 캐시에서 쿼리를 구분하는 값이다. 이 비교는 배열을 참조가 아니라 내용으로 한다. `['todos', 'list']`를 컴포넌트가 렌더링될 때마다 새로 만들어도 내용이 같으면 여전히 같은 쿼리로 묶이고, 내용이 조금이라도 다르면 다른 쿼리가 된다.

보통 맨 앞에는 엔티티 이름을 접두사로 붙인다. 이게 없으면 다른 기능에서 만든 키와 배열 모양이 같아져서, 서로 다른 데이터가 같은 쿼리로 캐시에 묶일 수 있다.

그런데 이렇게 써야 하는 값인데도, 실제로 검사해주는 건 없다.

```ts
useQuery({ queryKey: ['todos', 'list', filters], queryFn: () => fetchTodos(filters) });
```

같은 배열을 쓰는 곳마다 이 리터럴을 그대로 옮겨 적게 된다. 옮기다 어딘가에서 `['todo', 'list', filters]`로 오타가 나도 에러는 나지 않는다. 그냥 다른 쿼리가 하나 더 생길 뿐이다.

무효화할 때도 마찬가지다. `invalidateQueries`는 넘긴 배열로 시작하는 키를 모두 찾아서 무효화한다. `['todos', 'list']`만 넘기면 뒤에 어떤 필터가 붙어있든 목록 쿼리가 전부 걸린다. 이 매칭이 되려면 접두사를 정확히 맞춰 적어야 하는데, 이것도 검사해주는 게 없기는 똑같다.

```ts
queryClient.invalidateQueries({ queryKey: ['todos', 'list'] });
```

키 구조를 바꾸면 이런 자리를 전부 찾아 같이 고쳐야 하는데, 손으로 찾다 보면 몇 군데는 놓치기 쉽다. 놓친 자리는 예전 키 구조를 그대로 쓰게 되고, 새 접두사로 무효화해도 거기는 걸리지 않는다. 화면 어딘가는 계속 오래된 데이터를 보여주는데, 그래도 타입 검사에는 걸리지 않는다.

그래서 키를 한 파일에 모으는 팩토리 패턴이 나왔다. 직접 만들면 이렇게 된다.

```ts
const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: string) => [...todoKeys.lists(), filters] as const,
};
```

`@lukemorales/query-key-factory`를 쓰면 이 객체를 기능마다 만들고 `as const`와 스프레드를 관리하는 일을 대신해준다.

```ts
import { createQueryKeys } from '@lukemorales/query-key-factory';

export const todos = createQueryKeys('todos', {
  list: (filters: Filters) => ({
    queryKey: [{ filters }],
    queryFn: () => api.getTodos(filters),
  }),
  detail: (todoId: string) => ({
    queryKey: [todoId],
    queryFn: () => api.getTodo(todoId),
  }),
});
```

`todos.detail(id).queryKey`는 `['todos', 'detail', id]`가 되고, 무효화에 쓸 접두사는 `_def`로 나온다.

```ts
useQuery(todos.detail(todoId));
queryClient.invalidateQueries({ queryKey: todos.list._def }); // ['todos', 'list']
```

이 방식은 `queryKey`와 `queryFn`을 한 항목에 같이 적는다. 그전까지는 키만 팩토리에 있고 `queryFn`은 컴포넌트나, 재사용하려고 감싼 커스텀 훅에 따로 있었다.

`queryKey`는 캐시에서 쿼리를 구분하는 이름이면서 `queryFn`의 의존성 목록이다. `filters`가 바뀌면 다시 받아와야 하니 키에도 `filters`가 들어가야 한다. 둘이 다른 파일에 있으면 `api.getTodos(filters)`에는 필터를 넘기고 키에는 넣지 않는 일이 생기고, 그러면 필터를 바꿔도 이전 결과가 보인다. 한 항목에 적으면 함수에 넘기는 값과 키를 같이 보게 된다.

## queryOptions

v5의 `queryOptions`는 쿼리 정의를 코어 헬퍼로 끌어왔다.

```ts
import { queryOptions } from '@tanstack/react-query';

export const todoDetailQuery = (id: string) =>
  queryOptions({
    queryKey: ['todos', 'detail', id],
    queryFn: () => api.getTodo(id),
    staleTime: 5000,
  });
```

런타임에는 아무 일도 하지 않는다. 받은 객체를 그대로 돌려주는 함수다. 라이브러리 없이도 `queryKey`와 `queryFn`을 한 항목에 묶을 수 있게 된 것이다. 이 객체는 `useQuery`뿐 아니라 `useSuspenseQuery`, `prefetchQuery`, `ensureQueryData`에도 형태 그대로 넘길 수 있다.

`queryOptions`로 만든 `queryKey`에는 부수적으로 타입 정보도 붙는다. 캐시에서 직접 꺼낼 때도 이 키로는 `Todo`가 나온다는 걸 안다.

```ts
const todo = queryClient.getQueryData(todoDetailQuery(id).queryKey);
// Todo | undefined
```

옵션 이름 오타도 걸린다. TypeScript는 객체를 그 자리에서 바로 넘길 때만 모르는 속성을 걸러내기 때문에, 옵션을 변수로 빼면 `stallTime` 같은 오타가 지나간다. `staleTime` 기본값은 0이라 이 오타가 나면 화면에 들어올 때마다 다시 요청이 나가는데 동작은 멀쩡하다. `queryOptions`로 감싸면 그 안에서 검사가 걸린다.

## 커스텀 훅

이 재사용을 커스텀 훅으로 하던 방법도 있었다. 쿼리를 재사용하려고 만든 훅이다.

```ts
export function useTodoDetail(id: string) {
  return useQuery({
    queryKey: ['todos', 'detail', id],
    queryFn: () => api.getTodo(id),
  });
}
```

훅은 컴포넌트와 다른 훅 안에서만 부를 수 있다. 라우터 로더, 서버 컴포넌트, 이벤트 핸들러에서는 못 쓴다. `useQuery`에 묶여 있기도 해서, 같은 쿼리를 `useSuspenseQuery`로 쓰려면 훅을 하나 더 만들게 된다.

`queryOptions`가 돌려주는 건 객체라서 이 제약이 없다.

```ts
// 라우터 로더에서 미리 받아두기
loader: ({ params }) => queryClient.ensureQueryData(todoDetailQuery(params.id)),

// 컴포넌트에서 꺼내 쓰기
const { data } = useSuspenseQuery(todoDetailQuery(params.id));
```

예전에는 로더와 컴포넌트에 `queryKey`와 `queryFn`을 각각 적었다. 한쪽이라도 다르면 로더가 채워둔 캐시를 컴포넌트가 찾지 못하고 같은 요청이 다시 나간다. 화면은 정상으로 보이니 네트워크 탭을 열기 전에는 모른다.

커스텀 훅은 설정을 공유하는 일과 로직을 공유하는 일을 같이 떠안는다. 재사용하려는 건 앞쪽(쿼리 정의)뿐인데, 훅으로 감싸는 순간 뒤쪽(훅 호출 규칙)까지 따라붙는다.

## 팩토리의 몫

`queryOptions`는 쿼리 하나만 다룬다. `['todos', 'detail', id]`라는 배열을 받아 함수와 묶을 뿐이고, 그 배열이 `todos` 아래 `detail`에 속한다는 건 모른다. 라이브러리가 `_def`로 만들어주던 접두사는 여기 없다.

그래서 팩토리 객체는 그대로 두고, 실제 쿼리에 해당하는 항목만 `queryOptions`로 감싼다.

```ts
const todoQueries = {
  all: () => ['todos'],
  lists: () => [...todoQueries.all(), 'list'],
  list: (filters: string) =>
    queryOptions({
      queryKey: [...todoQueries.lists(), filters],
      queryFn: () => api.getTodos(filters),
    }),
  details: () => [...todoQueries.all(), 'detail'],
  detail: (id: string) =>
    queryOptions({
      queryKey: [...todoQueries.details(), id],
      queryFn: () => api.getTodo(id),
      staleTime: 5000,
    }),
};

queryClient.invalidateQueries({ queryKey: todoQueries.lists() });
```

`all`, `lists`, `details`는 키만 돌려준다. 실행할 쿼리가 없고 무효화 범위를 지정하는 데만 쓰인다.

## 공식 권장 방식

공식 문서를 확인해보니 `queryOptions` 쪽이 권장 방식이었다. Query Options 가이드는 `queryKey`와 `queryFn`을 여러 곳에서 공유하면서도 서로 붙여두는 가장 좋은 방법이 이 헬퍼라고 적어뒀다.

ESLint 플러그인에도 규칙이 생겼다. `@tanstack/eslint-plugin-query`의 `prefer-query-options`이고, 2026년 3월에 추가돼서 `recommended-strict` 설정에 들어 있다.

```tsx
// 규칙에 걸리는 코드
function Component({ id }) {
  return useQuery({
    queryKey: ['get', id],
    queryFn: () => Api.get(`/foo/${id}`),
  });
}
```

`useQuery`에 키와 함수를 직접 적으면 걸린다. 커스텀 훅으로 감싼 것도 마찬가지다. 같은 쿼리 키가 서로 다른 `queryFn`과 엮이는 걸 막는 게 이유라고 적혀 있다.

이 규칙은 키를 손으로 적는 것도 잡는다.

```tsx
// 걸리는 코드
queryClient.getQueryData(['todo', id]);
queryClient.invalidateQueries({ queryKey: ['todo', id] });

// 걸리지 않는 코드
queryClient.getQueryData(todoOptions(id).queryKey);
queryClient.invalidateQueries({ queryKey: todoOptions(id).queryKey });
```

앞에서 필터를 키에 안 넣으면 이전 결과가 보인다고 했는데, 그건 `exhaustive-deps` 규칙이 잡는다. `queryFn`이 쓰는 값이 키에 빠져 있으면 걸린다. 타입 검사로는 안 되고 lint로 막는 부분이다.

그렇다고 팩토리를 버리라는 쪽은 아니다. Query Keys 가이드는 지금도 더 읽을거리로 「Effective React Query Keys」와 query-key-factory 패키지를 링크해두고 있다. v5 문서에서 이 패키지의 커뮤니티 페이지는 없어졌고, 링크는 GitHub 저장소로 걸려 있다.

## 결론

이 라이브러리가 하던 일은 두 가지였다. 키 생성과 접두사 관리, 그리고 키와 `queryFn`을 한 항목에 적게 하는 것. 두 번째는 `queryOptions`가 가져갔고 타입 추론이 붙었다. 첫 번째는 지금도 팩토리 객체를 직접 써서 처리한다.

지금 새로 시작한다면 나는 라이브러리 없이 팩토리 객체와 `queryOptions`를 쓰고, `prefer-query-options` 규칙을 켜둘 것 같다.

---

### 참고

- [Query Options — TanStack Query 공식 문서](https://tanstack.com/query/latest/docs/framework/react/guides/query-options)
- [prefer-query-options — ESLint Plugin Query](https://tanstack.com/query/latest/docs/eslint/prefer-query-options)
- [The Query Options API — TkDodo's blog](https://tkdodo.eu/blog/the-query-options-api)
- [Creating Query Abstractions — TkDodo's blog](https://tkdodo.eu/blog/creating-query-abstractions)
- [Effective React Query Keys — TkDodo's blog](https://tkdodo.eu/blog/effective-react-query-keys)
- [lukemorales/query-key-factory](https://github.com/lukemorales/query-key-factory)

