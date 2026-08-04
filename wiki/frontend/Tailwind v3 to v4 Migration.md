---
title: Tailwind v3 to v4 Migration
category: frontend
tags: [frontend, tailwind, migration]
related: [[Tailwind Installation]], [[Tailwind Theme Configuration]], [[Tailwind Accessibility]]
---

# Tailwind v3 to v4 Migration

인터넷에서 찾는 예제 상당수가 아직 v3 기준이라, 그대로 붙여넣으면 동작하지 않거나 다르게 보인다. 주요 차이만 정리.

## 핵심 개념

### 구조적 변화

| 항목 | v3 | v4 |
|---|---|---|
| 진입점 | `@tailwind base; @tailwind components; @tailwind utilities;` | `@import "tailwindcss";` |
| 설정 | `tailwind.config.js` | CSS의 `@theme` (JS 설정은 `@config`로 계속 사용 가능) |
| 스캔 경로 | `content: [...]` 배열 | 자동 감지 + `@source` |
| 플러그인 | config의 `plugins: []` | `@plugin "..."` |
| PostCSS | `tailwindcss`, `autoprefixer`, `postcss-import` 필요 | `@tailwindcss/postcss` 하나 (나머지 내장) |
| 다크모드 class 전략 | `darkMode: 'class'` | `@custom-variant dark (...)` |

### 이름이 바뀐 유틸리티

| v3 | v4 |
|---|---|
| `shadow-sm` | `shadow-xs` |
| `shadow` | `shadow-sm` |
| `rounded-sm` | `rounded-xs` |
| `rounded` | `rounded-sm` |
| `blur`, `drop-shadow` 등 무접미사 | 각각 `-sm`이 붙는 형태로 이동 |
| `outline-none` | `outline-hidden` (진짜 `outline: none`은 `outline-none`) |
| `ring` (3px) | `ring` = 1px, v3와 같게 하려면 `ring-3` |
| `bg-opacity-50` | `bg-black/50` |
| `flex-shrink-0` / `flex-grow` | `shrink-0` / `grow` |

### 기본값 변화

- **테두리 색**: v3 `gray-200` → v4 `currentColor`. `<div class="border">`만 쓰면 글자색과 같은 테두리가 나온다. 색을 명시하거나 base 레이어에서 기본값을 지정한다.

```css
@layer base {
  *, ::after, ::before { border-color: var(--color-gray-200); }
}
```

- **`ring` 기본**: 3px 파란색 → 1px currentColor.
- **placeholder 색**: `gray-400` → 현재 글자색의 50% 투명도.

## Laravel 구현

### 마이그레이션 도구

```bash
npx @tailwindcss/upgrade
```

v3 프로젝트를 자동 변환한다. 별도 브랜치에서 실행하고 diff를 검토할 것. 커스텀 플러그인이나 복잡한 config는 수동 확인이 필요하다.

Laravel installer가 만든 프로젝트에 v3 잔재가 남아 충돌하는 문제는 [[Tailwind Installation]]의 "주의사항" 참고.

## 주의사항 / 안티패턴

- v3 예제를 그대로 복붙하면 `@tailwind base;` 3줄이나 `tailwind.config.js` 문법이 v4에서 동작하지 않는다.
- `outline-none`으로 포커스 링을 지우던 v3 관용구를 v4에 그대로 쓰면 이름이 달라져 있다 ([[Tailwind Accessibility]] 참고).
- 테두리 기본색이 `currentColor`로 바뀌었으므로, v3에서 마이그레이션한 프로젝트는 `border` 단독 클래스를 쓰는 곳마다 의도치 않은 색이 나올 수 있다.

## 참고

- [[Tailwind Installation]] — v3 잔재 충돌 확인
- [[Tailwind Theme Configuration]] — CSS-first `@theme` 설정
- [[Tailwind Accessibility]] — `outline-hidden` 이름 변경
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`
