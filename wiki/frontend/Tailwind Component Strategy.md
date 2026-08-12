---
title: Tailwind Component Strategy
category: frontend
tags: [frontend, tailwind, blade, component]
related: [[Tailwind CSS]], [[Tailwind Dynamic Class Pitfall]], [[Action Pattern]], [[Blade Component Basics]], [[Blade Component Attributes]], [[Blade Style Directives]]
---

# Tailwind Component Strategy

같은 클래스 조합이 프로젝트 전체에 수십 번 복사되는 문제를, `@apply`가 아니라 **템플릿 컴포넌트**로 푸는 전략.

## 핵심 개념

Tailwind 도입 후 흔히 마주치는 문제: 30개 클래스가 붙은 버튼이 47곳에 복사되어 있으면, 색을 바꾸라는 요청에 47곳을 고쳐야 한다.

`@apply`로 `.btn-primary`를 만들면 전역 클래스, 네이밍 고민, 삭제 못 하는 CSS가 다시 생기며 결국 예전 CSS 프레임워크로 되돌아간다 — Tailwind 제작자도 이 방식을 기본 전략으로 권장하지 않는다. 대신 **템플릿 엔진의 컴포넌트 기능**을 쓴다.

## Laravel 구현

이 패턴이 사용하는 `@props`/`$attributes`/`$slot`의 동작 원리 자체는 [[Blade Component Basics]]와 [[Blade Component Attributes]]에서 다룬다. 여기서는 Tailwind 클래스 관리 관점에 집중한다.

`resources/views/components/button.blade.php`:

```blade
@props([
    'variant' => 'primary',
    'size' => 'md',
    'type' => 'button',
])

@php
    $base = 'inline-flex items-center justify-center gap-2 rounded-md font-medium
             transition-colors focus-visible:outline-2 focus-visible:outline-offset-2
             disabled:pointer-events-none disabled:opacity-50';

    $variants = [
        'primary'   => 'bg-blue-600 text-white shadow-xs hover:bg-blue-700 focus-visible:outline-blue-600',
        'secondary' => 'bg-white text-gray-700 border border-gray-300 hover:bg-gray-50',
        'danger'    => 'bg-red-600 text-white hover:bg-red-700 focus-visible:outline-red-600',
        'ghost'     => 'text-gray-600 hover:bg-gray-100',
    ];

    $sizes = [
        'sm' => 'px-3 py-1.5 text-xs',
        'md' => 'px-4 py-2 text-sm',
        'lg' => 'px-5 py-2.5 text-base',
    ];

    $classes = trim("$base {$variants[$variant]} {$sizes[$size]}");
@endphp

<button type="{{ $type }}" {{ $attributes->class($classes) }}>
    {{ $slot }}
</button>
```

```blade
<x-button>저장</x-button>
<x-button variant="danger" size="sm" wire:click="delete">삭제</x-button>
<x-button variant="secondary" class="w-full">취소</x-button>
```

`$attributes->class(...)`가 컴포넌트 기본 클래스와 호출부에서 넘긴 클래스를 병합한다.

### 조건부 클래스 — Blade `@class`

```blade
<div @class([
    'rounded-md px-4 py-3 text-sm',
    'bg-green-50 text-green-800 border border-green-200' => $type === 'success',
    'bg-red-50 text-red-800 border border-red-200'       => $type === 'error',
    'opacity-60' => $dismissed,
])>
    {{ $message }}
</div>
```

내부적으로 `Illuminate\Support\Arr::toCssClasses()`가 동작하며, 조건이 true인 키만 포함된다. `@class`/`@style` 문법 자체의 전체 레퍼런스는 [[Blade Style Directives]] 참고.

### React/Vue를 쓴다면

```jsx
import clsx from 'clsx';
import { twMerge } from 'tailwind-merge';

const cn = (...inputs) => twMerge(clsx(inputs));

function Button({ variant = 'primary', className, ...props }) {
  return (
    <button
      className={cn(
        'inline-flex items-center rounded-md px-4 py-2 text-sm font-medium',
        variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
        variant === 'danger' && 'bg-red-600 text-white hover:bg-red-700',
        className,
      )}
      {...props}
    />
  );
}
```

`tailwind-merge`는 `px-4`와 `px-6`이 같이 들어오면 뒤엣것만 남긴다. Tailwind는 CSS 우선순위가 클래스 작성 순서가 아니라 CSS 파일 내 순서로 결정되므로, 이 병합 없이는 오버라이드가 의도대로 안 될 때가 있다. Blade에는 이런 라이브러리가 표준화되어 있지 않으므로, 위처럼 **변형을 배열로 관리하고 오버라이드는 최소화**하는 쪽이 안전하다.

### 언제 컴포넌트로 뽑을까

- 같은 클래스 조합이 **3번 이상** 반복되면.
- 디자인 변경 요청이 여러 곳을 동시에 건드릴 것 같으면.
- 반대로, 한 화면에만 있는 레이아웃은 굳이 뽑지 않는다 — 조기 추상화는 비용이다.

## 주의사항 / 안티패턴

- `@apply`는 서드파티 라이브러리가 생성하는 마크업(우리가 못 건드리는 HTML)이나 `body`, `a` 같은 기본 요소 스타일에 한정하는 것이 합리적이다. 기본 전략으로 삼지 말 것.
- Vue/Svelte `<style>` 블록이나 CSS Module에서 `@apply`를 쓰려면 파일 상단에 `@reference "../app.css";`가 필요하다 (별도 컴파일 단위이기 때문).

## 참고

- [[Tailwind CSS]] — 개요, 얻는 것/잃는 것 트레이드오프
- [[Tailwind Dynamic Class Pitfall]] — variant를 배열로 매핑해야 하는 또 다른 이유
- [[Action Pattern]] — Laravel에서 책임을 분리하는 다른 관용구와의 비교
- [[Blade Component Basics]] — `@props`/`$slot`/`@aware` 기본 동작
- [[Blade Component Attributes]] — `$attributes->merge()`, `:` prefix 타입 규칙
- [[Blade Style Directives]] — `@class`/`@style`/`@vite` 전체 레퍼런스
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`, `2026-08-12_Laravel Blade 상호작용 가이드.md`
