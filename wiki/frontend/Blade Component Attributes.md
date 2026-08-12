---
title: Blade Component Attributes
category: frontend
tags: [frontend, blade, laravel, component]
related: [[Blade Component Basics]], [[Tailwind Component Strategy]]
---

# Blade Component Attributes

`$attributes`(`ComponentAttributeBag`)로 컴포넌트 호출부의 나머지 속성을 다루는 방법과, `:` prefix가 결정하는 값 타입 규칙.

## 핵심 개념

`$attributes`는 연관배열처럼 보이지만 실제로는 `Illuminate\View\ComponentAttributeBag` **객체**다. [[Blade Component Basics]]의 `@props`에 **선언되지 않은** 나머지 호출부 속성만 여기 담긴다.

### 기본 — 그대로 펼치기

```blade
{{-- 호출: <x-alert id="error-box" class="mt-4" data-role="alert" /> --}}
<div {{ $attributes }}>
{{-- 결과: <div id="error-box" class="mt-4" data-role="alert"> --}}
```

### `merge()` — 컴포넌트 기본 속성과 병합

```blade
<div {{ $attributes->merge(['class' => 'alert alert-info']) }}>
```

`class`와 `style`은 **값이 이어붙고**, 나머지 속성은 호출부 값이 우선한다.

### `filter()` — 속성 골라내기

```blade
<div {{ $attributes->filter(fn ($value, $key) => str_starts_with($key, 'data-')) }}>

{{-- 축약형도 있다 --}}
<div {{ $attributes->whereStartsWith('data-') }}>
<div {{ $attributes->whereDoesntStartWith('data-') }}>
```

### `has()` / `get()` — 존재 확인과 기본값

```blade
@if ($attributes->has('onclick'))
    <p>이 버튼은 클릭 이벤트가 설정되어 있습니다.</p>
@endif

{{ $attributes->get('id', 'default-id') }}   {{-- id가 없으면 기본값 출력 --}}
```

## Laravel 구현

### 컴포넌트로 값 전달할 때의 타입 규칙

`:` prefix 유무가 **"문자열 리터럴이냐, PHP 표현식이냐"**를 가른다.

| 호출부 표기 | 컴포넌트가 받는 값 | 설명 |
|---|---|---|
| `type="danger"` | `(string) "danger"` | PHP 파서 개입 없이 정적 문자열 그대로 전달 |
| `:type="'danger'"` | `(string) "danger"` | PHP 파서가 표현식 `'danger'`를 평가해 전달 |
| `:type="$strDanger"` | `(string) "danger"` | 변수 값 전달 |
| `count="5"` | `(string) "5"` | 따옴표 안의 값은 **문자열** |
| `:count="5"` | `(int) 5` | PHP 파서가 정수 리터럴로 평가 |
| `:items="$collection"` | `Collection` 객체 | 배열/객체/컬렉션도 그대로 전달 가능 |
| `:is-active="$user->isAdmin()"` | `(bool)` | 메서드 호출 결과도 평가 |

```blade
@php $strDanger = 'danger'; @endphp

<x-component
    type="danger"          {{-- (string) "danger" --}}
    :type2="$strDanger"    {{-- (string) "danger" --}}
    strcount="5"           {{-- (string) "5"      --}}
    :numcount="5"          {{-- (int) 5           --}}
/>
```

```blade
{{-- 받는 쪽 --}}
@props(['type' => 'default', 'strcount', 'numcount'])
```

## 주의사항 / 안티패턴

> [!warning] 자주 나오는 실수
> `count="5"`로 넘기고 컴포넌트 안에서 `$count === 5`로 비교하면 **false**다(`"5" !== 5`). 숫자·불리언·배열은 `:`를 붙여 넘긴다.
> 특히 `active="false"`는 **비어있지 않은 문자열**이라 `if ($active)`에서 true가 된다. `:active="false"`로 써야 한다.

- `merge()`는 `class`/`style`만 값을 이어붙이고 그 외 속성(예: `id`)은 호출부 값으로 완전히 대체된다는 점에 주의한다.
- [[Tailwind Component Strategy]]의 variant 배열 패턴과 `$attributes->class()`를 함께 쓸 때, `class` 병합 순서(컴포넌트 기본 → 호출부 오버라이드)가 CSS 우선순위와 어긋나지 않는지 확인한다.

## 참고

- [[Blade Component Basics]] — `@props` 선언과 `$slot`
- [[Tailwind Component Strategy]] — `$attributes->class()`를 활용한 variant 컴포넌트 예제
- 소스: `2026-08-12_Laravel Blade 상호작용 가이드.md`
