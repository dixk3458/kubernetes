# Kubernetes CPU Limits & Throttling — cgroup 기반 CPU 제한 검증

## 1. 실습 개요

Kubernetes의 CPU Requests와 Limits가 어떤 역할을 하는지 확인하고, CPU Limit을 초과하는 부하를 발생시켜 실제 CPU Throttling 현상을 관찰한 실습입니다.

이번 실습에서는 컨테이너의 CPU Limit을 `100m`로 설정했습니다.

이후 Linux cgroup v2의 `cpu.max`와 `cpu.stat`을 직접 조회하여 CPU 제한 설정과 Throttling 발생 여부를 검증했습니다.

**핵심 목표는 CPU Throttling을 컨테이너 재시작이나 OOMKilled와 구분하고, 실제 커널 통계를 근거로 설명하는 것입니다.**

## 2. 실습 환경

| 항목              | 설정                 |
| ----------------- | -------------------- |
| OS                | Rocky Linux 9.8      |
| Kubernetes        | v1.36.3              |
| Container Runtime | containerd 2.3.3     |
| Namespace         | `probe-resource-lab` |
| Pod               | `cpu-throttle-demo`  |
| Container         | `cpu-test`           |
| Image             | `nginx:1.28-alpine`  |
| Node              | `k8s-worker1`        |
| CPU Requests      | `50m`                |
| CPU Limits        | `100m`               |
| QoS Class         | Burstable            |
| cgroup            | v2                   |

## 3. CPU Requests와 Limits

### 3.1 CPU Requests

CPU Requests는 Kubernetes Scheduler가 Pod를 Node에 배치할 때 고려하는 CPU 요청량입니다.

이번 실습에서는 다음과 같이 설정했습니다.

```yaml
resources:
  requests:
    cpu: "50m"
```

`50m`는 CPU 0.05개에 해당합니다.

Requests는 컨테이너가 실제로 항상 사용하는 CPU 양이 아닙니다.

### 3.2 CPU Limits

CPU Limits는 컨테이너의 CPU 실행 시간을 제한하는 설정입니다.

```yaml
resources:
  limits:
    cpu: "100m"
```

`100m`는 CPU 0.1개에 해당하는 CPU 처리 용량입니다.

CPU 부하가 Limit을 초과하려고 하면 Linux의 CPU 대역폭 제어 기능에 의해 실행이 일시적으로 제한될 수 있습니다.

이 현상을 **CPU Throttling**이라고 합니다.

CPU Throttling은 컨테이너를 종료시키는 것이 아니라 CPU 실행 시간을 제한하는 동작입니다.

---

## 4. 실습용 Pod 구성

**Manifest:** `cpu-throttle-demo.yaml`

컨테이너가 평소에는 대기 상태를 유지하도록 설정했습니다.

```yaml
command:
  - /bin/sh
  - -c
args:
  - "while true; do sleep 3600; done"
```

CPU 부하가 필요할 때는 `kubectl exec`로 별도의 명령어를 실행했습니다.

이렇게 구성하면 일반적인 대기 상태와 CPU 부하 발생 시점의 통계를 비교할 수 있습니다.

---

## 5. cgroup CPU Limit 설정 확인

### 5.1 확인 명령어

```bash
kubectl exec -n probe-resource-lab cpu-throttle-demo \
  -- cat /sys/fs/cgroup/cpu.max
```

### 5.2 실제 출력

```text
10000 100000
```

cgroup v2의 `cpu.max`는 다음 두 값으로 구성됩니다.

| 값       | 의미                                         |
| -------- | -------------------------------------------- |
| `10000`  | CPU 실행에 사용할 수 있는 할당량(마이크로초) |
| `100000` | CPU 대역폭 제어 기간(마이크로초)             |

이를 계산하면 다음과 같습니다.

```text
10000 / 100000 = 0.1 CPU
```

즉, 이번 컨테이너는 100ms의 제어 기간마다 총 10ms에 해당하는 CPU 실행 시간을 할당받도록 설정되어 있습니다.

이는 Kubernetes Manifest의 `limits.cpu: 100m`와 일치합니다.

단, 이 설정은 CPU 실행 시간에 대한 대역폭 제한이며 모든 순간에 CPU 사용률이 정확히 10%로 고정된다는 의미는 아닙니다.

---

## 6. CPU Throttling 발생 전 통계

### 6.1 확인 명령어

```bash
kubectl exec -n probe-resource-lab cpu-throttle-demo \
  -- cat /sys/fs/cgroup/cpu.stat
```

### 6.2 주요 측정값

CPU 부하 실습 전 다음 값을 확인했습니다.

```text
nr_periods 15
nr_throttled 8
throttled_usec 805688
```

각 항목의 의미는 다음과 같습니다.

| 항목             | 의미                               |
| ---------------- | ---------------------------------- |
| `nr_periods`     | CPU 대역폭 제어 기간의 누적 횟수   |
| `nr_throttled`   | CPU 실행이 제한된 기간의 누적 횟수 |
| `throttled_usec` | 누적 Throttling 시간 관련 통계     |

이 값들은 cgroup의 누적 통계이므로, 현재 한순간의 CPU 사용률을 나타내지 않습니다.

---

## 7. CPU 부하 발생

### 7.1 부하 발생 명령어

다음 명령어로 컨테이너 내부에서 CPU를 사용하는 반복 연산을 실행했습니다.

```bash
kubectl exec -n probe-resource-lab cpu-throttle-demo \
  -- sh -c 'timeout 5s sh -c "while :; do :; done"; rc=$?; echo "CPU 부하 종료 코드: $rc"'
```

이 명령어는 반복 연산으로 CPU 부하를 발생시키고, `timeout`을 이용해 약 5초 후 종료하도록 구성했습니다.

### 7.2 실제 실행 결과

```text
CPU 부하 종료 코드: 143
```

종료 코드 `143`은 일반적으로 SIGTERM 신호에 따른 종료를 의미합니다.

이번 결과는 CPU 부하 명령의 종료 결과이지, Pod의 주 컨테이너가 CPU Limit 때문에 종료되었다는 뜻이 아닙니다.

**CPU Throttling과 컨테이너 종료는 서로 다른 현상입니다.**

---

## 8. CPU 부하 이후 Throttling 통계

CPU 부하를 실행한 뒤 `cpu.stat`을 다시 조회했습니다.

### 8.1 실제 측정값 비교

| 지표             | 부하 전 |    부하 후 |      증가량 |
| ---------------- | ------: | ---------: | ----------: |
| `nr_periods`     |      15 |        188 |        +173 |
| `nr_throttled`   |       8 |        163 |        +155 |
| `throttled_usec` | 805,688 | 16,423,762 | +15,618,074 |

### 8.2 nr_periods 분석

```text
188 - 15 = 173
```

두 측정 시점 사이에 CPU 대역폭 제어 기간이 173회 추가로 기록되었습니다.

### 8.3 nr_throttled 분석

```text
163 - 8 = 155
```

두 측정 시점 사이에 Throttling이 발생한 기간이 155회 추가로 기록되었습니다.

따라서 CPU 부하 실습 전후에 실제 CPU Throttling이 발생했음을 확인할 수 있습니다.

### 8.4 Throttling 기간 비율

측정 구간에 새로 추가된 기간을 기준으로 계산하면 다음과 같습니다.

```text
155 / 173 × 100 ≈ 89.6%
```

측정 구간에 기록된 CPU 대역폭 제어 기간 중 약 89.6%에서 Throttling이 발생했습니다.

이 값은 **Throttling이 발생한 기간의 비율**입니다.

CPU 사용률이 89.6%였다는 의미도 아니고, 애플리케이션이 전체 시간의 89.6% 동안 멈춰 있었다는 의미도 아닙니다.

또한 두 측정 사이에는 명령어 실행 및 조회 시간이 포함되므로 이 값을 정확히 5초 동안의 통계로 해석하지 않았습니다.

### 8.5 throttled_usec 분석

```text
16423762 - 805688 = 15618074
```

`throttled_usec`도 증가했습니다.

이는 측정 구간에 CPU 실행 제한과 관련된 누적 시간이 증가했다는 추가 근거입니다.

다만 `throttled_usec`를 애플리케이션의 실제 HTTP 응답 지연 시간과 동일하게 해석할 수는 없습니다.

응답 지연을 분석하려면 별도의 요청 지연 측정과 애플리케이션 성능 지표가 필요합니다.

---

## 9. 컨테이너 재시작 여부 확인

CPU 부하 실습 직후 Pod 상태를 확인했습니다.

확인 결과:

| 항목           | 결과    |
| -------------- | ------- |
| Pod STATUS     | Running |
| Ready          | 1/1     |
| Restart Count  | 0       |
| CPU Throttling | 발생    |

이를 통해 CPU Throttling이 발생해도 컨테이너가 반드시 종료되거나 재시작되는 것은 아니라는 사실을 확인했습니다.

이후 다른 시점에 조회한 Pod에서는 Restart Count가 1로 표시되었지만, 가장 최근 종료 사유는 `Unknown`, 종료 코드는 `255`였습니다.

따라서 해당 재시작을 CPU Throttling 때문이라고 판단하지 않았습니다.

---

## 10. CPU Throttling과 OOMKilled의 차이

| 항목            | CPU Throttling                    | OOMKilled                                        |
| --------------- | --------------------------------- | ------------------------------------------------ |
| 관련 자원       | CPU                               | Memory                                           |
| 대표 원인       | CPU Limit에 따른 실행 시간 제한   | 메모리 한도 초과                                 |
| 주요 동작       | CPU 실행 일시 제한                | 프로세스 강제 종료                               |
| 컨테이너 재시작 | Throttling 자체로는 발생하지 않음 | 주 프로세스 종료 시 재시작 정책에 따라 발생 가능 |
| 조사 대상       | `cpu.max`, `cpu.stat`             | `memory.max`, `memory.events`, Pod 상태          |
| 주요 영향       | 작업 처리 속도 저하 가능          | 프로세스 또는 컨테이너 종료                      |

CPU와 Memory의 제한 방식은 서로 다릅니다.

CPU Limit은 일반적으로 실행 시간을 제어하지만, Memory Limit을 초과하면 Linux 커널이 프로세스를 종료하는 상황이 발생할 수 있습니다.

---

## 11. 운영 관점에서 배운 점

### 11.1 Running이라고 성능이 정상인 것은 아니다

CPU Throttling이 발생해도 Pod는 Running 상태를 유지할 수 있습니다.

따라서 Pod가 정상 실행 중이라는 사실만으로 성능 문제가 없다고 판단해서는 안 됩니다.

### 11.2 CPU Limit이 너무 낮으면 성능에 영향을 줄 수 있다

CPU Limit이 워크로드의 처리 요구량보다 낮게 설정되면 CPU 부하가 높은 구간에서 Throttling이 발생할 수 있습니다.

이는 애플리케이션의 처리량과 응답 지연에 영향을 줄 가능성이 있습니다.

다만 이번 실습에서는 HTTP 응답 지연을 직접 측정하지 않았으므로 실제 서비스 지연이 발생했다고 단정하지 않았습니다.

### 11.3 Requests와 Limits를 구분해야 한다

Requests는 주로 스케줄링과 자원 할당 기준에 사용됩니다.

Limits는 컨테이너의 자원 사용을 제한하는 기준입니다.

CPU Requests가 낮다고 해서 반드시 CPU Throttling이 발생하는 것은 아닙니다.

이번 실습에서는 `limits.cpu: 100m`에 의해 CPU 실행이 제한되는 현상을 확인했습니다.

### 11.4 Throttling 통계의 의미를 정확히 해석해야 한다

`nr_throttled`는 제한이 발생한 CPU 대역폭 제어 기간의 누적 횟수입니다.

`throttled_usec`는 CPU 제한과 관련된 누적 시간 통계입니다.

이 값들을 CPU 사용률 또는 애플리케이션 응답 시간으로 직접 해석하지 않아야 합니다.

### 11.5 CPU Limit은 실제 워크로드를 기준으로 설정해야 한다

실제 운영에서는 CPU 사용 패턴, 최대 부하, 처리량, 응답 지연 등의 지표를 확인해야 합니다.

CPU Limit을 지나치게 낮게 설정하면 불필요한 Throttling이 발생할 수 있습니다.

따라서 Requests와 Limits는 단순히 작은 값으로 설정하는 것이 아니라 실제 워크로드의 요구사항에 맞게 결정해야 합니다.

---

## 12. 주요 실습 명령어

| 명령어                                         | 목적                          |
| ---------------------------------------------- | ----------------------------- |
| `kubectl get pod -o yaml`                      | Requests, Limits 및 상태 확인 |
| `kubectl describe pod`                         | 컨테이너 자원 설정 확인       |
| `kubectl exec ... cat /sys/fs/cgroup/cpu.max`  | 실제 CPU Limit 설정 확인      |
| `kubectl exec ... cat /sys/fs/cgroup/cpu.stat` | Throttling 누적 통계 확인     |
| `kubectl top pod`                              | 현재 CPU·Memory 사용량 확인   |

## 13. 실습 파일

| 파일                     | 설명                                       |
| ------------------------ | ------------------------------------------ |
| `cpu-throttle-demo.yaml` | CPU Requests 50m, Limits 100m을 설정한 Pod |

## 14. 결론

이번 실습에서는 Kubernetes Pod에 CPU Limit을 설정한 뒤, Linux cgroup v2의 실제 제한값과 CPU Throttling 통계를 확인했습니다.

CPU 부하 발생 전후에 `nr_throttled`가 155회 증가한 것을 확인했으며, 해당 실습 직후 컨테이너가 재시작되지 않고 Running 상태를 유지하는 것도 검증했습니다.

이를 통해 **CPU Throttling은 컨테이너 종료가 아니라 CPU 실행 시간에 대한 제한이며, 실제 운영에서는 Pod 상태뿐 아니라 cgroup 통계와 애플리케이션 성능 지표를 함께 분석해야 한다는 점**을 학습했습니다.
