# ADR-0006: 직접 빌드 이미지의 CI와 GitOps 연계 CD

- 상태: 제안
- 날짜: 2026-11-25
- 관련: [#11](https://github.com/Phantom-s-Lab/Phantom-cloud/issues/11), [phantom-cloud-gitops#9](https://github.com/Phantom-s-Lab/phantom-cloud-gitops/issues/9), [ADR-0001](0001-repo-split.md), [ADR-0005](0005-app-of-apps-applicationset.md)

## 맥락
- 테넌트 파드가 외부 이미지(`latest` 태그)를 검증 없이 사용
- 비밀번호와 sudo 설정을 환경변수로 주입
- GitOps 도입(ADR-0005)으로 배포는 Git 기준이 됐으나, 이미지 생성과 태그 갱신은 수동

## 결정
- 직접 빌드하는 이미지의 소스는 `phantom-cloud-images` 레포에 둠 (이미지별 디렉터리)
- CI (GitHub 호스팅 러너)
  - PR: hadolint → 빌드 → 스모크 테스트(`sshd -t`) → Trivy(CRITICAL 발견 시 실패), 푸시 안 함
  - main: 같은 검증 통과 후 arm64/amd64 멀티 아키텍처 빌드 → GHCR 푸시
  - 태그는 커밋 SHA, `latest` 미사용
- CD
  - 푸시 후 gitops 레포에 이미지 태그 변경 PR 자동 생성, 병합 시 ArgoCD가 배포
  - CI는 클러스터 접근 권한을 갖지 않음
  - 자동 PR 권한: fine-grained PAT (gitops 레포 한정, Contents·Pull requests 쓰기, 만료 30일)
  - 현재 CD 대상은 파이프라인 검증용 샘플. 테넌트 차트 도입 시 차트 values로 전환
- gitops 레포 PR 검증: kubeconform, 차트가 있으면 helm lint
- SSH 호스트 키와 authorized_keys는 이미지에 넣지 않고 실행 시 Secret으로 주입

## 검토한 대안
| 대안 | 기각 사유 |
|---|---|
| Docker Hub | 별도 계정 토큰 관리 필요, 익명 pull 제한 |
| self-hosted 러너 | 공개 레포에서 외부 PR 코드가 내부 장비에서 실행될 위험 |
| CI에서 kubectl/helm으로 직접 배포 | CI에 클러스터 인증 정보가 필요하고 Git과 클러스터 상태가 어긋날 수 있음 |
| ArgoCD Image Updater | 배포 승인 지점(PR 리뷰)이 사라짐 |
| gitops main에 자동 커밋 | 리뷰 없이 배포됨 |

## 결과
### 이점
- 린트·테스트·스캔을 통과한 이미지만 배포됨
- 커밋 SHA 태그로 배포된 이미지의 소스 커밋을 추적 가능
- 배포 승인 지점이 PR로 남고, 롤백은 배포 PR revert

### 트레이드오프
- PAT는 개인 계정에 묶이고 만료마다 수동 갱신 필요 (GitHub App 전환 예정)
- arm64는 QEMU 에뮬레이션으로 빌드해 빌드 시간이 김
- GitHub(코드·CI·레지스트리)에 대한 의존도 증가
