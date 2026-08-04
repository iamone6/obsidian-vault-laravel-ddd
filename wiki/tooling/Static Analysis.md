---
title: Static Analysis (PHPInsights, Larastan, Deptrac)
category: tooling
tags: [static-analysis, phpstan, larastan, deptrac, phpinsights, architecture-enforcement]
related: [[Layered Architecture]], [[CI-CD Pipeline]], [[Directory Structure]]
---

# Static Analysis (PHPInsights, Larastan, Deptrac)

버그, 과도한 복잡도, 타입힌트 누락, 아키텍처 규칙 위반을 자동으로 잡아내는 세 가지 정적 분석 도구.

## phpinsights — 코드 품질 스코어

`./vendor/bin/phpinsights analyse`로 실행하며, 4개 카테고리를 1~100점으로 채점한다.

| 카테고리 | 검사 내용 |
|----------|-----------|
| Code | 사용되지 않는 private 속성/변수, 불필요한 final, switch 문 구조 등 |
| Complexity | 순환 복잡도(cyclomatic complexity) — 함수/클래스 안의 실행 경로 수. 평균 5 초과 시 경고 |
| Architecture | 클래스당 메서드/속성 수 제한, 함수 길이 등 |
| Style | 파일 끝 개행, 캐스트 뒤 공백 등 포맷 규칙 |

CI에서는 비대화형 모드로 최소 점수를 강제한다.

```bash
./vendor/bin/phpinsights --no-interaction --min-quality=80 --min-complexity=90 --min-architecture=70 --min-style=75
```

## Larastan — Laravel 특화 PHPStan

PHPStan 위에 구축된 Laravel 전용 정적 타입 분석기. `level`(1~9)로 엄격도를 조절한다.

```yaml
# phpstan.neon
includes:
  - ./vendor/nunomaduro/larastan/extension.neon
parameters:
  paths:
    - app
    - src
  level: 5
  ignoreErrors:
    - '#PHPDoc tag @var#'
  excludePaths:
    - 'app/Http/Kernel.php'
    - 'app/Console/Kernel.php'
    - 'app/Exceptions/Handler.php'
```

레벨 선택 가이드:
- **레거시 프로젝트**: 1부터 시작. 처음 도입이라면 다른 선택지가 없다.
- **신규 프로젝트**: 팀/프로젝트에 따라 다르지만 보통 4~6.
- **레벨 9는 지양**: 레벨 5만 넘어가도 급격히 까다로워진다. 레벨 7이 저자의 최고 기록이며 "최종 보스전" 수준이었다고 한다.

## Deptrac — 아키텍처 경계 강제

레이어 간 참조 규칙을 정의하고 위반 시 에러를 발생시키는 도구. `Controller`가 직접 비즈니스 로직을 구현하거나, `Model`이 알림 발송/API 호출/Job 디스패치를 하는 것을 원천 차단하고 싶을 때 사용한다.

```yaml
# deptrac.yaml
parameters:
  paths: [./app, ./src]
  layers:
    - name: Action
      collectors:
        - type: className
          regex: .*Actions\\.*
    # Model, Controller, DTO, ValueObject, Builder 등도 동일하게 정의

ruleset:
  Controller: [Action, ViewModel, Model, DTO, ValueObject]
  Action: [Event, Model, DTO, Builder, ValueObject]
  Model: [Builder, Model, DTO, ValueObject]   # Model은 Job이나 외부 API를 참조할 수 없음
  DTO: [Model]
  ValueObject: [ValueObject]
```

즉 `Model`이 `Job` 클래스를 `use`하는 순간 deptrac이 빌드를 실패시킨다. [[Layered Architecture]]에서 말로만 정의하던 "이 레이어는 저 레이어를 참조하면 안 된다"는 규칙을 실제로 강제 가능한 형태로 코드화한 것이다.

```bash
./vendor/bin/deptrac analyse
```

## 도입 가치

세 도구를 새 프로젝트에 도입하는 데는 15~30분이면 충분하지만, 이후 3~5년간 코드베이스 품질을 지속적으로 지켜준다. 특히 deptrac은 레거시 프로젝트의 아키텍처를 점진적으로 정리하는 데도 유용하다.

## 참고

- [[Layered Architecture]] — deptrac이 강제하는 레이어 개념
- [[CI-CD Pipeline]] — 이 도구들을 파이프라인에 통합하는 방법
- [[Directory Structure]] — deptrac 레이어 정의와 대응되는 폴더 구조
- 소스: Static Analysis And CI/CD Pipelines (Martin Joo)
