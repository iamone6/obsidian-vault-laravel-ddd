# Lint

위키의 일관성과 품질을 점검하고 문제를 수정한다.

## 사용법

```
/lint [--fix]
```

- 인자 없음: 문제만 보고 (수정 안 함)
- `--fix`: 자동 수정 가능한 항목을 즉시 수정

## 실행 절차

**1단계 — 전체 파일 수집**

`wiki/` 디렉토리의 모든 `.md` 파일 목록을 수집한다.

**2단계 — 항목별 점검**

아래 항목을 순서대로 점검한다.

### A. 끊어진 링크 (Broken Links)
각 페이지의 `[[링크]]`를 추출하여 해당 파일이 실제로 존재하는지 확인한다.
- 문제: `[[존재하지않는페이지]]`
- 수정: 링크 제거 또는 빈 페이지 생성 제안

### B. 고아 페이지 (Orphan Pages)
`index.md`에 등록되지 않았거나, 다른 어떤 페이지에서도 링크되지 않는 페이지를 찾는다.
- 문제: 어디서도 참조되지 않는 페이지
- 수정: `index.md`에 추가하거나 관련 페이지에서 링크 추가

### C. 누락된 Frontmatter
`title`, `category`, `tags`, `related` 중 누락된 필드를 찾는다.
- 수정 (`--fix`): 파일 내용 기반으로 자동 추론하여 추가

### D. 모순된 내용 (Contradictions)
같은 주제를 다루는 페이지들 간에 서로 다른 설명이 있는지 확인한다.
예: 한 페이지는 "Repository는 Entity를 반환해야 한다"고 하고, 다른 페이지는 Eloquent 모델을 반환하는 예제를 정답처럼 제시.
- 수정: 모순을 발견하면 어느 쪽이 올바른지 사용자에게 확인 후 수정

### E. index.md 누락 항목
`wiki/` 디렉토리에 실제로 존재하는 파일이 `index.md`에 없는 경우를 찾는다.
- 수정 (`--fix`): 적절한 카테고리에 자동 추가

### F. 오래된 날짜 / 통계
`index.md`의 "최종 업데이트" 날짜와 페이지 수가 실제와 다른지 확인한다.
- 수정 (`--fix`): 현재 날짜와 실제 파일 수로 갱신

**3단계 — 점검 결과 보고**

문제를 심각도별로 분류하여 보고한다:

```
## Lint 결과 — YYYY-MM-DD

### 오류 (즉시 수정 필요)
- [Broken Link] wiki/ddd-core/Entity.md → [[존재하지않는페이지]]

### 경고 (개선 권장)
- [Orphan] wiki/patterns/SomePattern.md — 어디서도 참조되지 않음
- [Contradiction] Value Object vs Eloquent and DDD — save() 동작 설명 불일치

### 정보 (자동 수정 가능)
- [Frontmatter] wiki/laravel/ServiceProvider.md — tags 필드 누락
- [Index] wiki/laravel/ServiceProvider.md — index.md에 미등록
- [Stats] index.md 통계 업데이트 필요 (현재: 24, 실제: 26)

총 오류: 1 / 경고: 2 / 정보: 3
```

**4단계 — 수정 적용 (`--fix` 시)**

자동 수정 후 `log.md`에 기록한다:
```
[YYYY-MM-DD] [LINT] N개 항목 수정 — broken links: X, orphans: Y, frontmatter: Z
```
