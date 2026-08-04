---
title: Policy and Gate (인가)
category: laravel
tags: [laravel, authorization, gate, policy]
related: [[Form Request]], [[Application Service]], [[Pipeline Pattern]], [[Eloquent Model Attributes]], [[Laravel Data]]
---

# Policy and Gate (인가)

사용자가 특정 동작을 수행할 권한이 있는지 판단하는 라라벨의 인가(Authorization) 시스템. 인증(누구인지 확인)과는 구분되는, "허용된 행위인지" 확인하는 계층이다.

## 핵심 개념

- 인가 규칙을 **어빌리티(ability)**라고 부르며, 문자열 키(`update-contact` 등)와 불리언을 반환하는 클로저/메서드로 구성된다.
- 어빌리티는 `AuthServiceProvider::boot()`에서 `Gate::define()`으로 등록한다.
- 클로저의 첫 번째 인자는 현재 인증된 사용자, 이후 인자는 대상 객체(Eloquent 모델 등)다.

```php
Gate::define('update-contact', function ($user, $contact) {
    return $user->id === $contact->user_id;
});
```

## Laravel 구현

### Gate 퍼사드로 확인하기

```php
if (Gate::denies('update-contact', $contact)) {
    abort(403);
}

if (! Gate::allows('create-contact', Contact::class)) {
    abort(403);
}

// 현재 사용자가 아닌 다른 사용자 기준으로 확인
if (Gate::forUser($user)->denies('create-contact')) {
    abort(403);
}
```

### 리소스 Gate (Policy 클래스)

관례적인 CRUD 어빌리티(`viewAny`, `view`, `create`, `update`, `delete`)를 한 번에 정의하려면 Policy 클래스와 `Gate::resource()`를 사용한다.

```php
Gate::resource('photos', PhotoPolicy::class);
// 아래와 동일하게 photos.viewAny, photos.view, photos.create,
// photos.update, photos.delete 어빌리티가 한 번에 등록된다.
```

### 미들웨어/블레이드에서 사용

```php
Route::get('people/{person}/edit', function () { /* ... */ })
    ->middleware('can:edit,person');
```

```blade
@can('update-contact', $contact)
    <a href="...">수정</a>
@endcan
```

## `#[UsePolicy]` — 모델에 정책 클래스를 직접 부착 (Laravel 12.18+)

모델명과 정책명이 네이밍 규칙(`Post` → `PostPolicy`)에서 벗어난 경우, `Gate`가 이 attribute를 네이밍 추론보다 먼저 확인한다.

```php
use Illuminate\Database\Eloquent\Attributes\UsePolicy;

#[UsePolicy(ArticlePolicy::class)]
class Post extends Model {}
```

상속 탐색을 하지 않으므로 추상 베이스 모델에 붙이는 방식은 동작하지 않는다 — 각 구체 모델에 개별 부착해야 한다. 다른 모델 설정 attribute와의 관계는 [[Eloquent Model Attributes]] 참고.

## DDD 관점에서의 활용

Gate/Policy는 "이 사용자가 이 리소스에 접근 가능한가"라는 **애플리케이션 경계의 접근 제어**에 적합하다. 하지만 다음을 구분해야 한다.

| 규칙 성격 | 위치 |
|-----------|------|
| 단순 소유권/역할 기반 ACL (`$user->id === $contact->user_id`) | Gate/Policy로 충분 |
| 복잡한 비즈니스 규칙과 얽힌 인가 (예: "주문이 특정 상태이고 사용자가 매니저 역할일 때만 취소 가능") | 도메인 서비스/[[Aggregate]] 내부 불변식으로 판단하고, Policy는 그 판단을 호출하는 얇은 래퍼로 유지 |

```php
class OrderPolicy
{
    public function cancel(User $user, Order $order): bool
    {
        // 복잡한 판단은 도메인 서비스에 위임
        return app(OrderCancellationPolicy::class)->isAllowed($user->toDomainId(), $order);
    }
}
```

이렇게 하면 인가 규칙의 프레임워크 배선(Gate 등록)과 실제 비즈니스 판단(도메인 서비스)이 분리되어, 도메인 로직을 라라벨 없이도 단위 테스트할 수 있다.

## 주의사항 / 안티패턴

- Policy/Gate 클로저 안에 여러 단계의 도메인 로직을 그대로 작성하지 말 것 — 재사용과 테스트가 어려워진다.
- 인가 실패를 조용히 무시하고 빈 결과를 반환하는 대신 명시적으로 403을 반환해 의도를 분명히 한다.

## 참고

- [[Form Request]] — `authorize()`에서 Gate/Policy 호출 조합
- [[Laravel Data]] — `Data` 클래스는 `authorize()`에 해당하는 인가 기능이 없으므로, Data 클래스를 쓰더라도 인가는 여기(Gate/Policy)에 남겨둬야 한다
- [[Pipeline Pattern]] — `can:` 미들웨어로 라우트 단위 인가 적용
- [[Eloquent Model Attributes]] — `#[UsePolicy]`를 포함한 모델 설정 attribute 전체 목록
- 소스: 처음부터 제대로 배우는 라라벨, 9장 사용자 인증과 인가
