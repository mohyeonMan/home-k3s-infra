# 오토스케일링 / DRAIN 정책 정리

## 0. 기본 전제

현재 구조에서는 GE를 다른 GE로 이전시키지 않는다.

scale-in 또는 Pod 종료가 발생하면 종료 대상 Pod가 `SIGTERM`을 받고, 각 서버가 자기 책임 범위 안에서 DRAIN 처리를 수행한다.

```text
GE DRAIN = game drain
- 방 / 게임을 정리하는 정책
- 새 방 생성과 quick 매칭 유입을 막음
- 기존 게임은 최대 보호 시간 안에서 마무리
- 게임 중이 아닌 방은 일정 시간 뒤 삭제

WS DRAIN = connection drain
- WebSocket 연결을 정리하는 정책
- 새 WebSocket 연결을 막음
- 기존 연결자에게 WS_DRAIN_NOTICE(452)를 보냄
- 기존 연결은 최대 보호 시간 안에서 유지
- 재연결 시점 판단은 클라이언트에 위임
```

현재 GE/WS 코드/설정에서 확인되는 공통 shutdown 기준은 다음이다.

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: ${KOPIC_SHUTDOWN_PHASE_TIMEOUT:70m}

kopic:
  drain:
    timeout: ${KOPIC_DRAIN_TIMEOUT:3600s}
    poll-interval: ${KOPIC_DRAIN_POLL_INTERVAL:60s}
    waiting-room-delete-delay: ${KOPIC_DRAIN_WAITING_ROOM_DELETE_DELAY:300s}
    in-game-notify-interval: ${KOPIC_DRAIN_IN_GAME_NOTIFY_INTERVAL:180s}
```

```text
각 값의 의미

70분
- Spring SmartLifecycle shutdown phase 전체 대기 한도

3600초
- DRAIN 진입 후 room들이 능동적으로 정리되기를 기다리는 시간

60초
- drain 완료 여부를 확인하는 poll 간격

300초
- 게임 중이 아닌 방이 삭제되기 전 countdown 시간

180초
- 게임 진행 중 drain 안내 반복 간격
```

WS는 `waiting-room-delete-delay`와 `in-game-notify-interval`을 사용하지 않는다.  
WS는 최대 3600초 동안 `activeSessions == 0`이 되기를 기다린다.

이 값은 게임의 이론상 최대 시간을 모두 보장하기 위한 값이 아니다.  
일반적인 플레이 패턴을 기준으로 둔 운영상 최대 보호 시간이다.

---

## 1. GE scale-in 정책

GE가 scale-in 또는 Pod 종료로 `SIGTERM`을 받으면 즉시 DRAIN 상태로 전환한다.

```text
ACTIVE → DRAIN
```

GE DRAIN의 목적은 다음이다.

```text
- 새 방이 더 이상 생성되지 않게 한다.
- quick 매칭 후보에서 즉시 빠진다.
- 기존 게임은 가능한 한 마무리한다.
- 게임 중이 아닌 방은 일정 시간 뒤 삭제한다.
- 모든 room이 정리되면 graceful period를 기다리지 않고 즉시 종료한다.
```

GE DRAIN 진입 시 반드시 수행해야 하는 작업은 다음이다.

```text
1. ge:{geId} = DRAIN 으로 갱신
2. ge:{geId} heartbeat TTL 계속 갱신
3. ge:load에서 자기 geId 제거
4. quick:available에서 자기 geId 후보 즉시 제거
5. 새 private room 생성 차단
6. 새 quick room 생성 차단
7. quick join 차단
8. 기존 private roomCode 입장은 허용
9. 보유 중인 room을 상태별로 정리
10. roomCount == 0 이면 즉시 정상 종료
11. graceful period 만료 전까지 남은 room은 강제 종료 후 정리
```

중요한 점은 DRAIN 중에도 `ge:{geId}` heartbeat를 유지해야 한다는 것이다.

```text
ge:{geId} = DRAIN
TTL 계속 갱신
```

GE는 아직 살아서 기존 게임과 방을 정리하고 있다.  
TTL이 끊기면 lobby나 다른 서버가 GE가 죽었다고 오판할 수 있다.

---

## 2. GE DRAIN 라우팅 정책

DRAIN 상태에서 막아야 하는 것은 “모든 입장”이 아니다.

정확히는 다음 흐름을 막는다.

```text
DRAIN 상태에서 차단

- 새 private room 생성
- 새 quick room 생성
- quick join
- quick:available 후보 노출
```

DRAIN 상태에서도 다음은 허용한다.

```text
DRAIN 상태에서 허용

- 이미 생성된 private roomCode를 통한 기존 방 입장
- 기존 연결자 유지
- 기존 진행 중 게임 유지
```

즉, DRAIN은 “기존 방 입장 차단” 상태가 아니라, **새 방 생성과 quick 매칭 후보 노출을 차단하는 상태**다.

quick 계열을 막는 이유는 quick join 과정에서 새로운 방이 생성될 수 있고, DRAIN 중인 GE가 quick 후보로 남아 있으면 종료 대상 서버로 매칭 유입이 계속 발생할 수 있기 때문이다.

반면 private roomCode 입장은 이미 존재하는 특정 방으로 들어오는 흐름이다.  
새 방을 만드는 것이 아니므로 DRAIN 상태에서도 허용한다.

정책은 다음과 같이 고정한다.

```text
GE DRAIN 상태

- 새 private room 생성: 금지
- 새 quick room 생성: 금지
- quick join: 금지
- quick:available 후보: 즉시 제거
- 기존 private roomCode 입장: 허용
- 기존 연결자: 유지
- 기존 진행 중 게임: 최대 3600초 보호
```

---

## 3. GE DRAIN 시 room 상태별 처리

GE가 DRAIN으로 전환되면 자신이 가진 모든 room을 순회한다.

room 상태에 따라 처리 방식이 달라야 한다.

### 3-1. IN_GAME room

게임이 진행 중인 방은 바로 삭제하지 않는다.

```text
IN_GAME room 처리

1. notification 이벤트(450) 발송
2. 현재 게임은 계속 진행
3. 게임 결과 화면에 들어가기 전까지 kopic.drain.in-game-notify-interval 간격으로 notification 반복 발송
4. 새 라운드 시작 여부는 기존 게임 정책에 따름
5. 게임 결과 화면 종료 시점에 현재 GE 상태를 확인
6. runtimeState.isDraining() 이면 로비 복귀 이벤트(412) 후 room closed 이벤트(451) 발송
7. runtimeState.isDraining() 이 아니면 기존 정상 게임 종료 흐름 수행
```

별도의 `drainAfterGame` 같은 room별 플래그는 사용하지 않는다.

게임 결과 화면 종료 시점에 현재 GE 상태를 확인한다.

```text
resultViewEnd(room):

clearGameAndBroadcastReturnToLobby(room)

if runtimeState.isDraining():
    broadcast roomClosed(451, r=DRAIN_GAME_ENDED, t="로비로 이동합니다.")
    requestClose(room)
    return

기존 quick restart 또는 none 흐름 수행
```

절대 놓치면 안 되는 점은 다음이다.

```text
게임 결과 화면 종료 시점에 반드시 `runtimeState.isDraining()`을 다시 확인한다.
DRAIN 진입 시점에만 판단하고 끝내면 안 된다.
```

DRAIN 진입 후 게임이 끝났다면 해당 방은 정리 대상이다.

---

### 3-2. WAITING / READY / IDLE room

게임 중이 아닌 방은 게임 종료 이벤트가 발생하지 않는다.  
따라서 별도의 삭제 스케줄이 필요하다.

```text
WAITING / READY / IDLE room 처리

1. 새 게임 시작 차단
2. countdown notification 이벤트(450) 발송
3. 남은 시간이 10초를 초과하면 1분 단위로 RoomJob follow-up 예약
4. 마지막 10초는 1초 단위로 RoomJob follow-up 예약
5. countdown 종료 시 room closed 이벤트(451, r=DRAIN_WAITING_ROOM) 발송
6. requestClose로 room actor/session 정리
```

이 상태의 방에서 절대 허용하면 안 되는 것은 새 게임 시작이다.

```text
DRAIN + WAITING/READY/IDLE room

- private roomCode 입장: 허용 가능
- 새 게임 시작: 금지
- quick 후보 등록: 금지
- kopic.drain.waiting-room-delete-delay 이후 삭제: 유지
```

private roomCode로 새 사용자가 들어오더라도 해당 방이 삭제 예정이라는 상태는 유지된다.

현재 GE 코드 기준으로 입장 응답(e:304)의 `RoomSnapshot`에는 drain 상태 필드가 없다.

클라이언트는 notification 이벤트(450)는 공지로 표시하고, room closed close frame을 받으면 현재 방 종료로 처리한다.


---

### 3-3. EMPTY room

참여자가 없는 방은 즉시 삭제한다.

```text
EMPTY room 처리

1. roomCode 제거
2. quick 후보 제거
3. room 삭제
4. roomCount 갱신
```

---

### 3-4. RESULT / GAME_ENDED room

이미 게임이 끝났고 결과 화면만 남은 방은 별도 drain phase를 추가하지 않는다.

```text
RESULT / GAME_ENDED room 처리

1. startDrain 시점에는 game != null 이므로 in-game drain 분기로 들어간다.
2. game.isGameResult() 이면 반복 notification은 보내지 않는다.
3. 기존 GAME_RESULT_TIMER_KEY가 만료되면 resultViewEnd가 실행된다.
4. resultViewEnd에서 로비 복귀 이벤트(412)를 발송한다.
5. runtimeState.isDraining() 이면 room closed 이벤트(451, r=DRAIN_GAME_ENDED)를 발송한다.
6. requestClose로 room actor/session을 정리한다.
```

---

## 4. Redis directory 정책

Redis directory는 다음 구조를 기준으로 한다.

```text
ge:{geId}
- GE 상태
- ACTIVE / DRAIN
- TTL 기반 heartbeat

ge:load
- 활성 GE의 부하 정보
- 새 방 생성 후보 판단에 사용

quick:available
- quick join 대상 ZSET
- member = {geId}:{roomId}
- score = joinedAvailableAt

quick:ge:{geId}:rooms
- 특정 GE가 quick 후보로 올린 roomId 목록
- SET
- member = {roomId}

roomCode:{roomCode}
- private roomCode가 위치한 geId
```

GE DRAIN 진입 시 Redis 정리 정책은 다음이다.

```text
ge:{geId}
- 삭제하지 않음
- status = DRAIN 으로 유지
- TTL heartbeat 계속 갱신

ge:load
- 자기 geId 제거

quick:available
- 자기 geId에 속한 후보 즉시 제거

quick:ge:{geId}:rooms
- 자기 quick 후보 제거 후 삭제

roomCode:{roomCode}
- DRAIN 진입 시 즉시 삭제하지 않음
```

`roomCode:{roomCode}`를 DRAIN 진입 시 삭제하지 않는 이유는 기존 private roomCode 입장을 허용하기 때문이다.

roomCode 조회 흐름은 다음과 같다.

```text
roomCode:{roomCode} 조회
→ geId 확인
→ ge:{geId} ACTIVE
→ 정상 입장

roomCode:{roomCode} 조회
→ geId 확인
→ ge:{geId} DRAIN
→ 기존 private roomCode 입장 허용
→ 현재 GE 입장 응답에는 별도 drain 상태 필드 없음

roomCode:{roomCode} 조회
→ geId 확인
→ ge:{geId} 없음
→ 입장 실패 또는 종료 안내
```

---

## 5. quick:available 정리 방식

`quick:available`은 ZSET이다.

```text
quick:available
ZSET
member = {geId}:{roomId}
score = joinedAvailableAt
```

DRAIN 중에는 quick join이 금지되므로, GE는 자기 후보를 즉시 제거해야 한다.

특정 geId 후보를 빠르게 제거하기 위해 보조 인덱스를 둔다.
현재 구현은 보조 SET 기반 제거 방식을 사용한다.
단, 보조 SET이 비어 있거나 없으면 `quick:available` prefix scan을 fallback으로 사용한다.

```text
quick:ge:{geId}:rooms
SET
member = {roomId}
```

quick 후보 등록 시:

```text
ZADD quick:available {score} {geId}:{roomId}
SADD quick:ge:{geId}:rooms {roomId}
```

quick 후보 제거 시:

```text
ZREM quick:available {geId}:{roomId}
SREM quick:ge:{geId}:rooms {roomId}
```

DRAIN 진입 시:

```text
SMEMBERS quick:ge:{geId}:rooms
각 roomId에 대해 ZREM quick:available {geId}:{roomId}
DEL quick:ge:{geId}:rooms
```

이 정리는 GE가 자기 책임으로 수행한다.

절대 놓치면 안 되는 점은 다음이다.

```text
DRAIN 진입 후 quick:available에 자기 후보가 남아 있으면 안 된다.
```

---

## 6. lobby 정책

lobby는 Kubernetes replica를 직접 수정하지 않는다.  
lobby의 역할은 Redis directory를 보고 적절한 GE를 안내하는 것이다.

lobby는 다음 정책을 따른다.

```text
새 private room 생성
- ge:load 조회
- ge:{geId} 값 확인
- ACTIVE GE만 후보로 사용
- DRAIN GE는 제외

새 quick room 생성
- ACTIVE GE만 후보로 사용
- DRAIN GE는 제외

quick join
- quick:available에서 후보 조회
- 후보의 geId 상태 확인
- DRAIN GE 후보는 사용하지 않음
- DRAIN 후보가 발견되면 제거 또는 skip

private roomCode 입장
- roomCode:{roomCode}로 geId 조회
- ge:{geId} 상태 확인
- ACTIVE면 정상 입장
- DRAIN이어도 기존 방 입장은 허용
- geId가 없으면 입장 실패 또는 종료 안내
```

lobby가 절대 놓치면 안 되는 점은 다음이다.

```text
DRAIN GE를 새 방 생성 후보로 쓰면 안 된다.
DRAIN GE를 quick join 후보로 쓰면 안 된다.
private roomCode 입장만 예외적으로 허용한다.
```

---

## 7. WS scale-in 정책

WS scale-in은 GE scale-in과 의미가 다르다.

```text
WS DRAIN = connection drain
GE DRAIN = game drain
```

WS는 게임 상태를 판단하지 않는다.  
다만 현재 구조에서는 `WS 연결 = 방 참여 세션`이므로, WS 재연결은 기존 참여자 복구가 아니라 새 참여자로 처리될 수 있다.

따라서 WS는 클라이언트에게 현재 WS가 종료 예정임을 알리되, 기존 연결은 GE와 같은 운영상 최대 보호 시간 안에서 유지한다.  
재연결 시점 판단은 클라이언트에 둔다.

WS가 `SIGTERM`을 받으면 다음을 수행한다.

```text
1. WS 상태를 DRAIN으로 전환
2. readiness false 처리
3. 새 WebSocket handshake 차단
   - MetadataInterceptor에서 503 SERVICE_UNAVAILABLE 반환
   - handshake race로 연결이 성립되면 WsConnHandler에서 GE join dispatch 전에 1012로 close
4. 기존 client에게 WS_DRAIN_NOTICE(452) 발송
5. 기존 연결은 즉시 끊지 않음
6. activeSessions == 0 또는 drain timeout까지 유지
7. WS_DRAIN_NOTICE(452) 수신 후 클라이언트가 적절한 시점에 나가기 + 입장 수행
8. WebSocket close 발생 시 roomId가 있는 세션만 GE에 disconnect 이벤트(102) 전송
```

GE에서 room closed 이벤트(451)가 오면 WS는 이를 클라이언트에 TextMessage로 전달하지 않는다.

```text
GE -> WS: 451 room closed 수신

1. payload.r 로 room closed 사유 확인
2. WsSession.roomId = null 로 변경
3. client WebSocket close
   - close code = 4001
   - close reason = ROOM_CLOSED:{reason}
4. afterConnectionClosed 실행
5. roomId == null 이므로 GE에 disconnect 이벤트(102)를 보내지 않음
6. sessionRegistry에서 세션 제거
```

현재 WS 구현의 `WS_DRAIN_NOTICE`는 다음과 같다.

```json
{
  "e": 452
}
```

payload는 없다.  
사용자 안내 문구가 아니라 클라이언트 내부 상태 전환 트리거다.

WS 종료 순서는 Spring `SmartLifecycle` phase로 제어한다.

```text
높은 phase가 먼저 stop 된다.

WsDrainService
- phase = SmartLifecycle.DEFAULT_PHASE - 512
- Boot web server graceful shutdown phase보다 먼저 stop
- WsRuntimeState ACTIVE -> DRAIN
- readiness REFUSING_TRAFFIC 전환
- 기존 세션에 WS_DRAIN_NOTICE(452) 발송
- activeSessions == 0 또는 drain timeout까지 대기

Rabbit listener container
- phase = 0
- WsDrainService callback 이후 stop
- drain 중에는 GE -> WS 이벤트 수신 유지
```

WS가 하지 않는 일은 다음이다.

```text
- GE에 세션별 재연결 가능 여부를 주기적으로 묻지 않는다.
- client 상태를 대신 판단하지 않는다.
- 게임 중인 client를 임의로 새 WS에 붙이지 않는다.
- drain timeout 도달 시 GE room을 강제로 닫지 않는다.
```

WS는 게임 상태를 판단하지 않지만, 기존 참여 연결은 최대 보호 시간 안에서 유지한다.

---

## 8. 클라이언트 재연결 정책

클라이언트는 서버 종료 관련 이벤트를 받으면 기존 입장/연결 흐름을 최대한 재사용한다.

현재 client는 별도의 drain 상태 머신을 두지 않는다.  
WS drain 이후 다시 입장해야 하는지만 최소 상태로 기록한다.

```ts
type SessionState = {
  wsDrainRejoinPending: boolean;
};
```

방 종료 / 재입장 판단 우선순위는 다음과 같다.

```text
room closed close frame(4001, ROOM_CLOSED:*)
> WS_DRAIN_NOTICE
> 일반 network close
```

notification 이벤트(450)는 단순 공지 이벤트다.  
방 종료나 재입장 판단 우선순위에 넣지 않는다.

### WS_DRAIN_NOTICE 수신

WS_DRAIN은 현재 연결 서버가 종료 예정이므로, 클라이언트가 적절한 시점에 나가기 + 입장을 다시 수행하라는 신호다.

WS_DRAIN_NOTICE는 payload 없이 이벤트 코드만 전달된다.

```json
{
  "e": 452
}
```

```text
WS_DRAIN_NOTICE 수신

- 사용자에게 자체 notification 메시지를 표시한다.
- wsDrainRejoinPending = true
- room closed close frame을 이미 받았으면 사용자 행동은 변경하지 않음
- roomState == LOBBY 이면 기존 입장 흐름으로 나가기 + 입장을 수행
- roomState == RUNNING / RESULT 이면 LOBBY가 될 때까지 대기
- private room이면 현재 roomCode로 private join
- quick/random room이면 quick join
- quick/random room은 같은 방으로 돌아간다는 보장을 하지 않는다.
```

### notification 이벤트(450) 수신

notification 이벤트는 GE가 사용자에게 실시간 공지 문구를 전달하는 신호다.

```text
notification 이벤트(450) 수신

- payload.t 를 사용자 안내 문구로 사용
- local notification 메시지로 표시
- room / session / connection 상태는 변경하지 않음
- drain 상태를 추론하지 않음
```

### room closed close frame 수신

room closed close frame은 현재 방이 정리되는 명령이다.  
반드시 수용한다.

```text
room closed close frame 수신

- close code = 4001
- close reason 으로 종료 사유 확인
  - DRAIN_GAME_ENDED
  - DRAIN_WAITING_ROOM
  - DRAIN_FORCE_CLOSE
- 현재 방은 종료된 것으로 처리
- 로비 이동 / 연결 종료 안내 흐름을 수행
- WS_DRAIN_NOTICE로 예약된 나가기 + 입장보다 우선
- 자동 재입장을 수행하지 않음
```

정책적으로 중요한 점은 다음이다.

```text
WS_DRAIN은 적절한 시점에 나가기 + 입장을 예약하는 신호다.
room closed close frame은 방/게임 종료 명령이다.
```

따라서 둘이 동시에 발생하면 room closed close frame이 우선한다.

```text
이미 room closed 상태이면 이후 WS_DRAIN_NOTICE는 사용자 행동을 바꾸지 않는다.
이미 WS_DRAIN 상태였더라도 room closed close frame을 받으면 room closed 정책으로 덮어쓴다.
```

---

## 9. WS와 GE가 동시에 scale-in 되는 경우

동시에 scale-in이 발생하면 클라이언트가 두 종류의 이벤트를 받을 수 있다.

```text
WS_DRAIN_NOTICE
- 현재 연결 서버가 종료 예정
- payload 없는 나가기 + 입장 유도 이벤트

notification 이벤트(450) / room closed 이벤트(451)
- notification 이벤트(450)는 공지
- room closed 이벤트(451)는 현재 방 또는 게임 종료 명령
- 451은 WS가 close frame(4001, ROOM_CLOSED:*)으로 변환
```

동시 발생 시 정책은 다음이다.

```text
1. room closed close frame이 있으면 room closed가 우선한다.
2. 사용자는 방/게임 종료 흐름을 따른다.
3. WS_DRAIN_NOTICE는 room closed가 없을 때만 나가기 + 입장 예약으로 처리한다.
4. WS_DRAIN_NOTICE를 받은 뒤 roomState가 LOBBY가 되면 기존 입장 흐름을 다시 수행한다.
```

사용자에게는 WS와 GE를 모두 설명하지 않는다.  
사용자 안내 기준은 “현재 방/게임이 유지되는가”다.

```text
WS만 DRAIN
- 게임은 유지된다.
- WS_DRAIN_NOTICE를 받으면 notification으로 안내한다.
- 게임 중이면 LOBBY가 될 때까지 기다린 뒤 나가기 + 입장한다.
- 게임 중이 아니면 즉시 나가기 + 입장한다.

GE DRAIN
- 현재 방 또는 게임이 정리될 수 있다.
- notification 이벤트(450)는 안내로 처리한다.
- room closed close frame은 로비 이동 명령으로 처리한다.

WS + GE DRAIN
- room closed close frame을 사용자 행동 기준으로 우선 처리한다.
- room closed가 없으면 WS_DRAIN_NOTICE에 따라 LOBBY 도달 후 나가기 + 입장한다.
```

---

## 10. RabbitMQ 처리

GE는 DRAIN 상태에서도 기존 게임과 방을 정리해야 한다.  
따라서 RabbitMQ consumer를 바로 닫으면 안 된다.

```text
GE DRAIN 진입

- RabbitMQ consumer 유지
- 기존 room/game 이벤트 처리 유지
- 기존 private roomCode 입장 처리 유지
- 새 방 생성/quick 계열 요청은 거부
```

구현은 `@PreDestroy`에 의존하지 않는다.
`@PreDestroy`는 Redis/Rabbit lifecycle stop 이후에 실행될 수 있으므로, Redis 상태 갱신이나 Rabbit을 통한 정리 작업의 시작점으로 쓰면 안 된다.

GE 종료 순서는 Spring `SmartLifecycle` phase로 제어한다.

```text
높은 phase가 먼저 stop 된다.

GeDrainService
- phase = 100000
- SIGTERM/shutdown 시 가장 먼저 DRAIN 전환
- GeRuntimeState ACTIVE -> DRAIN
- GeStateRecorder.heartbeat() 1회 호출로 ge:{geId} = DRAIN 기록
- GeStateRecorder.reportLoad() 1회 호출로 ge:load에서 자기 geId 제거
- quick:available에서 자기 GE 후보 제거
- 모든 room에 drain RoomJob 발행
- drain 완료 조건을 기다린 뒤 callback 호출

Rabbit listener container
- phase = 0
- GeDrainService callback 이후 stop
- drain 중에는 consumer 유지
```

WS 종료 순서도 Spring `SmartLifecycle` phase로 제어한다.

```text
WsDrainService
- phase = SmartLifecycle.DEFAULT_PHASE - 512
- Boot web server graceful shutdown phase보다 먼저 stop
- 새 handshake 차단
- 기존 세션에 WS_DRAIN_NOTICE(452) 발송
- activeSessions == 0 또는 drain timeout까지 대기

Rabbit listener container
- phase = 0
- WsDrainService callback 이후 stop
- drain 중에는 GE -> WS 이벤트 수신 유지
```

관련 설정은 다음을 기준으로 둔다.

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: ${KOPIC_SHUTDOWN_PHASE_TIMEOUT:70m}

kopic:
  drain:
    timeout: ${KOPIC_DRAIN_TIMEOUT:3600s}
    poll-interval: ${KOPIC_DRAIN_POLL_INTERVAL:60s}
    waiting-room-delete-delay: ${KOPIC_DRAIN_WAITING_ROOM_DELETE_DELAY:300s}
    in-game-notify-interval: ${KOPIC_DRAIN_IN_GAME_NOTIFY_INTERVAL:180s}
```

현재 구현은 `GeDrainService.stop()`에서 room을 직접 정리하지 않는다.
각 room에 drain RoomJob을 발행한 뒤, 최대 3600초 동안 60초 간격으로 roomCount만 감시한다.

```text
구현된 순서

1. GeDrainService stop 시작
2. GeRuntimeState ACTIVE -> DRAIN
3. GeStateRecorder.heartbeat() / reportLoad() 호출
4. quick 후보 제거
5. 모든 room에 drain RoomJob 발행
6. roomCount == 0 또는 drain timeout까지 1분 간격 감시
7. timeout이면 남은 room에 forceCloseAll 호출
   - 내부적으로 closeWithNotice RoomJob 발행
   - room closed 이벤트(451, r=DRAIN_FORCE_CLOSE) 발송 후 requestClose
8. GeDrainService callback 호출
9. Rabbit listener stop 시작
10. Redis/Rabbit connection 종료
```

room 종료 시:

```text
- 사용자에게 종료/로비 이동 이벤트 발송
- room actor/session close
- roomCode 정리
- quick 후보 정리
- roomCount 갱신
```

최종 종료 시:

```text
- 남은 room 강제 종료
- 남은 메시지 처리 또는 폐기 정책 적용
- consumer close
- publisher close
- Redis 정리
- 프로세스 종료
```

죽은 GE나 WS로 stale 메시지가 오래 쌓이지 않도록 queue 정책도 필요하다.

```text
권장
- queue auto-delete
- message TTL
- dead-letter 정책
```

---

## 11. 종료 이벤트 정의

현재 DRAIN 관련으로 발송하는 이벤트는 다음 세 가지다.

```text
450 = notification
451 = room closed
452 = WS_DRAIN_NOTICE
```

450/451은 GE가 발송한다.  
452는 WS가 발송한다.

### notification 이벤트(450)

GE가 사용자에게 실시간 공지 / 안내 문구를 보낼 때 사용한다.

```json
{
  "t": "이 방은 게임종료 후 삭제됩니다."
}
```

의미:

```text
- 안내 이벤트다.
- 이 이벤트만으로 room actor가 닫히지는 않는다.
- payload.t 를 사용자 안내 문구로 사용한다.
```

notification 이벤트(450)가 발송되는 대표 상황:

```text
- 일반 실시간 공지

- 게임 진행 중 반복 안내
  - "이 방은 게임종료 후 삭제됩니다."

- 게임 중이 아닌 방 countdown 안내
  - "{n}분 후 방이 삭제될 예정입니다."
  - "{n}초 후 방이 삭제될 예정입니다."

- DRAIN 상태에서 새 게임 시작 시도 차단 안내
  - "서버 종료 준비 중이라 새 게임을 시작할 수 없습니다."
```

---

### room closed 이벤트(451)

GE가 room을 실제로 정리할 때 WS에 발송한다.

```json
{
  "r": "DRAIN_WAITING_ROOM",
  "t": "로비로 이동합니다."
}
```

의미:

```text
- 종료 이벤트다.
- payload.r 은 종료 사유다.
- payload.t 는 사용자 안내 문구다.
- WS는 이 이벤트를 client TextMessage로 전달하지 않는다.
- WS는 payload.r 을 close reason으로 변환한 뒤 client WebSocket을 닫는다.
- payload.r 이 없거나 비어 있으면 UNKNOWN을 사용한다.
- 이벤트 발송 후 RoomJob은 requestClose를 반환한다.
```

종료 사유:

```text
DRAIN_GAME_ENDED
- 게임 결과 화면 종료 후 drain 상태라 방을 닫는 경우
- 게임 중 참가자 부족으로 게임을 정리했고 drain 상태라 방을 닫는 경우

DRAIN_WAITING_ROOM
- 게임 중이 아닌 방의 countdown이 끝나 방을 닫는 경우

DRAIN_FORCE_CLOSE
- GeDrainService drain timeout 이후 forceCloseAll로 남은 방을 닫는 경우
```

실제 payload 필드:

```json
{
  "r": "DRAIN_GAME_ENDED | DRAIN_WAITING_ROOM | DRAIN_FORCE_CLOSE",
  "t": "안내 문구"
}
```

WS 처리:

```text
- room closed 이벤트(451)를 받으면 WsSession.roomId를 null로 변경한다.
- payload.r 값을 그대로 close reason에 사용한다.
- payload.r 이 없거나 비어 있으면 UNKNOWN을 사용한다.
- client WebSocket을 close code 4001, reason ROOM_CLOSED:{payload.r} 로 닫는다.
- afterConnectionClosed에서는 roomId == null 이므로 GE에 102를 다시 보내지 않는다.
```

클라이언트 처리:

```text
- close code 4001과 reason ROOM_CLOSED:* 를 받으면 현재 방은 종료된 것으로 처리한다.
- reason 으로 세부 사유를 구분한다.
- 로비 이동 흐름을 수행한다.
```

---

### WS_DRAIN_NOTICE 이벤트(452)

WS가 DRAIN에 진입했으며, 클라이언트가 적절한 시점에 나가기 + 입장을 다시 수행해야 함을 알릴 때 사용한다.

```json
{
  "e": 452
}
```

의미:

```text
- payload는 없다.
- client는 자체 notification 메시지를 만들어 사용자에게 안내한다.
- client는 wsDrainRejoinPending을 true로 설정한다.
- roomState가 LOBBY이면 기존 입장 흐름으로 즉시 나가기 + 입장을 수행한다.
- roomState가 RUNNING / RESULT이면 LOBBY가 될 때까지 대기한다.
- private room은 현재 roomCode로 private join한다.
- quick/random room은 quick join한다.
- quick/random room은 같은 방으로 돌아간다는 보장이 없다.
```

---

## 12. 절대 놓치면 안 되는 구현 포인트

### GE

```text
1. SIGTERM/shutdown 수신 시 GeDrainService(SmartLifecycle phase=100000)가 먼저 실행
2. GeRuntimeState를 ACTIVE -> DRAIN으로 전환
3. ge:{geId}=DRAIN 으로 갱신
4. DRAIN 중 heartbeat 유지
5. ge:load에서 자기 geId 제거
6. quick:ge:{geId}:rooms 보조 인덱스로 quick:available의 자기 후보 즉시 제거
7. 새 private room 생성 차단
8. 새 quick room 생성 차단
9. quick join 차단
10. 기존 private roomCode 입장 허용
11. Rabbit listener는 GeDrainService callback 이후 종료되도록 낮은 phase 유지
12. IN_GAME room에는 notification 이벤트(450) 발송
13. resultViewEnd와 참가자 부족 정리 흐름에서 runtimeState.isDraining() 재확인
14. WAITING/READY/IDLE room은 waiting-room-delete-delay 후 삭제 countdown 등록
15. DRAIN 상태 대기방에서는 새 게임 시작 차단
16. EMPTY room은 즉시 삭제
17. roomCount == 0 이면 즉시 프로세스 정상 종료
18. drain timeout 만료 시 forceCloseAll로 남은 room에 room closed 이벤트(451, r=DRAIN_FORCE_CLOSE) 발송
19. Redis/Rabbit 의존 작업은 @PreDestroy가 아니라 drain phase에서 먼저 수행
```

### WS

```text
1. SIGTERM 수신 즉시 DRAIN 전환
2. readiness false 처리
3. 새 WebSocket handshake 차단
   - drain 중 handshake는 503 SERVICE_UNAVAILABLE
   - handshake race로 연결이 성립되면 GE join dispatch 전 1012 close
4. 기존 client에게 WS_DRAIN_NOTICE(452, payload 없음) 발송
5. activeSessions == 0 또는 drain timeout까지 대기
6. GE에 세션 상태를 주기적으로 묻지 않음
7. 재연결 시점 판단은 client에 위임
8. 기존 연결은 즉시 끊지 않음
9. WebSocket close 발생 시 roomId가 있는 세션만 GE에 disconnect 이벤트(102) 전송
10. GE room closed 이벤트(451)는 client TextMessage로 전달하지 않고 close frame으로 변환
11. 451 처리 시 WsSession.roomId를 null로 변경해 102 재전송을 막음
12. Rabbit listener는 WsDrainService callback 이후 종료되도록 낮은 phase 유지
13. 비정상 종료에 대비해 GE의 stale session 정리 필요
```

### Client

```text
1. 별도 drain 상태 머신을 만들지 않고 wsDrainRejoinPending만 유지
2. notification 이벤트(450)는 payload.t를 local notification으로 표시
3. notification 이벤트(450)로 drain 상태를 추론하지 않음
4. WS_DRAIN_NOTICE(452)를 받으면 자체 notification 메시지 표시
5. WS_DRAIN_NOTICE(452)를 받으면 wsDrainRejoinPending=true
6. roomState가 RUNNING / RESULT이면 LOBBY가 될 때까지 나가기 + 입장을 지연
7. roomState가 LOBBY이면 기존 입장 흐름으로 나가기 + 입장 수행
8. private room은 현재 roomCode로 private join
9. quick/random room은 quick join하며 같은 방 보장은 하지 않음
10. room closed close frame(4001, ROOM_CLOSED:*)이 WS_DRAIN보다 우선
11. room closed close frame은 방/게임 종료 명령으로 반드시 수용
12. room closed close frame 이후 자동 재입장은 수행하지 않음
```

### Lobby

```text
1. ACTIVE GE만 새 방 생성 후보로 사용
2. ACTIVE GE만 quick 후보로 사용
3. DRAIN GE는 quick join 후보에서 제외
4. private roomCode 입장은 DRAIN GE라도 허용 가능
5. ge:{geId} 없음이면 입장 실패 또는 종료 안내
6. 현재 GE 입장 응답에는 별도 drain 상태 필드 없음
```

---

## 13. 최종 요약

```text
GE scale-in
- 게임 서버 정리 정책
- 새 방 생성과 quick 매칭 차단
- 기존 private roomCode 입장은 허용
- 게임 중 방은 게임 종료 후 삭제
- 게임 중이 아닌 방은 waiting-room-delete-delay 후 삭제
- 모든 room이 정리되면 즉시 종료

WS scale-in
- 연결 서버 정리 정책
- 새 WebSocket 연결 차단
- 기존 연결자에게 WS_DRAIN_NOTICE(452, payload 없음) 발송
- 기존 참여 연결은 최대 보호 시간 안에서 유지
- activeSessions == 0 또는 drain timeout까지 대기
- 재연결 판단은 client에 위임
- WS는 GE에 세션 상태를 묻지 않음

Client
- room closed close frame(4001, ROOM_CLOSED:*)이 WS_DRAIN보다 우선
- WS_DRAIN_NOTICE(452)는 자체 notification을 표시하고 나가기 + 입장을 예약
- WS_DRAIN_NOTICE(452) 이후 나가기 + 입장은 roomState가 LOBBY일 때만 수행
- notification 이벤트(450)는 단순 공지
- room closed close frame은 방/게임 종료 명령
- quick/random 재입장은 같은 방 보장을 하지 않음

핵심
- GE는 방과 게임을 정리한다.
- WS는 연결 종료 예정 상태를 알리고 기존 참여 연결을 보호한다.
- Client는 room closed close frame을 우선하고, WS_DRAIN_NOTICE 이후 LOBBY 도달 시 기존 입장 흐름을 다시 수행한다.
```
