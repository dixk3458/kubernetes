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

## 6. 프로젝트 파일 구성

### Kubernetes Manifests

| 파일                                                     | 설명                         |
| -------------------------------------------------------- | ---------------------------- |
| [nginx-deployment.yaml](manifests/nginx-deployment.yaml) | nginx Deployment 및 Pod 구성 |
| [nginx-clusterip.yaml](manifests/nginx-clusterip.yaml)   | ClusterIP Service 구성       |
| [nginx-nodeport.yaml](manifests/nginx-nodeport.yaml)     | NodePort Service 구성        |

### 장애 재현 및 복구 기록

| 장애 사례                                             | 주요 원인                                  |
| ----------------------------------------------------- | ------------------------------------------ |
| [Selector 불일치](incidents/selector-mismatch.md)     | Service Selector와 Pod Label 불일치        |
| [TargetPort 불일치](incidents/targetport-mismatch.md) | Service TargetPort와 실제 수신 포트 불일치 |

## 7. 검증 범위

본 프로젝트의 장애 재현 및 복구 결과는 기존 Kubernetes 가상머신 실습에서 관찰한 내용을 기반으로 한다.

포트폴리오용 Manifest는 실습 구성을 참고하여 로컬 VS Code에서 별도로 작성했으며, 해당 파일 자체를 이용한 신규 배포 검증은 수행하지 않았다.
