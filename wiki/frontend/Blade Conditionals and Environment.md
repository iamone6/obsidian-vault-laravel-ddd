---
title: Blade Conditionals and Environment
category: frontend
tags: [frontend, blade, laravel, debugging]
related: [[Blade Includes and Loops]], [[Blade Template Inheritance]], [[Form Request]]
---

# Blade Conditionals and Environment

조건 분기, 인증 분기, 폼 관련 디렉티브, PHP→JS 데이터 전달, 디버깅, 환경 분기.

## 핵심 개념

### 조건문

```blade
@if ($count > 10)
    많음
@elseif ($count > 0)
    적음
@else
    없음
@endif

@isset($searchParams['q'])
    검색어: {{ $searchParams['q'] }}
@endisset

@empty($searchResult)
    결과가 없습니다.
@endempty
```

### 인증 분기

```blade
@auth
    {{ auth()->user()->name }} 님 로그인 중
@endauth

@guest
    로그인이 필요합니다.
@endguest
```

### 환경 분기

`.env`의 `APP_ENV` 값(`app()->environment()`)으로 분기한다.

```blade
@env('local')
    로컬 환경 전용 디버그 툴바
@endenv

@env(['staging', 'production'])
    알파 또는 프로덕션 공통 스크립트
@endenv

@production
    <script src="https://analytics.example.com/tag.js"></script>
@endproduction
```

`@production`은 `@env('production')`의 축약형이다.

## Laravel 구현

### 폼 관련

```blade
<form method="POST" action="{{ route('products.update', $product) }}">
    @csrf
    @method('PUT')   {{-- PATCH / DELETE 도 동일 --}}
    ...
</form>
```

HTML `<form>`은 `GET`/`POST`만 지원하므로, `@method`가 hidden input(`_method`)을 만들어 라우터가 PUT/PATCH/DELETE로 인식하게 한다. 반드시 `<form>` **안쪽**에 적는다. `@csrf`가 만드는 토큰은 [[Form Request]]로 넘어오는 요청의 CSRF 미들웨어 검증과 짝을 이룬다.

### PHP 배열 → JS

```blade
<script>
    const data = @json($searchResult);
    const params = @json($searchParams, JSON_PRETTY_PRINT);
</script>
```

### 디버깅

| 디렉티브 | 동작 |
|---|---|
| `@dd($var)` | `@php dd($var); @endphp`와 동일. 출력 후 **실행 중단** |
| `@dump($var)` | 출력하고 **계속 실행** |

## 주의사항 / 안티패턴

- `@method`를 `<form>` 태그 바깥에 두면 hidden input이 생성되어도 라우터가 인식하지 못한다.
- `@dd`는 실행을 중단시키므로 프로덕션 코드에 남기면 페이지 전체가 멈춘다 — 커밋 전 제거를 습관화할 것.
- `@json`은 내부적으로 `Illuminate\Support\Js::from()`을 사용해 `JSON_HEX_TAG`/`JSON_HEX_AMP`/`JSON_HEX_APOS`/`JSON_HEX_QUOT` 플래그로 인코딩하므로, `<script>` 태그 안에 삽입해도 `</script>`로 컨텍스트를 탈출하거나 따옴표를 깨는 공격은 기본적으로 막힌다. 다만 그렇게 만든 JS 값을 다시 `innerHTML` 등으로 HTML에 재해석시키는 코드에 넘기면 그 지점에서 별도의 XSS 위험이 생길 수 있다.

## 참고

- [[Blade Includes and Loops]] — 반복문과 `@once`
- [[Blade Template Inheritance]] — 레이아웃 상속 기본 구조
- [[Form Request]] — 서버 검증과 CSRF/HTTP 메서드 처리
- 소스: `2026-08-12_Laravel Blade 상호작용 가이드.md`
