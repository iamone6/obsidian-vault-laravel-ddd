# Laravel `#[Bind]` / `#[BindWhen]` 어트리뷰트 정리

> 서비스 컨테이너에서 인터페이스 ↔ 구현체 바인딩을 서비스 프로바이더 대신 어트리뷰트로 선언하는 방법.
> `#[BindWhen]`은 Laravel 13.22(2026-07-24)에 추가됨. PHP 8.5 이상 필요.

---

## 0. 문제 상황: 기존 방식 (서비스 프로바이더)

인터페이스를 타입힌트로 쓰려면, 원래는 서비스 프로바이더에 바인딩을 등록해야 했습니다.

```php
// app/Providers/AppServiceProvider.php

public function register(): void
{
    // EventPusher 인터페이스가 필요하면 RedisEventPusher를 주입하라고 알려줌
    $this->app->bind(EventPusher::class, RedisEventPusher::class);
}
```

이 방식의 불편한 점:

- 인터페이스와 구현체 관계가 프로바이더 파일에 흩어져 있어서, 인터페이스만 보고는 실제로 뭐가 주입되는지 알기 어려움
- 환경별로 다른 구현체를 쓰려면 `if (app()->environment(...))` 분기를 프로바이더 안에 직접 써야 함

`#[Bind]` / `#[BindWhen]`은 이 등록 코드를 **인터페이스 선언부 바로 위**로 옮겨서, 한눈에 "이 인터페이스는 이 구현체로 해석된다"를 알 수 있게 해줍니다.

---

## 1. `#[Bind]` — 조건이 "환경"뿐일 때

### 1-1. 기본 사용법

```php
namespace App\Contracts;

use App\Services\RedisEventPusher;
use Illuminate\Container\Attributes\Bind;

// EventPusher 타입힌트가 있으면 컨테이너가 자동으로 RedisEventPusher를 생성해서 넣어줌.
// 서비스 프로바이더에 별도 등록 코드가 필요 없음.
#[Bind(RedisEventPusher::class)]
interface EventPusher
{
    public function push(string $event): void;
}
```

```php
namespace App\Http\Controllers;

use App\Contracts\EventPusher;

class WebhookController extends Controller
{
    public function __construct(
        // 생성자에 인터페이스만 타입힌트하면 끝. 컨테이너가 알아서 해석.
        protected EventPusher $pusher,
    ) {}
}
```

### 1-2. 환경별 구현체 (`environments` 옵션)

```php
namespace App\Contracts;

use App\Services\FakeEventPusher;
use App\Services\RedisEventPusher;
use Illuminate\Container\Attributes\Bind;

// 위에서부터 순서대로 평가됨. local/testing이면 Fake, 그 외(운영 등)에는 Redis.
#[Bind(FakeEventPusher::class, environments: ['local', 'testing'])]
#[Bind(RedisEventPusher::class)]
interface EventPusher
{
    public function push(string $event): void;
}
```

> `#[Bind]`는 여러 개를 겹쳐 쓸 수 있습니다. 조건이 있는 것을 위에, 기본값(조건 없는 `#[Bind]`)을 맨 아래에 두는 게 안전합니다.

### 1-3. `#[Singleton]` / `#[Scoped]`와 함께 쓰기

```php
use App\Services\RedisEventPusher;
use Illuminate\Container\Attributes\Bind;
use Illuminate\Container\Attributes\Singleton;

// 이 인터페이스로 해석된 인스턴스는 요청 전체에서 단 한 번만 생성되어 재사용됨
#[Bind(RedisEventPusher::class)]
#[Singleton]
interface EventPusher
{
    public function push(string $event): void;
}
```

- `#[Singleton]` : 애플리케이션 생명주기 동안 한 번만 생성 (기존 `$this->app->singleton()`과 동일)
- `#[Scoped]` : 요청/잡(job) 1회 처리 동안만 유지 (Octane, 큐 워커 환경에서 유용)

### 1-4. `#[Bind]`의 한계

`environments` 하나만 조건으로 쓸 수 있습니다. 예를 들어 "설정 파일 값에 따라", "DB에 저장된 피처 플래그가 켜져 있으면" 같은 조건은 표현할 수 없습니다. 이런 경우를 위해 `#[BindWhen]`이 추가되었습니다.

---

## 2. `#[BindWhen]` — 조건이 "환경" 이상일 때 (Laravel 13.22+)

### 2-1. 기본 사용법

```php
namespace App\Contracts;

use App\Services\BetaPaymentGateway;
use App\Services\FakePaymentGateway;
use App\Services\StripePaymentGateway;
use Illuminate\Container\Attributes\Bind;
use Illuminate\Container\Attributes\BindWhen;

// #[BindWhen]은 컨테이너를 인자로 받는 클로저를 조건으로 사용한다.
// 클로저가 true를 반환하면 이 바인딩이 채택됨.
#[BindWhen(BetaPaymentGateway::class, static function ($app) {
    // config/features.php 에 정의된 베타 플래그를 확인
    return $app->make('config')->get('features.payments.beta');
})]
// 로컬/테스트 환경에서는 기존처럼 #[Bind]의 environments 옵션도 그대로 섞어 쓸 수 있다.
#[Bind(FakePaymentGateway::class, environments: ['local', 'testing'])]
// 어떤 조건에도 안 걸리면 이게 기본값으로 사용됨 (반드시 맨 아래에 둘 것)
#[Bind(StripePaymentGateway::class)]
interface PaymentGatewayInterface
{
    public function charge(int $amountInCents): void;
}
```

```php
namespace App\Http\Controllers;

use App\Contracts\PaymentGatewayInterface;

class CheckoutController extends Controller
{
    public function __construct(
        // 실행 시점의 조건에 따라 Beta / Fake / Stripe 중 하나가 자동으로 주입됨
        protected PaymentGatewayInterface $gateway,
    ) {}
}
```

### 2-2. 평가 순서에 대한 주의사항

- 조건은 **선언 순서대로** 위에서부터 평가됩니다.
- 조건부 바인딩(`#[BindWhen]`, `environments`가 있는 `#[Bind]`)을 위에, 조건 없는 기본 `#[Bind]`를 맨 아래에 두는 순서를 지켜야 의도한 대로 동작합니다.

### 2-3. PHP 8.5 요구사항

`#[BindWhen]`은 어트리뷰트 인자로 **클로저**를 전달합니다. PHP에서 어트리뷰트 인자에 클로저를 쓰는 문법 자체가 **PHP 8.5**부터 지원되는 기능이라, 프로젝트가 PHP 8.5 미만이면 이 어트리뷰트를 쓸 수 없습니다. (`#[Bind]`는 클로저를 쓰지 않으므로 이전 버전에서도 사용 가능)

---

## 3. 언제 뭘 쓸까 (요약)

| 상황 | 선택 |
|---|---|
| 구현체가 `local` / `staging` / `production` 등 환경별로만 갈릴 때 | `#[Bind(..., environments: [...])]` |
| 구현체 선택이 설정값, DB 피처 플래그, 요금제(플랜) 등 런타임 조건에 좌우될 때 | `#[BindWhen]` |
| 인터페이스 하나에 구현체가 딱 하나뿐일 때 | 조건 없는 `#[Bind]` 하나만 |
| 인스턴스를 요청/잡마다 한 번만 만들고 싶을 때 | 위 어트리뷰트 + `#[Singleton]` 또는 `#[Scoped]` |

---

## 참고

- [Service Container | Laravel 13.x 공식 문서](https://laravel.com/docs/13.x/container)
- [BindWhen Container Attribute in Laravel 13.22 - Laravel News](https://laravel-news.com/laravel-13-22-0)
