# Kubernetes Service 운영 실습 및 장애 분석

## 1. 프로젝트 개요

Kubernetes 환경에서 Pod IP 변경으로 발생하는 접근 문제를 이해하고, Service를 통한 안정적인 네트워크 접근 방식을 실습한 프로젝트입니다.

단순한 리소스 생성에 그치지 않고, Service 설정 오류를 직접 재현하고 원인 분석부터 복구 검증까지 수행했습니다.

## 2. 실습 환경

- OS: Rocky Linux 9.8
- Kubernetes: v1.36.3
- Container Runtime: containerd
- CNI: Calico
- Cluster: Control Plane 1대, Worker Node 2대
- Namespace: service-lab
- Application: nginx

## 3. 주요 실습

- Pod IP 변경과 Service의 필요성
- ClusterIP 및 NodePort 구성
- Service Selector와 Pod Label 관계
- Kubernetes DNS를 이용한 Service 접근
- EndpointSlice를 통한 백엔드 확인
- Service 장애 재현 및 트러블 슈팅

## 4. 장애 분석 사례

### Case 01. Selector 불일치

Service Selector와 Pod Label을 불일치시켜 백엔드가 선택되지 않는 장애를 재현하고 복구했습니다.

### Case 02. TargetPort 불일치

Service TargetPort를 실제 nginx 수신 포트와 다르게 설정하여 HTTP 연결 실패를 재현하고 복구했습니다.

## 5. 운영 관점에서 배운 점

Kubernetes Pod가 Running/Ready 상태라고 해서 Service 통신까지 정상임을 보장하지 않습니다.

장애가 발생하면 Service 설정, Pod Label, EndpointSlice 및 실제 HTTP 응답을 근거로 원인을 판단해야 합니다.
