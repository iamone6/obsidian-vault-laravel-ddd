---
title: Mapper Pattern
category: patterns
tags: [pattern, mapper, eloquent, domain]
related: [[Eloquent and DDD]], [[Repository Implementation]], [[Entity]]
---

# Mapper Pattern

Eloquent 모델(인프라)과 도메인 Entity 사이의 변환을 담당하는 클래스. [[Eloquent and DDD]] 전략 2의 핵심 구성요소.

## 역할

```
Domain Entity  ←──── Mapper ────→  Eloquent Model
   Order                              OrderModel
```

## 구현

```php
// Infrastructure/Persistence/OrderMapper.php
final class OrderMapper
{
    public function toDomain(OrderModel $model): Order
    {
        $order = Order::reconstruct(
            id:         OrderId::from($model->id),
            customerId: CustomerId::from($model->customer_id),
            status:     OrderStatus::from($model->status),
            total:      Money::of($model->total_amount, $model->currency),
            createdAt:  new \DateTimeImmutable($model->created_at),
        );

        foreach ($model->items as $itemModel) {
            $order->restoreItem(
                id:        OrderItemId::from($itemModel->id),
                productId: ProductId::from($itemModel->product_id),
                quantity:  $itemModel->quantity,
                unitPrice: Money::of($itemModel->unit_price, $itemModel->currency),
            );
        }

        return $order;
    }

    public function toPersistence(Order $order): array
    {
        return [
            'id'           => $order->id()->toString(),
            'customer_id'  => $order->customerId()->toString(),
            'status'       => $order->status()->value,
            'total_amount' => $order->total()->amount(),
            'currency'     => $order->total()->currency(),
            'items'        => array_map(
                fn($item) => [
                    'id'         => $item->id()->toString(),
                    'order_id'   => $order->id()->toString(),
                    'product_id' => $item->productId()->toString(),
                    'quantity'   => $item->quantity(),
                    'unit_price' => $item->unitPrice()->amount(),
                    'currency'   => $item->unitPrice()->currency(),
                ],
                $order->items()
            ),
        ];
    }
}
```

## Domain Entity에 `reconstruct` 메서드 추가

일반 `create()` 팩토리는 도메인 이벤트를 발행하지만, 저장소에서 복원할 때는 이벤트를 발행하면 안 된다.

```php
final class Order
{
    // 신규 생성 — 이벤트 발행
    public static function create(CustomerId $customerId): self
    {
        $order = new self(OrderId::generate(), $customerId);
        $order->recordEvent(new OrderCreated($order->id, $customerId));
        return $order;
    }

    // DB에서 복원 — 이벤트 발행 안 함
    public static function reconstruct(
        OrderId $id,
        CustomerId $customerId,
        OrderStatus $status,
        Money $total,
        \DateTimeImmutable $createdAt,
    ): self {
        $order = new self($id, $customerId);
        $order->status = $status;
        $order->total = $total;
        $order->createdAt = $createdAt;
        return $order;
    }
}
```

## 주의사항

- **Mapper 비대화**: 복잡한 변환 로직이 Mapper에 쌓이면 분리를 고려. 컬렉션 Mapper, 서브 객체 Mapper로 분리.
- **Lazy Loading 주의**: `toDomain()` 호출 전에 필요한 관계를 Eager Loading 해야 한다. Mapper 내부에서 `load()`를 호출하지 말 것.

## 참고

- [[Eloquent and DDD]] — Mapper 패턴이 필요한 배경
- [[Repository Implementation]] — Mapper를 사용하는 Repository 구현
