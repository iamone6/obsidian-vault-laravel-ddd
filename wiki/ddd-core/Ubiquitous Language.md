---
title: Ubiquitous Language
category: ddd-core
tags: [ddd, ubiquitous-language, communication]
related: [[Bounded Context]], [[Entity]]
---

# Ubiquitous Language

도메인 전문가와 개발자가 공유하는 공통 언어. 코드, 문서, 대화 모두에서 동일한 용어를 사용한다.

## 핵심 개념

- 코드의 클래스명, 메서드명, 변수명이 도메인 언어를 직접 반영한다
- [[Bounded Context]]마다 독립적인 Ubiquitous Language를 가질 수 있다
- 언어의 불일치를 발견하면 코드를 수정한다 (번역 레이어 금지)

## 나쁜 예 vs 좋은 예

```php
// ❌ 기술적 용어 중심
class OrderManager {
    public function processRecord(array $data): void { ... }
    public function updateStatus(int $id, int $statusCode): void { ... }
}

// ✅ 도메인 언어 중심
class Order {
    public function confirm(): void { ... }
    public function cancel(CancellationReason $reason): void { ... }
    public function ship(TrackingNumber $tracking): void { ... }
}
```

## Glossary 관리

팀 용어집을 문서화하고 코드와 동기화한다.

| 도메인 용어 | 코드 | 설명 |
|------------|------|------|
| 주문 | `Order` | 고객이 상품을 구매하는 행위의 단위 |
| 주문 확정 | `Order::confirm()` | 결제 완료 후 배송 준비 상태로 전환 |
| 상품 | `Product` | 판매 가능한 상품 |
| 재고 단위 | `StockKeepingUnit` (SKU) | 특정 옵션(색상, 사이즈)의 재고 |
| 배송 | `Shipment` | 주문의 물리적 배송 과정 |

## 주의사항

- **CRUD 용어 회피**: `create`, `update`, `delete` 대신 도메인 행위를 표현하는 동사를 사용.
- **동음이의어 주의**: 같은 단어가 다른 컨텍스트에서 다른 의미를 가질 수 있다. `Account`는 결제 컨텍스트에서 '계좌', 인증 컨텍스트에서 '계정'.

## 참고

- [[Bounded Context]] — 언어가 적용되는 경계
- [[Design Philosophy]] — 전략적 설계(언어)가 기술적 설계보다 중요하다는 원칙
