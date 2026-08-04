---
title: Tailwind Installation
category: frontend
tags: [frontend, tailwind, vite, setup]
related: [[Tailwind CSS]], [[Directory Structure]], [[CI-CD Pipeline]]
---

# Tailwind Installation

Laravel + Vite 조합을 기준으로 한 Tailwind CSS v4 설치 절차. v4는 `tailwind.config.js` 없이 CSS 파일에서 바로 설정하는 CSS-first 방식이다 (자세한 설정은 [[Tailwind Theme Configuration]]).

## 핵심 개념

- v4는 `@tailwindcss/vite` 플러그인 하나로 Vite와 통합된다. PostCSS/autoprefixer/postcss-import가 내장되어 별도 설치가 필요 없다(v3와의 차이는 [[Tailwind v3 to v4 Migration]] 참고).
- 진입점은 `@import "tailwindcss";` 한 줄이며, 스캔 대상 경로는 자동 감지 + `@source` 지시어로 보강한다.
- 빌드 도구가 없는 레거시 PHP 프로젝트에는 `@tailwindcss/cli`를 쓸 수 있다.
- `@tailwindcss/browser`(런타임 CSS 생성)는 프로토타이핑 전용이며 프로덕션에 쓰면 안 된다.

## Laravel 구현

```bash
npm install tailwindcss @tailwindcss/vite
```

```js
// vite.config.js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
        tailwindcss(),
    ],
});
```

```css
/* resources/css/app.css */
@import "tailwindcss";

/* Blade 템플릿과 페이지네이션 뷰까지 스캔 대상으로 지정 */
@source "../../vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php";
@source "../../storage/framework/views/*.php";
@source "../**/*.blade.php";
@source "../**/*.js";
```

```blade
{{-- 레이아웃 Blade --}}
<!doctype html>
<html lang="ko">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body class="bg-gray-50 text-gray-900">
    <h1 class="text-3xl font-bold underline">Hello</h1>
</body>
</html>
```

```bash
npm run dev     # 개발 (HMR)
npm run build   # 프로덕션 빌드
```

### 다른 환경

| 환경 | 방법 |
|---|---|
| 순수 Vite (React/Vue) | `@tailwindcss/vite` 플러그인, `@import "tailwindcss";` |
| PostCSS 파이프라인 (Webpack, Next.js) | `@tailwindcss/postcss` |
| Webpack 전용 (v4.2+) | `@tailwindcss/webpack` |
| 빌드 도구 없는 레거시 PHP | `npx @tailwindcss/cli -i ./src/input.css -o ./dist/output.css --watch`, 결과 CSS 파일만 배포 |

### 에디터 설정 (거의 필수)

- **Tailwind CSS IntelliSense**: 자동완성, 클래스 hover 시 실제 CSS 표시, 오타 경고. PhpStorm은 내장 지원.
- **prettier-plugin-tailwindcss**: 클래스 순서 자동 정렬 (팀 협업 관점은 [[Tailwind Component Strategy]] 참고).

## 주의사항 / 안티패턴

- Laravel installer가 만든 프로젝트에 v3 잔재(`tailwind.config.js`, `postcss.config.js`의 `tailwindcss`/`autoprefixer`, `app.css`의 `@tailwind base;` 3줄)가 남아 있으면 v4와 충돌한다. `npm ls tailwindcss`로 버전을 먼저 확인할 것.
- `php artisan view:cache`를 쓰는 프로젝트는 `storage/framework/views/*.php`도 `@source`에 포함해야 캐시된 뷰의 클래스가 빌드에서 누락되지 않는다.
- 배포 파이프라인에서 `npm run build`를 빠뜨리면 스타일 없는 페이지가 배포된다 — [[Tailwind Build and Performance]]의 배포 체크리스트 참고.

## 참고

- [[Tailwind CSS]] — 개요
- [[Tailwind Theme Configuration]] — `@theme`/`@source`/`@plugin` 설정
- [[Tailwind Build and Performance]] — 배포 체크리스트
- [[Tailwind v3 to v4 Migration]] — 마이그레이션 시 주의점
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`
