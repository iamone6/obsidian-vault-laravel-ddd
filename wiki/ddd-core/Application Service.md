---
title: Application Service
category: ddd-core
tags: [ddd, application-service, use-case]
related: [[Repository]], [[DTO]], [[CQRS]]
---

# Application Service

하나의 유스케이스를 오케스트레이션하는 레이어. 도메인 로직은 담지 않고, 도메인 객체들을 조합하여 트랜잭션을 완성한다.

## 핵심 개념

- **얇아야 한다**: 비즈니스 로직은 도메인 레이어에. Application Service는 조율만.
- **트랜잭션 경계**: 하나의 Application Service 메서드 = 하나의 트랜잭션.
- **입력**: [[DTO]] 또는 Command 객체로 받는다.
- **출력**: [[DTO]], 원시 타입, 또는 void.
- 도메인 객체(Entity, VO)를 외부로 직접 반환하지 않는다.

## 기본 구조

```php
// Application/Service/OrderApplicationService.php
final class OrderApplicationService
{
    public function __construct(
        private readonly OrderRepository $orders,
        private readonly CustomerRepository $customers,
        private readonly ProductRepository $products,
    ) {}

    public function createOrder(CreateOrderCommand $command): OrderId
    {
        $customer = $this->customers->findById($command->customerId)
            ?? throw new CustomerNotFoundException($command->customerId);

        $order = Order::create($command->customerId);

        foreach ($command->items as $item) {
            $product = $this->products->findById($item->productId)
                ?? throw new ProductNotFoundException($item->productId);

            $order->addItem(
                $product->id(),
                $item->quantity,
                $product->price()
            );
        }

        $this->orders->save($order);

        return $order->id();
    }

    public function confirmOrder(ConfirmOrderCommand $command): void
    {
        $order = $this->orders->findById($command->orderId)
            ?? throw new OrderNotFoundException($command->orderId);

        $order->confirm();

        $this->orders->save($order);
    }
}
```

## Command 객체

[[DTO]]의 일종. 의도를 명확히 표현한다.

```php
// Application/Command/CreateOrderCommand.php
final class CreateOrderCommand
{
    public function __construct(
        public readonly CustomerId $customerId,
        /** @var CreateOrderItemCommand[] */
        public readonly array $items,
    ) {}
}

final class CreateOrderItemCommand
{
    public function __construct(
        public readonly ProductId $productId,
        public readonly int $quantity,
    ) {}
}
```

## Controller와의 연결

Controller는 HTTP 요청을 Command로 변환하고, Application Service를 호출한다. 비즈니스 로직 없이.

```php
// Http/Controller/OrderController.php
final class OrderController extends Controller
{
    public function __construct(
        private readonly OrderApplicationService $service
    ) {}

    public function store(CreateOrderRequest $request): JsonResponse
    {
        $command = new CreateOrderCommand(
            customerId: CustomerId::from(Auth::id()),
            items: array_map(
                fn($item) => new CreateOrderItemCommand(
                    ProductId::from($item['product_id']),
                    $item['quantity']
                ),
                $request->items
            )
        );

        $orderId = $this->service->createOrder($command);

        return response()->json(['order_id' => $orderId->toString()], 201);
    }
}
```

## [[Action Pattern]] 대안 (단일 메서드)

Application Service 대신 Laravel에서 자주 쓰는 단일 책임 Action 클래스:

```php
// Application/Action/CreateOrderAction.php
final class CreateOrderAction
{
    public function __construct(
        private readonly OrderRepository $orders,
        private readonly CustomerRepository $customers,
    ) {}

    public function execute(CreateOrderCommand $command): OrderId
    {
        // ...동일한 로직
    }
}
```

## 주의사항 / 안티패턴

- **Fat Application Service**: 도메인 로직이 Application Service로 흘러들어오면 도메인 레이어가 빈 껍데기가 된다.
- **도메인 객체 직접 반환**: 컨트롤러에 `Order` 엔티티를 반환하면 View 로직이 도메인에 결합된다. [[DTO]]로 변환.
- **다중 Aggregate 수정**: 한 Application Service에서 두 Aggregate를 수정하면 트랜잭션 일관성 문제. [[Domain Event]]로 분리.

## 참고

- [[CQRS]] — Command/Query 분리로 Application Service 역할 세분화
- [[DTO]] — 입/출력 데이터 전달 객체
