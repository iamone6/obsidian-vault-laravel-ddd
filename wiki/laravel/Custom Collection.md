---
title: Custom Collection
category: laravel
tags: [laravel, eloquent, collection, aggregation]
related: [[Custom Query Builder]], [[Value Object]], [[Action Pattern]], [[Eloquent Model Attributes]]
---

# Custom Collection

Eloquent의 `newCollection()`을 오버라이드해 모델 전용 Collection 클래스를 만드는 기법. 여러 레코드에 걸친 집계/계산 로직을 "컬렉션 자체의 행동"으로 표현한다.

## 핵심 개념

가중평균, 합계처럼 **개별 모델 하나에는 의미가 없고 여러 개를 모았을 때만 의미가 있는 계산**은 어디에 둬야 할까? 모델의 정적 메서드에 두면 어색하다 — 그 메서드는 `$this`를 전혀 쓰지 않고, 개념적으로도 "거래(Transaction) 하나"가 아니라 "거래들의 모음"에 속하는 로직이기 때문이다.

> 모델에 정적 메서드가 있다면, 대부분 더 나은 위치가 있다는 신호다.

## Laravel 구현

```php
namespace App\Collections\Transaction;

use App\Models\Transaction;
use Illuminate\Database\Eloquent\Collection;

class TransactionCollection extends Collection
{
    public function sumQuantity(): float
    {
        return $this->sum('quantity');
    }

    public function sumTotalPrice(): float
    {
        return $this->sum('total_price');
    }

    public function weightedPricePerShare(): float
    {
        $sumOfProducts = $this->sum(
            fn (Transaction $t) => $t->quantity * $t->price_per_share
        );

        if ($this->sumQuantity() == 0.00) {
            return 0;
        }

        return $sumOfProducts / $this->sumQuantity();
    }
}

class Transaction extends Model
{
    public function newCollection(array $models = []): TransactionCollection
    {
        return new TransactionCollection($models);
    }
}
```

이제 `Transaction::all()`이 반환하는 컬렉션은 자동으로 `TransactionCollection`의 인스턴스이며, `->weightedPricePerShare()`를 바로 호출할 수 있다.

```php
$transactions = Transaction::all();
$holding->average_cost = $transactions->weightedPricePerShare();
```

**한 거래(Transaction)에는 "주당 가중평균가"라는 개념이 없다. 오직 거래들의 모음(Collection)에만 존재한다.** 이 문장 자체가 커스텀 컬렉션이 왜 도메인 언어에 가까운 코드를 만드는지 보여준다.

## `newCollection()` 대신 attribute로 연결하기 (Laravel 11.28+)

```php
use Illuminate\Database\Eloquent\Attributes\CollectedBy;

#[CollectedBy(TransactionCollection::class)]
class Transaction extends Model {}
```

`newCollection()` 오버라이드와 동일하게 동작한다. 부모 클래스 체인은 탐색하지만 trait은 탐색하지 않는다 — 자세한 내용은 [[Eloquent Model Attributes]] 참고.

## Custom Query Builder와의 관계

| | [[Custom Query Builder]] | Custom Collection |
|---|---|---|
| 적용 시점 | DB 쿼리 단계 (SQL로 변환됨) | 이미 메모리에 로드된 컬렉션 |
| 용도 | 조회 조건, 스코프 | 조회된 결과에 대한 집계/계산 |
| 예시 | `Mail::whereOpened()` | `$transactions->weightedPricePerShare()` |

두 기법은 상호 배타적이지 않다 — Builder로 필요한 레코드만 걸러온 뒤, Collection 메서드로 그 결과를 집계하는 조합이 자연스럽다.

## DDD 관점에서의 활용

`GetPerformanceAction`처럼 여러 모델에 걸쳐 재사용되는 계산은 Action으로, 특정 모델 컬렉션에 고유한 집계는 Custom Collection으로 분리하면 책임이 명확해진다. 계산 결과를 [[Value Object]](`Percent` 등)로 감싸면 반환 타입도 도메인 친화적으로 유지된다.

## 주의사항 / 안티패턴

- Custom Collection에 DB 저장/삭제 같은 부수 효과를 넣지 말 것 — 순수 계산/조회 결과 가공만 담당한다.
- 한 번만 쓰이는 계산까지 전부 Collection 클래스로 옮기지 않는다. 재사용되거나 여러 곳에서 같은 계산이 반복될 때 도입한다.

## 참고

- [[Custom Query Builder]] — DB 쿼리 단계의 대응 기법
- [[Value Object]] — 계산 결과를 감싸는 타입
- [[Eloquent Model Attributes]] — `#[CollectedBy]`를 포함한 모델 설정 attribute 전체 목록
- 소스: Case Study - Portfolio And Dividend Tracker (Martin Joo)
