# Sources

원본 소스를 이 디렉토리에 추가한다. 소스는 불변(immutable)이며 위키 페이지 생성의 기반이 된다.

## 파일 추가 절차

1. 소스 파일을 이 디렉토리에 추가
2. 파일명 형식: `YYYY-MM-DD_제목.md` 또는 `YYYY-MM-DD_제목.pdf`
3. [[log.md]] 에 `[YYYY-MM-DD] [INGEST] 소스명` 항목 추가
4. 관련 위키 페이지 생성/갱신 (보통 10–15개)
5. [[index.md]] 통계 업데이트

## 소스 유형

- 책 챕터 요약
- 아티클 / 블로그 포스트
- 공식 문서 발췌
- 코드베이스 분석 메모
- 컨퍼런스 발표 노트

## 등록된 소스 목록

- [2026-07-11] `2026-07-11_처음부터 제대로 배우는 라라벨.pdf` — 처음부터 제대로 배우는 라라벨 (맷 스타우퍼, 원제: Laravel: Up and Running), 717쪽. 라라벨 프레임워크 전반(라우팅, Eloquent, 컨테이너, 미들웨어, 인증/인가, 테스트, 큐/이벤트, API 등)을 다루는 종합 레퍼런스.
- [2026-07-11] `2026-07-11_Domain-Driven Design with Laravel.pdf` — Domain-Driven Design with Laravel (Martin Joo), 258쪽. DDD 핵심 개념(VO, DTO, Repository, Custom Query Builder, Action, ViewModel, CQRS, State/Transition, Domain/Application 구조)을 이메일 마케팅 SaaS 예제로 설명하는 라라벨 DDD 실전서. (Light/Dark 버전 중 Light만 등록)
- [2026-07-11] `2026-07-11_Case Study - Portfolio And Dividend Tracker.pdf` — 포트폴리오·배당 추적 앱 사례 연구, 20쪽. Custom Collection, 3rd party API 통합 패턴. (Light 버전만 등록)
- [2026-07-11] `2026-07-11_Testing Complex Features.pdf` — 복잡한 기능(CSV 임포트, 이벤트 기반 자동화, 시간 의존 스케줄링) 테스트 전략, 33쪽. (Light 버전만 등록)
- [2026-07-11] `2026-07-11_Static Analysis And CI-CD Pipelines.pdf` — PHPInsights/Larastan/Deptrac 정적 분석과 Github Actions/Gitlab CI 파이프라인 구성, 18쪽. (Light 버전만 등록)
- [2026-07-11] `2026-07-11_Layered Architectures with Laravel.pdf` — MVC → Invokable Controller → Service → Action → Domain/Module 구조로 이어지는 아키텍처 진화 에세이 (Martin Joo), 28쪽.
- [2026-07-11] `2026-07-11_Laravel Eloquent Recipes.pdf` — Eloquent 실전 팁 35가지 (관계, N+1, 팩토리, 마이그레이션) (Martin Joo), 34쪽.
- [2026-07-11] `2026-07-11_Proper API Design With Laravel.pdf` — JSON API 명세, Spatie Query Builder, API 버전 관리, TDD (Martin Joo), 24쪽.
- [2026-07-27] `2026-07-27_Laravel 13.22 Eloquent Model Attributes.md` — Laravel 13.22 기준 `Illuminate\Database\Eloquent\Attributes\*` 24개 총정리. 도입 버전, 적용 조건(병합형/기본값형), `resolveClassAttribute()` 내부 동작, trait/상속 탐색 지원 여부를 프레임워크 소스 기반으로 정리.
- [2026-07-29] `2026-07-29_Laravel Bind and BindWhen Attributes.md` — `Illuminate\Container\Attributes\Bind`/`BindWhen` 정리. 서비스 프로바이더 대신 인터페이스 선언부에서 구현체를 선택하는 방법, 환경별/조건별 바인딩, `#[Singleton]`/`#[Scoped]` 조합. `#[BindWhen]`은 Laravel 13.22(PHP 8.5+) 도입.
- [2026-07-29] `2026-07-29_Laravel MCP.md` — 공식 `laravel/mcp` 패키지 정리. Server/Tool/Resource/Prompt 구조, 의존성 주입, 조건부 도구 등록, `routes/ai.php`의 Web/Local 서버 등록, MCP Inspector를 통한 테스트.
- [2026-07-29] `2026-07-29_Laravel MCP Official Docs Reference.md` — Laravel 공식 블로그 발표문 + 공식 문서(12.x, API는 13.x와 동일) 전체를 종합한 개발 레퍼런스. 출력 스키마/구조화 응답/스트리밍 응답, Resource Template, `_meta` 메타데이터, OAuth(Passport)/Sanctum 인증, 유닛 테스트 어설션 등 기존 소스에 없던 세부사항을 포함. Claude Code가 코딩 중 바로 참고하도록 체크리스트 형식으로 구성.
- [2026-07-29] `2026-07-29_Laravel Boost 완전정복.md` — Laravel Boost(AI 코딩 에이전트에 프로젝트 컨텍스트를 제공하는 공식 MCP 도구) 입문~실전 가이드. 3기둥(AI Guidelines/Agent Skills/MCP Server), 설치, MCP 도구 15종, 커스텀 가이드라인·스킬 작성법, Documentation API, "싹부터 자르기" 운영 전략, Herd MCP, boost:update 자동화, 트러블슈팅/FAQ를 포함.
- [2026-07-30] `2026-07-30_Spatie Laravel Data 공식 문서 종합.md` — GitHub 소스(`docs/`) + 공식 문서 v4 + byzz 블로그 종합. `Data` 클래스 기본 사용법, 중첩/컬렉션/Optional/이름 매핑, 자동 규칙 추론과 91개 Validation Attribute 목록, `rules()` 수동 정의와 병합, 라우트 파라미터·인증 사용자·DB 제약 참조 등 고급 Attribute, 검증 실패 시 `ValidationException` 처리(자동 동작, `messages()`/`attributes()`/`withValidator()` 커스터마이징, 전역 핸들러 통합), FormRequest → Data 마이그레이션 비교표.
