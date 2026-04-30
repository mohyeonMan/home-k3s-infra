# Kopic Deployment Plan

## 목적

`../kopic/kopic-ge-k3s`, `../kopic/kopic-ws-k3s`를 현재 `home-k3s-infra` 구조에 맞춰 K3s + ArgoCD 환경에 배포한다.

## 현재 상태 요약

- `home-k3s-infra`는 ArgoCD가 `apps/` 하위를 동기화하는 구조다.
- 현재 최상위 배포 대상은 `sealed-secrets`, `traefik`, `monitoring`, `rabbitmq`, `kopic`이다.
- `home-k3s-infra` `main`에는 `apps/kopic`과 `kopic-app` ArgoCD `Application`이 이미 추가돼 있다.
- `kopic-ge-k3s`, `kopic-ws-k3s`, `kopic-client`에는 `Dockerfile`과 GitHub Actions 이미지 빌드 워크플로가 이미 추가돼 있다.
- 세 앱 레포는 `dev` 브랜치 push 기준으로 GHCR 이미지를 만들고, 성공 시 `home-k3s-infra` `main`의 `apps/kopic/*/kustomization.yaml` 태그를 갱신하도록 구성돼 있다.
- 현재 기준 태그는 `ge=sha-c0d9112`, `ws=sha-6ad33eb`, `client=sha-7d61c59`다.

## 배포 목표 구조

- `apps/kopic/application.yaml`
- `apps/kopic/kustomization.yaml`
- `apps/kopic/namespace.yaml`
- `apps/kopic/rabbitmq-auth-secret.yaml`
- `apps/kopic/client/*`
- `apps/kopic/ge/*`
- `apps/kopic/ws/*`

`kopic`는 별도 namespace를 두되, MQ는 기존 공용 `rabbitmq` 앱을 사용한다.

## 워크로드 설계

### 1. RabbitMQ

- `kopic`는 별도 RabbitMQ를 배포하지 않고, 기존 공용 `rabbitmq`를 사용한다.
- 접속 호스트는 `rabbitmq.rabbitmq.svc.cluster.local`을 기준으로 잡는다.
- Secret은 namespace를 넘겨 직접 참조할 수 없으므로, `kopic` namespace 안에도 `rabbitmq-auth`가 필요하다.
- 최종 목표는 `rabbitmq-auth`를 `SealedSecret`으로 두는 것이다.
- 현재 레포 상태는 임시로 plain `Secret` 파일을 두고 있으며, 이후 `SealedSecret`으로 교체해야 한다.

### 2. GE

- `kopic-ge-k3s`는 외부 Ingress 없이 내부 워커 성격으로 배포한다.
- 초기 배포는 `replicas: 1`로 둔다.
- `KOPIC_NODE_ID=ge-local`을 명시적으로 고정한다.
- 이유는 현재 클라이언트가 `geId=ge-local`을 고정값으로 사용하고 있기 때문이다.

### 3. WS

- `kopic-ws-k3s`는 WebSocket 엔드포인트를 외부에 노출한다.
- `Service + Ingress`를 구성한다.
- 현재 매니페스트 기준 외부 경로는 `/kopic/ws`다.
- `StatefulSet`으로 배포하고, 파드별 `KOPIC_NODE_ID`를 `metadata.name` 기반으로 주입한다.
- RabbitMQ 접속 정보는 공용 MQ 기준으로 `SPRING_RABBITMQ_HOST`, `SPRING_RABBITMQ_USERNAME`, `SPRING_RABBITMQ_PASSWORD`로 주입한다.
- `KOPIC_WS_ENDPOINT=/kopic/ws`를 환경변수로 주입한다.

### 4. Client

- `kopic-client`는 정적 SPA로 배포한다.
- 외부 경로는 `/kopic`이다.
- Vite build base는 `/kopic/` 기준으로 맞춘다.
- nginx는 `/kopic` 요청을 `/kopic/`으로 정리하고, `/kopic/*` 경로를 `/kopic/index.html`로 fallback 처리한다.

## 중요한 제약 사항

### 1. 이미지 빌드 경로와 브랜치 전략

- 두 `kopic-*` 레포의 이미지 빌드 경로는 이미 존재한다.
- `Dockerfile`
- `.github/workflows/build-image.yml`
- 현재 워크플로 트리거 브랜치는 `deploy`가 아니라 `dev`다.
- 성공 시 `home-k3s-infra` `main`의 `apps/kopic/ge/kustomization.yaml`, `apps/kopic/ws/kustomization.yaml` 태그를 갱신한다.

### 2. Node ID 주입 방식

- `kopic` 애플리케이션은 `nodeId`를 RabbitMQ queue/routing key 생성에 사용한다.
- 따라서 `KOPIC_NODE_ID`가 올바르게 주입되지 않으면 노드 간 메시지 라우팅이 깨진다.
- `c2c`에서 쓰는 `WS_NODE_ID` 이름을 그대로 복사하면 안 된다.

### 3. 공용 MQ credential 관리

- 공용 RabbitMQ를 사용하더라도 `rabbitmq` namespace의 Secret을 `kopic` namespace에서 직접 참조할 수는 없다.
- 따라서 `kopic` namespace에 동일 credential의 `rabbitmq-auth` Secret/SealedSecret을 별도로 둬야 한다.
- 현재는 임시 plain `Secret` 상태이며, 최종적으로는 `SealedSecret`으로 바꾸는 것이 맞다.

### 4. 클라이언트 WebSocket 주소

- 현재 서버 매니페스트는 `/kopic/ws` 경로를 사용한다.
- `kopic-client`는 `/kopic/ws` 기준으로 수정된 상태다.
- 현재 `traefik`은 기본적으로 `80/443` 기반 HTTP 진입점을 사용한다.
- 따라서 클라이언트는 상대 경로 또는 현재 origin 기반 경로로 맞추는 쪽이 자연스럽다.

## 현재 진행 상태

### 완료

- `kopic-ge-k3s` `Dockerfile` 추가
- `kopic-ws-k3s` `Dockerfile` 추가
- `kopic-client` `Dockerfile` 및 nginx 정적 서빙 설정 추가
- 세 레포에 `build-image.yml` 추가
- `kopic-ge-k3s`, `kopic-ws-k3s` `dev` 브랜치 push 및 GitHub Actions 빌드 성공 확인
- `kopic-client` `dev` 브랜치 커밋 완료
- `apps/kopic` 디렉터리 추가
- `apps/kustomization.yaml`에 `kopic/application.yaml` 등록
- `client`, `GE`, `WS`, `rabbitmq-auth`, namespace 매니페스트 추가
- 현재 이미지 태그를 `sha-c0d9112`, `sha-6ad33eb`, `sha-7d61c59` 기준으로 반영

### 남은 일

- `rabbitmq-auth`를 plain `Secret`에서 `SealedSecret`으로 교체
- `kopic-client` `dev` 브랜치 원격 push 및 GitHub Actions 빌드 확인
- ArgoCD sync 및 런타임 검증

## 초기 배포안

- namespace: `kopic`
- RabbitMQ: 기존 공용 `rabbitmq` 사용
- RabbitMQ host: `rabbitmq.rabbitmq.svc.cluster.local`
- RabbitMQ credential: `kopic` namespace의 `rabbitmq-auth`로 주입
- Client: `1 replica`
- GE: `1 replica`, `KOPIC_NODE_ID=ge-local`
- WS: `1 replica`
- Client external path: `/kopic`
- WS external path: `/kopic/ws`
- 이미지 레지스트리: `ghcr.io/mohyeonman/*`

초기 안정화 단계에서는 `GE 1개 + WS 1개`로 시작하고, 동작 확인 후 WS만 수평 확장하는 편이 안전하다.

## 후속 작업

- `rabbitmq-auth`를 `SealedSecret`으로 전환
- `kopic-client` `dev` 브랜치 push 및 GHCR 이미지 생성
- `kopic-client`에서 `geId`를 고정값이 아니라 서버/로비가 제공하는 값으로 전환
- GE 다중 replica 운영 시 `room ownership` 또는 `engine selection` 정책 정리
- readiness/liveness probe 전략 추가
- 필요 시 observability 연동

## 결론

현재는 `apps/kopic`와 세 앱의 이미지 빌드 파이프라인까지는 붙은 상태다.  
다음 핵심 작업은 `rabbitmq-auth`의 `SealedSecret` 전환, `kopic-client` 원격 push, 그리고 ArgoCD 런타임 검증이다.
