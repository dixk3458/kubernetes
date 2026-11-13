# Kubernetes Startup Probe — 느린 애플리케이션 시작과 반복 재시작 문제

## 1. 실습 개요

Kubernetes에서 애플리케이션의 시작 시간이 오래 걸릴 때 Liveness Probe가 너무 일찍 실패하여 컨테이너가 반복적으로 재시작되는 상황을 재현하고, Startup Probe를 적용해 복구한 실습입니다.

이번 실습에서는 nginx의 실행을 의도적으로 40초 지연시켰습니다.

Startup Probe 적용 전후의 동작을 비교하여 애플리케이션 초기화 시간과 Probe 설정 사이의 관계를 확인했습니다.

### 실습 목표

- Startup Probe가 필요한 이유 이해
- 애플리케이션 시작 지연 상황 재현
- Liveness Probe에 의한 불필요한 재시작 원인 분석
- Startup Probe와 Liveness·Readiness Probe의 실행 관계 이해
- Pod의 Ready 상태와 Deployment의 정상화 여부 확인

## 2. 실습 환경

| 항목                   | 설정                 |
| ---------------------- | -------------------- |
| OS                     | Rocky Linux 9.8      |
| Kubernetes             | v1.36.3              |
| Namespace              | `probe-resource-lab` |
| Deployment             | `startup-demo`       |
| Container Image        | `nginx:1.28-alpine`  |
| Replicas               | 1                    |
| 애플리케이션 시작 지연 | 40초                 |
| 검사 방식              | HTTP GET `/`         |

## 3. 장애 시나리오

일반적으로 nginx는 컨테이너가 시작되면 비교적 빠르게 HTTP 요청을 처리할 수 있습니다.

하지만 이번 실습에서는 nginx 실행 전에 40초 동안 대기하도록 구성했습니다.

사용한 명령어:

`sh -c "sleep 40; exec nginx -g 'daemon off;'"`

이 설정에서는 컨테이너가 시작되어도 40초가 지나기 전까지 nginx가 실행되지 않습니다.

따라서 초기화가 완료되기 전에 Liveness Probe가 HTTP 요청을 보내면 검사에 실패할 수 있습니다.

## 4. 장애 재현 — Startup Probe 적용 전

### 4.1 초기 Liveness Probe 설정

초기 실습에서는 애플리케이션의 시작 지연에 비해 Liveness 검사 시간이 짧도록 구성했습니다.

| 설정           | 값  |
| -------------- | --- |
| 검사 경로      | `/` |
| 최초 검사 지연 | 3초 |
| 검사 주기      | 5초 |
| 타임아웃       | 2초 |
| 실패 임계 횟수 | 3회 |

nginx 실행에는 40초가 필요하지만 Liveness Probe는 그보다 먼저 검사를 시작합니다.

### 4.2 장애 증상

실습 과정에서 Liveness Probe 실패로 인한 컨테이너 재시작과 `CrashLoopBackOff` 상태를 관찰했습니다.

이는 애플리케이션이 시작을 완료하기 전에 Liveness Probe가 실패 임계 횟수에 도달하는 상황이었습니다.

### 4.3 원인 분석

L
