# app_review_exchange
앱심사 품앗이 얌체를 박멸하기 위한 지식 공유

# 앱 품앗이 테스트 참여 확인 가이드

### Android Native · Flutter · iOS \| Firebase Analytics 기반

-   문서 목적: 품앗이 테스트 참여자가 **매일 앱을 실행했는지** 간단히
    확인한다.
-   기본 원칙: 각 앱에 짧은 추적 코드를 추가하고, 참여자 ID와 앱 식별
    정보를 Firebase Analytics로 전송한다.
-   권장 구성: Firebase Analytics + BigQuery 내보내기(선택). Analytics
    기본 화면만으로 사용자별 날짜를 편리하게 교차 조회하기는 제한이
    있어, 상세 통계표는 BigQuery를 이용하는 편이 적합하다.

------------------------------------------------------------------------

## 카드 1. 전체 진행 흐름

**목표:** 참여자가 앱에 최초 한 번 정보를 등록하면 이후에는 자동으로
방문 이벤트가 기록되게 한다.

1.  [ ] Firebase 프로젝트에 앱 등록
2.  [ ] Firebase Analytics SDK 설치 및 초기화
3.  [ ] 참여자 ID와 앱 식별값을 최초 한 번 저장
4.  [ ] 앱이 실행되거나 다시 활성화될 때 `tester_daily_visit` 이벤트
    기록
5.  [ ] DebugView에서 이벤트 수신 확인
6.  [ ] Google Analytics에서 이벤트를 확인
7.  [ ] 사용자별 날짜 통계가 필요하면 BigQuery 내보내기와 SQL 보고서
    구성

**중요:** 앱 실행 이벤트는 앱이 열렸다는 신호이지, 사용자가 실제로
내용을 충분히 테스트했다는 증명은 아니다. 테스트 정책은 실행 기록 외에
필요한 피드백이나 기능 확인도 포함하도록 정한다.

------------------------------------------------------------------------

## 카드 2. 이벤트 규격 통일하기

모든 참여 앱에서 아래 이름을 동일하게 사용한다. 이름을 통일해야 여러
앱의 통계를 함께 집계하기 쉽다.

  -----------------------------------------------------------------------
  항목                    값                      설명
  ----------------------- ----------------------- -----------------------
  이벤트 이름             `tester_daily_visit`    앱이 활성화될 때 기록

  사용자 ID               `tester_001` 같은 임의  품앗이 그룹에서 정한
                          ID                      고유 ID

  사용자 속성             `tester_name`           참여자 ID를 사용자
                                                  속성으로도 기록

  이벤트 매개변수         `tester_id`             이벤트 발생 시점의
                                                  참여자 ID

  이벤트 매개변수         `test_app_name`         테스트 대상 앱의 표시
                                                  이름

  이벤트 매개변수         `test_app_id`           패키지명 또는 Bundle ID
                                                  등 고유 식별자

  이벤트 매개변수         `test_round`            예: `2026-10-A`
  -----------------------------------------------------------------------

**ID 규칙 예시** - 참여자 ID: `tester_001`처럼 개인정보가 아닌 고정
식별자 - Android 앱 식별자: `com.example.sample` - iOS 앱 식별자:
`com.example.sample` - 회차: `2026-10-A`

**개인정보 주의:** 이름, 이메일, 전화번호 등 개인을 직접 식별할 수 있는
정보를 Analytics에 보내지 않는다. 참여자 ID는 그룹에서 합의한 임의의
코드로 사용한다.

------------------------------------------------------------------------

## 카드 3. Firebase 설정

1.  [ ] Firebase Console에서 프로젝트를 만든다.
2.  [ ] Android 앱과 iOS 앱을 각각 등록한다. 앱 식별자는 실제 패키지명 /
    Bundle ID와 일치해야 한다.
3.  [ ] Android용 `google-services.json`, iOS용
    `GoogleService-Info.plist`를 프로젝트에 추가한다.
4.  [ ] 플랫폼별 Firebase 설정 안내에 따라 SDK와 플러그인을 연결한다.
5.  [ ] Firebase Console의 Analytics가 활성화되어 있는지 확인한다.
6.  [ ] 앱을 실행하고 DebugView에서 이벤트 수신을 확인한다.

공식 문서: - Firebase Analytics 시작하기:
https://firebase.google.com/docs/analytics - Android 설정:
https://firebase.google.com/docs/analytics/get-started?platform=android -
Apple 플랫폼 설정:
https://firebase.google.com/docs/analytics/get-started?platform=ios -
Flutter 설정:
https://firebase.google.com/docs/analytics/get-started?platform=flutter

------------------------------------------------------------------------

## 카드 4. Android Native (Kotlin)

### 4-1. 의존성

앱 모듈의 Gradle dependencies에 Firebase Analytics를 추가한다. 버전은
Firebase Android BoM을 사용하는 프로젝트의 최신 설정을 따른다.

``` kotlin
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:YOUR_BOM_VERSION"))
    implementation("com.google.firebase:firebase-analytics")
}
```

`YOUR_BOM_VERSION`은 실제 프로젝트의 Firebase BoM 버전으로 바꾼다.
Google Services Gradle 플러그인 설정도 Firebase 공식 안내에 맞게
적용한다.

### 4-2. 방문 기록 함수

``` kotlin
import android.os.Bundle
import com.google.firebase.analytics.FirebaseAnalytics

fun recordTesterVisit(
    analytics: FirebaseAnalytics,
    testerId: String,
    testAppName: String,
    testAppId: String,
    testRound: String
) {
    analytics.setUserId(testerId)
    analytics.setUserProperty("tester_name", testerId)

    val params = Bundle().apply {
        putString("tester_id", testerId)
        putString("test_app_name", testAppName)
        putString("test_app_id", testAppId)
        putString("test_round", testRound)
    }

    analytics.logEvent("tester_daily_visit", params)
}
```

### 4-3. 호출 위치

`Application` 또는 메인 Activity가 앱을 활성화할 때 호출한다. 앱이
백그라운드에서 돌아올 때도 기록하고 싶다면 Activity 생명주기 또는
`ProcessLifecycleOwner`의 `ON_START` 등 적절한 앱-포그라운드 감지
지점에서 호출한다.

참여자 ID와 앱 정보는 최초 등록 후 `DataStore`나 안전한 앱 설정 저장소에
보관해 매번 입력하지 않도록 한다.

------------------------------------------------------------------------

## 카드 5. Flutter

### 5-1. 패키지

``` bash
flutter pub add firebase_core firebase_analytics
```

Firebase 프로젝트 연결은 FlutterFire 설정 절차에 따라 완료한다. 앱 시작
전에 Firebase를 초기화한다.

### 5-2. 방문 기록 함수

``` dart
import 'package:firebase_analytics/firebase_analytics.dart';

final analytics = FirebaseAnalytics.instance;

Future<void> recordTesterVisit({
  required String testerId,
  required String testAppName,
  required String testAppId,
  required String testRound,
}) async {
  await analytics.setUserId(id: testerId);
  await analytics.setUserProperty(
    name: 'tester_name',
    value: testerId,
  );

  await analytics.logEvent(
    name: 'tester_daily_visit',
    parameters: {
      'tester_id': testerId,
      'test_app_name': testAppName,
      'test_app_id': testAppId,
      'test_round': testRound,
    },
  );
}
```

### 5-3. 호출 위치

앱 시작 시 한 번만 호출하는 방식은 간단하지만, 앱을 종료하지 않고 다음
날까지 계속 켜 두는 경우 매일 방문을 기록하지 못할 수 있다. 매일 방문
여부가 중요하다면 앱이 포그라운드로 전환될 때 날짜를 확인해 **하루 한
번** 기록하도록 구현한다.

아래는 날짜별 중복 기록을 줄이는 간단한 예시다. `SharedPreferences`를
사용하므로 패키지를 추가한다.

``` bash
flutter pub add shared_preferences
```

``` dart
import 'package:shared_preferences/shared_preferences.dart';

Future<void> recordDailyVisitOnce({
  required String testerId,
  required String testAppName,
  required String testAppId,
  required String testRound,
}) async {
  final prefs = await SharedPreferences.getInstance();
  final today = DateTime.now();
  final dayKey =
      '${today.year.toString().padLeft(4, '0')}-'
      '${today.month.toString().padLeft(2, '0')}-'
      '${today.day.toString().padLeft(2, '0')}';
  final key = 'tester_visit_${testAppId}_$dayKey';

  if (prefs.getBool(key) == true) return;

  await recordTesterVisit(
    testerId: testerId,
    testAppName: testAppName,
    testAppId: testAppId,
    testRound: testRound,
  );
  await prefs.setBool(key, true);
}
```

위 함수는 `recordTesterVisit()`과 같은 파일에 두거나 해당 함수를
import해서 사용한다. 앱이 포그라운드로 전환될 때
`recordDailyVisitOnce()`를 호출한다. 앱의 생명주기를 감지하려면
`WidgetsBindingObserver`의 `didChangeAppLifecycleState`에서
`AppLifecycleState.resumed`를 처리할 수 있다.

**참고:** 기기 날짜 변경이나 앱 데이터 삭제 등으로 중복 방지 로직을
우회할 수 있다. 이 기능은 편의용이며 부정 참여를 완전히 방지하지는
않는다.

------------------------------------------------------------------------

## 카드 6. iOS Native (Swift)

### 6-1. 설정

1.  [ ] Firebase Console에 iOS 앱을 등록한다.
2.  [ ] `GoogleService-Info.plist`를 Xcode 프로젝트에 추가한다.
3.  [ ] Firebase Apple SDK의 Analytics 제품을 Swift Package Manager 또는
    프로젝트가 사용하는 지원 방식으로 연결한다.
4.  [ ] 앱 시작 시 `FirebaseApp.configure()`를 호출한다.

### 6-2. 방문 기록 함수

``` swift
import FirebaseCore
import FirebaseAnalytics

func recordTesterVisit(
    testerId: String,
    testAppName: String,
    testAppId: String,
    testRound: String
) {
    Analytics.setUserID(testerId)
    Analytics.setUserProperty(
        testerId,
        forName: "tester_name"
    )

    Analytics.logEvent("tester_daily_visit", parameters: [
        "tester_id": testerId,
        "test_app_name": testAppName,
        "test_app_id": testAppId,
        "test_round": testRound
    ])
}
```

앱 시작 시 `FirebaseApp.configure()`를 먼저 호출한다. 매일 방문 기록을
확인하려면 `sceneDidBecomeActive` 또는 SwiftUI의 `scenePhase` 변화 등
앱이 활성화되는 시점에서 날짜를 확인하고 하루 한 번 이벤트를 기록한다.
날짜 중복 방지 정보는 `UserDefaults`에 저장할 수 있다.

------------------------------------------------------------------------

## 카드 7. 이벤트가 실제로 들어오는지 확인하기

1.  [ ] 테스트 기기에서 앱을 실행한다.
2.  [ ] 앱을 종료하거나 백그라운드로 보낸 뒤 다시 활성화한다.
3.  [ ] Firebase Console의 Analytics DebugView에서 `tester_daily_visit`
    이벤트를 확인한다.
4.  [ ] 이벤트 매개변수에 `tester_id`, `test_app_name`, `test_app_id`,
    `test_round`가 들어왔는지 확인한다.
5.  [ ] 참여자 ID가 `setUserId` 및 `tester_name`으로 등록됐는지
    확인한다.

Analytics는 데이터 수집과 일반 보고서 반영에 시간이 걸릴 수 있다.
DebugView는 개발 중 이벤트 검증에 사용하고, 일반 보고서는 지연될 수
있음을 고려한다.

------------------------------------------------------------------------

## 카드 8. Google Analytics에서 보는 방법

### 빠른 확인

-   **실시간 / DebugView:** 이벤트가 들어오는지 개발 중 확인
-   **이벤트 보고서:** `tester_daily_visit` 발생량 확인
-   **맞춤 측정기준:** `test_app_name`과 같은 낮은 변동성의 속성으로
    앱별 분석

Google Analytics의 사용자 ID는 표준 보고서에서 원하는 형태로 곧바로
날짜별 행·열로 보기 어려울 수 있다. 또한 `tester_id`를 이벤트 매개변수로
보내는 것만으로 자동으로 맞춤 측정기준이 만들어지지는 않는다.

사용자별 식별자를 맞춤 측정기준으로 등록하는 것은 높은 카디널리티 문제를
일으킬 수 있으므로, 수십\~수백 명의 참여자를 날짜별로 비교하는 최종
통계에는 BigQuery를 권장한다.

------------------------------------------------------------------------

## 카드 9. BigQuery로 일별 방문표 만들기 (권장)

1.  [ ] Firebase Console → 프로젝트 설정 / 통합 메뉴에서 BigQuery를
    연결한다.
2.  [ ] Analytics 데이터 내보내기를 활성화한다.
3.  [ ] 데이터가 들어오기 시작할 때까지 기다린다. 최초 데이터는 즉시
    보이지 않을 수 있다.
4.  [ ] BigQuery에서 `tester_daily_visit` 이벤트를 기준으로
    날짜·참여자·앱별 방문 여부를 집계한다.
5.  [ ] 결과를 Google Sheets 또는 간단한 관리자 페이지로 표시한다.

### 기본 SQL 예시

아래의 `project_id.analytics_XXXXXXXXX.events_*`는 실제 BigQuery
데이터셋 이름으로 바꾼다.

``` sql
SELECT
  PARSE_DATE('%Y%m%d', event_date) AS visit_date,
  user_id AS tester_id,
  (
    SELECT value.string_value
    FROM UNNEST(event_params)
    WHERE key = 'test_app_name'
  ) AS test_app_name,
  (
    SELECT value.string_value
    FROM UNNEST(event_params)
    WHERE key = 'test_app_id'
  ) AS test_app_id,
  COUNT(*) AS event_count
FROM
  `project_id.analytics_XXXXXXXXX.events_*`
WHERE
  event_name = 'tester_daily_visit'
GROUP BY
  visit_date, tester_id, test_app_name, test_app_id
ORDER BY
  visit_date DESC, tester_id;
```

이 SQL은 방문 이벤트가 발생한 날짜를 참여자별로 집계한다. `user_id`가
비어 있는 이벤트도 있을 수 있으므로, 운영 전 실제 데이터에서 `user_id`가
채워지는지 확인한다. 필요하다면 `tester_id` 이벤트 매개변수를 대체
식별자로 활용하되, 이벤트별 ID 일관성을 검증한다.

### 보고서 예시

  참여자 ID    앱 이름   10/10   10/11   10/12
  ------------ --------- ------- ------- -------
  tester_001   Wemory    ✓       ✓       ✓
  tester_002   앱 B      ✓       ---     ✓
  tester_003   앱 C      ✓       ✓       ---

`✓`는 해당 날짜에 방문 이벤트가 존재함을 뜻한다. `—`는 기록이 없다는
뜻이며, 곧바로 미참여로 단정하기 전에 수집 오류, 네트워크, 앱 버전,
시간대 등을 확인한다.

------------------------------------------------------------------------

## 카드 10. 운영 규칙

-   [ ] 참여자 ID는 그룹에서 고유하게 발급하고 테스트 기간 중 바꾸지
    않는다.
-   [ ] 이벤트 이름과 매개변수 이름은 모든 앱에서 동일하게 유지한다.
-   [ ] 테스트할 앱 이름뿐 아니라 패키지명 / Bundle ID도 함께 기록한다.
-   [ ] 앱이 포그라운드로 전환될 때 날짜별 한 번 기록한다.
-   [ ] Analytics DebugView에서 이벤트를 먼저 검증한다.
-   [ ] 날짜별 통계표는 BigQuery 기반으로 만든다.
-   [ ] 기록이 없는 경우 즉시 제재하지 말고 오류나 사유를 확인한다.
-   [ ] 방문 기록은 앱 실행 지표로만 사용하고, 실제 테스트 품질은 별도
    피드백으로 확인한다.

## 최소 구현 체크리스트

-   [ ] Firebase Analytics 연결
-   [ ] 참여자 ID 최초 입력 및 저장
-   [ ] 앱 이름과 고유 식별자 저장
-   [ ] `tester_daily_visit` 이벤트 공통 적용
-   [ ] 앱 활성화 시 날짜별 하루 한 번 기록
-   [ ] DebugView 수신 확인
-   [ ] BigQuery 내보내기 연결
-   [ ] 참여자 × 날짜 통계표 구성

------------------------------------------------------------------------

### 참고 문서

-   Firebase Analytics: https://firebase.google.com/docs/analytics
-   Android에서 Analytics 시작:
    https://firebase.google.com/docs/analytics/get-started?platform=android
-   Apple 플랫폼에서 Analytics 시작:
    https://firebase.google.com/docs/analytics/get-started?platform=ios
-   Flutter에서 Analytics 시작:
    https://firebase.google.com/docs/analytics/get-started?platform=flutter
-   BigQuery로 Analytics 데이터 내보내기:
    https://support.google.com/analytics/answer/9358801
