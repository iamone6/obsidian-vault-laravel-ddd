---
title: Tailwind CSS
category: frontend
tags: [frontend, tailwind, css, utility-first]
related: [[Tailwind Installation]], [[Tailwind Utility Syntax]], [[Tailwind Component Strategy]], [[Directory Structure]]
---

# Tailwind CSS

미리 정의된 유틸리티 클래스를 HTML/Blade에서 조합해 스타일링하는 CSS 프레임워크. 이 위키에서는 Laravel 프로젝트의 기본 프론트엔드 스타일링 도구로 다루며, 향후 NativePHP 레이아웃에도 동일하게 사용된다 (기준 버전 v4.3.x).

## 핵심 개념

전통적인 CSS는 `.card`, `.card__title` 같은 이름을 짓고 별도 CSS 파일에 규칙을 쌓는 방식이라 네이밍 비용, 전역 스코프 오염, "지워도 되는지 알 수 없는 죽은 CSS" 문제가 구조적으로 발생한다. Tailwind는 "한 가지 일만 하는 작은 클래스"를 미리 만들어 두고 마크업에서 조합하는 **유틸리티 우선(utility-first)** 접근으로 이를 피한다.

```html
<div class="p-4 rounded-lg bg-white shadow-sm">
  <h2 class="text-lg font-semibold">제목</h2>
</div>
```

**얻는 것**: 네이밍 불필요, 스타일 수정 범위가 엘리먼트로 한정, 마크업을 지우면 스타일도 사라짐(죽은 CSS 없음), 값이 스케일로 제한되어 일관성 유지, 사용된 클래스만 빌드에 포함되어 최종 CSS가 작음(보통 10~50KB 미만).

**잃는 것**: HTML이 길어짐(→ [[Tailwind Component Strategy]]), 초기 학습 비용, 클래스명을 동적으로 문자열 조합하면 깨짐(→ [[Tailwind Dynamic Class Pitfall]], 실무에서 가장 자주 터지는 문제), 빌드 도구 필요.

> 비유: Tailwind는 Eloquent 같은 고수준 추상화가 아니라 **쿼리 빌더**에 가깝다. CSS 지식을 감춰주는 게 아니라, CSS 지식을 클래스 이름으로 안전하고 일관되게 조립하도록 돕는 도구다.

## Laravel 구현

Laravel 프로젝트에서는 `@tailwindcss/vite` 플러그인으로 Blade + Vite 파이프라인에 통합하는 것이 표준 경로다. 설치와 설정은 [[Tailwind Installation]] 참고.

브라우저 지원 범위가 좁아야 하는 프로젝트(공공/금융 등 구형 환경)라면 최신 CSS 기능(`@property`, `color-mix()`, cascade layers)에 의존하는 v4 대신 별도 유지보수되는 v3.4를 쓰는 것이 현실적이다.

## 주의사항 / 안티패턴

- Tailwind는 도구지 규율이 아니다. 디자인 시스템(색상/간격 토큰, 컴포넌트 규칙)이 팀에 없으면 결과물이 제각각일 수 있다 — [[Tailwind Theme Configuration]]으로 토큰을 강제하는 편이 낫다.
- 클래스가 20~30개 붙은 태그를 방치하지 말 것. 같은 조합이 3번 이상 반복되면 컴포넌트로 추출한다 ([[Tailwind Component Strategy]]).

## 참고

- [[Tailwind Installation]] — Laravel/Vite 설치
- [[Tailwind Utility Syntax]] — 클래스 이름 읽는 법
- [[Tailwind Layout]] — Flex/Grid
- [[Tailwind Variants]] — 상태/반응형/다크모드 접두사
- [[Tailwind Component Strategy]] — 반복 클래스 관리
- [[Tailwind Dynamic Class Pitfall]] — 동적 클래스 함정
- [[Tailwind Theme Configuration]] — `@theme`/`@utility` 설정
- [[Tailwind Dark Mode]] — 다크모드 구현
- [[Tailwind Accessibility]] — 접근성 체크리스트
- [[Tailwind Build and Performance]] — 빌드/배포/성능
- [[Tailwind v3 to v4 Migration]] — 버전 차이
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`
