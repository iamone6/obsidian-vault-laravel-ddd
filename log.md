---
title: Log — Laravel Wiki
type: meta
---

# Log

변경 이력. append-only. 최신 항목이 위에 위치한다.

형식: `[YYYY-MM-DD] [ACTION] 설명`

ACTION 종류:
- `INIT` — 초기 구성
- `ADD` — 새 페이지 추가
- `UPDATE` — 기존 페이지 갱신
- `INGEST` — 새 소스 처리
- `LINT` — 정리/일관성 수정

---

[2026-08-12] [INGEST] Livewire 완전 학습 가이드 (v4 기준) — 생성: Livewire Overview, Livewire Render Cycle, Livewire Installation and Components, Livewire Properties, Livewire Actions, Livewire Forms and Validation, Livewire Lifecycle Hooks, Livewire Computed Properties, Livewire Rendering and wire-key, Livewire Loading States, Livewire Events, Livewire Nested Components and Props, Livewire Pagination and File Uploads, Livewire URL and Navigation, Livewire Alpine Integration, Livewire Advanced v4 Features, Livewire Testing, Livewire Common Pitfalls, Livewire and NativePHP (전부 `wiki/frontend/` 신규, 19개)

[2026-08-12] [INGEST] Laravel Blade 상호작용 가이드 — 생성: Blade Template Inheritance, Blade Includes and Loops, Blade Conditionals and Environment, Blade Component Basics, Blade Component Attributes, Blade Style Directives (전부 `wiki/frontend/` 신규) / 갱신: Tailwind Component Strategy(Blade 컴포넌트 세부 페이지로 교차 링크 추가)

[2026-08-03] [ADD] `frontend` 카테고리 신설 — schema.md/AGENTS.md의 category enum 및 디렉토리 구조에 `wiki/frontend/`(Tailwind, NativePHP 레이아웃 등 Laravel 프로젝트의 프론트엔드 스택) 추가

[2026-08-03] [INGEST] Tailwind CSS 실전 가이드 (v4.3 기준) — 생성: Tailwind CSS(허브), Tailwind Installation, Tailwind Utility Syntax, Tailwind Layout, Tailwind Variants, Tailwind Component Strategy, Tailwind Dynamic Class Pitfall, Tailwind Theme Configuration, Tailwind Dark Mode, Tailwind Accessibility, Tailwind Build and Performance, Tailwind v3 to v4 Migration (전부 `wiki/frontend/` 신규)

[2026-07-30] [INGEST] Spatie Laravel Data 공식 문서 종합 (GitHub 소스 + 공식 문서 v4 + byzz 블로그) — 생성: Laravel Data(Attribute 기반 검증, FormRequest 대체, ValidationException 예외 처리·커스터마이징) / 갱신: DTO(Attribute 검증 문법 교차링크), Form Request(`rules()` 대체 및 authorize 미지원 명시, 조합 예제), Policy and Gate(Data 클래스가 인가를 대체하지 못함을 명시)

[2026-07-29] [UPDATE] 질의 응답 중 Laravel MCP 페이지에 "Laravel Boost와 함께 쓰기" 섹션 추가 — 같은 프로젝트에 병행 설치하는 실전 관계 명시

[2026-07-29] [UPDATE] vault 이름을 laravel-ddd → laravel로 변경 요청에 따라 내부 표기 정리 — index.md/log.md/schema.md의 title을 "Laravel DDD Wiki" → "Laravel Wiki"로, AGENTS.md/schema.md의 디렉토리 트리 예시 루트명을 laravel-ddd/ → laravel/로 갱신. 폴더 자체의 물리적 이름 변경은 이 세션의 claude.exe 프로세스가 해당 폴더를 작업 디렉터리로 점유 중이라("Device or resource busy") 세션 종료 후 별도로 처리 필요.
[2026-07-29] [INGEST] Laravel Boost 완전 정복 (laravel.com/ai/boost, 13.x 공식 문서, Hafiz Riaz 블로그 종합) — 생성: wiki/tooling/Laravel Boost(3기둥, MCP 도구 목록, AI Guidelines/Agent Skills 작성법, Documentation API, "싹부터 자르기" 전략, Herd MCP 관계, boost:update, 주의사항) / 갱신: Laravel MCP(Laravel Boost와의 방향 차이 명시 — 업무 기능 노출 툴킷 vs 프로젝트 컨텍스트 제공 완제품)
[2026-07-29] [LINT] 1개 항목 수정 — contradictions: 1 (Third-Party Service Integration의 MarketStackServiceProvider 바인딩을 boot()에서 register()로 이동, Service Provider 컨벤션과 일치시킴 + 이유 설명 추가), broken links: 0, orphans: 0, frontmatter: 0
[2026-07-29] [INGEST] Laravel MCP Official Docs Reference (공식 블로그 + 12.x/13.x 문서 종합) — 갱신: Laravel MCP(출력 스키마/구조화·스트리밍 응답, Resource Template, `_meta` 메타데이터, 인증(Sanctum/Passport OAuth 2.1)과 인가, 유닛 테스트 어설션 전체, 개발 체크리스트 섹션 추가; related에 Policy and Gate 추가)
[2026-07-29] [INGEST] Laravel #[Bind]/#[BindWhen] 어트리뷰트 정리 / Laravel MCP 정리 — 생성: Container Binding Attributes, Laravel MCP / 갱신: Container(#[Bind]/#[BindWhen] 선언 방식), Service Provider(순수 바인딩 프로바이더의 attribute 대체), Repository(#[Bind]로 대체 가능한 바인딩 예시), API Design(Laravel MCP를 대응 계층으로 교차링크), Form Request(MCP Tool 검증과의 유사 패턴 교차링크)
[2026-07-27] [INGEST] Laravel 13.22 기준 Eloquent Model Attribute 총정리 — 생성: Eloquent Model Attributes / 갱신: Eloquent Recipes(Casts\Attribute와의 이름 혼동 주의), Eloquent and DDD(선언부 설정과 얇은 모델 원칙), Custom Query Builder(#[UseEloquentBuilder]), Custom Collection(#[CollectedBy]), Policy and Gate(#[UsePolicy]), Laravel Events(#[ObservedBy]/#[ScopedBy]), API Design(#[UseResource]/#[UseResourceCollection])
[2026-07-11] [UPDATE] 질의 응답("lca-api 프로젝트가 위키의 DDD와 부합하는가") 중 Design Philosophy 페이지에 "사례 — 얕은 DDD (참조 데이터/계산 중심 도메인)" 섹션 추가 — Repository/VO/Aggregate 생략은 원칙 2에 부합하되, 정적/인스턴스 혼용·Action 단일책임 위반·Controller 비대화는 패턴 깊이와 무관한 일관성 붕괴임을 명시
[2026-07-11] [INGEST] Layered Architectures with Laravel / Laravel Eloquent Recipes / Proper API Design With Laravel (Martin Joo) — 생성: Eloquent Recipes, API Design, Test-Driven Development / 갱신: Action Pattern(교차 모델 쿼리 딜레마, 언제 안 쓰는지, 순환 의존), Directory Structure(composer psr-4 오토로드), Repository Implementation(추가 관계 레시피 링크)
[2026-07-11] [UPDATE] 질의 응답("아그리게이트 구성 고민") 중 Aggregate 페이지에 "설계 체크리스트 — 경계를 고민할 때 순서대로 물어볼 4가지 질문" 섹션 추가
[2026-07-11] [ADD] 질의 응답("laravel ddd 개발에 있어 가장 중요한 것") 중 Design Philosophy 페이지 생성 — Ubiquitous Language, Repository, Action Pattern, DTO, Bounded Context, Directory Structure, Layered Architecture, Eloquent and DDD에 역링크 추가
[2026-07-11] [LINT] 10개 항목 수정 — broken links: 7 (Domain Service, Hexagonal Architecture, Module Organization, Integration Testing, Factory, Specification, Event Sourcing 관련 링크를 언링크하고 index.md에서 실체 없는 7개 항목 제거), orphans: 3 (State Pattern, Testing Complex Features, Third-Party Service Integration에 관련 페이지 역링크 추가), frontmatter: 0
[2026-07-11] [INGEST] Domain-Driven Design with Laravel 패키지 4종(Book/Case Study/Testing Complex Features/Static Analysis And CI-CD) (Martin Joo) — 생성: Custom Query Builder, ViewModel, State Pattern, Testing Complex Features, Custom Collection, Third-Party Service Integration, Static Analysis, CI-CD Pipeline / 갱신: Directory Structure, Repository, Action Pattern, CQRS, DTO, Value Object / wiki/tooling/ 카테고리 신설
[2026-07-11] [INGEST] 처음부터 제대로 배우는 라라벨(맷 스타우퍼) — 생성: Service Provider, Container, Form Request, Pipeline Pattern, Laravel Events, Policy and Gate, Unit of Work, Feature Testing, Repository Implementation / 갱신: Eloquent and DDD
[2026-07-11] [INIT] 위키 초기 구성. schema.md, index.md, log.md 생성.
[2026-07-11] [ADD] wiki/ddd-core/ — Bounded Context, Entity, Value Object, Aggregate, Domain Event, Repository, Domain Service, Application Service, Ubiquitous Language
[2026-07-11] [ADD] wiki/laravel/ — Directory Structure, Eloquent and DDD, Service Provider, Action Pattern, DTO
[2026-07-11] [ADD] wiki/architecture/ — Layered Architecture, Hexagonal Architecture, CQRS, Module Organization
[2026-07-11] [ADD] wiki/testing/ — Domain Testing
