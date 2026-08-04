---
title: ViewModel
category: laravel
tags: [laravel, view-model, cqrs, presentation]
related: [[CQRS]], [[DTO]], [[Custom Query Builder]]
---

# ViewModel

특정 화면/응답이 필요로 하는 조회 데이터를 하나의 클래스로 캡슐화하는 패턴. Blade든 Inertia든 순수 API JSON이든 동일하게 적용된다.

## 핵심 개념

- ViewModel은 "이 페이지/응답이 필요로 하는 모든 쿼리"를 메서드로 표현하는 데이터 컨테이너다.
- 각 public 메서드가 응답의 한 필드에 대응한다. 기반 클래스가 `Arrayable`을 구현해, 메서드 이름을 스네이크 케이스 키로 자동 변환한 배열/JSON을 만들어준다.
- 도메인 언어와 밀접하다: 기획자가 "대시보드 페이지"라고 말하면 개발자는 즉시 `GetDashboardViewModel`을 떠올릴 수 있어야 한다.

## Laravel 구현

```php
class GetRevenueReportViewModel extends ViewModel
{
    public function totalRevenue(): int
    {
        return Order::sum('total');
    }

    public function totalNumberOfCustomers(): int
    {
        return Order::query()->groupBy('customer_id')->count('customer_id');
    }

    public function averageRevenuePerCustomer(): int
    {
        return $this->totalRevenue() / $this->totalNumberOfCustomers();
    }
}

class RevenueReportController extends Controller
{
    public function index()
    {
        return new GetRevenueReportViewModel();
    }
}
```

응답:

```json
{
  "total_revenue": 24500,
  "total_number_of_customers": 2311,
  "average_revenue_per_customer": 10.60
}
```

### DTO와 함께 사용

스칼라 값 대신 [[DTO]]를 반환하면 중첩 구조도 표현할 수 있다.

```php
class GetDashboardViewModel extends ViewModel
{
    public function newSubscribersCount(): NewSubscribersCountData
    {
        return new NewSubscribersCountData(
            today: Subscriber::whereSubscribedBetween(DateFilter::today())->count(),
            thisWeek: Subscriber::whereSubscribedBetween(DateFilter::thisWeek())->count(),
            thisMonth: Subscriber::whereSubscribedBetween(DateFilter::thisMonth())->count(),
            total: Subscriber::count(),
        );
    }
}
```

### Inertia와 함께 사용

```php
class BroadcastController
{
    public function index(): Response
    {
        return Inertia::render('Broadcast/List', [
            'model' => new GetBroadcastsViewModel(),
        ]);
    }
}
```

## DDD 관점에서의 활용: CQRS의 Query 절반

[[CQRS]]를 무겁게 구현(별도 Command Bus, 이벤트 소싱 등)하지 않아도, **Action = Command(쓰기), ViewModel = Query(읽기)** 라는 단순한 매핑만으로 책임 분리 효과를 충분히 얻을 수 있다. 이것이 이 책 저자가 실제로 CQRS를 몇 년째 "이름도 모르고" 써왔다고 말하는 이유다.

- ViewModel은 도메인 Entity나 [[Aggregate]]를 거치지 않고 직접 조회([[Custom Query Builder]] 등)해도 된다 — 조회는 불변식 검증이 필요 없다.
- ViewModel 메서드 안에서 쓰기 작업(저장, 상태 변경)을 하지 않는다 — 그 순간 Query 책임이 깨진다.

## 주의사항 / 안티패턴

- ViewModel 메서드에서 N+1 쿼리가 발생하지 않도록 주의한다. 리스트 페이지에서 각 항목마다 추가 쿼리를 실행하는 메서드(`performances()`처럼 컬렉션을 순회하며 개별 쿼리)를 만들 때는 배치 조회로 묶는 것을 고려한다.
- ViewModel에 비즈니스 로직(할인 계산, 상태 전이 판단)을 넣지 말 것 — 조회 결과를 조합/포맷팅하는 것까지만 담당한다.

## 참고

- [[CQRS]] — Action/ViewModel을 Command/Query로 매핑하는 실전 적용
- [[DTO]] — ViewModel이 반환하는 중첩 데이터 구조
- [[Custom Query Builder]] — ViewModel이 내부적으로 사용하는 조회 도구
- 소스: Domain-Driven Design with Laravel (Martin Joo), Working With Data 챕터
