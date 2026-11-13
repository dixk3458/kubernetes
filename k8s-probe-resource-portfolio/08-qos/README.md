# Kubernetes QoS Class & Node Eviction — 자원 요청량과 실제 사용량 분석

## 1. 실습 개요

Kubernetes의 세 가지 QoS Class인 Guaranteed, Burstable, BestEffort를 실제 Pod 설정과 실행 결과로 비교한 실습입니다.

또한 Node에 설정된 CPU·Memory Requests 및 Limits와 실제 자원 사용량을 비교하고, Node 자원 압박에 따른 Eviction과 컨테이너 Memory OOM의 차이를 학습했습니다.

이번 실습에서는 **QoS Class별 Pod를 확인하고 Node 자원 상태를 분석했으며, Node Eviction은 실제로 발생시키지 않았습니다.**

### 실습 목표

- Guaranteed, Burstable, BestEffort의 분류 기준 이해
- CPU·Memory Requests와 Limits에 따른 QoS Class 확인
- Node의 예약된 자원량과 실제 사용량 구분
- Node MemoryPressure 및 Eviction 개념 이해
- Memory cgroup OOM과 Node Eviction 구분
- 실제 운영 상황에서 확인해야 할 지표 정리

## 2. 실습 환경

| 항목              | 환경                  |
| ----------------- | --------------------- |
| OS                | Rocky Linux 9.8       |
| Kubernetes        | v1.36.3               |
| Container Runtime | containerd 2.3.3      |
| Namespace         | `probe-resource-lab`  |
| 분석 Node         | `k8s-worker1`         |
| QoS 실습 Pod      | `qos-guaranteed-demo` |
| Container Image   | `nginx:1.28-alpine`   |
| Metrics Server    | 설치 및 사용 가능     |

기존 Kubernetes 클러스터에서 독립적인 Pod를 생성하고, 이미 존재하는 실습 Pod들의 QoS Class를 비교했습니다.

---

## 3. Kubernetes QoS Class란?

QoS(Quality of Service) Class는 Kubernetes가 Pod에 설정된 CPU·Memory Requests와 Limits를 기반으로 분류하는 서비스 품질 클래스입니다.

Pod는 다음 세 가지 중 하나로 분류됩니다.

| QoS Class  | 분류 조건                                                                  |
| ---------- | -------------------------------------------------------------------------- |
| Guaranteed | 모든 컨테이너의 CPU·Memory Requests와 Limits가 각각 동일한 양수            |
| Burstable  | Guaranteed가 아니면서 CPU 또는 Memory의 Requests·Limits가 하나 이상 설정됨 |
| BestEffort | 모든 컨테이너에 CPU·Memory Requests와 Limits가 설정되지 않음               |

QoS Class는 Pod의 실제 CPU·Memory 사용률로 결정되지 않습니다.

또한 QoS가 Guaranteed라고 해서 애플리케이션의 정상 동작이나 장애가 발생하지 않는다는 사실을 보장하는 것은 아닙니다.

## 4. 실습 1 — Guaranteed QoS

### 4.1 Pod 구성

**Manifest:** `qos-guaranteed-demo.yaml`

nginx 컨테이너의 CPU와 Memory Requests·Limits를 각각 동일하게 설정했습니다.

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "100m"
    memory: "64Mi"
```

| 자원   | Requests | Limits |
| ------ | -------: | -----: |
| CPU    |     100m |   100m |
| Memory |     64Mi |   64Mi |

CPU와 Memory 모두 Requests와 Limits가 같으므로 Guaranteed 조건을 충족합니다.

### 4.2 QoS Class 확인

실제 적용된 Pod 설정을 확인했습니다.

```bash
kubectl get pod qos-guaranteed-demo \
  -n probe-resource-lab -o yaml
```

확인된 주요 결과:

```text
qosClass: Guaranteed
phase: Running
restartCount: 0
```

또한 `Ready: True`를 확인했습니다.

따라서 Manifest의 자원 설정을 기반으로 예상한 Guaranteed QoS Class와 실제 Kubernetes의 분류 결과가 일치함을 확인했습니다.

## 5. 실습 2 — Burstable QoS

이전 Resource 실습에서 생성했던 Pod들의 설정을 비교했습니다.

### 5.1 cpu-throttle-demo

| 설정            | 값     |
| --------------- | ------ |
| CPU Requests    | 50m    |
| CPU Limits      | 100m   |
| Memory Requests | 미설정 |
| Memory Limits   | 미설정 |

CPU Requests와 Limits가 다르고, Memory Requests·Limits도 설정되지 않았으므로 Guaranteed 조건을 충족하지 않습니다.

일부 자원 설정이 존재하므로 Burstable로 분류됩니다.

### 5.2 memory-oom-demo

| 설정            | 값     |
| --------------- | ------ |
| CPU Requests    | 미설정 |
| CPU Limits      | 미설정 |
| Memory Requests | 16Mi   |
| Memory Limits   | 128Mi  |

Memory Requests와 Limits가 서로 다르므로 Burstable로 분류됩니다.

### 5.3 핵심 분석

Burstable은 Requests나 Limits가 일부 설정되어 있지만 Guaranteed 조건을 충족하지 않는 Pod입니다.

Burstable이라는 이유만으로 실제 CPU·Memory 사용량이 많다는 의미는 아닙니다.

또한 Burstable Pod도 설정된 Memory Limit을 초과하면 메모리 cgroup OOM이 발생할 수 있습니다.

## 6. 실습 3 — BestEffort QoS

이전에 생성했던 `probe-http-demo` Pod에서는 CPU와 Memory Requests·Limits를 설정하지 않았습니다.

```yaml
containers:
  - name: nginx
    image: nginx:1.28-alpine
```

해당 Pod의 QoS Class가 `BestEffort`로 분류된 것을 확인했습니다.

BestEffort는 CPU와 Memory Requests·Limits가 지정되지 않은 경우의 분류입니다.

이것이 CPU나 Memory를 사용할 수 없다는 의미는 아닙니다.

사용 가능한 Node 자원이 있다면 BestEffort Pod도 CPU와 Memory를 사용할 수 있습니다.

다만 Node 자원 압박이 발생했을 때 Requests가 없는 점이 Eviction 우선순위 판단에 불리하게 작용할 수 있습니다.

---

## 7. 세 가지 QoS Class 조회

기존 Pod들의 QoS Class를 비교하기 위해 다음 명령어를 사용했습니다.

```bash
kubectl get pods -n probe-resource-lab \
  -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass,STATUS:.status.phase
```

실습을 통해 다음과 같은 사례를 확인했습니다.

| Pod                       | QoS Class  |
| ------------------------- | ---------- |
| qos-guaranteed-demo       | Guaranteed |
| cpu-throttle-demo         | Burstable  |
| memory-oom-demo           | Burstable  |
| cpu-request-pending의 Pod | Burstable  |
| probe-http-demo           | BestEffort |
| probe-tcp-demo            | BestEffort |
| probe-exec-demo           | BestEffort |

이를 통해 실제 클러스터에서 Guaranteed, Burstable, BestEffort의 세 가지 QoS Class가 모두 존재한다는 사실을 확인했습니다.

---

## 8. Node의 Requests·Limits 확인

### 8.1 Node 상태 조회

다음 명령어로 `k8s-worker1`의 자원 정보를 확인했습니다.

```bash
kubectl describe node k8s-worker1
```

`Allocated resources` 영역에서 다음 값을 확인했습니다.

| 자원   | Requests 합계 | Limits 합계 |
| ------ | ------------: | ----------: |
| CPU    |          150m |        200m |
| Memory |          80Mi |       192Mi |

이 값은 조회 당시 해당 Node에 배치된 Pod들의 Requests 및 Limits 합계입니다.

### 8.2 Requests 합계의 의미

CPU Requests 합계가 `150m`라는 것은 Node에서 실제 CPU를 150m만 사용하고 있다는 뜻이 아닙니다.

Scheduler가 Pod 배치를 고려할 때 사용되는 예약 자원량과 관련된 값입니다.

마찬가지로 Memory Requests 합계가 `80Mi`라고 해서 실제 Node 메모리 사용량이 80Mi인 것도 아닙니다.

### 8.3 Limits 합계의 의미

Limits 합계는 Pod에 설정된 자원 제한값을 집계한 것입니다.

실제 사용량을 측정한 값이 아닙니다.

따라서 Requests 합계, Limits 합계, 실제 자원 사용량은 각각 구분해서 해석해야 합니다.

---

## 9. Node의 실제 자원 사용량 확인

### 9.1 Metrics Server 조회

다음 명령어를 사용했습니다.

```bash
kubectl top node k8s-worker1
```

실제 측정 결과:

| 항목   | 측정값 | 사용률 |
| ------ | -----: | -----: |
| CPU    |   291m |    14% |
| Memory | 1116Mi |    69% |

이 값은 측정 시점의 실제 Node 자원 사용량입니다.

### 9.2 Requests와 실제 사용량 비교

| 항목   | Requests 합계 | 실제 Node 사용량 |
| ------ | ------------: | ---------------: |
| CPU    |          150m |             291m |
| Memory |          80Mi |           1116Mi |

Requests와 실제 사용량 사이에 차이가 있음을 확인했습니다.

특히 실제 Node 메모리 사용량에는 Pod 외에도 운영체제와 시스템 프로세스 등 Node의 자원 사용이 반영됩니다.

따라서 이 차이 전체를 특정 Pod의 메모리 초과 사용량으로 해석해서는 안 됩니다.

두 명령어는 측정 목적과 집계 대상이 다르며, 수치는 조회 당시의 일시적인 값입니다.

### 9.3 Pod별 자원 사용량 확인

다음 명령어로 Namespace 내 Pod의 실제 메모리 사용량을 비교했습니다.

```bash
kubectl top pod -n probe-resource-lab --sort-by=memory
```

Pod의 Requests·Limits 설정과 실제 측정값을 비교하면 자원 사용 패턴을 파악하는 데 도움이 됩니다.

다만 Metrics Server의 값은 순간적인 사용량에 가까우므로 장기간 추세나 최대 사용량까지 파악하려면 별도의 모니터링이 필요합니다.

---

## 10. Node Pressure 상태 확인

`kubectl describe node k8s-worker1`로 다음 조건을 확인했습니다.

| Condition      | 실제 결과 |
| -------------- | --------- |
| MemoryPressure | False     |
| DiskPressure   | False     |
| PIDPressure    | False     |
| Ready          | True      |

조회 당시 Node는 정상 Ready 상태였으며, 메모리·디스크·PID 압박 상태가 감지되지 않았습니다.

특히 `MemoryPressure: False`는 해당 시점에 kubelet이 Node의 메모리 압박 조건을 보고하지 않았다는 의미입니다.

**MemoryPressure가 False라고 해서 개별 컨테이너의 Memory Limit 초과로 인한 OOM이 발생하지 않는 것은 아닙니다.**

---

## 11. Node Eviction이란?

Node Eviction은 Node의 자원이 부족해져 kubelet이 안정성을 유지하기 위해 Pod를 종료하는 동작입니다.

대표적으로 다음과 같은 자원 압박과 관련될 수 있습니다.

- MemoryPressure
- DiskPressure
- PIDPressure

kubelet은 Node 자원 압박을 감지하면 자원 회수를 시도하고, 필요에 따라 Pod를 Eviction 대상으로 선택할 수 있습니다.

Eviction에 의해 제거된 Pod는 `Evicted` 상태로 표시될 수 있습니다.

Deployment가 관리하는 Pod가 Evicted되면 ReplicaSet은 원하는 Replica 수를 유지하기 위해 대체 Pod를 생성할 수 있습니다.

다만 Node의 자원 압박이 계속된다면 대체 Pod 역시 정상적으로 실행되지 못할 수 있습니다.

**이번 실습에서는 Node Eviction을 실제로 재현하지 않았습니다.**

따라서 이후 내용은 공식 동작 원리에 대한 개념 분석입니다.

---

## 12. QoS Class와 Eviction의 관계

### 12.1 Eviction 순서를 결정하는 요소

Node 자원 압박에 따른 Pod 선택 시 kubelet은 다음 요소를 고려합니다.

1. 부족한 자원에 대해 Pod의 실제 사용량이 Requests를 초과했는지
2. Pod Priority
3. Requests 대비 실제 자원 사용량

QoS Class를 이용하면 일반적인 Eviction 가능성을 예상할 수 있지만, **kubelet이 단순히 BestEffort → Burstable → Guaranteed라는 고정 순서만으로 Pod를 선택하는 것은 아닙니다.**

실제 Eviction 결정에는 자원 사용량과 Requests, Priority 등의 요소가 함께 작용합니다.

### 12.2 예시 비교

동일한 Priority의 Pod들이 있고 Node의 메모리가 부족하다고 가정합니다.

| Pod | Memory Requests | 실제 사용량 |
| --- | --------------: | ----------: |
| A   |             0Mi |        20Mi |
| B   |            64Mi |        32Mi |
| C   |           128Mi |        64Mi |

Pod A는 Memory Requests가 0이지만 실제 메모리를 사용하고 있습니다.

반면 B와 C는 실제 사용량이 Requests보다 낮습니다.

따라서 메모리 자원 압박에서 A가 먼저 Eviction 대상으로 고려될 가능성이 높습니다.

이는 QoS Class의 이름만으로 결정한 것이 아니라, **메모리 사용량이 Requests를 초과하는지**를 기준으로 분석한 것입니다.

### 12.3 Guaranteed의 의미

Guaranteed Pod는 CPU와 Memory Requests·Limits가 동일하게 설정된 Pod입니다.

일반적인 메모리 압박 상황에서 Eviction 가능성이 상대적으로 낮지만, 모든 장애로부터 보호되는 것은 아닙니다.

Memory Limit을 초과하면 Guaranteed Pod에서도 OOM이 발생할 수 있습니다.

또한 Node의 특수한 자원 고갈 상황에서는 Guaranteed Pod 역시 Eviction 대상이 될 수 있습니다.

---

## 13. Memory OOM과 Node Eviction 비교

| 항목                      | Memory cgroup OOM               | Node-pressure Eviction           |
| ------------------------- | ------------------------------- | -------------------------------- |
| 발생 원인                 | 컨테이너 메모리 한도 초과 등    | Node 자원 압박                   |
| 주요 처리 주체            | Linux 커널 OOM Killer           | kubelet                          |
| 종료 대상                 | 프로세스, 경우에 따라 컨테이너  | Pod                              |
| 대표적인 관찰 정보        | 종료 코드 137, 커널 OOM 로그    | Evicted, Node Conditions, Events |
| MemoryPressure False일 때 | 발생 가능                       | 현재 메모리 압박이 보고되지 않음 |
| Deployment 관리 시        | 주 컨테이너 종료 후 재시작 가능 | 대체 Pod 생성 가능               |

### 13.1 Memory OOM 실습과의 연결

이전 `memory-oom-demo`에서는 Memory Limit 48Mi인 컨테이너에서 80Mi 메모리 할당을 시도했습니다.

그 결과 Worker Node의 커널 로그에서 다음 메시지를 확인했습니다.

```text
oom-kill:constraint=CONSTRAINT_MEMCG
Memory cgroup out of memory: Killed process
```

이는 Memory cgroup 제한에 따른 OOM의 증거입니다.

따라서 해당 장애를 Node-pressure Eviction으로 해석하지 않았습니다.

### 13.2 Node Eviction 조사 시 확인할 항목

실제 운영에서 Node Eviction이 의심된다면 다음을 확인해야 합니다.

- Pod의 종료 상태와 Reason
- Pod Events
- Node Conditions
- Node의 실제 자원 사용량
- Pod별 Requests와 실제 사용량
- Pod Priority
- kubelet 로그 및 Eviction 관련 이벤트

Node 자원 압박은 여러 Pod에 영향을 줄 수 있으므로, 개별 컨테이너의 Memory Limit 문제와 구분해야 합니다.

---

## 14. 운영 관점에서 배운 점

### 14.1 QoS Class는 실제 사용률이 아닌 설정으로 결정된다

Guaranteed, Burstable, BestEffort는 CPU·Memory Requests와 Limits 설정을 기반으로 결정됩니다.

Pod가 적은 양의 CPU·Memory를 사용하더라도 Requests·Limits 설정이 없으면 BestEffort일 수 있습니다.

### 14.2 Requests와 실제 사용량은 다르다

`kubectl describe node`의 Allocated resources와 `kubectl top node`의 사용량은 의미가 다릅니다.

두 값을 혼동하면 Scheduler의 자원 판단이나 Node의 실제 압박 상황을 잘못 해석할 수 있습니다.

### 14.3 Guaranteed가 모든 장애를 방지하지 않는다

Guaranteed Pod도 메모리 한도 초과로 OOM이 발생할 수 있으며, Node의 특수한 자원 압박 상황에서는 Eviction 가능성이 존재합니다.

따라서 QoS Class만으로 애플리케이션의 안정성이 보장된다고 판단하지 않아야 합니다.

### 14.4 Eviction은 여러 조건을 함께 고려한다

Pod Priority, Requests 대비 실제 사용량, 부족한 자원의 종류 등을 함께 확인해야 합니다.

QoS Class는 관련된 중요한 분류 기준이지만, Eviction 순서를 단독으로 결정하는 절대적인 기준은 아닙니다.

### 14.5 실제 수행하지 않은 장애는 구분해서 기록한다

이번 실습에서는 세 가지 QoS Class와 Node의 자원 사용량을 실제로 확인했습니다.

그러나 의도적인 Node MemoryPressure나 Eviction은 발생시키지 않았습니다.

따라서 Eviction 동작 원리는 공식 문서를 참고하여 정리하고, 실제 관찰 결과와 구분했습니다.

---

## 15. 주요 검증 명령어

| 명령어                                   | 확인 목적                                          |
| ---------------------------------------- | -------------------------------------------------- |
| `kubectl get pods -o custom-columns=...` | Pod별 QoS Class                                    |
| `kubectl get pod -o yaml`                | Requests, Limits, QoS 확인                         |
| `kubectl describe node`                  | Allocatable, Requests·Limits 합계, Node Conditions |
| `kubectl top node`                       | 실제 Node CPU·Memory 사용량                        |
| `kubectl top pod`                        | Pod별 실제 자원 사용량                             |
| `kubectl describe pod`                   | Pod 종료 사유와 Events                             |

## 16. 실습 파일

| 파일                       | 설명                                                 |
| -------------------------- | ---------------------------------------------------- |
| `qos-guaranteed-demo.yaml` | CPU·Memory Requests와 Limits가 동일한 Guaranteed Pod |

Burstable과 BestEffort는 앞선 실습에서 생성한 Pod들을 이용해 비교했습니다.

## 17. 참고 자료

- [Kubernetes 공식 문서 — Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
- [Kubernetes 공식 문서 — Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

## 18. 결론

이번 실습에서는 Guaranteed, Burstable, BestEffort의 세 가지 QoS Class를 실제 Pod 설정과 Kubernetes의 분류 결과로 확인했습니다.

또한 Node의 Requests·Limits 합계와 Metrics Server가 측정한 실제 사용량을 비교하여 자원 예약량과 실제 자원 사용량이 서로 다른 개념임을 확인했습니다.

Node Eviction은 실제로 발생시키지 않았지만 공식 동작 원리를 바탕으로 Pod Priority, Requests 초과 여부, 실제 사용량이 Eviction 판단에 어떻게 연결되는지 학습했습니다.

이번 실습을 통해 **Kubernetes의 자원 관리는 단순한 Requests·Limits 설정을 넘어 QoS, 실제 사용량, Node 상태, OOM 및 Eviction을 함께 이해해야 한다는 점**을 학습했습니다.
