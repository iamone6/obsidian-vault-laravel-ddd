---
title: Entity
category: ddd-core
tags: [ddd, entity, identity]
related: [[Value Object]], [[Aggregate]], [[Eloquent and DDD]]
---

# Entity

고유한 식별자(ID)를 가지며, 생명 주기를 통해 상태가 변해도 동일한 객체로 취급되는 도메인 객체.

## 핵심 개념

- **동등성**: ID로 판단. 속성이 달라도 ID가 같으면 같은 객체.
- **가변성**: 상태가 변할 수 있으나, 불변 [[Value Object]]를 속성으로 갖는 것이 권장된다.
- **생명 주기**: 생성 → 변경 → 소멸의 사이클을 가진다.

[[Value Object]]와의 차이:

| | Entity | Value Object |
|--|--------|-------------|
| 동등성 기준 | ID | 모든 속성 |
| 가변성 | 가변 | 불변 |
| 예시 | Order, User, Product | Money, Address, Email |

## Laravel 구현

Eloquent를 사용하지 않는 순수 도메인 엔티티:

```php
// Domain/Model/Order.php
final class Order
{
    private OrderId $id;
    private CustomerId $customerId;
    private OrderStatus $status;
    private Money $totalAmount;
    /** @var OrderItem[] */
    private array $items = [];

    private function __construct(
        OrderId $id,
        CustomerId $customerId,
    ) {
        $this->id = $id;
        $this->customerId = $customerId;
        $this->status = OrderStatus::Pending;
        $this->totalAmount = Money::zero('KRW');
    }

    public static function create(CustomerId $customerId): self
    {
        return new self(OrderId::generate(), $customerId);
    }

    public function addItem(ProductId $productId, int $quantity, Money $unitPrice): void
    {
        $this->items[] = new OrderItem($productId, $quantity, $unitPrice);
        $this->recalculateTotal();
    }

    public function confirm(): void
    {
        if (!$this->status->isPending()) {
            throw new \DomainException('확인 가능한 상태가 아닙니다.');
        }
        $this->status = OrderStatus::Confirmed;
    }

    public function id(): OrderId { return $this->id; }
    public function status(): OrderStatus { return $this->status; }
}
```

## ID 타입 강타입 사용

문자열/정수 ID 대신 전용 Value Object ID를 쓰면 타입 안전성이 올라간다.

```php
// Domain/Model/OrderId.php
final class OrderId
{
    private function __construct(private readonly string $value) {}

    public static function generate(): self
    {
        return new self((string) Str::uuid());
    }

    public static function from(string $value): self
    {
        return new self($value);
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }

    public function toString(): string { return $this->value; }
}
```

## 주의사항 / 안티패턴

- **Anemic Domain Model**: getter/setter만 있고 비즈니스 로직이 없는 엔티티. 로직이 서비스 레이어에 흘러넘친다.
- **Entity를 DTO로 사용**: 엔티티를 View나 API 응답에 직접 노출하지 말 것. [[DTO]]로 변환.
- **Eloquent 모델 = 도메인 엔티티**: Eloquent 모델과 도메인 엔티티를 동일시하면 인프라 의존성이 도메인을 오염시킨다. [[Eloquent and DDD]] 참고.

## 참고

- [[Aggregate]] — 엔티티의 일관성 경계
- [[Value Object]] — 엔티티의 속성으로 사용되는 불변 객체
- [[Eloquent and DDD]] — Laravel에서 Eloquent 모델과 분리하는 방법
