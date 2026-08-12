---
title: Blade Includes and Loops
category: frontend
tags: [frontend, blade, laravel, loop]
related: [[Blade Template Inheritance]], [[Blade Conditionals and Environment]]
---

# Blade Includes and Loops

파일을 그 자리에 끼워넣는 `@include` 계열, 배열만큼 반복하는 `@each`, 일반 반복문과 `$loop` 객체, 중복 방지용 `@once`.

## 핵심 개념

`@include`/`@each`는 [[Blade Template Inheritance]]의 `@yield`/`@section`과 달리 **레이아웃 상속이 아니라 파일을 그 자리에 그대로 렌더링**하는 방식이다.

| 디렉티브 | 동작 |
|---|---|
| `@include(file, [k => v])` | 해당 blade 파일을 그 자리에 렌더링. `[k => v]`로 값 전달 |
| `@includeIf(file, data)` | 파일이 없으면 조용히 스킵 |
| `@includeWhen($cond, file, data)` | 조건이 true일 때만 |
| `@includeUnless($cond, file, data)` | 조건이 false일 때만 |
| `@each(file, $array, 'var', empty?)` | `$array`의 각 요소가 순차적으로 `$var`에 대입되어 반복 렌더링. 4번째 인자는 배열이 비었을 때 렌더링할 뷰(선택) |

## Laravel 구현

```blade
@include('partials.card', ['item' => $product, 'showPrice' => true])

@includeIf('partials.banner', ['ad' => $ad])
@includeWhen($user->isAdmin(), 'partials.admin-tools')
@includeUnless($user->isAdmin(), 'partials.notice')
```

```blade
@each('partials.row', $searchResult, 'row', 'partials.empty')
```

### 일반 반복문

```blade
@foreach ($items as $item)
    {{ $item->name }}
@endforeach

@forelse ($items as $item)
    {{ $item->name }}
@empty
    {{-- $items가 비었을 때만 실행 --}}
    항목이 없습니다.
@endforelse
```

### `$loop` 객체

반복문 안에서 자동으로 생성되는 객체.

| 프로퍼티 | 의미 |
|---|---|
| `$loop->index` | 인덱스, `0`부터 |
| `$loop->iteration` | 몇 번째 반복인지(표시용), `1`부터 |
| `$loop->remaining` | 앞으로 남은 횟수 |
| `$loop->count` | 총 반복 횟수 |
| `$loop->first` / `$loop->last` | 첫/마지막 반복이면 `true` |
| `$loop->even` / `$loop->odd` | 짝수/홀수 번째면 `true` |
| `$loop->depth` | 중첩 깊이 |
| `$loop->parent` | 중첩 반복문에서 **상위 루프의 `$loop`** |

```blade
@foreach ($categories as $category)
    @foreach ($category->items as $item)
        <tr class="{{ $loop->even ? 'bg-gray-50' : '' }}">
            <td>{{ $loop->parent->iteration }}-{{ $loop->iteration }}</td>
            <td>{{ $item->name }}</td>
        </tr>
        @if ($loop->last)
            <tr><td colspan="2">총 {{ $loop->count }}건</td></tr>
        @endif
    @endforeach
@endforeach
```

### `@once` — 반복/재사용 컴포넌트에서 한 번만 실행

```blade
@once
    @push('scripts')
        <script>console.log('이 스크립트는 페이지당 한 번만 로드됩니다.');</script>
    @endpush
@endonce
```

컴포넌트가 여러 번 호출되어도 의존 JS/CSS는 한 번만 로드하고 싶을 때 쓴다.

## 주의사항 / 안티패턴

- `@include`는 컴파일 시점에 텍스트가 삽입되는 게 아니라, 렌더링 시점마다 `$__env->make(...)->render()`를 호출해 대상 뷰를 매번 resolve·render하는 방식으로 동작한다. 같은 partial을 수백 번 `@each`로 반복하면 그만큼 렌더 호출이 누적되어 성능에 영향을 줄 수 있다 — 반복 항목이 컴포넌트 수준으로 무거워지면 [[Blade Component Basics]]로 옮기는 걸 고려한다.
- `@once`를 빠뜨리고 반복문 안에서 `@push`를 그대로 쓰면 같은 스크립트가 반복 횟수만큼 중복 삽입된다.

## 참고

- [[Blade Template Inheritance]] — `@yield`/`@section` 기반 레이아웃 상속과의 차이
- [[Blade Conditionals and Environment]] — `@if`/`@auth` 등 분기 디렉티브
- 소스: `2026-08-12_Laravel Blade 상호작용 가이드.md`
