---
layout: post
title:  "모아갤 프로젝트 #4 - 핵심 기능 구현하기"
description: 캘린더 UI, 로컬 DB 연동, 일정 CRUD 구현 과정
date:   2026-03-29 00:00:00 +000
categories: MoaGal Flutter SQLite TableCalendar
---

## 캘린더 라이브러리 선택

Flutter에서 캘린더를 직접 구현할 수도 있지만, 바퀴를 재발명할 필요는 없다. Claude와 함께 라이브러리를 조사했다:

- **table_calendar**: 가장 인기 있고 커스터마이징이 자유로움
- **flutter_calendar_carousel**: 심플하지만 업데이트가 느림
- **syncfusion_flutter_calendar**: 기능은 많지만 무겁다

**선택**: `table_calendar` - 활발한 유지보수와 좋은 문서화

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  table_calendar: ^3.0.9
  sqflite: ^2.3.0
  intl: ^0.18.1
  provider: ^6.1.1
```

## 로컬 데이터베이스 설계

백엔드 개발 경험이 여기서 빛을 발했다. 테이블 설계는 익숙한 영역:

```sql
CREATE TABLE events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  description TEXT,
  start_time TEXT NOT NULL,
  end_time TEXT NOT NULL,
  is_all_day INTEGER DEFAULT 0,
  color INTEGER,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```

Flutter에서는 `sqflite` 패키지로 구현:

```dart
class DatabaseHelper {
  static final DatabaseHelper instance = DatabaseHelper._init();
  static Database? _database;

  DatabaseHelper._init();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDB('moagal.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);

    return await openDatabase(
      path,
      version: 1,
      onCreate: _createDB,
    );
  }

  Future _createDB(Database db, int version) async {
    await db.execute('''
      CREATE TABLE events (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        description TEXT,
        start_time TEXT NOT NULL,
        end_time TEXT NOT NULL,
        is_all_day INTEGER DEFAULT 0,
        color INTEGER,
        created_at TEXT DEFAULT CURRENT_TIMESTAMP
      )
    ''');
  }
}
```

## 일정 CRUD 구현

### 모델 정의

```dart
class Event {
  final int? id;
  final String title;
  final String? description;
  final DateTime startTime;
  final DateTime endTime;
  final bool isAllDay;
  final int color;

  Event({
    this.id,
    required this.title,
    this.description,
    required this.startTime,
    required this.endTime,
    this.isAllDay = false,
    this.color = 0xFF2196F3,
  });

  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'title': title,
      'description': description,
      'start_time': startTime.toIso8601String(),
      'end_time': endTime.toIso8601String(),
      'is_all_day': isAllDay ? 1 : 0,
      'color': color,
    };
  }

  factory Event.fromMap(Map<String, dynamic> map) {
    return Event(
      id: map['id'],
      title: map['title'],
      description: map['description'],
      startTime: DateTime.parse(map['start_time']),
      endTime: DateTime.parse(map['end_time']),
      isAllDay: map['is_all_day'] == 1,
      color: map['color'],
    );
  }
}
```

### 데이터베이스 연산

```dart
class EventService {
  Future<int> createEvent(Event event) async {
    final db = await DatabaseHelper.instance.database;
    return await db.insert('events', event.toMap());
  }

  Future<List<Event>> getEventsByDate(DateTime date) async {
    final db = await DatabaseHelper.instance.database;
    final startOfDay = DateTime(date.year, date.month, date.day);
    final endOfDay = startOfDay.add(Duration(days: 1));

    final results = await db.query(
      'events',
      where: 'start_time >= ? AND start_time < ?',
      whereArgs: [startOfDay.toIso8601String(), endOfDay.toIso8601String()],
      orderBy: 'start_time ASC',
    );

    return results.map((map) => Event.fromMap(map)).toList();
  }

  Future<int> updateEvent(Event event) async {
    final db = await DatabaseHelper.instance.database;
    return await db.update(
      'events',
      event.toMap(),
      where: 'id = ?',
      whereArgs: [event.id],
    );
  }

  Future<int> deleteEvent(int id) async {
    final db = await DatabaseHelper.instance.database;
    return await db.delete('events', where: 'id = ?', whereArgs: [id]);
  }
}
```

## 캘린더 UI 구현

```dart
class CalendarScreen extends StatefulWidget {
  @override
  _CalendarScreenState createState() => _CalendarScreenState();
}

class _CalendarScreenState extends State<CalendarScreen> {
  DateTime _selectedDay = DateTime.now();
  DateTime _focusedDay = DateTime.now();
  Map<DateTime, List<Event>> _events = {};

  @override
  void initState() {
    super.initState();
    _loadEvents();
  }

  Future<void> _loadEvents() async {
    // 한 달치 이벤트 로드
    final events = await EventService().getEventsForMonth(_focusedDay);
    setState(() {
      _events = events;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('모아갤')),
      body: Column(
        children: [
          TableCalendar(
            firstDay: DateTime.utc(2020, 1, 1),
            lastDay: DateTime.utc(2030, 12, 31),
            focusedDay: _focusedDay,
            selectedDayPredicate: (day) => isSameDay(_selectedDay, day),
            eventLoader: (day) => _events[day] ?? [],
            onDaySelected: (selectedDay, focusedDay) {
              setState(() {
                _selectedDay = selectedDay;
                _focusedDay = focusedDay;
              });
            },
          ),
          Expanded(
            child: _buildEventList(),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddEventDialog(),
        child: Icon(Icons.add),
      ),
    );
  }

  Widget _buildEventList() {
    final events = _events[_selectedDay] ?? [];
    if (events.isEmpty) {
      return Center(child: Text('일정이 없습니다'));
    }
    return ListView.builder(
      itemCount: events.length,
      itemBuilder: (context, index) {
        final event = events[index];
        return ListTile(
          leading: CircleAvatar(
            backgroundColor: Color(event.color),
          ),
          title: Text(event.title),
          subtitle: Text(
            '${DateFormat('HH:mm').format(event.startTime)} - '
            '${DateFormat('HH:mm').format(event.endTime)}'
          ),
          onTap: () => _showEventDetails(event),
        );
      },
    );
  }
}
```

## Claude와의 협업 포인트

이 과정에서 Claude가 특히 도움이 됐던 부분:

1. **코드 구조 제안**: "이렇게 분리하는 게 어때?" 하는 아키텍처 제안
2. **에러 디버깅**: `Unhandled Exception` 발생 시 원인 분석
3. **코드 리팩토링**: 중복 코드를 함수로 추출

하지만 **모든 걸 Claude에게 맡기지는 않았다**. DB 설계나 비즈니스 로직은 내가 주도하고, Claude는 Flutter 문법이나 위젯 사용법을 보완했다.

## 다음 단계

MVP의 핵심 기능은 완성했다. 이제:
- 위젯 구현 (안드로이드 홈 화면)
- Google Calendar 연동
- 앱 아이콘 및 스플래시 화면

다음 편에서는 앱을 실제 기기에 배포하고 테스트하는 과정을 다룬다.

> "코드가 실행되는 순간의 희열. 개발자라서 행복하다."

---

**시리즈**
1. [모아갤 프로젝트를 시작하며](/posts/MoaGal-01-Why-I-Started/)
2. [기술 스택 선정과 초기 계획](/posts/MoaGal-02-Tech-Stack/)
3. [개발 환경 구축과 첫 번째 난관](/posts/MoaGal-03-Setup-Challenges/)
4. **핵심 기능 구현하기** (현재 글)
5. 테스트와 배포 준비
