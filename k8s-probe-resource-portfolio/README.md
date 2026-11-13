# Kubernetes Probe & Resource — 장애 재현 및 운영 Troubleshooting

## 1. 프로젝트 소개

Kubernetes의 Probe와 Resource Requests/Limits를 학습하고, 실제 클러스터에서 발생할 수 있는 장애를 직접 재현·분석·복구한 실습 프로젝트입니다.

단순히 Kubernetes Manifest를 작성하는 것에서 끝나지 않고, **장애 증상 확인 → 원인 수집 → 분석 → 설정 수정 → 복구 검증** 과정을 중심으로 진행했습니다.

특히 Pod가 Running이더라도 정상 서비스를 제공하지 못할 수 있다는 점과, Requests·Limits 설정이 스케줄링 및 컨테이너 실행에 미치는 영향을 집중적으로 학습했습니다.

### 주요 학습 목표

- Startup, Readiness, Liveness Probe의 목적과 차이 이해
- HTTP, TCP, Exec 방식의 Probe 구성
- CPU·Memory Requests와 Limits의 차이 이해
- FailedScheduling, CPU Throttling, Memory OOM 장애 재현
- Guaranteed, Burstable, BestEffort QoS Class 비교
- Deployment, Service, EndpointSlice 기반 장애 분석
- Kubernetes Events, 컨테이너 로그, Linux 커널 로그를 활용한 원인 조사

## 2. 실습 환경

| 항목              | 환경                     |
| ----------------- | ------------------------ |
| OS                | Rocky Linux 9.8          |
| Kubernetes        | v1.36.3                  |
| Container Runtime | containerd 2.3.3         |
| Control Plane     | k8s-cp1                  |
| Worker Nodes      | k8s-worker1, k8s-worker2 |
| Network Plugin    | Calico                   |
| Namespace         | probe-resource-lab       |
| Manifest 작성     | vi / vim                 |
| 클러스터 관리     | kubectl                  |
| 자원 사용량 확인  | Metrics Server           |
| 문서 및 형상 관리 | VS Code, Git, GitHub     |

실제 Kubernetes 리소스의 생성과 장애 재현은 Rocky Linux 가상머신에서 수행했습니다.

Windows 환경에서는 실습 결과를 YAML 및 Markdown 문서로 정리하고 GitHub 저장소에 관리했습니다.

## 3. 프로젝트 구조

| 디렉터리                                                                   | 주요 내용                                      |
| -------------------------------------------------------------------------- | ---------------------------------------------- |
| [01-probe-basics](./01-probe-basics/README.md)                             | HTTP, TCP, Exec Readiness Probe 비교           |
| [02-liveness-readiness](./02-liveness-readiness/README.md)                 | Liveness 재시작 및 Readiness 트래픽 제외       |
| [03-startup-probe](./03-startup-probe/README.md)                           | 시작 지연으로 인한 반복 재시작과 Startup Probe |
| [04-integrated-probes](./04-integrated-probes/README.md)                   | Startup·Readiness·Liveness 통합 구성           |
| [05-resource-requests](./05-resource-requests/README.md)                   | CPU Requests 초과에 따른 FailedScheduling      |
| [06-cpu-throttling](./06-cpu-throttling/README.md)                         | CPU Limits와 cgroup Throttling 분석            |
| [07-memory-oom](./07-memory-oom/README.md)                                 | Memory Limit 초과와 커널 OOM 분석              |
| [08-qos](./08-qos/README.md)                                               | QoS Class, Node 자원 사용량 및 Eviction        |
| [09-integrated-troubleshooting](./09-integrated-troubleshooting/README.md) | 연속 장애 재현·분석·복구 및 Service 검증       |

각 디렉터리에는 실제 실습에서 사용한 Kubernetes Manifest와 분석 문서를 정리했습니다.

## 4. 핵심 장애 대응 사례

### 4.1 CPU Requests 초과 — FailedScheduling

**증상**

Pod가 Pending 상태에서 Node를 배정받지 못했습니다.

**확인된 원인**

```text
FailedScheduling

0/3 nodes are available:
1 node(s) had untolerated taint(s),
2 Insufficient cpu.
```

CPU Requests를 `2500m`로 설정했지만, Worker Node 각각의 CPU Allocatable은 `2000m`였습니다.

**복구 조치**

CPU Requests를 `500m`로 수정했습니다.

**검증 결과**

신규 Pod가 정상적으로 Node에 배치되고 Deployment의 Ready 및 Available Replica가 1개로 확인되었습니다.

**운영 관점**

CPU 사용률이 낮더라도 Requests 조건을 만족하지 못하면 Pod가 Pending 상태가 될 수 있습니다.

[상세 장애 분석](./05-resource-requests/README.md)

### 4.2 CPU Limit — Throttling 분석

**실습 조건**

- CPU Requests: `50m`
- CPU Limits: `100m`
- CPU 부하: 컨테이너 내부 반복 연산

**주요 검증 결과**

cgroup v2의 `cpu.max`에서 다음 값을 확인했습니다.

```text
10000 100000
```

CPU 부하 전후의 통계:

| 지표           | 부하 전 |    부하 후 |
| -------------- | ------: | ---------: |
| nr_periods     |      15 |        188 |
| nr_throttled   |       8 |        163 |
| throttled_usec | 805,688 | 16,423,762 |

부하 전후 `nr_throttled`가 155회 증가하여 CPU Throttling이 발생한 것을 확인했습니다.

해당 실습 직후 컨테이너는 Running 상태를 유지했으며 재시작은 발생하지 않았습니다.

**운영 관점**

CPU Throttling은 컨테이너 종료와 다릅니다. Pod가 Running이더라도 CPU Limit에 의한 성능 제한이 발생할 수 있습니다.

[상세 실습 분석](./06-cpu-throttling/README.md)

### 4.3 Memory Limit 초과 — OOM 장애

**증상**

Memory Limit이 `48Mi`인 컨테이너에서 `80Mi` 메모리 할당을 시도했을 때 프로세스가 종료되었습니다.

```text
command terminated with exit code 137
```

**원인 조사**

Worker Node의 Linux 커널 로그에서 다음 메시지를 확인했습니다.

```text
oom-kill:constraint=CONSTRAINT_MEMCG
Memory cgroup out of memory: Killed process
```

이를 통해 메모리 cgroup 제한에 따른 OOM이 발생했음을 확인했습니다.

**복구 조치**

Memory Limit을 `48Mi`에서 `128Mi`로 변경했습니다.

**검증 결과**

변경 후 동일한 `80Mi` 메모리 할당 테스트가 정상 완료되었으며, 신규 Pod는 Running / Ready 상태를 유지했습니다.

**운영 관점**

종료 코드 137만으로 OOMKilled를 단정하지 않고, 실제 커널 로그와 자원 제한을 함께 확인해야 합니다.

[상세 장애 분석](./07-memory-oom/README.md)

### 4.4 Probe & Resource 통합 Troubleshooting

**첫 번째 장애: FailedScheduling**

CPU Requests `2500m`로 인해 Pod가 Pending 상태에 머물렀습니다.

Requests를 `500m`로 수정하여 신규 Pod의 스케줄링을 복구했습니다.

**두 번째 장애: Readiness Probe HTTP 404**

스케줄링 이후 새 Pod는 Running이었지만 Ready 0/1 상태였습니다.

```text
Readiness probe failed:
HTTP probe failed with statuscode: 404
```

컨테이너 내부 HTTP 요청으로 `/readyz` 경로의 404 응답을 직접 확인했습니다.

**복구 조치**

실습용 `lifecycle.postStart` 설정으로 `/readyz` 파일을 생성하도록 변경했습니다.

**최종 검증**

- 신규 Pod 2개 모두 Running / Ready 1/1
- Deployment RollingUpdate 성공
- EndpointSlice에 Ready Endpoint 2개 확인
- ClusterIP Service DNS를 통한 nginx HTTP 응답 성공

**운영 관점**

하나의 장애를 해결했다고 해서 전체 서비스가 정상화되는 것은 아닙니다.

Scheduler, kubelet, Probe, Deployment, EndpointSlice, Service의 상태를 순서대로 확인하고 최종 통신까지 검증해야 합니다.

[통합 Troubleshooting 상세 기록](./09-integrated-troubleshooting/README.md)

---

## 5. Probe 동작 비교

| Probe     | 검사 목적                   | 실패 시 주요 동작                               |
| --------- | --------------------------- | ----------------------------------------------- |
| Startup   | 애플리케이션 시작 완료 확인 | 실패 임계 횟수 도달 시 컨테이너 재시작          |
| Readiness | 트래픽 수신 준비 상태 확인  | Pod NotReady, 일반적인 Service 트래픽 대상 제외 |
| Liveness  | 컨테이너 생존 상태 확인     | 실패 임계 횟수 도달 시 컨테이너 재시작          |

Startup Probe가 설정되어 있다면 성공하기 전까지 Readiness와 Liveness 검사는 실행되지 않습니다.

실제 운영에서는 각 Probe가 애플리케이션의 시작 상태, 준비 상태, 생존 상태를 정확하게 반영하도록 구성해야 합니다.

## 6. Resource Requests와 Limits 비교

| 항목               | Requests                        | Limits                        |
| ------------------ | ------------------------------- | ----------------------------- |
| 목적               | 자원 요청 및 스케줄링 판단      | 자원 사용 상한 제어           |
| CPU                | Node 배치 및 CPU 자원 공유 기준 | CPU Throttling 발생 가능      |
| Memory             | Node 배치 및 자원 관리 기준     | 한도 초과 시 OOM 발생 가능    |
| 실제 사용량과 관계 | 실제 사용량과 다를 수 있음      | 실행 중 자원 사용 제한에 영향 |

CPU·Memory Requests와 Limits는 서로 다른 목적을 가지고 있으므로 애플리케이션의 실제 자원 요구사항을 고려하여 설정해야 합니다.

## 7. QoS Class와 Node 자원 관리

세 가지 QoS Class를 실제 Pod에서 확인했습니다.

| QoS Class  | 실습 예시                          |
| ---------- | ---------------------------------- |
| Guaranteed | qos-guaranteed-demo                |
| Burstable  | cpu-throttle-demo, memory-oom-demo |
| BestEffort | probe-http-demo                    |

`kubectl describe node`로 확인한 Requests·Limits 합계와 `kubectl top node`로 측정한 실제 사용량이 다르다는 사실을 확인했습니다.

Node-pressure Eviction은 안전상의 이유로 직접 발생시키지 않았습니다.

대신 Kubernetes의 Eviction 동작과 Memory cgroup OOM의 차이를 개념적으로 분석했습니다.

[QoS 및 Eviction 상세 정리](./08-qos/README.md)

## 8. 장애 유형별 점검 기준

| 증상                | 우선 확인할 정보                   | 대표적인 원인              |
| ------------------- | ---------------------------------- | -------------------------- |
| Pending / Node 없음 | PodScheduled, Events               | Requests 부족, Taint       |
| Running / NotReady  | Readiness Probe, Events            | HTTP 404, Timeout          |
| CrashLoopBackOff    | Last State, Events, 이전 로그      | 반복적인 컨테이너 종료     |
| CPU 성능 저하       | CPU Limits, cpu.stat               | CPU Throttling             |
| 종료 코드 137       | 컨테이너 상태, 커널 로그           | OOM 또는 기타 SIGKILL 원인 |
| Service 통신 실패   | Selector, EndpointSlice, Pod Ready | Endpoint 부재, 연결 문제   |
| Rollout 미완료      | Deployment, ReplicaSet, Pod Ready  | 신규 Pod 준비 실패         |

이는 장애 조사를 시작할 때 활용할 수 있는 기본적인 점검 기준입니다.

실제 원인은 Kubernetes Events, 컨테이너 로그, Node 상태 등 객관적인 증거로 확인해야 합니다.

## 9. 주요 실습 명령어

### Pod 및 Deployment 상태

```bash
kubectl get pods -n probe-resource-lab -o wide
kubectl describe pod <Pod명> -n probe-resource-lab
kubectl get deployments -n probe-resource-lab
kubectl rollout status deployment/<Deployment명> -n probe-resource-lab
```

### 이전 컨테이너 로그

```bash
kubectl logs <Pod명> -n probe-resource-lab --previous
```

### Service와 EndpointSlice

```bash
kubectl get svc -n probe-resource-lab
kubectl get endpointslices -n probe-resource-lab -o yaml
```

### Resource 및 Node 상태

```bash
kubectl describe node k8s-worker1
kubectl top node
kubectl top pod -n probe-resource-lab
```

### cgroup 정보

```bash
kubectl exec -n probe-resource-lab <Pod명> -- cat /sys/fs/cgroup/cpu.stat
kubectl exec -n probe-resource-lab <Pod명> -- cat /sys/fs/cgroup/memory.max
```

위 명령어의 `<Pod명>`과 `<Deployment명>`은 실제 리소스 이름으로 변경하여 사용합니다.

일부 명령어는 해당 컨테이너에 필요한 도구와 cgroup 인터페이스가 존재해야 실행할 수 있습니다.

## 10. 실습 재현 시 주의사항

이 저장소의 Manifest는 기존 실습 환경인 `probe-resource-lab` Namespace를 기준으로 작성했습니다.

다른 클러스터에서 사용할 경우 Namespace와 자원 가용성을 먼저 확인해야 합니다.

또한 일부 실습은 의도적으로 다음과 같은 장애를 발생시켰습니다.

- CPU Requests 초과로 인한 Pending
- Probe 실패로 인한 NotReady 및 재시작
- CPU 부하에 따른 Throttling
- Memory Limit 초과에 따른 OOM

따라서 실제 운영 환경이나 공유 클러스터에서 무분별하게 실행해서는 안 됩니다.

저장소에는 주로 **최종 복구된 Manifest**를 보관했으며, 장애 재현 당시의 변경 설정과 조사 과정은 각 폴더의 README에 기록했습니다.

동일한 이름의 리소스가 이미 있는 클러스터에 Manifest를 적용하면 기존 리소스가 변경될 수 있으므로 주의해야 합니다.

## 11. 운영 관점에서 배운 점

이번 프로젝트에서 중요하게 학습한 내용은 다음과 같습니다.

1. **상태를 구분해야 합니다.** Pod의 Running, Ready, Scheduled는 서로 다른 의미를 가집니다.
2. **원인을 증거로 확인해야 합니다.** Events, 이전 로그, cgroup 통계, 커널 로그 등을 활용해야 합니다.
3. **Requests와 실제 사용량은 다릅니다.** Scheduler가 사용하는 자원 요청량과 실제 측정값을 혼동하면 안 됩니다.
4. **재시작이 항상 해결책은 아닙니다.** 장애의 원인에 맞는 복구 조치가 필요합니다.
5. **복구는 검증까지 포함합니다.** Manifest 적용 후 Pod Ready, Deployment, EndpointSlice, 실제 통신을 확인해야 합니다.
6. **실습과 운영 환경을 구분해야 합니다.** 장애 재현용 설정을 운영 환경에 그대로 사용해서는 안 됩니다.

## 12. 참고 문서

- [Kubernetes — Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes — Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes — Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
- [Kubernetes — Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

## 13. 결론

이번 프로젝트에서는 Kubernetes Probe와 Resource Requests/Limits를 직접 구성하고, 여러 장애 상황에서 Kubernetes가 어떻게 동작하는지 확인했습니다.

특히 CPU Requests로 인한 FailedScheduling, Readiness Probe HTTP 404, CPU Throttling, Memory cgroup OOM을 실제로 재현하고 원인을 분석했습니다.

이후 설정을 수정하고 Pod, Deployment, EndpointSlice, Service 및 Linux 커널 정보를 통해 복구 결과를 검증했습니다.

**이 프로젝트를 통해 Kubernetes 장애 대응에서 중요한 것은 단순한 명령어 실행이 아니라, 증상을 구분하고 근거를 수집하여 원인을 해결한 뒤 정상 동작을 검증하는 과정이라는 점을 학습했습니다.**
