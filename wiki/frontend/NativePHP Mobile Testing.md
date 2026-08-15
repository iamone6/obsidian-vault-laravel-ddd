---
title: NativePHP Mobile Testing
category: frontend
tags: [frontend, nativephp, mobile, testing]
related: [[Livewire Testing]], [[NativePHP Mobile Routing and Navigation]], [[Domain Testing]]
---

# NativePHP Mobile Testing

v4에 포함된 전용 테스트 API 개요. Pest/PHPUnit 기반이라 기존 Laravel 테스트 작성 방식이 그대로 이어진다.

## 핵심 개념

공식 문서가 다루는 5개 카테고리:

| 영역 | 내용 |
|---|---|
| **Interactions** | 탭, 입력 등 사용자 상호작용 시뮬레이션 |
| **Native Events & the Bridge** | 네이티브 이벤트/브리지 호출 검증 |
| **Navigation & Flows** | 화면 전환 플로우 테스트 |
| **Accessibility** | 접근성 검증 |
| **Advanced** | 고급 시나리오 |

## Laravel 구현

`Navigation & Flows`는 [[NativePHP Mobile Routing and Navigation]]에서 다룬 `$this->navigate()`/`$this->back()`/`$this->replace()` 전환을 검증하는 영역이고, `Native Events & the Bridge`는 [[NativePHP Core Plugins]]의 플러그인 호출(카메라, 생체 인증 등)을 모킹·검증하는 영역이다. Livewire의 `Livewire::test()` 스타일([[Livewire Testing]])과 유사한 흐름으로 설계되어 있어, Livewire 테스트에 익숙하면 학습 비용이 낮다.

## 주의사항 / 안티패턴

- 이 소스에는 구체적인 assertion 메서드 목록까지는 없다 — 실제 코드를 작성할 때는 공식 문서의 Testing 섹션(각 카테고리별 페이지)을 직접 확인해야 한다. 이 페이지는 "어떤 영역이 테스트 가능한지"의 지도로만 활용한다.
- 네이티브 브리지 호출(카메라, 생체 인증 등)은 실기기/시뮬레이터 의존성이 있을 수 있으므로, CI 환경에서 돌리려면 어떤 부분이 모킹 가능한지 먼저 확인해야 한다.

## 참고

- [[Livewire Testing]] — 유사한 설계의 `Livewire::test()` API
- [[NativePHP Mobile Routing and Navigation]] — Navigation & Flows가 검증하는 대상
- [[Domain Testing]] — 이 위키의 일반 Laravel 테스트 전략과의 관계
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`
