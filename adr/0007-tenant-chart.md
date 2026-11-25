# ADR-0007: tenant 차트로 테넌트 격리 계층 선언

- 상태: 제안
- 날짜: 2026-11-25
- 관련: [#14](https://github.com/Phantom-s-Lab/Phantom-cloud/issues/14), [phantom-cloud-gitops#14](https://github.com/Phantom-s-Lab/phantom-cloud-gitops/issues/14), [ADR-0005](0005-app-of-apps-applicationset.md), [ADR-0006](0006-image-ci-gitops-cd.md)

## 맥락
- 테넌트별 Namespace, Quota, PVC, Pod 등을 개별 YAML과 bash 반복문으로 명령형 생성
- 비밀번호를 환경변수로 주입, 테넌트 ServiceAccount에 네임스페이스 admin 권한 부여
- 네트워크 정책이 없어 테넌트 간, 테넌트에서 노드·내부망으로의 통신이 열려 있음

## 결정
- `charts/tenant` 하나가 테넌트 1개의 격리 계층 전체를 선언
  - Namespace: PSA `baseline` 강제, `restricted`는 warn·audit
  - ResourceQuota, LimitRange: 티어(`small`, `standard`)별 값
  - NetworkPolicy: 기본 전부 차단 → SSH(2222) 인바운드, DNS, 인터넷 아웃바운드만 허용, 사설 대역(10/8, 172.16/12, 192.168/16) 아웃바운드 차단
  - PVC 1개를 서브 경로로 분리: `home`(홈 디렉터리), `hostkeys`(SSH 호스트 키)
  - Deployment: tenant-base 이미지, ServiceAccount 토큰 미마운트
- 테넌트 1개 = `tenants/<이름>/values.yaml` 파일 1개, ApplicationSet(Git 파일 생성기)이 Application 생성
- Secret 관리 도구는 도입하지 않음
  - 공개키는 비밀값이 아니므로 values에 두고 ConfigMap으로 배포
  - SSH 호스트 키는 최초 기동 시 PVC에 생성 후 재사용
- 테넌트에게 쿠버네티스 권한을 주지 않음 (SSH로만 접근)
- 이미지 태그는 차트 values에서 공통 관리, 이미지 CI의 배포 PR 대상을 차트 values로 전환 (ADR-0006)

## 검토한 대안
| 대안 | 기각 사유 |
|---|---|
| 테넌트별 매니페스트 복사 | 테넌트가 늘수록 같은 파일이 반복되고 정책 변경 시 모두 수정해야 함 |
| Kustomize overlay | 티어별 값 선택 같은 조건 처리가 번거로움 |
| Sealed Secrets / External Secrets | 현재 Git에 넣어야 할 비밀값이 없음 |
| PSA restricted 강제 | sshd가 사용자 전환을 위해 root로 시작해야 해서 불가 |

## 결과
### 이점
- 테넌트 추가·삭제가 파일 1개 PR로 끝남
- 격리 정책을 한 곳에서 바꾸면 모든 테넌트에 반영
- Git에 비밀값이 없음

### 트레이드오프
- PSA baseline이라 컨테이너가 root로 시작함 (capability는 필요한 것만 허용)
- 테넌트 삭제 시 PVC(Reclaim: Delete)의 데이터도 함께 삭제됨
- 호스트 키가 PVC에 있어 PVC를 잃으면 호스트 키가 바뀜
