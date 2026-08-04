---
title: Schema — Laravel Wiki
type: meta
---

# Schema

이 위키는 Karpathy의 LLM Wiki 패턴을 기반으로, Laravel + DDD(Domain-Driven Design)로 프로젝트를 구축하는 방법에 대한 지식을 점진적으로 축적한다.

## 목적

- 원본 소스(문서, 코드, 아티클)를 읽고 지식을 위키 페이지로 정제·저장한다
- 지식은 한 번 정제되면 재사용 가능한 형태로 영구 보존된다
- 새 소스를 추가할 때마다 관련 페이지를 갱신하거나 신규 페이지를 생성한다

## 디렉토리 구조

```
laravel/
├── schema.md          ← 이 파일 (위키 구조 정의)
├── index.md           ← 카테고리별 페이지 카탈로그
├── log.md             ← 추가/변경 이력 (append-only)
├── sources/           ← 원본 소스 (불변)
│   └── ...
└── wiki/
    ├── ddd-core/      ← DDD 핵심 개념
    ├── laravel/       ← Laravel 통합 및 구현
    ├── architecture/  ← 아키텍처 패턴
    ├── patterns/      ← 구현 패턴
    ├── testing/       ← 테스트 전략
    ├── tooling/       ← 정적 분석, CI/CD 등 개발 도구
    └── frontend/      ← Laravel 프로젝트의 프론트엔드 스택 (Tailwind, NativePHP 레이아웃 등)
```

## 페이지 형식

각 위키 페이지는 다음 구조를 따른다:

```markdown
---
title: 페이지 제목
category: ddd-core | laravel | architecture | patterns | testing | tooling | frontend
tags: [tag1, tag2]
related: [[관련페이지1]], [[관련페이지2]]
---

# 제목

한 줄 요약.

## 핵심 개념

## Laravel 구현

## 예제 코드

## 주의사항 / 안티패턴

## 참고
```

## 운영 워크플로

### Ingest (소스 추가)
새 소스(책, 아티클, 코드)를 `sources/`에 추가한 뒤, 관련 위키 페이지 10–15개를 생성하거나 갱신한다.

### Query (질의)
관련 페이지를 검색 후 종합 답변 생성. 유의미한 결과는 새 페이지로 저장.

### Lint (정리)
주기적으로 모순된 설명, 고아 페이지, 누락된 크로스레퍼런스를 검토한다.

## 태그 목록

| 태그 | 설명 |
|------|------|
| `ddd` | DDD 핵심 개념 |
| `laravel` | Laravel 프레임워크 |
| `eloquent` | Eloquent ORM |
| `aggregate` | Aggregate / Aggregate Root |
| `event` | 도메인 이벤트 |
| `repository` | 리포지토리 패턴 |
| `service` | 서비스 레이어 |
| `value-object` | 값 객체 |
| `cqrs` | CQRS 패턴 |
| `testing` | 테스트 |
| `anti-pattern` | 피해야 할 패턴 |
