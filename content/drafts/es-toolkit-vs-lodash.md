---
title: "es-toolkit은 lodash를 대체할 수 있을까"
description: "lodash, lodash-es 대안으로 자주 언급되는 es-toolkit을 번들 크기와 성능 수치로 찾아본 내용."
date: 2026-08-11
categories: [학습정리]
tags: [es-toolkit, lodash, JavaScript, Frontend, Performance]
---

lodash는 여전히 프론트엔드 프로젝트에서 흔하게 보이는 유틸리티 라이브러리다. 그런데 최근 들어 lodash 대안으로 es-toolkit이라는 이름이 자주 언급된다. 뭐가 다른지 궁금해서 찾아봤다.

## lodash와 lodash-es의 차이

lodash는 CommonJS로 배포된다. `import _ from 'lodash'`로 통째로 가져오면 쓰지 않는 함수까지 번들에 딸려온다. `lodash/pick`처럼 개별 경로로 import해야 필요한 함수만 가져올 수 있다.

lodash-es는 같은 함수들을 ES 모듈로 다시 배포한 패키지다. ESM이라 번들러가 tree-shaking을 할 수 있다. 다만 내부 구현 자체는 lodash 원본을 그대로 옮긴 것이라서, 오래된 브라우저를 지원하기 위한 폴리필과 예외 처리가 함수 안에 그대로 남아 있다. tree-shaking으로 안 쓰는 함수는 빼도, 실제로 쓰는 함수 하나하나는 여전히 무겁다.

## es-toolkit이 나온 배경

es-toolkit은 토스 프론트엔드 팀이 만든 라이브러리다. [토스 기술 블로그](https://toss.tech/article/50761)에 따르면 시작은 "lodash 안에 현대 웹 개발에는 필요 없는 로직이 계속 남아 있다"는 문제의식이었다. 오래된 브라우저 대응 코드를 걷어내고 최신 JavaScript API로 다시 짜면 같은 기능을 훨씬 가볍고 빠르게 만들 수 있다고 봤다.

사내 라이브러리로 시작했다가 Reddit 등 외부 커뮤니티에 공유되면서 외부 기여가 붙었다. 이후 외부 기여자였던 개발자가 `es-toolkit/compat`이라는 lodash 호환 레이어를 만들어 합류했고, 이 레이어 덕분에 기존 lodash 사용처를 그대로 옮기기가 쉬워졌다.

## 번들 크기

[es-toolkit 공식 문서](https://es-toolkit.dev/bundle-size.html)는 esbuild로 각 함수를 개별 번들링했을 때의 크기를 lodash-es와 비교해 공개하고 있다.

| 함수 | es-toolkit | lodash-es | 감소율 |
|---|---|---|---|
| sample | 94 bytes | 4,849 bytes | -98.1% |
| difference | 90 bytes | 7,992 bytes | -98.9% |
| pick | 132 bytes | 9,554 bytes | -98.6% |
| sum | 93 bytes | 726 bytes | -87.2% |
| debounce | 531 bytes | 2,901 bytes | -81.7% |

함수 하나만 놓고 보면 대부분 90% 넘게 줄어든다. lodash-es 쪽 함수들이 이렇게 큰 이유는 앞서 말한 레거시 대응 코드 때문이다.

## 성능

[성능 비교 페이지](https://es-toolkit.dev/performance.html)에는 벤치마크 수치도 있다. 테스트 환경은 13th Gen Intel Core i5-13400F, Node.js v24.11.1 기준이다.

- omit: 3.6배
- pick: 3.9배
- differenceWith: 2.9배
- intersectionWith: 3.5배
- intersection: 2.0배
- difference: 1.8배

공식 문서는 평균 2배, 일부 함수는 최대 11배까지 차이가 난다고 소개한다. omit, pick처럼 객체를 다루는 함수에서 차이가 크게 나는 편이다.

## API가 완전히 같지는 않다

es-toolkit 기본 패키지는 lodash보다 API 표면이 좁고 타입 정의가 엄격하다. 그래서 일부 함수는 인자 개수나 동작이 lodash와 다르다. 완전히 같은 동작이 필요하면 `es-toolkit/compat`을 쓰면 된다. 이 모듈은 lodash의 API를 그대로 따라가고 lodash 테스트 스위트로 검증되어 있어서, import 경로만 바꾸는 정도로 기존 코드를 옮길 수 있다고 안내한다.

대신 `compat` 쪽은 lodash 동작을 재현하는 코드가 붙는 만큼 기본 `es-toolkit` 패키지보다는 번들이 크다. 번들 크기를 최대한 줄이려면 기본 패키지 API에 맞춰 코드를 고치는 쪽이, 마이그레이션 자체를 쉽게 하려면 compat 쪽이 유리하다.

## 정리

번들 크기와 속도 수치, lodash 호환 레이어까지 갖춰진 걸 보면 새 프로젝트에서 lodash를 새로 도입할 이유는 크지 않아 보인다. 기존 lodash 프로젝트도 `es-toolkit/compat`으로 바꾸는 정도는 진입 장벽이 낮은 편이다.

---

### 참고

- [es-toolkit GitHub](https://github.com/toss/es-toolkit)
- [es-toolkit — Bundle Footprint](https://es-toolkit.dev/bundle-size.html)
- [es-toolkit — Performance](https://es-toolkit.dev/performance.html)
- [es-toolkit, 사내 작은 라이브러리가 전세계적인 표준 라이브러리가 되기까지 — Toss Tech](https://toss.tech/article/50761)
