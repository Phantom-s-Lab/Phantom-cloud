# ADR-0001: 레포를 gitops / infra / ctl로 분리

- 상태: 승인
- 날짜: 2026-11-19

## 맥락
- GitOps 전환을 위해 클러스터의 원하는 상태(desired state)를 Git으로 관리해야 함
- 게이트웨이 노드는 클러스터 밖에 있어 kube-apiserver가 없으므로 ArgoCD로 관리할 수 없음
- 이후 테넌트 운영을 위한 Go CLI 개발도 예정되어 있음

## 결정
- 쿠버네티스 API로 관리되는 것은 [`phantom-cloud-gitops`](https://github.com/Phantom-s-Lab/phantom-cloud-gitops), 클러스터 밖의 인프라는 [`phantom-cloud-infra`](https://github.com/Phantom-s-Lab/phantom-cloud-infra), CLI는 [`phantom-cloud-ctl`](https://github.com/Phantom-s-Lab/phantom-cloud-ctl)로 분리
- 기존 `Phantom-cloud`는 대표 페이지와 문서(ADR, 포스트모템, 패치노트)를 담당

## 검토한 대안
- 단일 레포(디렉터리로 구분): 관리할 레포는 줄지만, ArgoCD와 게이트웨이가 각자 필요 없는 설정까지 읽을 수 있게 됨

## 결과
### 이점
- 레포별 deploy key로 최소 권한을 지킬 수 있음 (ArgoCD는 gitops만, 게이트웨이는 infra만 읽음)
- 적용 방식(pull / push)에 따라 CI 워크플로를 분리하기 쉬움

### 트레이드오프
- 관리할 레포가 늘어남
- 레포 간에 공유하는 정보(테넌트 목록 등)의 참조 방식을 따로 정해야 함
