---
layout: post
title:  "모아갤 프로젝트 #2 - 기술 스택 선정과 초기 계획"
description: Flutter를 선택한 이유와 프로젝트 아키텍처 설계
date:   2026-03-15 00:00:00 +000
categories: MoaGal Flutter Architecture TechStack
---

## 기술 스택 선정

앱 개발 경험이 거의 없는 상태에서 가장 먼저 고민한 것은 **"어떤 기술로 만들 것인가?"**였다.

### Flutter vs React Native vs Native

Claude와 함께 각 선택지를 검토했다:

**Flutter**
- ✅ 하나의 코드베이스로 iOS/Android 동시 지원
- ✅ 성능이 우수하고 UI가 네이티브에 가까움
- ✅ Hot Reload로 빠른 개발 가능
- ❌ Dart라는 새로운 언어 학습 필요

**React Native**
- ✅ JavaScript/TypeScript 활용 가능
- ✅ 큰 커뮤니티와 풍부한 라이브러리
- ❌ 네이티브 브릿지로 인한 성능 이슈 가능성

**Native (Swift/Kotlin)**
- ✅ 최고의 성능과 네이티브 기능 활용
- ❌ iOS/Android 각각 개발 필요
- ❌ 학습 곡선이 가파름

### 결정: Flutter

최종적으로 **Flutter**를 선택했다. 이유는:

1. **빠른 프로토타이핑**: Hot Reload로 즉각적인 피드백
2. **크로스 플랫폼**: 양쪽 플랫폼을 동시에 커버
3. **Claude의 추천**: Dart는 직관적이고, Flutter는 문서화가 잘 되어 있음

## 프로젝트 구조 설계

```
moagal/
├── lib/
│   ├── main.dart
│   ├── models/          # 데이터 모델
│   ├── screens/         # 화면 UI
│   ├── widgets/         # 재사용 가능한 위젯
│   ├── services/        # API 통신, 데이터 처리
│   └── providers/       # 상태 관리 (Provider 패턴)
├── android/
├── ios/
└── test/
```

백엔드 개발자라면 익숙한 MVC 패턴과 유사하게 구조를 잡았다.

## 핵심 기능 명세

### Phase 1: MVP (Minimum Viable Product)
- 로컬 일정 생성/수정/삭제
- 주간/월간 캘린더 뷰
- 홈 화면 위젯 (안드로이드)

### Phase 2: 연동 및 확장
- Google Calendar 연동
- 위젯 지원
- 알림 기능

### Phase 3: 고도화
- 다크 모드
- 일정 검색
- 통계 및 리포트
- 백업 과 복원

## 개발 환경 준비

Claude와 함께 준비한 개발 환경:

```bash
# Flutter 설치
brew install flutter

# Android Studio 설치
# - Android SDK
# - Android Emulator

# VS Code Extensions
# - Flutter
# - Dart
```

## 다음 단계

다음 편에서는 실제 Flutter 프로젝트를 생성하고, 첫 번째 화면을 만들면서 겪은 시행착오를 다룰 예정이다.

특히:
- Flutter 설치 과정에서의 삽질
- 첫 번째 위젯 만들기
- 에뮬레이터 설정 문제

> "계획은 완벽했지만, 실전은 항상 다르다."

---

**시리즈**
1. [모아갤 프로젝트를 시작하며](/posts/MoaGal-01-Why-I-Started/)
2. **기술 스택 선정과 초기 계획** (현재 글)
3. 개발 환경 구축과 첫 번째 난관
4. 핵심 기능 구현하기
5. 테스트와 배포 준비