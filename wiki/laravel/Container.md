---
title: Container (의존성 주입 컨테이너)
category: laravel
tags: [laravel, container, dependency-injection, ioc, facade]
related: [[Service Provider]], [[Repository]], [[Application Service]], [[Container Binding Attributes]]
---

# Container (의존성 주입 컨테이너)

라라벨의 IoC(제어의 역전) 컨테이너. 클래스가 필요로 하는 의존 객체를 직접 생성하지 않고 컨테이너가 주입해준다.

## 핵심 개념

**제어의 역전(IoC)**: 전통적인 코드는 낮은 수준의 구상 클래스가 자신이 사용할 의존 객체를 직접 `new`로 생성한다. IoC는 이 방향을 뒤집어, 위쪽의 애플리케이션(설정)이 어떤 구현체를 쓸지 결정하고 아래쪽 클래스는 그저 "필요한 것을 달라"고 요청(타입힌트)만 한다.

의존성 주입 방식 세 가지:
- **생성자 주입** — 가장 흔한 방식. `__construct(Mailer $mailer)`처럼 타입힌트.
- **세터 주입** — 세터 메서드로 의존성을 받음.
- **메서드 주입** — 메서드 호출 시점에 의존성이 주입됨.

```php
class UserMailer
{
    public function __construct(protected Mailer $mailer) {}

    public function welcome($user)
    {
        return $this->mailer->mail($user->email, 'Welcome!');
    }
}
```

## Laravel 구현

### 오토와이어링(autowiring)

생성자 파라미터가 구체적인 클래스 타입힌트만 가지고 있다면(스칼라 값이 필요 없다면), 컨테이너에 별도 등록 없이도 자동으로 인스턴스를 해석해서 주입한다.

```php
class Foo
{
    public function __construct(Bar $bar, Baz $baz) {}
}

$foo = app(Foo::class); // Bar, Baz를 자동으로 생성해 주입
```

### 명시적 바인딩이 필요한 경우

생성자에 문자열/정수 같은 스칼라 파라미터가 필요하면(`$logPath`, `$minimumLogLevel` 등) 컨테이너가 값을 추론할 수 없으므로, [[Service Provider]]의 `register()`에서 바인딩 방법을 등록해야 한다.

```php
$this->app->singleton(Logger::class, function ($app) {
    return new Logger(config('logging.path'), config('logging.level'));
});
```

### 바인딩을 attribute로 선언하기 (`#[Bind]`/`#[BindWhen]`, 13.22+)

인터페이스-구현체 바인딩만 하는 경우라면, 서비스 프로바이더 대신 인터페이스 선언부 위에 attribute로 직접 표현할 수 있다.

```php
use Illuminate\Container\Attributes\Bind;

#[Bind(RedisEventPusher::class)]
interface EventPusher
{
    public function push(string $event): void;
}
```

자세한 조건부 바인딩(`environments`, `#[BindWhen]`)과 `#[Singleton]`/`#[Scoped]` 조합은 [[Container Binding Attributes]] 참고.

### 인스턴스를 가져오는 방법

```php
$logger = app(Logger::class);       // 글로벌 헬퍼
$logger = $app->make(Logger::class); // 컨테이너 인스턴스의 make()
$logger = $app[Logger::class];       // 배열 접근 문법 (동일 동작)
```

### 퍼사드(Facade)

퍼사드는 컨테이너 접근을 감싸는 정적 프록시다. 모든 퍼사드는 `getFacadeAccessor()` 메서드 하나만 가지며, 이 메서드가 반환하는 키(예: `'cache'`, `'log'`)로 컨테이너에서 실제 인스턴스를 찾아 메서드를 위임한다.

```php
Log::alert('Something has gone wrong!');
// 다음과 완전히 동일하다.
app('log')->alert('Something has gone wrong!');
```

## DDD 관점에서의 활용

- 애플리케이션 서비스나 컨트롤러에서 [[Repository]] 인터페이스를 생성자로 타입힌트하면, 컨테이너가 [[Service Provider]]에 등록된 구현체(Eloquent 기반 등)를 자동으로 주입한다 — 이것이 라라벨에서 의존성 역전 원칙(DIP)을 실현하는 메커니즘이다.
- 테스트에서는 인터페이스를 목(mock)으로 바인딩해 도메인/애플리케이션 로직을 인프라와 분리해 검증할 수 있다.

## 주의사항 / 안티패턴

- 도메인 레이어 안에서 `app()` 헬퍼나 퍼사드를 직접 호출하지 말 것 — 프레임워크 의존성이 도메인 코어까지 스며든다. 컨테이너 사용은 애플리케이션/인프라 레이어로 한정.
- 퍼사드는 정적 호출처럼 보이지만 실제로는 인스턴스 메서드 호출이므로 목킹이 가능하다. 그렇다고 도메인 코드에서 쓰는 것이 정당화되지는 않는다.

## 참고

- [[Service Provider]] — 컨테이너에 바인딩을 등록하는 위치
- [[Repository]] — 인터페이스 바인딩을 통한 DIP 적용 예시
- [[Third-Party Service Integration]] — 스칼라 생성자 인자를 갖는 외부 서비스 클래스의 실전 바인딩 예시
- [[Container Binding Attributes]] — `#[Bind]`/`#[BindWhen]`으로 바인딩을 선언부로 옮기는 방법
- 소스: 처음부터 제대로 배우는 라라벨, 11장 컨테이너
