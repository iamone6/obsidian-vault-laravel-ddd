---
title: DTO (Data Transfer Object)
category: laravel
tags: [laravel, dto, data-transfer]
related: [[Application Service]], [[Action Pattern]], [[Form Request]], [[Laravel Data]]
---

# DTO (Data Transfer Object)

레이어 간 데이터를 전달하기 위한 단순 객체. 비즈니스 로직 없이 데이터만 보유한다.

## 핵심 개념

- 레이어 경계에서 도메인 객체 대신 사용
- 불변(readonly)이 권장됨
- 직렬화/역직렬화에 친화적

## 사용 시점

| 방향 | 사용 |
|------|------|
| Controller → Application Service | Command DTO |
| Application Service → Controller | Response DTO |
| Domain → Application | 도메인 객체 직접 사용 (DTO 불필요) |

## PHP 8.x readonly 활용

```php
// Application/DTO/OrderData.php
final class OrderData
{
    public function __construct(
        public readonly string $id,
        public readonly string $customerId,
        public readonly string $status,
        public readonly int $totalAmount,
        public readonly string $currency,
        /** @var OrderItemData[] */
        public readonly array $items,
        public readonly \DateTimeImmutable $createdAt,
    ) {}

    public static function fromOrder(Order $order): self
    {
        return new self(
            id:          $order->id()->toString(),
            customerId:  $order->customerId()->toString(),
            status:      $order->status()->value,
            totalAmount: $order->total()->amount(),
            currency:    $order->total()->currency(),
            items:       array_map(
                fn($item) => OrderItemData::fromOrderItem($item),
                $order->items()
            ),
            createdAt:   $order->createdAt(),
        );
    }
}
```

## Spatie Laravel Data 패키지

복잡한 변환 로직에는 `spatie/laravel-data`를 활용할 수 있다.

```php
use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\Validation\Required;

class CreateOrderData extends Data
{
    public function __construct(
        #[Required]
        public readonly string $customerId,
        /** @var CreateOrderItemData[] */
        public readonly array $items,
    ) {}
}

// Form Request에서 자동 변환
public function store(CreateOrderRequest $request): JsonResponse
{
    $data = CreateOrderData::from($request);
    // ...
}
```

## Command DTO vs Query DTO

```php
// Command DTO — 상태 변경 의도
final class CreateOrderCommand
{
    public function __construct(
        public readonly CustomerId $customerId,
        public readonly array $items,
    ) {}
}

// Query DTO — 데이터 조회 의도
final class GetOrderQuery
{
    public function __construct(
        public readonly OrderId $orderId,
        public readonly CustomerId $requestingCustomerId,
    ) {}
}
```

## DTO vs Value Object: 무엇이 다른가?

『Domain-Driven Design with Laravel』(Martin Joo)이 제시하는 가장 실용적인 구분 기준:

- **DTO는 ID를 가진다** — 모델(엔티티)을 표현하기 때문이다.
- **[[Value Object]]는 ID를 갖지 않는다** — 값 자체를 표현하기 때문이다.

이 구분이 원칙이긴 하지만, 저자도 "엄격한 규칙이라기보다 가이드라인"이라고 말한다. 실무에서는 DTO와 VO를 혼용하거나, 상황에 따라 편의상 VO 자리에 DTO를 쓰는 경우도 흔하다 — 완벽한 이론적 분리보다 일관된 선택이 더 중요하다.

## Spatie laravel-data의 `Lazy` 속성

`spatie/laravel-data`의 `Lazy` 타입은 Laravel Resource의 `whenLoaded()`와 유사하게, 관계 데이터를 필요할 때만 포함시켜 N+1 문제를 피하게 해준다.

```php
class SubscriberData extends Data
{
    public function __construct(
        public readonly ?int $id,
        public readonly string $email,
        /** @var DataCollection<TagData> */
        public readonly null|Lazy|DataCollection $tags,
        public readonly null|Lazy|FormData $form,
    ) {}

    public static function fromRequest(Request $request): self
    {
        return self::from([
            ...$request->all(),
            'tags' => TagData::collection(
                Tag::whereIn('id', $request->collect('tags'))->get()
            ),
            'form' => FormData::from(Form::find($request->form_id)),
        ]);
    }
}
```

`Data` 클래스는 검증 규칙(`rules()`)까지 가질 수 있어, Request + Resource + DTO 역할을 하나의 클래스로 통합할 수 있다. 다만 이 패키지 없이도 순수 PHP readonly 클래스로 충분히 DTO를 구현할 수 있다 — 어디까지나 편의 도구다.

검증 규칙을 `rules()` 배열 대신 `#[Max(100)]`, `#[Email]` 같은 PHP Attribute로 프로퍼티 위에 선언할 수도 있다 — [[Form Request]]의 `rules()`를 대체하는 방식이다. 다만 `authorize()`에 해당하는 인가 기능은 없으므로 인가는 별도로 [[Policy and Gate]]에 남겨둬야 한다. Attribute 기반 검증 문법과 검증 실패 시 예외 처리(커스텀 메시지, 전역 핸들러 통합 등)는 [[Laravel Data]] 문서에서 자세히 다룬다.

## 주의사항

- **도메인 로직 포함 금지**: DTO에 `calculate()`, `validate()` 같은 비즈니스 메서드를 두지 말 것.
- **Eloquent 모델을 DTO로 오해**: Eloquent 모델은 DTO가 아니다. 변환 메서드(`toArray`, `toJson`)가 있어도 DTO가 아님.
- 대량 할당(`Model::create($request->all())`)을 그대로 DTO 도입의 이유로 삼지 말 것 — DTO의 목적은 타입 안전성과 구조화이며, 화이트리스트 검증은 별도로 필요하다.

## 참고

- [[Application Service]] — DTO를 입출력으로 사용하는 서비스
- [[Form Request]] — HTTP 레이어에서 DTO로 변환
- [[Laravel Data]] — Validation Attribute 문법과 검증 실패 예외 처리 상세
- [[Value Object]] — ID 없는 값 표현과의 구분
- [[Design Philosophy]] — 엄격한 규칙보다 가이드라인으로 접근하는 실용주의 원칙
- 소스: Domain-Driven Design with Laravel (Martin Joo), Data Transfer Objects 챕터 / `2026-07-30_Spatie Laravel Data 공식 문서 종합.md`
