---
title: Aggregate
category: ddd-core
tags: [ddd, aggregate, aggregate-root, consistency]
related: [[Entity]], [[Value Object]], [[Domain Event]], [[Repository]]
---

# Aggregate

일관성 경계(Consistency Boundary)를 가지는 [[Entity]]와 [[Value Object]]의 클러스터. 클러스터의 진입점을 **Aggregate Root**라고 한다.

## 핵심 개념

- 한 트랜잭션에서 하나의 Aggregate만 변경한다 (단일 Aggregate 규칙)
- 외부에서는 반드시 Aggregate Root를 통해서만 내부 객체에 접근한다
- [[Repository]]는 Aggregate Root 단위로만 존재한다
- Aggregate 경계 설계는 도메인의 불변 조건(Invariant)에 따라 결정한다

### Aggregate 경계 결정 기준

한 트랜잭션에서 반드시 함께 일관성이 유지되어야 하는 객체들을 하나의 Aggregate로 묶는다.

```
Order (Aggregate Root)
├── OrderItem[] (Entity — Order 없이 존재 불가)
└── ShippingAddress (Value Object)

Customer (별도 Aggregate Root)
```

Order와 Customer는 **다른 Aggregate**. Order는 `CustomerId`(참조)만 가지고, Customer 객체 자체를 포함하지 않는다.

### 설계 체크리스트 — 경계를 고민할 때 순서대로 물어볼 4가지 질문

Aggregate 경계는 "관련 있어 보이는 것"이 아니라 **진짜 불변식(invariant)**으로 결정한다. 두 객체를 하나의 Aggregate로 묶을지 고민될 때 이 순서로 판단한다.

1. **"이 두 객체가 동시에 무효한 상태가 될 수 있는가?"**
   그렇다면 같은 Aggregate. 예: "상품이 하나도 없는 확정된 주문"은 존재하면 안 되므로 `Order`와 `OrderItem`은 같은 Aggregate.
2. **"이 규칙이 깨지면 같은 트랜잭션에서 즉시 막아야 하는가, 나중에 보정해도 되는가?"**
   즉시 막아야 하면 같은 Aggregate. 나중에 보정해도 되면 [[Domain Event]]로 분리한다. 예: "재고 없이는 주문 생성 불가"는 즉시 막아야 하지만, "주문 확정 시 재고 차감"은 `OrderConfirmed` 이벤트로 별도 Aggregate(`Inventory`)에 최종적 일관성(Eventually Consistent)으로 반영해도 된다.
3. **"이 Aggregate를 로드할 때 얼마나 많은 데이터가 함께 로드되는가?"**
   Aggregate는 작을수록 좋다. `Customer`가 `orders` 배열 전체를 들고 있다면(아래 안티패턴 참고), 십중팔구 두 개의 Aggregate여야 한다는 신호다.
4. **"다른 Aggregate를 참조할 때 객체 전체가 필요한가, ID만 있으면 되는가?"**
   거의 항상 ID만으로 충분하다. `Order`는 `Customer` 객체가 아니라 `CustomerId`만 가진다.

이 네 질문에 답하기 어렵다면, 일단 작게 나누고 [[Domain Event]]로 연결하는 쪽을 기본값으로 삼는다 — Aggregate를 나중에 합치는 것보다 쪼개는 것이 훨씬 어렵다.

## Laravel 구현

```php
// Domain/Model/Order.php (Aggregate Root)
final class Order
{
    private OrderId $id;
    private CustomerId $customerId;   // 다른 Aggregate는 ID로만 참조
    private OrderStatus $status;
    private array $items = [];
    private array $domainEvents = [];

    private function __construct(OrderId $id, CustomerId $customerId)
    {
        $this->id = $id;
        $this->customerId = $customerId;
        $this->status = OrderStatus::Draft;
    }

    public static function create(CustomerId $customerId): self
    {
        $order = new self(OrderId::generate(), $customerId);
        $order->recordEvent(new OrderCreated($order->id, $customerId));
        return $order;
    }

    public function addItem(ProductId $productId, int $qty, Money $price): void
    {
        // 불변 조건 검사
        if ($this->status->isConfirmed()) {
            throw new \DomainException('확정된 주문에는 상품을 추가할 수 없습니다.');
        }
        if ($qty <= 0) {
            throw new \DomainException('수량은 1 이상이어야 합니다.');
        }

        $this->items[] = new OrderItem(
            OrderItemId::generate(),
            $productId,
            $qty,
            $price
        );
    }

    public function confirm(): void
    {
        if (empty($this->items)) {
            throw new \DomainException('상품이 없는 주문은 확정할 수 없습니다.');
        }
        $this->status = OrderStatus::Confirmed;
        $this->recordEvent(new OrderConfirmed($this->id));
    }

    // 도메인 이벤트 수집 (After Commit 패턴)
    private function recordEvent(object $event): void
    {
        $this->domainEvents[] = $event;
    }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }

    // Internal class — 외부에서 직접 생성 불가
    public function items(): array { return $this->items; }
    public function id(): OrderId { return $this->id; }
}
```

## Aggregate 크기 설계

**작게 유지**가 원칙이다. 큰 Aggregate는:
- 트랜잭션 충돌(Optimistic Locking) 빈도 증가
- 메모리 사용량 증가
- 로딩 성능 저하

```php
// 안티패턴: Customer가 orders를 직접 포함
class Customer {
    private array $orders = [];  // ❌ Customer와 Order는 별도 Aggregate
}

// 올바른 방식: ID 참조
class Order {
    private CustomerId $customerId;  // ✅ ID로만 참조
}
```

## 주의사항 / 안티패턴

- **Aggregate 간 객체 직접 참조**: `$order->customer->address` 대신 `$order->customerId`로 참조하고, 필요 시 별도 쿼리.
- **두 Aggregate 동시 수정**: 한 트랜잭션에서 두 Aggregate를 수정하면 설계 문제. [[Domain Event]] + Eventually Consistent로 해결.
- **Lazy Loading 남용**: Aggregate 내부 컬렉션을 Eloquent Lazy Loading으로 처리하면 N+1 문제 + 도메인 로직이 인프라에 의존.

## 참고

- [[Repository]] — Aggregate Root 단위 저장/조회
- [[Domain Event]] — Aggregate 간 결과적 일관성
- [[Unit of Work]] — 트랜잭션 관리
- [[State Pattern]] — Aggregate의 상태(`OrderStatus` 등)를 클래스로 캡슐화하는 방법
