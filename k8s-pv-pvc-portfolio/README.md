# Kubernetes PV/PVC — 데이터 영속성과 스토리지 프로비저닝 실습

## 1. 프로젝트 개요

Kubernetes에서 Pod는 교체되거나 재생성될 수 있지만, 애플리케이션이 사용하는 데이터까지 함께 사라져서는 안 되는 경우가 있습니다.

이번 프로젝트에서는 **Pod의 생명주기와 데이터 저장 공간의 생명주기를 분리하는 방법**을 학습하고, 정적 프로비저닝과 동적 프로비저닝의 차이를 실제 Kubernetes 클러스터에서 검증했습니다.

단순한 리소스 생성에 그치지 않고, 데이터 유실과 마운트 실패를 직접 재현하고 원인을 조사하여 복구하는 과정을 진행했습니다.

### 학습 목표

- 컨테이너 파일시스템과 `emptyDir`의 데이터 생명주기 비교
- PV, PVC, StorageClass의 역할과 관계 이해
- 정적 Local PV/PVC 구성 및 데이터 영속성 검증
- `Retain`, `Released`, `claimRef`를 활용한 PV 회수 및 재사용
- `WaitForFirstConsumer`와 Node Affinity의 동작 확인
- Local Path Provisioner를 통한 동적 PV 생성
- `FailedMount` 장애 원인 분석 및 복구

## 2. 실습 환경

| 구분              | 구성                                   |
| ----------------- | -------------------------------------- |
| Kubernetes        | v1.36.3                                |
| 운영체제          | Rocky Linux 9.8                        |
| Control Plane     | k8s-cp1                                |
| Worker Node       | k8s-worker1, k8s-worker2               |
| Container Runtime | containerd                             |
| Namespace         | storage-lab                            |
| 실습 이미지       | nginx:1.28-alpine                      |
| 정적 스토리지     | Local PersistentVolume                 |
| 동적 스토리지     | Rancher Local Path Provisioner v0.0.37 |

실제 스토리지는 `k8s-worker2`의 로컬 파일시스템을 사용했습니다.

**주의:** 이번 환경은 Kubernetes 스토리지 동작을 학습하기 위한 로컬 실습 환경입니다. 노드 장애에 대한 데이터 복제와 고가용성은 제공하지 않습니다.

## 3. 프로젝트 디렉터리

| 디렉터리                     | 실습 내용                                   |
| ---------------------------- | ------------------------------------------- |
| `01-data-loss/`              | 일반 Pod의 데이터 유실 검증                 |
| `02-emptydir/`               | emptyDir의 데이터 생명주기 검증             |
| `03-static-provisioning/`    | 정적 Local PV/PVC와 WFFC                    |
| `04-reclaim-recovery/`       | Retain, Released PV 회수 및 재사용          |
| `05-dynamic-provisioning/`   | Local Path Provisioner 기반 동적 프로비저닝 |
| `storage-lab-namespace.yaml` | 실습 Namespace                              |

각 디렉터리에는 실습에 사용한 Kubernetes Manifest를 분리하여 관리했습니다.

## 4. 실습 1 — Pod 교체 시 데이터 유실

### 문제

Volume을 사용하지 않는 Pod의 컨테이너 파일시스템에 데이터를 기록한 후 Pod를 삭제하고 재생성했습니다.

### 수행 과정

1. `data-loss-demo` Deployment 생성
2. Pod 내부 `/tmp/data.txt`에 파일 기록
3. 기존 Pod 삭제
4. 새 Pod에서 동일한 파일 조회

### 결과

새로운 Pod에서 기존 파일을 찾을 수 없었습니다.

Deployment는 원하는 Pod 개수를 유지하도록 새 Pod를 생성하지만, 삭제된 컨테이너의 쓰기 가능한 파일시스템까지 보존하지는 않습니다.

**확인한 내용:** Pod의 재생성과 데이터 영속성은 별개이며, 데이터를 유지하려면 별도의 Volume 전략이 필요합니다.

## 5. 실습 2 — emptyDir의 한계 확인

`emptyDir` Volume을 Pod의 `/data` 경로에 마운트했습니다.

컨테이너 파일시스템의 `/tmp/container-data.txt`와 emptyDir의 `/data/volume-data.txt`에 각각 파일을 기록했습니다.

### 검증 결과

| 상황               | 컨테이너 파일시스템 | emptyDir    |
| ------------------ | ------------------- | ----------- |
| 컨테이너 재시작    | 데이터 소실         | 데이터 유지 |
| Pod 삭제 후 재생성 | 데이터 소실         | 데이터 소실 |

`emptyDir`은 컨테이너 재시작에는 데이터를 유지하지만 Pod 자체가 삭제되면 데이터를 보존하지 못합니다.

따라서 **Pod 교체 이후까지 데이터를 유지하려면 PV/PVC와 같은 별도 스토리지 관리 방식이 필요합니다.**

## 6. 실습 3 — 정적 Local PV/PVC 구성

### 구성

- PV: `storage-local-pv`
- PVC: `storage-local-pvc`
- Pod: `storage-local-pod`
- StorageClass: 사용하지 않음
- ReclaimPolicy: `Retain`
- AccessMode: `ReadWriteOnce`
- Volume 크기: `1Gi`
- Node Affinity: `k8s-worker2`
- 실제 저장 경로: `/srv/k8s-local-pv/storage-lab`

관리자가 Worker Node의 저장 디렉터리와 마운트 지점을 준비한 뒤 PV를 직접 생성했습니다.

PVC는 필요한 용량과 접근 방식을 선언했고, Kubernetes가 조건에 맞는 PV와 바인딩했습니다.

Pod는 PVC를 참조하여 볼륨을 `/data` 경로에 마운트했습니다.

### 데이터 영속성 검증

Pod의 `/data/data.txt`에 `persistent-data-test`를 기록했습니다.

이후 Pod를 삭제하고 다시 생성했지만, 동일한 PVC를 마운트한 새 Pod에서 기존 데이터를 확인할 수 있었습니다.

또한 `k8s-worker2`의 실제 저장 경로에서도 파일을 확인했습니다.

**확인한 내용:** PV와 PVC를 유지하고 동일한 저장 공간을 다시 사용하면 Pod의 생명주기와 무관하게 데이터를 보존할 수 있습니다.

## 7. 실습 4 — WaitForFirstConsumer 검증

정적 Local PV 환경에서 `WaitForFirstConsumer`의 동작을 확인했습니다.

StorageClass `storage-local-wffc`를 다음 정책으로 구성했습니다.

- Provisioner: `kubernetes.io/no-provisioner`
- VolumeBindingMode: `WaitForFirstConsumer`
- ReclaimPolicy: `Retain`

관리자가 PV를 미리 생성했지만, PVC를 생성한 직후에는 `Pending` 상태였습니다.

`kubectl describe pvc`에서 다음 이벤트를 확인했습니다.

`WaitForFirstConsumer — waiting for first consumer to be created before binding`

이후 PVC를 사용하는 Pod를 생성하자 PV/PVC가 `Bound` 상태로 변경되었고, Pod는 PV의 Node Affinity 조건에 맞는 `k8s-worker2`에 배치되었습니다.

Pod의 `/data/wffc.txt`에 `wffc-storage-test`를 기록한 뒤 Worker Node에서도 동일한 값을 확인했습니다.

**확인한 내용:** `WaitForFirstConsumer`는 동적 프로비저닝 전용 기능이 아닙니다. 정적 PV에서도 Pod의 스케줄링 조건을 고려하여 바인딩을 지연할 수 있습니다.

## 8. 실습 5 — Retain 정책과 Released PV 회수

### 문제 상황

PVC를 삭제한 뒤에도 `Retain` 정책에 의해 PV와 실제 데이터가 남아 있었습니다.

그러나 PV는 `Released` 상태가 되었고, 기존 PVC의 `claimRef` 정보가 남아 있어 새로운 PVC가 즉시 사용할 수 없었습니다.

### 해결 과정

1. 실제 저장 디렉터리의 데이터 백업
2. 기존 데이터 보존 상태 확인
3. Released PV 객체 삭제
4. 동일한 저장 경로를 참조하는 새 PV 생성
5. 새로운 PVC 생성 및 바인딩
6. 새 Pod에서 기존 데이터 조회

### 결과

새 PV `storage-local-pv-reused`와 새 PVC `storage-local-pvc-reused`를 연결한 후, 기존 `persistent-data-test` 데이터를 다시 읽을 수 있었습니다.

**확인한 내용:** PV 객체와 실제 저장 데이터는 구분해야 하며, `Retain` 정책으로 보존된 데이터를 재사용하려면 적절한 수동 회수 작업이 필요합니다.

## 9. 트러블슈팅 — Local PV 경로 오류로 인한 FailedMount

### 장애 현상

Released PV를 회수하여 재사용하는 과정에서 새 Pod가 정상적으로 실행되지 못하고 `ContainerCreating` 상태에 머물렀습니다.

### 원인 분석

`kubectl describe pod`로 이벤트를 확인했고, `FailedMount`가 발생한 것을 발견했습니다.

PV의 저장 경로가 실제 Worker Node의 디렉터리와 일치하지 않았습니다.

| 구분        | 경로                            |
| ----------- | ------------------------------- |
| 잘못된 경로 | `/src/k8s-local-pv/storage-lab` |
| 올바른 경로 | `/srv/k8s-local-pv/storage-lab` |

### 복구

기존 리소스를 정리한 후 PV 경로를 올바르게 수정하고 PV/PVC/Pod를 재생성했습니다.

그 결과 Pod가 `Running` 상태로 전환되었고, 이전에 보존한 데이터도 정상적으로 읽을 수 있었습니다.

### 배운 점

- Pod가 생성되었다고 볼륨 마운트까지 성공한 것은 아닙니다.
- `ContainerCreating`이 지속되면 Pod Events를 확인해야 합니다.
- Local PV의 경로는 실제 Worker Node의 파일시스템 및 마운트 상태와 일치해야 합니다.
- 복구 이후에는 Pod 상태뿐 아니라 실제 데이터 접근까지 검증해야 합니다.

## 10. 실습 6 — 동적 프로비저닝

### 목적

정적 프로비저닝에서는 관리자가 PV와 실제 저장 경로를 직접 준비해야 합니다.

이러한 반복 작업을 줄이기 위해 Rancher Local Path Provisioner를 설치하고, PVC 요청에 따라 PV가 자동 생성되는지 검증했습니다.

### 구성

- Provisioner: `rancher.io/local-path`
- StorageClass: `local-path`
- ReclaimPolicy: `Retain`
- VolumeBindingMode: `WaitForFirstConsumer`
- PVC: `storage-dynamic-pvc`
- Pod: `storage-dynamic-pod`
- 저장 노드: `k8s-worker2`
- 기본 저장 경로: `/opt/local-path-provisioner`

공식 Local Path Provisioner v0.0.37 배포 Manifest를 사용했으며, 실습 중 StorageClass의 ReclaimPolicy를 `Delete`에서 `Retain`으로 변경했습니다.

`05-dynamic-provisioning/storage-local-path-sc.yaml`은 이 StorageClass 설정을 별도로 문서화한 파일입니다. 실제 프로비저너 설치에는 Deployment, RBAC, ConfigMap 등의 리소스가 추가로 필요합니다.

### 검증 과정

1. Provisioner Deployment 실행 및 Ready 상태 확인
2. `local-path` StorageClass 확인
3. PV를 생성하지 않고 PVC만 생성
4. WFFC로 인한 PVC `Pending` 확인
5. PVC를 사용하는 Pod 생성
6. Provisioner가 새로운 PV를 자동 생성
7. PV/PVC `Bound` 확인
8. Pod에서 데이터 기록 및 Worker Node 파일 확인
9. Pod 삭제 및 재생성 후 데이터 유지 검증

### 실제 결과

자동으로 생성된 PV는 다음과 같았습니다.

`pvc-5f65150e-3833-48bf-8edd-1184493bce18`

PV의 스토리지 유형은 `hostPath`였으며, Node Affinity는 `k8s-worker2`로 설정되었습니다.

Pod에서 `/data/dynamic.txt`에 `dynamic-provisioning-test`를 기록했고, Worker Node의 실제 저장 디렉터리에서도 동일한 파일을 확인했습니다.

이후 Pod를 삭제하고 재생성했을 때 Pod IP는 변경되었지만 기존 데이터는 유지되었습니다.

**확인한 내용:** 동적 프로비저닝은 PV 생성 및 스토리지 준비 작업을 자동화하지만, 노드 장애에 대한 데이터 복제나 고가용성까지 제공하는 것은 아닙니다.

## 11. 운영 관점에서 확인한 한계

이번 실습에서 사용한 Local PV와 Local Path Provisioner는 모두 특정 Worker Node의 저장 공간에 의존합니다.

따라서 다음 사항을 고려해야 합니다.

- 특정 노드의 장애가 스토리지 가용성에 영향을 줄 수 있습니다.
- `Retain`은 데이터 자동 삭제를 방지하지만 백업 기능은 아닙니다.
- `1Gi`로 선언한 볼륨 용량이 실제 디스크 사용량을 강제로 제한한다는 의미는 아닙니다.
- 동적 프로비저닝을 사용하더라도 스토리지의 권한, 삭제 정책, 장애 복구 방식은 별도로 설계해야 합니다.
- Local Path Provisioner의 기본 설정에는 넓은 디렉터리 권한과 볼륨 삭제용 스크립트가 포함되므로 운영 환경에 적용하기 전 검토가 필요합니다.

본 프로젝트의 YAML은 기존 실습 환경을 기준으로 작성되었으므로, 재적용 시 기존 리소스와의 이름 충돌 및 로컬 마운트 지점의 준비 상태를 확인해야 합니다.

특히 `03-static-provisioning`과 `04-reclaim-recovery`는 동일한 저장 경로를 사용하는 **서로 다른 실습 단계**입니다. 모든 Manifest를 동시에 적용하는 구성이 아닙니다.

## 12. 결론

이번 프로젝트를 통해 Kubernetes에서 데이터 영속성을 구현하는 방법을 단계적으로 확인했습니다.

처음에는 Pod 삭제로 데이터가 유실되는 문제를 재현했고, `emptyDir`의 한계를 확인했습니다.

이후 정적 PV/PVC를 사용해 데이터를 보존하고, `Retain` 정책과 `Released` 상태를 직접 다루면서 스토리지의 생명주기를 이해했습니다.

또한 `FailedMount` 장애를 분석하고 복구했으며, 마지막으로 Local Path Provisioner를 도입해 반복적인 PV 생성 작업을 자동화했습니다.

**가장 중요한 학습 결과는 단순히 PV/PVC를 생성하는 것이 아니라, 데이터가 실제 어디에 저장되는지 이해하고 문제가 발생했을 때 원인을 확인하여 복구할 수 있게 되었다는 점입니다.**

다음 학습에서는 애플리케이션의 안정적인 운영을 위해 Probe, Resource Request/Limit 등과 연결하여 Kubernetes 리소스의 상태와 운영 제약을 더욱 깊이 학습할 계획입니다.
