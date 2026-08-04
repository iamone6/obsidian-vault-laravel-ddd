---
title: Repository
category: ddd-core
tags: [ddd, repository, persistence]
related: [[Aggregate]], [[Eloquent and DDD]], [[Repository Implementation]], [[Container Binding Attributes]]
---

# Repository

[[Aggregate]] Root의 저장과 조회를 추상화하는 인터페이스. 도메인 레이어는 인터페이스만 알고, 구현체는 인프라 레이어에 위치한다.

## 핵심 개념

- Aggregate Root 하나당 Repository 하나
- 도메인 레이어에는 **인터페이스**만 존재
- 구현체(Eloquent, Doctrine 등)는 인프라 레이어
- 컬렉션처럼 동작해야 한다: `add`, `findById`, `remove`

## 인터페이스 (도메인 레이어)

```php
// Domain/Repository/OrderRepository.php
interface OrderRepository
{
    public function findById(OrderId $id): ?Order;
    public function findByCustomer(CustomerId $customerId): OrderCollection;
    public function save(Order $order): void;
    public function remove(Order $order): void;
    public function nextIdentity(): OrderId;
}
```

## Eloquent 구현체 (인프라 레이어)

```php
// Infrastructure/Persistence/EloquentOrderRepository.php
final class EloquentOrderRepository implements OrderRepository
{
    public function __construct(
        private readonly OrderMapper $mapper,
        private readonly Dispatcher $events,
    ) {}

    public function findById(OrderId $id): ?Order
    {
        $record = OrderModel::with('items')->find($id->toString());
        return $record ? $this->mapper->toDomain($record) : null;
    }

    public function save(Order $order): void
    {
        DB::transaction(function () use ($order) {
            $data = $this->mapper->toPersistence($order);
            OrderModel::updateOrCreate(['id' => $data['id']], $data);

            // OrderItem 동기화
            OrderItemModel::where('order_id', $data['id'])->delete();
            foreach ($data['items'] as $item) {
                OrderItemModel::create($item);
            }

            // 도메인 이벤트 발행
            foreach ($order->pullDomainEvents() as $event) {
                $this->events->dispatch($event);
            }
        });
    }

    public function nextIdentity(): OrderId
    {
        return OrderId::generate();
    }
}
```

## Service Provider 바인딩

```php
// OrderServiceProvider.php
class OrderServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(
            OrderRepository::class,
            EloquentOrderRepository::class
        );
    }
}
```

Laravel 13.22+에서는 이 바인딩만 있는 프로바이더 대신 `OrderRepository` 인터페이스 위에 `#[Bind(EloquentOrderRepository::class)]`를 붙여 선언할 수도 있다 — 자세한 내용은 [[Container Binding Attributes]] 참고.

## 쿼리 전용 리포지토리 (Read Model)

복잡한 목록 조회는 도메인 모델을 거치지 않고 직접 쿼리하는 Read Model을 별도 구성한다.

```php
// Application/Query/OrderQueryRepository.php
interface OrderQueryRepository
{
    public function findPaginatedByCustomer(
        CustomerId $customerId,
        int $page,
        int $perPage
    ): LengthAwarePaginator;
}

// Infrastructure/Persistence/EloquentOrderQueryRepository.php
final class EloquentOrderQueryRepository implements OrderQueryRepository
{
    public function findPaginatedByCustomer(
        CustomerId $customerId,
        int $page,
        int $perPage
    ): LengthAwarePaginator {
        return OrderModel::query()
            ->where('customer_id', $customerId->toString())
            ->with('items')
            ->orderByDesc('created_at')
            ->paginate($perPage, page: $page);
    }
}
```

## 주의사항 / 안티패턴

- **Repository에 비즈니스 로직**: `confirmOrder()` 같은 메서드를 Repository에 두면 안 된다. 조회/저장만.
- **Entity가 아닌 Eloquent 모델 반환**: Repository는 도메인 Entity를 반환해야 한다. Eloquent 모델을 직접 반환하면 도메인이 Eloquent에 의존.
- **N+1 문제**: `findAll()`에서 Lazy Loading으로 관계를 로딩하면 N+1이 발생한다. `with()`로 Eager Loading.

## 논쟁: Repository가 Laravel에 잘 맞는가?

Repository 패턴은 Laravel/PHP 커뮤니티에서 호불호가 갈린다. 『Domain-Driven Design with Laravel』(Martin Joo)이 제기하는 비판:

- 모델 하나당 Repository 하나를 두는 관행이 일반적인데, 이는 결국 쿼리를 모델에서 Repository로 "옮기는" 것에 불과하다 — 6개월 뒤 `ProductRepository`가 5000줄짜리 괴물이 되는 것은 `Product` 모델이 그렇게 되는 것과 동일한 문제다.
- Repository를 쓰기로 했다면 원칙상 모든 쿼리가 그 안에 있어야 하는데, 그러면 딱 한 번만 쓰이는 쿼리 메서드까지 쌓이게 된다. 반대로 일부만 넣으면 쿼리가 컨트롤러/모델/Repository에 걸쳐 흩어져 일관성이 깨진다.
- "DB 엔진 교체 용이성"이라는 이론적 장점은 실무에서 거의 발동되지 않는다 (수년간 실제로 DB 엔진 교체를 요청받은 적이 없다는 저자의 경험).
- Repository의 실질적 장점은 여러 테이블에 걸친 작은 기능(예: 6개 테이블짜리 이슈 트래커 모듈)을 모델 단위가 아니라 기능 단위로 하나의 클래스에 모을 수 있다는 점 정도다.

이런 이유로 저자는 Laravel에서는 [[Custom Query Builder]](`newEloquentBuilder()` 오버라이드)를 Repository의 실용적 대안으로 제안한다 — Eloquent의 체이닝을 그대로 살리면서 자주 쓰는 쿼리를 재사용 가능한 스코프로 추출할 수 있다.

**이 위키의 입장**: 도메인 레이어를 Eloquent로부터 완전히 격리해야 하는 프로젝트(순수 Entity, 테스트에서 인프라 목킹 필수 등)에서는 여전히 Repository 인터페이스 + [[Mapper Pattern]] 조합이 유효하다. 반면 Eloquent 모델을 도메인 모델로 겸용하는 실용적 접근(전략 1, [[Eloquent and DDD]] 참고)을 택했다면 Custom Query Builder가 더 자연스럽다.

## 참고

- [[Repository Implementation]] — 구체적인 구현 패턴
- [[Custom Query Builder]] — Laravel다운 대안과 트레이드오프
- [[Mapper Pattern]] — Eloquent 모델 ↔ 도메인 엔티티 변환
- [[Eloquent and DDD]] — Eloquent와 DDD 임피던스 불일치
- [[Design Philosophy]] — "일관성 > 이론적 순수성"이라는 실용주의 원칙
- [[Container Binding Attributes]] — Service Provider 바인딩을 attribute로 대체하는 방법
