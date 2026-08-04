---
title: Design Philosophy (DDD 핵심 원칙)
category: ddd-core
tags: [ddd, philosophy, ubiquitous-language, pragmatism]
related: [[Ubiquitous Language]], [[Bounded Context]], [[Layered Architecture]], [[Repository]], [[Action Pattern]], [[DTO]], [[Value Object]], [[Aggregate]]
---

# Design Philosophy (DDD 핵심 원칙)

이 위키에 쌓인 개별 패턴(VO, DTO, Repository, Action, CQRS...)을 관통하는 두 가지 원칙. 패턴 자체보다 이 두 원칙이 먼저다.

## 원칙 1 — 전략적 설계가 기술적 설계보다 중요하다

DDD가 가르치는 두 축은 전략적 설계(Strategic Design)와 기술적 설계(Technical Design)다. 이 위키의 대부분은 후자(Value Object, Repository, Action 같은 클래스 구조)를 다루지만, 실제로 더 중요한 것은 전자다: **코드가 비즈니스 언어를 얼마나 정직하게 반영하는가.**

Martin Joo(『Domain-Driven Design with Laravel』)는 자신이 6년 전에 작성한 코드를 예로 든다.

```php
class Search_View_Container_Factory_Project
{
    private static $_relationContainer;

    public static function createContainer(array $data) { ... }
    private static function createSimple(array $data) { ... }
}
```

저자 본인도 이 클래스가 정확히 무엇을 하는지 기억하지 못한다. 기술적으로는 흠잡을 데 없어 보여도, 비즈니스 언어를 전혀 반영하지 않기 때문이다. "프리랜서-프로젝트 매칭" 서비스의 검색 기능이라는 사실은 이름만 봐서는 절대 알 수 없다.

> 기술 용어와 과도하게 사용된 패턴은 고수준 비즈니스 애플리케이션에서 오히려 독이 된다.

이 원칙은 [[Ubiquitous Language]](클래스/메서드명이 도메인 언어를 직접 반영), [[Bounded Context]](컨텍스트마다 언어가 달라질 수 있음), [[Directory Structure]](폴더를 `Http`/`Controllers` 같은 기술 계층이 아니라 `Subscriber`, `Order` 같은 도메인 개념으로 구성)로 구체화된다. 이 세 페이지가 다루는 것은 결국 하나의 질문이다: **"제품 매니저가 '구독자 페이지에 회원 수 좀 보여주세요'라고 말했을 때, 개발자가 어떤 클래스를 고쳐야 할지 즉시 떠올릴 수 있는가?"**

## 원칙 2 — 패턴은 가이드라인이지 강제 규칙이 아니다

이 위키의 여러 페이지가 독립적으로 같은 결론에 도달한다: **이론적으로 "올바른" 패턴 적용보다, 팀/프로젝트 전체에서 하나의 방식으로 일관되게 가는 것이 더 중요하다.**

- [[Repository]] — Custom Query Builder와의 트레이드오프를 상세히 다루지만, 결론은 "프로젝트 상황에 맞는 쪽을 고르고 일관되게 적용하라"는 것이다. Repository가 이론적으로 더 "정통 DDD"에 가깝다는 이유만으로 Laravel의 자연스러운 Eloquent 체이닝을 포기할 필요는 없다.
- [[Action Pattern]] — Action을 작성하는 세 가지 방식(정적/인스턴스/invokable) 중 "정답은 없다"고 명시한다. 프로젝트 전체에서 하나로 통일하는 것만 중요하다.
- [[DTO]] — DTO와 Value Object를 ID 유무로 구분하는 것이 원칙이지만, "엄격한 규칙이라기보다 가이드라인"이며 상황에 따라 혼용해도 무방하다고 명시한다.
- [[Eloquent and DDD]] — Eloquent를 도메인 모델로 겸용하는 실용적 접근(전략 1)과 완전히 분리하는 접근(전략 2) 중 프로젝트 규모에 맞는 것을 선택하라고 안내한다.

> DDD의 기술적 측면은 사실 쉽다. 개발자들이 그것을 필요 이상으로 복잡하게 만든다.

패턴을 "100% 정확하게 지키거나, 아니면 DDD를 안 하는 것"이라는 이분법은 이 위키의 어떤 페이지도 지지하지 않는다. VO와 DTO의 경계가 헷갈리면 하나를 골라 일관되게 쓰면 된다 — 그것으로 충분하다.

## 두 원칙의 관계

원칙 1(언어)은 **무엇을 표현할지**를 결정하고, 원칙 2(실용주의)는 **어떻게 표현할지 고르는 태도**를 결정한다. [[Layered Architecture]]의 의존성 규칙(Domain은 프레임워크에 의존하지 않음)은 원칙 1을 지키기 위한 기술적 장치이며, 그 안에서 어떤 구체적 클래스 구조(Repository vs Custom Query Builder, Action vs Application Service)를 쓸지는 원칙 2에 따라 프로젝트마다 다르게 정해도 된다.

## 사례 — 얕은 DDD (참조 데이터/계산 중심 도메인)

원칙 2의 실용주의를 극단까지 밀면 "얕은 DDD"에 도달한다: 정통 DDD 기술 패턴(Repository, Value Object, Entity/Aggregate)을 아예 생략하고, Bounded Context식 폴더 분리와 Action/DTO/ViewModel/Form Request 같은 [[Directory Structure]]·[[Action Pattern]] 수준의 패턴만 채택하는 것이다.

건설 인프라 LCA(전과정평가) 탄소배출량 산정 API가 실제 사례다. 이 도메인은:

- **상태 전이형 애그리게잇이 거의 없다** — "주문 확정" 같은 트랜잭션 일관성 경계보다, 참조 데이터(자재/장비/LCI 계수) 매칭과 산식 계산이 대부분을 차지한다.
- **채택한 것**: `app/Domain/{Materials,Mechanics,Report,...}/{Actions,Services,DTOs,Requests,ViewModels}` 폴더 분리, Action Pattern, DTO, ViewModel, Form Request.
- **생략한 것**: Repository(Action이 Eloquent 모델을 직접 쿼리), Value Object(원시 타입 string/float 그대로 사용), Entity/Aggregate(Eloquent 모델은 로직 없는 빈 껍데기).

이 선택 자체는 원칙 2에 부합한다 — Repository나 VO가 이론적으로 "더 정통"이라는 이유만으로, 이득이 크지 않은 도메인에 무거운 패턴을 강제할 필요는 없다.

**단, 이 선택이 원칙 2의 "일관성" 요구까지 면제해주진 않는다.** 패턴을 생략하는 것과, 생략한 상태에서 스타일이 뒤섞이는 것은 다른 문제다. 실전에서 코딩 중 흔히 깨지는 지점:

- 같은 Action 클래스 안에서 정적 메서드와 인스턴스 메서드가 섞임 — [[Action Pattern]]이 요구하는 "세 방식 중 하나로 통일"이 클래스 단위가 아니라 파일 단위에서도 깨질 수 있다.
- 계산·판별·조회처럼 서로 다른 유스케이스가 한 Action 클래스에 몰림 — "클래스 하나 = 유스케이스 하나" 원칙 위반.
- 범용 유틸리티 함수(재귀 배열 탐색 등)가 도메인 Action 안에 섞여 [[Ubiquitous Language]]를 흐림.
- Controller가 응답 데이터를 직접 가공해 [[Layered Architecture]]의 "UI는 얇게" 규칙을 깸.

이런 이슈는 아키텍처 깊이를 얼마나 얕게 가져가든 상관없이 발생하는, **규칙을 몰라서가 아니라 코딩 중 잊어버려서** 생기는 종류의 붕괴다. [[Static Analysis]]의 deptrac 같은 도구로 아키텍처 규칙(레이어 간 의존 방향, 네임스페이스 규칙)을 기계적으로 강제하는 것이 패턴 깊이와 무관하게 일관성을 지키는 실질적 방법이다.

## 주의사항 / 안티패턴

- **패턴을 위한 패턴**: "DDD니까" VO/DTO/Repository/Action을 전부 도입하는 것 자체가 목적이 되면, [[Eloquent and DDD]]가 경고하는 임피던스 불일치와 불필요한 보일러플레이트만 늘어난다.
- **기술 패턴은 완벽한데 언어는 기술적인 경우**: `OrderService`, `OrderRepository`, `OrderAction`을 다 갖췄어도 내부 메서드명이 `processRecord()`, `handleData()`처럼 기술 용어면 원칙 1을 놓친 것이다. 패턴 완성도보다 이름이 비즈니스를 설명하는지를 먼저 점검한다.
- **팀 내 불일치**: 한 모듈은 정적 Action, 다른 모듈은 인스턴스 Action처럼 방식이 뒤섞이면 원칙 2가 깨진다. 새 프로젝트에서 방식을 정할 때는 [[Static Analysis]]의 deptrac 같은 도구로 아키텍처 규칙 자체를 강제하는 것도 고려한다.

## 참고

- [[Ubiquitous Language]] — 원칙 1의 구체적 실천 방법
- [[Bounded Context]], [[Directory Structure]] — 언어와 경계를 코드 구조로 표현하는 방법
- [[Layered Architecture]] — 원칙 1을 지키는 기술적 장치(의존성 규칙)
- [[Repository]], [[Action Pattern]], [[DTO]] — 원칙 2(실용주의)가 개별 패턴에 적용된 사례
- [[Static Analysis]] — 아키텍처 규칙을 도구로 강제하는 방법 (얕은 DDD에서도 일관성을 지키는 실질적 수단)
- 소스: Domain-Driven Design with Laravel (Martin Joo), Basic Concepts / Advantages And Disadvantages 챕터
