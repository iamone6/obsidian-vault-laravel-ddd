---
title: Directory Structure
category: laravel
tags: [laravel, ddd, structure, module]
related: [[Bounded Context]], [[Layered Architecture]]
---

# Directory Structure

DDD를 적용한 Laravel 프로젝트의 디렉토리 구성. 기본 Laravel 구조를 확장한다.

## 권장 구조 (Bounded Context 기반)

```
app/
├── Providers/
│   └── AppServiceProvider.php
│
├── Shared/                      ← 컨텍스트 공통 요소
│   ├── Domain/
│   │   └── ValueObject/
│   │       ├── Money.php
│   │       ├── Email.php
│   │       └── Pagination.php
│   └── Infrastructure/
│       └── Persistence/
│           └── BaseEloquentRepository.php
│
├── Order/                       ← 주문 Bounded Context
│   ├── Domain/
│   │   ├── Model/
│   │   │   ├── Order.php        ← Aggregate Root
│   │   │   ├── OrderId.php
│   │   │   ├── OrderItem.php    ← Entity
│   │   │   ├── OrderStatus.php  ← Enum / Value Object
│   │   │   └── Money.php        ← Value Object (VO)
│   │   ├── Event/
│   │   │   ├── OrderCreated.php
│   │   │   └── OrderConfirmed.php
│   │   ├── Repository/
│   │   │   └── OrderRepository.php  ← Interface
│   │   ├── Service/
│   │   │   └── PricingService.php   ← Domain Service
│   │   └── Exception/
│   │       └── OrderNotFoundException.php
│   │
│   ├── Application/
│   │   ├── Command/
│   │   │   ├── CreateOrderCommand.php
│   │   │   └── CreateOrderHandler.php
│   │   ├── Query/
│   │   │   ├── GetOrderQuery.php
│   │   │   └── GetOrderHandler.php
│   │   ├── DTO/
│   │   │   └── OrderData.php
│   │   └── EventHandler/
│   │       └── SendOrderConfirmationEmail.php
│   │
│   ├── Infrastructure/
│   │   ├── Persistence/
│   │   │   ├── EloquentOrderRepository.php
│   │   │   ├── OrderModel.php       ← Eloquent Model
│   │   │   └── OrderMapper.php
│   │   └── Http/
│   │       ├── Controller/
│   │       │   └── OrderController.php
│   │       └── Request/
│   │           └── CreateOrderRequest.php
│   │
│   └── OrderServiceProvider.php
│
├── Catalog/                     ← 상품 카탈로그 컨텍스트
│   └── ...
│
└── Identity/                    ← 인증·회원 컨텍스트
    └── ...
```

## 최소 구조 (소규모 프로젝트)

전체 분리가 과도하다면 컨텍스트만 분리하고 레이어는 단순화:

```
app/
├── Order/
│   ├── Models/          ← Eloquent 모델 + 도메인 로직 혼합
│   ├── Services/        ← Application Service
│   ├── Repositories/
│   ├── Events/
│   ├── Http/
│   │   ├── Controllers/
│   │   └── Requests/
│   └── OrderServiceProvider.php
└── ...
```

## Service Provider 자동 등록

각 컨텍스트의 ServiceProvider를 `bootstrap/providers.php`에 등록:

```php
// bootstrap/providers.php
return [
    App\Providers\AppServiceProvider::class,
    App\Order\OrderServiceProvider::class,
    App\Catalog\CatalogServiceProvider::class,
    App\Identity\IdentityServiceProvider::class,
];
```

## 컨텍스트 내 Service Provider

```php
// app/Order/OrderServiceProvider.php
class OrderServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(OrderRepository::class, EloquentOrderRepository::class);
        $this->app->bind(CreateOrderHandler::class);
    }

    public function boot(): void
    {
        $this->loadMigrationsFrom(__DIR__ . '/Infrastructure/database/migrations');
        $this->loadRoutesFrom(__DIR__ . '/Infrastructure/Http/routes.php');
    }
}
```

## Domain 폴더 vs Application 폴더 (Martin Joo 스타일)

『Domain-Driven Design with Laravel』은 더 단순한 2단 분리를 제안한다: **도메인 코드**(`Domain/`)와 **애플리케이션 코드**(`app/`)를 완전히 분리하는 것이다.

```
Domain/
├── Subscriber/
│   ├── Models/
│   ├── Actions/
│   ├── DataTransferObjects/
│   ├── Builders/
│   └── ViewModels/
├── Mail/
│   └── ...
└── Shared/
    └── ValueObjects/

app/
├── Http/
│   ├── Web/Controllers/       ← Inertia 등 로그인 사용자용
│   └── Api/Controllers/       ← 외부 API 소비자용
├── Console/
│   └── Commands/
└── Providers/
```

핵심 규칙:
- `Domain/` 안에는 **비즈니스 코드만** 존재한다. `Http`나 `Controllers` 폴더가 여기 있으면 안 된다.
- HTTP(웹/API), 콘솔 커맨드처럼 "환경에 의존하는" 코드는 전부 `app/`에 둔다. 저자는 이를 별개의 **애플리케이션**(Web 앱, API 앱, Console 앱)으로 구분한다 — 세 애플리케이션이 같은 도메인 코드를 공유한다.
- 이 구조는 프레임워크 자체를 건드리지 않는다. 설정/부트스트랩 변경이 필요 없어 라라벨 업그레이드에 영향을 주지 않는다.

이 구조의 이점은 "브로드캐스트 관련 기능을 작업 중이면 `Domain/Mail` 폴더로 가면 모든 것이 있다"는 탐색 편의성이다 — 컨트롤러/모델/리퀘스트/리소스가 기술 계층별로 흩어진 기본 Laravel 구조 대비 신규 합류자의 온보딩이 쉬워진다.

### composer.json으로 도메인 폴더를 오토로드하기

도메인 폴더를 `app/` 바깥(예: `src/Domains/`)에 두고 싶다면, Laravel 부트스트랩(`bootstrap.php`, `config/*`)은 전혀 건드리지 않고 컴포저 오토로드 설정만 추가하면 된다.

```json
"autoload": {
    "psr-4": {
        "App\\": "app/",
        "Domains\\": "src/Domains/"
    }
}
```

이렇게 하면 `src/Domains/Customers/`, `src/Domains/Orders/`처럼 각 도메인 폴더가 `Models`, `Events`, `Actions` 같은 하위 구조를 갖되, Controller나 마이그레이션은 두지 않는다(그건 `app/`의 "애플리케이션" 몫이다). **핵심은 프레임워크 부트스트랩 코드를 절대 건드리지 않는 것** — 그래야 Laravel 버전을 업그레이드할 때 이 구조가 발목을 잡지 않는다.



## 참고

- [[Bounded Context]] — 컨텍스트 경계 설계
- [[Layered Architecture]] — 레이어 구조 상세
- [[Design Philosophy]] — 이 구조가 지키려는 언어 우선 원칙
- 소스: Domain-Driven Design with Laravel (Martin Joo), Domains And Applications 챕터
- 소스: Layered Architectures with Laravel (Martin Joo)
