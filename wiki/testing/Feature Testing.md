---
title: Feature Testing
category: testing
tags: [testing, feature-test, http-test]
related: [[Domain Testing]], [[Testing Complex Features]]
---

# Feature Testing

라라벨 애플리케이션을 HTTP 요청 수준에서 엔드투엔드로 검증하는 테스트. `TestCase`가 애플리케이션 인스턴스 전체를 부트스트랩해, 실제 요청/응답 사이클과 거의 동일하게 동작한다.

## 핵심 개념

- 모든 애플리케이션 테스트는 `Illuminate\Foundation\Testing\TestCase`를 상속한 `tests/TestCase.php`를 거친다. 이 클래스는 각 테스트마다 애플리케이션을 "새로고침"해 테스트 간 데이터가 남지 않도록 한다.
- `InteractsWithContainer`, `MakesHttpRequests`, `InteractsWithConsole` 같은 트레이트가 커스텀 어서션과 테스트 메서드를 제공한다.
- PHPUnit, 목커리(Mockery), 페이커(Faker, 더미 데이터 생성)가 기본 통합되어 있어 별도 설정 없이 바로 테스트를 작성할 수 있다.

## Laravel 구현

### HTTP 요청 메서드

```php
$this->get($uri, $headers = [])
$this->post($uri, $data = [], $headers = [])
$this->put($uri, $data = [], $headers = [])
$this->patch($uri, $data = [], $headers = [])
$this->delete($uri, $data = [], $headers = [])
```

각 호출은 `Illuminate\Foundation\Testing\TestResponse` 인스턴스(`$response`)를 반환하며, 어서션 메서드를 체이닝할 수 있다.

```php
public function test_it_stores_new_packages()
{
    $response = $this->post(route('packages.store'), [
        'name' => 'The greatest package',
    ]);

    $response->assertOk();
}
```

### JSON API 테스트

```php
$this->getJson($uri, $headers = [])
$this->postJson($uri, $data = [], $headers = [])
```

기본 HTTP 호출과 동일하게 동작하되 `Accept`, `Content-Length`, `Content-Type` 헤더가 JSON에 맞게 자동 설정된다.

```php
public function test_the_api_route_stores_new_packages()
{
    $response = $this->postJson(route('api.packages.store'), [
        'name' => 'The greatest package',
    ], ['X-API-Version' => '17']);

    $response->assertOk();
}
```

## DDD 관점에서의 활용

Feature Test는 컨트롤러 → Form Request → Application Service → Repository → DB까지 이어지는 **전체 스택의 배선**이 올바른지 확인하는 데 적합하다. 계층별 테스트 전략을 이렇게 분리한다.

| 테스트 종류 | 검증 대상 |
|-------------|-----------|
| [[Domain Testing]] | 순수 도메인 로직/불변식 (프레임워크 없이 빠르게) |
| Integration Testing | Repository 구현체 ↔ 실제 DB 연동 (별도 페이지 예정) |
| Feature Testing | HTTP 요청 → 최종 응답까지의 전체 흐름, 라우팅/미들웨어/인가/직렬화 포함 |
| [[Testing Complex Features]] | 이벤트/큐/시간 의존 기능의 블랙박스 통합 테스트 |

도메인 로직의 모든 경우의 수를 Feature Test로 검증하려 하면 느리고 장황해진다. 핵심 비즈니스 규칙 분기는 [[Domain Testing]]으로 촘촘히 다루고, Feature Test는 "정상 경로가 200을 반환한다", "권한 없으면 403을 반환한다" 같은 대표 시나리오 위주로 작성하는 것이 효율적이다.

## 주의사항 / 안티패턴

- Feature Test에서 도메인 로직의 모든 엣지 케이스를 반복 검증하지 말 것 — 느리고 유지보수 비용이 크다. 대표 경로만 커버하고 세부 로직은 Domain Test로 내린다.
- 테스트 간 DB 상태가 새어나가지 않도록 `RefreshDatabase` 등 테스트 트레이트를 사용한다.

## 참고

- [[Domain Testing]] — 도메인 레이어 단위 테스트
- [[Testing Complex Features]] — 이벤트/큐/시간 의존 기능의 블랙박스 통합 테스트
- 소스: 처음부터 제대로 배우는 라라벨, 12장 테스트
