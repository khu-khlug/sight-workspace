---
type: backend
---

# 기존 정수 ID 채번·API 문자열 전환 TODO

## 작업 개요

그룹 시스템에 우선 도입하는 TSID를 기존 정수 ID 생성 경로와 API로 확대한다. 이번 그룹 시스템 구현과 분리한 후속 작업이며 아직 구현하지 않는다.

관련 저장소는 `sight-spring-backend`, `sight-frontend`와 내부 그룹 API 소비자다. 백엔드의 기존 채번·숫자 요청·응답과 소비자의 숫자 ID 처리를 함께 전환해 TSID 정밀도 손실 없이 왕복할 수 있어야 완료다. 구체적인 기존 동작은 [TSID 결정문서의 교체 지점](../../sight-spring-backend/docs/decisions/integer-id-tsid.md#기존-동작에서-교체할-지점)을 기준으로 추적한다.

## 비즈니스 동작

기존 그룹·그룹 로그·일정 생성 중 남아 있는 자체 채번을 공통 TSID 발급으로 대체한다. 그룹 시스템 구현에서 이미 전환한 생성 경로는 중복 변경하지 않는다. 기존 데이터의 ID와 참조·URL은 유지하고 재채번하지 않는다. 생성 외의 권한·상태 전이·중복 요청 처리 규칙은 변경 없음.

## HTTP API 계약

기존 API의 그룹·그룹 로그·일정 ID 및 중첩 참조를 JSON 숫자에서 선행 0 없는 양의 Long 범위 10진수 문자열로 바꾼다. Base32 대신 10진수를 쓰는 이유는 레거시 식별자·URL과의 호환성이다. 숫자 JSON을 받은 뒤 문자열로 변환하는 방식은 허용하지 않는다.

대상 필드는 다음과 같다. 아래 파일은 `sight-spring-backend/src/main/kotlin/com/sight/controllers/http/dto/` 기준이며 메서드·경로는 기존 API를 유지한다.

| 기존 representation | 전환 필드 |
| :-- | :-- |
| `ListGroupsResponse`, `CreateGroupResponse` | 그룹 `id` |
| `GetGroupMatchingGroupsResponse` | 최상위 그룹 `id` |
| `GetGroupMatchingAnswerResponse`, `CreateGroupMatchingAnswerResponse` | `matchedGroups[].groupId` |
| `ListGroupLogResponse` | 로그 `id` |
| `ActivityReportResponse`, `ListActivityReportsResponse` | `groupId` |
| `CreateGroupDiscordChannelResponse`, `GetGroupDiscordChannelResponse` | `groupId` |
| `CreateScheduleResponse`, `GetScheduleResponse`, `ListSchedulesResponse` | 일정 `id` 및 존재하는 `groupId` |
| `ListActiveSchedulesResponse` | 공유 일정 representation의 `id`·`groupId` |
| `CreateScheduleAttendanceResponse` | `scheduleId` |
| `CreateGroupActivityScheduleRequest` | 요청 `groupId` |

다음은 공통 ID 표현 예시이며 전체 응답을 뜻하지 않는다. 각 필드의 기존 필수 여부·nullable 여부는 유지한다.

```json
{
  "id": "906558544281600001",
  "groupId": "17200000"
}
```

요청의 대상 ID는 문자열로 받고 양의 Long 범위를 검사한다. JSON 숫자·빈 문자열·선행 0·음수·소수·범위 초과는 `400 Bad Request`로 거부한다. 경로·쿼리는 기존 10진수 표현을 유지한다. 인증·인가·성공 응답 상태·기타 오류·페이지네이션·필터·정렬·멱등성 계약은 변경 없음. 회원 ID·수량과 ULID 문자열은 변경하지 않는다.

## 데이터베이스 계약

ID와 참조는 기존 signed `BIGINT`를 유지하며 기존 행을 갱신하지 않는다. PK·유일 제약·연관관계·삭제 정책은 유지한다. TSID로 전환할 자동 증가 대상은 테이블의 모든 생성 경로를 사전 할당으로 함께 바꾸고 신규 행 저장 판정을 명시한다. 같은 컬럼에 TSID를 삽입한 뒤에도 기존 자동 증가 값이 작게 유지된다고 가정하지 않는다.

전환 전에 운영 담당자가 해당 테이블과 이전할 원본의 `MIN(id)`·`MAX(id)`·`AUTO_INCREMENT`를 확인하고 기존 ID 상한을 정한다. 기존 ID backfill과 재채번은 하지 않는다.

## 외부 시스템 계약

내부 그룹 채널 조회 API 소비자도 `groupId`를 10진수 문자열로 처리한다. 외부 채널 ID와 ULID, 연동 인증·재시도·실패 처리 계약은 변경 없음.

## 보안 및 개인정보

변경 없음. ID 표현 변경으로 조회·수정 권한을 완화하지 않는다.

## 비가역적 부작용 및 운영 영향

새로 발급된 TSID는 롤백해도 유지한다. 발급 활성화 시 `INTEGER_ID_ENABLED=true`, JVM마다 고유한 `INTEGER_ID_NODE_ID`, 조사한 `INTEGER_ID_LEGACY_MAX_ID`를 설정한다. 기존 백엔드와 전환 대상의 발급 작업을 종료한 뒤 일괄 배포한다. 노드 번호 재사용 조건은 결정문서를 따른다.

## 호환성 및 배포

JSON 숫자에서 문자열로 바뀌므로 기존 소비자와 하위 호환되지 않는다. 백엔드의 기존 채번 교체와 API 문자열 전환, 프런트 API 타입·호출·목업·로컬 저장의 ID 문자열 처리를 함께 준비한다. 기존 숫자 응답만 유지한 채 채번기를 TSID로 먼저 바꾸지 않는다.

그룹 시스템의 우선 적용 과정에서 같은 테이블의 TSID를 기존 API가 읽을 수 있다면 그 경로는 이 TODO까지 미룰 수 없다. 해당 응답·소비자를 그룹 시스템 배포에 포함하거나 그 대상의 TSID 쓰기를 보류한다. 특히 기존 그룹 목록·활동 로그 조회와 일정·매칭의 그룹 참조를 확인한다. 이후 남은 경로만 이 Task의 후속 범위로 유지한다.

그룹 상세의 기존 `Number(groupId)`·`Number(recordId)`와 초안의 숫자 ID 저장은 그룹 시스템 구현에서 함께 처리한다. 남은 기존 일정·매칭·목록 소비자의 ID 처리와 내부 API 소비자를 이 후속 작업에서 전환한다. 역할 번호·수량 등 무관한 숫자 처리는 유지한다.

구 백엔드가 새 TSID를 숫자로 반환하거나 자동 증가로 후속 ID를 발급할 수 있는 상태로 롤백하지 않는다.

## 검증

동작 확인은 담당자가 수행한다. 에이전트는 사용자의 명시적 요청 없이 테스트·브라우저 조작을 실행하지 않는다.

- 기존 작은 ID와 `2^53 - 1`을 넘는 ID가 요청·응답·중첩 참조·URL·로컬 저장에서 동일한 문자열로 왕복하는지 확인한다.
- 잘못된 문자열과 JSON 숫자 요청이 400으로 거부되고 기존 권한·상태 오류가 유지되는지 확인한다.
- 기존 채번 함수·채번용 재시도·자동 증가가 전환 대상에 남지 않고 신규 저장이 기존 행을 덮어쓰지 않는지 확인한다.
- 일반 일정·그룹 활동·빅세미나·출석·매칭·내부 채널 소비자가 같은 ID를 정밀도 손실 없이 참조하는지 확인한다.
- 운영 상한과 TSID 범위의 비중첩, 노드 번호 중복 방지 및 기존 발급 작업 종료를 확인한다.

## 비목표

그룹 시스템 우선 도입을 이 후속 작업 때문에 일괄 확대하지 않는다. 기존 ID 재채번, ULID 교체, 회원·수량의 일괄 문자열화, 전역 Long 직렬화, DB 노드 임대 테이블 추가는 하지 않는다.
