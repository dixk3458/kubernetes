# Incident 01. Service Selector 불일치 장애

## 1. 실습 목적

Kubernetes Service의 Selector와 Pod Label이 일치하지 않을 때 발생하는 통신 장애를 재현하고, EndpointSlice를 통해 원인을 분석한 뒤 복구하는 과정을 기록한다.

## 2. 실습 환경

- Namespace: `service-lab`
- Deployment: `nginx-pod-ip`
- 테스트 Service: `nginx-selector-incident`
- Service Type: `ClusterIP`
- Application: nginx
- Pod Label: `app=nginx-pod-ip`

기존 정상 Service에 영향을 주지 않도록 별도의 테스트 Service를 사용했다.

## 3. 정상 상태

Service Selector:

`app=nginx-pod-ip`

Pod Label:

`app=nginx-pod-ip`

Service와 Pod의 Label 조건이 일치하여 EndpointSlice에 백엔드 Pod가 등록되었다.

- 테스트 Service IP: `10.96.54.196`
- 당시 백엔드 Pod IP: `10.244.126.56`
- 백엔드 Port: `80`
- HTTP 요청 결과: `200 OK`

## 4. 장애 재현

Service의 Selector를 의도적으로 변경했다.

```bash
kubectl patch service nginx-selector-incident \
  -n service-lab \
  --type=merge \
  -p '{"spec":{"selector":{"app":"nginx-wrong"}}}'
```

변경 후:

- Service Selector: `app=nginx-wrong`
- Pod Label: `app=nginx-pod-ip`

두 값이 불일치하는 상황을 만들었다.

## 5. 장애 증상 및 증거

Service Selector가 변경된 후 다음 현상을 확인했다.

- nginx Pod는 계속 Running / Ready 상태였다.
- EndpointSlice 리소스는 남아 있었지만 백엔드 정보는 비어 있었다.
- EndpointSlice 조회 시 PORTS와 ENDPOINTS가 `<unset>`으로 표시되었다.
- Service ClusterIP로 HTTP 요청 시 `Connection refused`가 발생했다.

### 확인 명령어

```bash
kubectl describe svc nginx-selector-incident -n service-lab

kubectl get pods -n service-lab --show-labels

kubectl get endpointslices -n service-lab \
  -l kubernetes.io/service-name=nginx-selector-incident \
  -o wide

curl -sS -i --max-time 5 http://10.96.54.196:80/
```

## 6. 원인 분석

Service의 Selector와 Pod Label이 일치하지 않아 nginx Pod가 백엔드로 선택되지 않았다.

그 결과 EndpointSlice에서 사용 가능한 백엔드 주소가 사라졌고, Service를 통한 HTTP 요청이 실패했다.

이 장애는 Pod가 실행되지 않는 문제가 아니라 Service의 백엔드 선택 조건이 잘못 설정된 문제였다.

## 7. 복구 작업

Service Selector를 기존 값으로 복구했다.

```bash
kubectl patch service nginx-selector-incident \
  -n service-lab \
  --type=merge \
  -p '{"spec":{"selector":{"app":"nginx-pod-ip"}}}'
```

## 8. 복구 검증

복구 후 다음 결과를 확인했다.

- Service Selector와 Pod Label이 다시 일치했다.
- EndpointSlice에 `10.244.126.56:80` 백엔드가 등록되었다.
- Service ClusterIP로 HTTP 요청 시 `200 OK`가 반환되었다.

실습 완료 후 테스트 Service를 삭제했다.

## 9. 운영 관점에서 배운 점

1. Pod가 Running / Ready 상태여도 Service 통신이 실패할 수 있다.
2. Service Selector와 Pod Label의 일치 여부를 확인해야 한다.
3. EndpointSlice를 확인하면 Service가 실제로 선택한 백엔드를 파악할 수 있다.
4. 설정 복구 후에는 실제 HTTP 요청까지 검증해야 한다.

## 10. 실습 범위

본 장애는 개인 Kubernetes 실습 환경에서 의도적으로 재현한 사례다.

실제 운영 서비스에서 발생한 장애가 아니며, 기록된 IP 주소는 해당 실습 시점의 관찰 결과다.
