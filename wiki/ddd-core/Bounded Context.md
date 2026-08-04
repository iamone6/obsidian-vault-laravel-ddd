---
title: Bounded Context
category: ddd-core
tags: [ddd, bounded-context, context-map]
related: [[Ubiquitous Language]], [[Aggregate]], [[Directory Structure]]
---

# Bounded Context

모델이 유효하게 적용되는 명확한 경계. 하나의 Bounded Context 안에서는 [[Ubiquitous Language]]가 일관성 있게 사용된다.

## 핵심 개념

- 큰 도메인을 여러 개의 독립적 컨텍스트로 분리한다
- 각 컨텍스트는 자체적인 모델, 언어, 규칙을 가진다
- 컨텍스트 간 통신은 명시적인 인터페이스(이벤트, API)를 통한다

### Context Map 관계 유형

| 패턴 | 설명 |
|------|------|
| Shared Kernel | 두 컨텍스트가 공통 모델 공유 |
| Customer/Supplier | 상위 컨텍스트가 하위에 서비스 제공 |
| Conformist | 하위가 상위 모델을 그대로 따름 |
| Anti-Corruption Layer (ACL) | 외부 모델로부터 내부 모델 보호 |
| Published Language | 표준화된 공개 언어로 통신 |
| Separate Ways | 완전 독립, 통합 없음 |

## Laravel 구현

Laravel에서 Bounded Context는 보통 최상위 네임스페이스 + 디렉토리로 표현한다.

```
app/
├── Catalog/        ← 상품 카탈로그 컨텍스트
├── Order/          ← 주문 컨텍스트
├── Payment/        ← 결제 컨텍스트
└── Identity/       ← 인증·회원 컨텍스트
```

각 컨텍스트 내부 구조:

```
app/Order/
├── Domain/
│   ├── Model/          ← Entities, Value Objects, Aggregates
│   ├── Event/          ← Domain Events
│   ├── Repository/     ← Repository interfaces
│   └── Service/        ← Domain Services
├── Application/
│   ├── Command/        ← Commands + Handlers (CQRS)
│   ├── Query/          ← Queries + Handlers (CQRS)
│   └── Service/        ← Application Services
└── Infrastructure/
    ├── Persistence/    ← Repository implementations
    └── Messaging/      ← Event dispatching
```

## Anti-Corruption Layer 예제

외부 결제 API의 모델이 내부 도메인 모델을 오염시키지 않도록 ACL로 변환한다.

```php
// Infrastructure/PaymentGateway/StripeAdapter.php
final class StripeAdapter implements PaymentGatewayInterface
{
    public function charge(PaymentIntent $intent): PaymentResult
    {
        // Stripe의 외부 모델 → 내부 도메인 모델로 변환
        $stripeResponse = $this->stripe->paymentIntents->create([...]);
        return PaymentResult::fromStripeResponse($stripeResponse);
    }
}
```

## 주의사항 / 안티패턴

- **God Context**: 모든 것을 하나의 컨텍스트에 넣는 것. 컨텍스트 분리의 의미가 사라진다.
- **과도한 분리**: 너무 세분화하면 컨텍스트 간 통합 비용이 급증한다. 팀 크기·배포 단위와 맞출 것.
- **공유 데이터베이스 테이블**: 컨텍스트 경계를 DB 수준에서 무시하면 경계가 형식적으로만 존재한다.

## 참고

- [[Directory Structure]] — Laravel에서 컨텍스트를 디렉토리로 구성하는 방법
- [[Layered Architecture]] — 각 컨텍스트 내부의 레이어 구조
- [[Design Philosophy]] — 컨텍스트 경계가 지키려는 언어 우선 원칙
