---
title: Value Object
category: ddd-core
tags: [ddd, value-object, immutable]
related: [[Entity]], [[Aggregate]]
---

# Value Object

속성의 조합으로 동등성을 판단하는 불변 객체. 식별자가 없다.

## 핵심 개념

- **불변(Immutable)**: 생성 후 상태 변경 불가. 변경이 필요하면 새 객체를 생성.
- **동등성**: ID가 아닌 모든 속성 값이 같으면 동일.
- **자기 유효성 검증**: 생성 시점에 유효하지 않은 값을 거부.
- **도메인 로직 포함**: 단순 데이터 보유 이상의 행동을 가질 수 있다.

좋은 Value Object 후보: `Money`, `Email`, `Address`, `PhoneNumber`, `DateRange`, `Percentage`, `Temperature`

## Laravel 구현

```php
// Domain/Model/Money.php
final class Money
{
    private function __construct(
        private readonly int $amount,     // 원 단위 정수 (소수점 오차 방지)
        private readonly string $currency,
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException('금액은 음수일 수 없습니다.');
        }
    }

    public static function of(int $amount, string $currency): self
    {
        return new self($amount, $currency);
    }

    public static function zero(string $currency): self
    {
        return new self(0, $currency);
    }

    public function add(self $other): self
    {
        $this->assertSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function multiply(int $quantity): self
    {
        return new self($this->amount * $quantity, $this->currency);
    }

    public function equals(self $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }

    private function assertSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('통화가 다른 금액은 연산할 수 없습니다.');
        }
    }

    public function amount(): int { return $this->amount; }
    public function currency(): string { return $this->currency; }
}
```

### Email Value Object

```php
final class Email
{
    private function __construct(private readonly string $value) {}

    public static function from(string $value): self
    {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("유효하지 않은 이메일: {$value}");
        }
        return new self(strtolower($value));
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }

    public function toString(): string { return $this->value; }
    public function __toString(): string { return $this->value; }
}
```

## Eloquent와 함께 사용

Eloquent 모델에서 Value Object를 자동으로 캐스팅하려면 커스텀 Cast를 사용한다.

```php
// Infrastructure/Casts/MoneyCast.php
class MoneyCast implements CastsAttributes
{
    public function get($model, string $key, $value, array $attributes): Money
    {
        return Money::of(
            (int) $attributes['amount'],
            $attributes['currency']
        );
    }

    public function set($model, string $key, $value, array $attributes): array
    {
        return [
            'amount'   => $value->amount(),
            'currency' => $value->currency(),
        ];
    }
}

// Eloquent 모델
class OrderEloquent extends Model
{
    protected $casts = [
        'total' => MoneyCast::class,
    ];
}
```

## 팩토리 메서드로 예외 상황 캡슐화

값 계산에 참여하는 예외 케이스(0으로 나누기 등)를 VO 생성 로직 안에 캡슐화하면, 호출부마다 반복되는 방어 코드를 없앨 수 있다.

```php
final class Percent
{
    public readonly float $value;
    public readonly string $formatted;

    public function __construct(float $value)
    {
        $this->value = $value;
        $this->formatted = number_format($value * 100, 1) . '%';
    }

    public static function from(float $numerator, float $denominator): self
    {
        if ($denominator <= 0.0) {
            return new self(0); // 분모가 0이어도 호출부는 신경 쓸 필요 없음
        }
        return new self($numerator / $denominator);
    }
}

// 호출부는 분모가 0인지 매번 확인할 필요가 없다
$openRate = Percent::from($openedCount, $totalSent);
```

이렇게 원시 값(raw float)과 포맷된 문자열(formatted string)을 함께 들고 있으면, "이 메서드가 포맷된 문자열을 반환하는지 원시 숫자를 반환하는지"를 고민할 필요 없이 항상 `Percent` 하나로 통일해서 다룰 수 있다.

## 상태를 표현하는 Value Object

문자열 상태 컬럼을 상태별 클래스로 승격시키고 싶다면 [[State Pattern]]을 참고한다. Enum 기반 VO와 State 클래스는 자연스럽게 결합된다 (PHP 8.1 enum을 상태 클래스 팩토리로 활용).

## 주의사항 / 안티패턴

- **가변 Value Object**: setter를 추가하는 순간 Value Object가 아니다.
- **DB 컬럼 하나에 복합 VO**: `money` 컬럼 하나에 `100 KRW` 형태로 저장하면 쿼리가 어렵다. 컬럼을 분리하고 Cast를 사용.
- **null 허용**: null 대신 Null Object 패턴이나 Optional을 사용.
- **VO에 ID를 추가**: ID가 필요해지는 순간 그것은 더는 Value Object가 아니라 [[DTO]]나 Entity다.

## 참고

- [[Entity]] — Value Object를 속성으로 갖는 도메인 객체
- [[Mapper Pattern]] — VO와 Eloquent 모델 간 변환
- [[DTO]] — ID 유무로 VO와 구분되는 개념
- [[State Pattern]] — Enum 기반 상태 값을 클래스로 캡슐화
- 소스: Domain-Driven Design with Laravel (Martin Joo), Value Objects 챕터
