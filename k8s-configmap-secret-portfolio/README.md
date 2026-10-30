# Kubernetes ConfigMap & Secret — 설정 관리 및 장애 대응

## 1. 프로젝트 개요

Kubernetes에서 애플리케이션 설정과 민감정보를 관리하는 ConfigMap과 Secret의 동작 방식을 이해하고, 실제 설정 오류를 재현하여 장애 원인을 분석하고 복구한 실습 프로젝트입니다.

단순히 리소스를 생성하는 것에서 나아가, **설정 변경이 실행 중인 컨테이너에 어떻게 반영되는지 확인하고 운영 관점에서 장애 대응 과정을 기록하는 것**을 목표로 했습니다.

### 학습 목표

- ConfigMap과 Secret의 필요성과 차이 이해
- 환경변수 및 Volume을 활용한 설정 전달
- 설정 변경 시 환경변수와 Volume의 갱신 방식 비교
- `CreateContainerConfigError` 재현 및 원인 분석
- Deployment 롤링 재시작 및 복구 검증
- Secret 보안과 RBAC의 기본 원칙 이해

## 2. 실습 환경

| 항목              | 환경                     |
| ----------------- | ------------------------ |
| OS                | Rocky Linux 9.8          |
| Kubernetes        | v1.36.3                  |
| Container Runtime | containerd 2.3.3         |
| CNI               | Calico                   |
| Control Plane     | k8s-cp1                  |
| Worker Node       | k8s-worker1, k8s-worker2 |
| Namespace         | config-lab               |
| Container Image   | nginx:1.27               |

Kubernetes 리소스의 생성과 검증은 Rocky Linux VM에서 진행했으며, Manifest와 문서는 Windows VS Code에서 작성하여 GitHub로 관리했습니다.

## 3. 프로젝트 파일 구조

```text
k8s-configmap-secret-portfolio/
├── namespace.yaml
├── web-config.yaml
├── configmap-env-pod.yaml
├── configmap-volume-pod.yaml
├── configmap-deployment.yaml
├── secret-example.yaml
├── secret-env-pod.yaml
├── secret-volume-pod.yaml
├── troubleshooting/
│   ├── configmap-key-error-pod.yaml
│   └── secret-key-error-pod.yaml
└── README.md
```

`troubleshooting/` 폴더의 Manifest는 의도적으로 잘못된 Key를 참조하는 장애 재현용 파일이며, 정상 리소스와 함께 일괄 적용하지 않습니다.

`secret-example.yaml`은 가짜 인증정보만 사용하는 학습용 예제입니다. 기존 클러스터에 동일한 이름의 Secret이 존재할 경우 적용 시 데이터가 변경될 수 있으므로 별도 실습 환경에서 사용해야 합니다.

## 4. ConfigMap과 Secret의 역할

### ConfigMap

비밀정보가 아닌 애플리케이션 설정을 Kubernetes 리소스로 관리합니다.

실습에서는 다음 데이터를 사용했습니다.

| Key       | Value            |
| --------- | ---------------- |
| LOG_LEVEL | INFO             |
| APP_MODE  | production       |
| DB_HOST   | prod-db.internal |

ConfigMap을 통해 컨테이너 이미지와 설정을 분리하여 관리하는 방법을 학습했습니다.

### Secret

사용자 이름, 비밀번호, API 토큰과 같은 민감정보를 Kubernetes 리소스로 관리합니다.

실습에서는 `db-secret`을 생성하고 `DB_USERNAME`, `DB_PASSWORD`를 환경변수와 Volume으로 전달했습니다.

Secret의 Base64는 암호화가 아닌 인코딩이므로, 실제 운영에서는 RBAC 최소 권한, 저장 시 암호화, 비밀정보 노출 방지 등의 추가 보안 조치가 필요합니다.

## 5. 환경변수와 Volume 비교

ConfigMap과 Secret은 환경변수 또는 Volume 형태로 컨테이너에 전달할 수 있습니다.

| 항목             | 환경변수                      | Volume                                       |
| ---------------- | ----------------------------- | -------------------------------------------- |
| 데이터 전달      | 컨테이너 환경변수             | 컨테이너 내부 파일                           |
| 원본 데이터 변경 | 기존 환경변수는 변경되지 않음 | 일반적인 마운트에서는 일정 시간 후 파일 갱신 |
| 최신 값 사용     | 컨테이너 재생성 필요          | 애플리케이션의 파일 재읽기 필요              |
| 주의사항         | 민감정보 노출 가능성          | subPath 마운트 등 자동 갱신 예외 존재        |

### 실제 ConfigMap 변경 실험

`web-config`의 `LOG_LEVEL`을 `INFO`에서 `DEBUG`로 변경했습니다.

변경 전:

```text
ENV: INFO
FILE: INFO
```

ConfigMap 변경 후 기존 컨테이너:

```text
ENV: INFO
FILE: DEBUG
```

환경변수는 기존 값을 유지했지만, Volume 파일은 변경된 값을 반영했습니다.

이후 Deployment를 롤링 재시작했습니다.

```bash
kubectl rollout restart deployment/configmap-web -n config-lab
kubectl rollout status deployment/configmap-web -n config-lab
```

새로 생성된 Pod에서는 다음 결과를 확인했습니다.

```text
ENV: DEBUG
FILE: DEBUG
```

실습 종료 후 ConfigMap을 다시 `INFO`로 변경하고 Deployment를 롤링 재시작하여, Replica 2개 모두에서 `ENV: INFO`, `FILE: INFO`를 확인했습니다.

**결론:** ConfigMap 원본 데이터의 변경과 애플리케이션에 실제 적용되는 시점은 구분해야 합니다.

참고로 이번 실습은 nginx에서 환경변수와 파일이 전달되는 방식을 검증한 것입니다. nginx 자체가 `LOG_LEVEL` 환경변수를 읽어 로그 수준을 변경하도록 설정한 것은 아닙니다.

## 6. 장애 사례 ① ConfigMap Key 참조 오류

### 증상

존재하지 않는 ConfigMap Key인 `LOG_LEVEL_TYPO`를 참조하는 Pod를 생성했습니다.

```yaml
configMapKeyRef:
  name: web-config
  key: LOG_LEVEL_TYPO
```

Server Dry Run은 성공했지만 실제 Pod 생성 후 다음 상태가 발생했습니다.

```text
READY: 0/1
STATUS: CreateContainerConfigError
```

### 원인 분석

다음 명령어로 Pod의 Events를 확인했습니다.

```bash
kubectl describe pod configmap-key-error-pod -n config-lab
```

실제 오류:

```text
Error: couldn't find key LOG_LEVEL_TYPO
in ConfigMap config-lab/web-config
```

ConfigMap을 조회한 결과 실제 존재하는 Key는 `LOG_LEVEL`이었습니다.

### 복구 및 검증

Manifest의 `LOG_LEVEL_TYPO`를 `LOG_LEVEL`로 수정했습니다.

독립 Pod의 환경변수 참조 필드는 생성 후 직접 변경할 수 없으므로 기존 Pod를 삭제하고 수정한 Manifest로 재생성했습니다.

이후 Pod의 `Running 1/1` 상태와 환경변수 `LOG_LEVEL=INFO`를 확인했습니다.

## 7. 장애 사례 ② Secret Key 참조 오류

### 증상

Secret에 존재하지 않는 `DB_PASSWORD_TYPO`를 참조하도록 설정했습니다.

```yaml
secretKeyRef:
  name: db-secret
  key: DB_PASSWORD_TYPO
```

Pod의 상태는 다음과 같았습니다.

```text
READY: 0/1
STATUS: CreateContainerConfigError
```

### 원인 분석

```bash
kubectl describe pod secret-key-error-pod -n config-lab
```

Events에서 다음 오류를 확인했습니다.

```text
Error: couldn't find key DB_PASSWORD_TYPO
in Secret config-lab/db-secret
```

실제 Secret에는 `DB_PASSWORD` Key가 존재했지만 `DB_PASSWORD_TYPO`는 존재하지 않았습니다.

### 복구 및 검증

Manifest의 Key를 `DB_PASSWORD`로 수정한 뒤 Pod를 재생성했습니다.

복구 결과:

```text
READY: 1/1
STATUS: Running
RESTARTS: 0
```

비밀번호 원문을 출력하지 않고 컨테이너 내부 환경변수 존재 여부를 확인했습니다.

```bash
kubectl exec -n config-lab secret-key-error-pod -c nginx -- \
  sh -c 'test -n "$DB_PASSWORD" && echo "DB_PASSWORD is set"'
```

결과:

```text
DB_PASSWORD is set
```

## 8. 장애 사례 ③ 여러 Pod의 재시작 원인 조사

여러 Pod가 동일한 Worker Node에서 비슷한 시각에 재시작한 기록을 발견했습니다.

두 Pod에서 확인한 공통 기록:

```text
Last State: Terminated
Reason: Unknown
Exit Code: 255
Finished At: 2026-10-30 14:12:19
```

공통 Node인 `k8s-worker2`를 조사했습니다.

```bash
kubectl describe node k8s-worker2
```

Linux의 부팅 기록도 확인했습니다.

```bash
uptime -s
last -x reboot shutdown -n 5
journalctl --list-boots
```

`k8s-worker2`는 2026년 10월 30일 14:11:52에 부팅된 것으로 확인되었습니다.

이 기록을 바탕으로 Node 재부팅이 여러 컨테이너의 재시작에 영향을 미쳤을 가능성이 높다고 판단했습니다.

다만 이전 부팅의 Journal이 조회되지 않아 **Node 재부팅 자체의 근본 원인은 확정하지 못했습니다.**

또한 `/var/log/journal`이 존재하지 않고 `/run/log/journal`이 존재하는 것을 확인했습니다. 추후 장애 조사에 필요한 로그를 보존할 수 있도록 Journal 저장 및 보존 정책을 점검할 필요가 있음을 학습했습니다.

## 9. 장애 대응에서 배운 점

### 증상과 원인을 구분해야 한다

`CreateContainerConfigError`는 장애 상태를 나타내지만 그 자체로 정확한 원인을 알려주지는 않습니다.

`kubectl describe pod`의 Events와 실제 ConfigMap·Secret 데이터를 비교해 원인을 확인해야 합니다.

### Server Dry Run만으로 실행 성공을 보장할 수 없다

Server Dry Run은 API Server의 요청 검증을 수행하지만, 실제 컨테이너 시작에 필요한 모든 참조 데이터의 존재와 실행 결과까지 보장하지는 않습니다.

### 설정 변경 후 실제 동작을 검증해야 한다

Deployment 롤링 업데이트가 성공하더라도 각 Pod가 올바른 설정을 사용하고 있는지 확인해야 합니다.

### 장애 복구와 근본 원인 분석은 별개다

Pod가 다시 Running 상태가 되었다고 해서 이전 장애의 근본 원인이 모두 밝혀진 것은 아닙니다.

확인된 사실, 유력한 가설, 확인하지 못한 사항을 구분해서 기록해야 합니다.

## 10. 보안 및 사용 시 주의사항

- 이 저장소에는 실제 비밀번호와 API 토큰을 포함하지 않습니다.
- `secret-example.yaml`은 가짜 데이터로 작성된 교육용 파일입니다.
- Secret의 Base64 데이터는 암호화된 비밀번호가 아닙니다.
- RBAC 최소 권한 원칙을 적용해야 합니다.
- `troubleshooting/`의 Manifest는 장애 재현용이므로 정상 배포 대상으로 사용하지 않습니다.
- 실제 운영 환경에서는 ConfigMap·Secret 변경이 애플리케이션에 미치는 영향과 롤링 업데이트 전략을 함께 고려해야 합니다.

## 11. 프로젝트를 통해 얻은 역량

이번 프로젝트를 통해 Kubernetes에서 설정값을 분리하여 관리하는 방법을 익혔습니다.

특히 ConfigMap과 Secret의 참조 오류로 인한 컨테이너 시작 실패를 직접 재현하고, Events를 이용해 원인을 확인하고 복구했습니다.

또한 ConfigMap 변경 시 환경변수와 Volume 파일의 동작 차이를 검증하고, Deployment의 롤링 재시작으로 변경된 설정을 전체 Replica에 반영하는 과정을 경험했습니다.

이를 통해 **리소스를 생성하는 것뿐만 아니라 설정 변경의 영향, 장애 원인 분석, 복구 후 검증까지 고려하는 운영 관점**을 학습했습니다.
