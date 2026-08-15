---
title: NativePHP Mobile Deployment
category: frontend
tags: [frontend, nativephp, mobile, deployment]
related: [[NativePHP Mobile Environment Setup]], [[NativePHP Mobile Installation]], [[CI-CD Pipeline]]
---

# NativePHP Mobile Deployment

Android/iOS 스토어 배포를 위한 최소 절차와 버전 관리 환경 변수.

## 핵심 개념

### Android

- Google Play Console 등록
- 서명 키(keystore) 생성 및 설정
- AAB 빌드 및 업로드

### iOS

- Apple Developer Program 가입 필수 ($99/년) — [[NativePHP Mobile Environment Setup]]에서 다룬 실기기 테스트 요구사항과 동일한 계정
- 프로비저닝 프로파일 / 인증서 설정
- App Store Connect 업로드

## Laravel 구현

### 버전 관리 env

```dotenv
NATIVEPHP_APP_VERSION="1.0.0"
NATIVEPHP_APP_VERSION_CODE="1"
```

버전 문자열(사용자에게 보이는 버전)과 빌드 코드(스토어 내부 증가값)를 분리해서 관리한다.

## 주의사항 / 안티패턴

- 이 소스는 배포 절차의 개요만 제공한다 — keystore 생성, 프로비저닝 프로파일 발급 같은 세부 단계는 공식 문서의 `Publishing Your App` 섹션을 플랫폼별로 직접 따라야 한다.
- iOS 배포는 [[NativePHP Mobile Environment Setup]]에서 다룬 Mac 필수 제약이 그대로 적용된다 — 빌드 자체도 Mac에서만 가능하다.
- keystore/인증서 파일은 분실하면 이후 같은 앱의 업데이트를 배포할 수 없게 되는 등 되돌리기 어려운 문제로 이어진다 — 안전한 백업이 필수다.

## 참고

- [[NativePHP Mobile Environment Setup]] — Apple Developer Program이 필요해지는 조건
- [[NativePHP Mobile Installation]] — `NATIVEPHP_APP_ID` 등 초기 설정과의 연결
- [[CI-CD Pipeline]] — 이 위키의 일반 Laravel 배포 파이프라인과의 비교
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`
