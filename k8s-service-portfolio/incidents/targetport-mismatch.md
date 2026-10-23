# Incident 02. Service TargetPort 불일치 장애

## 1. 실습 목적

Kubernetes Service의 TargetPort와 실제 애플리케이션 수신 포트가 일치하지 않을 때 발생하는 통신 장애를 재현한다.

Service, Pod, EndpointSlice의 정보를 비교하여 원인을 분석하고, 설정 복구 후 실제 HTTP 통신이 정상화되는지 검증한다.

## 2. 실습 환경

- Namespace: `service-lab`
- Deployment: `nginx-pod-ip`
- 테스트 Service: `nginx-port-incident`
- Service Type: `ClusterIP`
- Application: nginx
- nginx 수신 포트: `80`
- 당시 Pod IP: `10.244.126.56`
- 테스트 Service IP: `10.107.35.80`

기존 `nginx-stable` Service에 영향을 주지 않도록 별도의 테스트 Service를 생성하여 장애를 재현했다.

## 3. 정상 상태

테스트 Service는 다음과 같이 구성했다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-port-incident
  namespace: service-lab
spec:
  type: ClusterIP
  selector:
    app: nginx-pod-ip
  ports:
    - name: http
      port: 8081
      targetPort: 80
      protocol: TCP
```

정상 상태에서 다음 정보를 확인했다.

- Service Port: `8081`
- Service TargetPort: `80`
- EndpointSlice 백엔드: `10.244.126.56:80`
- HTTP 요청 결과: `200 OK`

```bash
curl -sS -i --max-time 5 http://10.107.35.80:8081/
```

Service의 8081번 포트로 전달된 요청이 nginx Pod의 80번 포트에서 정상적으로 처리되었다.

## 4. 장애 재현

Service의 TargetPort만 80에서 8080으로 변경했다.

```bash
kubectl patch service nginx-port-incident \
  -n service-lab \
  --type=json \
  -p='[{"op":"replace","path":"/spec/ports/0/targetPort","value":8080}]'
```

변경 전:

`Service:8081 → Pod:80`

변경 후:

`Service:8081 → Pod:8080`

Service Selector와 Pod Label은 변경하지 않았다.

## 5. 장애 증상 및 증거

### 5-1. EndpointSlice 확인

```bash
kubectl get endpointslices -n service-lab \
  -l kubernetes.io/service-name=nginx-port-incident \
  -o wide
```

장애 발생 후 EndpointSlice에 다음 정보가 반영되었다.

- 백엔드 Pod IP: `10.244.126.56`
- 백엔드 Port: `8080`

Pod IP는 유지됐지만 백엔드 포트가 80에서 8080으로 변경되었다.

### 5-2. Service HTTP 요청

```bash
curl -sS -i --max-time 5 http://10.107.35.80:8081/
```

관찰 결과:

`curl: (7) Connection refused`

Service를 통한 HTTP 통신에 실패했다.

### 5-3. Pod IP 직접 통신

```bash
curl -sS -i --max-time 5 http://10.244.126.56:8080/
```

관찰 결과:

`curl: (7) Connection refused`

Service를 거치지 않고 Pod의 8080번 포트에 직접 접근해도 연결에 실패했다.

당시 nginx Pod는 Running / Ready 상태였으며, Service의 백엔드 Pod 선택 자체는 정상적으로 이루어지고 있었다.

## 6. 원인 분석

Service Selector와 Pod Label이 일치했고, EndpointSlice에도 정상적인 Pod IP가 등록되어 있었다.

따라서 백엔드 Pod를 선택하지 못하는 Selector 불일치 장애는 아니었다.

문제는 Service의 TargetPort가 실제 nginx 수신 포트와 달랐다는 것이다.

- nginx 실제 수신 포트: `80`
- 잘못 설정된 TargetPort: `8080`

Service가 Pod의 8080번 포트로 요청을 전달했지만, 해당 포트에서 요청을 수신하는 애플리케이션이 없어 TCP 연결이 거부되었다.

이를 통해 Pod의 Running / Ready 상태와 EndpointSlice의 백엔드 IP 등록만으로는 HTTP 통신 성공을 보장할 수 없다는 점을 확인했다.

## 7. 복구 작업

Service TargetPort를 기존 값인 80으로 복구했다.

```bash
kubectl patch service nginx-port-incident \
  -n service-lab \
  --type=json \
  -p='[{"op":"replace","path":"/spec/ports/0/targetPort","value":80}]'
```

복구 후 요청 경로:

`Client → Service:8081 → Pod:80 → nginx`

## 8. 복구 검증

### 8-1. EndpointSlice 검증

복구 후 EndpointSlice의 백엔드 정보가 다음과 같이 변경된 것을 확인했다.

`10.244.126.56:80`

잘못된 8080번 포트가 아닌 실제 nginx 수신 포트 80으로 복구되었다.

### 8-2. HTTP 통신 검증

```bash
curl -sS -i --max-time 5 http://10.107.35.80:8081/
```

관찰 결과:

`HTTP/1.1 200 OK`

Service를 통한 HTTP 요청이 다시 정상적으로 처리되었다.

### 8-3. 테스트 리소스 정리

실습 완료 후 테스트 Service를 삭제했다.

```bash
kubectl delete service nginx-port-incident -n service-lab
```

이후 해당 Service에 연결된 EndpointSlice 리소스가 더 이상 존재하지 않는 것을 확인했다.

기존 nginx Deployment와 `nginx-stable` Service는 유지했다.

## 9. 운영 관점에서 배운 점

1. Service Port와 TargetPort는 서로 다른 역할을 한다.
2. EndpointSlice에 백엔드 Pod IP가 존재해도 실제 HTTP 통신은 실패할 수 있다.
3. Pod가 Running / Ready 상태여도 애플리케이션의 특정 포트가 정상이라는 의미는 아니다.
4. Service TargetPort와 실제 애플리케이션 수신 포트를 비교해야 한다.
5. Pod IP 직접 접근 테스트는 Service 설정 문제와 백엔드 수신 문제를 구분하는 데 도움이 된다.
6. 설정 변경 성공만으로 복구를 확정하지 않고 EndpointSlice와 HTTP 응답을 검증해야 한다.

## 10. 실습 범위 및 한계

본 장애는 개인 Kubernetes 실습 환경에서 의도적으로 재현한 사례다.

실제 운영 환경에서 발생한 장애가 아니며, 기록된 IP 주소는 해당 실습 시점의 관찰 결과다.

이번 실습은 잘못된 TargetPort로 인한 연결 실패를 확인한 것이며, 모든 `Connection refused` 오류가 TargetPort 문제에서 발생한다고 일반화할 수는 없다.
