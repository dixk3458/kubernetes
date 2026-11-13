# Kubernetes Startup·Readiness·Liveness Probe 통합 실습

## 1. 프로젝트 개요

Kubernetes Deployment에 Startup, Readiness, Liveness Probe를 함께 구성하고, 각 Probe가 애플리케이션의 실행 상태와 서비스 가용성에 어떤 영향을 미치는지 확인한 실습입니다.

nginx 컨테이너의 시작 시간을 의도적으로 지연시키고, Readiness와 Liveness 검사 경로에 각각 장애를 발생시켰습니다.

이를 통해 **애플리케이션 시작 완료, 트래픽 수신 준비, 컨테이너 생존 상태**를 구분하여 관리하는 방법을 학습했습니다.

### 실습 목표

- Startup·Readiness·Liveness Probe의 역할 이해
- 시작 시간이 긴 애플리케이션에서 Startup Probe 동작 확인
- Readiness 실패에 따른 Service 트래픽 대상 변화 확인
- Liveness 실패에 따른 컨테이너 재시작 확인
- EndpointSlice, Events, 로그를 이용한 장애 분석
- 장애 복구 후 정상 상태 검증

## 2. 실습 환경

| 항목            | 설정                   |
| --------------- | ---------------------- |
| OS              | Rocky Linux 9.8        |
| Kubernetes      | v1.36.3                |
| Namespace       | `probe-resource-lab`   |
| Deployment      | `probe-integrated`     |
| Container Image | `nginx:1.28-alpine`    |
| Replicas        | 2                      |
| Service         | `probe-integrated-svc` |
| Service Type    | ClusterIP              |
| 시작 지연       | 25초                   |

### 주요 파일

- `probe-integrated.yaml`: 세 가지 Probe를 함께 구성한 Deployment
- `probe-integrated-service.yaml`: ClusterIP Service

## 3. 통합 Probe 설계

### 3.1 애플리케이션 시작 지연

nginx 실행 전에 25초 동안 대기하도록 설정했습니다.

```yaml
command:
  - /bin/sh
  - -c
args:
  - "sleep 25; exec nginx -g 'daemon off;'"
```

컨테이너가 시작된 직후에는 nginx가 실행되지 않으므로 HTTP 요청에 정상적으로 응답할 수 없습니다.

이러한 상황에서 Startup Probe를 사용해 초기화 완료 여부를 먼저 확인하도록 구성했습니다.

### 3.2 Probe별 검사 경로

| Probe     | 검사 경로 | 목적                       |
| --------- | --------- | -------------------------- |
| Startup   | `/`       | nginx 시작 완료 확인       |
| Readiness | `/readyz` | 트래픽 수신 준비 상태 확인 |
| Liveness  | `/livez`  | 컨테이너 생존 상태 확인    |

모든 Probe는 HTTP GET 방식을 사용했습니다.

### 3.3 Probe 설정값

| 설정               | Startup | Readiness | Liveness |
| ------------------ | ------- | --------- | -------- |
| `periodSeconds`    | 5       | 5         | 5        |
| `timeoutSeconds`   | 2       | 2         | 2        |
| `failureThreshold` | 12      | 2         | 3        |
| 검사 경로          | `/`     | `/readyz` | `/livez` |

Startup Probe는 약 60초에 해당하는 시작 허용 구간을 확보하도록 설계했습니다.

실제 검사 및 재시작 시점은 Probe 실행 시점과 응답 시간에 따라 달라질 수 있습니다.

### 3.4 검사 엔드포인트 생성

nginx 기본 이미지에는 `/readyz`와 `/livez`가 별도로 존재하지 않으므로 `lifecycle.postStart`를 이용해 검사 파일을 생성했습니다.

```yaml
lifecycle:
  postStart:
    exec:
      command:
        - /bin/sh
        - -c
        - "echo ready > /usr/share/nginx/html/readyz; echo alive > /usr/share/nginx/html/livez"
```

이 설정은 Probe의 동작 차이를 확인하기 위한 실습용 구성입니다.

실제 운영에서는 단순한 파일 존재 여부보다 애플리케이션의 실제 상태를 반영하는 검사 엔드포인트가 적절합니다.

또한 `postStart`는 컨테이너의 시작 명령과 비동기적으로 실행될 수 있으므로 실행 순서를 절대적으로 보장하는 방식은 아닙니다.

---

## 4. 실습 1 — Startup Probe

### 4.1 실습 목적

nginx 실행을 25초 지연시킨 상태에서 Startup Probe가 시작 완료 여부를 확인하는 과정을 관찰했습니다.

### 4.2 동작 과정

1. Pod가 Node에 배치됩니다.
2. nginx 컨테이너가 시작됩니다.
3. 시작 명령에 따라 25초 동안 대기합니다.
4. Startup Probe가 `/` 경로를 주기적으로 검사합니다.
5. nginx가 실행된 이후 HTTP 응답이 성공하면 Startup Probe가 성공합니다.
6. 이후 Readiness 및 Liveness Probe가 실행됩니다.

### 4.3 확인 결과

Startup Probe가 포함된 Deployment에서 Pod 두 개가 Ready 상태에 도달했습니다.

실제 Deployment 조회 결과:

| 항목               | 값  |
| ------------------ | --- |
| Desired Replicas   | 2   |
| Ready Replicas     | 2   |
| Available Replicas | 2   |
| Updated Replicas   | 2   |

### 4.4 핵심 분석

Startup Probe가 성공하기 전에는 Liveness와 Readiness Probe가 실행되지 않습니다.

따라서 애플리케이션이 시작하는 도중 Liveness Probe에 의해 불필요하게 재시작되는 문제를 방지할 수 있습니다.

다만 Startup Probe 자체도 연속 실패 횟수가 설정된 임계값에 도달하면 컨테이너 재시작을 유발합니다.

---

## 5. 실습 2 — Readiness Probe 장애 재현

### 5.1 장애 재현 목적

애플리케이션은 실행 중이지만 요청을 받을 준비가 되지 않은 상황에서 Kubernetes가 트래픽을 어떻게 처리하는지 확인했습니다.

### 5.2 장애 발생

실습 과정에서 nginx의 `/readyz` 경로가 정상적으로 응답하지 못하도록 구성하여 Readiness Probe 실패를 재현했습니다.

이때 컨테이너는 계속 실행 중이지만 해당 Pod는 NotReady 상태가 되었습니다.

### 5.3 원인 조사

다음 명령어로 Pod 상태와 Events를 확인했습니다.

```bash
kubectl get pods -n probe-resource-lab -l app=probe-integrated
```

```bash
kubectl describe pod <Pod명> -n probe-resource-lab
```

Readiness Probe가 `/readyz` 경로에서 정상적인 응답을 받지 못하면 해당 컨테이너의 Ready 조건이 False로 전환될 수 있습니다.

중요한 점은 **Readiness Probe 실패 자체로 컨테이너가 재시작되지는 않는다**는 것입니다.

### 5.4 EndpointSlice 변화 확인

Service의 EndpointSlice를 조회했습니다.

```bash
kubectl get endpointslices \
  -n probe-resource-lab \
  -l kubernetes.io/service-name=probe-integrated-svc \
  -o yaml
```

실습에서는 Readiness Probe 실패에 따라 해당 Endpoint의 `ready` 조건이 False가 되는 것을 확인했습니다.

반면 정상 Ready 상태를 유지하는 다른 Pod는 Service의 트래픽 대상으로 남아 있었습니다.

이 과정에서 **Deployment에 Pod가 존재하는 것과 Service가 해당 Pod를 실제 트래픽 대상으로 사용하는 것은 다르다**는 사실을 확인했습니다.

### 5.5 복구 및 검증

장애를 발생시킨 `/readyz` 경로를 정상 응답하도록 복구했습니다.

이후 다음 항목을 다시 확인했습니다.

- Pod의 Ready 상태
- Readiness Probe 성공 여부
- EndpointSlice의 `ready` 조건
- Service의 정상 트래픽 대상

Readiness Probe 복구 이후 해당 Pod가 다시 정상적인 Service 트래픽 대상에 포함되는 것을 확인했습니다.

### 5.6 운영 관점

Readiness Probe는 Pod에 문제가 생겼을 때 트래픽을 차단하거나 다른 정상 Pod로 전달할 수 있도록 하는 데 중요한 역할을 합니다.

다만 모든 Pod가 NotReady가 된다면 Service에 정상적인 트래픽 대상이 없어질 수 있으므로, Readiness 조건을 지나치게 엄격하게 구성하지 않아야 합니다.

---

## 6. 실습 3 — Liveness Probe 장애 재현

### 6.1 장애 재현 목적

애플리케이션의 생존 상태 검사에 실패했을 때 Kubernetes가 컨테이너를 어떻게 처리하는지 확인했습니다.

### 6.2 장애 발생

실습 과정에서 `/livez` 경로가 정상적으로 응답하지 못하도록 구성했습니다.

Liveness Probe가 반복적으로 실패하면서 kubelet의 컨테이너 재시작 동작이 발생했습니다.

### 6.3 장애 조사

다음 명령어로 Pod 상태와 이벤트를 확인했습니다.

```bash
kubectl get pods -n probe-resource-lab -l app=probe-integrated
```

```bash
kubectl describe pod <Pod명> -n probe-resource-lab
```

실제 실습에서는 Liveness 실패와 관련된 `Unhealthy`, `Killing` 이벤트를 확인했습니다.

이를 통해 kubelet이 Liveness Probe의 연속 실패를 감지하고 컨테이너 종료 및 재시작 절차를 수행한다는 사실을 확인했습니다.

### 6.4 이전 컨테이너 로그 조사

다음 명령어로 재시작 전 컨테이너의 로그를 조사했습니다.

```bash
kubectl logs <Pod명> \
  -n probe-resource-lab \
  --previous
```

실습 과정에서 Liveness 검사 경로에 대한 HTTP 404 응답을 확인했습니다.

따라서 단순히 Restart Count만 보고 판단한 것이 아니라, Probe 실패 이벤트와 HTTP 응답 기록을 함께 사용해 장애 원인을 분석했습니다.

컨테이너 종료 코드만으로 Liveness 실패 여부를 확정해서는 안 됩니다. Probe 실패 이벤트와 실제 종료 시점을 함께 확인해야 합니다.

### 6.5 복구 및 검증

Liveness 실패 이후 컨테이너가 재시작되는 것을 확인했습니다.

이번 실습 구성에서는 새 컨테이너가 시작될 때 `postStart` 훅으로 `/livez` 파일이 다시 생성되므로, 검사 경로가 복구될 수 있습니다.

이후 Pod가 Ready 상태로 돌아오는 과정을 확인했습니다.

### 6.6 운영 관점

Liveness Probe는 컨테이너를 재시작해야 하는 상황을 판단하는 데 사용해야 합니다.

외부 API 일시 장애 등 컨테이너 재시작으로 해결되지 않는 문제까지 Liveness 실패 조건으로 연결하면 불필요한 재시작이 반복될 수 있습니다.

따라서 Liveness와 Readiness의 실패 조건을 구분해 설계해야 합니다.

---

## 7. 세 가지 Probe의 장애 대응 비교

| 구분         | Startup               | Readiness             | Liveness                         |
| ------------ | --------------------- | --------------------- | -------------------------------- |
| 검사 목적    | 시작 완료             | 트래픽 수신 준비      | 생존 상태                        |
| 실습 경로    | `/`                   | `/readyz`             | `/livez`                         |
| 연속 실패 시 | 컨테이너 재시작       | NotReady              | 컨테이너 재시작                  |
| Service 영향 | 초기 준비 상태에 영향 | 정상 트래픽 대상 제외 | 재시작 과정에서 가용성 영향      |
| 주요 검증    | Pod Ready, 시작 단계  | Ready, EndpointSlice  | Events, Restart Count, 이전 로그 |

세 가지 Probe는 모두 HTTP 요청으로 구성할 수 있지만, 실패했을 때 Kubernetes가 수행하는 동작은 다릅니다.

## 8. Service와 EndpointSlice

이번 Deployment는 Replica 2개로 구성했습니다.

Service는 다음 Label을 가진 Pod를 선택합니다.

```yaml
selector:
  app: probe-integrated
```

Readiness Probe가 정상인 Pod는 Service의 트래픽 대상에 포함될 수 있습니다.

반면 Readiness에 실패한 Pod는 일반적인 Service 트래픽 전달 대상에서 제외됩니다.

EndpointSlice의 `conditions.ready`를 확인하면 어떤 Pod가 트래픽을 받을 준비가 된 상태인지 판단할 수 있습니다.

이때 Pod IP가 EndpointSlice에 존재한다는 사실만으로 해당 Pod가 Ready 상태임을 의미하지는 않습니다.

---

## 9. 주요 장애 조사 명령어

| 명령어                               | 목적                               |
| ------------------------------------ | ---------------------------------- |
| `kubectl get pods`                   | Running, Ready, Restart Count 확인 |
| `kubectl describe pod`               | Probe 설정 및 장애 이벤트 확인     |
| `kubectl logs --previous`            | 재시작 전 컨테이너 로그 확인       |
| `kubectl get deployment`             | Ready 및 Available Replica 확인    |
| `kubectl get endpointslices -o yaml` | 실제 Service 트래픽 대상 확인      |

## 10. 운영 관점에서 배운 점

### 10.1 세 Probe는 서로 다른 문제를 해결한다

Startup은 애플리케이션의 시작 완료 여부를 확인합니다.

Readiness는 트래픽을 받을 수 있는지 판단합니다.

Liveness는 컨테이너를 계속 실행할 수 있는지 판단합니다.

이 차이를 이해하지 못하면 정상적인 애플리케이션을 불필요하게 재시작시키거나, 준비되지 않은 Pod에 트래픽을 전달할 수 있습니다.

### 10.2 애플리케이션 상태에 맞는 검사 조건이 중요하다

이번 실습에서는 각 Probe에 별도 경로를 사용했습니다.

하지만 실제 운영에서는 단순히 경로를 분리하는 것만으로 충분하지 않습니다.

Startup은 초기화 완료를, Readiness는 요청 처리 준비를, Liveness는 재시작이 필요한 비정상 상태를 적절하게 반영해야 합니다.

### 10.3 정상 Replica 수만으로 Service 상태를 단정하지 않는다

Deployment에 Pod 두 개가 존재하더라도 Ready Endpoint 수가 줄어들면 실제 트래픽을 받을 수 있는 Pod가 감소할 수 있습니다.

따라서 Deployment, Pod, EndpointSlice를 함께 확인해야 합니다.

### 10.4 재시작 횟수만으로 장애 원인을 확정하지 않는다

Restart Count가 증가했다면 컨테이너가 재시작된 것은 알 수 있습니다.

하지만 왜 재시작됐는지는 Events, 이전 로그, 종료 상태 등을 함께 확인해야 합니다.

### 10.5 실습용 상태 검사와 운영용 상태 검사를 구분한다

이번 실습의 `/readyz`, `/livez`는 파일 기반의 단순한 HTTP 검사입니다.

실제 운영에서는 애플리케이션의 내부 상태와 요청 처리 능력을 고려해 검사 조건을 설계해야 합니다.

또한 모든 의존성 장애를 Liveness 실패로 연결하지 않도록 주의해야 합니다.

---

## 11. 최종 검증 결과

| 검증 항목            | 결과                                |
| -------------------- | ----------------------------------- |
| nginx 시작 지연 25초 | 구성 및 확인                        |
| Startup Probe        | 적용 후 정상 시작 확인              |
| Readiness 장애       | 실패 및 NotReady 상태 확인          |
| EndpointSlice 변화   | Readiness에 따른 대상 변경 확인     |
| Liveness 장애        | 실패 이벤트 및 컨테이너 재시작 확인 |
| 최종 Deployment      | Ready Replicas 2                    |
| Available Replicas   | 2                                   |

## 12. 결론

이번 실습에서는 하나의 Kubernetes Deployment에 Startup, Readiness, Liveness Probe를 함께 구성하고 각각의 역할을 검증했습니다.

애플리케이션 시작 지연, 트래픽 수신 준비 실패, 생존 검사 실패를 구분하여 재현했으며, Kubernetes가 각 상황에서 수행하는 동작을 관찰했습니다.

이를 통해 **안정적인 Kubernetes 운영을 위해서는 단순히 Probe를 추가하는 것이 아니라, 시작 상태·준비 상태·생존 상태를 구분해 검사하고 장애 결과를 검증해야 한다**는 점을 학습했습니다.
