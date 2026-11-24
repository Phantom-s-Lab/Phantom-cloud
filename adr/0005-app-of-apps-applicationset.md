# ADR-0005: 플랫폼은 App-of-Apps, 테넌트는 ApplicationSet으로 관리

- 상태: 제안
- 날짜: 2026-11-25
- 관련: [#8](https://github.com/Phantom-s-Lab/Phantom-cloud/issues/8), [phantom-cloud-gitops#5](https://github.com/Phantom-s-Lab/phantom-cloud-gitops/issues/5), [ADR-0002](0002-argocd-placement.md)

## 맥락
- ArgoCD Application을 수동으로 만들면 변경 이력이 Git에 남지 않음
- 관리 대상이 두 종류로 나뉨
  - 플랫폼 구성요소(ArgoCD, Traefik, 모니터링 등): 수가 적고, 차트·설정·설치 순서가 제각각
  - 테넌트: 수가 많고 자주 추가·삭제됨. 같은 차트에서 이름과 티어만 다름

## 결정
- 플랫폼 구성요소: App-of-Apps
  - `bootstrap/root.yaml`만 최초 1회 수동 적용, root가 `apps/`를 동기화
  - 실제 매니페스트·values는 `platform/<이름>/`에 둠
  - 자동 동기화(prune, selfHeal) 사용
  - root에는 resources-finalizer를 걸지 않음 (root 삭제 시 하위 앱 연쇄 삭제 방지)
- 테넌트: ApplicationSet(Git 파일 생성기), `tenants/<이름>/tenant.yaml` 단위
  - 구현은 테넌트 차트 작업에서 진행

## 검토한 대안
| 대안 | 기각 사유 |
|---|---|
| Application 수동 적용 | Git에 이력이 남지 않고 명령형 관리로 돌아감 |
| 전부 ApplicationSet | 플랫폼 구성요소는 설정이 제각각이라 템플릿 하나에 맞추기 어려움 |
| 전부 App-of-Apps | 테넌트마다 거의 같은 Application 파일을 복사해야 함 |

## 결과
### 이점
- root 하나만 적용하면 나머지는 Git에서 복구 가능 (클러스터 재구성 시 "ArgoCD 설치 → root 적용")
- 모든 변경이 PR로 남음

### 트레이드오프
- 긴급 수정도 Git으로 해야 함 (selfHeal이 직접 수정을 되돌림)
- root가 망가지면 하위 앱 동기화가 멈춤 (배포된 워크로드는 계속 동작)

## 참고
- [Argo CD: Cluster Bootstrapping (App of Apps)](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [Argo CD: ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
