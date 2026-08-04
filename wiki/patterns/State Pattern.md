---
title: State Pattern (States and Transitions)
category: patterns
tags: [state-pattern, transition, enum, laravel]
related: [[Value Object]], [[Aggregate]], [[Action Pattern]]
---

# State Pattern (States and Transitions)

문자열 상태 컬럼을 조건문으로 분기하는 대신, 상태 하나당 클래스 하나를 두어 상태별 행동을 캡슐화하는 패턴. 상태 "전이" 역시 별도 클래스로 다뤄 일급 시민으로 승격시킨다.

## 핵심 개념

- **State**: 모델이 가질 수 있는 상태 각각을 표현하는 클래스. 공통 추상 클래스/인터페이스를 상속한다.
- **Transition**: 한 상태에서 다른 상태로 이동하는 동작 자체를 표현하는 클래스. 실행 전 현재 상태가 유효한지 검증하고, 유효하지 않으면 예외를 던진다.

## Laravel 구현

### State 클래스

```php
abstract class OrderStatus
{
    public function __construct(protected Order $order) {}
    abstract public function canBeChanged(): bool;
}

class DraftOrderStatus extends OrderStatus
{
    public function canBeChanged(): bool { return true; }
}

class PaidOrderStatus extends OrderStatus
{
    public function canBeChanged(): bool { return false; }
}
```

### PHP 8.1 Enum을 팩토리로 활용

Enum에 메서드를 정의해 상태 문자열 → 상태 클래스 변환을 팩토리처럼 사용할 수 있다 (PHP 8.1 enum의 잘 알려지지 않은 활용법).

```php
enum OrderStatuses: string
{
    case Draft = 'draft';
    case Pending = 'pending';
    case Paid = 'paid';
    case PaymentFailed = 'payment-failed';

    public function createOrderStatus(Order $order): OrderStatus
    {
        return match ($this) {
            self::Draft => new DraftOrderStatus($order),
            self::Pending => new PendingOrderStatus($order),
            self::Paid => new PaidOrderStatus($order),
            self::PaymentFailed => new PaymentFailedOrderStatus($order),
        };
    }
}

class Order extends Model
{
    protected function status(): Attribute
    {
        return new Attribute(
            get: fn (string $value) => OrderStatuses::from($value)->createOrderStatus($this),
        );
    }
}

// 사용
abort_if(! $order->status->canBeChanged(), 400);
```

### Transition 클래스

```php
interface Transition
{
    /** @throws Exception */
    public function execute(Order $order): Order;
}

class DraftToPendingTransition implements Transition
{
    public function execute(Order $order): Order
    {
        if ($order->status::class !== DraftOrderStatus::class) {
            throw new Exception('Transition not allowed');
        }

        $order->status = OrderStatuses::Pending;
        $order->save();

        return $order;
    }
}
```

## 장점

- **캡슐화**: 특정 상태와 관련된 모든 로직이 한 곳에 있다.
- **관심사 분리**: 상태마다 클래스가 분리되어 있어 각 상태의 규칙을 독립적으로 이해/테스트할 수 있다.
- **단순한 로직**: 문자열 속성 주변의 지저분한 if-else/switch가 사라진다.

## 주의사항 / 안티패턴

- **오버엔지니어링 주의**: 상태가 2~3개뿐이고 로직도 단순하면 이 패턴은 과한 오버헤드다. 상태별 규칙이 실제로 복잡해질 때만 도입한다.
- Transition 클래스 없이 컨트롤러에서 직접 `$order->status = 'paid'; $order->save();`처럼 문자열을 대입하는 방식은 상태 전이 유효성 검증을 누락하기 쉽다 — 복잡한 상태 머신일수록 Transition을 강제해 검증 로직을 우회할 수 없게 만든다.

## 참고

- [[Value Object]] — Enum 기반 상태도 일종의 값 객체로 취급 가능
- [[Action Pattern]] — Transition을 Action으로 감싸 트랜잭션/이벤트 발행과 결합
- 소스: Domain-Driven Design with Laravel (Martin Joo), States And Transitions 챕터
