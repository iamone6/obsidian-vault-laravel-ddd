---
title: Blade Template Inheritance
category: frontend
tags: [frontend, blade, laravel, view-layer]
related: [[Blade Includes and Loops]], [[Blade Component Basics]], [[Directory Structure]]
---

# Blade Template Inheritance

Controller → Child Blade → Parent Layout으로 이어지는 렌더링 흐름과, parent/child 디렉티브 대응 관계.

## 핵심 개념

Blade의 상속은 "parent가 child를 부른다"가 아니라 **child가 parent를 선언(`@extends`)하고, parent의 빈 자리(`@yield`)를 child가 채운다**는 방향이다. Controller가 호출하는 view는 항상 **child** 쪽이다.

```
Controller --view() 호출·데이터 배열 주입--> Child Blade --@extends('layout')로 상속--> Parent Blade --> 최종 HTML
                                              Child Blade --@section/@push/@prepend로 자리 채움--> Parent Blade
```

```php
return view("pages.search.{$device->value}", [
    'title'        => BuildTitle::get($this),
    'searchResult' => $searchResult,
    'device'       => $device,
]);
```

여기서 넘긴 키(`title`, `searchResult` 등)는 **child blade 안에서 그대로 변수명**(`$title`, `$searchResult`)이 된다. child가 `@extends`로 상속한 parent 레이아웃 안에서도 같은 변수에 접근할 수 있다 — 데이터는 상속 체인 전체에 공유된다.

### Parent ↔ Child 디렉티브 대응 표

| Parent 쪽 | Child 쪽 | 결과 |
|---|---|---|
| `@yield('title')` | `@section('title') … @endsection` 또는 `@section('title', $value)` | `@yield`의 자리를 child의 `@section` 내용이 메운다 |
| `@section('x') … @show` | (대응 없음) | child에 같은 이름의 `@section`이 없으면 **parent의 기본 내용이 그대로 출력** |
| `@section('x') … @show` | `@section('x') … @endsection` | child의 내용이 **parent의 내용을 덮어쓴다** |
| `@section('x') … @show` | `@section('x') … @parent … @endsection` | `@parent` 위치에 **parent의 원래 내용이 삽입**(덮어쓰기가 아니라 합성) |
| `@stack('scripts')` | `@push('scripts') … @endpush` / `@prepend('scripts') … @endprepend` | `@push`는 스택 **뒤**, `@prepend`는 **앞**. 여러 번 호출하면 순서대로 누적 |

## Laravel 구현

### `@yield` ↔ `@section` — 가장 기본

```blade
{{-- layout.blade.php --}}
<html>
<head>
    <title>@yield('title')</title>
    @yield('script')
</head>
<body>
    @yield('content')
</body>
</html>
```

```blade
{{-- pages/search/mobile.blade.php --}}
@extends('layout')

@section('title', $title)   {{-- 한 줄 형태 --}}

@section('content')
    <div>검색 결과 {{ count($searchResult) }}건</div>
@endsection
```

### `@show` vs `@parent` — 기본값 유지와 합성

```blade
{{-- layout.blade.php --}}
@section('sidebar')
    <aside>기본 사이드바</aside>
@show
```

- child에 `@section('sidebar')`가 **없으면** → `기본 사이드바` 출력
- child에 `@section('sidebar') … @endsection`이 **있으면** → child 내용으로 **교체**

```blade
{{-- child: 덮어쓰기 대신 합성 --}}
@section('sidebar')
    @parent                      {{-- 여기에 parent의 "기본 사이드바"가 들어온다 --}}
    <aside>추가 메뉴</aside>
@endsection
```

출력:

```html
<aside>기본 사이드바</aside>
<aside>추가 메뉴</aside>
```

> [!tip] `@show`는 "정의하고 **즉시 그 자리에 출력**", `@endsection`은 "정의만 하고 출력은 `@yield`에 위임". parent에서 기본값을 주려면 `@show`, child에서 값을 채우기만 하려면 `@endsection`.

### `@stack` / `@push` / `@prepend` — 스크립트 누적

```blade
{{-- layout.blade.php --}}
<body>
    @yield('content')
    @stack('scripts')   {{-- 여기에 모든 push/prepend가 모인다 --}}
</body>
```

```blade
{{-- child 또는 include된 partial, component 어디서든 --}}
@push('scripts')
    <script src="/js/chart.js"></script>
@endpush

@push('scripts')
    <script>initChart();</script>   {{-- 위 push 뒤에 붙음 --}}
@endpush

@prepend('scripts')
    <script src="/js/jquery.js"></script>   {{-- 스택 맨 앞으로 --}}
@endprepend
```

최종 순서: `jquery.js` → `chart.js` → `initChart()`.

## 주의사항 / 안티패턴

- `@show`와 `@endsection`을 혼동하면 parent의 기본값이 예상과 다르게 출력되거나 아예 출력되지 않는다. parent 쪽에서 기본값을 두고 싶은지, child가 항상 채워야 하는지부터 결정할 것.
- `@push`/`@prepend`는 어디서 호출하든(child, partial, component) 같은 스택에 누적된다 — 스크립트 로드 순서가 중요한 프로젝트라면 호출 위치를 문서화해 둘 필요가 있다.

## 참고

- [[Blade Includes and Loops]] — `@include`/`@each`로 파일 자체를 끼워넣는 방식과의 차이
- [[Blade Component Basics]] — 컴포넌트 기반 재사용과 상속의 차이
- [[Directory Structure]] — Blade 뷰 파일이 위치하는 Laravel 디렉토리 구조
- 소스: `2026-08-12_Laravel Blade 상호작용 가이드.md`
