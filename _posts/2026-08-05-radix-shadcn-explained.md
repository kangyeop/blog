---
title: "Radix, shadcn/ui는 뭔데 다들 이걸 쓰는 걸까"
description: "MUI 대신 Radix와 shadcn/ui가 기본값처럼 깔려 있는 이유를 확인해본 내용. 스타일 없는 컴포넌트와 복사해서 소유하는 방식."
date: 2026-08-05
categories: [Frontend]
tags: [Radix, shadcn, Frontend, UI, Design-System]
---

[지난 글]({% post_url 2026-07-31-function-component-convention %})까지는 폴더 구조와 파일 안의 선언 스타일이었다. 그다음에 눈에 들어온 건 UI 스택이었다.

돌아와서 본 프로젝트들은 예전과 달랐다. MUI나 Ant Design 대신 `radix-ui`와 `shadcn/ui`가 기본값처럼 깔려 있었다. 이름만 보면 컴포넌트 라이브러리 같은데 쓰는 방식이 낯설었다. `npm install`이 아니라 CLI로 컴포넌트 코드를 내 프로젝트에 복사해 온다.

이게 뭔지, 요즘은 왜 이걸 많이 쓰는지, 기존에 쓰던 라이브러리랑 뭐가 다른지 궁금해서 좀 찾아봤다.

## 기존 컴포넌트 라이브러리의 문제

MUI, Ant Design은 컴포넌트와 스타일이 한 세트로 묶여 있다. 설치하면 바로 그럴듯한 UI가 나온다.

대신 디자인을 조금만 다르게 가져가려고 하면 라이브러리가 정해둔 클래스명과 CSS 특이도를 뚫어야 한다.

```tsx
<Button
  sx={{
    borderRadius: 0,
    // 기본 스타일이 먼저 걸려 있어서 클래스까지 짚어줘야 먹는다
    '&.MuiButton-contained': { backgroundColor: '#111' },
  }}
>
  저장
</Button>
```

버튼 하나 모양 바꾸는 데 테마 오버라이드 문서를 뒤지게 되고, 그래도 안 되면 `!important`를 쓰게 된다.

내부 구조도 열려 있지 않다. 라이브러리 코드는 `node_modules` 안에 있고, 동작을 바꾸려면 props로 열어준 범위 안에서만 가능하다. 버전을 올릴 때마다 breaking change를 확인해야 하는 것도 같은 이유다.

## Radix: 스타일 없는 컴포넌트

Radix(정확히는 Radix Primitives)는 스타일을 아예 빼는 쪽으로 접근한다. Dialog, DropdownMenu, Popover, Tooltip, Accordion 같은 컴포넌트를 제공하지만 CSS는 한 줄도 주지 않는다. 대신 접근성과 동작을 채워준다.

Dialog라면 열렸을 때 포커스를 안에 가두는 것(focus trap), `Esc`로 닫히는 것, 배경 스크롤을 막는 것, `role="dialog"`나 `aria-modal` 같은 ARIA 속성을 붙이는 것까지 처리한다. 직접 구현하려면 키보드 네비게이션과 스크린 리더 대응까지 WAI-ARIA 명세를 하나씩 따라가야 한다.

코드로 보면 이렇게 생겼다.

```tsx
import { Dialog } from 'radix-ui';

<Dialog.Root>
  <Dialog.Trigger>열기</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay className="fixed inset-0 bg-black/50" />
    <Dialog.Content className="fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 rounded-lg bg-white p-6">
      <Dialog.Title>정말 삭제할까요?</Dialog.Title>
      <Dialog.Close>닫기</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>;
```

여기서 `className`을 전부 지우면 모달처럼 보이는 부분이 하나도 남지 않는다. 화면에 보이는 모양은 전부 내가 붙인 것이고, Radix가 준 건 열고 닫히는 동작과 ARIA 속성뿐이다.

그래서 클래스명을 붙이든 CSS-in-JS를 쓰든 상관없다. 라이브러리 쪽에서 정해둔 모양이 없다.

## shadcn/ui: 컴포넌트를 내 코드로

Radix만 쓰면 스타일을 매번 처음부터 입혀야 한다. shadcn/ui는 Radix 위에 Tailwind CSS로 스타일을 입힌 컴포넌트를 미리 만들어둔다. 다만 npm 패키지로 설치하지 않는다.

```bash
npx shadcn@latest add button
```

이 명령을 실행하면 `Button` 컴포넌트의 소스 코드가 통째로 `components/ui` 폴더에 복사된다. 공식 문서도 스스로를 컴포넌트 라이브러리가 아니라고 소개한다. 복사해서 붙여 넣는 컴포넌트 모음이라는 것이다.

복사되어 온 `components/ui/button.tsx`는 이런 파일이다.

```tsx
const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        outline: 'border border-input bg-background hover:bg-accent',
      },
      size: {
        default: 'h-9 px-4 py-2',
        sm: 'h-8 px-3',
      },
    },
    defaultVariants: { variant: 'default', size: 'default' },
  },
);
```

버튼 모서리를 각지게 하고 싶으면 `rounded-md`를 지우면 되고, variant를 하나 더 만들고 싶으면 `variants`에 키를 추가하면 된다. 특이도를 뚫을 일도, 오버라이드 문서를 찾을 일도 없다. 내 저장소 안의 파일이라서 그렇다.

Radix가 접근성과 동작을, Tailwind가 스타일을, `class-variance-authority(cva)`가 크기나 색상 같은 variant 관리를 맡고, shadcn은 이 조합을 미리 짜서 붙여넣기 좋은 형태로 만들어둔 것에 가깝다.

## 요즘 많이 쓰이는 이유

Tailwind CSS가 사실상 표준처럼 자리 잡은 게 먼저다. 유틸리티 클래스로 스타일을 짜는 흐름에 shadcn이 그대로 맞는다.

그리고 AI로 코드를 짜는 흐름과도 맞는다. 컴포넌트가 `node_modules` 안이 아니라 내 저장소 안의 평범한 소스 코드로 있으면, AI 에이전트가 내부까지 읽고 직접 고칠 수 있다. MUI 같은 라이브러리는 내부가 패키지 안에 있어서 AI든 사람이든 props로 열어준 범위 밖은 손댈 수 없다.

디자인 시스템을 새로 짜는 경우도 이유가 된다. Figma로 만든 커스텀 디자인을 그대로 구현하려면, 스타일이 이미 정해진 라이브러리보다 접근성과 동작만 들어 있는 Radix 쪽이 다루기 편하다. shadcn은 그 스타일을 처음부터 채우는 수고를 줄여준다.

## 가져온 코드의 업데이트

Radix는 접근성과 동작을 맡는 스타일 없는 컴포넌트이고, shadcn/ui는 그 위에 Tailwind로 스타일을 입혀 복사해 쓰게 만든 컴포넌트 모음이다. 설치해서 쓰는 게 아니라 코드를 내 프로젝트로 가져와 소유하는 방식이라 커스터마이징 자유도가 높고 AI로 다루기도 쉽다.

대신 가져온 다음부터는 라이브러리가 챙겨주지 않는다. 접근성 개선이나 버그 수정이 업스트림에 올라와도 내 폴더의 파일에는 자동으로 반영되지 않는다. 필요하면 직접 확인해서 반영해야 한다.

---

### 참고

- [Radix Primitives 공식 문서](https://www.radix-ui.com/primitives)
- [shadcn/ui 공식 문서](https://ui.shadcn.com/)
- [shadcn/ui — Introduction](https://ui.shadcn.com/docs)
