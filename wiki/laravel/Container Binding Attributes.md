---
title: Container Binding Attributes
category: laravel
tags: [laravel, container, attributes, php8, dependency-injection, dip]
related: [[Container]], [[Service Provider]], [[Repository]], [[Eloquent Model Attributes]]
---

# Container Binding Attributes

`Illuminate\Container\Attributes\Bind` / `BindWhen`. 인터페이스 ↔ 구현체 바인딩을 [[Service Provider]]의 `register()` 대신, **인터페이스 선언부 바로 위**에 PHP native attribute로 선언하는 방법. `#[BindWhen]`은 Laravel 13.22(2026-07-24)에 추가되었고 **PHP 8.5 이상**이 필요하다.

## 핵심 개념

기존 방식은 인터페이스와 구현체의 관계가 서비스 프로바이더 파일에 흩어져 있어, 인터페이스 파일만 보고는 실제로 무엇이 주입되는지 알기 어렵다.

```php
// 기존 방식 — app/Providers/AppServiceProvider.php
public function register(): void
{
    $this->app->bind(EventPusher::class, RedisEventPusher::class);
}
```

`#[Bind]`/`#[BindWhen]`은 이 등록 코드를 인터페이스 옆으로 옮겨 "이 인터페이스는 이 구현체로 해석된다"를 한눈에 보이게 한다. [[Eloquent Model Attributes]]가 다루는 모델 설정 attribute들과 같은 흐름(설정을 선언부로 옮기는 것) 위에 있다.

## Laravel 구현

### `#[Bind]` — 조건이 "환경"뿐일 때

```php
use Illuminate\Container\Attributes\Bind;

#[Bind(RedisEventPusher::class)]
interface EventPusher
{
    public function push(string $event): void;
}

// 생성자에 인터페이스만 타입힌트하면 컨테이너가 자동으로 해석
class WebhookController extends Controller
{
    public function __construct(protected EventPusher $pusher) {}
}
```

환경별로 다른 구현체가 필요하면 `#[Bind]`를 여러 번 겹쳐 쓴다. **위에서부터 순서대로 평가**되므로 조건 있는 것을 위에, 조건 없는 기본값을 맨 아래에 둔다.

```php
#[Bind(FakeEventPusher::class, environments: ['local', 'testing'])]
#[Bind(RedisEventPusher::class)]
interface EventPusher {}
```

### `#[BindWhen]` — 조건이 "환경" 이상일 때 (13.22+)

`#[Bind]`의 `environments` 옵션 하나로는 "설정값에 따라", "DB 피처 플래그가 켜져 있으면" 같은 조건을 표현할 수 없다. `#[BindWhen]`은 컨테이너를 인자로 받는 **클로저**를 조건으로 받는다.

```php
use Illuminate\Container\Attributes\Bind;
use Illuminate\Container\Attributes\BindWhen;

#[BindWhen(BetaPaymentGateway::class, static function ($app) {
    return $app->make('config')->get('features.payments.beta');
})]
#[Bind(FakePaymentGateway::class, environments: ['local', 'testing'])]
#[Bind(StripePaymentGateway::class)] // 어떤 조건에도 안 걸리면 기본값, 반드시 맨 아래
interface PaymentGatewayInterface
{
    public function charge(int $amountInCents): void;
}
```

어트리뷰트 인자로 클로저를 넘기는 PHP 문법 자체가 PHP 8.5부터 지원되므로, 프로젝트가 PHP 8.5 미만이면 `#[BindWhen]`을 쓸 수 없다(`#[Bind]`는 클로저를 쓰지 않으므로 이전 버전에서도 가능).

### `#[Singleton]` / `#[Scoped]`와 조합

```php
use Illuminate\Container\Attributes\Bind;
use Illuminate\Container\Attributes\Singleton;

#[Bind(RedisEventPusher::class)]
#[Singleton]
interface EventPusher {}
```

- `#[Singleton]` — 애플리케이션 생명주기 동안 한 번만 생성 (`$this->app->singleton()`과 동일)
- `#[Scoped]` — 요청/잡(job) 1회 처리 동안만 유지 (Octane, 큐 워커 환경에서 유용)

## 선택 기준

| 상황 | 선택 |
|---|---|
| 구현체가 `local`/`staging`/`production` 등 환경별로만 갈릴 때 | `#[Bind(..., environments: [...])]` |
| 구현체 선택이 설정값·DB 피처 플래그·요금제 등 런타임 조건에 좌우될 때 | `#[BindWhen]` |
| 인터페이스 하나에 구현체가 딱 하나뿐일 때 | 조건 없는 `#[Bind]` 하나만 |
| 인스턴스를 요청/잡마다 한 번만 만들고 싶을 때 | 위 attribute + `#[Singleton]` 또는 `#[Scoped]` |

## DDD 관점에서의 활용

[[Repository]] 인터페이스 ↔ Eloquent 구현체 바인딩처럼 [[Service Provider]]의 `register()`에 흔히 쌓이는 순수 배선(wiring) 코드는 `#[Bind]`로 옮길 수 있는 후보다.

```php
// 기존: OrderServiceProvider::register()
$this->app->bind(OrderRepository::class, EloquentOrderRepository::class);

// attribute 방식: OrderRepository 인터페이스 선언부로 이동
#[Bind(EloquentOrderRepository::class)]
interface OrderRepository { /* ... */ }
```

다만 `#[BindWhen]`의 조건 클로저 안에 복잡한 도메인 판단을 직접 넣기 시작하면, 인터페이스 파일이 다시 "숨겨진 배선 로직"의 새 위치가 될 뿐이다 — 조건이 단순 환경/설정 분기를 넘어서면 여전히 Service Provider의 `register()`나 별도 팩토리로 옮기는 편이 낫다.

## 주의사항 / 안티패턴

- **선언 순서가 곧 평가 순서다**: 조건부 바인딩을 아래에, 기본값을 위에 두면 기본값이 항상 먼저 채택되어 조건부 바인딩이 죽은 코드가 된다.
- **`#[BindWhen]`의 클로저에 무거운 로직을 넣지 말 것**: 클로저는 컨테이너가 해당 인터페이스를 해석할 때마다 평가될 수 있다. DB/HTTP 호출 같은 비용이 큰 작업은 피한다.
- **PHP 8.5 미만 프로젝트**: `#[BindWhen]`을 쓸 수 없으므로 여러 개의 `#[Bind(..., environments:)]` 조합이나 기존 Service Provider 방식을 유지한다.

## 참고

- [[Container]] — 오토와이어링과 명시적 바인딩의 기본 개념
- [[Service Provider]] — attribute로 옮기기 전, 바인딩이 등록되던 기존 위치
- [[Repository]] — 인터페이스-구현체 바인딩의 대표적 활용 사례
- [[Eloquent Model Attributes]] — 같은 "설정을 선언부로 옮기는" 흐름의 모델 설정 attribute 24종
- 소스: Laravel `#[Bind]` / `#[BindWhen]` 어트리뷰트 정리, Laravel 13.x 공식 문서
