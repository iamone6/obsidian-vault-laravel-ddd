---
title: Tailwind Dynamic Class Pitfall
category: frontend
tags: [frontend, tailwind, anti-pattern]
related: [[Tailwind CSS]], [[Tailwind Component Strategy]], [[Tailwind Theme Configuration]]
---

# Tailwind Dynamic Class Pitfall

실무에서 가장 자주, 그리고 배포 후에야 터지는 Tailwind 문제: 동적으로 조합한 클래스명은 빌드에 포함되지 않는다.

## 핵심 개념

Tailwind는 소스 파일을 **정규식 기반 텍스트 스캔**으로 훑어 등장하는 클래스명만 CSS로 생성한다. 코드를 실행하거나 파싱해서 이해하지 않는다. 문자열을 조립해서 만든 클래스는 스캐너 눈에 보이지 않는다.

```blade
{{-- 동작하지 않음 --}}
<div class="bg-{{ $color }}-500">
<div class="text-{{ $status === 'ok' ? 'green' : 'red' }}-600">
```

```jsx
// 동작하지 않음
<div className={`text-${size} bg-${color}-500`} />
```

개발 중에는 우연히 다른 곳에서 그 클래스를 써서 잘 되는 것처럼 보이다가, 그 코드를 지우는 순간 조용히 깨진다.

## Laravel 구현

### 해법 1 — 완전한 클래스명 매핑 (권장)

```php
$badgeClasses = match ($status) {
    'pending'   => 'bg-yellow-100 text-yellow-800 ring-yellow-600/20',
    'shipped'   => 'bg-blue-100 text-blue-800 ring-blue-600/20',
    'delivered' => 'bg-green-100 text-green-800 ring-green-600/20',
    'canceled'  => 'bg-gray-100 text-gray-800 ring-gray-600/20',
};
```

문자열 안에 **완전한 클래스명이 그대로 존재**하므로 스캐너가 인식한다. Enum에 넣으면 관리가 깔끔하다.

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Shipped = 'shipped';

    public function badgeClasses(): string
    {
        return match ($this) {
            self::Pending => 'bg-yellow-100 text-yellow-800',
            self::Shipped => 'bg-blue-100 text-blue-800',
        };
    }
}
```

### 해법 2 — CSS 변수로 우회 (값이 DB에서 오는 경우)

카테고리 색상처럼 값이 런타임에 결정되면 클래스로는 해결이 안 된다.

```blade
<div class="bg-(--cat-color) text-white"
     style="--cat-color: {{ $category->hex_color }}">
```

> 사용자 입력이 그대로 들어가는 경로라면 반드시 값 검증(hex 패턴 화이트리스트)을 할 것. CSS 값 주입도 공격 표면이 된다.

### 해법 3 — 안전목록 (`@source inline`)

PHP 클래스 밖(예: DB에 저장된 설정값)에서 클래스명이 오는 경우:

```css
@source inline("bg-{red,yellow,green,blue}-{100,500,800}");
```

중괄호 확장을 지원한다. CSS 크기가 늘어나므로 최소한으로 쓴다.

### 배포 파이프라인 주의

- `resources/views` 외 경로(패키지, `app/View/Components`, PHP 클래스 안의 클래스 문자열)에 클래스명이 있으면 `@source`로 명시적으로 추가해야 한다.
- **PHP 클래스 파일 안에 Tailwind 클래스 문자열을 쓴다면 그 경로도 스캔 대상**이어야 한다 (위 Enum 예시).

```css
@source "../../app/**/*.php";
```

- `.gitignore`된 파일은 기본적으로 스캔에서 제외된다. 빌드 산출물이나 캐시 경로에 의존하지 말 것.

## 주의사항 / 안티패턴

- 이 문제는 **PHP 클래스/헬퍼 안의 클래스 문자열이 `@source`에서 빠졌을 때도 동일하게** 발생한다. Enum이나 서비스 클래스에 배지 클래스를 넣었다면 해당 디렉토리를 스캔 대상에 포함했는지 항상 확인한다.

## 참고

- [[Tailwind CSS]] — 개요의 "잃는 것" 항목
- [[Tailwind Component Strategy]] — variant를 배열로 관리하는 동일한 발상
- [[Tailwind Theme Configuration]] — `@source`, `@source inline` 지시어
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`
