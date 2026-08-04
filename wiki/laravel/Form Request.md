---
title: Form Request
category: laravel
tags: [laravel, form-request, validation, authorization]
related: [[DTO]], [[Application Service]], [[Policy and Gate]], [[Laravel Data]]
---

# Form Request

컨트롤러 메서드에서 반복되는 "입력 유효성 검사 + 인가 확인 + 실패 시 리다이렉트/거부" 패턴을 별도 클래스로 추출한 커스텀 Request 객체.

## 핵심 개념

컨트롤러 메서드들은 대개 다음 패턴을 반복한다: 사용자 입력 검증 → 인증/권한 확인 → 필요 시 리다이렉트. Form Request는 이 공통 행위를 표준화한 구조다.

`php artisan make:request CreateCommentRequest`로 생성하면 `app/Http/Requests/`에 파일이 만들어지며, 모든 Form Request는 `public` 메서드 2개를 가진다.

- `rules()` — 유효성 검증 규칙 배열 반환
- `authorize()` — 현재 사용자가 이 요청을 수행할 권한이 있는지 `true`/`false` 반환. `false`면 403 Forbidden.

## Laravel 구현

```php
class CreateCommentRequest extends FormRequest
{
    public function authorize()
    {
        $blogPostId = $this->route('blogPost');

        return auth()->check() && BlogPost::where('id', $blogPostId)
            ->where('user_id', auth()->id())
            ->exists();
    }

    public function rules()
    {
        return [
            'body' => 'required|max:1000',
        ];
    }
}
```

컨트롤러/라우트 클로저에서는 타입힌트만으로 사용한다. 나머지는 라라벨이 내부적으로 처리한다.

```php
Route::post('comments', function (App\Http\Requests\CreateCommentRequest $request) {
    // 이 지점에 도달했다면 이미 유효성 검증과 인가를 통과한 것이다.
});
```

- 유효성 검증 실패 → 오류 메시지와 함께 이전 페이지로 리다이렉트 (Request 객체의 `validate()`를 호출한 것과 동일한 동작)
- 인가 실패(`authorize()` → `false`) → 403 Forbidden, 컨트롤러 코드 실행 자체가 안 됨

## DDD 관점에서의 활용

Form Request는 어디까지나 **HTTP 경계의 입력 검증/인가**를 위한 것이며, 비즈니스 규칙 자체를 담는 곳이 아니다.

- `rules()`는 형식적 유효성(타입, 길이, 필수 여부)만 검증한다. "이 주문을 확정할 수 있는 상태인가?" 같은 도메인 불변식 검증은 도메인 계층([[Aggregate]], 도메인 서비스)의 책임.
- `authorize()`의 단순 ACL 성격 권한 확인은 [[Policy and Gate]]와 겹치는 영역이다 — 복잡한 인가 규칙은 Gate/Policy로 분리하고 Form Request에서는 이를 호출하는 것으로 충분하다.
- 검증을 통과한 입력은 [[DTO]]로 변환해 Application Service에 전달하는 것이 좋다. Form Request 자체를 Application Service까지 전달하지 않는다(HTTP 계층 객체가 도메인/애플리케이션 계층으로 새는 것을 방지).

```php
public function store(CreateCommentRequest $request, CreateCommentService $service): RedirectResponse
{
    $service->handle(CreateCommentData::from($request->validated()));
    return redirect()->back();
}
```

## 주의사항 / 안티패턴

- Form Request 안에서 Eloquent 모델을 직접 저장하는 로직을 넣지 말 것 — 검증/인가 책임만 갖는다.
- `$request->all()`을 그대로 Eloquent `create()`에 넘기는 대량 할당은 필드 화이트리스트가 없으면 위험하다. `rules()`로 검증된 필드만 사용.

## `spatie/laravel-data`로 `rules()` 대체하기

[[Laravel Data]] 패키지의 `Data` 클래스를 쓰면 `rules()`를 배열로 작성하는 대신, 프로퍼티 타입 자동 추론 + `#[Max(100)]` 같은 Attribute로 검증 규칙을 선언할 수 있다. 검증 실패 시 예외(`ValidationException`)와 기본 응답 방식(리다이렉트/422 JSON)은 Form Request와 동일하다.

단, **`Data` 클래스는 `authorize()`에 해당하는 인가 기능이 없다.** 인가 로직이 필요한 엔드포인트는 `authorize()`만 담당하는 얇은 Form Request를 남겨두고, 검증/DTO 변환은 `Data::from($request)`에 위임하는 조합이 실용적이다.

```php
class UpdateSongController
{
    // authorize() 는 SongRequest(FormRequest)가 담당, 검증 규칙/DTO 변환은 SongData(Data)가 담당
    public function __invoke(Song $model, SongRequest $request): RedirectResponse
    {
        $model->update(SongData::from($request)->all());

        return redirect()->back();
    }
}
```

## 참고

- [[DTO]] — 검증된 입력을 애플리케이션 계층으로 전달하는 형태
- [[Laravel Data]] — Attribute 기반 검증으로 `rules()`를 대체하는 방법과 예외 처리 상세
- [[Policy and Gate]] — 재사용 가능한 인가 규칙
- [[Laravel MCP]] — MCP Tool의 `$request->validate()`도 같은 "HTTP 경계 검증" 역할을 AI 클라이언트 입력에 대해 수행하는 유사 패턴 (별도 클래스 계층, Form Request 자체를 재사용하진 않음)
- 소스: 처음부터 제대로 배우는 라라벨, 7장 사용자 데이터의 조회 및 처리 / `2026-07-30_Spatie Laravel Data 공식 문서 종합.md`
