# Kubernetes Resource Requests — FailedScheduling 장애 재현 및 복구

## 1. 실습 개요

Kubernetes에서 CPU Requests를 Node의 할당 가능한 자원보다 크게 설정했을 때 발생하는 `FailedScheduling` 장애를 재현하고 복구한 실습입니다.

이번 실습에서는 CPU Requests를 `2500m`로 설정하여 Pod가 Pending 상태에 머무르는 상황을 만들었습니다.

이후 Scheduler의 Events를 분석하고 CPU Requests를 `500m`로 수정하여 정상적으로 Pod가 배치되는 과정을 확인했습니다.

**핵심 학습 목표는 CPU Requests와 실제 CPU 사용률을 구분하고, Scheduler의 자원 기반 배치 판단을 이해하는 것입니다.**

## 2. 실습 환경

| 항목                        | 설정                         |
| --------------------------- | ---------------------------- |
| OS                          | Rocky Linux 9.8              |
| Kubernetes                  | v1.36.3                      |
| Namespace                   | `probe-resource-lab`         |
| Deployment                  | `cpu-request-pending`        |
| Container Image             | `nginx:1.28-alpine`          |
| Replicas                    | 1                            |
| Worker Nodes                | `k8s-worker1`, `k8s-worker2` |
| Worker Node CPU Allocatable | 각각 2000m                   |

기존 Kubernetes 클러스터를 사용했으며, 실습용 Deployment의 CPU Requests를 변경하여 장애를 재현했습니다.

## 3. Resource Requests 개념

Kubernetes에서 `resources.requests`는 컨테이너가 필요로 하는 자원량을 선언하는 설정입니다.

Scheduler는 Pod를 Node에 배치할 때 Requests를 고려합니다.

### CPU 단위

| CPU 값  | 의미      |
| ------- | --------- |
| `1000m` | CPU 1개   |
| `500m`  | CPU 0.5개 |
| `2500m` | CPU 2.5개 |

CPU Requests는 실제 CPU 사용량을 나타내는 값이 아닙니다.

예를 들어 CPU Requests가 `500m`라고 해서 컨테이너가 항상 CPU 500m를 사용하는 것은 아닙니다.

Requests는 스케줄링과 자원 보장에 사용되는 설정이며, 실제 CPU 사용량은 워크로드의 실행 상태에 따라 달라집니다.

## 4. 장애 재현 — CPU Requests 2500m

### 4.1 초기 설정

장애 재현을 위해 다음과 같이 CPU Requests를 설정했습니다.

```yaml
resources:
  requests:
    cpu: "2500m"
    memory: "64Mi"
```

Worker Node의 CPU Allocatable은 각각 `2000m`였습니다.

따라서 Pod 하나가 요청한 CPU 2500m는 Worker Node 한 대의 CPU Allocatable을 초과합니다.

### 4.2 장애 증상

Deployment를 생성한 뒤 Pod가 Pending 상태로 유지되는 것을 확인했습니다.

대표적인 증상은 다음과 같습니다.

```text
READY   STATUS    NODE
0/1     Pending   <none>
```

`NODE <none>`은 아직 Pod가 Node에 배정되지 않았음을 의미합니다.

이 상태에서는 컨테이너가 실행되지 않았으므로 애플리케이션 내부의 HTTP 오류보다 스케줄링 원인을 먼저 조사해야 합니다.

## 5. 장애 원인 조사

### 5.1 Pod 상태 확인

다음 명령어를 사용했습니다.

```bash
kubectl get pods -n probe-resource-lab \
  -l app=cpu-request-pending -o wide
```

### 5.2 Scheduler 이벤트 확인

Pod의 상세 상태와 Events를 확인했습니다.

```bash
kubectl describe pod <Pod명> \
  -n probe-resource-lab
```

실습에서 다음과 같은 스케줄링 실패 사유를 확인했습니다.

```text
FailedScheduling

0/3 nodes are available:
1 node(s) had untolerated taint(s),
2 Insufficient cpu.
```

세 Node 중 한 Node는 허용되지 않은 Taint 조건 때문에, 나머지 두 Node는 CPU Requests를 충족하지 못해 Pod를 배치할 수 없었습니다.

### 5.3 근본 원인 분석

CPU Requests `2500m`는 Worker Node 각각의 CPU Allocatable `2000m`보다 큽니다.

Kubernetes는 하나의 일반적인 Pod를 단일 Node에 배치하므로 여러 Node의 CPU를 합산하여 Pod 하나에 할당할 수 없습니다.

또한 Scheduler는 단순히 Node의 현재 CPU 사용률만을 기준으로 배치하지 않습니다.

Node의 Allocatable과 이미 배치된 Pod의 Requests, Taint 및 기타 스케줄링 조건을 고려합니다.

따라서 **CPU 사용률이 반드시 100%가 아니더라도 `Insufficient cpu`로 인해 Pod가 Pending 상태에 머무를 수 있습니다.**

## 6. 복구 조치

### 6.1 CPU Requests 수정

실습용 nginx의 자원 요구량을 고려하여 CPU Requests를 다음과 같이 변경했습니다.

| 항목            | 변경 전 | 변경 후 |
| --------------- | ------- | ------- |
| CPU Requests    | 2500m   | 500m    |
| Memory Requests | 64Mi    | 64Mi    |
| Replicas        | 1       | 1       |

수정된 Manifest:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "64Mi"
```

500m는 이번 실습에서 스케줄링 동작을 검증하기 위해 선택한 값입니다.

실제 운영에서는 애플리케이션의 CPU 사용량과 성능 요구사항을 측정해 적절한 Requests를 설정해야 합니다.

### 6.2 변경 사항 적용

```bash
kubectl apply --dry-run=server -f cpu-request-pending.yaml
kubectl apply -f cpu-request-pending.yaml
```

Server Dry-run으로 API Server의 유효성 검증을 수행한 뒤 변경 사항을 적용했습니다.

## 7. 복구 결과 검증

### 7.1 Pod 스케줄링 성공

CPU Requests를 수정한 이후 새로운 Pod가 생성되었으며, `k8s-worker2`에 배치되어 Running 상태에 도달했습니다.

이전에는 Node가 배정되지 않았지만, 복구 후에는 Pod IP와 Node가 할당되었습니다.

이를 통해 CPU Requests 수정이 스케줄링 장애를 해결했음을 확인했습니다.

### 7.2 Deployment 상태 확인

다음 명령어로 최종 Deployment 상태를 확인했습니다.

```bash
kubectl get deployment cpu-request-pending \
  -n probe-resource-lab -o yaml
```

실제 조회 결과:

| 항목                | 확인된 값 |
| ------------------- | --------- |
| Desired Replicas    | 1         |
| Ready Replicas      | 1         |
| Available Replicas  | 1         |
| Updated Replicas    | 1         |
| Deployment Revision | 2         |

최종 Deployment의 `Available` 조건은 True였으며, 새로운 ReplicaSet이 정상적으로 진행된 것을 확인했습니다.

## 8. Requests와 Limits 비교

| 항목                 | Requests                                          | Limits                                            |
| -------------------- | ------------------------------------------------- | ------------------------------------------------- |
| 주요 목적            | 자원 요청 및 스케줄링                             | 컨테이너 자원 사용 상한 제어                      |
| CPU                  | Scheduler 배치 판단, CPU 공유 시 가중치 등에 영향 | 설정 시 CFS 대역폭 제한 등을 통한 Throttling 가능 |
| Memory               | Scheduler 배치 판단, 자원 관리 기준               | 설정된 메모리 한도 초과 시 OOM 발생 가능          |
| 실제 사용량과의 관계 | 실제 사용량과 다를 수 있음                        | 실제 사용을 제한하는 기준                         |

이번 Deployment에는 CPU와 Memory Requests만 지정하고 Limits는 명시하지 않았습니다.

따라서 이 실습의 Pending 장애는 Limits가 아니라 **CPU Requests가 Node의 스케줄링 가능 자원량을 초과한 문제**입니다.

## 9. 운영 관점에서 배운 점

### 9.1 Pending 상태에서는 Scheduler Events를 우선 확인한다

Pod가 Pending이고 Node가 배정되지 않았다면 `kubectl describe pod`로 `PodScheduled` 조건과 `FailedScheduling` 이벤트를 확인해야 합니다.

원인을 확인하기 전에 Pod를 반복해서 삭제하거나 재생성하는 것은 근본적인 해결책이 아닙니다.

### 9.2 Requests를 실제 사용률과 혼동하지 않는다

CPU 사용률이 낮더라도 CPU Requests가 과도하면 새 Pod의 배치에 실패할 수 있습니다.

따라서 `kubectl top`으로 확인하는 실제 사용량과 `kubectl describe node`에 표시되는 예약된 Requests를 구분해서 해석해야 합니다.

### 9.3 Requests를 무조건 낮추는 것이 정답은 아니다

이번 실습에서는 nginx의 자원 요구량에 맞춰 CPU Requests를 줄여 문제를 해결했습니다.

그러나 운영 환경에서는 Requests를 지나치게 낮추면 자원 경합 상황에서 성능 저하가 발생할 수 있습니다.

실제 사용량과 서비스 요구사항을 측정한 뒤 Requests를 조정하거나 Node의 자원을 확충해야 합니다.

### 9.4 장애 복구는 Pod의 정상 상태까지 검증한다

Manifest가 정상적으로 적용됐다는 사실만으로 장애가 해결된 것은 아닙니다.

이번 실습에서는 변경 적용 후 신규 Pod의 Node 배정, Running 상태와 Deployment의 Ready·Available Replica를 확인했습니다.

## 10. 주요 실습 파일

| 파일                       | 설명                                              |
| -------------------------- | ------------------------------------------------- |
| `cpu-request-pending.yaml` | CPU Requests 수정 후 정상화된 Deployment Manifest |

장애 재현 당시의 CPU Requests `2500m` 설정은 이 README에 기록했고, YAML 파일에는 최종 복구된 `500m` 설정을 저장했습니다.

## 11. 결론

이번 실습에서는 CPU Requests를 Worker Node의 Allocatable보다 크게 설정하여 `FailedScheduling` 장애를 재현했습니다.

Pod의 Pending 상태와 Scheduler Events를 근거로 원인을 확인하고, CPU Requests를 수정하여 정상적으로 스케줄링되는 결과를 검증했습니다.

이번 실습을 통해 **Kubernetes 장애 대응에서는 현재 자원 사용률뿐 아니라 Scheduler가 어떤 기준으로 Node를 선택하는지도 이해해야 한다는 점**을 학습했습니다.
