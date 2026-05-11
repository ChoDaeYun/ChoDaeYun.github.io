---
layout: post
title:  "모아갤 프로젝트 #5 - 테스트와 배포 준비"
description: 위젯 구현, 실제 기기 테스트, 그리고 앱스토어 배포를 향한 마지막 단계
date:   2026-04-05 00:00:00 +000
categories: MoaGal Flutter Widget Testing Deployment
---

## 안드로이드 위젯 구현

모아갤의 핵심 기능 중 하나는 **홈 화면 위젯**이다. 앱을 열지 않고도 오늘의 일정을 한눈에 볼 수 있어야 한다.

### home_widget 패키지

Flutter에서 네이티브 위젯을 만들려면 플랫폼별 코드가 필요하다. `home_widget` 패키지를 사용:

```yaml
dependencies:
  home_widget: ^0.3.0
```

### Kotlin으로 위젯 구현

`android/app/src/main/kotlin/.../CalendarWidget.kt`:

```kotlin
class CalendarWidget : AppWidgetProvider() {
    override fun onUpdate(
        context: Context,
        appWidgetManager: AppWidgetManager,
        appWidgetIds: IntArray
    ) {
        appWidgetIds.forEach { widgetId ->
            val views = RemoteViews(
                context.packageName,
                R.layout.calendar_widget
            )

            // SharedPreferences에서 오늘 일정 가져오기
            val prefs = context.getSharedPreferences("HomeWidget", Context.MODE_PRIVATE)
            val todayEvents = prefs.getString("today_events", "일정 없음")

            views.setTextViewText(R.id.widget_events, todayEvents)

            // 앱 실행 인텐트
            val intent = Intent(context, MainActivity::class.java)
            val pendingIntent = PendingIntent.getActivity(context, 0, intent, 0)
            views.setOnClickPendingIntent(R.id.widget_container, pendingIntent)

            appWidgetManager.updateAppWidget(widgetId, views)
        }
    }
}
```

### Flutter에서 위젯 데이터 업데이트

```dart
import 'package:home_widget/home_widget.dart';

Future<void> updateWidget() async {
  final today = DateTime.now();
  final events = await EventService().getEventsByDate(today);

  final eventText = events.isEmpty
      ? '오늘 일정이 없습니다'
      : events.map((e) => '${e.title} (${DateFormat('HH:mm').format(e.startTime)})').join('\n');

  await HomeWidget.saveWidgetData<String>('today_events', eventText);
  await HomeWidget.updateWidget(
    androidName: 'CalendarWidget',
  );
}
```

### 삽질 포인트: 위젯이 업데이트 안 됨

위젯을 추가해도 데이터가 표시되지 않았다. 원인은 **권한 문제**:

```xml
<!-- AndroidManifest.xml -->
<receiver android:name=".CalendarWidget"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/calendar_widget_info" />
</receiver>
```

`android:exported="true"`를 추가하니 해결됐다.

## 실제 기기 테스트

에뮬레이터에서는 잘 돌아가지만, 실제 기기에서는 어떨까?

### Galaxy S23에서 테스트

```bash
$ flutter run --release
```

**발견한 버그들:**

1. **위젯 크기 문제**: 갤럭시에서는 위젯이 너무 작게 표시됨
   - 해결: `minWidth`, `minHeight`를 dp 단위로 조정

2. **다크 모드 대응**: 다크 모드에서 텍스트가 안 보임
   - 해결: 시스템 테마 감지하여 색상 자동 변경

3. **성능 이슈**: 일정이 100개 이상일 때 스크롤이 버벅임
   - 해결: `ListView.builder` 사용 + 페이지네이션

## 앱 아이콘과 스플래시 화면

### 아이콘 생성

Figma로 간단한 아이콘 디자인 후 `flutter_launcher_icons` 사용:

```yaml
dev_dependencies:
  flutter_launcher_icons: ^0.13.1

flutter_launcher_icons:
  android: true
  ios: true
  image_path: "assets/icon/app_icon.png"
  adaptive_icon_background: "#FFFFFF"
  adaptive_icon_foreground: "assets/icon/foreground.png"
```

```bash
$ flutter pub run flutter_launcher_icons
```

### 스플래시 화면

`flutter_native_splash` 패키지 사용:

```yaml
dev_dependencies:
  flutter_native_splash: ^2.3.5

flutter_native_splash:
  color: "#FFFFFF"
  image: assets/splash/splash_logo.png
  android: true
  ios: true
```

```bash
$ flutter pub run flutter_native_splash:create
```

## 앱 서명 및 빌드

### Android 키스토어 생성

```bash
$ keytool -genkey -v -keystore ~/moagal-key.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias moagal
```

`android/key.properties`:
```
storePassword=<password>
keyPassword=<password>
keyAlias=moagal
storeFile=/Users/dycho/moagal-key.jks
```

### Release 빌드

```bash
$ flutter build apk --release
$ flutter build appbundle --release
```

생성된 APK 크기: **18.2 MB** (생각보다 작다!)

## iOS 빌드 (예정)

iOS는 Apple Developer Program 가입($99/년)이 필요해서 일단 보류. 안드로이드 배포 후 반응을 보고 결정할 예정.

## 배포 전 체크리스트

- [x] 모든 기능 테스트 완료
- [x] 실제 기기에서 정상 동작 확인
- [x] 앱 아이콘 및 스플래시 생성
- [x] 개인정보 처리방침 작성
- [x] Google Play Console 계정 생성
- [ ] 앱 스크린샷 준비 (5장)
- [ ] 앱 설명 작성 (한글/영문)
- [ ] Play Store 업로드

## 회고: Claude와 함께한 한 달

Claude Pro를 결제하고 정확히 한 달. 이 여정을 돌아보면:

### 잘한 점
- **작은 목표 설정**: MVP부터 시작해서 완성도를 높였다
- **Claude 활용**: 문법이나 패턴은 Claude에게, 설계는 내가 주도
- **기록**: 블로그에 기록하며 배운 점을 정리했다

### 아쉬운 점
- **테스트 코드 부족**: 단위 테스트를 작성하지 못했다
- **디자인**: UI/UX가 아직 투박하다
- **iOS 미지원**: 안드로이드만 개발했다

### 배운 것
- Flutter는 생각보다 배우기 쉽다
- Claude는 동료 개발자처럼 협업할 수 있다
- **완벽함보다 완성이 중요하다**

## 다음 목표

1. **Google Play 배포** (이번 주 내)
2. 사용자 피드백 수집
3. Google Calendar 연동 (Phase 2)
4. iOS 버전 개발 (수요가 있다면)

드디어 배포 직전이다. 처음으로 **내가 만든 앱이 세상에 나간다**는 설렘이 크다.

> "시작이 반이라면, 배포는 90%다."

---

**시리즈**
1. [모아갤 프로젝트를 시작하며](/posts/MoaGal-01-Why-I-Started/)
2. [기술 스택 선정과 초기 계획](/posts/MoaGal-02-Tech-Stack/)
3. [개발 환경 구축과 첫 번째 난관](/posts/MoaGal-03-Setup-Challenges/)
4. [핵심 기능 구현하기](/posts/MoaGal-04-Core-Features/)
5. **테스트와 배포 준비** (현재 글)
6. [드디어 출시했다](/posts/MoaGal-06-Launch/)

---

**업데이트 예정**
- 첫 사용자 반응과 버그 수정기
- Google Calendar 연동 개발기
