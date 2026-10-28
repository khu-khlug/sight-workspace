# 그룹 시스템 데이터베이스 스키마

## 개요

그룹 시스템에 직접 관여하는 테이블과, 그룹 기능이 의존하는 공통 테이블을 정리합니다.
마이그레이션 파일 기준이며, 이후 직접 DDL로 추가된 컬럼(예: `group.interest`, `group_card.incharge`, `group_card.sort_record`)은 컨트롤러/모델 코드에서 역추적했습니다.

---

## 1. 그룹 핵심 테이블

### `group`

그룹 본체.

| 컬럼 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `id` | bigint | (sequence) | 전역 고유 ID (`sequence` 테이블에서 생성) |
| `category` | varchar | - | `study` \| `project` \| `documentation` \| `education` \| `program` \| `manage` |
| `title` | varchar | - | 그룹 이름 |
| `author` | bigint | - | 그룹 생성자 (`members.id`) |
| `master` | bigint | - | 현재 그룹장 (`members.id`) |
| `purpose` | longtext | NULL | 그룹 목적 |
| `state` | varchar | `'pending'` | `progress` \| `suspend` \| `end-success` \| `end-fail` |
| `interest` | varchar | NULL | 관심 분야 (`members_interest.name` 값을 `\|`로 구분, 마이그레이션 이후 추가) |
| `technology` | varchar | NULL | 기술 스택 (`,`로 구분, 양쪽 공백 trim 처리) |
| `allow_join` | tinyint(bool) | `0` | 참여 신청 허용 여부 |
| `grade` | tinyint | `3` | 공개 범위: `0`=비공개, `2`=운영진, `3`=회원, `4`=오픈 |
| `count_member` | bigint | `0` | 멤버 수 (캐시, `group_member` 변경 시 동기화) |
| `count_list` | bigint | `0` | 리스트 수 (캐시) |
| `count_card` | bigint | `0` | 활성 카드 수 (archive 제외, 캐시) |
| `count_record` | bigint | `0` | 활성 기록 수 (archive 제외, 캐시) |
| `last_updater` | bigint | NULL | 마지막 수정자 (`members.id`) |
| `repository` | varchar | NULL | 저장소 URL |
| `portfolio` | tinyint(bool) | `0` | 포트폴리오 발행 여부 |
| `updated_at` | timestamp | - | 기록 추가 등 데이터 변경 시점 |
| `changed_at` | timestamp | NULL | 그룹 활동 갱신 시점 (`changed()` 호출 시 갱신, 자동 중단 판정 기준) |
| `created_at` | timestamp | - | 그룹 생성 시점 |

**주의사항**:
- `id`는 auto-increment가 아니라 `sequence` 테이블에서 채번 (→ [GROUP_SYSTEM.md](./GROUP_SYSTEM.md) 참조)
- `count_*` 컬럼은 정규화 대신 카운터 캐시 방식 사용. 각 insert/delete 시 수동으로 `+1` / `-1` 처리
- `changed_at`은 50일 미갱신 시 자동 중단 스케줄러의 기준값

---

### `group_member`

그룹-멤버 다대다 연결 테이블.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `group` | bigint | `group.id` |
| `member` | bigint | `members.id` |

**주의사항**:
- PK 없음 (복합 unique constraint도 없음)
- 고객센터 그룹(`id=15265`)은 이 테이블에 아무 레코드도 없지만, 비즈니스 로직에서 모든 회원이 멤버로 취급됨 (→ `Group::basic_permission()` 참조)
- 그룹장(`master`)은 이 테이블에서 일반 멤버와 동일하게 저장됨. `group.master` 컬럼이 별도로 존재

---

### `group_list`

그룹 내 리스트 (칸반 보드의 컬럼).

| 컬럼 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `id` | bigint | (sequence) | 전역 고유 ID |
| `group` | bigint | - | `group.id` |
| `list_order` | bigint | - | 표시 순서 (0부터 시작, 오름차순) |
| `name` | varchar | - | 리스트 이름 |
| `description` | longtext | NULL | 리스트 설명 |
| `count_card` | bigint | `0` | 활성 카드 수 (캐시) |
| `updated_at` | timestamp | - | |
| `created_at` | timestamp | - | |

---

### `group_card`

리스트 내 카드.

| 컬럼 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `id` | bigint | (sequence) | 전역 고유 ID |
| `list` | bigint | - | `group_list.id`. 아카이브된 카드는 이 값이 `group.id`로 변경됨 |
| `card_order` | bigint | - | 리스트 내 표시 순서 (내림차순으로 표시) |
| `name` | varchar | - | 카드 이름 |
| `author` | bigint | - | 카드 생성자 (`members.id`) |
| `incharge` | bigint | NULL | 담당자 (`members.id`, 마이그레이션 이후 추가) |
| `description` | longtext | NULL | 카드 설명 |
| `labels` | int | `0` | 레이블 비트마스크 (아래 별도 설명) |
| `state` | varchar | `'pending'` | `public` \| `deactivate` \| `archive` |
| `count_record` | bigint | `0` | 활성 기록 수 (캐시) |
| `sort_record` | varchar | `'desc'` | 기록 정렬 방향: `asc` \| `desc` (마이그레이션 이후 추가) |
| `portfolio` | tinyint(bool) | `0` | 포트폴리오 포함 여부 |
| `from` | varchar | NULL | 이어받기 설정. `"{group_id}#{card_id}"` 형식 |
| `updated_at` | timestamp | - | |
| `created_at` | timestamp | - | |

**레이블 비트마스크**:

| 레이블 | 비트값 |
|--------|--------|
| red | 1 |
| yellow | 2 |
| green | 4 |
| blue | 8 |
| violet | 16 |
| white | 32 |
| gray | 64 |
| black | 128 |

토글: `labels XOR 비트값`. 확인: `(labels AND 비트값) == 비트값`.

**아카이브 처리 시 특이사항**:
- 카드가 아카이브되면 `list` 컬럼 값이 원래 리스트 ID에서 **해당 그룹의 `group.id`**로 변경됨
- `state = 'archive'`로 변경됨
- `Card::group()`은 `list` 값으로 `group_list`를 조회했을 때 없으면 `list` 값을 직접 group ID로 사용하는 fallback 로직 포함

---

### `group_record`

카드 내 기록.

| 컬럼 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `id` | bigint | (sequence) | 전역 고유 ID |
| `card` | bigint | - | `group_card.id` |
| `author` | bigint | - | 기록 작성자 (`members.id`) |
| `content` | longtext | NULL | HTML 내용 (CKEditor 입력값, XSS 필터 적용 후 저장) |
| `state` | varchar | `'pending'` | `public` \| `archive` |
| `portfolio` | tinyint(bool) | `1` | 포트폴리오 포함 여부 (기본값 `1` — 기록 생성 시 자동 포함) |
| `updated_at` | timestamp | - | |
| `created_at` | timestamp | - | |

---

### `group_log`

그룹 활동 로그 (그룹 내에서 발생한 모든 변경 이력).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | bigint | 전역 고유 ID |
| `group` | bigint | `group.id` |
| `member` | bigint | 행위자 (`members.id`, 0이면 시스템) |
| `message` | longtext | HTML 메시지 (앵커 태그 포함) |
| `created_at` | timestamp | |

**주의사항**: 고객센터 그룹(`id=15265`)은 `Group::logs()`, `Group::last_log()` 호출 시 빈 배열/null 반환 — 로그가 DB에는 쌓이지만 열람 불가

---

### `group_bookmark`

그룹 즐겨찾기.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `member` | bigint | `members.id` |
| `group` | bigint | `group.id` |
| `created_at` | timestamp | |

~~**주의사항**: `User::groups_bookmark()`는 이 테이블에 없더라도 `group.id=7549` (활용실습)와 `group.id=15265` (고객센터, 교류 회원 제외)를 항상 목록 끝에 추가~~

**특수그룹은 이제 없음**: 특수그룹은 더이상 그룹 시스템을 사용하지 않고 별도의 페이지나 서비스로 이전될 예정입니다. 따라서 그룹 관련 서비스에서 특수그룹에 대한 예외처리는 이제 필요하지 않습니다. 

---

### `group_seminar`

세미나 세션 등록 (그룹-기수 연결).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `group` | bigint | `group.id` |
| `term` | varchar | `group_seminar_term.term` |
| `created_at` | timestamp | |

---

### `group_seminar_term`

세미나 기수 정보.

| 컬럼 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `term` | varchar (unique) | - | `YYYYSS` 형식. SS: `10`=1학기, `15`=여름학기, `20`=2학기, `25`=겨울학기 |
| `active` | tinyint(bool) | `0` | `0`=접수 전, `1`=접수 중, `2`=마감 |
| `updated_at` | timestamp | | |
| `created_at` | timestamp | | |

---

### `group_record_saved`

회원별 기록 보관함. (마이그레이션 파일 없음 — 직접 DDL 추가된 것으로 추정)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `member` | bigint | `members.id` |
| `record` | bigint | `group_record.id` |
| `saved_at` | timestamp | 보관 시점 |

---

## 2. 공통 의존 테이블

### `sequence`

전역 ID 채번 테이블. 그룹/카드/기록 등 모든 엔티티 ID의 원천.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `seq` | bigint (auto_increment PK) | 채번값 |

사용 방법: `DB::table('sequence')->insertGetId([])` — insert 후 반환된 `seq` 값을 ID로 사용. 시스템 전체에서 유니크한 ID가 보장됨.

자세한 내용 → [GROUP_SYSTEM.md](./GROUP_SYSTEM.md)

---

### `members`

회원 정보 (User 모델의 실제 테이블명은 `members`).

그룹 기능과 관련된 주요 컬럼:

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | bigint | 회원 ID |
| `state` | int | `-1`=교류, `0`=휴학, `1`=재학, `2`=졸업 |
| `manager` | tinyint(bool) | 운영진 여부 |
| `expoint` | bigint | 활동 포인트 (그룹 활동마다 직접 업데이트) |
| `active` | tinyint(bool) | 계정 활성 여부 |

---

### `expoint_log`

ExPoint 변동 이력.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | bigint (auto_increment) | |
| `member` | bigint | 포인트 수령 회원 |
| `from` | bigint | 포인트 부여 주체 (0=시스템) |
| `message` | longtext | HTML 메시지 |
| `point` | bigint | 변동량 (음수 가능) |
| `updated_at` | timestamp | |
| `created_at` | timestamp | |

---

### `notification`

알림 레거시 테이블. 새 시스템에서는 알림 저장소를 단일화하여 재설계합니다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | bigint (auto_increment) | |
| `member` | bigint | 수신자 (`members.id`) |
| `from` | bigint | 발신자 (0=익명/시스템) |
| `category` | varchar | 알림 카테고리 코드 |
| `message` | longtext | HTML 메시지 |
| `readed` | tinyint(bool) | 읽음 여부 |
| `updated_at` | timestamp | |
| `created_at` | timestamp | |

그룹 관련 카테고리 코드 및 알림 상세 → [GROUP_SYSTEM.md](./GROUP_SYSTEM.md) 참조

---

### `files`

파일 및 이미지 첨부 테이블. 기록 첨부파일과 카드 커버 이미지 모두 여기 저장됨.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `article` | bigint | 연결 대상 ID. 기록이면 `group_record.id`, 카드 커버면 `group_card.id` |
| `author` | bigint | 업로드한 회원 (`members.id`, 0=비회원) |
| `original` | varchar | 원본 파일명 |
| `name` | varchar | 저장된 파일명 (`rand(100,999) + uniqid() + 확장자`) |
| `extension` | varchar | 확장자 |
| `mime` | varchar | MIME 타입 |
| `size` | bigint | 바이트 단위 크기 |
| `type` | varchar | `file` \| `image` |
| `count_download` | bigint | 다운로드 횟수 |
| `updated_at` | timestamp | |
| `created_at` | timestamp | |

---

### `autosave`

기록 작성/수정 자동 저장. 기록이 최종 저장되면 해당 항목 삭제됨.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `author` | bigint (unique) | 작성 회원 (`members.id`) |
| `location` | varchar | 저장 위치 식별자 (요청 바디의 `location` 필드) |
| `content` | longtext | 자동 저장 내용 |
| `extra` | longtext | 추가 데이터 |
| `ip_address` | varchar | 작성자 IP |
| `created_at` | timestamp | |

---

### `members_interest`

관심 분야 목록. `group.interest` 및 회원 관심 분야의 유효값 검증에 사용.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | bigint (auto_increment 추정) | |
| `name` | varchar | 관심 분야 이름 (e.g., `"웹"`, `"AI"`) |

---

### `special_dept`

특수 조직 (team, company 등).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | bigint | 조직 ID |
| `name` | varchar | 내부 식별자 (`team`, `company` 등) |
| `real` | varchar | 표시 이름 |
| `updated_at` | timestamp | NULL 가능 |
| `created_at` | timestamp | |

---

### `special_dept_group`

특수 조직-그룹 연결.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `group` | bigint | `group.id` |
| `dept` | bigint | `special_dept.id` |
| `created_at` | timestamp | |

---

### `special_dept_member`

특수 조직-멤버 연결. 그룹 생성/수정 시 사용자의 조직 소속 검증에 사용.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `member` | bigint | `members.id` |
| `dept` | bigint | `special_dept.id` |
| `created_at` | timestamp | |

---

### `ideacloud`

아이디어클라우드. 그룹 생성 시 아이디어 연동에 사용.

| 컬럼 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `id` | bigint | - | ID |
| `idea` | varchar | - | 아이디어 내용 |
| `author` | bigint | - | 작성자 (`members.id`) |
| `state` | varchar | `'pending'` | `public` \| `expired` |
| `updated_at` | timestamp | | |
| `created_at` | timestamp | | |

그룹 생성 시 아이디어로 그룹을 만들면 해당 아이디어의 `state`가 `expired`로 변경됨.

---

## 3. 테이블 관계도 (간략)

```
sequence
  └─ ID 채번 → group, group_list, group_card, group_record, group_log

group
  ├─ group_member      (1:N, group.id ↔ members.id)
  ├─ group_list        (1:N)
  │    └─ group_card   (1:N)
  │         └─ group_record  (1:N)
  ├─ group_log         (1:N)
  ├─ group_bookmark    (N:M, group.id ↔ members.id)
  ├─ group_seminar     (N:M, group.id ↔ group_seminar_term.term)
  └─ special_dept_group (N:M, group.id ↔ special_dept.id)

group_record
  └─ group_record_saved (N:M, group_record.id ↔ members.id)

group_card / group_record
  └─ files (article = group_card.id 또는 group_record.id)

members
  ├─ expoint → expoint_log
  └─ notification (알림)
```
