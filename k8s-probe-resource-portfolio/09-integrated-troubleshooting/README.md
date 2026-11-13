# Kubernetes Probe & Resource 통합 Troubleshooting

## 1. 프로젝트 개요

Kubernetes Deployment에서 발생하는 스케줄링 장애와 Readiness Probe 장애를 직접 재현하고, 원인을 조사하여 복구한 실습입니다.

단순히 Pod를 정상 실행하는 것이 아니라 다음 과정을 검증하는 데 목적이 있습니다.

- Kubernetes Events를 이용한 장애 원인 분석
- CPU Requests와 Scheduler의 배치 판단 이해
- Running과 Ready 상태의 차이 확인
- Readiness Probe 실패 원인 조사 및 복구
- RollingUpdate와 ReplicaSet 교체 과정 확인
- EndpointSlice 및 Service를 통한 최종 통신 검증

## 2. 실습 환경

| 항목            | 환경                        |
| --------------- | --------------------------- |
| OS              | Rocky Linux 9.8             |
| Kubernetes      | v1.36.3                     |
| Control Plane   | k8s-cp1                     |
| Worker Nodes    | k8s-worker1, k8s-worker2    |
| Namespace       | probe-resource-lab          |
| Container Image | nginx:1.28-alpine           |
| Deployment      | troubleshoot-integrated     |
| Replicas        | 2                           |
| Service         | troubleshoot-integrated-svc |
| Service Type    | ClusterIP                   |

이번 실습은 기존 Kubernetes 클러스터를 사용했으며, 다른 단원의 리소스는 변경하지 않았습니다.

## 3. 장애 시나리오

하나의 Deployment에 두 가지 문제를 의도적으로 구성했습니다.

1. CPU Requests를 Worker Node의 Allocatable보다 크게 설정하여 스케줄링 장애 재현
2. nginx가 제공하지 않는 경로를 Readiness Probe에 지정하여 NotReady 상태 재현

각 문제를 개별적으로 조사하고 복구한 뒤, Deployment와 Service가 정상 동작하는지 확인했습니다.

---

## 4. 장애 1 — FailedScheduling

### 4.1 장애 증상

Deployment 생성 후 Pod 두 개가 Pending 상태로 유지되었습니다.

```text
READY   STATUS    IP       NODE
0/1     Pending   <none>   <none>
0/1     Pending   <none>   <none>
```

Pod IP와 Node가 할당되지 않은 것을 보고, 애플리케이션 내부 오류보다 스케줄링 문제를 우선 의심했습니다.

### 4.2 원인 조사

다음 명령어로 Pod의 상세 상태와 Events를 확인했습니다.

```bash
kubectl describe pod troubleshoot-integrated-67bcfb9759-47sjq \
  -n probe-resource-lab
```

주요 출력:

```text
PodScheduled   False

Warning FailedScheduling
0/3 nodes are available:
1 node(s) had untolerated taint(s),
2 Insufficient cpu.
```

당시 Deployment의 CPU Requests는 다음과 같았습니다.

```yaml
resources:
  requests:
    cpu: "2500m"
    memory: "64Mi"
  limits:
    memory: "128Mi"
```

두 Worker Node의 CPU Allocatable은 각각 2000m였습니다.

신규 Pod가 요청한 2500m는 Worker Node 한 대의 Allocatable보다 크기 때문에 해당 Node에 배치할 수 없었습니다.

### 4.3 원인 분석

Kubernetes Scheduler는 Node의 실제 CPU 사용률만 보고 Pod를 배치하지 않습니다.

CPU Requests를 비롯한 자원 요청량과 Node의 Allocatable, 기존 Pod가 예약한 Requests, Taint 등의 스케줄링 조건을 고려합니다.

따라서 CPU 사용률이 100%가 아니더라도 `Insufficient cpu`로 인해 Pending 상태가 발생할 수 있습니다.

### 4.4 복구 조치

애플리케이션의 실습용 자원 요구량을 고려하여 CPU Requests를 변경했습니다.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "64Mi"
  limits:
    memory: "128Mi"
```

수정 후 다음 명령어를 실행했습니다.

```bash
kubectl apply --dry-run=server -f troubleshoot-integrated.yaml
kubectl apply -f troubleshoot-integrated.yaml
```

### 4.5 복구 검증

신규 Pod가 `k8s-worker1`에 배치되었고 Pod IP도 할당되었습니다.

```text
READY   STATUS    NODE
0/1     Running   k8s-worker1
```

스케줄링 장애는 해결되었지만, 신규 Pod는 여전히 Ready 상태가 되지 못했습니다.

따라서 Readiness Probe 관련 장애를 추가로 조사했습니다.

---

## 5. 장애 2 — Readiness Probe HTTP 404

### 5.1 장애 증상

신규 Pod는 Running 상태였지만 Ready 값이 0/1로 유지되었습니다.

```text
STATUS    Running
Ready     False
Restart Count: 0
```

### 5.2 원인 조사

Pod 상세 정보를 확인했습니다.

```bash
kubectl describe pod troubleshoot-integrated-7d7df9f75c-s9n5p \
  -n probe-resource-lab
```

주요 이벤트:

```text
Warning Unhealthy
Readiness probe failed:
HTTP probe failed with statuscode: 404
```

Readiness Probe 설정은 다음과 같았습니다.

```yaml
readinessProbe:
  httpGet:
    path: /readyz
    port: 80
  periodSeconds: 5
  failureThreshold: 2
```

실제 컨테이너 내부에서 해당 경로에 HTTP 요청을 전송했습니다.

```bash
kubectl exec -n probe-resource-lab \
  troubleshoot-integrated-7d7df9f75c-s9n5p \
  -- wget -S -O /dev/null http://127.0.0.1/readyz
```

실제 출력:

```text
HTTP/1.1 404 Not Found
wget: server returned error: HTTP/1.1 404 Not Found
command terminated with exit code 1
```

이를 통해 `/readyz` 경로가 정상적인 성공 응답을 반환하지 못한다는 사실을 확인했습니다.

여기서 종료 코드 1은 wget 명령어의 종료 코드이며, nginx 컨테이너가 종료됐다는 의미는 아닙니다.

### 5.3 원인 분석

nginx 기본 웹 페이지는 `/` 경로에서 응답하지만, 실습에 사용한 이미지에는 `/readyz` 파일이나 별도 엔드포인트가 구성되어 있지 않았습니다.

따라서 Readiness Probe의 HTTP 요청이 404를 반환했고, Kubernetes는 컨테이너를 NotReady로 판단했습니다.

Readiness Probe 실패는 그 자체로 컨테이너를 재시작시키지 않고, 일반적인 Service 트래픽 대상에서 제외하도록 합니다.

### 5.4 복구 조치

실습에서는 기존 Readiness 경로를 유지하고, 컨테이너 시작 시 해당 파일이 생성되도록 `lifecycle.postStart`를 추가했습니다.

```yaml
lifecycle:
  postStart:
    exec:
      command:
        - /bin/sh
        - -c
        - "touch /usr/share/nginx/html/readyz"
```

이후 Deployment Manifest를 다시 적용했습니다.

```bash
kubectl apply --dry-run=server -f troubleshoot-integrated.yaml
kubectl apply -f troubleshoot-integrated.yaml
```

### 5.5 복구 검증

최종적으로 신규 ReplicaSet의 Pod 두 개가 모두 Ready 상태에 도달했습니다.

```text
NAME                                       READY   STATUS
troubleshoot-integrated-5d5c5d8d7b-57ggk  1/1     Running
troubleshoot-integrated-5d5c5d8d7b-hdktb  1/1     Running
```

두 Pod 모두 Restart Count는 0이었습니다.

---

## 6. Deployment RollingUpdate 검증

Pod Template 변경 시 새로운 ReplicaSet이 생성되었습니다.

업데이트 중에는 기존 Pending Pod와 신규 Running/NotReady Pod가 동시에 존재했습니다.

이는 Deployment가 RollingUpdate 전략에 따라 Pod 교체를 진행하기 때문입니다.

신규 Pod의 준비 상태가 정상화된 뒤 다음 명령어로 업데이트 완료 여부를 검증했습니다.

```bash
kubectl rollout status deployment/troubleshoot-integrated \
  -n probe-resource-lab --timeout=30s
```

실제 출력:

```text
deployment "troubleshoot-integrated" successfully rolled out
```

이를 통해 최종 Deployment 업데이트가 정상 완료됐음을 확인했습니다.

---

## 7. Service 및 EndpointSlice 검증

### 7.1 Service 구성

ClusterIP Service를 생성했습니다.

```bash
kubectl apply --dry-run=server -f troubleshoot-service.yaml
kubectl apply -f troubleshoot-service.yaml
```

Service의 Selector는 다음과 같습니다.

```yaml
selector:
  app: troubleshoot-integrated
```

### 7.2 EndpointSlice 확인

다음 명령어를 실행했습니다.

```bash
kubectl get endpointslices -n probe-resource-lab \
  -l kubernetes.io/service-name=troubleshoot-integrated-svc \
  -o yaml
```

확인 결과:

| Pod IP         | Node        | Ready | Serving | Terminating |
| -------------- | ----------- | ----- | ------- | ----------- |
| 10.244.194.101 | k8s-worker1 | true  | true    | false       |
| 10.244.126.12  | k8s-worker2 | true  | true    | false       |

두 Pod 모두 EndpointSlice에 정상 등록되어 있음을 확인했습니다.

Pod IP는 실행 환경에 따라 달라질 수 있으며, Service는 Label Selector를 통해 대상 Pod를 선택합니다.

### 7.3 실제 HTTP 통신 확인

동일 Namespace의 기존 실습 Pod에서 Service DNS 이름으로 HTTP 요청을 전송했습니다.

```bash
kubectl exec -n probe-resource-lab cpu-throttle-demo \
  -- wget -qO- -T 5 http://troubleshoot-integrated-svc/
```

실제 응답에는 다음 내용이 포함되어 있었습니다.

```html
<h1>Welcome to nginx!</h1>
```

이를 통해 Service DNS를 이용한 HTTP 요청이 정상적으로 처리된 것을 확인했습니다.

단, 한 번의 HTTP 요청 성공만으로 두 Pod 모두의 응답을 개별적으로 확인한 것은 아닙니다. 두 Pod의 Ready Endpoint 등록 여부는 EndpointSlice를 통해 검증했습니다.

---

## 8. 최종 장애 복구 결과

| 검증 항목           | 장애 발생 당시   | 복구 후            |
| ------------------- | ---------------- | ------------------ |
| CPU Requests        | 2500m            | 500m               |
| Pod Scheduling      | FailedScheduling | 성공               |
| Pod Ready           | 0/1              | 1/1                |
| Readiness Probe     | HTTP 404         | 정상               |
| 정상 준비된 Replica | 0개              | 2개                |
| RollingUpdate       | 미완료           | 완료               |
| EndpointSlice       | 미검증           | Ready Endpoint 2개 |
| Service HTTP        | 미검증           | 성공               |

## 9. 운영 관점에서 배운 점

### 9.1 Running은 Ready를 의미하지 않는다

Pod가 Running 상태라도 Readiness Probe에 실패하면 일반적인 Service 트래픽 대상에서 제외될 수 있습니다.

따라서 Pod 상태를 확인할 때 STATUS뿐만 아니라 READY, Conditions, Events를 함께 확인해야 합니다.

### 9.2 Requests와 실제 리소스 사용량을 구분해야 한다

CPU Requests는 Scheduler가 Pod를 배치할 때 사용하는 중요한 기준입니다.

Node의 실제 CPU 사용률이 낮아도 Requests 조건을 만족하지 못하면 Pod가 Pending 상태가 될 수 있습니다.

### 9.3 문제를 순서대로 해결해야 한다

이번 실습에서는 스케줄링 장애가 해결된 이후에야 Readiness Probe 장애를 확인할 수 있었습니다.

따라서 장애를 조사할 때는 Pod의 현재 단계와 Events를 근거로 원인을 좁혀야 합니다.

### 9.4 Probe는 실제 애플리케이션의 상태를 반영해야 한다

이번 실습에서는 `/readyz` 파일 생성으로 Readiness Probe를 복구했습니다.

하지만 이는 Probe 동작 원리를 이해하기 위한 실습용 설정입니다.

실제 운영에서는 단순한 파일 존재 여부보다 애플리케이션이 요청을 정상적으로 처리할 수 있는지 확인하는 Readiness 엔드포인트가 적절합니다.

또한 `postStart` 훅은 컨테이너 진입점과 비동기적으로 실행될 수 있으므로, 시작 직후 짧은 시간 동안 Readiness 요청이 실패할 가능성도 고려해야 합니다.

### 9.5 장애 복구는 검증까지 포함한다

설정을 수정하고 Apply하는 것으로 복구를 완료했다고 판단하면 안 됩니다.

이번 실습에서는 다음을 순서대로 검증했습니다.

1. 신규 Pod 스케줄링
2. Pod Ready 상태
3. Deployment RollingUpdate 완료
4. EndpointSlice의 Ready 조건
5. Service를 통한 실제 HTTP 응답

## 10. 주요 파일

| 파일                           | 설명                   |
| ------------------------------ | ---------------------- |
| `troubleshoot-integrated.yaml` | 최종 복구된 Deployment |
| `troubleshoot-service.yaml`    | ClusterIP Service      |

## 11. 결론

이번 실습에서는 CPU Requests 설정 오류로 인한 FailedScheduling과 Readiness Probe의 HTTP 404 장애를 연속적으로 재현했습니다.

각 장애를 Pod 상태와 Events, 컨테이너 내부 HTTP 요청으로 조사하고, Manifest를 수정하여 복구했습니다.

이후 RollingUpdate, EndpointSlice, Service HTTP 통신을 검증했습니다.

이를 통해 Kubernetes 장애 대응에서 중요한 것은 단순히 Pod를 재시작하는 것이 아니라, **증상에 맞는 증거를 수집하고 원인을 구분한 뒤 복구 결과를 검증하는 과정**임을 학습했습니다.
