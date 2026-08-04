---
title: Service Provider
category: laravel
tags: [laravel, service-provider, bootstrapping, container]
related: [[Container]], [[Directory Structure]], [[Repository]], [[Laravel Events]], [[Container Binding Attributes]]
---

# Service Provider

애플리케이션 부트스트랩 코드를 기능 단위로 캡슐화하는 클래스. 라라벨 커널은 요청을 처리하기 전에 등록된 모든 서비스 프로바이더를 통해 부트스트랩 과정을 거친다.

## 핵심 개념

- 커널(HTTP 커널 / 콘솔 커널)은 요청을 받아 미들웨어에 전달하기 전에 부트스트랩 과정을 수행하며, 그 대부분이 서비스 프로바이더로 구성된다.
- 서비스 프로바이더는 두 가지 메서드를 가진다.
  - `register()`: [[Container]]에 바인딩(클래스/인터페이스의 인스턴스 생성 방법)을 등록. 다른 프로바이더의 바인딩에 의존하는 로직을 넣으면 안 된다.
  - `boot()`: 모든 프로바이더의 `register()`가 끝난 뒤 호출. 이벤트 리스너 바인딩, 라우트 정의 등 부트스트랩 완료 이후 필요한 작업 수행.
- `AuthServiceProvider`, `RouteServiceProvider`처럼 라라벨 기본 프로바이더도 각 서브시스템의 부트스트랩을 캡슐화한 것이다.

## Laravel 구현

```php
class GitHubServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register()
    {
        $this->app->singleton(GitHubClient::class, function ($app) {
            return new GitHubClient();
        });
    }

    // register()만 있고 boot()가 필요 없다면 등록 자체를 지연시킬 수 있다.
    public function provides()
    {
        return [GitHubClient::class];
    }
}
```

- `DeferrableProvider` 인터페이스(라라벨 5.8+, 이전 버전은 `protected $defer = true`)와 `provides()`를 함께 쓰면, 컨테이너가 실제로 해당 클래스를 요청할 때까지 프로바이더 등록 자체를 미뤄 부트스트랩 시간을 줄인다.
- 컴포저로 설치한 외부 패키지도 자체 서비스 프로바이더로 라라벨에 통합되는 경우가 많다.

## DDD 관점에서의 활용

Bounded Context(모듈)마다 전용 서비스 프로바이더를 두고, 그 안에서 도메인 [[Repository]] 인터페이스 → Eloquent 구현체 바인딩을 등록하면 인프라 조립 코드를 한 곳에 모을 수 있다.

```php
class OrderServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(OrderRepository::class, EloquentOrderRepository::class);
    }

    public function boot(): void
    {
        // 도메인 이벤트 → 라라벨 리스너 바인딩 등
    }
}
```

## `register()`가 바인딩 등록뿐이라면 attribute로 대체 가능

`register()`에 인터페이스-구현체 바인딩(`$this->app->bind(...)`)만 나열되어 있고 다른 부트스트랩 로직이 없다면, Laravel 13.22+의 `#[Bind]`/`#[BindWhen]` attribute로 대체해 인터페이스 옆에 바인딩을 선언할 수 있다 — 자세한 내용과 트레이드오프는 [[Container Binding Attributes]] 참고. 반대로 `provides()`를 통한 지연 로딩(`DeferrableProvider`)처럼 프로바이더 고유의 기능이 필요하다면 여전히 프로바이더가 필요하다.

## 주의사항 / 안티패턴

- `register()` 안에서 다른 서비스의 완성된 인스턴스를 사용해 무언가를 처리하지 말 것 — 그 시점에 다른 프로바이더의 바인딩이 아직 등록되지 않았을 수 있다. 그런 로직은 `boot()`에 둔다.
- 서비스 프로바이더에 비즈니스 로직을 두지 말 것. 오직 부트스트랩/배선(wiring) 코드만.

## 참고

- [[Container]] — 서비스 프로바이더가 바인딩을 등록하는 대상
- [[Repository]] — 인터페이스-구현체 바인딩 예시
- [[Third-Party Service Integration]] — 전용 서비스 프로바이더로 외부 API 클라이언트를 바인딩하는 실전 예시
- [[Container Binding Attributes]] — 순수 바인딩 등록 코드를 attribute로 대체하는 방법
- 소스: 처음부터 제대로 배우는 라라벨, 10장 요청·응답·미들웨어, 11장 컨테이너
