---
title: Tailwind Build and Performance
category: frontend
tags: [frontend, tailwind, build, performance, deployment]
related: [[Tailwind Installation]], [[CI-CD Pipeline]], [[Tailwind CSS]]
---

# Tailwind Build and Performance

빌드 흐름, 배포 체크리스트, 실제 성능 특성.

## 핵심 개념

### 빌드 흐름

```bash
npm run dev     # 개발: HMR, 요청된 클래스를 즉시 생성
npm run build   # 배포: 사용된 클래스만 포함한 최소 CSS + 해시 파일명
```

Laravel의 `@vite()` 디렉티브는 개발 모드에서는 dev 서버를, 프로덕션에서는 `public/build/manifest.json`을 읽어 해시된 파일을 참조한다. 캐시 무효화가 자동 처리된다.

### 성능 관련 사실

- 최종 CSS는 대부분 **10~50KB(gzip 후 그보다 작음)** 수준이다. 페이지 수가 늘어도 클래스 종류가 늘지 않으면 크기가 거의 증가하지 않는다 — 전통적 CSS와 가장 큰 차이.
- v4 엔진(Oxide, Rust 기반)은 전체 빌드가 수십 ms, 증분 빌드는 마이크로초 단위다.
- CSS는 렌더링 차단 리소스이므로 `<head>`에 두는 게 맞다. Tailwind 결과물은 작아서 대부분의 프로젝트에서 critical CSS 분리는 불필요하다.
- 웹폰트가 CSS보다 큰 경우가 많다. 성능 최적화는 폰트 서브셋과 `font-display: swap`부터 보는 편이 효과적이다.

## Laravel 구현

배포 스크립트/CI 체크리스트:

```bash
npm ci
npm run build        # ← 이 단계를 빠뜨리면 스타일 없는 페이지가 배포된다
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

- `public/build/`를 저장소에 커밋할지는 팀 규칙으로 정한다. 커밋하지 않는 쪽이 일반적이며, 이 경우 **배포 서버에 Node가 있거나 CI에서 빌드 후 산출물을 전송**해야 한다.
- `php artisan view:cache`를 쓴다면 `storage/framework/views/*.php`도 `@source`에 넣어두는 것이 안전하다 (자세한 건 [[Tailwind Installation]]).
- 무중단 배포 시 이전 버전 사용자가 구 CSS를 참조할 수 있으므로, 이전 빌드 산출물을 일정 기간 남겨두는 전략을 고려한다.

## 주의사항 / 안티패턴

- 브라우저 런타임 빌드(`@tailwindcss/browser`)를 프로덕션에 사용하지 말 것.
- 빌드 없이 `node_modules`의 CSS를 직접 링크하지 말 것.
- 생성된 CSS 파일을 손으로 수정하지 말 것.
- `npm run build`를 CI/배포 스크립트에서 빠뜨리면 스타일 없는 페이지가 배포된다 — 가장 흔한 배포 실수.

## 참고

- [[Tailwind Installation]] — 초기 설정과 `@source` 스캔 경로
- [[CI-CD Pipeline]] — Laravel 프로젝트 전체 배포 파이프라인
- [[Tailwind CSS]] — 개요
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`
