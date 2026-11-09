---
type: backend
---

# 그룹 활동 일정 API 그룹 경로 전환

## 작업 개요

그룹 활동(`GROUP_ACTIVITY`) 일정의 생성·수정·삭제 API는 `/schedules/group-activity` 아래에 있고, 생성 요청은 대상 그룹을 요청 바디의 `groupId`로 받는다. 그룹 권한 선언 `@GroupAuth`는 경로 변수로 그룹을 식별하므로 이 API에는 적용할 수 없고, 그룹원 판정이 `ScheduleService`에 남아 있다.

그룹 활동 일정 API를 `/schedules/group/{groupId}/...` 아래로 옮겨 그룹 권한을 `@GroupAuth`로 선언하도록 전환하기 위한 후속 TODO다. Frontend의 그룹 대시보드에 일정 탭을 추가하는 작업과 함께 진행하며, 일정 API 구조를 바꾸는 [세미나 중심 생성·관리 전환 설계](seminar-management-design-backend.md)의 리팩터링과 같은 시점에 처리한다.

관련 저장소는 `sight-spring-backend`와 `sight-frontend`다. 이 Task의 완료 결과는 그룹 활동 일정의 새 경로 체계, 그룹 권한 선언, 기존 경로의 전환 범위와 저장소별 구현 범위를 확정한 설계 문서다. 이 Task를 기록한 시점에는 코드를 변경하거나 기존 API를 폐기하지 않는다.

- Backend: 그룹 활동 일정 API를 그룹 경로 아래로 옮기고 그룹 권한을 `@GroupAuth`로 선언하는 계약을 설계한다.
- Frontend: 확정된 API 계약을 기준으로 그룹 대시보드 일정 탭과 기존 일정 화면의 호출 경로 전환을 후속 Frontend Task로 작성한다.
- 의존 관계: 세미나 중심 전환의 일정 API 설계와 함께 공통 API 계약을 확정한 뒤 각 저장소의 구현 Task로 이어간다.

## 비즈니스 동작

이 Task에서 실제 동작 변경은 없다. 후속 설계는 다음 기존 동작을 기준으로 한다.

- 그룹 활동 일정은 진행 중(`PROGRESS`)인 그룹에만 등록할 수 있고, 해당 그룹의 그룹원만 등록할 수 있다.
- 그룹 활동 일정은 경험치(`expoint`)와 출첵 코드를 가지지 않는다.
- 수정은 일정 작성자 본인만 할 수 있다.
- 삭제는 일정 작성자 본인 또는 운영진이 할 수 있으며, 삭제해도 출석 기록은 유지한다.

설계에서는 경로의 그룹과 대상 일정의 소속 그룹이 일치하는지에 대한 판정 기준을 함께 정한다.

## HTTP API 계약

변경 없음. 이 Task는 설계 작업이며 새 endpoint를 도입하거나 기존 endpoint를 변경하지 않는다.

다음 기존 계약을 `/schedules/group/{groupId}/...` 경로 체계로 옮기는 방안을 설계한다.

- [그룹 활동 일정 생성](../../sight-spring-backend/docs/plans/schedule/create-group-activity-schedule.md): `POST /schedules/group-activity`. 요청 바디의 `groupId`를 경로 변수로 옮긴다.
- [그룹 활동 일정 수정](../../sight-spring-backend/docs/plans/schedule/update-group-activity-schedule.md): `PATCH /schedules/group-activity/{scheduleId}`
- [그룹 활동 일정 삭제](../../sight-spring-backend/docs/plans/schedule/delete-group-activity-schedule.md): `DELETE /schedules/group-activity/{scheduleId}`

후속 구현 전에 각 endpoint의 method·path·식별자·요청·응답·권한·오류와 기존 경로의 유지·전환 범위를 확정한다. 그룹 대시보드 일정 탭에서 사용하는 그룹 일정 계약도 같은 경로 체계로 확정한다.

## 데이터베이스 계약

변경 없음.

## 외부 시스템 계약

변경 없음.

## 보안 및 개인정보

변경 없음. 설계에서는 그룹 권한 판정을 서비스 검사에서 [그룹 권한 어노테이션](../../sight-spring-backend/docs/plans/group-auth.md)의 `@Auth`·`@GroupAuth` 선언으로 옮기는 범위를 정한다. 작성자 본인 여부처럼 그룹 권한이 아닌 제한은 서비스에 둔다.

## 비가역적 부작용 및 운영 영향

변경 없음. 이 Task에서는 데이터 삭제·변경, 알림 발송 및 외부 호출을 수행하지 않는다.

## 호환성 및 배포

변경 없음. 현재 `/schedules/group-activity` API는 이 Task로 변경되지 않는다.

기존 경로의 consumer는 `sight-frontend`의 `src/api/public/schedule.ts`다. 후속 설계는 Backend 제공과 Frontend 전환의 순서, 기존 경로의 유지·제거 범위 및 rollback 범위를 포함한다. 착수 시 기존 그룹 활동 일정 API 문서에 `deprecated`(폐기 예정)와 이 Task 링크를 표시한다.

## 검증

코드 테스트나 브라우저 조작이 아닌 설계 문서 검토로 완료 여부를 판단한다.

- 그룹 활동 일정의 생성·수정·삭제가 `/schedules/group/{groupId}/...` 경로로 정의되어 있다.
- 그룹 권한이 `@Auth`·`@GroupAuth` 선언으로 정의되고, 서비스에 남는 검사는 작성자 본인 여부 등 그룹 권한이 아닌 제한뿐이다.
- 경로의 그룹과 일정의 소속 그룹이 다를 때의 처리가 정의되어 있다.
- 기존 동작(진행 중인 그룹만 등록, 경험치 0, 삭제 시 출석 기록 유지)이 유지되는 계약이다.
- 그룹 대시보드 일정 탭이 사용할 계약과 기존 경로의 전환·배포 순서가 정리되어 있다.
- 세미나 중심 전환 설계와 일정 API 경로 체계가 충돌하지 않는다.

## 비목표

- 이 Task에서 그룹 대시보드 일정 탭을 직접 구현하는 것
- `@GroupAuth`가 요청 바디의 값을 읽도록 확장하는 것
- 운영진 일정·세미나 일정의 권한 정책을 변경하는 것
- 설계 확정 전에 기존 endpoint를 변경·폐기하는 것
