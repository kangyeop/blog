---
title: "(가제) Labor0 없이 Hermes로 되는지"
description: ""
date: 2026-08-13
categories: [사이드프로젝트]
tags: [Hermes, AI-Agent, Labor0]
---

<!--
구성 초안. 실제 경험이 필요한 자리는 [ ]로 표시해뒀다. 채워 넣고 순서/문장은 다시 다듬을 것.
-->

Labor0를 보다가 든 생각을 정리한다. 작업을 그래프로 쪼개고, 선행 작업이 끝나야 다음 작업이 열리고, 사람 판단이 필요한 지점에서 멈췄다가 다시 물어보는 구조 — 지금 Hermes로 만들고 있는 것과 방향이 겹친다. 다른 건 Labor0는 돈을 받는다는 것이다.

## Labor0가 파는 것

- 요청을 작업 그래프로 쪼개고, 의존성이 풀린 작업만 자동 실행
- 사람 판단이 필요한 지점에서 체크포인트로 멈추고 웹 푸시로 알림
- Codex, Claude Code, OpenCode 등 여러 실행 런타임을 붙일 수 있음
- 요금: Sandbox $0(평생 $5 한도) → Pay-as-you-go $99/월 → Team $199/월 → Business $1,500/월 → Enterprise $24,000/년, 전부 AI 사용료에 마크업(10~25%) 추가

## Hermes에 이미 있는 것

- 보드별 SQLite로 작업을 관리하는 칸반 디스패처 + 알림 워처 (`docs/kanban/multi-gateway.md`)
- 정식 스펙 문서(`hermes-kanban-v1-spec.pdf`)까지 있음
- 자체 서버에서 돌기 때문에 마크업 없이 쓰는 AI 사용료가 곧 전체 비용

[Labor0가 강조하는 것 중 Hermes 칸반에 실제로 없는 게 뭔지 확인 필요 — 작업 그래프 시각화? 선행 작업 완료 시 후속 작업 자동 오픈?]

## 지금 구성에서 힘들었던 것

[여기가 핵심. 실제로 겪은 걸 채워야 함. 후보로 짚이는 것들 — 맞으면 살리고 아니면 걷어낼 것]

- [멀티 프로필(default/writer/admin/coder/researcher) 구성하면서 헤맨 지점?]
- [systemd user service로 gateway 띄우는 과정에서 걸린 것?]
- [Slack 연동 붙이면서 겪은 것?]
- [그 외 실제로 며칠씩 잡아먹은 부분?]

## 정리

[Labor0를 안 쓰고 가는 이유를 비용 대비로 정리 — "돈 내고 사는 것"과 "이미 있는데 안 써본 것"의 차이가 뭔지]
