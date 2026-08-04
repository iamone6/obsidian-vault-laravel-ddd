---
title: Pipeline Pattern (미들웨어)
category: patterns
tags: [laravel, middleware, pipeline]
related: [[Layered Architecture]], [[Container]], [[Form Request]]
---

# Pipeline Pattern (미들웨어)

요청/응답이 여러 겹의 계층(미들웨어)을 순서대로 통과하며 각 계층이 처리를 하거나 거부할 수 있게 하는 아키텍처 패턴. 라라벨 미들웨어가 이 패턴의 구현체다.

## 핵심 개념

미들웨어는 애플리케이션이 케이크나 양파처럼 여러 겹으로 감싸져 있다는 개념이다. 모든 요청은 애플리케이션 핵심 로직에 도달하기 전에 모든 미들웨어 계층을 통과하고, 응답은 최종 사용자에게 돌아가는 길에 미들웨어 계층을 반대 순서로 다시 통과한다.

- 미들웨어는 애플리케이션 핵심 로직과 구분되며, 대개 어떤 애플리케이션에도 적용 가능한 방식으로 구성된다 (예: 속도 제한, 인증 확인, CSRF 보호, 세션 시작).
- 미들웨어는 요청 단계에서 조회/거부하거나, 응답 단계에서 추가 작업(쿠키 추가 등)을 할 수 있다.
- 요청/응답 사이클의 처음과 끝 모두에 관여할 수 있는 것이 미들웨어의 강력한 지점이다 (세션 시작/종료 같은 작업에 적합).

## Laravel 구현

```php
class BanDeleteMethod
{
    public function handle($request, Closure $next)
    {
        if ($request->method() === 'DELETE') {
            return response('금지된 메서드!', 403);
        }

        // 요청을 다음 미들웨어(혹은 최종적으로 핵심 로직)로 넘긴다.
        $response = $next($request);

        // 핵심 로직 실행 후, 응답이 반환되기 직전에 추가 작업 가능.
        $response->cookie('visited-our-site', true);

        return $response;
    }
}
```

`handle($request, Closure $next)`가 이해의 핵심이다:
- `$next($request)` 호출 전 코드 = 요청이 핵심 로직에 도달하기 전 처리 (미들웨어 스택을 따라 "내려가는" 방향)
- `$next($request)` 호출 후 코드 = 핵심 로직이 반환한 응답에 대한 처리 (미들웨어 스택을 "거슬러 올라오는" 방향)

`php artisan make:middleware BanDeleteMethod`로 스텁을 생성하고, `app/Http/Kernel.php`(또는 라라벨 11+의 `bootstrap/app.php`)에 등록한다.

## DDD 관점에서의 활용

미들웨어는 인증, 속도 제한, 로깅, CORS, 응답 헤더 조작 같은 **횡단 관심사(cross-cutting concern)**를 처리하는 데 적합하다. 이런 관심사를 [[Application Service]]나 도메인 계층에 섞지 않고 HTTP 경계에 고정해두면 [[Layered Architecture]]가 유지된다.

- 도메인/애플리케이션 계층은 "요청이 어떤 미들웨어를 거쳤는지" 알 필요가 없어야 한다.
- 인가(authorization)의 아주 단순한 케이스(예: `can:` 미들웨어)는 미들웨어에서 처리해도 되지만, 복잡한 도메인 규칙 기반 인가는 [[Policy and Gate]]나 도메인 서비스로 위임한다.

## 주의사항 / 안티패턴

- 미들웨어에서 비즈니스 로직(예: 주문 확정 가능 여부 판단)을 처리하지 말 것 — 재사용성과 테스트 용이성이 떨어진다.
- 미들웨어 순서에 의존하는 로직을 만들 때는 등록 순서를 명확히 문서화한다. 순서가 바뀌면 동작이 달라질 수 있다.

## 참고

- [[Layered Architecture]] — 미들웨어가 속하는 위치(프레젠테이션/인프라 경계)
- [[Container]] — 미들웨어도 컨테이너를 통해 의존성을 주입받을 수 있음
- 소스: 처음부터 제대로 배우는 라라벨, 10장 요청·응답·미들웨어
