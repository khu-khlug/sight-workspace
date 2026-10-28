# 그룹 시스템 — 공통 인프라 및 외부 연동

## 개요

이 문서는 그룹 기능이 의존하는 공통 인프라(ID 채번, 알림, ExPoint, HTML 필터, 스케줄러)와 외부 서비스(Slack, Discord) 연동 방식을 기술합니다.

> **참고**: 레거시 시스템에서는 알림/Discord 연동을 별도 서버에 HTTP 요청하는 방식이었으나, 새 시스템에서는 그룹 기능이 같은 서버에 구현되므로 내부 API 호출이 불필요합니다. 아래 내용은 **구현해야 할 동작**을 이해하기 위한 참고용입니다.

---

## 1. 전역 ID 채번 (`sequence` 테이블)

### 동작 방식

그룹, 리스트, 카드, 기록, 로그 등 **모든 엔티티의 ID는 auto-increment가 아니라 `sequence` 테이블에서 채번**됩니다.

```php
// Controller::getSequence()
public static function getSequence()
{
    return DB::table('sequence')->insertGetId([]);
}
```

`sequence` 테이블에 빈 행을 insert하고, MySQL이 반환하는 `last_insert_id`(=`seq` 컬럼의 auto-increment 값)를 ID로 사용합니다.

### 목적

시스템 전체에서 ID가 겹치지 않도록 보장합니다. 예를 들어 그룹 ID와 카드 ID가 같은 숫자를 가지는 상황이 발생하지 않습니다. 이는 `group_card.list` 컬럼이 아카이브 시 group ID 값을 저장하는 것처럼, 서로 다른 엔티티 ID가 같은 컬럼에 혼용되는 구조에서 중요합니다.

### 영향 범위

`getSequence()`로 ID를 생성하는 테이블:

- `group`
- `group_list`
- `group_card`
- `group_record`
- `group_log`

---

## 2. 알림 시스템 (`Controller::message()`)

### 전체 구조

레거시 시스템에서는 알림이 이중으로 기록되었습니다. 새 시스템에서는 알림 저장소를 단일화하여 구현합니다.

```php
public static function message($category, $message, $to = 0, $secret = false, $from = 0)
```

### 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `$category` | string | - | 알림 카테고리 코드 (예: `'21'`, `'22'`) |
| `$message` | string | - | HTML 메시지 |
| `$to` | int \| string | `0` | 수신자. `0`=자기 자신, `'manager'`=운영진 전체, `'broadcast'`=전 회원 |
| `$secret` | bool | `false` | `true`이면 `from`이 0(익명)으로 처리 |
| `$from` | int | `0` | 발신자. `0`이면 로그인 회원(또는 `secret`이면 익명) |

### 특수 수신자 처리

**`$to = 'manager'`**: 운영진 전체에게 발송
```php
$managers = DB::table('members')->select('id')->where('manager', '1')->get();
foreach ($managers as $manager)
    Controller::message($category, $message, $manager->id, $secret, $from);
```

**`$to = 'broadcast'`**: state 0(휴학), 1(재학)인 모든 회원에게 익명으로 발송
```php
$undergraduates = DB::table('members')->select('id')
    ->where('state', '0')->orWhere('state', '1')->get();
foreach ($undergraduates as $undergraduate)
    Controller::message($category, $message, $undergraduate->id, true, 0);
```

### 알림 카테고리 분류

레거시에서는 숫자 코드를 사용했으나, 새 시스템에서는 문자열 카테고리로 통일합니다:

| 레거시 코드 | 카테고리 |
|------------|----------|
| `2`로 시작 (`21`, `22`, `23` 등) | `GROUP` |
| 그 외 | `SYSTEM` |

### 그룹 관련 카테고리 코드 전체 목록

| 코드 | 의미 | 발송 조건 |
|------|------|-----------|
| `21` | 그룹 정보 변경 알림 | 그룹 생성/상태변경/정보수정/멤버변경 등 |
| `22` | 카드 관련 알림 | 카드 추가/활성화/비활성화 |
| `23` | 기록 관련 알림 | 기록 추가, 담당자 지정 |
| `25` | 내 활동 알림 | 참여/탈퇴/즐겨찾기/보관 등 본인 행동 |
| `26` | 타인의 참여/탈퇴 알림 | 그룹장에게 멤버 변동 알림 |
| `83` | 세미나 방송 알림 | 세미나 접수 시작/마감 (broadcast) |
| `91` | 운영진/고객센터 알림 | 고객센터 기록 추가, 세미나 변경 등 |
| `92` | 고객센터 카드 추가 | 익명 알림 |

### 고객센터 그룹 (id=15265) 알림 특수 처리

고객센터는 익명성을 위해 일반 멤버 알림을 보내지 않고 별도 처리합니다:

- **카드 추가** 시: `secret=true`로 운영진에게 `category 92` 알림
- **기록 추가** 시: `secret=true`로 운영진에게 `category 91`, 카드 작성자에게 `category 91` 알림
- 일반 멤버(그룹 전체)에게는 알림 미발송

---

## 3. ExPoint 시스템 (`Controller::expoint()`)

### 동작 방식

```php
public static function expoint($message, $point, $to = 0)
```

1. `members.expoint` 컬럼을 `expoint + point`로 직접 업데이트
2. `expoint_log` 테이블에 변동 이력 insert

```sql
-- members 업데이트
UPDATE members SET expoint = expoint + (:point) WHERE id = :to

-- 로그 기록
INSERT INTO expoint_log (member, from, message, point, created_at)
VALUES (:to, :from, :message, :point, now())
```

### 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `$message` | string | - | HTML 메시지 (로그용) |
| `$point` | int | - | 변동량 (음수 가능) |
| `$to` | int | `0` | 대상 회원. `0`이면 로그인한 회원 |

### 그룹 관련 경험치 증감

경험치 기준과 등급은 [비즈니스 규칙의 경험치 체계](./GROUP_BUSINESS_RULES.md#14-경험치-체계)를 따른다.

| 행동 | 변동(exp) | 대상 |
|------|-----------|------|
| 그룹을 만듦 | +50 | 생성자 |
| 그룹에 참여함 | +40 | 참여 회원 |
| 그룹에서 나갔거나 내보내짐 | -40 | 나가거나 내보내진 회원 |
| 그룹이 성공적으로 종료됨 | +90 | 그룹 멤버 전체 |
| 그룹이 실패로 종료됨 | +70 | 그룹 멤버 전체 |
| 성공적으로 종료된 그룹이 다시 진행 중임 | -90 | 그룹 멤버 전체 |
| 실패로 종료된 그룹이 다시 진행 중임 | -70 | 그룹 멤버 전체 |
| 포트폴리오를 발행함 | +10 | 그룹 멤버 전체 |
| 포트폴리오 발행이 취소됨 | -10 | 그룹 멤버 전체 |
| 세미나 발표 등록을 함 | +150 | 그룹 멤버 전체 |
| 세미나 발표 등록을 취소함 | -150 | 그룹 멤버 전체 |
| 운영진이 세미나 세션에서 그룹을 강제 제거함 | -150 | 그룹 멤버 전체 |
| 보고서를 제출함 | +150 | 그룹 멤버 전체 |
| 보고서를 취소·삭제함 | -150 | 그룹 멤버 전체 |
| 카드를 만듦 | +1 | 카드를 만든 회원 |
| 카드에 기록을 작성함 | +2 | 기록을 작성한 회원 |
| 카드를 삭제함 | -1 | 카드를 만든 회원 |
| 기록을 삭제함 | -2 | 기록을 만든 회원 |
| 카드를 복구함 | +1 | 카드를 복구한 회원 |
| 기록을 복구함 | +2 | 기록을 복구한 회원 |

---

## 4. HTML XSS 필터 (`Controller::filterHTML()`)

기록 내용 저장/표시 시 XSS 방어를 위해 적용됩니다.

### 처리 단계

**1단계: `<script>` 태그 무력화**

```php
preg_replace('#<script(.*?)>#is', '&lt;script$1&gt;', $content)
```

여는 태그만 인코딩 (`</script>`는 이미 태그가 비활성화됐으므로 해롭지 않음).

**2단계: 이벤트 핸들러 (`on*`) 무력화**

`on` 두 글자(대소문자 무관)가 나오면 `o`를 `&#111;` 또는 `&#79;`로 치환합니다:

```php
preg_replace_callback(
    "/([^a-z])(o)(n)/i",
    function ($matches) {
        $matches[2] = ($matches[2] === "o") ? "&#111;" : "&#79;";
        return $matches[1] . $matches[2] . $matches[3];
    },
    $content
)
```

`onclick`, `onload`, `onerror` 등 모든 이벤트 핸들러를 무력화합니다.

**3단계: 위험 URI 스킴 치환**

```php
preg_replace('#j\s*a\s*v\s*a\s*s\s*c\s*r\s*i\s*p\s*t\s*:#i', 'javascript&#760;', $content)
str_replace('data:', 'data&#760;', $content)
```

`javascript:` (공백 포함 변형도 처리)와 `data:` URI를 무력화합니다.

### 적용 위치

- `Record::content()` — 기록 표시 시
- `Record` 클래스 내 `html_filter()` (private, 동일 로직의 내부 복사본)

### 주의사항

저장 시에는 필터를 적용하지 않고 원본 HTML을 저장합니다. **표시 시에만 필터를 적용합니다.** 따라서 `$record->content` 프로퍼티를 직접 출력하면 안 되고, 반드시 `$record->content()`를 사용해야 합니다.

---

## 5. 자동 중단 스케줄러

### 등록 위치

`app/Console/Kernel.php`:

```php
$schedule->call(function() {
    \App\Http\Controllers\GroupController::progressToSuspend();
})->hourly(); // 매 시간마다 실행
```

### 동작

`GroupController::progressToSuspend()` (static 메서드):

1. 다음 조건의 그룹을 조회:
   - `id != 15265` (고객센터 제외)
   - `state = 'progress'`
   - `changed_at < 현재 - 50일`

2. 각 그룹의 `state`를 `suspend`로 변경

3. 그룹 멤버 전체에게 알림: `"<그룹명> 그룹은 활동한지 50일이 넘어 중단 상태가 되었습니다."`

### 자동 복귀

중단된 그룹에서 카드 추가, 기록 추가 등 활동이 발생하면 `Group::changed()` 내에서 `suspend → progress` 로 자동 복귀됩니다. 별도 명시적 복귀 절차 없음.

### 특이사항

- `changed_at` 기준이므로 그룹 정보만 수정해선 중단 방지 안 됨
- `changed_at`을 갱신하는 동작: 카드 추가/삭제/복구, 기록 추가, 순서 변경, 멤버 참여/탈퇴 등 대부분의 그룹 활동
- 그룹 정보 수정(`postGroupModify`)은 `changed()` 호출이 있으나, 종료 상태면 그룹 수정 자체가 불가

---

## 6. 외부 연동

### 6-1. Discord 연동

그룹 시스템에서 필요한 Discord 관련 기능:

1. **그룹별 Discord 채널 URL 조회**: 그룹에 연결된 Discord 채널의 URL을 생성하여 표시
2. **사용자별 Discord 채널 참여 여부 조회**: 현재 사용자가 해당 그룹의 Discord 채널에 참여 중인지 확인

두 기능 모두 새 서버 내부에서 직접 구현합니다. 조회 실패 시 관련 UI를 표시하지 않는 방식으로 graceful 처리합니다.

---

### 6-2. Slack 연동

#### 세미나 상태 변경 시 알림

`GroupController::postSeminarTermState()` 에서 `active`가 `1`(접수 시작) 또는 `2`(접수 마감)로 변경될 때:

세미나 접수 시작/마감 시 알림을 전송합니다.
- 메시지 예시: "2026년 1학기 세미나의 세션 접수가 시작되었습니다."
- **변경사항**: 레거시에서는 Slack 채널로 발송했으나, 새 시스템에서는 **Discord 채널로 발송**합니다.
- 대상 채널은 환경 변수로 관리합니다.

---

### 6-3. Slack 기록 추가 연동

`POST /group/card/{id}/slack` 엔드포인트:

- **인증 없음** (그룹 컨트롤러에서 이 메서드만 auth 미들웨어 제외)
- 대신 `User-Agent: KHLUG SIGHT (slack)` 헤더로 요청 검증
- `member` 파라미터로 회원 ID를 받아 해당 회원 명의로 기록 추가
- 내부적으로 `postAddCardRecord()` 호출

```php
public function __construct()
{
    $this->middleware(
        'auth',
        ['except' => ['postAddCardRecordFromSlack']]
    );
}

public function postAddCardRecordFromSlack(Request $request, $id)
{
    if ($request->header('User-Agent') == 'KHLUG SIGHT (slack)')
        return $this->postAddCardRecord($request, $id, \App\User::find($request->member));
}
```

---

## 7. 환경 변수 정리

새 시스템에서 필요한 환경 변수:

| 용도 | 설명 |
|------|------|
| Discord 서버(길드) ID | Discord 채널 URL 생성용 |
| Slack 알림 채널 ID | 세미나 알림 발송 대상 채널 |
| Slack 봇 인증 정보 | Slack API 호출용 |

---

## 8. 공통 헬퍼 메서드 요약 (`Controller`)

| 메서드 | 시그니처 | 설명 |
|--------|----------|------|
| `getSequence()` | `static(): int` | sequence 테이블에서 전역 고유 ID 채번 |
| `message()` | `static($category, $message, $to, $secret, $from)` | 알림 발송 |
| `expoint()` | `static($message, $point, $to)` | ExPoint 변동 및 로그 기록 |
| `filterHTML()` | `static($content): string` | XSS 필터 적용 |
| `getError()` | `static($message, $code): Response` | 에러 응답 반환. 401이면 로그인 뷰 |
| `getCache()` | `static($id): object` | cache 테이블에서 캐시 조회 |
| `saveCache()` | `static($id, $data): bool` | cache 테이블에 캐시 저장 |
