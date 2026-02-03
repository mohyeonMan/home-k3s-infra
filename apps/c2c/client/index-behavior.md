# index.html 동작 요약 (현재)

이 문서는 최신 변경을 반영한 `index.html`의 현재 동작을 정리합니다.

## 1. 화면 구성
- Home view: 방 생성/입장 시작 화면.
- Chat view: 방 헤더, entries 목록, 메시지 스트림, 입력창.
- Modals: 생성/입장, 승인, 진행 상태, 재연결, 시스템 알림, 나가기 확인, 토스트.

## 2. 상태 구조(State)
- `State.join`: 입장 흐름 상태(`nickname`, `roomId`, `requestId`, `joinRequestId`, `pending`).
- `State.create`: 생성 흐름 상태(`nickname`, `roomId`, `requestId`).
- `State.session`:
  - `roomId`, `nickname`, `userId`, `ownerId`, `autoDeleteAt`
  - `entries[]` (각 entry는 `online` 포함)
- `State.reconnect`: 재연결 흐름(`pending`, `mode`, `blocked`).
- `State.pendingJoinRequest`: 방장이 승인해야 하는 요청 payload.

## 3. 저장소/복원
- **localStorage**
  - `c2cUserId`: `CONN_OPENED`로 내려온 userId.
  - `c2cRoomId`, `c2cNickname`: 마지막 접속 방/닉네임.
  - 기존 `sessionStorage` 값이 있으면 자동 마이그레이션.
- **복원**
  - 앱 시작 시 `roomId`/`nickname` 복원 후 “마지막 방”으로 사용.

## 4. 단일 탭 락
- localStorage 키: `c2cActiveTabId`, `c2cActiveAt`.
- 탭 시작 시 락 획득 시도.
- 다른 탭이 활성이라면 이 탭은 연결 차단 + 안내 모달.
- `storage` 이벤트로 락 상실 감지 시 연결 종료.
- `beforeunload` 시 락 해제.

## 5. WebSocket
- URL: `ws://jhhomehub.gonetis.com/ws`
- `userId`가 있으면 쿼리로 전달.
- Heartbeat: 주기적으로 `HEARTBEAT`, ACK 없으면 재연결.
- 재연결 시 `JOIN`이 아니라 **`ONLINE` 전송**.

## 6. 프로토콜 의미(중요)
- `JOIN`: **멤버십 추가**
- `LEAVE`: **멤버십 삭제**
- `ONLINE`: **입장**
- `OFFLINE`: **퇴장**

## 7. 수신 처리
### SYSTEM
- `CONN_OPENED`:
  - `userId` 저장(localStorage 포함).
  - 즉시 `ROOM_LIST` 요청.

### RESULT
- `ROOM_CREATE`
  - `roomId`/`nickname` 저장.
  - 바로 `ONLINE` 전송.
- `JOIN`
  - `roomId`/`nickname` 저장.
  - 바로 `ONLINE` 전송.
- `ONLINE`
  - **RoomSummary 반환**(roomId, ownerId, entries, autoDeleteAt).
  - 방 상태 반영 후 Chat view 진입.
  - 재연결 모달이 열려 있으면 종료.
- `ROOM_LIST`
  - 홈 화면의 방 목록 UI 갱신.
  - 현재 `roomId`가 목록에 있으면 요약 반영.
  - 목록에 없으면 저장된 `roomId`/`nickname` 삭제.

### NOTIFY
- `JOIN`: 멤버 추가(`online=false`).
- `LEAVE`: 멤버 삭제.
- `ONLINE`: 멤버 `online=true`.
- `OFFLINE`: 멤버 `online=false`.
- `JOIN_REQUEST`: 방장일 경우 승인 모달 표시.
- `JOIN_APPROVE`: 승인 시 `JOIN` 전송, 거절 시 취소.

### MESSAGE
- `CLIENT_MESSAGE`: 채팅 메시지 표시.

## 8. 송신 커맨드
- `ROOM_CREATE`, `JOIN_REQUEST`, `JOIN`, `ONLINE`, `OFFLINE`, `ROOM_LIST`,
  `CLIENT_MESSAGE`, `HEARTBEAT`

## 9. 입장/퇴장 흐름
1. **입장**
   - `JOIN_REQUEST` → `JOIN_APPROVE` → `JOIN` → `ONLINE`
   - 입장 확정은 `ONLINE` 결과에서 처리
2. **퇴장**
   - `OFFLINE` 전송
   - 저장된 `roomId`/`nickname` 삭제
   - Home view로 이동
3. **연결 끊김**
   - 저장값 유지
   - 재연결 시 `ONLINE` 전송
4. **멤버 탈퇴**
   - `LEAVE` 전송
   - 저장된 `roomId`/`nickname` 삭제
   - Home view로 이동

## 10. 홈 화면 방 목록
- 홈 화면 진입 시 `ROOM_LIST` 요청.
- 목록은 “이미 JOIN된 방들”만 표시.
- 각 항목은 **입장 버튼**을 제공하며, 버튼 클릭 시 `ONLINE` 전송.
- 목록 표기: `ownerId`의 nickname을 찾아 **“{방장닉네임}님의 방”** 형태로 표시.

## 11. UI
- Entries 버튼: `Entries (온라인/전체)`.
- Entries 목록: 온라인 상태 점 표시.
- Room meta: `Connected as {nickname} · Auto delete {autoDeleteAt}`.
