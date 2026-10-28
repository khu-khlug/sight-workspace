# 그룹 페이지 정책

이 문서는 그룹 페이지의 프론트엔드 구현 정책을 정의한다. 개발 판단의 기준은 [개발 원칙](principles.md), 용어는 [용어 정의](terminology.md)를 따른다.

## 구현

동작을 추가하거나 변경할 때 기존 패턴과 구조를 확장하여 처리할 수 있다면 그 방식을 따른다. 사용처에 별도 분기를 추가하기 전에 해당 동작의 소유자와 확장 지점을 확인한다. 기존 구조로 책임과 흐름을 유지하기 어렵다면 개발 원칙에 따라 구조 변경을 제안한다.

아래 경로는 별도 표기가 없으면 `sight-frontend/src/features/member/GroupDetailContainer/` 기준이다.

### 창의 기본 상태전이와 조건별 패치

[WindowLayer/windowManager.ts](../../../sight-frontend/src/features/member/GroupDetailContainer/WindowLayer/windowManager.ts)는 추상적인 창 상태전이, 역할과 물리 슬롯의 대응, 역할 정규화를 담당한다. 실제 콘텐츠와 창 인스턴스는 WindowLayer가 관리한다.

| 변경 대상 | 확장 방법 |
| --- | --- |
| 새 창 열기·닫기·내리기·올리기·펼치기·교환·보내기의 기본 규칙 | 해당 `*_MAP` 상태전이표를 수정한다. 유효한 상태 조합을 명시하고, 정의되지 않은 입력은 오류로 처리한다. |
| 호출 맥락이나 조건에 따라 기본 전이를 대체하는 동작 | 독립된 패치 함수를 만들고 해당 `*_PATCHES` 체인에 등록한다. 호출부에 같은 조건 분기를 반복하지 않는다. |
| 새로운 종류의 창 액션 | `WindowAction` 유니온과 `applyWindowAction` 분기에 함께 추가한다. |
| 역할과 실제 좌우 위치의 변환 | `roleForSide`, `sideForRole` 등 기존 변환 함수를 사용한다. 사용처마다 좌우와 메인·서브의 대응을 다시 판단하지 않는다. |

기본 규칙 자체가 바뀌는 경우는 전이표를 수정한다. 특정 요청에만 다른 규칙이 필요한 경우는 패치로 처리한다. 예를 들어 기록 편집창으로 출발 카드창을 교체하는 동작은 `patchReplaceSourceWindow`, 편집 종료 후 콘텐츠를 복귀시키는 동작은 `patchRestoreContentWhenWindowClosed`가 처리한다.

패치 체인은 배열 앞에서부터 실행하며, 처음으로 `null`이 아닌 결과를 반환한 함수의 결과를 채택한다. 중간 함수의 `null`은 다음 패치로 넘긴다는 뜻이다. 마지막 기본 함수의 결과는 그대로 채택하며, 닫기 체인의 마지막 함수가 반환하는 `null`은 창 레이어가 종료된다는 뜻이다. 등록 순서가 우선순위이므로 새 패치를 추가할 때 기존 패치와의 적용 순서를 함께 결정한다.

### 창 배치와 상태 적용

[WindowLayer/index.tsx](../../../sight-frontend/src/features/member/GroupDetailContainer/WindowLayer/index.tsx)는 실제 창 인스턴스, 너비 비율, 가용 영역, 상태 적용과 복원을 담당한다.

| 변경 대상 | 사용할 지점 |
| --- | --- |
| 최소 너비·듀얼 사용 가능 조건·좌우 너비 계산 | `dualWidths`와 `canGoDual` 계산을 수정한다. 관련 CSS 변수와 상수도 일치시킨다. |
| 너비에 따른 메인 역할 결정 | `mainSideForSplit`과 `reassignMainSide`를 사용한다. 역할 재배정은 사용자에 의한 내용 교환과 구분한다. |
| 좌우 보내기 등 비율 변경과 상태전이를 함께 수행하는 조작 | WindowLayer에서 목표 비율과 역할을 결정하고 기존 전이 함수와 상태 적용 경로를 사용한다. |
| 일반 조작의 결과 확정 | `commit`을 거쳐 역할을 정규화하고 상태를 적용한 뒤 히스토리를 갱신한다. |
| 현재 상태와 목표 상태의 차이에 따른 화면 반영 | `applySnapshot`에서 처리한다. 닫힘 확인, 포커스 이동, 닫힘·이동 표현을 각 호출부에 복제하지 않는다. |

창 이동이나 역할 변경에서는 기존 `windowId`를 유지한다. React key에는 임시 역할 번호가 아니라 창 인스턴스 ID를 사용한다. `normalizeRoles`로 역할을 다시 붙일 때 콘텐츠와 창 ID도 같은 대응으로 재배정한다.

분할 드래그 중에는 `applySnapshot`으로 화면을 갱신하고, 놓을 때 히스토리 갱신과 개인설정 저장을 수행한다. 취소하면 시작 스냅샷을 복원한다. 이동 이벤트마다 내비게이션이나 영구 저장을 추가하지 않는다.

### 창의 공개 인터페이스와 콘텐츠 추가

[actions.ts](../../../sight-frontend/src/features/member/GroupDetailContainer/actions.ts)는 창 도메인 외부에 공개할 함수와 타입을 재익스포트한다. 외부에서 필요한 인터페이스가 생기면 이 파일에 공개 항목을 추가한다. 내부 전용 함수는 공개하지 않는다. `WindowLayer/windowManager.ts` 직접 import는 WindowLayer 내부와 `actions.ts`를 제외하고 ESLint로 제한한다.

창 콘텐츠는 `WindowBinding`으로 제공받은 `onClose`, `onExpand`, `onSendToSide` 등의 함수를 사용한다. 저장하지 않은 변경에 대한 종료 확인이 필요하면 `registerCloseGuard`에 확인 함수를 등록하고 필요가 없어지면 해제한다. WindowLayer가 창이 실제로 사라지는 경로에서 확인하므로 닫기 버튼에만 별도 확인 로직을 넣지 않는다.

새로운 창 콘텐츠 종류를 추가할 때는 다음 지점을 함께 확장한다.

1. 콘텐츠 도메인의 타입 파일에 콘텐츠 키 인코딩·디코딩을 정의한다.
2. `WindowOpenRequest`와 `encodeWindowOpenRequest`에 요청 타입과 키 변환을 추가한다.
3. 페이지의 `openWindow` 진입점을 사용하고, `renderContent`에 해당 콘텐츠 렌더링을 연결한다.

WindowLayer는 콘텐츠 키를 불투명한 식별자로 다룬다. 파일·카드·편집창별 데이터 조회나 표시 규칙을 WindowLayer에 추가하지 않는다.

`Window/index.tsx`는 개별 창의 헤더와 콘텐츠 영역을 담당하고, 전체 배치·바깥 클릭·겹침 처리는 WindowLayer가 담당한다. 콘텐츠의 스크롤은 해당 콘텐츠가 관리한다.

### URL과 히스토리

공유 가능한 창 배치의 해시 문법은 `WindowLayer/windowHash.ts`, 창 인스턴스 ID와 비율 등 히스토리 데이터는 `WindowLayer/windowHistory.ts`에서 관리한다. 주소나 복원 항목을 확장할 때 인코딩·디코딩과 입력 검증을 함께 수정한다.

뒤로가기·앞으로가기·직접 진입은 `syncFromHash`에서 목표 스냅샷을 만들고 `applySnapshot`으로 적용한다. 새 창을 여는 액션을 다시 실행하여 복원하지 않는다. 주소 정규화나 인스턴스 ID 보충은 현재 히스토리 항목을 `replace`한다.

현재 일반 조작의 히스토리 갱신은 해시가 달라지면 `push`, 같으면 `replace`한다. 비율만 달라진 조작도 이 규칙을 따른다. 히스토리 저장 기준을 변경할 때는 `updateHistory`에서 처리한다.

### 페이지 공유 상태와 개인설정

[pageState.tsx](../../../sight-frontend/src/features/member/GroupDetailContainer/pageState.tsx)는 여러 도메인이 공유하는 페이지 수명의 상태를 React Context로 제공한다. 현재 줄바꿈 상태인 `textWrap`과 변경 함수인 `toggleTextWrap`이 여기에 있다.

같은 수명과 공유 범위를 가진 상태를 추가할 때는 상태 타입·초기값·변경 함수를 이 파일에 정의하고, 소비자는 `useGroupDetailPageState`를 사용한다. 도메인 내부 상태는 해당 도메인에 유지한다. 변경 함수는 직접 수정하는 setter를 여러 사용처에 노출하기보다 의미가 드러나는 인터페이스로 제공한다.

페이지를 닫아도 유지할 개인설정은 `groupPagePreferences.ts`의 `groupPagePreferences` 객체로 조회·변경하고, React에서는 `useGroupPagePreferences`로 구독한다. 현재 저장 항목은 듀얼창 사용 여부와 분할 비율이며, 저장 키는 `sight:group-page-preferences`이다. 저장 항목을 추가할 때 타입·기본값·파싱·저장을 함께 확장한다. 컴포넌트가 이 설정의 localStorage를 직접 읽거나 쓰지 않는다.

### 서버 데이터와 보관 기록

그룹 API 요청과 DTO는 `sight-frontend/src/api/public/group/`의 해당 API 파일에 둔다. UI는 API 함수의 입력·반환 타입을 사용하며 요청 경로나 응답 변환을 각 사용처에 작성하지 않는다.

그룹 데이터 변경에는 `sight-frontend/src/hooks/group/useGroupAction.ts`의 변경 실행·성공 시 그룹 쿼리 무효화·오류 표시 패턴을 사용한다. 그룹에 속하지 않는 사용자 전역 설정 변경에는 그룹 쿼리 무효화를 적용하지 않는다. 직접 캐시를 갱신하는 도메인은 해당 도메인의 기존 변경 인터페이스를 따른다.

보관한 기록 목록의 소유자는 QueryClient 캐시다. `savedRecords.ts`의 `groupSavedRecords.queryOptions`, `insert`, `remove`를 사용하여 조회와 변경을 공유한다. 보관 버튼이나 보관함에 별도의 목록을 만들거나 각각 API 호출과 캐시 수정 로직을 구현하지 않는다.

### 파일 요청과 기록 표시

파일의 실제 요청 소스 선택은 `actions.ts`의 `resolveFileSource`가 담당한다. 소비자는 `fileApi.get` 또는 `getMetadata`를 사용한다. 새 소스를 추가할 때 소스 선택과 API 연결을 이곳에서 확장하고, 실제 HTTP 요청은 해당 API 파일에 둔다.

기록 종류별 뷰어 선택과 지연 로딩은 `RecordViewer.tsx`에서 관리한다. 기록을 표시하는 곳은 이 컴포넌트를 사용한다. 새 기록 종류를 추가하면 API의 기록 타입과 이 분기를 함께 확장한다.

### 기록 편집과 임시저장

임시저장은 `EditWindow/useRecordDraft.ts`를 사용한다. 입력 시 `touch`로 변경을 알리고, 본문은 `getContent`를 통해 저장이 필요한 시점에 직렬화한다. 종료·제출 흐름에서는 기존 `flush`, `pause`, `resume`, `clear` 인터페이스를 사용하며 별도의 타이머나 복원 흐름을 추가하지 않는다.

서버 임시저장의 요청과 쓰기 순서는 `sight-frontend/src/api/public/group/AutosaveApi.ts`가 담당한다. 저장과 삭제가 같은 쓰기 큐를 사용하도록 유지한다.

편집기와 팁탭 뷰어의 공통 포맷은 `sight-frontend/src/components/BlockContent/extensions.ts`의 `createBlockExtensions`에 등록한다. 편집 전용 기능은 BlockEditor에 둔다. 직접 입력 기호는 `WRAP_CHARS`를 확정 규칙과 도움말이 함께 참조하도록 유지한다.

### 탭 구성

탭 ID·라벨·아이콘·배치와 표시 옵션은 `groupDetailTabs.ts`에서 관리한다. 새 탭은 이 정의와 `Dashboard/index.tsx`의 콘텐츠 연결을 함께 추가한다. 탭 표시 정보가 필요한 사용처마다 별도 목록을 만들지 않는다.
