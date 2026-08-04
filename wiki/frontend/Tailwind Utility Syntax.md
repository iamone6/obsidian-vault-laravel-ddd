---
title: Tailwind Utility Syntax
category: frontend
tags: [frontend, tailwind, css]
related: [[Tailwind CSS]], [[Tailwind Theme Configuration]]
---

# Tailwind Utility Syntax

Tailwind 클래스 이름과 실제 CSS 속성의 대응 규칙, 간격/색상 스케일, 임의 값(arbitrary value) 문법.

## 핵심 개념

대부분의 클래스는 `속성약어-값` 구조다.

| Tailwind | CSS | 비고 |
|---|---|---|
| `p-4` / `px-4` / `py-2` | `padding` / 가로축 / 세로축 | t/r/b/l로 방향 지정 |
| `m-4` / `mx-auto` / `-mt-2` | `margin` (음수는 앞에 `-`) | |
| `w-full` `h-screen` `size-10` | `width`/`height` 동시 지정 | |
| `text-lg` | `font-size` + line-height | |
| `text-gray-700` | `color` | `text-` 접두사가 크기·색 둘 다 담당 |
| `bg-blue-500` | `background-color` | |
| `border-2` `border-t` | `border-width` | |
| `rounded-lg` | `border-radius` | |
| `flex` `grid` `hidden` | `display` | |
| `items-center` `justify-between` `gap-4` | flex/grid 정렬·간격 | |
| `absolute` `relative` `sticky` `inset-0` `z-10` | 위치/z-index | |

IntelliSense 자동완성으로 확인하며 익히는 편이 암기보다 빠르다.

### 간격 스케일

기본 단위는 `--spacing: 0.25rem`(4px). 숫자 × 4 = px로 암산한다.

```
p-1 → 4px   p-2 → 8px   p-4 → 16px   p-6 → 24px   p-8 → 32px   p-12 → 48px
```

v4에서는 정의되지 않은 숫자(`p-13`, `mt-27`)도 자동 계산되어 동작한다. 특수 값: `p-px`(1px), `w-1/2`(50%), `w-screen`(100vw), `h-dvh`(동적 뷰포트 높이, 모바일 주소창 대응).

### 색상 스케일

`색상이름-숫자` 형식, 숫자는 **50(가장 밝음) ~ 950(가장 어두움)**.

```
bg-gray-50   /* 페이지 배경용 */
bg-gray-100  /* 카드 배경, 구분선 */
bg-gray-500  /* 중간, 보조 텍스트 */
bg-gray-900  /* 본문 텍스트 */
```

투명도는 슬래시로 붙인다: `bg-black/50`, `text-white/80`. v4 기본 색상은 `oklch` 색 공간이라 넓은 색역 디스플레이에서 더 선명하고, 디자이너가 준 hex와 미세하게 다르게 보일 수 있다.

### 임의 값 (arbitrary values)

스케일에 없는 값이 필요하면 대괄호를 쓴다.

```html
<div class="top-[117px] w-[calc(100%-2rem)] bg-[#1da1f2] text-[13px]">
<div class="grid-cols-[1fr_500px_2fr]">   <!-- 공백 대신 언더스코어 -->
<div class="bg-(--brand-color)">          <!-- bg-[var(--brand-color)]의 축약 -->
<div class="[mask-type:luminance] [--my-var:12px]">   <!-- 임의 속성 -->
```

## 주의사항 / 안티패턴

- 임의 값은 탈출구다. 남용하면 Tailwind를 쓰는 이유(디자인 토큰 일관성)가 사라진다. **같은 임의 값을 두 번 이상 쓰면 [[Tailwind Theme Configuration]]의 `@theme`에 토큰으로 등록**할 것.

## 참고

- [[Tailwind CSS]] — 개요
- [[Tailwind Theme Configuration]] — 커스텀 토큰(`@theme`)으로 승격
- [[Tailwind Layout]] — 레이아웃 관련 클래스
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`
