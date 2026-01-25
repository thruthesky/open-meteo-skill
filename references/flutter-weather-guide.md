# Flutter 날씨 구현 가이드

Open-Meteo API를 활용한 Flutter 앱 날씨 기능 구현 가이드입니다.

## 관련 소스 코드

- 날씨 모델: [lib/weather/models/weather.model.dart](../../../lib/weather/models/weather.model.dart)
- 날씨 서비스: [lib/weather/services/weather.service.dart](../../../lib/weather/services/weather.service.dart)
- 날씨 화면: [lib/weather/screens/weather.screen.dart](../../../lib/weather/screens/weather.screen.dart)
- 홈 화면 (퀵메뉴): [apps/singapore/lib/screens/home/main.home.dart](../../../apps/singapore/lib/screens/home/main.home.dart)

---

## 1. 시간대별 날씨 표시 기법

### 1.1 핵심 개념

날씨 데이터를 효과적으로 표시하기 위해 시간대별로 다른 간격을 사용합니다:

| 기간 | 표시 간격 | 이유 |
|------|----------|------|
| **오늘** | 2시간 단위 | 현재 시간 근처의 상세한 날씨 정보 필요 |
| **내일~7일** | 4시간 단위 | 대략적인 일별 날씨 패턴 파악 |

### 1.2 오늘 날씨 (2시간 단위) - 핵심 로직

```dart
/// 오늘 날씨 목록 (2시간 단위, 현재 시간 이후)
///
/// 오늘 남은 시간의 날씨를 2시간 간격으로 필터링
List<HourlyWeather> getTodayWeather() {
  final now = DateTime.now();
  final today = DateTime(now.year, now.month, now.day);
  final tomorrow = today.add(const Duration(days: 1));

  // 오늘 날씨만 필터링 (현재 시간 이후)
  final todayWeather = hourlyWeather
      .where((w) =>
          w.time.isAfter(now.subtract(const Duration(hours: 1))) &&
          w.time.isBefore(tomorrow))
      .toList();

  // 2시간 단위로 필터링 (짝수 시간대만: 00, 02, 04, 06, 08, 10, 12, 14, 16, 18, 20, 22)
  return todayWeather.where((w) => w.time.hour % 2 == 0).toList();
}
```

**핵심 포인트:**
- `now.subtract(Duration(hours: 1))`: 현재 시간 1시간 전부터 포함 (사용자가 방금 지난 시간대도 볼 수 있도록)
- `w.time.hour % 2 == 0`: 짝수 시간대만 필터링 (0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22시)

### 1.3 내일~7일 날씨 (4시간 단위) - 핵심 로직

```dart
/// 특정 날짜의 날씨 목록 (4시간 단위)
///
/// [date] 조회할 날짜
/// 4시간 간격으로 필터링 (00:00, 04:00, 08:00, 12:00, 16:00, 20:00)
List<HourlyWeather> getDayWeather(DateTime date) {
  final dayStart = DateTime(date.year, date.month, date.day);
  final dayEnd = dayStart.add(const Duration(days: 1));

  // 해당 날짜 날씨만 필터링
  final dayWeather = hourlyWeather
      .where((w) => w.time.isAfter(dayStart) || w.time.isAtSameMomentAs(dayStart))
      .where((w) => w.time.isBefore(dayEnd))
      .toList();

  // 4시간 단위로 필터링 (0, 4, 8, 12, 16, 20시)
  return dayWeather.where((w) => w.time.hour % 4 == 0).toList();
}
```

**핵심 포인트:**
- `w.time.hour % 4 == 0`: 4시간 간격 (0, 4, 8, 12, 16, 20시)
- 하루 6개의 시간대만 표시하여 화면 공간 효율적 사용

### 1.4 내일~7일간 날짜 목록 생성

```dart
/// 내일~7일간 날짜 목록
List<DateTime> getUpcomingDays() {
  final now = DateTime.now();
  final today = DateTime(now.year, now.month, now.day);

  return List.generate(7, (i) => today.add(Duration(days: i + 1)));
}
```

### 1.5 화면에서 사용하는 방법

```dart
// WeatherScreen에서 사용 예시
final data = _weatherData!;
final todayWeather = data.getTodayWeather();      // 오늘: 2시간 단위
final upcomingDays = data.getUpcomingDays();      // 내일~7일간 날짜 목록

// 오늘 날씨 섹션 표시
_buildDaySection(
  label: '오늘',
  weather: todayWeather,
  isToday: true,
);

// 내일~7일 날씨 섹션 표시
for (var i = 0; i < upcomingDays.length; i++) {
  _buildDaySection(
    label: _getDayLabel(upcomingDays[i], i == 0),  // "내일" 또는 "1/27" 형식
    weather: data.getDayWeather(upcomingDays[i]),  // 4시간 단위
    isToday: false,
  );
}
```

---

## 2. 캐시 기반 실시간 날씨 아이콘 표시

### 2.1 핵심 개념

홈 화면의 퀵 메뉴에서 날씨 아이콘을 정적 아이콘이 아닌 **실제 현재 날씨**에 맞는 동적 아이콘으로 표시합니다.

**구현 흐름:**
1. `WeatherService.getCurrentWeather()` 호출
2. 캐시된 날씨 데이터에서 현재 시간과 가장 가까운 날씨 반환
3. `HourlyWeather.icon` 및 `HourlyWeather.iconColor`로 아이콘 표시

### 2.2 현재 날씨 조회 로직

#### WeatherData.getCurrentWeather() - 모델 레벨

```dart
/// 현재 시간의 날씨 반환
///
/// 현재 시간과 가장 가까운 시간대의 날씨를 반환합니다.
/// 데이터가 없으면 null을 반환합니다.
HourlyWeather? getCurrentWeather() {
  if (hourlyWeather.isEmpty) return null;

  final now = DateTime.now();

  // 현재 시간과 가장 가까운 날씨 찾기
  HourlyWeather? closest;
  int? minDiff;

  for (final weather in hourlyWeather) {
    final diff = (weather.time.difference(now).inMinutes).abs();
    if (minDiff == null || diff < minDiff) {
      minDiff = diff;
      closest = weather;
    }
  }

  return closest;
}
```

**핵심 포인트:**
- `weather.time.difference(now).inMinutes.abs()`: 현재 시간과의 차이를 분 단위로 계산
- 가장 작은 차이를 가진 시간대의 날씨를 반환

#### WeatherService.getCurrentWeather() - 서비스 레벨

```dart
/// 현재 날씨만 조회
///
/// 캐시된 날씨 데이터에서 현재 시간과 가장 가까운 날씨를 반환합니다.
/// 데이터가 없으면 null을 반환합니다.
Future<HourlyWeather?> getCurrentWeather() async {
  try {
    final data = await getWeather();  // 캐시 우선 조회
    return data.getCurrentWeather();
  } catch (e) {
    return null;  // 에러 시 null 반환 (기본 아이콘 표시)
  }
}
```

**핵심 포인트:**
- `getWeather()`: 캐시된 데이터 우선 사용 (30분 TTL)
- 에러 발생 시 null 반환 → UI에서 기본 아이콘 표시

### 2.3 FutureBuilder를 사용한 동적 아이콘 표시

```dart
// 날씨 퀵메뉴 - 실시간 날씨 아이콘 표시
FutureBuilder<HourlyWeather?>(
  future: WeatherService.instance.getCurrentWeather(),
  builder: (context, snapshot) {
    // 현재 날씨 데이터 (로딩 중이거나 에러면 기본 아이콘 표시)
    final weather = snapshot.data;
    final weatherIcon = weather?.icon ?? Icons.wb_cloudy;
    final weatherIconColor = weather?.iconColor ?? const Color(0xFFFF9800);

    return _buildQuickMenuItem(
      icon: weatherIcon,
      label: '날씨',
      color: const Color(0xFFFFF3E0),
      iconColor: weatherIconColor,
      onTap: () {
        WeatherScreen.show(context);
      },
    );
  },
),
```

**핵심 포인트:**
- `weather?.icon ?? Icons.wb_cloudy`: 날씨 데이터가 없으면 기본 구름 아이콘
- `weather?.iconColor ?? Color(0xFFFF9800)`: 날씨 데이터가 없으면 기본 주황색
- 환율 퀵메뉴와 동일한 FutureBuilder 패턴 사용

### 2.4 WMO 날씨 코드 → Flutter 아이콘 매핑

```dart
/// WMO 코드 → Flutter Icon 변환
///
/// 낮/밤에 따라 다른 아이콘 표시
IconData get icon {
  // 맑음
  if (weatherCode == 0) {
    return isDay ? Icons.wb_sunny : Icons.nights_stay;
  }

  // 대체로 맑음
  if (weatherCode == 1 || weatherCode == 2) {
    return isDay ? Icons.wb_sunny : Icons.nights_stay;
  }

  // 흐림
  if (weatherCode == 3) {
    return Icons.cloud;
  }

  // 안개
  if (weatherCode >= 45 && weatherCode <= 48) {
    return Icons.foggy;
  }

  // 이슬비
  if (weatherCode >= 51 && weatherCode <= 57) {
    return Icons.grain;
  }

  // 비
  if (weatherCode >= 61 && weatherCode <= 67) {
    return Icons.water_drop;
  }

  // 눈
  if (weatherCode >= 71 && weatherCode <= 77) {
    return Icons.ac_unit;
  }

  // 소나기
  if (weatherCode >= 80 && weatherCode <= 82) {
    return Icons.shower;
  }

  // 뇌우
  if (weatherCode >= 95 && weatherCode <= 99) {
    return Icons.thunderstorm;
  }

  // 기본값
  return Icons.cloud;
}
```

### 2.5 낮/밤 판단 로직

```dart
/// 낮/밤 여부 (6시-18시 = 낮)
bool get isDay {
  final hour = time.hour;
  return hour >= 6 && hour < 18;
}
```

### 2.6 WMO 코드 → 아이콘 색상 매핑

```dart
/// WMO 코드 → 아이콘 색상
Color get iconColor {
  // 맑음 (낮)
  if (weatherCode == 0 && isDay) {
    return const Color(0xFFFFB300); // 노란색
  }

  // 맑음 (밤)
  if (weatherCode == 0 && !isDay) {
    return const Color(0xFF5C6BC0); // 보라색
  }

  // 대체로 맑음
  if (weatherCode == 1 || weatherCode == 2) {
    return isDay ? const Color(0xFFFFB300) : const Color(0xFF5C6BC0);
  }

  // 흐림
  if (weatherCode == 3) {
    return const Color(0xFF78909C); // 회색
  }

  // 안개
  if (weatherCode >= 45 && weatherCode <= 48) {
    return const Color(0xFF90A4AE); // 연한 회색
  }

  // 이슬비/비/소나기
  if ((weatherCode >= 51 && weatherCode <= 67) ||
      (weatherCode >= 80 && weatherCode <= 82)) {
    return const Color(0xFF42A5F5); // 파란색
  }

  // 눈
  if (weatherCode >= 71 && weatherCode <= 77) {
    return const Color(0xFF81D4FA); // 하늘색
  }

  // 뇌우
  if (weatherCode >= 95 && weatherCode <= 99) {
    return const Color(0xFF5C6BC0); // 보라색
  }

  // 기본값
  return const Color(0xFF78909C);
}
```

---

## 3. 캐시 설정

### 3.1 FileCache 설정

```dart
final _weatherCache = FileCache<WeatherData>(
  cacheName: 'singapore_weather',
  defaultTtl: Duration(minutes: 30),  // 30분 캐시
  fromJson: WeatherData.fromJson,
  toJson: (data) => data.toJson(),
  useMemoryCache: true,  // 메모리 캐시도 사용 (더 빠른 응답)
);
```

### 3.2 캐시 우선 조회 패턴

```dart
Future<WeatherData> getWeather({bool forceRefresh = false}) async {
  // 1. 캐시 확인 (강제 새로고침이 아닌 경우)
  if (!forceRefresh) {
    final cached = await _weatherCache.get(_cacheKey);
    if (cached != null) {
      return cached;
    }
  }

  // 2. API 호출
  final response = await dio.get(_baseUrl, queryParameters: {...});

  // 3. 응답 파싱
  final data = WeatherData.fromApiResponse(response.data);

  // 4. 캐시 저장
  await _weatherCache.set(_cacheKey, data);

  return data;
}
```

---

## 4. API 호출 설정

### 4.1 싱가포르 좌표

```dart
static const double _latitude = 1.3521;
static const double _longitude = 103.8198;
static const String _timezone = 'Asia/Singapore';
```

### 4.2 API 파라미터

```dart
final response = await dio.get(
  'https://api.open-meteo.com/v1/forecast',
  queryParameters: {
    'latitude': _latitude,
    'longitude': _longitude,
    'hourly': 'temperature_2m,weather_code,relative_humidity_2m',
    'forecast_days': 8,
    'timezone': _timezone,
  },
);
```

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| `latitude` | 1.3521 | 싱가포르 위도 |
| `longitude` | 103.8198 | 싱가포르 경도 |
| `hourly` | temperature_2m,weather_code,relative_humidity_2m | 시간별 데이터 |
| `forecast_days` | 8 | 오늘 포함 8일간 예보 |
| `timezone` | Asia/Singapore | 싱가포르 시간대 |

---

## 5. WMO 날씨 코드 요약

| 코드 | 설명 | Flutter 아이콘 |
|------|------|---------------|
| 0 | 맑음 | `Icons.wb_sunny` (낮) / `Icons.nights_stay` (밤) |
| 1-2 | 대체로 맑음 | `Icons.wb_sunny` (낮) / `Icons.nights_stay` (밤) |
| 3 | 흐림 | `Icons.cloud` |
| 45-48 | 안개 | `Icons.foggy` |
| 51-57 | 이슬비 | `Icons.grain` |
| 61-67 | 비 | `Icons.water_drop` |
| 71-77 | 눈 | `Icons.ac_unit` |
| 80-82 | 소나기 | `Icons.shower` |
| 95-99 | 뇌우 | `Icons.thunderstorm` |

---

## 6. 전체 구현 체크리스트

### 6.1 모델 (weather.model.dart)

- [ ] `HourlyWeather` 클래스 - 시간별 날씨 데이터
  - [ ] `icon` getter - WMO 코드 → Flutter 아이콘
  - [ ] `iconColor` getter - WMO 코드 → 아이콘 색상
  - [ ] `isDay` getter - 낮/밤 판단
  - [ ] `description` getter - 한글 날씨 설명
- [ ] `WeatherData` 클래스 - 전체 날씨 응답
  - [ ] `getTodayWeather()` - 오늘 2시간 단위
  - [ ] `getDayWeather(date)` - 특정일 4시간 단위
  - [ ] `getUpcomingDays()` - 내일~7일 날짜 목록
  - [ ] `getCurrentWeather()` - 현재 시간 날씨

### 6.2 서비스 (weather.service.dart)

- [ ] 싱글톤 패턴 적용
- [ ] 30분 FileCache 설정
- [ ] `getWeather()` - 캐시 우선 조회
- [ ] `getCurrentWeather()` - 현재 날씨 조회

### 6.3 화면 (홈 퀵메뉴)

- [ ] FutureBuilder로 동적 아이콘 표시
- [ ] 로딩/에러 시 기본 아이콘 폴백
- [ ] 아이콘 탭 시 날씨 화면 표시
