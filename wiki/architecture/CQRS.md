---
title: CQRS (Command Query Responsibility Segregation)
category: architecture
tags: [architecture, cqrs, command, query]
related: [[Application Service]], [[Domain Event]], [[Repository]]
---

# CQRS

Command(상태 변경)와 Query(데이터 조회)를 별도 모델과 경로로 분리하는 패턴.

## 핵심 원칙

- **Command**: 상태를 변경하고, 반환값은 ID 또는 void
- **Query**: 상태를 변경하지 않고, 데이터를 반환
- 두 책임을 같은 클래스에 두지 않는다

## Command 측 (Write Model)

```php
// Application/Command/CreateOrderCommand.php
final class CreateOrderCommand
{
    public function __construct(
        public readonly string $customerId,
        public readonly array $items,
    ) {}
}

// Application/Command/CreateOrderHandler.php
final class CreateOrderHandler
{
    public function __construct(
        private readonly OrderRepository $orders,
        private readonly CustomerRepository $customers,
    ) {}

    public function handle(CreateOrderCommand $command): string
    {
        $customer = $this->customers->findById(
            CustomerId::from($command->customerId)
        ) ?? throw new CustomerNotFoundException();

        $order = Order::create($customer->id());

        foreach ($command->items as $item) {
            $order->addItem(
                ProductId::from($item['product_id']),
                $item['quantity'],
                Money::of($item['unit_price'], 'KRW')
            );
        }

        $this->orders->save($order);

        return $order->id()->toString();
    }
}
```

## Query 측 (Read Model)

Read Model은 도메인 모델을 거치지 않고 직접 DB를 쿼리한다. 최적화된 View를 반환.

```php
// Application/Query/GetOrderQuery.php
final class GetOrderQuery
{
    public function __construct(
        public readonly string $orderId,
    ) {}
}

// Application/Query/GetOrderHandler.php
final class GetOrderHandler
{
    public function __construct(
        private readonly OrderReadRepository $readRepo,
    ) {}

    public function handle(GetOrderQuery $query): OrderViewModel
    {
        return $this->readRepo->findById($query->orderId)
            ?? throw new OrderNotFoundException();
    }
}

// Application/ViewModel/OrderViewModel.php
final class OrderViewModel
{
    public function __construct(
        public readonly string $id,
        public readonly string $status,
        public readonly string $customerName,
        public readonly array $items,
        public readonly string $formattedTotal,
    ) {}
}
```

## Read Repository (쿼리 최적화)

```php
// Infrastructure/Persistence/EloquentOrderReadRepository.php
final class EloquentOrderReadRepository implements OrderReadRepository
{
    public function findById(string $id): ?OrderViewModel
    {
        $record = DB::table('orders')
            ->join('customers', 'orders.customer_id', '=', 'customers.id')
            ->select([
                'orders.id',
                'orders.status',
                'customers.name as customer_name',
                'orders.total_amount',
                'orders.currency',
            ])
            ->where('orders.id', $id)
            ->first();

        return $record ? new OrderViewModel(
            id:            $record->id,
            status:        $record->status,
            customerName:  $record->customer_name,
            items:         $this->loadItems($id),
            formattedTotal: number_format($record->total_amount) . ' ' . $record->currency,
        ) : null;
    }
}
```

## Command Bus (선택적)

Command를 Handler와 느슨하게 연결하려면 Command Bus 패턴을 사용한다.

```php
// Laravel Bus 활용
Bus::dispatch(new CreateOrderCommand(...));

// 또는 직접 해결
$handler = app(CreateOrderHandler::class);
$handler->handle($command);
```

## CQRS 도입 시점

- 읽기/쓰기 부하 비율이 크게 다를 때
- 조회 쿼리가 복잡하여 도메인 모델을 경유하면 성능 저하가 날 때
- 이벤트 소싱과 함께 사용할 때

**단순한 CRUD 앱에는 과도한 복잡도**가 될 수 있다.

## 가벼운 CQRS: Action + ViewModel

풀스케일 CQRS(별도 Command Bus, 읽기/쓰기 DB 분리, 이벤트 소싱)까지 가지 않아도, Laravel에서는 다음 매핑만으로 CQRS의 핵심 이점(책임 분리)을 실용적으로 얻을 수 있다 (『Domain-Driven Design with Laravel』, Martin Joo):

- **[[Action Pattern|Action]] = Command** — 상태를 변경하는 쓰기 작업
- **[[ViewModel]] = Query** — 상태를 변경하지 않는 읽기 작업

```php
// Command 역할
class CreateTodoAction
{
    public static function execute(TodoData $data): Todo { /* ... */ }
}

// Query 역할
class GetDashboardViewModel extends ViewModel
{
    public function newSubscribersCount(): NewSubscribersCountData { /* ... */ }
}
```

이 정도의 가벼운 분리만으로도 코드가 이해하기 쉬워지고, 클래스가 작고 유지보수하기 쉬워지며, 책임이 명확히 나뉘는 효과를 얻는다. 이벤트 소싱이나 별도 Command Bus는 마이크로서비스/이벤트 기반 아키텍처처럼 정말로 읽기·쓰기 부하가 크게 다르거나 별도 서비스로 분리해야 하는 상황에서만 고려한다.

## 참고

- [[Domain Event]] — Command 처리 결과로 발행되는 이벤트
- [[Application Service]] — CQRS를 적용하기 전 단계
- [[Repository]] — Write Model의 저장소
- [[Action Pattern]], [[ViewModel]] — Laravel에서의 가벼운 CQRS 구현체
