# Kubernetes Namespace·Label·Selector 실습

## 장애 사례: Pod는 Ready인데 Service 접속이 거부됐다

`selector-fault-lab`에서 nginx Pod는 `Running`, `READY 1/1`이었지만,
Service의 Selector를 변경한 뒤 같은 ClusterIP로 보낸 요청이
`Connection refused`로 실패했다.

Service의 `spec.selector.app`을 `selector-web`에서 `selector-web-typo`로
바꾸자 Pod의 실제 Label인 `app=selector-web`과 일치하는 대상이 사라졌다.
Pod IP는 `10.244.126.28`로 유지됐고, EndpointSlice에는 `endpoints: null`이 표시됐다.

| 검증 시점               | Service Selector        | EndpointSlice에서 확인한 목적지   | `k8s-cp1`에서 `10.97.212.217:80`으로 요청한 결과 |
| ----------------------- | ----------------------- | --------------------------------- | ------------------------------------------------ |
| 정상                    | `app=selector-web`      | `10.244.126.28:80`, `ready: true` | `HTTP 200 OK`                                    |
| 장애 재현               | `app=selector-web-typo` | `endpoints: null`                 | `Connection refused`                             |
| 정상 Manifest 재적용 후 | `app=selector-web`      | `10.244.126.28:80`, `ready: true` | `HTTP 200 OK`, nginx 환영 페이지                 |

복구 전후 EndpointSlice의 `targetRef.uid`를 대조해 같은 Pod가 다시 선택됐음을 확인했다.
이후 정상 여부를 판단할 때는 Pod의 준비 상태, Service에 등록된 목적지,
클라이언트의 실제 HTTP 응답을 각각 검증했다.

### 장애 재현과 진단

정상 상태에서 `selector-web-svc`의 ClusterIP로 접속해 HTTP 200을 확인한 뒤, 실습용 Service의 Selector 값 하나를 변경했다.

```bash
kubectl patch service selector-web-svc -n selector-fault-lab --type=merge -p '{"spec":{"selector":{"app":"selector-web-typo"}}}'
```

같은 호스트에서 같은 주소로 다시 요청해 변경 전후를 비교했다.

```text
$ curl --max-time 5 -i http://10.97.212.217:80
curl: (7) Failed to connect to 10.97.212.217 port 80: Connection refused
```

**가설:** Service의 새 Selector와 일치하는 Pod가 없어 요청을 전달할 목적지가 사라졌을 것이다. 이를 확인하려고 Namespace 안의 Pod 전체와 해당 Service의 EndpointSlice를 각각 조회했다.

| 조사 명령                                                                                                 | 실제 출력에서 확인한 근거                                                                               |
| --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `kubectl get pods -n selector-fault-lab -o wide --show-labels`                                            | Pod는 `READY 1/1`, `Running`, 재시작 0회였다. IP는 `10.244.126.28`, Label은 `app=selector-web`이었다.   |
| `kubectl get endpointslices -n selector-fault-lab -l kubernetes.io/service-name=selector-web-svc -o yaml` | 기존 EndpointSlice `selector-web-svc-cgs24`는 남아 있었고, `endpoints: null`, `ports: null`이 표시됐다. |
| `kubectl get events -n selector-fault-lab --sort-by=.metadata.creationTimestamp`                          | 과거 `NodeNotReady` 경고는 있었으나, 조회한 Events에 Selector 불일치 경고는 없었다.                     |

Pod 목록을 조회할 때 `-l`을 생략한 이유는 잘못된 Selector로 필터링하면 조사 대상 Pod 자체가 결과에서 빠지기 때문이다. 전체 목록에서 실제 Label을 확인하고 Service의 Selector와 대조해 한 글자라도 값이 다르면 선택되지 않는다는 것을 확인했다.

### 복구와 재발 방지

정상 Selector인 `app=selector-web`이 선언된 [Service Manifest](./manifests/selector-fault-lab-service.yaml)를 다시 적용했다. 아래 명령은 해당 파일이 있는 디렉터리에서 실행했다.

```text
$ kubectl apply -f selector-fault-lab-service.yaml
service/selector-web-svc configured
```

복구는 두 단계로 검증했다. 먼저 EndpointSlice에 `10.244.126.28:80`이 `ready: true`로 다시 등록됐고, `targetRef.uid`가 장애 재현 전과 같았다. 이어서 같은 ClusterIP로 요청해 `HTTP/1.1 200 OK`와 `Welcome to nginx!` 본문을 확인했다.

이 사례에서 배포 전 점검할 항목은 **같은 Namespace에 배포할 Service의 `spec.selector`와 Deployment의 `spec.template.metadata.labels`**다. Selector의 모든 키와 값이 Pod 템플릿에 있는지 비교한다. 배포 후에는 EndpointSlice의 목적지와 준비 상태를 확인하고, 클라이언트 위치에서 실제 HTTP 요청을 보내 응답까지 검증한다.

## 환경별 설정과 접근 권한

개발과 운영에서 설정 이름을 `app-config`로 통일하고, Namespace로 대상을 구분했다. 두 ConfigMap에는 같은 앱 Label을 붙여 `shop`의 설정을 한 번에 찾을 수 있게 했다.

| Namespace            | ConfigMap 이름 | `data.APP_MODE` | `metadata.labels`             |
| -------------------- | -------------- | --------------- | ----------------------------- |
| `namespace-lab-dev`  | `app-config`   | `development`   | `app.kubernetes.io/name=shop` |
| `namespace-lab-prod` | `app-config`   | `production`    | `app.kubernetes.io/name=shop` |

`APP_MODE` 값은 각 ConfigMap을 `-o yaml`로 조회해 확인했다. 두 환경에 같은 앱 Label을 적용한 뒤에는 아래 명령으로 두 리소스가 함께 선택되는지 검증했다.

```bash
kubectl get configmaps -A -l app.kubernetes.io/name=shop --show-labels
```

조회 결과에는 `app-config`가 두 줄로 표시됐고, `NAMESPACE` 열로 개발용과 운영용을 구분할 수 있었다. Namespace에 붙인 `environment` Label은 내부 ConfigMap에 자동으로 복사되지 않았으므로, 앱 기준 조회에 사용할 Label은 ConfigMap의 `metadata.labels`에 직접 선언했다.

### 개발 계정에 ConfigMap 읽기 권한 부여

개발용 자동화 계정이 `namespace-lab-dev`의 ConfigMap을 읽을 수 있도록 세 리소스를 구성했다.

| Manifest                                                            | 설계 이유                                                                        |
| ------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| [ServiceAccount](./manifests/namespace-lab-dev-serviceaccount.yaml) | 개발 자동화가 사용할 신원 `config-reader`를 만들었다.                            |
| [Role](./manifests/namespace-lab-dev-role.yaml)                     | 개발 Namespace의 `configmaps`에 개별 조회(`get`)와 목록 조회(`list`)를 허용했다. |
| [RoleBinding](./manifests/namespace-lab-dev-rolebinding.yaml)       | `config-reader`에 `configmap-reader` Role을 연결했다.                            |

권한 변경 전후를 비교하기 위해 같은 ServiceAccount 신원으로 다음 명령을 반복했다.

```bash
kubectl auth can-i get configmaps -n namespace-lab-dev --as=system:serviceaccount:namespace-lab-dev:config-reader
```

| 검증 항목                                   | 실제 결과                                                                   |
| ------------------------------------------- | --------------------------------------------------------------------------- |
| 개발 ConfigMap `get` 권한                   | ServiceAccount 생성 후 `no` → Role 생성 후 `no` → RoleBinding 연결 후 `yes` |
| 개발 ConfigMap 실제 조회                    | `app-config`의 `APP_MODE: development` 확인                                 |
| 개발 ConfigMap 목록 조회                    | `app-config`와 `kube-root-ca.crt` 두 개 조회                                |
| 개발 ConfigMap `patch` 권한 검사            | `no`                                                                        |
| 운영 ConfigMap `get` 권한 검사 및 실제 조회 | `no`, 실제 요청은 `Forbidden`                                               |

목록에 `kube-root-ca.crt`도 표시된 이유는 Role이 개발 Namespace의 ConfigMap 전체에 `get`, `list`를 허용했기 때문이다. 운영 조회에서는 같은 신원을 유지하고 대상 Namespace만 바꿨다. `Forbidden`은 개발 설정만 읽도록 한 권한 범위가 의도대로 적용됐다는 근거였다.

## Label 변경과 Selector 조회 결과

### 같은 조회 조건에서 ConfigMap이 사라진 이유

`namespace-lab-dev`의 `app-config`에 `environment=development` Label을 추가한 뒤, 다음 조건으로 조회했다.

```bash
kubectl get configmaps -n namespace-lab-dev -l environment=development --show-labels
```

| ConfigMap에 수행한 작업                 | 실제 조회 결과                                                                                               |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `environment=development` 추가          | 위 명령으로 `app-config` 1개 조회                                                                            |
| `--overwrite`로 값을 `staging`으로 변경 | 같은 명령은 `No resources found`를 반환했고, `-l environment=staging`으로 조회하면 `app-config` 1개가 표시됨 |
| `environment-`로 Label 키 삭제          | 이름으로 직접 조회했을 때 `app-config`가 존재했고 `LABELS`는 `<none>`이었음                                  |

Selector 조회 결과가 비어 있을 때는 먼저 **같은 Namespace에서 이름으로 조회**해 리소스의 존재와 현재 Label을 확인할 수 있다.

```bash
kubectl get configmap app-config -n namespace-lab-dev --show-labels
```

### 값 제외 조건에 Label이 없는 Pod도 포함된 사례

`config-lab`에서 Pod 8개를 대상으로 조회 조건을 바꿨다. `-l`로 대상을 선택하고 `--show-labels`로 각 Pod가 선택된 근거를 확인했다.

| `-l`에 전달한 조건                                        | 실제 결과                                             |
| --------------------------------------------------------- | ----------------------------------------------------- |
| `app=configmap-web`                                       | 웹 Pod 2개                                            |
| `app in (configmap-web,configmap-env)`                    | 웹 Pod 2개와 `configmap-env-pod`, 총 3개              |
| `app in (configmap-web,configmap-env),pod-template-hash`  | `pod-template-hash` 키가 있는 웹 Pod 2개              |
| `app in (configmap-web,configmap-env),!pod-template-hash` | 해당 키가 없는 `configmap-env-pod` 1개                |
| `app notin (configmap-web,configmap-env)`                 | `configmap-volume-pod`와 Label이 없는 Pod 4개, 총 5개 |
| `app,app notin (configmap-web,configmap-env)`             | `app=configmap-volume`인 `configmap-volume-pod` 1개   |

마지막 두 조회에서 결과가 5개에서 1개로 줄어든 이유는 맨 앞의 `app` 조건이다. `notin`만 쓰면 `app` 키가 없는 Pod도 선택된다. **앱으로 분류된 Pod 중 특정 앱을 제외하려면** 키 존재 조건을 함께 지정한다.

```bash
kubectl get pods -n config-lab -l 'app,app notin (configmap-web,configmap-env)' --show-labels
```

## Deployment·ReplicaSet·Pod의 선택 조건과 소유 관계

`config-lab`에서 `app=configmap-web`으로 조회한 웹 Pod는 2개였다. 두 Pod 모두 `pod-template-hash=ccc6dff9`를 가지고 있었다. 그중 `configmap-web-ccc6dff9-r5t5z`의 직접 관리자를 확인하기 위해 실습 당시 다음 세 객체의 YAML을 대조했다.

```bash
kubectl get deployment configmap-web -n config-lab -o yaml
kubectl get replicaset configmap-web-ccc6dff9 -n config-lab -o yaml
kubectl get pod configmap-web-ccc6dff9-r5t5z -n config-lab -o yaml
```

### 1. Pod가 Selector에 일치하는지 확인

| 확인한 필드                                          | Deployment `configmap-web` | ReplicaSet `configmap-web-ccc6dff9`               |
| ---------------------------------------------------- | -------------------------- | ------------------------------------------------- |
| `metadata.labels` — 객체 자체의 Label                | 없음                       | `app=configmap-web`, `pod-template-hash=ccc6dff9` |
| `spec.selector.matchLabels` — 선택 조건              | `app=configmap-web`        | `app=configmap-web`, `pod-template-hash=ccc6dff9` |
| `spec.template.metadata.labels` — 생성할 Pod의 Label | `app=configmap-web`        | `app=configmap-web`, `pod-template-hash=ccc6dff9` |

Deployment 자체에는 `app` Label이 없었지만 Pod 템플릿에는 있었다. 실제 Pod의 `metadata.labels`에는 ReplicaSet의 Selector가 요구한 두 값이 모두 있었다. `pod-template-hash` 덕분에 롤링 업데이트 중 같은 앱의 Pod라도 어느 템플릿에서 생성됐는지 구분할 수 있다.

### 2. Pod를 실제로 관리하는 컨트롤러 확인

각 자식 객체의 `metadata.ownerReferences`에서 `controller: true`인 항목을 찾았다. 그 항목의 `kind`와 `name`으로 부모 객체를 식별하고, 참조된 `uid`가 부모 객체 자신의 `metadata.uid`와 일치하는지 대조했다.

| 자식 객체                           | 직접 관리자로 확인한 객체           | 양쪽에서 일치한 UID                    |
| ----------------------------------- | ----------------------------------- | -------------------------------------- |
| Pod `configmap-web-ccc6dff9-r5t5z`  | ReplicaSet `configmap-web-ccc6dff9` | `8f0be244-12dd-4ae8-9b7f-c023af0e2442` |
| ReplicaSet `configmap-web-ccc6dff9` | Deployment `configmap-web`          | `d4e0ef66-913e-49de-bc79-3fcab6a8d5ea` |

이렇게 특정 Pod에서 출발해 직접 관리자인 ReplicaSet을 찾고, 다시 그 ReplicaSet을 관리하는 Deployment까지 추적했다.

## Namespace 삭제 전후 검증

삭제 범위를 확인하기 위해 [폐기용 Namespace](./manifests/namespace-delete-lab.yaml) `namespace-delete-lab`을 만들고, 그 안에 [확인용 ConfigMap](./manifests/namespace-delete-lab-configmap.yaml) `delete-marker`를 생성했다. 삭제 전에는 이 객체가 실제로 존재하는지, 삭제 후에는 전체 ConfigMap 목록에서 어떤 항목이 사라졌는지 비교했다.

### 삭제 전 확인

| 확인 대상                  | 실행한 명령                                                                      | 관찰한 결과                                                                                  |
| -------------------------- | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| 현재 컨텍스트              | `kubectl config current-context`                                                 | `kubernetes-admin@kubernetes`                                                                |
| 현재 컨텍스트의 API Server | `kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}{"\n"}'` | `https://192.168.56.10:6443` — 실습용 Control Plane 주소와 일치                              |
| 삭제할 Namespace           | `kubectl get namespace namespace-delete-lab -o wide`                             | `Active`                                                                                     |
| 내부 리소스                | `kubectl get configmaps,serviceaccounts,pods -n namespace-delete-lab`            | ConfigMap `delete-marker`, `kube-root-ca.crt`와 ServiceAccount `default`가 조회됨. Pod는 0개 |

Pod가 0개여도 Namespace 안에는 삭제 영향을 받는 설정과 계정이 있었다. 마지막 조회에서 확인한 범위는 ConfigMap·ServiceAccount·Pod 세 종류였다. 운영 환경에서 삭제 범위를 조사할 때는 `kubectl get all`의 빈 결과만으로 판단하지 않고, 해당 Namespace에 남아 있는 리소스 종류와 용도를 확인해야 한다.

### 삭제 후 확인

```text
$ kubectl delete namespace namespace-delete-lab
namespace "namespace-delete-lab" deleted

$ kubectl get namespace namespace-delete-lab
Error from server (NotFound): namespaces "namespace-delete-lab" not found
```

이어서 `kubectl get configmaps -A -o wide`로 모든 Namespace의 ConfigMap을 조회했다. 삭제 전에 확인한 `namespace-delete-lab/delete-marker`는 목록에서 사라졌고, `namespace-lab-dev/app-config`와 `namespace-lab-prod/app-config`는 각각 남아 있었다.
