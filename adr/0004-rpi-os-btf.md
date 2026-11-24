# ADR-0004: Raspberry Pi 노드 OS를 Ubuntu Server 24.04 LTS로 교체 (커널 BTF 확보)

- 상태: 제안
- 날짜: 2026-11-24
- 관련: Issue #5, [ADR-0002](0002-argocd-placement.md), ADR-0003(클러스터 업그레이드)

## 맥락

- 클러스터를 v1.31에서 v1.35로 올리는 중, v1.33에서 Cilium을 1.18.17 → 1.19.11로 올리자
  Raspberry Pi 노드 3대(master, node01, node02)에서 eBPF 프로그램 로드가 실패했다.
  - 에러: `apply CO-RE relocations: load kernel spec: no BTF found for kernel version 6.12.47+rpt-rpi-2712`
  - 영향: 기존 트래픽은 유지됐으나 RPi 노드에서 새 파드 생성이 멈췄다.
  - x86 노드(node03, Debian 커널)는 정상이었다.
- 원인
  - Cilium 1.19부터 커널 BTF(`CONFIG_DEBUG_INFO_BTF=y`)가 기본 요구사항이다.
    이 내용은 1.19 시스템 요구사항 문서에만 있고 업그레이드 가이드에는 없다 (cilium/cilium#48778).
  - KPR을 켜면 해당 프로그램이 무조건 로드되어 Helm 값으로 피할 수 없다.
  - Raspberry Pi OS 커널은 `CONFIG_DEBUG_INFO_NONE=y`로 빌드되어 BTF가 없고,
    공식 지원 계획도 없다 (raspberrypi/linux#6622).
- Helm 롤백으로 Cilium 1.18.17에 복구했다. Cilium 1.18은 K8s 1.33까지만 공식 지원하므로
  K8s 1.34 이상, containerd 2.2 전환이 막혀 있다. K8s 1.33은 이미 지원이 종료된 버전이다.

## 결정

- Raspberry Pi 노드 3대의 OS를 Ubuntu Server 24.04 LTS로 교체한다.
  - linux-raspi 커널은 2024-07부터 BTF를 켜고 빌드된다 (Launchpad #2065829, Fix Released).
  - 현재 배포 패키지 `linux-modules-6.8.0-1068-raspi`의 설정에서
    `CONFIG_DEBUG_INFO_BTF=y`, `CONFIG_DEBUG_INFO_BTF_MODULES=y`를 확인했다.
- 교체 전까지는 K8s v1.33.13 + Cilium 1.18.17(공식 지원 조합)로 운영한다.
- 진행 순서
  1. GitOps(ArgoCD) 도입 후 클러스터 구성을 Git에 올린다.
  2. SD 카드로 RPi 5에서 Ubuntu 24.04를 부팅해 `/sys/kernel/btf/vmlinux`와 NVMe, 이더넷을 확인한다.
  3. GitOps로 복구되지 않는 것(PV 데이터, Git 밖의 Secret, etcd 스냅샷)을 백업한다.
  4. 워커 → master 순으로 OS를 교체하고 클러스터를 새로 구성한 뒤 Git 기준으로 복구한다.
  5. 새 클러스터는 K8s v1.35 + Cilium 1.19 이상으로 구성한다.
  6. 커널 보안 패치는 unattended-upgrades + kured로 자동화한다.
- node03(x86, Debian 13)은 BTF가 있으므로 OS를 유지한다.

## 검토한 대안

| 대안 | 기각 사유 |
|---|---|
| RPi 커널을 BTF 켜고 직접 빌드 (CI로 자동화) | 가능하지만 RPi 공식 지원 밖이고, 커널 보안 패치마다 다시 빌드해야 한다 |
| detached BTF 파일을 따로 생성해 제공 | Cilium 공식 지원 방식이 아니고, 커널이 바뀔 때마다 다시 만들어야 한다 |
| Cilium 1.19.0~1.19.7로 고정 | BTF가 1.19의 공식 요구사항이라 지원 범위 밖이고, 이후 패치를 받을 수 없다 |
| Cilium 1.18 + K8s 1.34 이상 운영 | 공식 테스트 범위 밖의 조합이다 |
| Debian 범용 arm64 커널 사용 | BTF는 있으나 RPi 5의 팬·온도 제어, RP1 이더넷 지원이 부족하다 |
| 현 상태(K8s 1.33) 유지 | K8s 1.33은 지원 종료 버전이고 ArgoCD 3.6 테스트 범위(1.34~1.37) 밖이다 |

## 결과

### 이점
- 배포판 공식 커널로 BTF를 확보하고, 커널 업데이트를 apt로 받을 수 있다.
- Cilium 1.19 이상, K8s 1.34 이상으로의 업그레이드 경로가 열린다.
- GitOps 기반 클러스터 재구성(재해 복구) 절차를 실제로 검증할 수 있다.

### 트레이드오프
- 노드 3대를 재설치하고 클러스터를 새로 구성해야 한다. master 교체 시 클러스터 전체가 재구성된다.
- 노드 OS가 Ubuntu(RPi)와 Debian(x86)으로 섞인다.
- Ubuntu 24.04 RPi 커널(6.8)이 현재 Raspberry Pi OS 커널(6.12)보다 버전이 낮다.
- 실제 RPi 5 부팅 확인 전이므로, 확인 결과에 따라 이 결정은 거부 또는 대체될 수 있다.
