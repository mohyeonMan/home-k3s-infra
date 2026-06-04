# Kopic dev 오토스케일링 매니페스트 수정 계획

## 0. 범위

이 계획은 `home-k3s-infra`의 dev 매니페스트 선적용만 다룬다.

대상:

```text
apps/kopic/dev/ge
apps/kopic/dev/ws
apps/kopic/application-dev.yaml
```

prod 매니페스트는 이번 1차 적용 범위에서 제외한다.

`../kopic/kopic-ge-k3s`, `../kopic/kopic-ws-k3s`, `../kopic/kopic-lobby-k3s`, `../kopic/kopic-client`는 분석 목적으로만 참고하며, 이 계획에서는 수정하지 않는다.

---

## 1. 핵심 판단

GE와 WS는 StatefulSet보다 Deployment가 현재 구조에 더 맞다.

이유:

```text
- GE/WS는 PVC나 디스크 상태를 복원하지 않는다.
- room/session 상태는 프로세스 메모리 기반이다.
- StatefulSet ordinal identity는 상태 복원에 쓰이지 않는다.
- 오히려 같은 ordinal 재시작 시 KOPIC_NODE_ID가 재사용되어 stale Redis directory 문제를 만들 수 있다.
- scale-in 대상 Pod 순서를 ordinal로 고정해야 할 이유가 현재 없다.
```

따라서 dev 1차 적용은 다음 방향으로 잡는다.

```text
GE StatefulSet -> Deployment
WS StatefulSet -> Deployment
KOPIC_NODE_ID -> Pod UID를 포함한 재기동별 고유 id
HPA target -> Deployment
Deployment.spec.replicas -> manifest에서 제거
```

---

## 2. stale roomCode 문제 정리

현재 dev GE는 `KOPIC_NODE_ID`를 `metadata.name`에서 가져온다.

```yaml
- name: KOPIC_NODE_ID
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
```

StatefulSet에서는 Pod가 비정상 종료 후 재시작되어도 같은 ordinal 이름을 다시 쓴다.

예:

```text
kopic-dev-ge-0 종료
kopic-dev-ge-0 재생성
KOPIC_NODE_ID = kopic-dev-ge-0 그대로 유지
```

이때 GE 메모리에는 기존 room이 없지만 Redis에는 아래 stale mapping이 남을 수 있다.

```text
kopic-dev.roomcode:{roomCode} -> kopic-dev-ge-0
```

lobby는 `roomCode -> geId` 조회 후 `ge:{geId}=ACTIVE`이면 해당 GE로 안내한다. 재시작한 GE가 같은 geId로 `ACTIVE`를 다시 쓰면, lobby는 존재하지 않는 방을 가진 새 GE를 기존 방의 GE로 오인할 수 있다.

GE 쪽 `privateJoin()`은 로컬 roomCode index miss 시 입장 거절만 하고, 현재 구조에서는 Redis roomCode를 직접 삭제하지 않는다.

따라서 manifest 차원의 1차 방어는 `KOPIC_NODE_ID` 재사용을 막는 것이다.

---

## 3. KOPIC_NODE_ID 정책

GE와 WS 모두 Pod 재생성마다 바뀌는 id를 사용한다.

권장:

```yaml
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_UID
  valueFrom:
    fieldRef:
      fieldPath: metadata.uid
- name: KOPIC_NODE_ID
  value: "$(POD_NAME)-$(POD_UID)"
```

효과:

```text
- 같은 Deployment replica가 재생성되어도 KOPIC_NODE_ID가 바뀐다.
- stale roomCode는 old geId를 가리킨다.
- old geId heartbeat TTL이 만료되면 lobby cleanup 대상이 된다.
- 재시작된 새 GE가 기존 roomCode의 실제 소유자인 것처럼 보이지 않는다.
```

대안으로 `KOPIC_NODE_ID=$(POD_UID)`만 쓸 수도 있지만, 로그와 Redis 확인 편의를 위해 `podName-podUid` 조합을 우선한다.

---

## 4. GE 매니페스트 변경 계획

대상 파일:

```text
apps/kopic/dev/ge/deployment.yaml
apps/kopic/dev/ge/statefulset.yaml
apps/kopic/dev/ge/kustomization.yaml
apps/kopic/dev/ge/hpa.yaml
```

`deployment.yaml`은 새로 만들고, 기존 `statefulset.yaml`은 삭제하거나 kustomization에서 제외한다.

### 4-1. StatefulSet을 Deployment로 전환

`apps/kopic/dev/ge/statefulset.yaml`은 파일명을 유지할 수도 있지만, 의미가 어긋나므로 다음처럼 바꾸는 편이 낫다.

```text
statefulset.yaml 삭제 또는 미사용
deployment.yaml 추가
kustomization.yaml resources에서 statefulset.yaml -> deployment.yaml
```

Deployment 기본 형태:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kopic-dev-ge
spec:
  selector:
    matchLabels:
      app: kopic-dev-ge
  template:
    metadata:
      labels:
        app: kopic-dev-ge
```

`spec.replicas`는 의도적으로 작성하지 않는다. 실제 replica 수는 HPA가 관리한다.

### 4-2. 종료 보호 시간 명시

GE drain 정책은 최대 3600초 drain과 Spring shutdown phase 70분을 기준으로 한다.

Kubernetes가 SIGTERM 후 충분히 기다리도록 Pod spec에 추가한다.

```yaml
terminationGracePeriodSeconds: 4500
```

4500초는 70분보다 약간 긴 값이다.

### 4-3. drain 관련 env 명시

애플리케이션 기본값과 같더라도 dev manifest에 명시해 운영 의도를 드러낸다.

```yaml
- name: KOPIC_SHUTDOWN_PHASE_TIMEOUT
  value: 70m
- name: KOPIC_DRAIN_TIMEOUT
  value: 3600s
- name: KOPIC_DRAIN_POLL_INTERVAL
  value: 60s
- name: KOPIC_DRAIN_WAITING_ROOM_DELETE_DELAY
  value: 300s
- name: KOPIC_DRAIN_IN_GAME_NOTIFY_INTERVAL
  value: 180s
```

`/actuator/shutdown`은 사용하지 않는다. 종료 트리거는 Kubernetes의 Pod termination에 따른 SIGTERM이다.

### 4-4. Redis quick 보조 인덱스 prefix 추가

현재 dev GE에는 아래 값들이 있다.

```yaml
KOPIC_REDIS_KEY_GE_PREFIX: "kopic-dev.ge:"
KOPIC_REDIS_KEY_GE_LOAD: "kopic-dev.ge:load"
KOPIC_REDIS_KEY_QUICK_AVAILABLE: "kopic-dev.quick:available"
KOPIC_REDIS_KEY_ROOM_CODE_PREFIX: "kopic-dev.roomcode:"
```

하지만 GE의 quick 보조 SET prefix인 `KOPIC_REDIS_KEY_QUICK_GE_PREFIX`가 빠져 있다. 앱 기본값은 `quick:ge:`라 dev prefix와 분리되지 않는다.

추가:

```yaml
- name: KOPIC_REDIS_KEY_QUICK_GE_PREFIX
  value: "kopic-dev.quick:ge:"
```

lobby도 같은 prefix를 사용해야 한다. 현재 lobby env에는 `KOPIC_REDIS_KEY_QUICK_GE_ROOMS_PREFIX`가 없으므로 dev lobby manifest에도 같은 계열의 값을 추가하는 것이 맞다.

```yaml
- name: KOPIC_REDIS_KEY_QUICK_GE_ROOMS_PREFIX
  value: "kopic-dev.quick:ge:"
```

### 4-5. probe 정책

GE는 외부 트래픽을 Service로 직접 받는 서버가 아니다. 신규 방 생성과 quick 유입 차단은 readiness가 아니라 Redis `ACTIVE/DRAIN` 상태와 lobby 정책으로 제어한다.

1차 적용에서는 probe를 과하게 넣지 않는다.

권장:

```text
- startupProbe 또는 livenessProbe는 후속 검증 뒤 추가
- readinessProbe로 GE drain 라우팅을 제어하지 않음
```

이유:

```text
- GE에는 현재 WS처럼 ReadinessState.REFUSING_TRAFFIC publish가 확인되지 않았다.
- GE는 drain 중에도 Rabbit consumer를 유지해야 한다.
- readiness false가 Service endpoint 제외 외에 운영상 의미가 크지 않다.
```

---

## 5. WS 매니페스트 변경 계획

대상 파일:

```text
apps/kopic/dev/ws/deployment.yaml
apps/kopic/dev/ws/statefulset.yaml
apps/kopic/dev/ws/kustomization.yaml
apps/kopic/dev/ws/hpa.yaml
```

`deployment.yaml`은 새로 만들고, 기존 `statefulset.yaml`은 삭제하거나 kustomization에서 제외한다.

### 5-1. StatefulSet을 Deployment로 전환

WS도 안정 ordinal identity가 필요하지 않다. per-pod Rabbit routing id만 필요하므로 Pod UID 기반 id가 더 안전하다.

```text
statefulset.yaml 삭제 또는 미사용
deployment.yaml 추가
kustomization.yaml resources에서 statefulset.yaml -> deployment.yaml
```

### 5-2. 종료 보호 시간 명시

WS drain도 최대 3600초 동안 기존 연결을 보호한다.

```yaml
terminationGracePeriodSeconds: 4500
```

### 5-3. drain 관련 env 명시

```yaml
- name: KOPIC_SHUTDOWN_PHASE_TIMEOUT
  value: 70m
- name: KOPIC_DRAIN_TIMEOUT
  value: 3600s
- name: KOPIC_DRAIN_POLL_INTERVAL
  value: 60s
```

WS는 `waiting-room-delete-delay`, `in-game-notify-interval`을 사용하지 않는다.

### 5-4. readiness probe 추가

WS는 코드에서 drain 진입 시 `ReadinessState.REFUSING_TRAFFIC`을 publish한다.

따라서 readiness probe를 추가하는 것이 의미가 있다.

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: http
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 2
```

목적:

```text
- scale-in 또는 Pod 종료 시 readiness false 반영
- Service endpoint에서 빠르게 제외
- 새 WebSocket handshake 유입 감소
- 앱 내부 handshake 차단 로직과 중복 방어
```

---

## 6. HPA 계획

대상:

```text
apps/kopic/dev/ge/hpa.yaml
apps/kopic/dev/ws/hpa.yaml
```

1차는 native HPA resource metric으로 시작한다.

이유:

```text
- 현재 Prometheus와 ServiceMonitor는 있지만 HPA custom metric API 구성이 확인되지 않았다.
- KEDA나 prometheus-adapter를 추가하면 범위가 커진다.
- dev 선적용은 drain/HPA/ArgoCD 동작 검증이 우선이다.
```

조건은 CPU와 memory를 함께 사용한다.

```text
- CPU: 평균 사용률 기반
- memory: JVM 특성을 고려해 request 대비 사용률보다 absolute averageValue 기반을 우선 검토
```

HPA는 여러 metric을 동시에 쓰면 각 metric이 계산한 desired replica 중 가장 큰 값을 선택한다. 즉 CPU 또는 memory 중 하나라도 기준을 넘으면 scale-out 된다.

native HPA만으로는 "memory가 2Gi 이상인 상태가 5분 이상 유지되면 scale-out" 같은 Prometheus alert식 조건을 정확하게 표현하지 못한다. 1차에서는 빠른 scale-out을 우선하고, 정확한 지속시간 조건은 후속 custom metric 단계에서 검토한다.

후속에서 정확한 지속 조건이 필요하면 Prometheus Adapter 또는 KEDA로 PromQL을 사용한다.

```promql
avg_over_time(container_memory_working_set_bytes{pod=~"kopic-dev-ge.*"}[5m])
```

GE 예시:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kopic-dev-ge
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kopic-dev-ge
  minReplicas: 2
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 2Gi
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 3600
      policies:
        - type: Pods
          value: 1
          periodSeconds: 1800
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
```

WS도 동일 구조로 시작한다.

memory 기준은 초기값으로 `2Gi`를 제안한다. 현재 dev manifest의 memory request는 `1Gi`, limit은 `3Gi`이므로 `averageUtilization`을 쓰면 request 기준으로 계산되어 너무 일찍 scale-out될 수 있다.

scale 정책은 GE/WS 모두 같은 성향으로 둔다.

```text
- scale-up: 빠르게
- scale-down: 느리게
```

dev에서는 `minReplicas: 2`, `maxReplicas: 3`으로 테스트한다. 이후 리소스를 아끼는 방향으로 전환할 때 `minReplicas: 1`, `maxReplicas: 2`를 검토한다.

단, JVM memory metric은 CPU보다 보수적으로 해석해야 한다.

```text
- heap이 커진 뒤 바로 줄지 않으면 scale-down이 지연될 수 있다.
- memory leak이나 cache 증가를 autoscaling으로 해결할 수는 없다.
- memory HPA는 장애 완충 장치이지 주 부하 지표로 보지 않는다.
```

---

## 7. ArgoCD와 HPA 충돌 방지

방식 A를 사용한다.

```text
Deployment manifest에서 spec.replicas를 작성하지 않는다.
HPA가 scale subresource를 통해 실제 replica 수를 관리한다.
```

이 방식에서는 Git이 replica 실제값을 소유하지 않는다. Git에는 HPA 정책만 커밋한다.

```text
Git이 관리하는 것:
- Deployment template
- resource requests/limits
- HPA minReplicas/maxReplicas/metrics/behavior

HPA가 관리하는 것:
- Deployment의 실제 replica 수
```

검증 기준:

```text
- HPA가 replica를 조정한 뒤 ArgoCD가 OutOfSync를 계속 띄우지 않는지
- ArgoCD selfHeal이 replica 수를 되돌리지 않는지
```

만약 dev 검증에서 ArgoCD가 live `spec.replicas`를 계속 drift로 본다면, 그때만 `/spec/replicas` ignore를 추가한다. 1차 계획에는 ignore 설정을 기본으로 넣지 않는다.

---

## 8. Service / Ingress 계획

### GE

GE는 외부 Ingress가 없다.

현재 headless Service는 StatefulSet용 `serviceName` 목적이 컸다. Deployment 전환 후에는 필수는 아니다.

다만 ServiceMonitor가 Service label을 selector로 쓰므로, 1차에서는 Service를 유지한다.

선택지:

```text
1차 권장: 기존 kopic-dev-ge-headless Service 유지
후속 정리: 필요하면 일반 ClusterIP Service로 전환
```

### WS

WS는 기존 ClusterIP Service와 Ingress를 유지한다.

Deployment 전환 후 headless Service는 필수는 아니지만, 제거는 1차 목적과 직접 관련이 없다.

선택지:

```text
1차 권장: 기존 kopic-dev-ws Service 유지, headless Service는 당장 제거하지 않음
후속 정리: StatefulSet 제거 안정화 후 headless Service 삭제 검토
```

---

## 9. 적용 순서

1. GE/WS `deployment.yaml` 작성
2. GE/WS `kustomization.yaml`에서 `statefulset.yaml`을 `deployment.yaml`로 교체
3. GE/WS `hpa.yaml` 추가 후 `kustomization.yaml`에 등록
4. GE `KOPIC_REDIS_KEY_QUICK_GE_PREFIX` 추가
5. lobby dev manifest에 `KOPIC_REDIS_KEY_QUICK_GE_ROOMS_PREFIX` 추가
6. WS readiness probe 추가
7. Deployment manifest에 `spec.replicas`가 없는지 확인
8. kustomize build 또는 ArgoCD dry-run 성격의 검증 수행
9. dev ArgoCD sync
10. 필요 시에만 `apps/kopic/application-dev.yaml`에 ArgoCD replica ignore 추가
11. 다음 시나리오를 확인

검증 시나리오:

```text
- GE Pod 강제 삭제 후 새 GE id가 Redis에 생기는지
- old geId TTL 만료 후 stale roomCode가 lobby lookup에서 제거되는지
- stale roomCode로 private join 시 새 GE로 오인 라우팅되지 않는지
- WS Pod scale-in 시 readiness false와 WS_DRAIN_NOTICE가 동작하는지
- GE Pod scale-in 시 DRAIN 진입 후 ge:load/quick 후보 제거가 동작하는지
- HPA가 replicas를 조정해도 ArgoCD가 되돌리지 않는지
- ArgoCD가 replicas drift로 지속적인 OutOfSync를 만들지 않는지
- CPU 또는 memory metric 중 하나가 기준을 넘을 때 scale-out 되는지
```

---

## 10. 추가 고려사항

### 10-1. metrics-server 확인

CPU/memory 기반 HPA는 Kubernetes resource metrics API가 필요하다.

확인 대상:

```text
- metrics-server가 설치되어 있는지
- kubectl top pod가 동작하는지
- HPA status에서 cpu/memory metric을 읽는지
```

metrics-server가 없으면 HPA는 생성되더라도 metric을 읽지 못해 scale 판단을 하지 못한다.

### 10-2. minReplicas 결정

Deployment에서 `spec.replicas`를 제거하면 최초 생성 기본값은 1이다. 이후 HPA의 `minReplicas`가 하한을 결정한다.

현재 dev GE/WS는 각각 `replicas: 2`로 배포되어 있다. 1차 dev 테스트에서는 이 baseline을 유지하기 위해 HPA의 `minReplicas`를 2로 둔다.

이번 계획의 기준:

```text
minReplicas: 2
- 현재 dev replica 수와 동일한 baseline 유지
- Pod 1개 장애 중에도 다른 Pod가 남음
- maxReplicas 3에서 minReplicas 2로 내려오는 scale-in/drain을 검증

maxReplicas: 3
- 리소스 부담을 제한하면서 scale-out 동작 확인
- GE/WS 각각 한 단계 scale-out만 검증
```

후속으로 리소스를 줄이는 단계에서는 `minReplicas: 1`, `maxReplicas: 2`를 검토한다.

### 10-3. memory HPA의 한계

memory metric은 JVM 프로세스에서 scale-down을 방해할 수 있다.

```text
- JVM heap은 부하가 줄어도 즉시 OS에 반환되지 않을 수 있다.
- memory 평균값이 높게 유지되면 HPA가 replica를 줄이지 않을 수 있다.
- memory target은 OOM 방지 보조 기준으로 보고 CPU를 주 기준으로 해석한다.
```

필요하면 후속으로 JVM 옵션과 requests/limits를 같이 조정한다.

예:

```text
-XX:MaxRAMPercentage
-XX:InitialRAMPercentage
container memory request/limit 재조정
```

### 10-4. rolling update와 긴 drain

`terminationGracePeriodSeconds=4500`은 최대 대기 시간이다. 앱이 drain을 끝내고 먼저 종료하면 그 전에 끝난다.

다만 실제로 방/연결이 오래 남으면 scale-in뿐 아니라 rolling update도 그만큼 늦어질 수 있다. 이는 drain 정책상 의도한 동작으로 본다.

후속으로 Deployment strategy를 명시할 수 있다.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

1차에서는 기본 RollingUpdate로 시작해도 되지만, dev 검증에서 rollout 중 가용성 문제가 보이면 명시한다.

### 10-5. 리소스 request/limit 계산

Kubernetes resource request는 Deployment 전체에 나눠지는 값이 아니라 Pod마다 적용된다. replica 수가 늘면 request도 replica 수만큼 곱해진다.

현재 dev 기준:

```text
GE 1 Pod: cpu request 300m, memory request 1Gi, memory limit 3Gi
WS 1 Pod: cpu request 300m, memory request 1Gi, memory limit 3Gi
```

HPA `minReplicas: 2`, `maxReplicas: 3` 적용 시:

```text
기본 min 상태:
GE 2 + WS 2 = cpu request 1.2 core, memory request 4Gi

최대 scale 상태:
GE 3 + WS 3 = cpu request 1.8 core, memory request 6Gi
```

memory limit 기준 최악은 다음과 같다.

```text
GE/WS 6 Pods * 3Gi = 18Gi
```

현재 클러스터 메모리가 약 16Gi라면 request 기준으로는 충분해 보이지만, 모든 JVM이 동시에 limit에 가까워지면 memory pressure가 생길 수 있다. dev 테스트에서는 monitoring, rabbitmq, redis, lobby, client까지 포함해 실제 사용량을 확인한다.

### 10-6. PodDisruptionBudget

PDB는 이번 문서에서 자세히 다루지 않는다. dev 1차 적용에는 넣지 않고, 별도 질문 또는 prod 전환 검토 때 따로 정리한다.

---

## 11. 후속 코드 보강 후보

1차 manifest 적용과 별개로, GE 코드에 다음 방어를 추가하면 더 견고하다.

```text
privateJoin(roomCode)에서 로컬 roomCode index miss 발생
-> Redis roomCode:{roomCode} 값이 현재 geId와 일치하는지 확인
-> 일치하면 해당 Redis roomCode 삭제
```

이 보강은 stale roomCode가 GE까지 도달했을 때 빠르게 정리하는 보조 방어다.

하지만 첫 라우팅 자체를 줄이려면 manifest에서 `KOPIC_NODE_ID` 재사용을 막는 것이 우선이다.

---

## 12. 1차 결론

dev 선적용의 최소 변경 축은 다음이다.

```text
1. GE/WS를 StatefulSet에서 Deployment로 전환
2. GE/WS KOPIC_NODE_ID를 podName-podUid 기반으로 변경
3. terminationGracePeriodSeconds=4500 설정
4. drain 관련 env 명시
5. GE quick 보조 Redis prefix 누락 보완
6. lobby quick 보조 Redis prefix 누락 보완
7. WS readiness probe 추가
8. GE/WS HPA 추가
9. Deployment manifest에서 spec.replicas 제거
10. ArgoCD replica ignore는 dev 검증 후 필요할 때만 추가
11. dev HPA는 minReplicas 2, maxReplicas 3으로 시작
```

이 범위는 autoscaling과 drain 안전성에 직접 필요한 변경만 포함한다.
