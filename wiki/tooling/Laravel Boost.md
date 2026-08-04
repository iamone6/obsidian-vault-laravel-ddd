---
title: Laravel Boost (AI 코딩 에이전트 컨텍스트 도구)
category: tooling
tags: [laravel, boost, mcp, ai-guidelines, agent-skills, ai-coding]
related: [[Laravel MCP]], [[Static Analysis]], [[CI-CD Pipeline]], [[Directory Structure]]
---

# Laravel Boost (AI 코딩 에이전트 컨텍스트 도구)

Laravel 팀이 공식 제작한 `--dev` 전용 패키지. AI 코딩 에이전트(Claude Code, Cursor, Copilot 등)에게 "이 프로젝트가 실제로 어떻게 생겼는지"(프레임워크/패키지 버전, DB 스키마, 라우트, 로그, 컨벤션)를 알려준다. MIT 라이선스, 무료.

## 왜 필요한가

AI 에이전트가 코드베이스 접근·DB 스키마·프레임워크 버전 정보 없이 작업하면 "그럴듯하지만 틀린" 코드를 만든다 — 구버전 문법을 쓰거나, 이미 있는 스코프/패키지를 다시 만들거나, 존재하지 않는 컬럼을 참조하거나, 라우트 이름이 충돌한다. Boost는 AI가 코드를 짜기 전에 **실제 프로젝트 상태를 조회**할 수 있게 해서 이 문제를 줄인다.

## 3가지 기둥

| 기둥 | 로드 시점 | 역할 | 비유 |
|---|---|---|---|
| **AI Guidelines** | 항상 (세션 시작 시) | 핵심 컨벤션, 버전별 문법 차이 | 신입에게 첫날 주는 팀 코딩 컨벤션 문서 |
| **Agent Skills** | 관련 작업 시에만 | 상세 구현 패턴 (컨텍스트 절약) | 필요할 때 꺼내 보는 레퍼런스 매뉴얼 |
| **MCP Server** | 요청마다 | 실시간 조회/실행 도구 | 언제든 물어볼 수 있는 사수 개발자 |

Guidelines가 항상 전부 로드되던 초기 버전에서는 컨텍스트가 비대해지는 문제가 있었는데(패키지별 상세 패턴을 다 합치면 세션 시작에만 만 토큰 이상 소모), Boost 2.0에서 상세 내용을 Skills로 분리하면서 기본 가이드라인이 약 40% 가벼워졌다.

## 설치

```bash
composer require laravel/boost --dev   # --dev 필수 — 운영에 올릴 필요 없음
php artisan boost:install               # 대화형 설치 마법사
```

설치 마법사는 (1) Guidelines/Skills/MCP Server 중 무엇을 켤지, (2) 어떤 AI 에이전트(Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, Junie 등)를 쓰는지 물어보고, 선택한 에디터에 맞는 설정 파일을 자동 생성한다.

요구사항: PHP 8.2+, Laravel 10.x 이상(가이드라인은 10/11/12/13.x 지원).

### 생성되는 파일과 커밋 여부

```
프로젝트 루트/
├── [자동 생성 — .gitignore 권장]
│   ├── .mcp.json          MCP 서버 등록 정보
│   ├── CLAUDE.md           Claude Code용 가이드라인
│   ├── AGENTS.md           범용 에이전트 가이드라인
│   ├── boost.json          Boost 설정
│   └── .claude/skills/     설치된 스킬 (에이전트별 경로 다름)
│
└── [직접 작성 — 반드시 커밋 ★]
    └── .ai/
        ├── guidelines/     팀이 쓴 커스텀 규칙
        └── skills/         팀이 쓴 커스텀 스킬
```

`.mcp.json`/`CLAUDE.md`/`AGENTS.md`/`boost.json`/`.claude/skills/`는 `boost:install`/`boost:update` 실행 시 다시 만들어지므로 `.gitignore`에 넣어도 된다 — 개발자마다 쓰는 에디터가 다르므로 각자 생성하는 게 자연스럽다. 반면 **`.ai/`는 팀이 직접 쓴 자산이므로 절대 gitignore하면 안 된다** — 이걸 무시하면 팀원 전체가 손해를 본다.

에디터 수동 등록이 필요하면(자동 설정이 안 될 때) 등록 정보는 항상 이 형태다:

```json
{
    "mcpServers": {
        "laravel-boost": {
            "command": "php",
            "args": ["artisan", "boost:mcp"]
        }
    }
}
```

Claude Code라면 CLI로도 등록 가능: `claude mcp add -s local -t stdio laravel-boost php artisan boost:mcp`

## MCP 도구 — AI가 실시간으로 앱에 던지는 질문

| 도구 | 하는 일 |
|---|---|
| Application Info | PHP/Laravel 버전, DB 엔진, 설치된 패키지(버전 포함), Eloquent 모델 목록 |
| Database Schema / Connections | DB 스키마 전체 조회, 커넥션 목록 확인 |
| Database Query | 실제 쿼리 실행 |
| Route Inspector | 라우트 목록·이름 조회 |
| Tinker Integration | 앱 컨텍스트 안에서 임의 PHP 코드 실행 |
| Artisan Commands | 사용 가능한 Artisan 명령 조회·실행 |
| Search Docs | 설치된 패키지 버전 기준 공식 문서 검색 |
| Last Error / Read Log Entries | 최근 에러/로그 조회 |
| Browser Logs | 브라우저에서 발생한 로그·에러 읽기 |
| Configuration Access | config 값 조회 |
| Get Absolute URL | 상대 경로를 절대 URL로 변환 |

실제 활용 예: AI가 코드를 짜기 전에 `Tinker`로 `User::active()->count()`를 직접 실행해 스코프가 실제로 동작하는지, 데이터 규모가 얼마인지 확인한 뒤 그 결과(예: 12만 건)를 근거로 청크 처리 여부를 결정한다 — "그럴듯해 보이지만 안 돌아가는 코드"가 줄어드는 핵심 지점이다.

### 안전 주의사항 — 이 도구들은 읽기 전용이 아니다

`Database Query`(DELETE/UPDATE 포함), `Tinker`(임의 PHP 실행), `Artisan Commands`는 **실제로 쓰기/실행 권한을 가진다.**

- 로컬 개발 DB에서만 사용하고, `.env`가 운영 DB를 가리키지 않는지 항상 확인한다.
- 처음에는 에이전트의 자동 승인(auto-approve)을 끄고 수동 승인 모드로 어떤 도구가 언제 호출되는지 관찰한다.
- 필요하다면 읽기 전용 DB 유저로 접속하는 별도 커넥션을 구성한다.

## AI Guidelines — 프로젝트에 맞춘 규칙집

`composer.json`을 스캔해 **설치된 패키지의 버전에 맞는 가이드라인만** 조립한다. Livewire 3을 쓰면 Livewire 3 전용 문법(`wire:model`이 기본 defer)이, Inertia를 안 쓰면 Inertia 가이드라인은 아예 포함되지 않는다 — 이 버전 인식이 "구버전 문법을 쓰는" 흔한 AI 실수를 줄이는 핵심이다.

### 커스텀 가이드라인 작성

`.ai/guidelines/`에 `.md` 또는 `.blade.php` 파일을 넣으면 `boost:install` 시 기본 가이드라인과 자동으로 합쳐진다.

```markdown
<!-- .ai/guidelines/project-conventions.md -->
## 우리 프로젝트 컨벤션

### 아키텍처
- 비즈니스 로직은 반드시 app/Actions/ 아래 Action 클래스에 작성한다.
  컨트롤러는 요청 검증 → Action 호출 → 응답 반환만 담당한다.

### 테스트
- 테스트는 Pest로 작성한다. PHPUnit 문법(class ... extends TestCase) 금지.
```

`.blade.php` 확장자를 쓰면 조건부 로직도 가능하다 — 단, 코드 예제는 `@verbatim`으로 감싸야 Blade가 `{{ }}`를 해석하지 않는다.

```blade
{{-- .ai/guidelines/database.blade.php --}}
@if (config('database.default') === 'mysql')
- MySQL 8.0 사용 중. SKIP LOCKED, CTE, 윈도우 함수 사용 가능.
@endif
```

**기본 가이드라인 덮어쓰기**: Boost 기본 가이드라인이 팀 컨벤션과 안 맞으면, 같은 경로(`{패키지명}/{버전}/{주제}.blade.php`)에 파일을 만들면 내 것이 우선한다.

**패키지 개발자**라면 `resources/boost/guidelines/core.blade.php`를 패키지에 포함시켜, 사용자가 `boost:install`을 실행할 때 자동으로 로드되게 할 수 있다.

## Agent Skills — 필요할 때만 로드되는 상세 지식

기본 제공 스킬은 `composer.json`에 해당 패키지(Livewire, Pest, Tailwind, Inertia, Flux UI, Folio, Pennant, Volt, Wayfinder, MCP)가 있으면 자동 설치된다. Filament은 스킬 목록에는 없지만 Documentation API 대상에는 포함된다.

### 커스텀 스킬 작성

`.ai/skills/{스킬이름}/SKILL.md` 형식이며, [Agent Skills 포맷](https://agentskills.io/what-are-skills)을 따른다.

```markdown
---
name: creating-invoices
description: 청구서(Invoice) 생성, 수정, 발행 작업 시 사용. 청구서 번호 채번, 세금 계산, PDF 생성 로직을 포함한다.
---

# 청구서 생성 가이드

## 핵심 규칙

### 1. 절대 Invoice를 직접 생성하지 말 것
`Invoice::create()`를 직접 호출하면 청구서 번호 채번이 누락된다.
반드시 CreateInvoiceAction을 통한다.
```

`name`/`description` frontmatter는 필수다. 특히 `description`은 AI가 "지금 이 스킬을 켜야 하나?"를 판단하는 근거이므로, "청구서 관련 스킬"처럼 모호하게 쓰면 필요할 때 활성화되지 않는다 — 활성화 조건(어떤 작업, 어떤 도메인 용어)을 명확히 적어야 한다.

기본 스킬 덮어쓰기도 가이드라인과 동일하게 같은 이름(`.ai/skills/livewire-development/`)으로 만들면 우선한다. **차이점: 커스텀 가이드라인은 `boost:install` 시 병합되고, 커스텀 스킬은 `boost:update` 시 설치된다** — 헷갈리면 둘 다 실행해도 무방하다.

## Guidelines vs Skills — 선택 기준

```
"이 규칙이 모든 작업에 적용되나?"
   ├─ YES → Guidelines (예: "비즈니스 로직은 Action 클래스에", "우리는 Pest를 쓴다")
   └─ NO  → Skills     (예: "청구서 세금 계산은 TaxCalculator로", "Livewire 폼 검증 상세 패턴")
```

가이드라인은 가능한 짧게 유지한다 — 100줄을 넘어가면 스킬로 뺄 시점이다.

## Documentation API — 17,000개 이상의 문서 조각

Laravel 생태계(Framework, Filament, Livewire, Inertia, Pest, Tailwind, Nova 등) 문서를 임베딩 기반 **시맨틱 검색**으로 찾는다("다대다 관계 어떻게 설정하지?"처럼 의미로 질문해도 `belongsToMany` 문서를 찾아준다). 설치된 패키지의 실제 버전 기준으로 필터링되므로, Livewire 3을 쓰는데 Livewire 4 문서가 섞여 들어오지 않는다.

**프라이버시**: 문서 검색 시 검색어와 설치된 패키지 버전 정보가 Laravel 호스팅 엔드포인트로 전송된다. 애플리케이션 코드 자체는 로컬에 남고, MCP 서버 본체도 전적으로 로컬에서 동작한다. 보안 정책이 엄격한 환경이라면 이 외부 통신 여부를 조직 정책과 대조해 확인한다.

정적으로 관리하는 `.ai/guidelines`/`.ai/skills`(팀 고유 규칙, 오프라인 가능)와 `search-docs`(공식 API 시그니처, 항상 최신)는 서로 대체재가 아니라 **함께 쓰는 게 정석**이다.

## "싹부터 자르기" 전략 — 살아있는 가이드라인 운영법

Jeffrey Way(Laracasts)가 소개한 워크플로로, Boost 활용법 중 가장 실전적인 팁으로 꼽힌다.

> AI가 실수했을 때, 그 실수만 고치고 넘어가지 말고 **가이드라인을 갱신하라.** 사례가 아니라 패턴을 고쳐라.

```
AI가 실수함 → 수정함 → (여기서 멈추지 않고) → .ai/guidelines/에 규칙 한 줄 추가 → 같은 실수가 영구적으로 사라짐
```

이렇게 누적된 가이드라인은 시간이 지나며 "왜 안 되는지"까지 담긴 팀의 살아있는 문서가 되고, 신규 팀원 온보딩 자료로도 재활용된다. 운영 팁: 거창하게 쓰지 말고 한 줄로, 이유도 짧게 함께 적고, 길어지면 Skills로 쪼갠다. `boost:update`는 `.ai/`를 건드리지 않으므로 안심하고 실행해도 된다.

## Herd MCP와의 관계 — 애플리케이션 레벨 vs 인프라 레벨

macOS/Windows에서 [Laravel Herd](https://herd.laravel.com)를 쓰는 경우, Boost(애플리케이션 레벨: 모델·스키마·라우트·로그)와 Herd MCP(인프라 레벨: 로컬 URL, 실행 중인 서비스, 디버그 세션)는 경쟁이 아니라 보완 관계다. Herd MCP의 `debug_site` 기능은 페이지 요청 중 실행된 쿼리를 캡처해 N+1 패턴을 즉시 잡아낼 수 있다. Boost 자체는 Herd 없이도 완전히 동작하며, WSL/Docker 환경이라면 Herd MCP는 해당 사항이 없다.

## 유지보수 — `boost:update`

```bash
php artisan boost:update              # 기존 가이드라인/스킬을 최신 버전으로 갱신
php artisan boost:update --discover   # + 새로 설치된 패키지를 스캔해 신규 가이드라인/스킬 제안
```

`composer.json`의 `post-update-cmd`에 등록해두면 `composer update`마다 자동 갱신된다.

```json
{
  "scripts": {
    "post-update-cmd": [
      "@php artisan boost:update --ansi"
    ]
  }
}
```

`.ai/` 아래 커스텀 자산은 `boost:update`가 건드리지 않으므로 그대로 유지된다.

## 커스텀 에이전트 확장 (고급)

Boost 기본 지원 목록에 없는 AI 도구를 쓴다면, `Laravel\Boost\Install\Agents\Agent`를 상속하고 필요한 계약(`SupportsGuidelines`/`SupportsMcp`/`SupportsSkills`)만 구현해 `Boost::registerAgent()`로 [[Service Provider|서비스 프로바이더]]의 `boot()`에서 등록할 수 있다. 사내 자체 AI 도구를 쓰거나 Boost가 아직 지원하지 않는 신규 에디터를 통합할 때만 필요한, 대부분의 개발자에게는 불필요한 고급 기능이다.

## 주의사항 / 안티패턴

- **운영 배포에 포함하지 말 것**: `--dev`로 설치하면 `composer install --no-dev`인 배포 환경엔 포함되지 않는다 — 개발 도구이므로 운영에 올릴 이유가 없다. 배포 스크립트가 실제로 `--no-dev`를 쓰는지 확인한다.
- **자동 생성 파일과 `.ai/`를 혼동해 전부 gitignore하지 말 것**: `.mcp.json`/`CLAUDE.md`/`AGENTS.md`/`boost.json`/`.claude/skills/`는 gitignore해도 되지만, `.ai/guidelines/`와 `.ai/skills/`는 팀 자산이므로 반드시 커밋한다.
- **자동 승인 모드로 방치하지 말 것**: `Database Query`/`Tinker`/`Artisan Commands`는 쓰기·실행 권한이 있으므로, 특히 도입 초기에는 수동 승인으로 어떤 도구가 언제 호출되는지 관찰한다.
- **Docker/WSL 환경에서 DB_HOST 불일치를 주의할 것**: MCP 서버가 컨테이너 밖(호스트)에서 실행되면 `.env`의 `DB_HOST=mysql`(Compose 서비스명)을 해석하지 못해 연결이 실패한다 — MCP 서버 자체를 `docker compose exec` 또는 `wsl` 커맨드로 감싸 같은 환경 안에서 실행해야 한다.
- **Laravel 9 이하는 지원하지 않는다** — 업그레이드가 선행돼야 한다.

## 참고

- [[Laravel MCP]] — Boost의 "MCP Server" 기둥이 사용하는 기반 프로토콜. 다만 `laravel/mcp`는 **내 앱의 업무 기능**(주문 조회 등)을 AI 클라이언트에 노출하는 툴킷이고, Boost는 **AI 에이전트에게 내 프로젝트 자체의 상태**(스키마·라우트·버전·컨벤션)를 알려주는 완제품 MCP 서버라는 점에서 방향이 다르다.
- [[Directory Structure]] — `.ai/guidelines/`, `.ai/skills/`도 프로젝트 구조에 커밋되는 팀 자산이라는 점에서 같은 맥락
- [[Static Analysis]] / [[CI-CD Pipeline]] — 코드 품질을 자동으로 지켜주는 다른 개발 도구들과 같은 카테고리
- 소스: Laravel Boost 완전 정복 (laravel.com/ai/boost, Laravel 13.x 공식 문서, Hafiz Riaz 블로그 종합)
