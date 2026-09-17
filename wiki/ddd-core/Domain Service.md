---
title: Domain Service
category: ddd-core
tags: [ddd, domain-service, service]
related: [[Application Service]], [[Action Pattern]], [[Entity]], [[Aggregate]], [[Layered Architecture]]
---

# Domain Service

특정 [[Entity]]나 [[Value Object]] 하나에 자연스럽게 속하지 않는 도메인 로직을 담는 무상태(stateless) 클래스. Domain 레이어에 속하며, [[Application Service]]와 달리 트랜잭션이나 저장을 다루지 않는다.

## 핵심 개념

- **무상태**: 인스턴스 변수에 상태를 갖지 않는다. 같은 입력에는 항상 같은 결과.
- **부작용 없음(side-effect free)**: DB 저장, 이벤트 디스패치, 메일 발송 등을 하지 않는다. 값을 계산/판단해서 반환만 한다.
- **진입점이 아니다**: Controller나 Job이 직접 호출하지 않고, [[Application Service]]나 [[Action Pattern]]의 Action이 내부에서 호출한다.
- **여러 Entity를 인자로 받을 수 있다**: 로직이 한 Entity 안에 있으면 어색해지는 경우(예: 두 Entity를 비교하는 계산)에 적합하다.

## 왜 필요한가 — Entity 메서드로는 부족한 경우

"가격 계산"을 예로 들면, 단순한 경우는 Entity 메서드로 충분하다.

```php
final class Order
{
    public function totalAmount(): Money
    {
        return array_reduce($this->items, fn(Money $sum, OrderItem $i) => $sum->add($i->subtotal()), Money::zero('KRW'));
    }
}
```

하지만 "고객 등급별 할인율 + 프로모션 쿠폰 + 재고 상황"처럼 **여러 Entity/외부 정책이 얽힌 계산**을 `Order` 안에 넣으면, `Order`가 `Customer`, `Coupon`, `Inventory`를 전부 알아야 하는 부자연스러운 의존이 생긴다. 이럴 때 Domain Service로 분리한다.

## Laravel 구현

```php
// Domain/Order/Service/PricingService.php
final class PricingService
{
    public function calculateTotal(Order $order, Customer $customer, ?Coupon $coupon): Money
    {
        $subtotal = $order->totalAmount();

        $discount = $this->discountRateFor($customer);
        $subtotal = $subtotal->multiply(1 - $discount);

        if ($coupon !== null && $coupon->isValidFor($order)) {
            $subtotal = $coupon->apply($subtotal);
        }

        return $subtotal;
    }

    private function discountRateFor(Customer $customer): float
    {
        return match ($customer->tier()) {
            CustomerTier::Vip => 0.15,
            CustomerTier::Regular => 0.05,
            default => 0.0,
        };
    }
}
```

호출하는 쪽([[Application Service]] 또는 Action)은 이 결과를 받아 저장까지 책임진다:

```php
final class ConfirmOrderAction
{
    public function __construct(
        private readonly OrderRepository $orders,
        private readonly PricingService $pricing,
    ) {}

    public function execute(OrderId $orderId, CustomerId $customerId): void
    {
        $order = $this->orders->findById($orderId) ?? throw new OrderNotFoundException($orderId);
        $customer = /* ... */;

        $total = $this->pricing->calculateTotal($order, $customer, null);   // 계산만
        $order->confirm($total);
        $this->orders->save($order);   // 저장은 Action의 책임
    }
}
```

## Domain Service vs Application Service — 구분 기준

이름이 비슷해 혼동하기 쉽지만 역할이 다르다.

| 기준 | Domain Service | [[Application Service]] |
|---|---|---|
| 레이어 | Domain | Application |
| 부작용 | 없음 — 계산/판단만, 반환값이 결과 | 있음 — Repository 저장, 이벤트 디스패치, 메일 발송 |
| 트랜잭션 경계 | 열지 않음 | 보통 여기서 시작됨(메서드 하나 = 트랜잭션 하나) |
| 호출 주체 | Action/Application Service가 내부에서 호출 | Controller, Job, Console Command가 직접 호출(진입점) |
| 프레임워크 의존 | 없음(순수 PHP) | Laravel 컨테이너/파사드 등에 얕게 의존 가능 |
| 예시 | `PricingService`, `DiscountCalculator`, `LciMatchingService` | `OrderApplicationService`, `CreateOrderAction` |

실전에서 가장 빠른 판별법: **"이 클래스가 `save()`나 이벤트 디스패치를 호출하는가?"** — 호출하면 Application 레이어, 순수 계산만 하고 끝나면 Domain Service다.

## 디렉토리 배치

### BC별 4레이어 완전분리 구조

[[Directory Structure]]의 기본 구조에서는 Domain 레이어 하위에 별도 폴더로 존재한다.

```
Order/
├── Domain/
│   ├── Model/
│   ├── Service/
│   │   └── PricingService.php   ← Domain Service
│   └── Repository/
├── Application/
│   └── Command/
│       └── CreateOrderHandler.php   ← Application Service 역할
```

`Domain/`과 `Application/`이 최상위에서부터 분리되어 있어, 폴더 위치만으로도 레이어가 구분된다.

### Martin Joo 2단 구조 (`Domain/{Feature}/...`)

이 구조는 애초에 `Domain`/`Application`을 최상위로 나누지 않는다. [[Action Pattern]]이 이미 "Application Service의 단순화 버전"이라 `Domain/{Feature}/Actions/`가 Application 레이어 역할을 흡수하고 있으므로, Domain Service는 그 형제 폴더로 둔다.

```
Domain/
└── Order/
    ├── Models/
    ├── Actions/      ← 유스케이스 진입점 (Application Service 역할)
    ├── Services/      ← Domain Service (순수 계산/판단)
    ├── DataTransferObjects/
    └── ViewModels/
```

이 조합은 이론적 제안이 아니라 [[Design Philosophy]]의 "얕은 DDD" 사례(LCA 탄소배출량 산정 API)가 실제로 채택한 구조다 — `app/Domain/{Materials,Mechanics,Report,...}/{Actions,Services,DTOs,Requests,ViewModels}`. 폴더가 layer를 강제하지 않는 만큼, 위 "구분 기준" 표(부작용 여부)로 매번 판단해야 한다.

## 주의사항 / 안티패턴

- **Domain Service 남발**: Entity 메서드 하나로 충분한 로직까지 Domain Service로 빼면 [[Entity]]가 getter/setter만 남는 Anemic Domain Model이 된다. 하나의 Entity 안에서 자연스럽게 풀리면 그냥 Entity 메서드로 둔다.
- **Domain Service에서 저장까지 처리**: Repository의 `save()`를 Domain Service 안에서 호출하기 시작하면 사실상 Application Service가 된 것이다. 저장은 항상 호출하는 쪽(Action/Application Service)의 책임으로 남긴다.
- **Laravel Facade 의존**: `Auth::user()`, `Cache::get()` 같은 Facade를 Domain Service에서 쓰면 [[Layered Architecture]]가 요구하는 "Domain은 프레임워크에 의존하지 않음" 규칙이 깨진다.
- **이름은 Service인데 실제로는 Repository 조회 껍데기**: 계산/판단 로직 없이 Repository 메서드를 그대로 위임만 하는 클래스는 Domain Service가 아니라 불필요한 간접 레이어다.

## 참고

- [[Application Service]] — 부작용(저장, 트랜잭션)을 책임지는 대응 개념
- [[Action Pattern]] — Martin Joo 스타일에서 Application Service 역할을 대신하는 패턴
- [[Entity]] — 로직이 하나의 Entity에 자연스럽게 속하면 Domain Service 대신 Entity 메서드를 우선 고려
- [[Aggregate]] — Domain Service가 여러 Aggregate를 참조할 때도 쓰기(수정)는 하나만 허용되는 규칙은 동일하게 적용됨
- [[Layered Architecture]] — Domain Service가 속하는 레이어와 의존성 규칙
- [[Directory Structure]] — BC별 4레이어 구조와 Martin Joo 2단 구조에서의 배치 차이
- [[Design Philosophy]] — "얕은 DDD" 사례(LCA API)가 채택한 `Actions/`+`Services/` 형제 폴더 구조
