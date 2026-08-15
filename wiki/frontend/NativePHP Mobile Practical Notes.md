---
title: NativePHP Mobile Practical Notes
category: frontend
tags: [frontend, nativephp, mobile]
related: [[NativePHP Mobile Overview]], [[NativePHP SuperNative Architecture]], [[Livewire and NativePHP]]
---

# NativePHP Mobile Practical Notes

심화 주제 지도, 실무 진행 순서 제안, 그리고 프로덕션 도입 전 확인해야 할 주의사항.

## 핵심 개념

### 심화 주제 (Digging Deeper)

공식 문서에 별도 섹션으로 정리된 항목들. 실제 앱을 만들 때 필요해지는 순서대로 참고한다.

| 주제 | 왜 중요한가 |
|---|---|
| **Lifecycle Hooks** | 앱 시작/일시정지/재개 시점 처리 |
| **Data Binding** | 컴포넌트 프로퍼티 ↔ UI 양방향 바인딩 |
| **Reactivity** | 상태 변경 시 UI 갱신 메커니즘 |
| **Theming** | 다크 모드, 플랫폼별 디자인 토큰 |
| **Gestures & Animation** | 스와이프, 드래그, 전환 효과 |
| **Databases** | 기기 내 SQLite — Eloquent 그대로 사용 |
| **Queues** | 기기 내 큐 워커 |
| **Authentication** | 로컬 인증 + 서버 연동 |
| **Security** | 번들에 포함되는 코드/자산 보호 |
| **Deep Links** | 외부에서 앱 특정 화면 진입 |
| **WebSockets** | 실시간 통신 |
| **Push Notifications** | Firebase 기반 |

이 위키에는 아직 각 주제의 상세 페이지가 없다 — 실제로 필요해지면 공식 문서를 소스로 추가 ingest하는 것을 권장한다.

### 실무 진행 순서 제안

처음 시작한다면 이 순서를 권한다.

1. **환경부터 완전히 검증** — `java -version`, `adb devices`, `pod --version`이 모두 통과하는지 확인. 여기서 막히는 시간이 의외로 길다 ([[NativePHP Mobile Environment Setup]]).
2. **Kitchen Sink 데모 먼저 실행** — 공식 데모 앱을 돌려서 환경이 정상인지 확인하고, 어떤 컴포넌트가 있는지 눈으로 본다.
3. **SuperNative 데모 클론**

```bash
git clone https://github.com/nativephp/super-native
cd super-native
composer install
php artisan native:install
php artisan native:run
```

소스를 읽으면서 화면 구성 방식을 익힌 뒤, 하나씩 자기 화면으로 바꿔 나가는 방식이 가장 빠르다.

4. **작은 앱 하나를 끝까지** — 화면 3~4개짜리로 라우팅·상태·네이티브 API 하나(카메라나 위치)·빌드까지 한 사이클을 완주한다.
5. **그다음 실제 프로젝트로** — 이 시점에 데이터베이스, 인증, 푸시 등 심화 주제를 붙인다.

## Laravel 구현

### 참고 링크

| 항목 | URL |
|---|---|
| Mobile v4 공식 문서 | https://nativephp.com/docs/mobile/4/getting-started/introduction |
| Quick Start | https://nativephp.com/docs/mobile/4/getting-started/quick-start |
| Environment Setup | https://nativephp.com/docs/mobile/4/getting-started/environment-setup |
| SuperNative 소개 | https://nativephp.com/docs/mobile/4/architecture/super-native |
| EDGE 컴포넌트 목록 | https://nativephp.com/docs/mobile/4/edge-components/introduction |
| 코어 플러그인 | https://nativephp.com/docs/mobile/4/plugins/introduction |
| GitHub (mobile) | https://github.com/nativephp/mobile-air |
| SuperNative 데모 | https://github.com/nativephp/super-native |
| Packagist | https://packagist.org/packages/nativephp/mobile |
| Discord 커뮤니티 | https://discord.gg/nativephp |
| Bifrost (클라우드 빌드) | https://bifrost.nativephp.com |

## 주의사항 / 안티패턴

**프로덕션 도입 전 반드시 확인**:

- **라이선스**: 초기 버전(v1~)에서는 유료 라이선스 구매가 필수였으나, 현재 Packagist에는 MIT 라이선스로 표기되어 있다. 상용 프로젝트에 적용할 계획이라면 최신 라이선스 조건을 공식 채널에서 한 번 더 확인하는 게 안전하다.
- **PHP 버전 고정**: 모바일 번들 PHP는 특정 버전(최근 기준 8.4대)으로 고정된다. 애플리케이션 코드가 그 버전에서 동작하는지 확인이 필요하다.
- **성숙도**: v1에서 v4까지 1년 남짓한 사이에 아키텍처가 크게 바뀌었다. 빠르게 발전하는 만큼 breaking change 가능성도 있으니, 프로덕션 도입 전에는 Upgrade Guide와 Support Policy를 확인하는 걸 권한다.

## 참고

- [[NativePHP Mobile Overview]] — 전체 개념
- [[NativePHP SuperNative Architecture]] — 성숙도 이슈와 관련된 v4 아키텍처
- [[Livewire and NativePHP]] — Livewire 학습 우선순위와의 연결점
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`
