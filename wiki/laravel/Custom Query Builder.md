---
title: Custom Query Builder
category: laravel
tags: [laravel, eloquent, query-builder, repository-alternative]
related: [[Repository]], [[Repository Implementation]], [[Action Pattern]], [[Eloquent Model Attributes]]
---

# Custom Query Builder

Eloquent의 `newEloquentBuilder()`를 오버라이드해 모델 전용 쿼리 빌더 클래스를 만드는 기법. Martin Joo(『Domain-Driven Design with Laravel』)는 이를 "라라벨다운 [[Repository]] 대안"으로 제시한다.

## 핵심 개념

`Product::where(...)`를 호출하면 실제로는 `Illuminate\Database\Eloquent\Builder` 인스턴스와 상호작용한다. 모델과 Builder를 연결하는 지점이 `newEloquentBuilder()`이며, 이를 오버라이드하면 커스텀 Builder 클래스를 반환할 수 있다.

```php
class MailBuilder extends Builder
{
    public function whereOpened(): self
    {
        return $this->whereNotNull('opened_at'); // self를 반환해야 체이닝 가능
    }

    public function sumByDate(DateFilter $dates, User $user): float
    {
        // 체이닝이 필요 없는 메서드는 스칼라 값을 바로 반환해도 된다
        return $this->whereBelongsTo($user)->wherePayedBetween($dates)->sum('amount');
    }
}

class Mail extends Model
{
    public function newEloquentBuilder($query): MailBuilder
    {
        return new MailBuilder($query);
    }
}

// 사용
Mail::whereOpened()->where('title', 'First Mail')->get();
$dividendThisMonth = DividendPayout::sumByDate(DateFilter::thisMonth());
```

## `newEloquentBuilder()` 대신 attribute로 연결하기 (Laravel 12.19+)

`newEloquentBuilder()` 오버라이드 대신 `#[UseEloquentBuilder]` attribute로도 동일하게 연결할 수 있다 — 표기 방식만 다를 뿐 동작은 같다.

```php
use Illuminate\Database\Eloquent\Attributes\UseEloquentBuilder;

#[UseEloquentBuilder(MailBuilder::class)]
class Mail extends Model {}
```

이 attribute는 **상속 탐색을 하지 않는다** — 추상 베이스 모델에 붙여 자식이 물려받게 하는 설계는 동작하지 않으므로, 각 구체 모델 클래스에 개별 부착해야 한다. 자세한 내용은 [[Eloquent Model Attributes]] 참고.

## Repository와의 비교

| | [[Repository]] | Custom Query Builder |
|---|---|---|
| 위치 | 별도 클래스 (`ProductRepository`) | 모델에 연결된 Builder 클래스 |
| 체이닝 | 불가 (메서드 하나 = 쿼리 하나) | 가능 (`where()->whereOpened()->get()`) |
| Laravel다움 | 다소 이질적 (`$products->search()`) | Eloquent 관용구와 자연스럽게 통합 |
| DB 교체 용이성 | 이론상 유리 | 낮음 (Eloquent에 결합) |

저자의 관찰: "지난 10년간 아무도 나에게 프로그래밍 언어를 바꿔달라거나 운영 중인 프로젝트의 DB 엔진을 바꿔달라고 요청한 적이 없다" — Repository의 "DB 교체 용이성"이라는 이론적 장점은 실무에서 거의 발동되지 않는 반면, 매 모델마다 반복되는 보일러플레이트와 체이닝 불가라는 단점은 매일 체감된다는 것이 핵심 주장이다.

## 사용 원칙

- 자주 쓰이는 스코프/쿼리는 Custom Query Builder 메서드로 추출한다.
- 단발성 쿼리까지 전부 Builder에 몰아넣지 않는다 — 모델을 얇게 유지한다. 단발성 쿼리는 그 쿼리를 실제로 사용하는 [[Action Pattern|Action]]이나 컨트롤러에 남겨도 무방하다.

## DDD 관점에서의 활용

여러 모델(예: `Broadcast`, `SequenceMail`)이 동일한 조회 로직(예: "발송 대상(sendable)별 발송 건수")을 공유해야 한다면, 공통 인터페이스(`Sendable`)를 정의하고 Builder가 그 인터페이스를 받도록 하면 모델 구체 타입에 의존하지 않는 재사용 가능한 쿼리를 만들 수 있다.

```php
interface Sendable
{
    public function id(): int;
    public function type(): string;
}

class SentMailBuilder extends Builder
{
    public function whereSendable(Sendable $sendable): self
    {
        return $this->where('sendable_id', $sendable->id())
            ->where('sendable_type', $sendable->type());
    }

    public function whereOpened(): self { return $this->whereNotNull('opened_at'); }

    public function openRate(Sendable $sendable, int $total): Percent
    {
        $opened = $this->whereSendable($sendable)->whereOpened()->count();
        return Percent::from($opened, $total);
    }
}
```

이 패턴은 [[Value Object]](`Percent`)와 결합해 반환값도 도메인 친화적으로 만들 수 있다.

## 주의사항 / 안티패턴

- Builder에 비즈니스 규칙 판단(예: "이 주문을 취소해도 되는가")을 넣지 말 것 — 조회 로직만 담당한다.
- 체이닝이 필요 없는 최종 집계 메서드(`sum`, `count` 반환)는 스칼라를 그대로 반환해도 되지만, 명명을 통해 체이닝 가능 여부(스코프처럼 `self` 반환 vs 값 반환)를 명확히 구분한다.

## 참고

- [[Repository]] — 대안으로서의 전통적 Repository 패턴과 트레이드오프
- [[Repository Implementation]] — Eloquent 연관관계/Eager Loading 세부사항
- [[Eloquent Model Attributes]] — `#[UseEloquentBuilder]`를 포함한 모델 설정 attribute 전체 목록
- 소스: Domain-Driven Design with Laravel (Martin Joo), Working With Data 챕터
