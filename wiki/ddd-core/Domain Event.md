---
title: Domain Event
category: ddd-core
tags: [ddd, event, domain-event]
related: [[Aggregate]], [[Laravel Events]], [[Application Service]]
---

# Domain Event

도메인에서 **발생한 사실(past tense)**을 표현하는 불변 객체. "무언가가 일어났다"는 것을 나타낸다.

## 핵심 개념

- 이름은 과거형 동사: `OrderConfirmed`, `PaymentProcessed`, `UserRegistered`
- 불변: 생성 후 수정 불가
- Aggregate 내에서 기록되고, 저장 후 발행된다
- Aggregate 간 결과적 일관성(Eventual Consistency)을 달성하는 주요 수단

## 이벤트 클래스 구조

```php
// Domain/Event/OrderConfirmed.php
final class OrderConfirmed
{
    public readonly \DateTimeImmutable $occurredAt;

    public function __construct(
        public readonly OrderId $orderId,
        public readonly CustomerId $customerId,
        public readonly Money $totalAmount,
    ) {
        $this->occurredAt = new \DateTimeImmutable();
    }
}
```

## Aggregate에서 이벤트 수집

Aggregate Root는 이벤트를 내부에 수집(record)하고, [[Repository]]가 저장 후 발행한다.

```php
// Domain/Model/Order.php
final class Order
{
    private array $domainEvents = [];

    public function confirm(): void
    {
        // ... 비즈니스 로직

        $this->domainEvents[] = new OrderConfirmed(
            $this->id,
            $this->customerId,
            $this->calculateTotal()
        );
    }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}
```

## Laravel 이벤트 시스템과 연결

두 가지 접근법이 있다.

### 방법 1: Repository에서 직접 발행

```php
// Infrastructure/Persistence/EloquentOrderRepository.php
final class EloquentOrderRepository implements OrderRepository
{
    public function __construct(
        private readonly Dispatcher $events
    ) {}

    public function save(Order $order): void
    {
        DB::transaction(function () use ($order) {
            $this->persist($order);

            foreach ($order->pullDomainEvents() as $event) {
                $this->events->dispatch($event);
            }
        });
    }
}
```

### 방법 2: Doctrine-style EventDispatcher 트레이트

```php
trait RecordsDomainEvents
{
    private array $domainEvents = [];

    protected function recordEvent(object $event): void
    {
        $this->domainEvents[] = $event;
    }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}
```

## 이벤트 핸들러

도메인 이벤트 핸들러는 **다른 컨텍스트**에서 반응한다.

```php
// Application/EventHandler/SendOrderConfirmationEmail.php
final class SendOrderConfirmationEmail
{
    public function __construct(
        private readonly CustomerRepository $customers,
        private readonly Mailer $mailer,
    ) {}

    public function handle(OrderConfirmed $event): void
    {
        $customer = $this->customers->findById($event->customerId);
        $this->mailer->send(new OrderConfirmationMail($customer, $event->orderId));
    }
}
```

```php
// EventServiceProvider.php
protected $listen = [
    OrderConfirmed::class => [
        SendOrderConfirmationEmail::class,
        UpdateInventory::class,
        CreateInvoice::class,
    ],
];
```

## 주의사항 / 안티패턴

- **이벤트 발행 전 DB 롤백**: 이벤트를 트랜잭션 외부에서 발행해야 한다. `DB::afterCommit()` 활용.
- **이벤트에 Aggregate 참조 포함**: 이벤트에 Aggregate 객체 전체를 담으면 직렬화 문제 및 Bounded Context 결합이 생긴다. ID와 필요한 데이터만 담을 것.
- **도메인 이벤트 ≠ Laravel 이벤트**: 도메인 이벤트는 순수 PHP 객체. Laravel 이벤트 시스템은 발행 메커니즘으로만 사용.

## 참고

- [[Aggregate]] — 이벤트를 수집하는 주체
- [[Laravel Events]] — Laravel 이벤트 시스템 연동 방법
- [[CQRS]] — 이벤트 기반 커맨드/쿼리 분리
