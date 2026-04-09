---
layout: post
title:  "모아갤 프로젝트 #3 - 개발 환경 구축과 첫 번째 난관"
description: Flutter 설치부터 첫 앱 실행까지, 예상치 못한 문제들과의 씨름
date:   2026-03-22 00:00:00 +000
categories: MoaGal Flutter Setup Troubleshooting
---

## Flutter 설치의 늪

"간단하게 설치하고 시작하자"는 나의 순진한 생각은 첫 단계부터 산산조각났다.

### 문제 1: PATH 설정 지옥

```bash
$ flutter doctor
zsh: command not found: flutter
```

Homebrew로 설치했는데 왜 안 되는 거야? Claude에게 물어보니:

```bash
# .zshrc에 추가 필요
export PATH="$HOME/flutter/bin:$PATH"
source ~/.zshrc
```

하지만 여전히 안 됐다. 알고 보니 **Homebrew 설치 경로가 달랐다**:

```bash
# M1 Mac의 경우
export PATH="/opt/homebrew/bin:$PATH"
```

> 교훈: M1 Mac은 Homebrew 경로가 다르다. Claude도 이건 몰랐다.

### 문제 2: Android SDK 라이선스

```bash
$ flutter doctor
[!] Android toolchain - develop for Android devices
    ✗ Android licenses not accepted.
```

단순히 라이선스 동의인 줄 알았는데...

```bash
$ flutter doctor --android-licenses
Exception in thread "main" java.lang.NoClassDefFoundError
```

**해결**: Android Studio를 실행하고 SDK Manager에서 수동으로 라이선스 동의.

Claude가 제안한 `sdkmanager --licenses`는 작동하지 않았다. 때로는 GUI가 답이다.

## 드디어 첫 앱 실행

2시간의 삽질 끝에:

```bash
$ flutter create moagal
$ cd moagal
$ flutter run
```

Android Emulator가 실행되면서 샘플 카운터 앱이 떴다!

```
Launching lib/main.dart on sdk gphone64 arm64 in debug mode...
✓ Built build/app/outputs/flutter-apk/app-debug.apk.
Installing build/app/outputs/flutter-apk/app-debug.apk...
Debug service listening on ws://127.0.0.1:xxxxx
Synced 15.3MB

🎉 DONE 🎉
```

**감동의 순간**이었다. 버튼을 누르니 숫자가 올라간다. Hot Reload도 바로바로 작동한다.

> iOS 개발 환경(Xcode, CocoaPods)은 일단 보류. 안드로이드부터 완성하고 나중에 추가하기로 결정.

## 첫 번째 위젯 만들기

Claude와 함께 첫 화면을 만들어보기로 했다.

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(MoaGalApp());
}

class MoaGalApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: '모아갤',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: CalendarScreen(),
    );
  }
}

class CalendarScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('모아갤'),
      ),
      body: Center(
        child: Text('캘린더가 여기에 표시됩니다'),
      ),
    );
  }
}
```

Hot Reload를 누르자마자 화면이 바뀌었다. **이 속도감, 이거다!**

## 삽질하며 배운 것들

1. **Claude는 만능이 아니다**: 환경별 차이(M1 Mac, 버전 등)는 직접 해결해야 한다
2. **공식 문서가 최고**: `flutter doctor -v`는 진짜 의사처럼 정확한 진단을 해준다
3. **에러 메시지를 믿어라**: 에러 메시지 전체를 Claude에게 보여주면 더 정확한 답을 얻는다

## 다음 단계

환경 구축은 끝났다. 이제 본격적으로:
- 캘린더 UI 라이브러리 선정
- 로컬 데이터베이스 연동 (sqflite)
- 일정 생성 화면 만들기

다음 편에서는 실제 캘린더 UI를 구현하고, 로컬 데이터베이스에 일정을 저장하는 과정을 다룬다.

> "설치 3시간, 코딩 10분. 개발자의 일상이다."

---

**시리즈**
1. [모아갤 프로젝트를 시작하며](/posts/MoaGal-01-Why-I-Started/)
2. [기술 스택 선정과 초기 계획](/posts/MoaGal-02-Tech-Stack/)
3. **개발 환경 구축과 첫 번째 난관** (현재 글)
4. 핵심 기능 구현하기
5. 테스트와 배포 준비
