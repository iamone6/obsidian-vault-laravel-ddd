# Laravel Boost 완전 정복 — 입문부터 실전까지

> **한 줄 요약**
> Laravel Boost는 **AI 코딩 에이전트(Claude Code, Cursor, Copilot 등)에게 "내 Laravel 프로젝트가 어떻게 생겼는지"를 알려주는 공식 도구**다.
> Laravel 팀이 직접 만들었고, MIT 라이선스 오픈소스이며 무료다.

**작성 기준**
- 출처: [laravel.com/ai/boost](https://laravel.com/ai/boost), [Laravel 13.x 공식 문서 – Boost](https://laravel.com/docs/13.x/boost), [Hafiz Riaz 블로그](https://hafiz.dev/blog/laravel-boost-and-mcp-servers-the-context-your-ai-agent-is-missing)
- 문서 작성일: 2026-07-29
- Boost 버전: 공식 사이트 기준 **v2.4.13**

---

## 목차

**[입문편]**
1. [왜 Boost가 필요한가 — 문제부터 이해하기](#1-왜-boost가-필요한가--문제부터-이해하기)
2. [MCP가 뭔데? — 5분 개념 정리](#2-mcp가-뭔데--5분-개념-정리)
3. [Boost의 3가지 기둥](#3-boost의-3가지-기둥)
4. [설치 — 3분이면 끝](#4-설치--3분이면-끝)
5. [에디터별 설정 방법](#5-에디터별-설정-방법)

**[중급편]**

6. [MCP 도구 전체 목록과 활용법](#6-mcp-도구-전체-목록과-활용법)
7. [AI Guidelines — 항상 로드되는 규칙집](#7-ai-guidelines--항상-로드되는-규칙집)
8. [Agent Skills — 필요할 때만 꺼내 쓰는 지식](#8-agent-skills--필요할-때만-꺼내-쓰는-지식)
9. [Guidelines vs Skills — 언제 뭘 쓰나](#9-guidelines-vs-skills--언제-뭘-쓰나)
10. [Documentation API — 17,000개의 문서 조각](#10-documentation-api--17000개의-문서-조각)

**[실전편]**

11. [실전 워크플로우 — Boost가 있고 없고의 차이](#11-실전-워크플로우--boost가-있고-없고의-차이)
12. ["싹부터 자르기" 전략 — 커스텀 가이드라인 운영법](#12-싹부터-자르기-전략--커스텀-가이드라인-운영법)
13. [Herd MCP 서버와 함께 쓰기](#13-herd-mcp-서버와-함께-쓰기)
14. [유지보수 — boost:update 자동화](#14-유지보수--boostupdate-자동화)
15. [Boost 확장하기 — 커스텀 에이전트 만들기](#15-boost-확장하기--커스텀-에이전트-만들기)
16. [트러블슈팅 & FAQ](#16-트러블슈팅--faq)
17. [도입 체크리스트](#17-도입-체크리스트)

---

# 입문편

## 1. 왜 Boost가 필요한가 — 문제부터 이해하기

### 상황을 상상해 보자

아주 뛰어난 시니어 개발자를 채용했다고 하자. 그런데 그 사람에게:

- 코드베이스 접근 권한을 안 줬다
- DB 스키마를 안 알려줬다
- 우리가 쓰는 프레임워크 버전도 안 알려줬다

그러고서 "회원 CSV 내보내기 기능 만들어주세요"라고 시켰다.

그 사람은 **추측**할 수밖에 없다. 그리고 자주 틀린다.

**AI 코딩 에이전트를 컨텍스트 없이 쓰는 게 정확히 이 상황이다.**

### AI가 실제로 틀리는 것들

| AI가 모르는 것 | 그래서 벌어지는 일 |
|---|---|
| 내 프로젝트가 Laravel 13인지 10인지 | 구버전 문법으로 코드를 짬 |
| `User` 모델에 `scopeActive()`가 있는지 | 똑같은 스코프를 또 만듦 |
| 지난달 Inertia → Livewire로 갈아탄 것 | Inertia 코드를 생성함 |
| Livewire 3에서 `wire:model.defer`가 없어진 것 | 존재하지 않는 문법을 씀 |
| `users` 테이블에 어떤 컬럼이 있는지 | 없는 컬럼을 참조하는 쿼리를 짬 |
| 이미 `users.export`라는 라우트가 있는 것 | 라우트 이름 충돌을 일으킴 |
| Laravel Excel이 이미 설치돼 있는 것 | `fputcsv`로 직접 다 짜버림 |

> 블로그 저자의 표현을 빌리면: AI가 나쁜 Laravel 코드를 쓰는 건 **멍청해서가 아니라 앞이 안 보여서**다.

### Boost가 하는 일

```
[Boost 없을 때]

개발자: "CSV 내보내기 만들어줘"
   ↓
AI: (파일 몇 개 읽고) "음... 아마 이럴 것 같은데?"
   ↓
결과: 동작은 하지만 우리 프로젝트에 안 맞는 코드


[Boost 있을 때]

개발자: "CSV 내보내기 만들어줘"
   ↓
AI: → app-info 도구 호출     → "Laravel Excel 설치돼 있네"
    → schema 도구 호출       → "users 테이블 컬럼은 이거구나"
    → routes 도구 호출       → "라우트 네이밍은 이 패턴이구나"
    → search-docs 도구 호출  → "이 버전 Excel API는 이렇구나"
   ↓
결과: 우리 패키지, 우리 컨벤션, 우리 스키마에 맞는 코드
```

---

## 2. MCP가 뭔데? — 5분 개념 정리

### MCP = Model Context Protocol

**AI 에이전트와 개발 환경 사이의 표준 API 규약**이다.

Laravel 개발자에게 익숙한 비유로 설명하면:

```
[MCP 없을 때] — AI가 파일을 그냥 읽음
    grep으로 소스 뒤지는 것과 비슷하다.
    파일은 읽지만 "실제로 지금 DB에 뭐가 있는지"는 모른다.

[MCP 있을 때] — AI가 앱에 질문을 던짐
    Artisan 명령어나 API 엔드포인트를 호출하는 것에 가깝다.
    "DB 스키마 알려줘" → 앱이 진짜 스키마를 응답
```

REST API로 비유하면 이렇다:

```
클라이언트(브라우저)  ←→  REST API  ←→  Laravel 앱
AI 에이전트          ←→  MCP       ←→  Laravel 앱
```

REST API가 "웹 클라이언트가 서버 기능을 쓰는 표준"이라면,
MCP는 **"AI가 로컬 환경 기능을 쓰는 표준"**이다.

### Laravel 생태계의 MCP 서버 2개

| 서버 | 담당 영역 | 제공자 |
|---|---|---|
| **Laravel Boost** | 애플리케이션 레벨 (모델, 라우트, 스키마, 로그, 문서) | Laravel 공식 |
| **Herd MCP** | 인프라 레벨 (로컬 URL, 실행 중인 서비스, 디버그) | Laravel Herd |

둘은 경쟁 관계가 아니라 **보완 관계**다. (13번 섹션에서 함께 쓰는 법 다룸)

---

## 3. Boost의 3가지 기둥

Boost를 이해하려면 이 3가지 구분이 핵심이다.

```
┌──────────────────────────────────────────────────┐
│               Laravel Boost                      │
├──────────────────────────────────────────────────┤
│                                                  │
│  ① AI Guidelines   ─── 항상 로드되는 규칙집       │
│     "이 프로젝트는 Laravel 13, Livewire 4다"      │
│     "Laravel 컨벤션은 이렇다"                     │
│     → 시작할 때 컨텍스트에 통째로 들어감           │
│                                                  │
│  ② Agent Skills    ─── 필요할 때만 로드되는 지식   │
│     "Pest 테스트 작성법 상세"                     │
│     "Livewire 컴포넌트 패턴 상세"                 │
│     → 관련 작업 시에만 활성화 (컨텍스트 절약)      │
│                                                  │
│  ③ MCP Server      ─── 실시간 조회 도구 모음       │
│     "지금 DB 스키마 뭐야?" → 실제 조회             │
│     "이 쿼리 돌려봐" → 실제 실행                  │
│     → AI가 앱과 직접 대화                         │
│                                                  │
└──────────────────────────────────────────────────┘
```

**한 줄 비유**

| 기둥 | 비유 |
|---|---|
| Guidelines | 신입에게 첫날 나눠주는 **팀 코딩 컨벤션 문서** |
| Skills | 책장에 꽂혀 있다가 필요할 때 꺼내 보는 **레퍼런스 매뉴얼** |
| MCP Tools | 언제든 물어볼 수 있는 **사수 개발자** |

---

## 4. 설치 — 3분이면 끝

### 4-1. 요구사항

- PHP 8.2 이상
- Laravel 10.x 이상 (가이드라인은 10.x / 11.x / 12.x / 13.x 지원)

> 정확한 최소 요구 버전은 설치 시점의 `composer.json` 제약을 따르므로, `composer require` 실행 시 에러가 나면 메시지를 확인하자.

### 4-2. 설치 명령어

```bash
# 1단계: 개발 의존성으로 설치 (--dev 필수! 운영에 올릴 필요 없다)
composer require laravel/boost --dev

# 2단계: 대화형 설치 마법사 실행
php artisan boost:install
```

### 4-3. 설치 마법사가 물어보는 것

`boost:install`은 대화형이다. 두 가지를 물어본다.

**질문 1: 어떤 기능을 쓸 건가?**

```
? Which features would you like to install?
  ◉ AI Guidelines    ← 켜자 (기본 컨벤션)
  ◉ Agent Skills     ← 켜자 (상세 패턴, 컨텍스트 절약)
  ◉ MCP Server       ← 켜자 (핵심 기능)
```

**질문 2: 어떤 AI 에이전트를 쓰나?**

```
? Which agents do you use?
  ◉ Claude Code
  ◯ Cursor
  ◯ Codex
  ◯ Gemini CLI
  ◯ GitHub Copilot (VS Code)
  ◯ Junie (JetBrains)
```

선택한 에디터에 맞는 설정 파일이 자동으로 생성된다.

### 4-4. 생성되는 파일들

```
your-laravel-project/
├── .mcp.json          # MCP 서버 등록 정보 (에디터가 읽음)
├── CLAUDE.md          # Claude Code용 가이드라인 (자동 생성)
├── AGENTS.md          # Codex 등 다른 에이전트용 가이드라인
├── boost.json         # Boost 자체 설정
├── .claude/
│   └── skills/        # Claude Code용 스킬 (에이전트마다 경로 다름)
│       ├── pest-testing/
│       │   └── SKILL.md
│       └── livewire-development/
│           └── SKILL.md
└── .ai/               # ★ 내가 직접 만드는 커스텀 자산 (7, 8번 섹션 참고)
    ├── guidelines/
    └── skills/
```

> **주의**: 스킬 설치 경로는 에이전트마다 다르다. Claude Code는 `.claude/skills/`, GitHub Copilot은 `.github/skills/` 식이다. 여러 에이전트를 동시에 선택하면 각각의 경로에 생성된다.

### 4-5. `.gitignore` 설정 — 중요한 구분

공식 문서는 **자동 생성 파일들을 `.gitignore`에 넣어도 된다**고 안내한다. `boost:install` / `boost:update` 실행 시 다시 만들어지기 때문이다.

```gitignore
# .gitignore

# Boost가 자동 생성하는 파일들 — 커밋 안 해도 됨
/.mcp.json
/CLAUDE.md
/AGENTS.md
/boost.json
/.junie/
```

**단, `.ai/` 디렉터리는 절대 무시하면 안 된다.**

```
.mcp.json, CLAUDE.md 등  → 자동 생성물. gitignore 가능.
.ai/guidelines/*         → 내가 직접 쓴 팀 규칙. 반드시 커밋! ★
.ai/skills/*             → 내가 직접 쓴 커스텀 스킬. 반드시 커밋! ★
```

이 구분이 중요한 이유:
- 자동 생성물은 개발자마다 쓰는 에디터가 다르므로 각자 생성하는 게 낫다
- `.ai/`는 **팀의 자산**이다. 이걸 gitignore하면 팀원 전체가 손해다

---

## 5. 에디터별 설정 방법

대부분 `boost:install`이 자동으로 처리하지만, 안 될 때를 위한 수동 방법이다.

### Claude Code

```bash
# 보통은 자동 설정된다. 안 되면 프로젝트 디렉터리에서:
claude mcp add -s local -t stdio laravel-boost php artisan boost:mcp
```

**옵션 설명**
- `-s local` : 이 프로젝트에서만 사용 (scope = local)
- `-t stdio` : 통신 방식 = 표준입출력
- `laravel-boost` : MCP 서버 이름
- `php artisan boost:mcp` : 실제 실행 명령

### Cursor

```
1. 커맨드 팔레트 열기 (Cmd+Shift+P / Ctrl+Shift+P)
2. "/open MCP Settings" 선택 후 Enter
3. laravel-boost 토글을 ON으로
```

### Codex CLI

```bash
codex mcp add laravel-boost -- php "artisan" "boost:mcp"
```

### Gemini CLI

```bash
gemini mcp add -s project -t stdio laravel-boost php artisan boost:mcp
```

### GitHub Copilot (VS Code)

```
1. 커맨드 팔레트 열기 (Cmd+Shift+P / Ctrl+Shift+P)
2. "MCP: List Servers" 선택
3. laravel-boost 로 이동 후 Enter
4. "Start server" 선택
```

### PhpStorm / Junie (JetBrains)

```
1. Shift 두 번 눌러 커맨드 팔레트 열기
2. "MCP Settings" 검색 후 Enter
3. laravel-boost 옆 체크박스 체크
4. 우측 하단 "Apply" 클릭
5. 초록색 체크마크가 보이면 성공
```

### 완전 수동 등록 (위 방법이 다 안 될 때)

MCP 서버 등록 정보는 딱 두 줄이다.

| 항목 | 값 |
|---|---|
| Command | `php` |
| Args | `artisan boost:mcp` |

JSON 형식:

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

---

# 중급편

## 6. MCP 도구 전체 목록과 활용법

### 6-1. 공식 문서에 명시된 도구

| 도구 이름 | 하는 일 |
|---|---|
| **Application Info** | PHP·Laravel 버전, DB 엔진, 설치된 생태계 패키지 목록(버전 포함), Eloquent 모델 목록 조회 |
| **Browser Logs** | 브라우저에서 발생한 로그와 에러 읽기 |
| **Database Connections** | 사용 가능한 DB 커넥션 목록 및 기본 커넥션 확인 |
| **Database Query** | DB에 실제 쿼리 실행 |
| **Database Schema** | DB 스키마 전체 읽기 |
| **Get Absolute URL** | 상대 경로를 절대 URL로 변환 (AI가 올바른 URL을 생성하도록) |
| **Last Error** | 로그 파일에서 마지막 에러 읽기 |
| **Read Log Entries** | 최근 N개 로그 엔트리 읽기 |
| **Search Docs** | Laravel 호스팅 문서 API 검색 (설치된 패키지 기준) |

### 6-2. 제품 소개 페이지에서 추가로 언급되는 도구

공식 제품 페이지(`laravel.com/ai/boost`)는 **총 15개의 전문 도구**를 제공한다고 소개하며, 위 목록 외에 다음도 언급한다:

| 도구 | 하는 일 |
|---|---|
| **Route Inspector** | 애플리케이션 라우트 조회·분석 |
| **Artisan Commands** | 사용 가능한 Artisan 명령 목록 조회 및 상세 확인 |
| **Tinker Integration** | 앱 컨텍스트 안에서 제안된 코드를 실제로 실행 |
| **Configuration Access** | config 값 및 사용 가능한 키 조회 |

> 문서 표와 제품 페이지 목록이 조금 다르다. 버전에 따라 도구 구성이 바뀔 수 있으므로, **실제 사용 가능한 도구는 에디터의 MCP 도구 목록에서 확인**하는 게 확실하다.

### 6-3. 도구가 실제로 어떻게 쓰이는지 — 대화 예시

**예시 1: 스키마 확인**

```
나: "주문 목록 페이지 만들어줘. 상태별 필터링 필요해."

AI: (내부적으로)
    → Database Schema 도구 호출
    ← orders 테이블: id, user_id, status(enum), total_amount, created_at
    ← status enum 값: pending, paid, shipped, cancelled

AI: "orders 테이블의 status는 pending/paid/shipped/cancelled
     4가지 enum이네요. 이 값들로 필터를 구성하겠습니다."
```

이게 없었다면 AI는 `status` 값을 `'active'`, `'inactive'` 같이 **지어냈을** 것이다.

**예시 2: Tinker로 가설 검증** (블로그 저자가 가장 저평가됐다고 꼽은 기능)

```
나: "활성 사용자 대상으로 이메일 발송 기능 만들어줘"

AI: (내부적으로)
    → Tinker 도구로 실행: User::active()->count()
    ← 결과: 1,247

AI: "User::active() 스코프가 실제로 동작하고 1,247명이 조회되네요.
     이 스코프를 재사용해서 구현하겠습니다."
```

**AI가 코드를 짜기 전에 자기 가정을 검증**한다. "그럴듯해 보이는데 안 돌아가는 코드" 생성이 크게 줄어든다.

**예시 3: 에러 디버깅**

```
나: "사용자 목록 페이지가 백지로 나와. 고쳐줘."

AI: (내부적으로)
    → Last Error 도구 호출
    ← "Undefined property: App\Models\User::$profile_image
       at resources/views/users/index.blade.php:24"

AI: "24번 줄에서 존재하지 않는 profile_image 속성을 참조하고 있습니다.
     스키마를 보니 컬럼명은 avatar_path네요."
```

스택트레이스를 복사해서 붙여넣을 필요가 없어진다.

### 6-4. 안전 관련 주의사항

Boost의 MCP 도구는 **쓰기 작업도 가능하다.** 앞서 "읽기 전용"이라고 알고 계셨다면 정정이 필요하다.

- `Database Query` → 실제 쿼리 실행 (DELETE/UPDATE도 원칙적으로 가능)
- `Tinker` → 임의의 PHP 코드 실행
- `Artisan Commands` → 명령 실행

따라서:

```
✅ 반드시 지킬 것
- 로컬 개발 DB에서만 쓴다
- .env의 DB 접속 정보가 운영 DB를 가리키지 않는지 확인한다
- --dev 로 설치해서 운영 배포에 포함되지 않게 한다
- 에이전트의 자동 실행(auto-approve) 설정을 신중하게 한다
```

특히 Claude Code 같은 도구에서 "모든 도구 자동 승인"을 켜두면, AI가 판단 착오로 파괴적인 쿼리를 실행할 여지가 생긴다. 처음에는 **수동 승인 모드**로 쓰면서 어떤 도구를 언제 호출하는지 관찰해 보는 편이 안전하다.

---

## 7. AI Guidelines — 항상 로드되는 규칙집

### 7-1. 개념

**가이드라인은 AI 에이전트가 시작할 때 컨텍스트에 통째로 들어가는 지시문 파일**이다.

Laravel 개발자 비유로는 이렇다:

```
Guidelines  ≈  AppServiceProvider의 boot()
              → 앱 시작할 때 무조건 한 번 실행

Skills      ≈  Lazy loading 관계
              → 실제로 접근할 때만 로드
```

가이드라인에는 이런 게 들어간다:
- Laravel 코어 컨벤션
- 설치된 패키지별 베스트 프랙티스
- 버전별 문법 차이

### 7-2. 버전 인식(Version-aware)이 핵심

이게 생각보다 큰 차이다.

```
[일반 AI]
"Livewire에서 입력값 지연시키려면 wire:model.defer 쓰세요"
    → Livewire 3에서는 존재하지 않는 문법. 조용히 안 먹힘.

[Boost 가이드라인 적용]
"Livewire 3에서 wire:model은 기본이 defer입니다.
 실시간 반응이 필요할 때만 wire:model.live를 쓰세요."
    → 정확함.
```

Boost는 `composer.json`을 스캔해서 **설치된 패키지의 버전에 맞는 가이드라인만** 조립한다. Inertia를 안 쓰면 Inertia 가이드라인은 아예 들어가지 않으므로 컨텍스트 낭비도 없다.

### 7-3. 기본 제공 가이드라인 목록

| 패키지 | 지원 버전 |
|---|---|
| Core & Boost | core |
| Laravel Framework | core, 10.x, 11.x, 12.x, 13.x |
| Livewire | core, 2.x, 3.x, 4.x |
| Flux UI | core, free, pro |
| Folio | core |
| **Herd** | core |
| Inertia Laravel | core, 1.x, 2.x, 3.x |
| Inertia React | core, 1.x, 2.x, 3.x |
| Inertia Vue | core, 1.x, 2.x, 3.x |
| Inertia Svelte | core, 1.x, 2.x, 3.x |
| MCP | core |
| Pennant | core |
| Pest | core, 3.x, 4.x |
| PHPUnit | core |
| Pint | core |
| Sail | core |
| Tailwind CSS | core, 3.x, 4.x |
| Livewire Volt | core |
| Wayfinder | core |
| Enforce Tests | conditional |

> `core`는 버전 무관하게 적용되는 일반 조언이다.

### 7-4. 내 가이드라인 추가하기 ★

**여기가 Boost의 진짜 힘이 나오는 지점이다.**

`.ai/guidelines/` 디렉터리에 `.md` 또는 `.blade.php` 파일을 넣으면, `boost:install` 시 Boost의 기본 가이드라인과 **자동으로 합쳐진다.**

```bash
mkdir -p .ai/guidelines
```

**예시: `.ai/guidelines/project-conventions.md`**

```markdown
## 우리 프로젝트 컨벤션

### 아키텍처
- 비즈니스 로직은 반드시 `app/Actions/` 아래 Action 클래스에 작성한다.
  컨트롤러는 요청 검증 → Action 호출 → 응답 반환만 담당한다.
- 도메인별로 `app/Modules/{도메인}/` 아래에 모듈을 구성한다.
- 모듈 간 직접 참조 금지. 반드시 인터페이스를 통해 접근한다.

### 테스트
- 테스트는 Pest로 작성한다. PHPUnit 문법(`class ... extends TestCase`) 금지.
- 모든 Action 클래스는 대응하는 Feature 테스트가 있어야 한다.

### 요청/응답
- 요청 검증은 Laravel Data DTO를 사용한다. FormRequest 신규 작성 금지.
- API 응답은 반드시 `App\Support\ApiResponse` 클래스를 거친다.
  `response()->json()` 직접 호출 금지.

### 큐
- 큐 이름 규칙: `{도메인}-{동작}`
  예: `billing-process-payment`, `notification-send-email`

### 라우팅
- 라우트 모델 바인딩은 가능한 커스텀 키를 사용한다. (id 노출 최소화)
```

**Blade 문법도 쓸 수 있다** (`.blade.php` 확장자 사용 시). 조건부 가이드라인 작성에 유용하다.

```blade
{{-- .ai/guidelines/database.blade.php --}}

## 데이터베이스 규칙

@if (config('database.default') === 'mysql')
- MySQL 8.0을 사용 중이다. `SKIP LOCKED`, CTE, 윈도우 함수 사용 가능.
@endif

- 마이그레이션은 되돌릴 수 있게(`down()` 구현) 작성한다.
- 외래 키는 항상 명시적으로 선언한다.
- 대량 조회는 `chunk()` 또는 `lazy()`를 사용한다.
```

> 코드 예제를 넣을 때는 `@verbatim` ~ `@endverbatim`으로 감싸야 Blade가 `{{ }}`를 해석하지 않는다.

### 7-5. 기본 가이드라인 덮어쓰기(Override)

Boost의 기본 가이드라인이 우리 팀 방식과 안 맞을 수 있다. 이때는 **같은 경로에 파일을 만들면 내 것이 우선**한다.

```
Boost 기본:  Inertia React v2 Form Guidance
내가 덮어쓰기: .ai/guidelines/inertia-react/2/forms.blade.php
                                ↑           ↑    ↑
                            패키지명      버전  주제

→ boost:install 실행 시 기본 대신 내 파일이 포함됨
```

### 7-6. 패키지 개발자를 위한 가이드라인 제공

내가 만든 패키지에 Boost 가이드라인을 포함시킬 수도 있다.

```
my-package/
└── resources/
    └── boost/
        └── guidelines/
            └── core.blade.php   ← 이 파일을 추가
```

사용자가 `php artisan boost:install`을 실행하면 자동으로 로드된다.

**작성 예시:**

```blade
## MyPackage

이 패키지는 [기능 요약]을 제공합니다.

### 주요 기능

- 기능 1: [간결한 설명]
- 기능 2: [간결한 설명]. 사용 예:

@verbatim
<code-snippet name="기능 2 사용법" lang="php">
$result = MyPackage::featureTwo($param1, $param2);
</code-snippet>
@endverbatim
```

**작성 원칙**: 짧게, 실행 가능하게, 베스트 프랙티스 중심으로. 장황한 문서는 오히려 컨텍스트만 잡아먹는다.

---

## 8. Agent Skills — 필요할 때만 꺼내 쓰는 지식

### 8-1. Skills가 등장한 이유: 컨텍스트 비대화

가이드라인은 **항상, 전부** 로드된다. 그래서 이런 문제가 생긴다.

```
[가이드라인만 있던 시절]

Laravel 컨벤션      (2,000 토큰)
Livewire 상세 패턴  (3,000 토큰)
Pest 상세 패턴      (2,500 토큰)
Tailwind 상세 패턴  (2,000 토큰)
Inertia 상세 패턴   (3,000 토큰)
─────────────────────────────
합계               12,500 토큰이 시작하자마자 소모

→ 정작 내 코드를 읽을 컨텍스트가 부족해짐
→ 지금 하는 작업과 무관한 정보가 노이즈로 작용
```

Boost 2.0에서 상세 내용을 Skills로 옮기면서 **기본 가이드라인이 약 40% 가벼워졌다.**

### 8-2. Skills 동작 방식

```
에이전트 시작
   ↓
가이드라인만 로드 (가벼움)
   ↓
"Livewire 컴포넌트 만들어줘" ← 작업 시작
   ↓
livewire-development 스킬 자동 활성화 ★
   ↓
Livewire 상세 패턴이 그때 로드됨
```

Pest 테스트를 쓸 땐 Pest 스킬만, Tailwind 작업을 할 땐 Tailwind 스킬만 켜진다.

### 8-3. 기본 제공 스킬 목록

| 스킬 이름 | 대상 패키지 |
|---|---|
| `fluxui-development` | Flux UI |
| `folio-routing` | Folio |
| `inertia-react-development` | Inertia React |
| `inertia-svelte-development` | Inertia Svelte |
| `inertia-vue-development` | Inertia Vue |
| `livewire-development` | Livewire |
| `mcp-development` | MCP |
| `pennant-development` | Pennant |
| `pest-testing` | Pest |
| `tailwindcss-development` | Tailwind CSS |
| `volt-development` | Volt |
| `wayfinder-development` | Wayfinder |

**자동 설치된다.** `composer.json`에 `livewire/livewire`가 있으면 `livewire-development` 스킬이 알아서 깔린다.

> 참고: Filament은 **스킬 목록에는 없지만** Documentation API 대상에는 포함된다(10번 섹션). Filament 관련 정보는 `search-docs` 도구를 통해 조회된다.

### 8-4. 커스텀 스킬 만들기 ★

우리 도메인 로직에 특화된 스킬을 직접 만들 수 있다.

```
.ai/skills/{스킬이름}/SKILL.md
```

**예시: `.ai/skills/creating-invoices/SKILL.md`**

```markdown
---
name: creating-invoices
description: 청구서(Invoice) 생성, 수정, 발행 작업 시 사용. 청구서 번호 채번, 세금 계산, PDF 생성 로직을 포함한다.
---

# 청구서 생성 가이드

## 언제 이 스킬을 쓰나

청구서(Invoice), 청구 항목(InvoiceLine), 세금 계산, 청구서 PDF 발행과
관련된 작업을 할 때 사용한다.

## 핵심 규칙

### 1. 절대 Invoice를 직접 생성하지 말 것

`Invoice::create()`를 직접 호출하면 청구서 번호 채번이 누락된다.
반드시 `CreateInvoiceAction`을 통한다.

```php
// ❌ 잘못된 방법 — 채번 로직을 건너뜀
$invoice = Invoice::create([
    'customer_id' => $customer->id,
    'total' => 100000,
]);

// ✅ 올바른 방법
$invoice = app(CreateInvoiceAction::class)->execute(
    customer: $customer,
    lines: $lines,           // InvoiceLineData[] 배열
    issuedAt: now(),
);
```

### 2. 금액은 항상 정수(원 단위)로 다룬다

부동소수점 오차를 피하기 위해 금액은 `integer` 컬럼에 원 단위로 저장한다.
`decimal`, `float` 사용 금지.

```php
// ❌ 소수점 금액
$invoice->total = 100000.50;

// ✅ 정수 원 단위
$invoice->total = 100000;
```

### 3. 세금 계산은 TaxCalculator를 쓴다

부가세율은 국가/시점에 따라 다르므로 하드코딩 금지.

```php
// ❌ 하드코딩
$tax = $subtotal * 0.1;

// ✅ 계산기 사용
$tax = app(TaxCalculator::class)->calculate(
    amount: $subtotal,
    country: $customer->country,
    issuedAt: $invoice->issued_at,
);
```

### 4. 발행된 청구서는 수정 불가

`status`가 `issued` 이상인 청구서는 수정할 수 없다.
수정이 필요하면 취소(`CancelInvoiceAction`) 후 재발행한다.

## 상태 흐름

```
draft → issued → paid
  ↓        ↓
cancelled  cancelled
```

## 자주 하는 실수

- `invoice_number`를 직접 채번하려는 시도 → Action이 처리한다
- 청구 항목 없이 청구서 생성 → 최소 1개 이상 필요
- 취소된 청구서를 다시 issued로 변경 → 불가능. 새로 만들어야 한다
```

### 8-5. SKILL.md 파일 형식

Boost 스킬은 [Agent Skills 포맷](https://agentskills.io/what-are-skills)을 따른다.

```markdown
---
name: package-name-development        # 필수
description: 이 스킬을 언제 쓰는지 설명  # 필수
---

# 스킬 제목

## 언제 이 스킬을 쓰나
[활성화 조건을 명확히]

## 기능
- 기능 1: [설명]
- 기능 2: [설명 + 코드 예제]
```

**frontmatter의 `name`과 `description`은 필수**다. 특히 `description`이 중요한데, AI가 이 설명을 보고 "지금 이 스킬을 켜야 하나?"를 판단하기 때문이다. 애매하게 쓰면 필요할 때 활성화되지 않는다.

```yaml
# ❌ 너무 모호함
description: 청구서 관련 스킬

# ✅ 활성화 조건이 명확함
description: 청구서(Invoice) 생성, 수정, 발행 작업 시 사용. 청구서 번호 채번, 세금 계산, PDF 생성 로직을 포함한다.
```

### 8-6. 기본 스킬 덮어쓰기

가이드라인과 마찬가지로, **같은 이름으로 만들면 내 것이 우선**한다.

```bash
# Boost의 livewire-development 스킬을 우리 방식으로 교체
mkdir -p .ai/skills/livewire-development
vim .ai/skills/livewire-development/SKILL.md

php artisan boost:update   # 내 스킬이 적용됨
```

### 8-7. 커스텀 스킬 반영 명령어 주의

- 커스텀 **가이드라인**: `boost:install` 시 병합
- 커스텀 **스킬**: `boost:update` 시 설치

공식 문서 기준으로 이렇게 나뉘어 있다. 헷갈리면 둘 다 실행해도 무방하다.

---

## 9. Guidelines vs Skills — 언제 뭘 쓰나

### 9-1. 비교표

| 구분 | Guidelines | Skills |
|---|---|---|
| **로드 시점** | 시작할 때 항상 | 관련 작업 시에만 |
| **범위** | 넓고 기초적 | 좁고 작업 특화 |
| **목적** | 핵심 컨벤션, 베스트 프랙티스 | 상세 구현 패턴 |
| **컨텍스트 비용** | 항상 소모 | 필요할 때만 소모 |
| **파일 위치** | `.ai/guidelines/*.md` | `.ai/skills/{이름}/SKILL.md` |

### 9-2. 판단 기준

```
"이 규칙이 모든 작업에 적용되나?"
   ├─ YES → Guidelines
   └─ NO  → Skills


예시:
"비즈니스 로직은 Action 클래스에"        → 모든 작업에 적용 → Guidelines
"우리 프로젝트는 Pest를 쓴다"            → 모든 작업에 적용 → Guidelines
"청구서 세금 계산은 TaxCalculator로"     → 청구서 작업만    → Skills
"Livewire 폼 검증 상세 패턴"            → Livewire 작업만  → Skills
```

### 9-3. 실무 배분 전략

```
.ai/guidelines/
├── project-conventions.md    ← 아키텍처 원칙, 네이밍, 금지사항 (짧게!)
└── stack.md                  ← 우리가 쓰는 스택 명시

.ai/skills/
├── creating-invoices/        ← 청구 도메인
├── tenant-scoping/           ← 멀티테넌시 규칙
├── payment-processing/       ← 결제 도메인
└── legacy-integration/       ← 레거시 시스템 연동
```

**가이드라인은 가능한 짧게** 유지하는 게 좋다. 100줄 넘어가기 시작하면 "이거 스킬로 뺄 수 있나?"를 고민할 시점이다.

---

## 10. Documentation API — 17,000개의 문서 조각

### 10-1. 개념

Boost는 Laravel 생태계 문서 **17,000개 이상의 조각**을 담은 지식 베이스에 접근한다. 임베딩 기반 **시맨틱 검색**을 사용한다.

```
[일반 키워드 검색]
"belongsToMany" 검색 → 정확히 그 단어가 있는 문서만

[시맨틱 검색]
"다대다 관계 어떻게 설정하지?" → belongsToMany 문서를 찾아줌
                              → 의미로 검색
```

### 10-2. 대상 패키지와 버전

| 패키지 | 지원 버전 |
|---|---|
| Laravel Framework | 10.x, 11.x, 12.x, 13.x |
| Filament | 2.x, 3.x, 4.x, 5.x |
| Flux UI | 2.x Free, 2.x Pro |
| Inertia | 1.x, 2.x |
| Livewire | 1.x, 2.x, 3.x, 4.x |
| Nova | 4.x, 5.x |
| Pest | 3.x, 4.x |
| Tailwind CSS | 3.x, 4.x |

**핵심은 "설치된 패키지 버전 기준"으로 필터링된다는 점**이다. Livewire 3을 쓰는데 Livewire 4 문서가 딸려오지 않는다.

### 10-3. 정적 md 파일 대비 장단점

| 항목 | 정적 md (직접 관리) | Documentation API |
|---|---|---|
| 오프라인 사용 | ✅ | ❌ (네트워크 필요) |
| 최신성 | ❌ 수동 갱신 | ✅ 항상 최신 |
| 컨텍스트 비용 | 로드할 때마다 소모 | 질의할 때만 소모 |
| 버전 정합성 | 직접 관리 | ✅ 자동 |
| 우리 팀 규칙 | ✅ 담을 수 있음 | ❌ 공식 문서만 |

**둘 다 쓰는 게 정석이다.**
- `.ai/guidelines/`, `.ai/skills/` → 우리 팀만의 컨벤션
- `search-docs` → 정확한 공식 API 시그니처 확인

### 10-4. 프라이버시

> 문서 검색 API는 **검색어와 설치된 패키지 버전 정보**를 Laravel 호스팅 검색 엔드포인트로 보낸다. 애플리케이션 코드 자체는 로컬에 남는다. MCP 서버 본체는 전적으로 로컬에서 동작한다.

보안 정책이 엄격한 환경(금융권 등)이라면 이 외부 통신을 조직 정책에 비추어 확인해 볼 필요가 있다.

---

# 실전편

## 11. 실전 워크플로우 — Boost가 있고 없고의 차이

블로그에 나온 실제 시나리오를 재구성해 본다.

### 요청: "회원을 CSV로 내보내는 기능 추가해줘"

#### Boost 없이

```
AI의 처리 과정:
1. 파일 몇 개 훑어봄
2. User 모델 발견
3. 일반적인 CSV 내보내기 코드 생성

결과물의 문제:
- fputcsv()로 직접 다 구현 (Laravel Excel 설치돼 있는데 안 씀)
- 설치 안 된 패키지를 require (League\Csv 등)
- 이미 존재하는 라우트 이름과 충돌 (users.export)
- 존재하지 않는 컬럼 참조 (user.full_name — 실제론 name)

→ "코드는 돌아가는데 우리 앱에 안 맞음"
```

#### Boost 적용 후

```
AI의 처리 과정:
1. app-info 도구      → "Laravel Excel 3.1 설치돼 있음. Laravel 13."
2. routes 도구        → "users.index, users.show 있음. 네이밍 패턴 파악."
                        "users.export는 아직 없음. 사용 가능."
3. schema 도구        → "users: id, name, email, phone, created_at"
4. search-docs 도구   → "Laravel Excel 3.1의 FromQuery, WithHeadings API 확인"
5. 코드 생성
6. tinker 도구        → "User::count()로 데이터 양 확인 → 12만 건. 청크 필요."

결과물:
- Laravel Excel의 FromQuery 사용 (메모리 안전)
- 라우트 네이밍 컨벤션 준수
- 실제 존재하는 컬럼만 참조
- 대용량 대응 (12만 건 확인했으므로)
```

#### 실제로 생성될 만한 코드

```php
<?php
// app/Exports/UsersExport.php

namespace App\Exports;

use App\Models\User;
use Illuminate\Database\Eloquent\Builder;
use Maatwebsite\Excel\Concerns\FromQuery;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;

class UsersExport implements FromQuery, WithHeadings, WithMapping
{
    /**
     * FromQuery를 쓰는 이유:
     * 전체를 메모리에 올리지 않고 청크 단위로 처리한다.
     * (Boost가 tinker로 12만 건임을 확인했기 때문에 선택)
     */
    public function query(): Builder
    {
        return User::query()->select([
            'id', 'name', 'email', 'phone', 'created_at',
        ]);
    }

    /** CSV 첫 줄에 들어갈 헤더 */
    public function headings(): array
    {
        return ['ID', '이름', '이메일', '연락처', '가입일'];
    }

    /**
     * 각 행을 어떻게 변환할지 정의.
     * 실제 스키마에 있는 컬럼만 사용한다.
     */
    public function map($user): array
    {
        return [
            $user->id,
            $user->name,
            $user->email,
            $user->phone,
            $user->created_at->format('Y-m-d'),
        ];
    }
}
```

```php
<?php
// app/Http/Controllers/UserExportController.php

namespace App\Http\Controllers;

use App\Exports\UsersExport;
use Maatwebsite\Excel\Facades\Excel;
use Symfony\Component\HttpFoundation\BinaryFileResponse;

class UserExportController extends Controller
{
    public function __invoke(): BinaryFileResponse
    {
        // 파일명에 날짜를 넣어 중복을 방지한다
        $filename = sprintf('users-%s.csv', now()->format('Ymd-His'));

        return Excel::download(new UsersExport, $filename);
    }
}
```

```php
<?php
// routes/web.php

// 기존 라우트 네이밍 패턴(users.*)을 따른다.
// Boost가 routes 도구로 users.export가 비어 있음을 확인했다.
Route::get('/users/export', UserExportController::class)
    ->name('users.export')
    ->middleware(['auth', 'can:users.export']);
```

---

## 12. "싹부터 자르기" 전략 — 커스텀 가이드라인 운영법

블로그에서 Jeffrey Way(Laracasts)의 워크플로우로 소개된 방법인데, **Boost 활용법 중 가장 실전적인 팁**이다.

### 핵심 원칙

> AI가 실수했을 때, 그 실수만 고치고 넘어가지 말고 **가이드라인을 갱신하라.**
> 사례가 아니라 **패턴**을 고쳐라.

### 실전 흐름

```
1. AI가 실수함
      ↓
2. "아 이거 아닌데" 하고 수정
      ↓
3. ★ 여기서 멈추지 말고 ★
      ↓
4. .ai/guidelines/project-conventions.md 에 규칙 추가
      ↓
5. 다음부터는 같은 실수 안 함 (영구적으로)
```

### 누적되는 가이드라인의 예

시간이 지나면서 이렇게 자란다.

```markdown
## 프로젝트 컨벤션
<!-- 이 파일은 AI가 실수할 때마다 한 줄씩 추가된다 -->

<!-- 2026-03-15 추가: 컨트롤러에 로직을 넣는 실수 -->
- 비즈니스 로직은 app/Actions/ 아래 Action 클래스에.
  컨트롤러가 20줄을 넘으면 잘못된 신호다.

<!-- 2026-03-20 추가: PHPUnit 문법으로 테스트를 짬 -->
- 테스트는 Pest 문법으로만 작성한다.
  `it('...')`, `test('...')` 사용. `class ... extends TestCase` 금지.

<!-- 2026-04-02 추가: FormRequest를 새로 만듦 -->
- 요청 검증은 Laravel Data DTO를 쓴다. FormRequest 신규 작성 금지.
  기존 FormRequest는 점진적으로 DTO로 이관 중이다.

<!-- 2026-04-10 추가: response()->json() 직접 호출 -->
- API 응답은 App\Support\ApiResponse를 거친다.
  성공: ApiResponse::success($data)
  실패: ApiResponse::error($message, $code)

<!-- 2026-04-18 추가: 큐 이름을 아무렇게나 지음 -->
- 큐 이름 규칙: {도메인}-{동작}
  예: billing-process-payment, notification-send-email

<!-- 2026-05-03 추가: Livewire 3에서 .defer를 씀 -->
- Livewire 3에서 wire:model은 기본이 defer다.
  .defer 수식자는 존재하지 않는다.
  실시간 반응이 필요할 때만 wire:model.live를 쓴다.

<!-- 2026-05-21 추가: N+1 발생 -->
- 목록 조회 시 관계는 반드시 eager loading한다.
  Model::preventLazyLoading()이 로컬에서 켜져 있으므로
  누락 시 예외가 발생한다.
```

### 왜 이게 강력한가

```
[일반적인 방식]
실수 발생 → 수정 → 다음에 또 같은 실수 → 또 수정 → 무한 반복

[싹부터 자르기]
실수 발생 → 수정 + 규칙 추가 → 다시는 그 실수 안 함
                                    ↓
                        가이드라인이 팀의 살아있는 문서가 됨
                                    ↓
                        신규 팀원 온보딩 문서로도 쓸 수 있음 (보너스)
```

### 운영 팁

1. **거창하게 쓰지 마라.** 한 줄이면 충분하다.
2. **왜 안 되는지도 짧게 적어라.** AI가 이유를 알면 유사 상황에도 적용한다.
3. **정기적으로 정리하라.** 가이드라인이 길어지면 Skills로 쪼갠다.
4. **반드시 커밋하라.** `.ai/`는 팀 자산이다.
5. **`boost:update`는 `.ai/`를 건드리지 않는다.** 안심하고 써도 된다.

---

## 13. Herd MCP 서버와 함께 쓰기

> macOS / Windows에서 Laravel Herd를 쓰는 경우에만 해당한다. WSL/Docker 환경이라면 이 섹션은 건너뛰어도 된다.

### 13-1. 역할 분담

```
┌─────────────────────┐   ┌─────────────────────┐
│   Laravel Boost     │   │      Herd MCP       │
│  (애플리케이션 레벨)  │   │   (인프라 레벨)      │
├─────────────────────┤   ├─────────────────────┤
│ • Eloquent 모델     │   │ • 로컬 사이트 URL    │
│ • DB 스키마         │   │ • PHP/Node 버전      │
│ • 라우트            │   │ • 환경 변수          │
│ • Artisan 명령      │   │ • 실행 중인 서비스    │
│ • 로그              │   │   (MySQL/Redis/     │
│ • 공식 문서         │   │    Typesense 등)    │
│                     │   │ • 디버그 세션 캡처   │
└─────────────────────┘   └─────────────────────┘
```

### 13-2. Herd MCP의 킬러 기능: `debug_site`

블로그 저자가 "가장 놀라웠다"고 꼽은 기능이다.

```
나: "사용자 목록 페이지가 느려. 원인 찾아줘."

AI: → Herd의 debug_site 프롬프트 실행
    ← 캡처된 내용:
       • 실행된 쿼리 148개 (!)
       • 그중 145개가 동일 패턴 → N+1 확정
       • dispatch된 잡: 없음
       • dump() 호출: 없음
       • 외부 HTTP 요청: 없음

AI: "N+1 문제입니다. UserController@index에서
     $users->each(fn($u) => $u->profile->name) 부분이
     쿼리를 사용자 수만큼 발생시킵니다.
     User::with('profile')로 eager loading하면 해결됩니다."
```

쿼리 로그를 직접 켜고, 재현하고, 복사해서 붙여넣는 과정이 사라진다.

### 13-3. 설정

**Boost가 Herd를 감지하면 자동으로 설치한다.** 수동으로 하려면:

```json
{
  "mcpServers": {
    "laravel-boost": {
      "command": "php",
      "args": ["artisan", "boost:mcp"]
    },
    "herd": {
      "command": "php",
      "args": ["/Applications/Herd.app/Contents/Resources/herd-mcp.phar"],
      "env": {
        "SITE_PATH": "/path/to/your/project"
      }
    }
  }
}
```

> `SITE_PATH`는 실제 프로젝트 절대 경로로 바꿔야 한다. Windows Herd는 `.phar` 경로가 다르므로 Herd 설치 경로를 확인하자.

### 13-4. Herd를 쓸지 말지에 대한 참고

Herd는 로컬 개발 속도가 빠르고 SSL 설정이 간편한 대신, 운영 환경(Linux 컨테이너)과 환경이 달라진다는 트레이드오프가 있다. Kafka·Elasticsearch 같은 복합 인프라가 필요한 프로젝트라면 Docker Compose 쪽이 맞다.

Boost 자체는 Herd 없이도 완전히 동작한다. Herd MCP는 어디까지나 선택적 보완재다.

---

## 14. 유지보수 — boost:update 자동화

### 14-1. 기본 명령어

```bash
# 이미 퍼블리시된 가이드라인/스킬을 최신 버전으로 갱신
php artisan boost:update
```

Laravel 생태계 패키지가 새 버전을 내면 가이드라인/스킬도 갱신된다. `composer update` 이후, 특히 메이저 버전 업그레이드 후에 실행하는 게 좋다.

### 14-2. 새로 설치한 패키지 발견하기

```bash
# 새로 설치된 패키지를 스캔해서 해당 가이드라인/스킬 설치를 제안
php artisan boost:update --discover
```

`--discover` 없이 실행하면 **이미 있는 것만** 갱신한다. 새 패키지를 추가했다면 이 옵션이 필요하다.

```
boost:update              → 기존 자산 갱신만
boost:update --discover   → 기존 갱신 + 신규 패키지 탐지
```

### 14-3. Composer 스크립트로 자동화 ★

`composer.json`에 등록해서 잊지 않게 만들자.

```json
{
  "scripts": {
    "post-update-cmd": [
      "@php artisan vendor:publish --tag=laravel-assets --ansi --force",
      "@php artisan boost:update --ansi"
    ]
  }
}
```

이제 `composer update`를 실행할 때마다 Boost 자산도 자동 갱신된다.

### 14-4. 팀 운영 규칙 제안

```
1. 패키지를 업데이트한 사람이 boost:update도 실행한다
2. 실행 후 AI 에이전트가 여전히 정상 동작하는지 간단히 확인한다
3. .ai/ 아래 커스텀 자산은 update가 건드리지 않으므로 그대로 유지된다
4. 새 패키지 도입 시에는 --discover 옵션을 잊지 않는다
```

---

## 15. Boost 확장하기 — 커스텀 에이전트 만들기

내가 쓰는 AI 도구가 Boost 기본 지원 목록에 없다면 직접 만들 수 있다.

### 15-1. 에이전트 클래스 작성

```php
<?php

declare(strict_types=1);

namespace App;

use Laravel\Boost\Contracts\SupportsGuidelines;
use Laravel\Boost\Contracts\SupportsMcp;
use Laravel\Boost\Contracts\SupportsSkills;
use Laravel\Boost\Install\Agents\Agent;

/**
 * 커스텀 AI 에이전트 통합 클래스.
 *
 * 필요한 기능에 해당하는 계약(contract)만 골라서 구현하면 된다.
 *   - SupportsGuidelines : AI 가이드라인 파일 생성 지원
 *   - SupportsMcp        : MCP 서버 등록 지원
 *   - SupportsSkills     : Agent Skills 설치 지원
 *
 * 셋 다 필요 없으면 필요한 것만 implements 하면 된다.
 */
class CustomAgent extends Agent implements
    SupportsGuidelines,
    SupportsMcp,
    SupportsSkills
{
    // 각 계약이 요구하는 메서드를 구현한다.
    // 실제 구현 예시는 Boost 저장소의 ClaudeCode.php를 참고하면 된다:
    // https://github.com/laravel/boost/blob/main/src/Install/Agents/ClaudeCode.php
}
```

### 15-2. 에이전트 등록

```php
<?php
// app/Providers/AppServiceProvider.php

namespace App\Providers;

use App\CustomAgent;
use Illuminate\Support\ServiceProvider;
use Laravel\Boost\Boost;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // 첫 번째 인자는 boost:install 선택지에 표시될 식별자
        Boost::registerAgent('customagent', CustomAgent::class);
    }
}
```

등록하면 `php artisan boost:install` 실행 시 선택 목록에 나타난다.

### 15-3. 언제 필요한가

솔직히 대부분의 개발자에게는 필요 없는 기능이다. 다음 경우에만 고려하자.

- 사내 자체 개발 AI 코딩 도구를 쓰는 경우
- Boost가 아직 지원하지 않는 신규 에디터를 쓰는 경우
- OSS 기여로 새 에이전트 지원을 추가하려는 경우

---

## 16. 트러블슈팅 & FAQ

### Q1. Docker나 WSL 환경에서도 되나?

**된다. 다만 추가 설정이 필요할 수 있다.**

가장 흔한 문제는 **DB 접속**이다.

```
[문제 상황]
.env:  DB_HOST=mysql        ← Docker Compose 서비스명
        ↓
MCP 서버가 컨테이너 밖(호스트)에서 실행됨
        ↓
호스트에서는 'mysql' 호스트명을 해석할 수 없음 → 연결 실패
```

**해결책: MCP 서버를 같은 컨테이너 안에서 실행한다.**

```json
{
  "mcpServers": {
    "laravel-boost": {
      "command": "docker",
      "args": [
        "compose",
        "exec",
        "-T",              // TTY 할당 안 함 (MCP는 stdio 통신이라 필수)
        "app",             // docker-compose.yml의 서비스명
        "php",
        "artisan",
        "boost:mcp"
      ]
    }
  }
}
```

**WSL 환경(Ubuntu on Windows)의 경우:**

```json
{
  "mcpServers": {
    "laravel-boost": {
      "command": "wsl",
      "args": [
        "-d", "Ubuntu",                        // 배포판 이름
        "--cd", "/home/user/projects/myapp",   // 프로젝트 경로
        "php", "artisan", "boost:mcp"
      ]
    }
  }
}
```

> WSL 안에서 실행하는 Claude Code나 PhpStorm이라면 경로가 이미 리눅스 기준이므로 별도 래핑이 필요 없다. Windows 쪽 에디터에서 WSL 안의 프로젝트에 붙을 때만 위 설정이 필요하다.

### Q2. 예전 Laravel 버전에서도 되나?

블로그 작성 시점(2026년 2월) 기준으로 **Laravel 10 / 11 / 12, PHP 8.2 이상**을 지원한다고 안내한다. 현재 공식 문서의 가이드라인 표에는 13.x까지 포함돼 있다.

Laravel 9 이하는 지원하지 않으므로 업그레이드가 선행돼야 한다.

### Q3. 내 코드가 외부 서버로 전송되나?

**MCP 서버는 전적으로 로컬에서 동작한다.**

외부 통신은 딱 하나, **문서 검색 API**뿐이다. 이때 전송되는 것은:
- 검색어
- 설치된 패키지 버전 목록

애플리케이션 코드 자체는 로컬에 남는다.

### Q4. 생성된 파일을 커밋해야 하나?

| 파일 | 커밋 여부 | 이유 |
|---|---|---|
| `.mcp.json` | ❌ gitignore 권장 | 개발자마다 에디터가 다름 |
| `CLAUDE.md` | ❌ gitignore 권장 | 자동 재생성됨 |
| `AGENTS.md` | ❌ gitignore 권장 | 자동 재생성됨 |
| `boost.json` | ❌ gitignore 권장 | 자동 재생성됨 |
| `.claude/skills/` | ❌ gitignore 권장 | 자동 재생성됨 |
| **`.ai/guidelines/`** | **✅ 반드시 커밋** | **팀이 직접 쓴 자산** |
| **`.ai/skills/`** | **✅ 반드시 커밋** | **팀이 직접 쓴 자산** |

### Q5. 그냥 CLAUDE.md 잘 쓰는 것과 뭐가 다른가?

**솔직히 말하면, 잘 관리된 CLAUDE.md만으로도 꽤 멀리 갈 수 있다.**

Boost의 차별점은 **규모와 유지보수**다.

| 항목 | 수동 CLAUDE.md | Boost |
|---|---|---|
| 패키지별 가이드라인 | 직접 작성 | 16개 이상 자동 |
| 버전 인식 | 직접 관리 | 자동 |
| 패키지 업데이트 시 갱신 | 직접 | `boost:update` |
| DB 스키마 반영 | 직접 붙여넣기 | 실시간 조회 |
| 실행 중인 앱과 상호작용 | 불가능 | 15개 도구 |
| 공식 문서 검색 | 직접 복붙 | 시맨틱 검색 |

즉 Boost는 **자동화의 가치**다. 처음 만드는 건 수동으로도 할 수 있지만, 패키지 업데이트를 따라가며 최신 상태를 유지하는 게 진짜 어려운 부분이고 그걸 Boost가 대신한다.

### Q6. 유료인가?

**Laravel Boost 자체는 무료 오픈소스(MIT)다.** GitHub 스타 3,500개 이상, 기여자 117명 규모로 활발히 개발 중이다.

Boost 위에 얹는 서드파티 유료 확장(예: Filament Blueprint)이 별도로 존재하지만, Boost를 쓰는 데 필수는 아니다.

### Q7. 운영 서버에 올라가면 안 되나?

`--dev`로 설치했다면 `composer install --no-dev`를 쓰는 배포 환경에는 포함되지 않는다. 개발 도구이므로 **운영에 올릴 이유가 없다.** 배포 스크립트를 확인해 두자.

### Q8. AI가 DB를 망가뜨릴 위험은?

`Database Query`, `Tinker`, `Artisan Commands` 도구는 실제 실행 권한을 갖는다. 다음을 지키자.

```
1. 로컬 개발 DB에서만 사용 (.env의 DB_HOST 확인)
2. 처음에는 수동 승인 모드로 사용
3. 운영 DB 자격증명이 로컬 .env에 없는지 확인
4. 필요하다면 읽기 전용 DB 유저로 접속하도록 별도 커넥션 구성
```

---

## 17. 도입 체크리스트

### 오늘 당장 (10분)

```
□ composer require laravel/boost --dev
□ php artisan boost:install
□ 에디터에서 MCP 연결 확인 (초록 체크마크 등)
□ .gitignore에 자동 생성 파일 추가
□ AI에게 "이 프로젝트 정보 알려줘" 물어보고 응답 확인
```

**동작 확인 프롬프트 예시:**

```
"이 프로젝트의 Laravel 버전, 설치된 주요 패키지,
 그리고 users 테이블 구조를 알려줘"
```

MCP가 제대로 붙었다면 실제 값을 정확히 답한다. 붙지 않았다면 두루뭉술하게 답하거나 파일을 뒤진다.

### 첫 주 (누적 1시간)

```
□ .ai/guidelines/project-conventions.md 생성
□ AI가 실수할 때마다 규칙 한 줄씩 추가 ("싹부터 자르기")
□ .ai/ 디렉터리를 git에 커밋
□ 팀원들에게 공유하고 각자 boost:install 실행하게 하기
```

### 첫 달

```
□ 가이드라인이 100줄 넘으면 Skills로 분리 검토
□ 도메인별 커스텀 스킬 1~2개 작성
□ composer.json의 post-update-cmd에 boost:update 등록
□ 팀 규칙 정립: 패키지 업데이트 담당자가 boost:update 실행
```

### 지속적으로

```
□ composer update 후 boost:update 실행
□ 새 패키지 추가 시 boost:update --discover
□ 가이드라인/스킬 주기적 정리 (분기 1회 정도)
```

---

## 부록: 명령어 요약

```bash
# 설치
composer require laravel/boost --dev
php artisan boost:install

# 갱신
php artisan boost:update              # 기존 자산 갱신
php artisan boost:update --discover   # 신규 패키지 탐지 포함

# MCP 서버 직접 실행 (수동 등록용)
php artisan boost:mcp

# 에이전트별 수동 MCP 등록
claude mcp add -s local -t stdio laravel-boost php artisan boost:mcp
codex  mcp add laravel-boost -- php "artisan" "boost:mcp"
gemini mcp add -s project -t stdio laravel-boost php artisan boost:mcp
```

## 부록: 디렉터리 구조 요약

```
프로젝트 루트/
│
├── [자동 생성 — gitignore 가능]
│   ├── .mcp.json                    MCP 서버 등록 정보
│   ├── CLAUDE.md                    Claude Code용 가이드라인
│   ├── AGENTS.md                    범용 에이전트 가이드라인
│   ├── boost.json                   Boost 설정
│   └── .claude/skills/              설치된 스킬 (에이전트별 경로)
│
└── [직접 작성 — 반드시 커밋 ★]
    └── .ai/
        ├── guidelines/
        │   ├── project-conventions.md
        │   └── {패키지}/{버전}/{주제}.blade.php   ← 기본 덮어쓰기용
        └── skills/
            └── {스킬이름}/
                └── SKILL.md
```

---

## 참고 링크

- [Laravel Boost 제품 페이지](https://laravel.com/ai/boost)
- [Laravel 13.x 공식 문서 — Boost](https://laravel.com/docs/13.x/boost)
- [Boost GitHub 저장소](https://github.com/laravel/boost)
- [Agent Skills 포맷 명세](https://agentskills.io/what-are-skills)
- [Laravel Skills 디렉터리](https://skills.laravel.cloud/)
- [Hafiz Riaz — Laravel Boost and MCP Servers](https://hafiz.dev/blog/laravel-boost-and-mcp-servers-the-context-your-ai-agent-is-missing)
- [Boost 소개 영상](https://www.youtube.com/watch?v=sUtRcpma8iU)
