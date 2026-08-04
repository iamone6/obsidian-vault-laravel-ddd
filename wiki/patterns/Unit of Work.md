---
title: Unit of Work
category: patterns
tags: [transaction, unit-of-work, consistency]
related: [[Aggregate]], [[Repository]], [[Repository Implementation]]
---

# Unit of Work

여러 변경 작업을 하나의 트랜잭션 경계로 묶어, 전부 성공하거나 전부 롤백되도록 보장하는 패턴. 라라벨은 별도의 Unit of Work 추상화를 제공하지 않지만 `DB::transaction()`으로 근사한다.

## 핵심 개념

- 여러 쿼리를 하나의 단위로 묶어 처리한다 — 모두 실행되거나, 하나라도 실패하면 전체를 롤백한다.
- 클로저 안에서 예외가 발생하면 자동으로 롤백하고, 클로저가 성공적으로 끝나면 자동으로 커밋한다.

## Laravel 구현

```php
DB::transaction(function () use ($userId, $numVotes) {
    DB::table('users')
        ->where('id', $userId)
        ->update(['votes' => $numVotes]);

    // 위 쿼리가 실패하면 아래 쿼리는 실행되지 않고 전체가 롤백된다.
    DB::table('votes')
        ->where('user_id', $userId)
        ->delete();
});
```

## DDD 관점에서의 활용

[[Aggregate]]는 하나의 트랜잭션 경계와 일치해야 한다는 원칙이 있다. [[Repository]] 구현체의 `save()`에서 Aggregate의 영속화와 [[Domain Event]] 발행을 하나의 트랜잭션으로 묶는 것이 실질적인 Unit of Work 역할을 한다.

```php
public function save(Order $order): void
{
    DB::transaction(function () use ($order) {
        $data = $this->mapper->toPersistence($order);
        OrderModel::updateOrCreate(['id' => $data['id']], $data);

        OrderItemModel::where('order_id', $data['id'])->delete();
        foreach ($data['items'] as $item) {
            OrderItemModel::create($item);
        }

        foreach ($order->pullDomainEvents() as $event) {
            $this->events->dispatch($event);
        }
    });
}
```

- 여러 Repository에 걸친 저장(예: `Order`와 `Inventory`를 동시에 갱신)이 필요하다면, Application Service 레벨에서 `DB::transaction()`으로 감싸 여러 Repository 호출을 하나의 트랜잭션에 묶는다. 단, 서로 다른 Aggregate를 하나의 트랜잭션에 묶는 것은 Aggregate 경계 설계상 예외적인 경우로 취급해야 한다 — 원칙적으로는 하나의 트랜잭션에 하나의 Aggregate만 변경하고, Aggregate 간 일관성은 [[Domain Event]]를 통한 최종적 일관성(eventual consistency)으로 처리하는 것이 DDD의 권장 방향이다.

## 주의사항 / 안티패턴

- 트랜잭션 커밋 전에 도메인 이벤트를 dispatch하면, 롤백 시 "일어나지 않은 일"에 대한 이벤트가 이미 리스너(특히 큐로 처리되는 리스너)에 전달될 위험이 있다. 커밋 후 dispatch하거나 큐 리스너의 `afterCommit` 옵션을 사용한다.
- 트랜잭션 클로저 안에서 외부 API 호출처럼 되돌릴 수 없는 부수 효과를 실행하지 말 것 — 롤백돼도 그 부수 효과는 취소되지 않는다.
- 트랜잭션 범위를 필요 이상으로 크게 잡으면 DB 락 경합이 늘어난다. 트랜잭션은 하나의 Aggregate 저장 단위로 좁게 유지한다.

## 참고

- [[Repository]] — 트랜잭션 안에서 저장과 이벤트 발행을 함께 처리하는 예시
- [[Aggregate]] — 트랜잭션 경계와 일치해야 하는 일관성 경계
- 소스: 처음부터 제대로 배우는 라라벨, 5장 데이터베이스와 엘로퀀트 (5.4.4 트랜잭션)
