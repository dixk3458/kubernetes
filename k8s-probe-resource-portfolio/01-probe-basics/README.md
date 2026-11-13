# Kubernetes Probe 기본 실습 — HTTP / TCP / Exec

## 1. 실습 개요

Kubernetes에서 컨테이너의 준비 상태를 확인하는 세 가지 Probe 검사 방식인 **HTTP GET, TCP Socket, Exec**를 직접 구성하고 비교한 실습입니다.

동일한 nginx 이미지를 사용하되 검사 방식만 다르게 설정하여, Probe가 무엇을 검사하고 어떤 한계를 갖는지 확인했습니다.

### 실습 목표

- HTTP, TCP, Exec Probe의 동작 방식 이해
- `readinessProbe`를 이용한 Pod 준비 상태 검사
- Probe 검사 주기와 실패 기준 확인
- Pod의 Running과 Ready 상태 구분
- 실제 Kubernetes 리소스의 설정과 상태 검증

## 2. 실습 환경

| 항목            | 설정                         |
| --------------- | ---------------------------- |
| OS              | Rocky Linux 9.8              |
| Kubernetes      | v1.36.3                      |
| Namespace       | `probe-resource-lab`         |
| Container Image | `nginx:1.28-alpine`          |
| Probe 종류      | Readiness Probe              |
| Probe 방식      | HTTP GET / TCP Socket / Exec |

세 가지 실습 모두 동일한 Namespace에서 독립적인 Pod로 진행했습니다.

## 3. HTTP Readiness Probe

**Manifest:** `probe-http-demo.yaml`

HTTP GET 요청으로 nginx의 기본 웹 페이지에 접근할 수 있는지 확인했습니다.

주요 설정:

- 검사 경로: `/`
- 포트: `80`
- 최초 검사 지연: `3초`
- 검사 주기: `5초`
- 타임아웃: `2초`
- 실패 임계 횟수: `3회`

HTTP Probe는 지정된 엔드포인트의 응답 상태를 확인합니다. HTTP 상태 코드가 200 이상 400 미만이면 성공으로 판단합니다.

실습에서는 nginx 기본 페이지를 검사 대상으로 사용했습니다.

### 확인 명령어

`kubectl get pod probe-http-demo -n probe-resource-lab`

`kubectl describe pod probe-http-demo -n probe-resource-lab`

### 확인 결과

실제 Pod 조회 결과 `Ready: True`와 `ContainersReady: True`를 확인했습니다.

이를 통해 설정된 HTTP Readiness 검사 조건을 충족하고 있음을 확인했습니다.

## 4. TCP Readiness Probe

**Manifest:** `probe-tcp-demo.yaml`

nginx의 80번 포트에 TCP 연결을 수립할 수 있는지 검사했습니다.

주요 설정:

- 검사 포트: `80`
- 최초 검사 지연: `3초`
- 검사 주기: `5초`
- 타임아웃: `2초`
- 실패 임계 횟수: `3회`

TCP Probe는 지정된 포트에 연결할 수 있는지 확인합니다.

HTTP 요청의 응답 본문이나 애플리케이션의 기능적인 정상 여부까지 검사하지는 않습니다.

### 확인 명령어

`kubectl get pod probe-tcp-demo -n probe-resource-lab`

`kubectl describe pod probe-tcp-demo -n probe-resource-lab`

### 확인 결과

실제 Pod 조회 결과 `Ready: True`와 `ContainersReady: True`를 확인했습니다.

이를 통해 80번 포트에 대한 TCP Readiness 검사 조건을 충족하고 있음을 확인했습니다.

## 5. Exec Readiness Probe

**Manifest:** `probe-exec-demo.yaml`

컨테이너 내부에서 명령어를 실행하여 nginx 기본 페이지 파일이 존재하는지 검사했습니다.

검사 명령어:

`test -f /usr/share/nginx/html/index.html`

주요 설정:

- 실행 환경: `/bin/sh -c`
- 검사 대상: `/usr/share/nginx/html/index.html`
- 최초 검사 지연: `3초`
- 검사 주기: `5초`
- 타임아웃: `2초`
- 실패 임계 횟수: `3회`

Exec Probe는 컨테이너 내부 명령어의 종료 코드로 검사 결과를 판단합니다.

- 종료 코드 `0`: 성공
- 종료 코드 `0` 이외: 실패

### 확인 명령어

`kubectl get pod probe-exec-demo -n probe-resource-lab`

`kubectl describe pod probe-exec-demo -n probe-resource-lab`

### 확인 결과

실제 Pod 조회 결과 `Ready: True`와 `ContainersReady: True`를 확인했습니다.

이를 통해 `index.html` 파일 존재 여부를 검사하는 조건을 충족하고 있음을 확인했습니다.

단, 파일이 존재한다는 사실만으로 실제 HTTP 요청 처리가 정상이라는 것을 보장하지는 않습니다.

## 6. 세 가지 Probe 비교

| 비교 항목           | HTTP                    | TCP                        | Exec                      |
| ------------------- | ----------------------- | -------------------------- | ------------------------- |
| 검사 대상           | HTTP 엔드포인트         | TCP 포트                   | 내부 명령어               |
| 성공 기준           | HTTP 200~399            | TCP 연결 성공              | 종료 코드 0               |
| 검사 위치           | kubelet의 HTTP 요청     | kubelet의 TCP 연결         | 컨테이너 내부             |
| HTTP 응답 상태 확인 | 가능                    | 불가능                     | 명령어 구현에 따라 가능   |
| 주요 한계           | 검사 경로의 품질에 의존 | 내부 기능 이상 감지에 한계 | 검사 명령어의 품질에 의존 |

어떤 방식이 항상 더 좋은 것은 아닙니다.

애플리케이션의 특성과 무엇을 준비 상태로 판단해야 하는지에 따라 적절한 검사 방식을 선택해야 합니다.

## 7. Probe 설정값의 의미

| 설정                  | 의미                                                    |
| --------------------- | ------------------------------------------------------- |
| `initialDelaySeconds` | 컨테이너 시작 후 최초 검사 전 대기 시간                 |
| `periodSeconds`       | Probe 검사 주기                                         |
| `timeoutSeconds`      | 한 번의 검사에 허용되는 시간                            |
| `successThreshold`    | 실패 후 성공 상태로 판단하기 위해 필요한 연속 성공 횟수 |
| `failureThreshold`    | 실패 상태로 판단하기 위해 필요한 연속 실패 횟수         |

Probe 설정은 애플리케이션의 실제 시작 시간과 응답 특성을 고려해야 합니다.

검사 간격이 너무 짧거나 타임아웃이 지나치게 작으면 일시적인 지연에도 불필요하게 NotReady 상태가 될 수 있습니다.

반대로 검사 간격이 지나치게 길면 장애 감지가 늦어질 수 있습니다.

## 8. 운영 관점에서 배운 점

### 8.1 Running과 Ready는 다르다

Pod가 Running 상태라고 해서 애플리케이션이 트래픽을 처리할 준비가 되었다고 단정할 수 없습니다.

따라서 `STATUS`와 함께 `READY`, Conditions, Events를 확인해야 합니다.

### 8.2 Readiness Probe 실패는 컨테이너를 재시작시키지 않는다

이번 세 가지 실습은 모두 Readiness Probe를 사용했습니다.

Readiness 검사에 실패하면 컨테이너의 준비 상태가 False가 되며, 일반적인 Service 트래픽 대상에서 제외됩니다.

컨테이너 재시작 여부를 판단하는 Liveness Probe와 목적이 다릅니다.

### 8.3 검사 성공이 전체 서비스 정상 상태를 보장하지 않는다

HTTP 기본 페이지 응답, TCP 포트 연결, 특정 파일의 존재 여부는 서로 다른 조건을 검사합니다.

실제 운영에서는 애플리케이션이 요청을 받을 준비가 되었는지를 정확히 반영하는 검사 조건을 구성해야 합니다.

### 8.4 재시작 횟수만으로 원인을 단정하지 않는다

조회 당시 각 Pod의 Restart Count는 5였으며, 이전 컨테이너 종료 사유는 `Unknown`으로 표시되었습니다.

이 정보만으로 재시작의 정확한 원인이나 Readiness Probe와의 인과관계를 판단할 수 없습니다.

원인 분석을 위해서는 관련 Events, 컨테이너 로그, Node 및 kubelet 상태 등을 추가 확인해야 합니다.

## 9. 실습 파일

| 파일                   | 내용                       |
| ---------------------- | -------------------------- |
| `probe-http-demo.yaml` | HTTP GET Readiness Probe   |
| `probe-tcp-demo.yaml`  | TCP Socket Readiness Probe |
| `probe-exec-demo.yaml` | Exec Readiness Probe       |

## 10. 결론

이번 실습을 통해 HTTP, TCP, Exec Probe의 검사 대상과 성공 조건이 서로 다르다는 것을 확인했습니다.

또한 Pod의 실행 상태와 트래픽 수신 준비 상태를 구분하고, Readiness Probe가 Service의 트래픽 대상 결정에 어떤 역할을 하는지 학습했습니다.

핵심은 **Probe를 설정하는 것 자체보다, 검사 조건이 실제 애플리케이션의 상태를 올바르게 반영하도록 설계하는 것**입니다.
