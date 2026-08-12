---
title: Blade Style Directives
category: frontend
tags: [frontend, blade, laravel, css, vite]
related: [[Tailwind Component Strategy]], [[Tailwind Installation]], [[Blade Component Attributes]]
---

# Blade Style Directives

에셋 로더 `@vite`와, 조건부 인라인 스타일/클래스를 위한 `@style`/`@class`.

## 핵심 개념

### `@vite`

```blade
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

- 구버전 Laravel Mix의 `mix()`를 대체하는 에셋 로더.
- `vite.config.js`의 `laravel({ input: [...] })`에 경로를 등록해 두어야 한다.
- **dev/prod를 자동 판별**: dev면 Vite dev 서버 경로를, prod면 `public/build/manifest.json`을 참조해 해시된 빌드 파일을 가져온다. 설정 상세는 [[Tailwind Installation]] 참고.

### `@style` — 조건부 인라인 스타일

boolean 값에 따라 스타일을 붙인다.

```blade
<div @style([
    'background-color: red' => $isCritical,
    'font-weight: bold'     => true,
    'color: gray'           => !$isCritical,
])>
    위험 알림
</div>
```

### `@class` — 조건부 CSS 클래스

`@style`의 class 버전. 문자열만 있으면 무조건 추가, `'클래스' => 조건` 형태면 조건부 추가.

```blade
<div @class([
    'p-4',                          {{-- 무조건 추가 --}}
    'font-bold'     => $isActive,
    'text-red-500'  => $isError,
])>
    내용
</div>
```

## Laravel 구현

`@class`는 Tailwind 유틸리티 클래스를 조건부로 조합할 때 특히 자주 쓰인다 — [[Tailwind Component Strategy]]의 `Illuminate\Support\Arr::toCssClasses()` 기반 배지/알림 예제가 동일한 메커니즘이다.

```blade
<div @class([
    'rounded-md px-4 py-3 text-sm',
    'bg-green-50 text-green-800 border border-green-200' => $type === 'success',
    'bg-red-50 text-red-800 border border-red-200'       => $type === 'error',
])>
    {{ $message }}
</div>
```

컴포넌트 내부에서 `@class`와 [[Blade Component Attributes]]의 `$attributes->merge()`를 함께 쓸 때는, `@class`로 만든 결과 문자열을 `merge(['class' => ...])`에 넘기는 조합이 흔하다.

## 주의사항 / 안티패턴

- `@class`/`@style`은 클래스명이 **정적 문자열**이어야 스캐너가 인식한다 — 배열 키 자체를 동적으로 조합하면 Tailwind의 동적 클래스 함정에 그대로 걸린다. 자세한 건 [[Tailwind Dynamic Class Pitfall]] 참고.
- `@vite()`에 등록하지 않은 CSS/JS 파일은 아무리 `<link>`/`<script>`를 직접 써도 HMR이나 해시 캐시 무효화 대상이 되지 않는다.

## 참고

- [[Tailwind Component Strategy]] — `@class` 기반 조건부 클래스 조합 실전 예제
- [[Tailwind Installation]] — `@vite` 설정과 빌드 파이프라인
- [[Blade Component Attributes]] — `$attributes->merge()`와의 조합
- 소스: `2026-08-12_Laravel Blade 상호작용 가이드.md`
