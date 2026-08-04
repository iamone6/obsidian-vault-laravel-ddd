---
title: Laravel Data (spatie/laravel-data)
category: laravel
tags: [laravel, dto, validation, data-transfer]
related: [[DTO]], [[Form Request]], [[Policy and Gate]]
---

# Laravel Data (spatie/laravel-data)

`spatie/laravel-data`는 **DTO + FormRequest(검증) + API Resource(응답 변환)**를 하나의 `Data` 클래스로 통합해주는 패키지다. "데이터 구조를 한 번만 정의하면 된다"가 핵심 철학이며, [[DTO]] 문서에서 이미 소개된 패키지를 검증(validation) 관점에서 더 깊이 다룬다.

## 핵심 개념

- `Spatie\LaravelData\Data`를 상속하고 생성자 프로퍼티 승격으로 필드를 선언한다.
- **프로퍼티 타입만으로 검증 규칙이 자동 추론**된다 (`string` → `required, string`, `?int` → `nullable, integer` 등).
- 컨트롤러 인자로 타입힌트하면 [[Form Request]]처럼 **자동 검증 후 채워진 객체가 주입**된다.
- 부족한 제약은 Attribute(`#[Max(100)]`, `#[Email]` 등)로 선언적으로 추가한다 — `FormRequest::rules()` 배열을 대체하는 방식.
- **인가(authorize) 기능은 없다.** Data 클래스는 형식 검증만 담당하고, 권한 확인은 여전히 [[Policy and Gate]]나 미들웨어의 몫이다.

## Laravel 구현

```php
<?php

namespace App\Data;

use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\Validation\Email;
use Spatie\LaravelData\Attributes\Validation\Max;
use Spatie\LaravelData\Attributes\Validation\Min;

// Data 를 상속받으면 DTO + Request 검증을 동시에 지원하는 클래스가 된다.
class SongData extends Data
{
    public function __construct(
        #[Min(2), Max(100)] // required, string 은 타입에서 자동 추론되고, min/max 만 Attribute 로 추가
        public readonly string $title,

        public readonly string $artist,

        #[Email] // 형식 검증만 Attribute 로 선언
        public readonly ?string $contactEmail,
    ) {
    }
}
```

```php
class UpdateSongController
{
    // SongData 를 타입힌트하는 순간: 규칙 생성 -> 현재 Request 검증 -> 통과 시 객체 생성/주입 이 자동으로 일어난다.
    // 실패하면 FormRequest 를 쓸 때와 동일하게 Illuminate\Validation\ValidationException 이 던져진다.
    public function __invoke(Song $model, SongData $data): RedirectResponse
    {
        $model->update($data->all());

        return redirect()->back();
    }
}
```

기존 `FormRequest`를 인가(authorize)에만 남겨두고, 검증/DTO 변환만 `Data`에 위임하는 조합도 흔하다.

```php
class UpdateSongController
{
    // authorize() 는 SongRequest(FormRequest)가 담당, 검증 규칙/DTO 변환은 SongData가 담당
    public function __invoke(Song $model, SongRequest $request): RedirectResponse
    {
        $model->update(SongData::from($request)->all());

        return redirect()->back();
    }
}
```

## 예외 처리 (검증 실패 시)

laravel-data는 내부적으로 Laravel 표준 `Validator`를 그대로 사용하므로, 검증 실패 시 던져지는 예외도 익숙한 `Illuminate\Validation\ValidationException`이다. 별도 처리를 하지 않아도 [[Form Request]]와 동일하게 동작한다 — 웹 요청은 이전 페이지로 리다이렉트(+ 세션 에러), API 요청(JSON 기대)은 `422` + `{"message", "errors"}` 바디가 자동 반환된다.

`FormRequest`의 `messages()` / `attributes()`를 오버라이드하던 습관 그대로, Data 클래스에도 동일한 이름의 static 메서드를 둘 수 있다.

```php
class SongData extends Data
{
    public function __construct(
        public string $title,
    ) {
    }

    // FormRequest::messages() 와 동일한 역할
    public static function messages(): array
    {
        return ['title.required' => '제목은 필수 입력 항목입니다.'];
    }

    // FormRequest::attributes() 와 동일한 역할
    public static function attributes(): array
    {
        return ['title' => '노래 제목'];
    }
}
```

배열을 직접 검증해야 하는 서비스/커맨드 계층에서는 `from()` 대신 `validateAndCreate()`를 명시적으로 호출하고 직접 `try/catch` 한다 (기본 전략은 "요청으로부터 생성할 때만 검증"이기 때문).

```php
try {
    $song = SongData::validateAndCreate($payload);
} catch (\Illuminate\Validation\ValidationException $e) {
    return response()->json(['errors' => $e->errors()], 422);
}
```

API 전체의 에러 응답 포맷을 통일하려면, laravel-data 전용 설정이 아니라 Laravel 표준대로 `bootstrap/app.php`의 `withExceptions()`에서 `ValidationException` 렌더링을 한 번만 오버라이드하면 모든 Data 클래스에 동일하게 적용된다.

## 주의사항 / 안티패턴

- **`authorize()` 없음**: Data 클래스로 [[Form Request]]를 완전히 대체하려 하면 인가 로직이 빠지기 쉽다. 인가는 반드시 [[Policy and Gate]]나 미들웨어에 남겨둘 것.
- **`rules()`를 직접 정의하면 자동 타입 추론이 꺼진다**: `rules()`에서 `'title' => ['max:20']`만 쓰면 `required`, `string`이 자동으로 붙지 않는다. 자동 추론과 병합하려면 클래스에 `#[MergeValidationRules]`를 붙여야 한다.
- **`required`/`string` 같은 기본 규칙을 Attribute로 중복 선언하지 말 것**: 타입에서 이미 추론되므로, Attribute는 "타입만으로 표현 못 하는 제약"에만 사용한다.
- **배열로 `from()` 호출 시 기본적으로 검증되지 않는다**: `config/data.php`의 `validation_strategy` 기본값이 `OnlyRequests`이기 때문. 서비스 계층에서 배열을 검증하려면 `validateAndCreate()`를 명시적으로 사용해야 한다.
- DDD 관점에서는 여전히 **애플리케이션/HTTP 경계의 형식 검증**만 책임지도록 하고, "이 주문을 확정할 수 있는 상태인가?" 같은 도메인 불변식 검증은 도메인 계층의 몫으로 남겨둔다 ([[Form Request]] 문서의 DDD 관점 섹션과 동일한 원칙).

## 참고

- [[DTO]] — Data 클래스의 `Lazy` 속성, DTO/VO 구분 기준
- [[Form Request]] — 인가(authorize) 책임 분리, DDD 관점에서 검증의 위치
- [[Policy and Gate]] — Data 클래스가 대체하지 못하는 인가 로직
- 소스: `2026-07-30_Spatie Laravel Data 공식 문서 종합.md` (GitHub 소스, 공식 문서 v4, byzz 블로그 종합)
