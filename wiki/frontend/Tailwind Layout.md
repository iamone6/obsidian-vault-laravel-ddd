---
title: Tailwind Layout
category: frontend
tags: [frontend, tailwind, css, layout, flexbox, grid]
related: [[Tailwind CSS]], [[Tailwind Utility Syntax]], [[Tailwind Variants]]
---

# Tailwind Layout

Flexbox, Grid, 컨테이너 중앙 정렬, 컨테이너 쿼리로 레이아웃을 구성하는 방법.

## 핵심 개념

### Flexbox

```html
<div class="flex items-center justify-between gap-4">
  <span>왼쪽</span><span>오른쪽</span>
</div>

<div class="flex flex-col gap-2">...</div>  <!-- 세로 스택 -->

<div class="flex">
  <aside class="w-64 shrink-0">사이드바</aside>
  <main class="flex-1 min-w-0">본문</main>  <!-- 남는 공간 채우기 -->
</div>
```

| 클래스 | 의미 |
|---|---|
| `flex-row` / `flex-col` | 주축 방향 (기본 row) |
| `items-start/center/end/stretch/baseline` | 교차축 정렬 |
| `justify-start/center/end/between/around/evenly` | 주축 정렬 |
| `flex-1` | `flex: 1 1 0%` (남은 공간 차지) |
| `shrink-0` / `grow` | 줄어들지 않음 / 늘어남 |

> `min-w-0`은 flex 아이템 안에서 긴 텍스트가 넘칠 때 필요한 관용구다. flex 아이템의 기본 `min-width: auto` 때문에 `truncate`가 동작하지 않는 문제를 해결한다.

### Grid

```html
<div class="grid grid-cols-3 gap-6">...</div>

<!-- 반응형: 모바일 1열 → 태블릿 2열 → 데스크톱 4열 -->
<div class="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-4">...</div>

<!-- 아이템 병합 -->
<div class="grid grid-cols-12 gap-4">
  <div class="col-span-12 lg:col-span-8">본문</div>
  <div class="col-span-12 lg:col-span-4">사이드</div>
</div>

<!-- 자동 반응형 (미디어쿼리 없이) -->
<div class="grid grid-cols-[repeat(auto-fill,minmax(16rem,1fr))] gap-4">...</div>
```

### 컨테이너와 중앙 정렬

```html
<div class="mx-auto max-w-5xl px-4 sm:px-6 lg:px-8">...</div>
```

`container` 유틸리티도 있지만, 위 조합이 더 명시적이고 실무에서 흔히 쓰인다.

### 컨테이너 쿼리 (v4 기본 내장)

뷰포트가 아니라 **부모 요소의 폭**을 기준으로 반응한다. 사이드바 안이든 본문이든 재사용되는 카드 컴포넌트에 유용하다.

```html
<div class="@container">
  <div class="flex flex-col @md:flex-row @md:items-center gap-4">
    <img class="w-full @md:w-32" src="...">
    <div>...</div>
  </div>
</div>
```

`@sm:` `@md:` `@lg:`는 컨테이너 폭 기준, `@max-md:`는 그 이하일 때. 이름 지정: `@container/card` → `@md/card:flex-row`.

## 참고

- [[Tailwind Utility Syntax]] — 간격/색상 스케일
- [[Tailwind Variants]] — `md:` 같은 반응형 접두사 상세, 상태 variant
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`
