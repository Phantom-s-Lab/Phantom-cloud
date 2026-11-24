# ADR-0002: ArgoCD를 클러스터 내 x86 노드에 배치

- 상태: 승인
- 날짜: 2026-11-23
- 관련: [#3](https://github.com/Phantom-s-Lab/Phantom-cloud/issues/3), [phantom-cloud-gitops#2](https://github.com/Phantom-s-Lab/phantom-cloud-gitops/issues/2)

## 맥락
- GitOps 도입을 위해 ArgoCD를 설치해야 함
- 클러스터는 라즈베리파이 노드(arm64)와 x86 노드 1대(amd64)로 구성되어 있고, 라즈베리파이는 메모리 여유가 적음
- x86 노드(node03)는 4 vCPU, 메모리 16GiB이며, 현재 시스템 파드(Cilium, kube-proxy, node-exporter) 외에 올라간 워크로드가 없음
- ArgoCD의 repo-server는 Helm 렌더링 시 메모리 사용량이 순간적으로 커져 OOM 위험이 있음
- 관리 대상 클러스터는 하나뿐임

## 결정
- ArgoCD를 관리 대상 클러스터 안에 설치하고, nodeSelector(`kubernetes.io/arch: amd64`)로 x86 노드에 배치
- non-HA 모드로 설치하고, 사용하지 않는 dex(SSO)는 비활성화
- 최초 설치만 Helm으로 수동 진행하고, 이후 ArgoCD 자신도 Git으로 관리

## 검토한 대안
- 별도 관리 클러스터(hub-spoke): 여러 클러스터를 관리할 때 적합하지만, 단일 클러스터에는 과하고 추가 자원이 필요함
- 라즈베리파이 노드에 배치: 메모리 여유가 적어 repo-server OOM 위험이 큼
- HA 모드: 가용성은 높아지지만 Redis HA 등으로 리소스 사용량이 크게 늘어남

## 검증 (2026-11-25)
- 설치: 차트 `argo/argo-cd` 10.11.7 (앱 v3.5.7), K8s v1.33.13
  - 3.5는 공식 테스트 범위가 v1.33~v1.36이라 현재 버전과 목표 버전(v1.35)을 모두 포함
- 배치: 파드 5개(application-controller, applicationset-controller, repo-server, server, redis) 모두 x86 노드(node03)에서 Running
- dex, notifications 미생성 확인 (`helm template` 렌더링과 실제 설치 결과)
- 대기 상태 리소스 사용량 (`kubectl top`, Application 0개)

  | 컴포넌트 | CPU | 메모리 |
  |---|---|---|
  | application-controller | 1m | 33Mi |
  | applicationset-controller | 1m | 31Mi |
  | repo-server | 2m | 29Mi |
  | server | 3m | 33Mi |
  | redis | 4m | 9Mi |
  | 합계 | 약 11m | 약 135Mi |

  - node03 전체 사용량: CPU 2%, 메모리 8%
  - Application이 늘면 controller와 repo-server 사용량이 증가하므로, 앱 추가 후 재측정해 requests/limits를 조정한다
- "ArgoCD 자기 관리" 결정은 이후 작업에서 구현·검증한다

## 결과
### 이점
- 메모리 여유가 있는 노드에서 안정적으로 동작
- 클러스터가 망가져도 "ArgoCD 설치 → root 앱 적용"으로 Git에서 전체 상태 복구 가능

### 트레이드오프
- x86 노드가 단일 장애점이 됨 (노드 장애 시 동기화가 멈춤. 이미 배포된 워크로드는 계속 동작)
- 관리 대상 클러스터와 운명을 같이함
