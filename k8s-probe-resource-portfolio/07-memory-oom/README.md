# Kubernetes Memory Limits & OOM — 메모리 한도 초과 장애 재현 및 복구

## 1. 실습 개요

Kubernetes 컨테이너의 Memory Limit을 초과하는 상황을 재현하고, Linux 커널 로그와 Kubernetes 상태 정보를 이용해 OOM(Out Of Memory) 장애의 원인을 분석한 실습입니다.

이번 실습에서는 메모리 제한이 `48Mi`인 Python 컨테이너에서 `80Mi`의 메모리 할당을 시도했습니다.

이후 프로세스 종료, 컨테이너 재시작, cgroup 메모리 설정과 Worker Node 커널 로그를 확인했습니다.

장애 원인을 확인한 뒤 Memory Limit을 `128Mi`로 수정하고 동일한 메모리 할당 테스트가 성공하는지 검증했습니다.

**핵심 목표는 Memory Requests와 Limits의 차이를 이해하고, 종료 코드만으로 OOM 원인을 단정하지 않고 실제 커널 증거를 기반으로 분석하는 것입니다.**

## 2. 실습 환경

| 항목                  | 설정                 |
| --------------------- | -------------------- |
| OS                    | Rocky Linux 9.8      |
| Kubernetes            | v1.36.3              |
| Container Runtime     | containerd 2.3.3     |
| Namespace             | `probe-resource-lab` |
| Deployment            | `memory-oom-demo`    |
| Container             | `memory-test`        |
| Image                 | `python:3.12-alpine` |
| Replicas              | 1                    |
| Deployment Strategy   | Recreate             |
| Memory Requests       | 16Mi                 |
| 초기 Memory Limits    | 48Mi                 |
| 복구 후 Memory Limits | 128Mi                |
| 메모리 할당 테스트    | 80Mi                 |

기존 Kubernetes 클러스터에서 독립적인 Deployment를 사용하여 실습했습니다.

## 3. Memory Requests와 Limits

### 3.1 Memory Requests

Memory Requests는 Kubernetes Scheduler가 Pod를 Node에 배치할 때 고려하는 메모리 요청량입니다.

이번 실습에서는 다음과 같이 설정했습니다.

`requests.memory: 16Mi`

Requests는 컨테이너가 실제로 사용할 수 있는 메모리의 최대값을 의미하지 않습니다.

컨테이너의 실제 메모리 사용량은 Requests보다 커질 수 있습니다.

### 3.2 Memory Limits

Memory Limits는 컨테이너의 메모리 사용량에 대한 상한을 설정합니다.

이번 실습의 초기 설정은 다음과 같았습니다.

```yaml
resources:
  requests:
    memory: "16Mi"
  limits:
    memory: "48Mi"
```

Linux는 cgroup을 통해 컨테이너의 메모리 사용을 관리합니다.

메모리 한도에 도달하여 메모리를 충분히 회수할 수 없으면 커널 OOM Killer가 프로세스를 종료할 수 있습니다.

CPU Limit에 의한 Throttling과 달리 Memory Limit 초과는 실제 프로세스 종료로 이어질 수 있습니다.

---

## 4. 실습용 Deployment 구성

Python 프로세스를 실행하고 일정 시간 동안 유지하도록 구성했습니다.

```yaml
command:
  - python
  - -u
  - -c
args:
  - "import time; print('memory demo ready', flush=True); time.sleep(3600)"
```

메모리 부하를 발생시키는 코드는 컨테이너의 시작 명령에 포함하지 않았습니다.

대신 실행 중인 컨테이너에서 `kubectl exec`를 이용해 별도 Python 프로세스를 실행하여 메모리 할당을 시도했습니다.

이를 통해 메모리 한도를 초과하는 상황을 의도적으로 재현했습니다.

## 5. 장애 재현 — Memory Limit 48Mi

### 5.1 cgroup 메모리 한도 확인

다음 명령어로 컨테이너에 적용된 실제 메모리 한도를 확인했습니다.

```bash
kubectl exec -n probe-resource-lab deployment/memory-oom-demo \
  -- cat /sys/fs/cgroup/memory.max
```

실제 출력:

```text
50331648
```

`memory.max`의 단위는 Byte입니다.

```text
48 × 1024 × 1024 = 50,331,648 Bytes
```

따라서 Kubernetes에서 설정한 `48Mi`가 실제 cgroup 메모리 한도로 적용된 것을 확인했습니다.

### 5.2 초기 메모리 사용량 확인

메모리 부하 테스트 전 다음 파일을 확인했습니다.

```bash
kubectl exec -n probe-resource-lab deployment/memory-oom-demo \
  -- cat /sys/fs/cgroup/memory.current
```

실제 측정값:

```text
4169728
```

약 `3.98Mi`에 해당하는 값입니다.

이는 측정 당시 해당 cgroup에 기록된 메모리 사용량입니다.

또한 `memory.events`를 확인했을 때 초기 이벤트 카운터는 모두 0이었습니다.

### 5.3 메모리 할당 테스트

실행 중인 컨테이너에서 다음 Python 코드를 실행했습니다.

```bash
kubectl exec -n probe-resource-lab deployment/memory-oom-demo \
  -- python -c 'x = bytearray(80 * 1024 * 1024); print(len(x))'
```

이 코드는 약 `80Mi` 크기의 `bytearray`를 생성합니다.

```text
80 × 1024 × 1024 = 83,886,080 Bytes
```

초기 Memory Limit인 48Mi보다 큰 메모리 할당을 시도한 것입니다.

**주의:** 해당 명령어는 실제 메모리 부족 상황을 유발하므로 운영 서비스나 공유 리소스에서는 임의로 실행하면 안 됩니다.

### 5.4 실제 장애 결과

테스트 결과 명령어가 정상적으로 완료되지 않았으며 다음 종료 코드가 확인되었습니다.

```text
command terminated with exit code 137
```

종료 코드 137은 일반적으로 SIGKILL에 따른 프로세스 종료를 나타냅니다.

그러나 **종료 코드 137만으로 OOMKilled를 확정할 수는 없습니다.**

SIGKILL은 OOM 외의 다른 원인으로도 발생할 수 있기 때문입니다.

따라서 Pod 상태와 Worker Node 커널 로그를 추가로 조사했습니다.

---

## 6. Kubernetes 컨테이너 상태 확인

### 6.1 Pod 상세 정보 조사

다음 명령어로 Pod 상태를 조사했습니다.

```bash
kubectl describe pod <Pod명> -n probe-resource-lab
```

실습 당시 다음 정보를 확인했습니다.

```text
Last State: Terminated
Reason: Error
Exit Code: 137
Restart Count: 1
```

컨테이너 종료 후 재시작이 발생했음을 확인했습니다.

다만 Kubernetes의 `Last State`에는 `OOMKilled` 대신 `Error`가 표시되었습니다.

따라서 이 출력만으로 정확한 종료 원인을 단정하지 않고 Linux 커널 로그를 확인했습니다.

### 6.2 이전 컨테이너 로그 확인

다음 명령어로 이전 컨테이너 로그를 조회했습니다.

```bash
kubectl logs <Pod명> \
  -n probe-resource-lab \
  --previous
```

이전 로그에서는 다음 메시지를 확인했습니다.

```text
memory demo ready
```

애플리케이션이 시작되었다는 사실은 확인할 수 있었지만, 이 로그만으로 메모리 부족에 의한 강제 종료를 확정할 수는 없었습니다.

OOM 원인 분석을 위해 Worker Node의 커널 로그를 조사했습니다.

---

## 7. Linux 커널 로그를 이용한 OOM 원인 확인

### 7.1 커널 로그 조사

장애가 발생한 Worker Node에서 `journalctl -k`를 이용해 해당 시점의 커널 메시지를 조회했습니다.

실습 당시 장애 발생 시각은 2026년 11월 13일 16시 29분경(KST)이었습니다.

```bash
sudo journalctl -k \
  --since "2026-11-13 16:29:00" \
  --until "2026-11-13 16:30:30" \
  --no-pager
```

### 7.2 실제 확인한 OOM 기록

커널 로그에서 다음 메시지를 확인했습니다.

```text
python invoked oom-killer
```

```text
memory: usage 49152kB, limit 49152kB, failcnt 100
```

```text
oom-kill:constraint=CONSTRAINT_MEMCG
```

```text
Memory cgroup out of memory: Killed process
```

또한 Python 프로세스들이 종료된 기록을 확인했습니다.

이 메시지들은 이번 장애가 단순한 애플리케이션 종료가 아니라 **메모리 cgroup의 제한으로 발생한 OOM 상황**이라는 직접적인 증거입니다.

### 7.3 CONSTRAINT_MEMCG의 의미

`CONSTRAINT_MEMCG`는 OOM 판단이 메모리 cgroup의 제한과 관련되어 있음을 나타냅니다.

이번 경우에는 Node 전체의 메모리가 부족해서 발생한 장애라고 단정할 수 없습니다.

컨테이너의 Memory Limit인 `48Mi`에 도달하여 cgroup 범위에서 OOM이 발생했습니다.

### 7.4 컨테이너 전체 종료 원인

커널 로그에서는 `memory.oom.group` 설정과 관련된 기록도 확인했습니다.

실습 당시에는 메모리 한도 초과로 실행 중이던 Python 프로세스뿐 아니라 컨테이너의 주 프로세스도 종료되었습니다.

그 결과 kubelet이 컨테이너의 종료를 감지하고 재시작 정책에 따라 컨테이너를 다시 시작했습니다.

**모든 cgroup OOM에서 반드시 컨테이너 전체가 종료되는 것은 아닙니다.**

실제 종료 범위는 커널의 OOM 처리와 cgroup 설정 등에 따라 달라질 수 있습니다.

---

## 8. memory.events 조회 결과 해석

OOM 발생 직후 `memory.events`를 조회했지만 이벤트 카운터는 모두 0으로 표시되었습니다.

처음에는 커널 OOM이 발생하지 않은 것처럼 보일 수 있습니다.

그러나 이번 실습에서는 OOM 이후 컨테이너가 재시작되었습니다.

따라서 조회한 `memory.events`는 이미 종료된 컨테이너의 이전 cgroup 통계와 동일하지 않을 수 있습니다.

**재시작 이후의 cgroup 통계가 0이라는 사실만으로 이전 OOM 발생을 부정할 수는 없습니다.**

이번 장애는 다음 증거를 종합하여 판단했습니다.

1. 메모리 한도 48Mi 확인
2. 80Mi 할당 테스트 중 종료 코드 137 발생
3. 컨테이너 종료 및 재시작 확인
4. Worker Node 커널 로그의 `CONSTRAINT_MEMCG` 확인
5. `Memory cgroup out of memory` 메시지 확인

특히 커널 로그가 OOM 발생을 확인하는 핵심 근거였습니다.

---

## 9. 장애 복구 — Memory Limit 128Mi

### 9.1 복구 조치

실습에 필요한 메모리 할당량을 고려하여 Memory Limit을 변경했습니다.

| 항목               | 변경 전 | 변경 후 |
| ------------------ | ------- | ------- |
| Memory Requests    | 16Mi    | 16Mi    |
| Memory Limits      | 48Mi    | 128Mi   |
| 메모리 할당 테스트 | 80Mi    | 80Mi    |

최종 설정:

```yaml
resources:
  requests:
    memory: "16Mi"
  limits:
    memory: "128Mi"
```

이번에는 Memory Requests가 아니라 Limits를 수정했습니다.

Requests만 높여도 기존 Memory Limit인 48Mi는 그대로 유지되기 때문에 동일한 메모리 할당 테스트의 직접적인 해결책이 되지 못합니다.

### 9.2 Deployment 변경 적용

```bash
kubectl apply --dry-run=server -f memory-oom-demo.yaml
kubectl apply -f memory-oom-demo.yaml
```

이번 Deployment는 `Recreate` 전략으로 구성되어 있으므로, 변경된 Pod Template을 적용할 때 기존 Pod를 종료한 후 새 Pod를 생성하도록 설정했습니다.

### 9.3 변경된 cgroup 제한 확인

신규 컨테이너에서 `memory.max`를 확인했습니다.

실제 출력:

```text
134217728
```

계산하면 다음과 같습니다.

```text
128 × 1024 × 1024 = 134,217,728 Bytes
```

변경한 Memory Limit인 `128Mi`가 실제 cgroup 설정에 반영된 것을 확인했습니다.

---

## 10. 동일 메모리 할당 테스트로 복구 검증

### 10.1 테스트 재실행

메모리 한도를 변경한 뒤 기존과 동일한 `80Mi` 할당 테스트를 수행했습니다.

```bash
kubectl exec -n probe-resource-lab deployment/memory-oom-demo \
  -- python -c 'x = bytearray(80 * 1024 * 1024); print(len(x))'
```

### 10.2 실제 실행 결과

```text
83886080
```

이번에는 Python 프로세스가 정상적으로 메모리를 할당하고 할당 크기를 출력했습니다.

실패했던 동일한 작업이 Memory Limit을 변경한 이후 성공한 것입니다.

### 10.3 컨테이너 상태 확인

복구 이후 신규 Pod는 다음 상태로 확인되었습니다.

| 항목             | 결과                               |
| ---------------- | ---------------------------------- |
| Pod              | `memory-oom-demo-57bc88cbc7-c8jjt` |
| STATUS           | Running                            |
| READY            | 1/1                                |
| Restart Count    | 0                                  |
| Memory Limit     | 128Mi                              |
| 80Mi 할당 테스트 | 성공                               |

해당 테스트 이후 확인한 `memory.events` 카운터도 모두 0이었습니다.

이를 통해 **복구된 설정에서 동일한 메모리 할당 작업이 정상적으로 완료되었음**을 확인했습니다.

단, 이번 한 번의 테스트 성공으로 모든 메모리 부하 상황에서 OOM이 발생하지 않는다고 보장할 수는 없습니다.

---

## 11. Deployment 최종 상태 확인

최종 Deployment를 조회했습니다.

```bash
kubectl get deployment memory-oom-demo \
  -n probe-resource-lab -o yaml
```

실제 결과:

| 항목                | 값       |
| ------------------- | -------- |
| Desired Replicas    | 1        |
| Ready Replicas      | 1        |
| Available Replicas  | 1        |
| Updated Replicas    | 1        |
| Deployment Revision | 2        |
| Strategy            | Recreate |

Deployment의 Available 조건이 True이고, 새 ReplicaSet이 정상적으로 준비된 것을 확인했습니다.

---

## 12. CPU Throttling과 Memory OOM 비교

| 구분             | CPU Throttling                       | Memory OOM                                      |
| ---------------- | ------------------------------------ | ----------------------------------------------- |
| 관련 설정        | `limits.cpu`                         | `limits.memory`                                 |
| 주요 제어 방식   | CPU 실행 시간 제한                   | 메모리 사용 한도 적용                           |
| 한도 초과 시     | CPU 실행이 일시적으로 제한될 수 있음 | OOM Killer가 프로세스를 종료할 수 있음          |
| 컨테이너 재시작  | Throttling 자체로는 발생하지 않음    | 주 프로세스 종료 시 재시작 가능                 |
| 주요 cgroup 파일 | `cpu.max`, `cpu.stat`                | `memory.max`, `memory.current`, `memory.events` |
| 추가 확인        | CPU Throttling 통계                  | 종료 상태, 커널 OOM 로그                        |

두 현상을 구분하는 것이 중요합니다.

CPU Throttling은 일반적으로 컨테이너를 종료시키지 않지만, Memory OOM은 프로세스 강제 종료로 이어질 수 있습니다.

---

## 13. Memory OOM과 Node Eviction의 차이

이번 실습에서 발생한 장애는 메모리 cgroup 제한에 따른 OOM이었습니다.

Node Eviction과는 구분해야 합니다.

### Memory cgroup OOM

컨테이너가 설정된 메모리 한도를 초과하여 충분한 메모리를 확보할 수 없을 때 발생할 수 있습니다.

Linux 커널이 해당 cgroup의 프로세스를 종료할 수 있습니다.

Node의 `MemoryPressure`가 False이더라도 발생할 수 있습니다.

### Node MemoryPressure Eviction

Node 전체에 메모리 압박이 발생했을 때 kubelet이 Eviction 정책에 따라 Pod를 종료하는 상황입니다.

이 경우 Pod가 Evicted 상태가 될 수 있으며, Deployment가 관리하는 Pod라면 ReplicaSet이 대체 Pod를 생성할 수 있습니다.

이번 실습에서는 Node Eviction을 재현하지 않았습니다.

**따라서 이번 장애를 Eviction이 아니라 메모리 cgroup OOM으로 분류했습니다.**

---

## 14. 운영 관점에서 배운 점

### 14.1 종료 코드 137만으로 OOMKilled를 확정하지 않는다

종료 코드 137은 SIGKILL에 의한 종료를 의미하지만, SIGKILL이 발생한 원인은 여러 가지일 수 있습니다.

따라서 Kubernetes의 종료 상태뿐 아니라 Events, 컨테이너 로그, Worker Node 커널 로그 등을 함께 확인해야 합니다.

이번 실습에서는 커널 로그의 `CONSTRAINT_MEMCG`와 `Memory cgroup out of memory` 메시지로 원인을 확인했습니다.

### 14.2 Memory Requests와 Limits를 혼동하지 않는다

Requests는 주로 스케줄링 판단에 사용되며, Limits는 메모리 사용 한도를 설정합니다.

Requests보다 실제 메모리 사용량이 많다는 이유만으로 바로 OOM이 발생하는 것은 아닙니다.

이번 장애의 직접적인 원인은 Memory Limit인 48Mi를 초과한 것입니다.

### 14.3 OOM 발생 후 통계가 초기화될 수 있다

컨테이너가 재시작되면 새 cgroup에서 조회한 메모리 통계가 이전 종료된 cgroup의 상태를 그대로 보여주지 않을 수 있습니다.

따라서 장애 직후 현재 통계가 정상으로 표시되더라도 이전 장애 발생 시점의 이벤트와 커널 로그를 확인해야 합니다.

### 14.4 Memory Limit을 무조건 높이는 것은 정답이 아니다

이번 실습에서는 의도된 80Mi 메모리 할당 작업이 48Mi 제한을 초과했으므로 Limits를 128Mi로 조정하여 복구했습니다.

그러나 실제 운영에서는 다음을 함께 검토해야 합니다.

- 애플리케이션의 정상 메모리 사용량
- 최대 부하 시 메모리 사용량
- 메모리 누수 가능성
- Node의 실제 가용 메모리
- 적절한 Requests 및 Limits 설정

Limits를 높이는 것만으로 메모리 누수나 비효율적인 메모리 사용 문제가 해결되는 것은 아닙니다.

### 14.5 장애 복구는 동일한 조건에서 검증해야 한다

단순히 Memory Limit을 변경하고 Deployment가 Ready 상태가 되었다고 해서 복구가 검증된 것은 아닙니다.

이번 실습에서는 장애를 일으킨 것과 동일한 80Mi 할당 테스트를 다시 실행했습니다.

**동일한 테스트가 실패에서 성공으로 바뀌었다는 사실**이 복구 조치의 효과를 보여주는 중요한 근거입니다.

---

## 15. 주요 조사 명령어

| 명령어                                               | 확인 목적                   |
| ---------------------------------------------------- | --------------------------- |
| `kubectl get pods`                                   | Pod 상태 및 재시작 횟수     |
| `kubectl describe pod`                               | 컨테이너 종료 상태와 Events |
| `kubectl logs --previous`                            | 이전 컨테이너 로그          |
| `kubectl exec ... cat /sys/fs/cgroup/memory.max`     | 실제 메모리 한도            |
| `kubectl exec ... cat /sys/fs/cgroup/memory.current` | 현재 cgroup 메모리 사용량   |
| `kubectl exec ... cat /sys/fs/cgroup/memory.events`  | 메모리 이벤트 카운터        |
| `journalctl -k`                                      | Worker Node 커널 OOM 기록   |

## 16. 실습 파일

| 파일                   | 설명                                          |
| ---------------------- | --------------------------------------------- |
| `memory-oom-demo.yaml` | Memory Limit을 128Mi로 수정한 최종 Deployment |

장애 발생 당시의 48Mi 설정과 복구 과정은 이 README에 기록했습니다.

## 17. 결론

이번 실습에서는 메모리 제한이 48Mi인 컨테이너에서 80Mi의 메모리 할당을 시도하여 OOM 장애를 재현했습니다.

종료 코드 137과 컨테이너 재시작을 확인한 뒤, Worker Node 커널 로그의 `CONSTRAINT_MEMCG` 및 `Memory cgroup out of memory` 메시지를 근거로 실제 원인을 확인했습니다.

이후 Memory Limit을 128Mi로 변경하고 동일한 80Mi 할당 테스트가 성공하는 것을 검증했습니다.

이번 실습을 통해 **메모리 장애 대응에서는 단순히 컨테이너가 종료됐다는 사실보다, 어떤 자원 제한에 의해 종료됐는지 확인하고 동일 조건에서 복구 결과를 검증하는 과정이 중요하다**는 점을 학습했습니다.
