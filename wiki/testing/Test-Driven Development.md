---
title: Test-Driven Development (TDD)
category: testing
tags: [testing, tdd, red-green-refactor]
related: [[Feature Testing]], [[Testing Complex Features]], [[Domain Testing]]
---

# Test-Driven Development (TDD)

테스트가 구현 코드를 이끌어가는 개발 방식. 테스트를 먼저 작성하고, 그 테스트를 통과시키는 최소한의 코드를 작성한 뒤, 리팩터링한다.

## 핵심 개념

TDD를 한 문장으로 다시 쓰면: **클래스/함수를 어떻게 사용하고 싶은지 먼저 명세하고, 그 다음에야 실제 코드를 작성한다.** Red → Green → Refactor 세 단계를 반복한다.

### 1. Red — 실패하는 테스트 작성

아직 존재하지 않는 `Calculator::divide()`를 어떻게 쓰고 싶은지부터 테스트로 적는다.

```php
class CalculatorTest extends TestCase
{
    /** @test */
    public function it_should_divide_valid_numbers()
    {
        $calculator = new Calculator();

        $this->assertEquals(5.00, $calculator->divide(10, 2));
        $this->assertEquals(3.33, $calculator->divide(10, 3));
    }

    /** @test */
    public function it_should_return_zero_when_the_divider_is_zero()
    {
        $calculator = new Calculator();

        $this->assertEquals(0.00, $calculator->divide(10, 0));
    }
}
```

이 시점에는 `Calculator` 클래스 자체가 없으므로 테스트는 실패한다 — 그래서 "Red" 단계다. 이미 이 단계에서 스펙이 나온다: 나눗셈 결과는 항상 소수점 2자리 float이고, 0으로 나누면 예외 대신 0을 반환한다.

**테스트 이름은 영어 문장처럼 읽혀야 한다.** `testDivideWhenTheDividerIsZero`처럼 camelCase로 붙이지 말고, PHPUnit의 `@test` 애너테이션을 활용해 `it_should_return_zero_when_the_divider_is_zero`처럼 스네이크 케이스 + 자연어 문장으로 작성한다. 좋은 테스트 이름은 클래스의 문서 역할을 겸한다.

또한 유스케이스별로 테스트 함수를 분리한다 (정상 케이스와 0으로 나누기 케이스를 하나의 함수에 몰아넣지 않는다).

### 2. Green — 테스트를 통과시키는 최소한의 코드

```php
class Calculator
{
    public function divide(float $a, float $b): float
    {
        if ($b == 0.0) {
            return 0;
        }
        return round($a / $b, 2);
    }
}
```

이 단계의 목표는 "가장 아름다운 코드"가 아니라 **테스트를 통과시키는 가장 빠른 방법**이다. 새로운 엣지 케이스가 떠오르면 코드를 먼저 고치지 않고 테스트부터 추가한다.

### 3. Refactor — 안전하게 개선

모든 유스케이스에 대한 자동 테스트가 이미 있으므로, 이제 안심하고 리팩터링할 수 있다.

```php
class Calculator
{
    public function divide(float $a, float $b): float
    {
        try {
            return round($a / $b, 2);
        } catch (DivisionByZeroError) {
            return 0;
        }
    }
}
```

테스트가 계속 통과하는 한, 내부 구현을 자유롭게 바꿔도 회귀(regression)를 걱정할 필요가 없다.

## DDD 관점에서의 활용

TDD의 "먼저 사용법을 정의한다"는 태도는 [[Ubiquitous Language]]와 맞닿아 있다 — 테스트 코드에서 클래스를 어떻게 부르고 싶은지 먼저 적어보면, 자연스럽게 도메인 언어에 가까운 API가 나온다. 또한:

- API 개발에서는 TDD로 먼저 요청/응답 스펙을 테스트로 못박은 뒤 [[API Design]]의 Query Builder/Resource 구현을 채워나가는 순서가 자연스럽다.
- 복잡한 유스케이스는 [[Testing Complex Features]]의 블랙박스 전략과 결합해, "이 시나리오가 이렇게 동작해야 한다"를 먼저 테스트로 적고 [[Action Pattern|Action]]을 구현한다.

## 주의사항 / 안티패턴

- Green 단계에서 과도하게 일반화된 코드를 미리 작성하지 말 것 — 아직 없는 요구사항까지 예측해서 구현하면 YAGNI를 위반한다. 테스트가 요구하는 만큼만 구현한다.
- 테스트 이름이 여전히 기술적이면(`testDivide1`, `testCase2`) TDD의 "테스트 = 문서" 효과가 사라진다.

## 참고

- [[Feature Testing]], [[Testing Complex Features]] — 이 흐름을 적용하는 실전 테스트 전략
- [[Domain Testing]] — 분기가 많은 로직에서 TDD가 특히 유용한 지점
- 소스: Proper API Design With Laravel (Martin Joo)
