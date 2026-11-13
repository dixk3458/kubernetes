# Kubernetes Liveness & Readiness Probe — 장애 재현 및 복구

## 1. 실습 개요

Kubernetes의 Liveness Probe와 Readiness Probe가 장애 상황에서 어떻게 다르게 동작하는지 확인하기 위한 실습입니다.

nginx 컨테이너에서 HTTP 검사 실패를 의도적으로 발생시키고, Kubernetes의 상태 변화와 복구 동작을 관찰했습니다.

단순히 Probe를 설정하는 것뿐 아니라 **장애 발생 → 원인 확인 → Kubernetes 반응 분석 → 복구 검증** 과정을 학습하는 것이 목적입니다.

### 실습 목표

- Liveness Probe와 Readiness Probe의 역할 구분
- Liveness 실패에 따른 컨테이너 재시작 확인
- Readiness 실패에 따른 Pod NotReady 상태 확인
- Service와 EndpointSlice를 이용한 트래픽 대상 변화 검증
- Events, Logs, Restart Count를 이용한 장애 분석

## 2. 실습 환경

| 항목               | 환경                        |
| ------------------ | --------------------------- |
| OS                 | Rocky Linux 9.8             |
| Kubernetes         | v1.36.3                     |
| Namespace          | `probe-resource-lab`        |
| Container Image    | `nginx:1.28-alpine`         |
| Liveness 실습      | `liveness-http-demo` Pod    |
| Readiness 실습     | `readiness-demo` Deployment |
| Readiness Replicas | 2                           |
| Service            | `readiness-demo-svc`        |
| Service Type       | ClusterIP                   |

기존 Kubernetes 클러스터에서 독립적인 실습 리소스를 이용했습니다.

---

## 3. 실습 1 — Liveness Probe 장애 재현

### 3.1 Liveness Probe 구성

**Manifest:** `liveness-http-demo.yaml`

nginx 컨테이너에 HTTP Liveness Probe를 구성했습니다.

| 설정           | 값            |
| -------------- | ------------- |
| 검사 경로      | `/index.html` |
| 검사 포트      | 80            |
| 최초 검사 지연 | 5초           |
| 검사 주기      | 5초           |
| 타임아웃       | 2초           |
| 실패 임계 횟수 | 3회           |

Liveness Probe는 애플리케이션이 계속 실행 가능한 상태인지 판단하기 위한 검사입니다.

설정된 실패 횟수에 도달하면 kubelet이 컨테이너 재시작을 수행합니다.

### 3.2 장애 재현

실행 중인 nginx 컨테이너에서 `/usr/share/nginx/html/index.html` 파일을 삭제했습니다.

이 파일은 Liveness Probe가 검사하는 `/index.html` 경로에 대응합니다.

파일 삭제 후 nginx는 해당 요청에 정상적인 HTTP 응답을 제공하지 못했습니다.

### 3.3 장애 증상 및 원인 확인

다음 명령어를 이용해 Pod의 상태와 이벤트를 확인했습니다.

- `kubectl get pod liveness-http-demo -n probe-resource-lab`
- `kubectl describe pod liveness-http-demo -n probe-resource-lab`
- `kubectl logs liveness-http-demo -n probe-resource-lab --previous`

실습 과정에서 HTTP 404 응답, Liveness Probe 실패 및 컨테이너 재시작을 확인했습니다.

이전 컨테이너 로그에서도 HTTP 404 응답 관련 기록을 확인했습니다.

이를 통해 Liveness 검사 대상인 `/index.html`이 정상 응답하지 않는 상황에서 kubelet이 컨테이너를 재시작한다는 사실을 관찰했습니다.

### 3.4 복구 결과

컨테이너 재시작 후 nginx 이미지에 포함된 기본 파일을 다시 사용할 수 있게 되었고, Liveness Probe가 정상적으로 수행되는 상태를 확인했습니다.

이번 실습에서 확인한 핵심은 다음과 같습니다.

- Liveness Probe는 컨테이너의 생존 상태를 확인합니다.
- 설정된 실패 임계 횟수에 도달하면 kubelet이 재시작을 수행합니다.
- Probe 실패 원인은 Pod Events와 이전 컨테이너 로그를 이용해 조사할 수 있습니다.
- 재시작 후에도 원인이 해결되지 않으면 동일한 장애가 반복될 수 있습니다.

### 3.5 재시작 횟수 해석 시 주의사항

이후 Pod YAML을 조회했을 때 누적 Restart Count는 5였으며, 가장 최근 종료 상태는 `reason: Unknown`, `exitCode: 255`로 표시되었습니다.

따라서 **누적 재시작 5회가 모두 Liveness Probe 실패 때문이라고 해석하지 않았습니다.**

Liveness 장애로 인한 재시작은 해당 실습 당시 관찰한 Events와 이전 로그를 근거로 판단했습니다.

운영 환경에서도 Restart Count 숫자만으로 원인을 확정하지 않고, 종료 시점과 Events, 로그, Node 상태를 함께 확인해야 합니다.

---

## 4. 실습 2 — Readiness Probe 장애 재현

### 4.1 Readiness Probe 구성

**Manifest:** `readiness-demo.yaml`

nginx Deployment를 Replica 2개로 생성하고 HTTP Readiness Probe를 구성했습니다.

| 설정           | 값        |
| -------------- | --------- |
| Replicas       | 2         |
| 검사 경로      | `/readyz` |
| 검사 포트      | 80        |
| 최초 검사 지연 | 3초       |
| 검사 주기      | 5초       |
| 타임아웃       | 2초       |
| 실패 임계 횟수 | 2회       |

nginx 기본 이미지에는 `/readyz` 경로가 별도로 구성되어 있지 않기 때문에, 실습에서는 `lifecycle.postStart`를 이용해 파일을 생성하도록 설정했습니다.

사용한 명령어는 다음과 같습니다.

`echo ready > /usr/share/nginx/html/readyz`

이 설정은 Probe 동작을 확인하기 위한 실습용 구성입니다.

실제 운영에서는 애플리케이션이 요청을 처리할 준비가 되었는지 확인할 수 있는 엔드포인트를 사용하는 것이 적절합니다.

### 4.2 Service 구성

**Manifest:** `readiness-demo-service.yaml`

ClusterIP Service를 생성하고 다음 Label을 가진 Pod를 선택하도록 설정했습니다.

`app: readiness-demo`

Service와 Pod의 연결 포트는 모두 TCP 80으로 설정했습니다.

### 4.3 장애 재현

실행 중인 nginx 컨테이너에서 Readiness 검사 대상인 `/readyz`가 정상적으로 응답하지 못하는 상황을 재현했습니다.

Readiness Probe가 실패하면서 해당 Pod는 NotReady 상태가 되었습니다.

그러나 **Readiness 실패 자체로 컨테이너가 재시작되지는 않았습니다.**

### 4.4 장애 상태 확인

다음 명령어를 사용해 Pod의 준비 상태와 이벤트를 조사했습니다.

- `kubectl get pods -n probe-resource-lab -l app=readiness-demo`
- `kubectl describe pod <Pod명> -n probe-resource-lab`

여기서 중요한 점은 Pod가 Running 상태여도 Ready 상태는 False가 될 수 있다는 것입니다.

Running은 컨테이너가 실행되고 있음을 나타내지만, Ready는 트래픽을 받을 준비가 되었는지를 나타냅니다.

### 4.5 EndpointSlice 확인

Readiness 실패가 Service의 트래픽 전달 대상에 미치는 영향을 확인했습니다.

사용한 명령어:

`kubectl get endpointslices -n probe-resource-lab -l kubernetes.io/service-name=readiness-demo-svc -o yaml`

실습에서 Ready 상태가 변경될 때 EndpointSlice의 준비 조건도 달라지는 것을 확인했습니다.

일반적인 Service에서는 Ready 상태가 False인 Endpoint를 정상적인 트래픽 전달 대상에서 제외합니다.

즉, Deployment에 Pod가 두 개 존재하더라도 두 Pod가 모두 실제 트래픽을 받을 수 있는 것은 아닙니다.

### 4.6 장애 복구

Readiness 검사 대상인 `/readyz`가 정상적으로 응답하도록 복구했습니다.

이후 Pod의 Ready 상태와 EndpointSlice 조건을 다시 확인하여 정상적인 트래픽 대상에 포함되는 것을 검증했습니다.

이 과정에서 **Readiness Probe는 애플리케이션을 재시작시키는 기능이 아니라, 트래픽 수신 가능 여부를 판단하는 기능**임을 확인했습니다.

---

## 5. Liveness와 Readiness 비교

| 비교 항목           | Liveness                         | Readiness                             |
| ------------------- | -------------------------------- | ------------------------------------- |
| 목적                | 계속 실행 가능한지 판단          | 트래픽 처리 준비 여부 판단            |
| 실습 검사 경로      | `/index.html`                    | `/readyz`                             |
| 실패 시 주요 동작   | 컨테이너 재시작                  | NotReady 상태 전환                    |
| Service에 대한 영향 | 재시작으로 가용성에 영향         | 일반적인 트래픽 대상에서 제외         |
| 주요 조사 대상      | Events, 이전 로그, Restart Count | Pod Conditions, Events, EndpointSlice |
| 복구 방향           | 생존 검사 실패 원인 해결         | 준비 상태 검사 실패 원인 해결         |

두 Probe는 비슷한 형태의 HTTP 검사를 사용할 수 있지만 목적과 실패 시 동작이 다릅니다.

## 6. 운영 관점에서 배운 점

### 6.1 Probe
