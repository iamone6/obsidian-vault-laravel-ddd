---
title: Eloquent and DDD
category: laravel
tags: [laravel, eloquent, ddd, impedance-mismatch]
related: [[Entity]], [[Repository]], [[Mapper Pattern]], [[Repository Implementation]], [[Eloquent Model Attributes]]
---

# Eloquent and DDD

Eloquent는 Active Record 패턴을 사용하는 반면 DDD는 순수 도메인 모델을 요구한다. 이 두 가지의 임피던스 불일치(Impedance Mismatch)를 해결하는 방법.

## 문제

Eloquent 모델은:
- DB 스키마에 직결되어 있어 도메인 모델로 쓰기엔 인프라 의존성이 생긴다
- Magic Properties(`$model->name`)로 타입 안전성이 없다
- 지속성 로직(save, delete)이 도메인 객체에 섞인다

## 세 가지 접근 전략

### 전략 1: Eloquent 모델 = 도메인 모델 (Pragmatic DDD)

작은 프로젝트나 빠른 개발에 적합. Eloquent 모델에 도메인 로직을 직접 추가.

```php
class Order extends Model
{
    protected $casts = [
        'status'       => OrderStatus::class,
        'total_amount' => MoneyCast::class,
    ];

    public function confirm(): void
    {
        if ($this->status !== OrderStatus::Pending) {
            throw new \DomainException('...');
        }
        $this->status = OrderStatus::Confirmed;
        $this->save();
        event(new OrderConfirmed($this->id));
    }
}
```

**장점**: 빠른 개발, Eloquent 생태계 풀 활용  
**단점**: 도메인이 Eloquent에 결합, 순수 단위 테스트 불가

---

### 전략 2: Eloquent 모델 ↔ 도메인 모델 분리 (Mapper Pattern)

권장 방법. Eloquent 모델은 데이터 접근만 담당하고, 별도 도메인 Entity로 변환.

```
Domain:         Order (순수 PHP)
                  ↑ ↓ (Mapper)
Infrastructure: OrderModel (Eloquent)
```

자세한 구현은 [[Mapper Pattern]] 참고.

---

### 전략 3: Eloquent 없이 Raw Query / 다른 ORM

완전한 분리. 도메인이 완전히 순수. 그러나 Laravel 생태계 이점을 포기.

## Eloquent 모델을 순수하게 유지하는 팁

전략 1을 선택하더라도 Eloquent 모델을 가능한 얇게 유지한다:

```php
class Order extends Model
{
    // DTO로 export
    public function toOrderData(): OrderData
    {
        return new OrderData(
            id: $this->id,
            status: $this->status,
            totalAmount: $this->total_amount,
        );
    }

    // 팩토리 메서드로 생성 통제
    public static function createForCustomer(Customer $customer): self
    {
        $order = new self();
        $order->customer_id = $customer->id;
        $order->status = OrderStatus::Pending;
        return $order;
    }
}
```

## 어떤 전략을 선택할까?

| 상황 | 추천 전략 |
|------|-----------|
| 소규모 CRUD 앱 | 전략 1 |
| 복잡한 비즈니스 로직, 중간 규모 | 전략 2 |
| 엔터프라이즈, 완전한 테스트 격리 필요 | 전략 2 또는 3 |

## 주의사항

- **Eloquent 모델을 도메인 이벤트에 담지 말 것**: Eloquent 모델은 직렬화 시 문제가 많다.
- **Repository에서 Eloquent 모델 반환 금지**: 항상 도메인 타입으로 변환 후 반환.

## 선언부 설정과 "얇은 모델" 원칙

전략 1(Eloquent = 도메인 모델)을 선택했더라도, `$fillable`/`$hidden`/`$guarded` 같은 설정 프로퍼티가 늘어나는 것 자체가 모델이 "두꺼워지는" 신호는 아니다 — 순수 설정과 도메인 로직(행동 메서드)을 시각적으로 구분해두면 얇게 유지하기 쉬워진다. Laravel 13은 이런 설정을 [[Eloquent Model Attributes]](`#[Fillable]`, `#[Hidden]`, `#[Table]` 등)로 클래스 선언부에 모을 수 있게 했다 — 클래스 본문에는 행동 메서드만 남기고, 설정은 attribute로 위에 몰아두는 구성이 "얇은 모델"을 강제하는 컨벤션으로 쓰일 수 있다. 단, `#[Unguarded]`처럼 대량 할당 방어를 끄는 attribute는 외부 입력이 직접 닿는 도메인 모델에는 쓰지 않는다.

## Eloquent 연관관계와 도메인 경계

Eloquent의 연관관계 메서드(`hasOne`, `belongsTo`, `hasMany`, `belongsToMany`, `hasManyThrough`, 다형성 `morphTo` 등)는 강력하지만, 그 자체가 인프라 개념이다. 관계 메서드 호출과 Lazy/Eager Loading 판단은 [[Repository Implementation]]에 정리된 대로 Repository 구현체(인프라 레이어) 안에 가둬야 하며, 도메인 Entity는 이미 조립된 값/컬렉션만 받아야 한다.

```php
// 인프라 레이어에서만 등장해야 하는 코드
$order = OrderModel::with('items', 'customer')->find($id);

// 도메인 Entity는 이렇게 "이미 준비된" 데이터를 받는다 (Mapper Pattern)
$domainOrder = new Order(
    id: OrderId::fromString($order->id),
    items: $order->items->map(fn($i) => OrderItem::fromModel($i))->all(),
);
```

## 참고

- [[Mapper Pattern]] — Eloquent ↔ Domain 객체 변환
- [[Repository Implementation]] — Eloquent 연관관계·쿼리 빌더를 활용한 리포지토리 구현
- [[Value Object]] — Eloquent Cast로 VO 사용
- [[Unit of Work]] — 저장 시 트랜잭션 경계 관리
- [[Design Philosophy]] — 전략 1/2/3 중 선택은 실용주의 원칙에 따른다
- [[Eloquent Model Attributes]] — 설정을 선언부 attribute로 분리해 모델을 얇게 유지하는 방법
