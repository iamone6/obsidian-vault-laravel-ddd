---
title: Laravel Events
category: laravel
tags: [laravel, event, listener, domain-event]
related: [[Domain Event]], [[Application Service]], [[Repository]], [[Eloquent Model Attributes]]
---

# Laravel Events

라라벨 프레임워크가 제공하는 관찰자(Observer)/발행-구독(pub-sub) 패턴 구현체. "무언가 일어났다"는 사실을 애플리케이션 전역에 알리는 인프라 메커니즘이다.

## 핵심 개념

- 잡(Job)이 "무엇을 해야 하는지"를 호출자가 지시하는 것이라면, 이벤트는 "무슨 일이 일어났는지"를 알리는 것이다.
- 이벤트 자체는 데이터를 캡슐화하는 객체일 뿐, 로직을 수행하지 않는다 (`handle()`이나 `fire()` 메서드가 없다). 실제 처리는 **이벤트 리스너**가 담당한다.
- 이벤트는 리스너가 0개일 수도, 여러 개일 수도 있다 — 이벤트는 리스너의 존재 여부를 신경 쓰지 않는다.
- 일부 이벤트(Eloquent 모델의 저장/생성/삭제 등)는 프레임워크가 자동으로 발생시키고, 나머지는 애플리케이션 코드에서 직접 발생시킨다.

## Laravel 구현

```php
class UserSubscribed
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly User $user,
        public readonly Plan $plan,
    ) {}
}
```

이벤트를 발생시키는 세 가지 방법:

```php
Event::fire(new UserSubscribed($user, $plan));
// 또는
app(Illuminate\Contracts\Events\Dispatcher::class)->fire(new UserSubscribed($user, $plan));
// 또는 (가장 흔히 사용)
event(new UserSubscribed($user, $plan));
```

`php artisan make:event UserSubscribed`로 생성하고, `EventServiceProvider`(또는 라라벨 11+ 자동 탐색)에 리스너를 등록한다.

## DDD 관점에서의 활용

Laravel Events는 [[Domain Event]] 개념을 실제로 전달하는 **배선(wiring) 메커니즘**이다. 개념적으로 구분해야 한다.

| 구분 | 역할 |
|------|------|
| [[Domain Event]] | 도메인에서 발생한 사실 자체 (예: `OrderConfirmed`) — 프레임워크 독립적인 순수 객체여야 한다 |
| Laravel Event | 그 사실을 애플리케이션 전역에 전파하는 라라벨 인프라 메커니즘 |

[[Repository]]의 `save()` 구현에서 흔히 볼 수 있는 패턴:

```php
public function save(Order $order): void
{
    DB::transaction(function () use ($order) {
        // ... 영속화 ...

        foreach ($order->pullDomainEvents() as $event) {
            $this->events->dispatch($event); // 도메인 이벤트를 라라벨 Dispatcher로 전달
        }
    });
}
```

이렇게 하면 도메인 계층은 `event()` 헬퍼나 `Event` 퍼사드를 몰라도 되고, 인프라 레이어(Repository 구현체)만 라라벨 이벤트 시스템과 도메인을 연결한다.

## `#[ObservedBy]` — Observer 등록을 모델 선언부로 (Laravel 10.44+)

`Model::observe()`를 서비스 프로바이더에서 호출하는 대신, 모델 클래스 위에 직접 붙일 수 있다.

```php
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([PostObserver::class, SearchIndexObserver::class])]
class Post extends Model {}
```

반복 가능(`IS_REPEATABLE`)하며, `Model`을 바로 상속하지 않고 중간 부모가 있는 경우(grandchild) 부모 체인의 옵저버를 전부 누적한다 — 같은 계열의 `#[ScopedBy]`(글로벌 스코프 등록)도 동일한 누적 규칙을 따르되 trait에 붙은 것까지 수집한다는 점이 다르다. 자세한 내용은 [[Eloquent Model Attributes]] 참고.

## 주의사항 / 안티패턴

- 도메인 이벤트를 도메인 계층 안에서 직접 `event()`로 발행하지 말 것 — 프레임워크 의존성이 도메인 코어로 침투한다. 대신 Aggregate가 이벤트를 내부에 쌓아두고(`pullDomainEvents()`), 인프라 레이어(Repository)가 저장 성공 후 실제로 dispatch한다.
- 트랜잭션 커밋 전에 이벤트를 dispatch하면, 트랜잭션이 롤백될 경우 "일어나지 않은 일"에 대한 이벤트가 이미 리스너에 전달되는 문제가 생길 수 있다. 커밋 이후 dispatch하거나 라라벨의 `afterCommit` 큐 옵션을 고려한다.

## 참고

- [[Domain Event]] — 이벤트가 전달하는 도메인 개념
- [[Repository]] — 영속화와 이벤트 발행을 함께 처리하는 지점
- [[Eloquent Model Attributes]] — `#[ObservedBy]`/`#[ScopedBy]`를 포함한 모델 설정 attribute 전체 목록
- 소스: 처음부터 제대로 배우는 라라벨, 16장 큐·잡·이벤트·브로드캐스팅·스케줄러
