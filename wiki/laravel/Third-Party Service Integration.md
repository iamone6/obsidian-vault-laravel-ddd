---
title: Third-Party Service Integration
category: laravel
tags: [laravel, service, third-party-api, service-provider, container]
related: [[Container]], [[Service Provider]], [[DTO]], [[Custom Collection]]
---

# Third-Party Service Integration

외부 API(결제, 시세 조회, 이메일 발송 등)를 하나의 "미니 애플리케이션"으로 취급해 전용 폴더/클래스 구조로 캡슐화하는 패턴.

## 핵심 개념

외부 서비스를 통합할 때, 다음처럼 그 서비스 전용 하위 구조를 만든다.

```
App/Services/MarketStack/
├── MarketStackService.php     ← HTTP 요청/응답 처리 메인 클래스
├── DataTransferObjects/
│   └── DividendData.php
├── Collections/
│   └── DividendCollection.php
└── MarketStackServiceProvider.php
```

- 메인 `XyzService` 클래스가 실제 HTTP 요청을 보내고 응답을 파싱한다.
- 응답은 원시 배열이 아니라 [[DTO]]/[[Custom Collection]]으로 변환해 반환한다 — 거대한 3rd party 응답을 그대로 다루면 버그의 온상이 된다.

## Laravel 구현

### 설정값은 `config/services.php`를 통해 주입

```env
MARKET_STACK_ACCESS_KEY=YOUR_TOKEN_HERE
MARKET_STACK_URI=http://api.marketstack.com/v1/
MARKET_STACK_TIMEOUT=5
```

```php
// config/services.php
return [
    'market_stack' => [
        'access_key' => env('MARKET_STACK_ACCESS_KEY'),
        'uri' => env('MARKET_STACK_URI', 'http://api.marketstack.com/v1/'),
        'timeout' => env('MARKET_STACK_TIMEOUT', 5),
    ],
];
```

`.env`에서는 서비스 이름을 접두어로 붙이지만(`MARKET_STACK_*`), `config/services.php` 배열 키 안에서는 접두어 없이 짧게 쓰는 것이 관례다.

### 서비스 클래스

```php
class MarketStackService
{
    public function __construct(
        private readonly string $uri,
        private readonly string $accessKey,
        private readonly int $timeout,
    ) {}

    public function dividends(string $ticker): DividendCollection
    {
        $response = $this->buildRequest()
            ->get($this->uri . 'dividends', $this->buildQuery($ticker))
            ->throw();

        $items = collect($response->json('data'))
            ->map(fn (array $item) => DividendData::fromArray($item))
            ->toArray();

        return new DividendCollection($items);
    }
}
```

### 전용 서비스 프로바이더로 바인딩

생성자가 스칼라 값(문자열, 정수)을 받으므로 [[Container]]가 자동으로 해석할 수 없다 — [[Service Provider]]에서 명시적으로 바인딩해야 한다.

```php
class MarketStackServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton(MarketStackService::class, fn () => new MarketStackService(
            config('services.market_stack.uri'),
            config('services.market_stack.access_key'),
            config('services.market_stack.timeout'),
        ));
    }
}
```

이 바인딩이 없으면 `MarketStackService`를 주입하려 할 때 라라벨이 문자열 생성자 인자를 해석하지 못해 예외가 발생한다 ([[Container]] 문서의 "명시적 바인딩이 필요한 경우" 참고). 바인딩 등록은 [[Service Provider]] 컨벤션대로 `register()`에 둔다 — `boot()`에 두면 다른 프로바이더가 `register()` 단계에서 이 서비스를 먼저 resolve하려 할 때 아직 바인딩이 없어 예외가 날 수 있다(모든 프로바이더의 `register()`가 끝난 뒤에야 `boot()` 단계로 넘어가므로).

이제 다른 클래스(Action 등)는 평범하게 생성자 주입으로 사용한다.

```php
class UpdateMarketValuesAction
{
    public function __construct(
        private readonly MarketStackService $marketStackService
    ) {}
}
```

## 과유불급

모든 응답을 DTO/Collection으로 감쌀 필요는 없다. 단일 스칼라 값만 필요하면 그냥 반환한다.

```php
public function price(string $ticker): float
{
    $response = $this->buildRequest()
        ->get($this->uri . 'eod', ['limit' => 1, ...$this->buildQuery($ticker)])
        ->throw()
        ->json('data');

    return (float) $response[0]['close'];
}
```

## 주의사항 / 안티패턴

- 여러 곳에서 같은 외부 API를 직접 `Http::get()`으로 호출하지 말 것 — 인증, 타임아웃, 응답 파싱이 흩어져 중복된다. 항상 전용 Service 클래스를 경유한다.
- 서비스 클래스 생성자에 설정값을 하드코딩하지 말 것 — `config()`를 통해서만 값을 주입해 테스트 환경에서 쉽게 교체할 수 있게 한다.

## 참고

- [[Container]] — 스칼라 생성자 인자에 명시적 바인딩이 필요한 이유
- [[Service Provider]] — 외부 서비스 바인딩을 담는 위치
- [[DTO]], [[Custom Collection]] — 응답 데이터를 구조화하는 도구
- 소스: Case Study - Portfolio And Dividend Tracker (Martin Joo), Integrating 3rd Party APIs
