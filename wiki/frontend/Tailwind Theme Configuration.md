---
title: Tailwind Theme Configuration
category: frontend
tags: [frontend, tailwind, theme, css]
related: [[Tailwind CSS]], [[Tailwind Installation]], [[Tailwind Dynamic Class Pitfall]]
---

# Tailwind Theme Configuration

v4의 가장 큰 변화: `tailwind.config.js`가 아니라 **CSS 파일에서 설정**한다 (CSS-first).

## 핵심 개념

### `@theme` — 디자인 토큰 정의

```css
@import "tailwindcss";

@theme {
  /* 브랜드 색상 → bg-brand-500, text-brand-700 ... 자동 생성 */
  --color-brand-50:  oklch(0.97 0.02 250);
  --color-brand-500: oklch(0.62 0.19 250);
  --color-brand-700: oklch(0.48 0.18 250);

  --font-sans: "Pretendard Variable", Pretendard, system-ui, sans-serif;
  --breakpoint-3xl: 120rem;                 /* 3xl:flex */
  --shadow-card: 0 1px 3px rgb(0 0 0 / 0.08), 0 1px 2px rgb(0 0 0 / 0.04);  /* shadow-card */
  --animate-fade-in: fade-in 0.3s ease-out; /* animate-fade-in */
}

@keyframes fade-in {
  from { opacity: 0; transform: translateY(4px); }
  to   { opacity: 1; transform: none; }
}
```

**핵심**: `@theme`에 정의한 변수는 (1) 유틸리티 클래스로 자동 생성되고, (2) 동시에 `:root`의 CSS 변수로도 노출된다. 일반 CSS에서 `var(--color-brand-500)`으로도 쓸 수 있다.

| 변수 접두사 | 생성되는 유틸리티 |
|---|---|
| `--color-*` | `bg-*` `text-*` `border-*` `fill-*` 등 |
| `--font-*` | `font-*` |
| `--spacing` | `p-*` `m-*` `gap-*` 등의 기준 단위 |
| `--breakpoint-*` | `sm:` `md:` 반응형 접두사 |
| `--container-*` | `max-w-*`, 컨테이너 쿼리 `@md:` |
| `--radius-*` / `--shadow-*` | `rounded-*` / `shadow-*` |
| `--ease-*`, `--animate-*` | `ease-*`, `animate-*` |

기본값을 지우고 우리 팔레트만 쓰려면:

```css
@theme {
  --color-*: initial;          /* 기본 팔레트 전부 제거 */
  --color-white: #fff;
  --color-brand-500: #0066cc;
}
```

디자인 시스템을 엄격히 강제하려는 조직에서 유용하다.

### `@utility` — 커스텀 유틸리티

```css
@utility scrollbar-hidden {
  scrollbar-width: none;
  &::-webkit-scrollbar { display: none; }
}

@utility tab-* {              /* 함수형 유틸리티: tab-2, tab-4 ... */
  tab-size: --value(integer);
}
```

`@utility`로 만든 것은 `hover:scrollbar-hidden`처럼 variant와도 조합된다. 일반 CSS 클래스로 만들면 이게 안 된다.

### `@custom-variant` — 커스텀 조건

```css
@custom-variant dark (&:where(.dark, .dark *));   /* class 기반 다크모드 (자세한 건 [[Tailwind Dark Mode]]) */
@custom-variant loading (&[wire\:loading]);        /* Livewire 로딩 상태 */
```

### `@plugin`, `@source`, `@config`

```css
@plugin "@tailwindcss/forms";
@plugin "@tailwindcss/typography";

@source "../../packages/ui/src/**/*.blade.php";   /* 스캔 경로 추가 */
@source not "../legacy";                          /* 특정 경로 제외 */
@source inline("bg-red-100 bg-yellow-100 bg-green-100 text-red-700 text-yellow-700 text-green-700");  /* 안전목록 */

@config "../../tailwind.config.js";  /* v3 스타일 JS 설정 파일을 계속 쓰고 싶을 때 */
```

`@tailwindcss/typography`는 CMS/에디터 HTML(기사 본문 등)에 `prose` 클래스 하나로 읽기 좋은 타이포그래피를 적용한다.

```blade
<div class="prose prose-lg max-w-none prose-headings:font-semibold">
  {!! $article->body !!}
</div>
```

### `@apply`와 `@layer`

`@apply`는 유틸리티를 CSS 규칙 안으로 끌어온다. 동작은 하지만 기본 전략으로 삼지 않는 편이 좋다 (이유는 [[Tailwind Component Strategy]]). 서드파티 마크업이나 `body`/`a` 같은 기본 요소 스타일에는 합리적이다.

```css
@layer base {
  body { @apply bg-gray-50 text-gray-900 antialiased; }
  a    { @apply text-blue-600 underline-offset-2 hover:underline; }
}
```

## 주의사항 / 안티패턴

- `@apply`로 `.btn-primary` 같은 컴포넌트 클래스를 대량으로 만들면 예전 CSS 프레임워크로 회귀한다 — [[Tailwind Component Strategy]] 참고.
- Vue/Svelte `<style>` 블록이나 CSS Module에서 `@apply`를 쓰려면 파일 상단에 `@reference "../app.css";`가 필요하다.

## 참고

- [[Tailwind Installation]] — 설치 시 기본 진입점 설정
- [[Tailwind Dynamic Class Pitfall]] — `@source`/`@source inline`으로 스캔 누락 방지
- [[Tailwind Dark Mode]] — `@custom-variant dark`를 활용한 다크모드
- [[Tailwind Utility Syntax]] — 임의 값을 토큰으로 승격하는 이유
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`
