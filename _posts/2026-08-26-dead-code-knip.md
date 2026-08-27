---
title: "굳이 이걸 Agent로 해야 할까 싶어서 knip으로 바꿨다"
description: "죽은 코드 제거, FSD 감사, 컨벤션 체크를 모두 Claude Code에 맡겨두고 돌리다가, 죽은 코드 탐지만 knip으로 넘긴 이야기."
date: 2026-08-26
categories: [업무회고]
tags: [AI, Claude-Code, knip, TypeScript, Frontend]
---

[앞선 글]({% post_url 2026-08-14-claude-code-audit-workflow %})에서 죽은 코드 제거, FSD 감사, 컨벤션 체크를 Claude Code에 맡기는 워크플로우를 만들었다. 세 개로 나눈 기준은 감사 대상이었고, 실행은 셋 다 Claude Code가 했다. 코드를 읽고 문제를 찾는 일이니까 세 개 다 Agent가 하면 된다고 생각했다.

FSD 감사는 ESLint 규칙을 적용하면서 기계가 할 일과 AI가 할 일을 나눠두었다. 죽은 코드 제거와 컨벤션 체크에도 기계적으로 할 수 있는 부분이 있지 않을까 궁금했다.

죽은 코드는 예전에 knip이라는 라이브러리로 찾았던 게 떠올랐다. Agent가 올린 후보 중에는 실제로는 쓰이고 있는 것도 있었는데, 그 오탐을 하나씩 확인하다가 굳이 이걸 Agent로 해야 하나 싶었다.

## 죽은 코드 탐지

어떤 export가 어디서도 참조되지 않는지는 답이 하나로 정해져 있는 문제였다. 참조 그래프를 끝까지 따라가면 나오는 것이고, 중간에 무엇을 판단할 여지가 없었다.

그런데 Agent에게 맡기면 실행할 때마다 코드베이스를 훑으면서 참조를 따라가기 때문에 시간과 토큰을 계속 쓰게 된다. 게다가 그렇게 해서 나온 결과도 실행마다 조금씩 달랐다. 잡히는 후보가 매번 같지 않았다. Agent가 같은 입력에 같은 출력을 내주는 도구는 아니기 때문에 그럴 수 있다고 생각하는데, 죽은 코드를 찾는 일에는 그런 여지가 필요하지 않았다.

## knip과 remove-unused-vars

knip은 프로젝트를 전체 스캔해서 사용하지 않는 파일, export, 의존성을 찾아주는 도구다. remove-unused-vars는 knip과 같은 webpro-nl 조직에서 만든 도구인데, 찾아낸 것을 실제로 제거해준다.

그래서 knip이 찾고 remove-unused-vars가 제거하는 구조로 바꾸었다. 참조를 따라가고 판단하는 부분을 Claude Code에서 이 두 도구에게 넘긴 것이다.

워크플로우 자체는 그대로 두었다. GitHub Action이 주기적으로 실행되고 결과를 PR로 만드는 흐름은 그대로이고, 바뀐 것은 그 안에서 참조를 보던 부분이다.

이렇게 해두고 돌려보니 실행할 때마다 PR이 쌓여서 노이즈가 됐다. 그래서 실행 주기를 주 1회로 줄였다.

## 기계가 할 일과 AI가 할 일

처음 감사 워크플로우를 만들 때는 자동화할 것은 다 Agent로 하면 된다고 생각했다. 그런데 세 가지를 다시 보니 knip처럼 이미 도구가 할 수 있는 일이 섞여 있었다. 그래서 죽은 코드 탐지는 knip으로 넘겼다. FSD 감사와 컨벤션 체크도 레이어 경계를 넘는 import처럼 규칙으로 적을 수 있는 것은 ESLint에게 넘기고, 남은 부분만 Claude Code가 보도록 두었다.

FSD 감사를 돌린 결과가 그 예다. ESLint 경계 규칙에서는 위반이 하나도 나오지 않았다.

![ESLint 경계 검사 결과](/assets/img/posts/fsd-audit-eslint-clean.png)

같은 실행에서 Claude Code는 `src/lib/`를 짚었다. FSD 레이어 트리 밖에 있는 디렉터리라 `eslint-plugin-boundaries`의 `boundaries/elements` 패턴에 걸리지 않아 아예 검사 대상이 아니었고, 다른 레이어들이 상대경로로 이 디렉터리를 직접 참조하고 있었다.

![ESLint가 잡지 못한 구조적 발견](/assets/img/posts/fsd-audit-structural-finding.png)

AI가 있으니까 자동화는 다 AI에게 맡기면 된다고 생각했었는데, Agent로 전부 처리하는 것보다 기계적으로 할 수 있는 것과 AI가 판단해야 하는 것을 구분하는 게 중요하다는 것을 느꼈다.
