---
title: Blade Component Basics
category: frontend
tags: [frontend, blade, laravel, component]
related: [[Blade Component Attributes]], [[Tailwind Component Strategy]], [[Blade Template Inheritance]]
---

# Blade Component Basics

`@props`로 데이터를 선언하고 `$slot`/named slot으로 HTML 덩어리를 전달하며, `@aware`로 부모 컴포넌트의 props를 상속하는 방법.

## 핵심 개념

```bash
php artisan make:component Alert
# → app/View/Components/Alert.php
# → resources/views/components/alert.blade.php
```

```blade
<x-alert type="danger" :data="$data" />
<x-alert> 내용이 있는 형태 </x-alert>
```

### `@props`

컴포넌트 blade **최상단**에서 받을 데이터를 선언한다. 기본값 지정 가능.

```blade
{{-- resources/views/components/alert.blade.php --}}
@props([
    'type' => 'info',   {{-- 기본값 있음 --}}
    'title',            {{-- 기본값 없음 (필수처럼 취급) --}}
])

<div {{ $attributes->merge(['class' => "alert alert-{$type}"]) }}>
    <strong>{{ $title }}</strong>
    {{ $slot }}
</div>
```

> [!important] `@props`에 **선언된** 이름은 변수(`$type`)로 들어오고 **`$attributes`에서는 제거**된다. `@props`에 **선언되지 않은** 나머지 속성만 `$attributes`에 남는다 (자세한 건 [[Blade Component Attributes]]).

### `$slot` — 기본 슬롯

**큰 HTML 덩어리**를 통째로 받아 넣는 용도.

```blade
{{-- 호출부 --}}
<x-child><b>이 부분</b>이 $slot을 대체합니다</x-child>
```

```blade
{{-- components/child.blade.php --}}
<div>{{ $slot }}</div>
```

### named slot — 여러 덩어리 받기

`<x-slot:var>`로 정의한 내용은 컴포넌트 안에서 `$var`가 된다.

```blade
{{-- 호출부 --}}
<x-child>
    <x-slot:title>
        <h3>알림 메시지</h3>
    </x-slot:title>

    <x-slot:body>
        본문 내용입니다.
    </x-slot:body>
</x-child>
```

```blade
{{-- components/child.blade.php --}}
<div>
    <div>{{ $title }}</div>
    <div>{{ $body }}</div>
</div>
```

### `@aware` — 부모 컴포넌트의 props 상속

부모가 컴포넌트일 때, 부모의 `@props`에 정의된 값을 **자식에게 명시적으로 넘기지 않아도** 자식이 복사해 온다. 상위 클래스 프로퍼티 참조와 비슷하지만 **부모 값을 바꿀 수는 없다**(읽기 전용 복사).

```blade
{{-- 부모: components/menu.blade.php --}}
@props(['color' => 'gray'])
<ul {{ $attributes }}>{{ $slot }}</ul>
```

```blade
{{-- 자식: components/menu/item.blade.php --}}
@aware(['color' => 'gray'])
<li class="text-{{ $color }}-600">{{ $slot }}</li>
```

```blade
{{-- 사용처: item마다 color를 적어줄 필요가 없다 --}}
<x-menu color="purple">
    <x-menu.item>홈</x-menu.item>
    <x-menu.item>검색</x-menu.item>
</x-menu>
```

## Laravel 구현

[[Tailwind Component Strategy]]에서 다루는 variant/size 기반 버튼 컴포넌트는 이 `@props` + `$attributes->merge()` 패턴을 그대로 활용한다. Blade 컴포넌트는 이 위키가 반복 클래스 문제(같은 유틸리티 클래스 조합이 여러 곳에 복사되는 문제)를 해결하는 표준 도구다.

## 주의사항 / 안티패턴

- `@aware`는 부모가 **컴포넌트**일 때만 동작한다. 부모가 일반 Blade include나 뷰라면 값을 명시적으로 넘겨야 한다.
- `@props`에 기본값 없이 선언한 이름(`'title'`)은 필수처럼 취급되지만 실제로 강제되지는 않는다 — 호출부에서 누락하면 `undefined variable` 경고 없이 빈 값으로 렌더링될 수 있으므로, 정말 필수라면 컴포넌트 클래스에서 검증하는 편이 안전하다.

## 참고

- [[Blade Component Attributes]] — `$attributes` 객체의 merge/filter/has/get, `:` prefix 타입 규칙
- [[Tailwind Component Strategy]] — 이 패턴을 활용한 variant 기반 버튼 컴포넌트 예제
- [[Blade Template Inheritance]] — 레이아웃 상속과 컴포넌트의 차이
- 소스: `2026-08-12_Laravel Blade 상호작용 가이드.md`
