---
title: CI/CD Pipeline (Github Actions, Gitlab CI)
category: tooling
tags: [ci-cd, github-actions, gitlab-ci, pipeline]
related: [[Static Analysis]], [[Feature Testing]]
---

# CI/CD Pipeline (Github Actions, Gitlab CI)

정적 분석과 테스트를 자동화하는 파이프라인 구성. Github Actions(간단)와 Gitlab CI(Docker 기반, 더 복잡)의 전형적인 구조.

## Github Actions

`.github/workflows/laravel.yml`에 워크플로를 정의한다.

```yaml
name: Quality Check
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
jobs:
  quality-check:
    runs-on: ubuntu-latest
    steps:
      - uses: shivammathur/setup-php@...
        with:
          php-version: '8.1'
      - uses: actions/checkout@v2
      - name: Copy .env
        run: php -r "file_exists('.env') || copy('.env.example', '.env');"
      - name: Install Dependencies
        run: composer install -q --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist
      - name: Generate key
        run: php artisan key:generate
      - name: Directory Permissions
        run: chmod -R 777 storage bootstrap/cache
      - name: Create Database
        run: |
          mkdir -p database
          touch database/database.sqlite
      - name: Run phpinsights
        run: ./vendor/bin/phpinsights --no-interaction --min-quality=80 --min-complexity=90 --min-architecture=70 --min-style=75
      - name: Run phpstan
        run: ./vendor/bin/phpstan analyse --memory-limit=1G
      - name: Run deptrac
        run: ./vendor/bin/deptrac analyse
      - name: Test
        env:
          DB_CONNECTION: sqlite
          DB_DATABASE: database/database.sqlite
        run: php artisan test
```

핵심 개념:
- `on`: 언제 실행할지 정의 (main 브랜치 push, main으로의 PR)
- `jobs` → `steps`: 각 job은 매번 새 Ubuntu 컨테이너("runner")에서 실행되므로, 코드 체크아웃부터 의존성 설치까지 매번 처음부터 진행한다.
- CI에서는 빠른 실행을 위해 SQLite를 테스트 DB로 흔히 사용한다 (MySQL/Redis가 필요하면 Docker Compose로 별도 구성).

### 병렬화

기본 구성은 스텝이 순차 실행된다. 정적 분석 도구들은 서로 의존하지 않으므로 병렬화할 수 있다.

- **워크플로 파일을 분리**: phpinsights, phpstan, deptrac, test를 각각 별도 `.yml` 워크플로로 만들면 병렬 실행되고, 하나라도 실패하면 전체가 실패로 표시된다.
- **한 워크플로 안에서 job을 분리**: 여러 job으로 나누면 Github Actions UI에서 병렬 실행 상태를 한눈에 볼 수 있다. 이후 단계(배포 등)가 모든 job을 기다려야 한다면 `needs`로 순서를 강제한다.

```yaml
jobs:
  test: ...
  phpinsights: ...
  phpstan: ...
  deptrac: ...
  deploy:
    needs: [test, phpinsights, phpstan, deptrac]
```

## Gitlab CI

Docker/Docker Compose 기반이라 더 복잡하지만 dev/prod 이미지를 분리하는 실전 패턴을 제공한다.

```yaml
stages: [build, analyze, test, publish, deploy]

variables:
  JOB_IMAGE_NAME_DEV: $CI_REGISTRY_IMAGE:dev
  JOB_IMAGE_NAME_PROD: $CI_REGISTRY_IMAGE/prod:$CI_COMMIT_SHORT_SHA

build-dev:
  stage: build
  script:
    - docker build --target=dev -t $JOB_IMAGE_NAME_DEV -f ./docker/Dockerfile .

phpstan:
  stage: analyze
  script:
    - docker run --rm -t $JOB_IMAGE_NAME_DEV ./vendor/bin/phpstan analyze --memory-limit=1G

test:
  stage: test
  before_script:
    - docker-compose -f docker-compose.ci.yml up -d --force-recreate
  script:
    - docker-compose -f docker-compose.ci.yml exec -T demo-project php artisan test
  after_script:
    - docker-compose -f docker-compose.ci.yml down
```

dev 이미지와 prod 이미지의 전형적 차이:

| | dev 이미지 | prod 이미지 |
|---|---|---|
| xdebug | 포함 (커버리지 리포트용) | 미포함 |
| composer 의존성 | require-dev 포함 | production만 |
| php.ini | `error_reporting = E_ALL`, xdebug 활성화 | `display_errors = off`, 메모리 제한 설정 |

테스트 스테이지는 MySQL, Redis 등 전체 스택이 필요하므로 `docker-compose`로 서비스를 띄운 뒤 컨테이너 안에서 `php artisan test`를 실행한다.

## 참고

- [[Static Analysis]] — 파이프라인에 통합하는 세 가지 정적 분석 도구
- [[Feature Testing]] — 파이프라인의 테스트 단계에서 실행되는 테스트 종류
- 소스: Static Analysis And CI/CD Pipelines (Martin Joo)
