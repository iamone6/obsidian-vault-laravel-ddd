---
title: Tailwind Dark Mode
category: frontend
tags: [frontend, tailwind, dark-mode]
related: [[Tailwind Variants]], [[Tailwind Theme Configuration]], [[Tailwind CSS]]
---

# Tailwind Dark Mode

OS 설정 추종 방식부터 사용자 토글 저장, 유지보수하기 쉬운 시맨틱 토큰 전략까지.

## 핵심 개념

### 기본 (OS 설정 추종)

```html
<div class="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100">
```

별도 설정 없이 `prefers-color-scheme`을 따른다.

### 수동 토글 (사용자 선택 저장)

```css
/* app.css */
@custom-variant dark (&:where(.dark, .dark *));
```

```blade
{{-- <head> 안, CSS보다 먼저. 화면 깜빡임(FOUC) 방지용 인라인 스크립트 --}}
<script>
    (function () {
        const stored = localStorage.getItem('theme');
        const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
        if (stored === 'dark' || (!stored && prefersDark)) {
            document.documentElement.classList.add('dark');
        }
    })();
</script>
```

```js
function setTheme(theme) { // 'light' | 'dark' | 'system'
    localStorage.setItem('theme', theme);
    const dark = theme === 'dark'
        || (theme === 'system' && matchMedia('(prefers-color-scheme: dark)').matches);
    document.documentElement.classList.toggle('dark', dark);
}
```

> 이 스크립트는 번들 파일이 아니라 인라인으로, `<head>` 상단에 두는 것이 중요하다. 늦게 실행되면 밝은 화면이 한 프레임 보였다가 어두워진다.

### 유지보수 관점: 시맨틱 토큰

`dark:` 접두사를 수백 군데 붙이는 대신, 의미 기반 토큰을 정의하면 관리가 쉬워진다.

```css
@theme inline {
  --color-surface: var(--surface);
  --color-content: var(--content);
  --color-muted: var(--muted);
}

:root {
  --surface: oklch(1 0 0);
  --content: oklch(0.21 0.01 260);
  --muted:   oklch(0.55 0.01 260);
}

.dark {
  --surface: oklch(0.21 0.01 260);
  --content: oklch(0.98 0 0);
  --muted:   oklch(0.71 0.01 260);
}
```

```html
<div class="bg-surface text-content">
  <p class="text-muted">보조 설명</p>
</div>
```

`@theme inline`은 값을 그대로 인라인해서 다른 CSS 변수를 참조할 수 있게 한다. 이 패턴을 쓰면 다크모드 대응이 **CSS 한 곳**에서 끝난다. 규모가 있는 프로젝트라면 처음부터 이렇게 잡는 편이 낫다.

## 주의사항 / 안티패턴

- 다크모드 초기화 스크립트를 번들 JS에 넣으면 첫 화면 깜빡임(FOUC)이 발생한다. 반드시 인라인 `<script>`로 `<head>` 최상단에 둘 것.
- `dark:` 접두사를 개별 컴포넌트마다 반복하는 방식은 프로젝트가 커지면 관리 비용이 커진다 — 시맨틱 토큰으로 전환을 고려한다.

## 참고

- [[Tailwind Variants]] — `dark:` variant 문법과 조합 순서
- [[Tailwind Theme Configuration]] — `@custom-variant`, `@theme inline`
- [[Tailwind CSS]] — 개요
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`
