---
title: Domain Testing
category: testing
tags: [testing, domain, unit-test]
related: [[Entity]], [[Aggregate]], [[Value Object]]
---

# Domain Testing

도메인 레이어([[Entity]], [[Value Object]], [[Aggregate]])의 단위 테스트. 프레임워크나 DB 없이 순수 PHP로 테스트한다.

## 특징

- 빠름: 외부 의존성 없음
- 안정적: 인프라 변경에 영향받지 않음
- 도메인 규칙을 문서화하는 역할

## 예제 — Value Object 테스트

```php
// tests/Unit/Domain/ValueObject/MoneyTest.php
class MoneyTest extends TestCase
{
    public function test_두_금액을_더할_수_있다(): void
    {
        $a = Money::of(1000, 'KRW');
        $b = Money::of(2000, 'KRW');

        $result = $a->add($b);

        $this->assertSame(3000, $result->amount());
        $this->assertSame('KRW', $result->currency());
    }

    public function test_통화가_다른_금액은_더할_수_없다(): void
    {
        $this->expectException(\DomainException::class);

        Money::of(1000, 'KRW')->add(Money::of(1, 'USD'));
    }

    public function test_금액은_음수일_수_없다(): void
    {
        $this->expectException(\InvalidArgumentException::class);

        Money::of(-1, 'KRW');
    }
}
```

## 예제 — Aggregate 테스트

```php
// tests/Unit/Domain/Model/OrderTest.php
class OrderTest extends TestCase
{
    public function test_주문_생성시_Pending_상태다(): void
    {
        $order = Order::create(CustomerId::from('customer-1'));

        $this->assertEquals(OrderStatus::Pending, $order->status());
    }

    public function test_상품을_추가할_수_있다(): void
    {
        $order = Order::create(CustomerId::from('customer-1'));
        $order->addItem(
            ProductId::from('product-1'),
            2,
            Money::of(10000, 'KRW')
        );

        $this->assertCount(1, $order->items());
        $this->assertEquals(Money::of(20000, 'KRW'), $order->total());
    }

    public function test_비어있는_주문은_확정할_수_없다(): void
    {
        $this->expectException(\DomainException::class);
        $this->expectExceptionMessage('상품이 없는 주문');

        $order = Order::create(CustomerId::from('customer-1'));
        $order->confirm();
    }

    public function test_주문_확정시_OrderConfirmed_이벤트가_발행된다(): void
    {
        $order = Order::create(CustomerId::from('customer-1'));
        $order->addItem(ProductId::from('p-1'), 1, Money::of(5000, 'KRW'));
        $order->confirm();

        $events = $order->pullDomainEvents();

        $this->assertCount(2, $events); // OrderCreated + OrderConfirmed
        $this->assertInstanceOf(OrderConfirmed::class, $events[1]);
    }

    public function test_확정된_주문에는_상품을_추가할_수_없다(): void
    {
        $this->expectException(\DomainException::class);

        $order = $this->createConfirmedOrder();
        $order->addItem(ProductId::from('p-2'), 1, Money::of(5000, 'KRW'));
    }

    private function createConfirmedOrder(): Order
    {
        $order = Order::create(CustomerId::from('customer-1'));
        $order->addItem(ProductId::from('p-1'), 1, Money::of(5000, 'KRW'));
        $order->confirm();
        return $order;
    }
}
```

## 테스트 명명 규칙

메서드명에 행위와 기대 결과를 담는다:

```
test_{상황}_{기대결과}
test_{행위}_{조건}_{기대결과}
```

한글 테스트명도 가독성이 좋다:
```php
public function test_비어있는_주문은_확정할_수_없다(): void
public function test_취소된_주문에_아이템_추가시_예외_발생(): void
```

## 도메인 객체 생성 헬퍼

중복 설정을 줄이기 위해 빌더나 팩토리 헬퍼를 테스트 레이어에 둔다.

```php
// tests/Unit/Domain/Builder/OrderBuilder.php
final class OrderBuilder
{
    private CustomerId $customerId;
    private array $items = [];
    private bool $confirmed = false;

    public static function new(): self { return new self(); }

    public function withCustomer(string $id): self
    {
        $clone = clone $this;
        $clone->customerId = CustomerId::from($id);
        return $clone;
    }

    public function withItem(string $productId, int $qty, int $price): self
    {
        $clone = clone $this;
        $clone->items[] = [$productId, $qty, $price];
        return $clone;
    }

    public function confirmed(): self
    {
        $clone = clone $this;
        $clone->confirmed = true;
        return $clone;
    }

    public function build(): Order
    {
        $order = Order::create($this->customerId ?? CustomerId::from('default-customer'));
        foreach ($this->items as [$pid, $qty, $price]) {
            $order->addItem(ProductId::from($pid), $qty, Money::of($price, 'KRW'));
        }
        if ($this->confirmed) $order->confirm();
        return $order;
    }
}

// 사용
$order = OrderBuilder::new()
    ->withItem('product-1', 2, 10000)
    ->confirmed()
    ->build();
```

## 참고

- [[Entity]] — 테스트 대상 도메인 객체
- [[Aggregate]] — 불변 조건 검증의 핵심 테스트 대상
