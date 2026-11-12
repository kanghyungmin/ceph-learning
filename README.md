# Ceph 학습 가이드 — 하루 2시간, 6주

주 5회 × 6주 × 2시간 = **60시간** 과정입니다. 주말은 복습·보충에 사용합니다. 매일 공부한다면 나머지 이틀은 새 진도 대신 명령 재실행과 기록 정리에 쓰세요.

0주차 사전 학습을 별도로 진행하면 총 62시간입니다. 본 과정은 기존 6주·60시간을 유지합니다.

목표는 Kubernetes에서 CephFS를 사용하고, CSI 장애를 진단하며, Ceph 내부 상태를 근거로 설명하는 것입니다. 설치·OSD 장애 주입·복구·관리 설정 변경은 별도 환경이 필요한 후속 과정입니다.

## 주차별 하루 학습 가이드

각 문서는 개념 설명, 2시간 진행 순서, 실습, 출력 해석, 확인 질문과 해설을 포함합니다.

- [0주차 — 프로비저닝과 Kubernetes 스토리지 인터페이스](weeks/week-00.md) · 사전 학습 2시간
- [1주차 — 공유 스토리지를 직접 사용하기](weeks/week-01.md) · Day 1~5
- [2주차 — PVC 수명주기와 CSI 문제 해결](weeks/week-02.md) · Day 6~10
- [3주차 — Ceph 내부 구조와 상태 읽기](weeks/week-03.md) · Day 11~15
- [4주차 — CephFS와 Kubernetes 연결하기](weeks/week-04.md) · Day 16~20
- [5주차 — 운영 관찰과 진단 능력](weeks/week-05.md) · Day 21~25
- [6주차 — 종합 과제와 다음 단계](weeks/week-06.md) · Day 26~30

학습 기록은 [노트 양식](notes/TEMPLATE.md)을 복사해 작성하세요. 아래 일정은 전체 개요이며 상세 실습은 주차별 문서를 기준으로 진행합니다.

## 공개 예제의 환경 설정

이 저장소는 실제 환경에서 확인한 학습 과정을 바탕으로 작성했습니다. 도메인·계정·subvolume group은 예시 값으로 치환했습니다. `qks-shared`, `qks-block`, `qks-ceph` 등 환경별 이름은 자신의 클러스터에 맞게 수정하세요. 명령은 이 저장소를 클론한 디렉터리에서 실행합니다. 과거 확인 결과가 새 환경의 권한이나 구성을 보장하지는 않습니다.

```bash
git clone https://github.com/kanghyungmin/ceph-learning.git
cd ceph-learning
```

## 현재 환경과 범위

2026-11-12 확인 기준입니다. 권한과 상태는 이후 바뀔 수 있습니다.

| 항목 | 확인 내용 |
|---|---|
| Kubernetes | 전체 리소스 wildcard 권한 검사 및 Pod exec 검사에서 yes |
| CephFS StorageClass | `qks-shared`, `cephfs.csi.ceph.com`, Immediate, 확장 가능, Delete |
| 블록 StorageClass | `qks-block`, `csi.kubevirt.io`, WaitForFirstConsumer. Ceph RBD 백엔드인지는 미확인 |
| CephFS | `qks-ceph`, 데이터 Pool `cephfs.qks-ceph.data` |
| CSI subvolume group | `csi-example-cluster` |
| Ceph 계정 | `client.example-cephfs` |
| 실제 성공한 조회 | status, health detail, df, osd tree, osd pool ls detail, pg stat, fs status, subvolumegroup/subvolume ls, orch status, config dump |
| 제한 | auth get/ls 거부. 정확한 caps 및 쓰기·OSD 제어 권한은 미확인 |

기존 `code-server` PVC와 시스템 CSI는 변경하지 않습니다. 실습 리소스는 `ceph-lab`에만 생성합니다. 현재 Ceph는 여러 Pool과 서비스가 있는 사용 중인 클러스터이므로 전체 클러스터 성능 부하·OSD 중단·Pool 삭제·설정 변경은 이 과정에서 실행하지 않습니다.

## 매일 진행 방식

- 25분: 그날 개념 읽기, 실행 결과 예상하기
- 75분: 실습 및 출력 해석
- 20분: 명령·관찰·해석·남은 질문 기록

매일 아래 양식을 복사해 `notes/day-01.md`처럼 기록하세요. 인증 키, kubeconfig, Secret 원문은 기록하거나 Git에 올리지 않습니다.

```markdown
# Day NN — 주제
- 예상:
- 실행한 명령:
- 관찰한 결과:
- 결과를 설명하는 원리:
- 문제가 생겼다면 원인과 해결:
- 남은 질문:
- 완료 기준 충족 여부:
```

## 준비: Kubernetes와 Ceph CLI

가이드의 `kubectl`은 이미 설정한 `k`로 줄여 입력해도 됩니다. 기본 kubeconfig를 사용합니다.

```bash
kubectl config current-context
kubectl get nodes
kubectl get sc
```

Ceph 명령 예시에서 `ceph ...`는 **인증 설정이 완료된 Ceph CLI 환경에서** 실행한다는 뜻입니다. 로컬 Mac에서 그대로 실행되는 상태는 아직 아닙니다. 앞선 권한 검사는 CSI 컨테이너의 Ceph CLI와 임시 keyring으로 수행했습니다.

1. 1~2주차는 kubectl만으로 진행할 수 있습니다.
2. 3주차 전에 전용 toolbox 또는 관리 호스트의 CLI 접근 경로를 준비합니다. 가능하면 운영자가 발급한 별도 학습용 조회 계정을 사용하세요.
3. 운영 CSI 컨테이너는 학습용 셸로 상시 사용하지 않습니다. 기존 CSI Secret을 복사해 저장소에 보관하지 마세요.
4. CLI 환경에서 먼저 `ceph -s`를 실행해 인증과 연결을 확인합니다. 실패하면 3주차의 CLI 실습은 보류하고 아래 공식 문서와 1~2주차 복습을 진행합니다.

조회 명령도 다른 Pool 이름·주소·운영 설정을 보여줄 수 있으므로 결과를 공개하기 전 내용을 확인하세요.

## 1주차 — 공유 스토리지를 직접 사용하기

### Day 1: 전체 구조와 현재 환경

개념: StorageClass는 저장소 제공 방식, PVC는 요청, PV는 할당된 볼륨, CSI는 Kubernetes와 스토리지 간 연결입니다.

```bash
kubectl get sc qks-shared -o yaml
kubectl get csidrivers
kubectl get pods -n kube-system | rg ceph
```

`PROVISIONER`, `reclaimPolicy`, `volumeBindingMode`, `allowVolumeExpansion`을 찾아 설명하세요. `qks-block`과 `qks-shared`의 차이를 기록합니다.

완료 기준: Pod → PVC → PV → CSI → CephFS 흐름을 말로 설명할 수 있습니다.

### Day 2: 1Gi RWX PVC와 Pod 두 개 생성

저장소 루트에서 실행합니다. 이 명령부터 실제 실습 리소스를 생성합니다.

```bash
kubectl apply -f labs/01-shared.yaml
kubectl rollout status deployment/shared-reader -n ceph-lab --timeout=180s
kubectl get pods,pvc -n ceph-lab -o wide
```

Pod는 anti-affinity로 서로 다른 노드에 배치됩니다. 두 노드 환경에서도 재시작이 진행되도록 Recreate 전략을 사용하며, 재시작 중에는 두 Pod가 잠시 중단됩니다. 사용 가능한 노드가 2개 미만이면 한 Pod는 Pending이 됩니다. 이는 스토리지 오류가 아닐 수 있으므로 `kubectl describe pod`의 스케줄링 이벤트를 확인하세요.

완료 기준: PVC Bound, Pod 두 개 Ready, 서로 다른 노드에서 같은 PVC 사용.

### Day 3: 서로 다른 노드에서 파일 공유

아래 변수는 같은 터미널에서 사용합니다. 새 터미널에서는 다시 설정하세요.

```bash
POD_A=$(kubectl get pods -n ceph-lab -l app=shared-reader -o jsonpath='{.items[0].metadata.name}')
POD_B=$(kubectl get pods -n ceph-lab -l app=shared-reader -o jsonpath='{.items[1].metadata.name}')
kubectl exec -n ceph-lab "$POD_A" -- sh -c 'echo hello-ceph > /data/day03.txt; sync'
kubectl exec -n ceph-lab "$POD_B" -- cat /data/day03.txt
kubectl exec -n ceph-lab "$POD_B" -- sh -c 'echo from-pod-b >> /data/day03.txt'
kubectl exec -n ceph-lab "$POD_A" -- cat /data/day03.txt
```

완료 기준: 양방향 공유 확인. RWX가 애플리케이션의 동시 쓰기 충돌까지 해결해 주는 것은 아님을 설명합니다.

### Day 4: Pod 수명과 데이터 수명

```bash
kubectl rollout restart deployment/shared-reader -n ceph-lab
kubectl rollout status deployment/shared-reader -n ceph-lab --timeout=180s
kubectl exec -n ceph-lab deployment/shared-reader -- cat /data/day03.txt
kubectl get pvc -n ceph-lab
```

완료 기준: Pod가 교체되어도 파일이 남는 이유와 PVC를 삭제했을 때의 차이를 설명합니다.

### Day 5: 권한과 주간 복습

```bash
kubectl exec -n ceph-lab deployment/shared-reader -- id
kubectl exec -n ceph-lab deployment/shared-reader -- ls -ldn /data
kubectl exec -n ceph-lab deployment/shared-reader -- sh -c 'umask; touch /data/permissions.txt; ls -ln /data/permissions.txt'
```

UID, GID, fsGroup, 파일 mode를 비교하세요. fsGroup은 모든 기존 파일의 소유 UID를 바꾸는 기능이 아닙니다.

완료 기준: YAML을 보지 않고 PVC와 Pod 관계를 설명하고 1주차 결과를 한 페이지로 정리.

## 2주차 — PVC 수명주기와 CSI 문제 해결

### Day 6: PVC에서 PV까지 추적

```bash
PV_NAME=$(kubectl get pvc shared-workspace -n ceph-lab -o jsonpath='{.spec.volumeName}')
kubectl get pv "$PV_NAME" -o yaml
kubectl describe pvc shared-workspace -n ceph-lab
```

`claimRef`, `csi.driver`, `volumeHandle`, `volumeAttributes`를 확인합니다. volumeHandle은 드라이버 식별자이므로 사람이 읽을 수 있는 subvolume 이름과 동일하다고 가정하지 않습니다.

완료 기준: Kubernetes 객체와 Ceph 내부 객체를 구분.

### Day 7: 1Gi에서 2Gi로 확장

먼저 `labs/01-shared.yaml`의 요청 용량을 `2Gi`로 수정한 뒤 적용합니다. 원본 YAML에도 변경을 남겨 이후 재적용 시 축소 요청이 생기지 않게 합니다.

```bash
kubectl apply -f labs/01-shared.yaml
kubectl get pvc shared-workspace -n ceph-lab -w
# 용량 반영을 확인한 뒤 Ctrl-C
kubectl exec -n ceph-lab deployment/shared-reader -- df -h /data
kubectl describe pvc shared-workspace -n ceph-lab
```

완료 기준: 요청량·PVC capacity·Pod에서 보이는 용량을 비교. CephFS 클라이언트에서 보이는 값은 quota/클라이언트 설정에 영향을 받을 수 있으므로 df 하나만으로 성공을 판정하지 않습니다. PVC 축소는 하지 않습니다.

### Day 8: 의도적으로 Pending 만들고 진단

새 파일 `labs/02-pending.yaml`에 namespace가 `ceph-lab`, 이름이 `missing-class`, StorageClass가 `does-not-exist`, 용량이 `1Gi`, accessModes가 `ReadWriteMany`인 PVC를 작성합니다.

```bash
kubectl apply -f labs/02-pending.yaml
kubectl describe pvc missing-class -n ceph-lab
kubectl get events -n ceph-lab --sort-by=.metadata.creationTimestamp
kubectl delete pvc missing-class -n ceph-lab
```

완료 기준: 상태만 보고 판단하지 않고 이벤트 메시지로 원인을 설명. 기존 StorageClass와 CSI는 수정하지 않습니다.

### Day 9: 스냅샷 가능 여부 판단

```bash
kubectl api-resources --api-group=snapshot.storage.k8s.io
kubectl get volumesnapshotclass
```

VolumeSnapshot API와 `driver: cephfs.csi.ceph.com`인 SnapshotClass가 있는지 확인합니다. CSI에 snapshotter가 있다는 것만으로 사용 가능하다고 판단하지 않습니다.

지원된다면 해당 Class를 사용하는 VolumeSnapshot YAML을 공식 예제로 작성해 실습 PVC만 스냅샷합니다. `readyToUse: true` 확인 후 새로운 이름의 PVC로 복원하고 파일을 비교합니다. 지원되지 않으면 객체 간 관계와 필요한 구성요소를 문서로 정리합니다. 애플리케이션 일관성은 별도 문제이므로 실습 동안 파일 쓰기를 멈춥니다.

완료 기준: VolumeSnapshot → VolumeSnapshotContent → 새 PVC의 관계 또는 지원되지 않는 구체적 이유 기록.

### Day 10: CSI 로그로 요청 추적

```bash
kubectl logs -n kube-system deployment/ceph-csi-cephfs-provisioner -c csi-cephfsplugin --since=15m --tail=200
kubectl logs -n kube-system deployment/ceph-csi-cephfs-provisioner -c csi-resizer --since=15m --tail=100
```

PVC UID·PV volumeHandle·타임스탬프를 바탕으로 실습 요청과 로그를 연결합니다. 로그가 없으면 시간 범위를 조정하되 전체 운영 로그를 대량 수집하지 않습니다.

완료 기준: Pending 진단 절차를 이벤트 → CSI controller → nodeplugin 순서로 정리.

## 3주차 — Ceph 내부 구조와 상태 읽기

이번 주부터 인증된 Ceph CLI가 필요합니다. 아래는 전부 조회 명령입니다.

### Day 11: 서비스와 계층 이해

공식 입문 문서를 읽고 RADOS, CephFS, RBD, RGW 및 MON·MGR·OSD·MDS를 정리합니다.

```bash
ceph -s
ceph versions
```

`ceph versions`는 이번 과정에서 처음 확인하는 명령입니다. 거부되면 에러를 기록하고 status 조회만 진행합니다. 현재 모든 명령의 권한이 보장된 것은 아닙니다.

완료 기준: 저장 인터페이스와 클러스터 데몬의 역할을 구분.

### Day 12: health와 quorum

```bash
ceph -s
ceph health detail
```

MON quorum, active/standby MGR, OSD up/in, PG 상태를 읽습니다. `HEALTH_OK`가 곧 백업이나 모든 애플리케이션의 정상 상태를 보장하지는 않습니다.

완료 기준: 출력 각 항목을 설명하고 확인 시각을 기록.

### Day 13: OSD와 장애 도메인

```bash
ceph osd tree
```

root → host → OSD 구조와 weight를 확인합니다. 호스트 수·OSD 수를 기록하고 디스크 장애와 호스트 장애의 차이를 설명합니다. OSD out/down 명령은 실행하지 않습니다.

완료 기준: 현재 CRUSH 트리에서 확인 가능한 장애 도메인을 그림으로 정리.

### Day 14: Pool과 용량

```bash
ceph df
ceph osd pool ls detail
```

Pool의 replicated/erasure 유형, size/min_size, PG 수를 확인합니다. raw used, stored, available을 구분하세요. 복제 배수 외에도 메타데이터·정렬·압축·EC 등으로 실제 사용량이 달라질 수 있습니다.

완료 기준: 한 Pool을 골라 논리 데이터와 물리 사용량 차이를 설명.

### Day 15: PG와 배치

```bash
ceph pg stat
ceph osd pool ls detail
```

object → PG → OSD 관계, active와 clean의 의미를 학습합니다. degraded, undersized, peering은 문서 사례로 비교합니다.

완료 기준: MON이 파일 데이터를 직접 저장하는 서버가 아닌 이유와 CRUSH의 역할을 설명.

## 4주차 — CephFS와 Kubernetes 연결하기

### Day 16: CephFS 메타데이터와 데이터

```bash
ceph fs status qks-ceph
ceph osd pool ls detail
```

MDS가 다루는 메타데이터와 OSD에 저장되는 데이터를 구분합니다. 파일 내용 전체가 MDS를 통과한다고 가정하지 않습니다.

완료 기준: 파일 열기와 데이터 읽기의 개념적 흐름 설명.

### Day 17: PVC와 subvolume 연결

```bash
ceph fs subvolumegroup ls qks-ceph
ceph fs subvolume ls qks-ceph --group_name csi-example-cluster
```

Day 6의 PV 정보와 CSI 로그를 함께 사용해 실습 볼륨을 추적합니다. 이름만 보고 운영 subvolume을 삭제하거나 변경하지 않습니다.

완료 기준: PVC 하나가 CephFS 파일시스템 전체와 동일한 것이 아님을 설명.

### Day 18: 권한의 여러 계층

Kubernetes RBAC, CephX caps, CephFS POSIX 권한, Pod securityContext를 표로 비교합니다. 이전 확인에서 `ceph auth get`과 `auth ls`는 거부되었습니다. caps를 모르는 상태에서 자신의 쓰기 권한을 추측하지 않습니다.

완료 기준: `kubectl exec` 성공, `ceph -s` 성공, 파일 쓰기 성공이 각각 어떤 권한을 검증하는지 설명.

### Day 19: CSI controller와 nodeplugin

```bash
kubectl get pods -n kube-system -o wide | rg ceph
kubectl get csidriver cephfs.csi.ceph.com -o yaml
kubectl describe pod -n ceph-lab -l app=shared-reader
```

생성·확장 등 controller 동작과 노드의 mount 동작을 구분합니다. secret 값 대신 StorageClass의 참조 이름만 확인합니다.

완료 기준: PVC 생성 실패와 Pod 마운트 실패의 조사 경로를 따로 작성.

### Day 20: 스토리지 선택 비교

code-server YAML을 읽고 RWO/RWX, 블록/파일 저장소, Immediate/WaitForFirstConsumer 차이를 표로 정리합니다. RWO는 일반적으로 단일 노드의 읽기·쓰기 마운트를 뜻하며 단일 Pod 제한과 같지 않습니다.

완료 기준: code-server를 공유 저장소로 옮길 때 새 PVC와 데이터 복사가 필요한 이유, RWX여도 애플리케이션 복제 안전성이 보장되지 않는 이유 설명. 실제 기존 PVC는 변경하지 않습니다.

## 5주차 — 운영 관찰과 진단 능력

### Day 21: 3회 관측으로 변화 읽기

실습 시작·중간·종료에 `ceph -s`, `ceph df`, `ceph pg stat`을 실행해 시각과 함께 기록합니다. 관측치가 변해도 자신의 실습 때문이라고 단정하지 않습니다.

완료 기준: 누적 용량과 순간 I/O 지표를 구분하고 관측의 한계를 적기.

### Day 22: 작은 파일 I/O와 메타데이터

실습 PVC에서만 1MiB 파일 하나를 쓰고 읽습니다. 성능 벤치마크가 아니라 I/O 경로 확인입니다.

```bash
kubectl exec -n ceph-lab deployment/shared-reader -- sh -c 'dd if=/dev/zero of=/data/io-sample.bin bs=1024 count=1024; sync; cksum /data/io-sample.bin'
```

다른 Pod에서도 cksum을 비교합니다. 캐시·부하·파일 크기를 통제하지 않았으므로 이 결과로 Ceph 성능을 평가하지 않습니다.

완료 기준: 데이터 I/O와 파일 생성·목록 같은 메타데이터 작업의 차이 설명.

### Day 23: 장애 진단 표 만들기

다음 증상마다 확인 명령과 가능한 원인을 2개 이상 작성합니다: PVC Pending, Pod Pending, ContainerCreating, Permission denied, 용량 부족, 파일이 안 보임.

완료 기준: 증상 → Kubernetes 이벤트 → CSI → Ceph 상태 → 애플리케이션 순서의 조사표 작성. 장애 재현은 Day 8의 실습 PVC 범위로 제한.

### Day 24: 운영 구성과 orchestrator

```bash
ceph orch status
ceph config dump
```

orchestrator의 역할과 설정 계층을 읽습니다. config dump는 기본값까지 전부 보여주는 명령이 아님을 확인합니다. 설정 변경과 서비스 재배포는 하지 않습니다.

완료 기준: CSI가 Ceph 클러스터 자체를 설치·운영하는 도구와 다른 이유 설명.

### Day 25: 복구·백업과 삭제 정책

Day 9 복원 결과를 복습합니다. 스냅샷이 없다면 실습 텍스트 파일 하나를 로컬 임시 폴더로 복사해 원본과 cksum을 비교합니다. 스냅샷은 같은 클러스터의 장애까지 보호하는 독립 백업과 같지 않습니다.

완료 기준: Pod 삭제, PVC 삭제, Ceph 클러스터 장애 각각에서 필요한 복구 수단 정리. 현재 StorageClass가 Delete임을 다시 확인.

## 6주차 — 종합 과제와 다음 단계

### Day 26: RBD와 RGW 개념 확장

공식 RBD/RGW 문서를 읽고 CephFS/RBD/RGW를 파일·블록·오브젝트 API로 비교합니다. `qks-block`을 Ceph RBD라고 단정하지 않습니다. RGW 프로세스가 있다고 S3 자격 증명까지 있다는 뜻도 아닙니다.

완료 기준: DB 디스크, 공유 워크스페이스, S3 백업 대상의 선택 근거 설명.

### Day 27: 처음부터 다시 배포하고 검증 설계

기존 실습 데이터를 지우지 않고 별도 이름의 두 번째 PVC와 Pod로 Day 2~4를 재현합니다. YAML의 namespace와 모든 참조 이름을 검토한 뒤 적용합니다.

완료 기준: 생성 → 공유 확인 → Pod 교체 → 영속성 확인을 스스로 수행. 추가 PVC 사용량을 기록.

### Day 28: 종합 진단 과제

Day 8의 없는 StorageClass PVC를 새 이름으로 재현합니다. 20분 이내에 이벤트에서 원인을 찾고 올바른 Class를 사용하는 새 PVC YAML을 제시합니다. 바인딩된 PVC의 StorageClass를 직접 변경하지 않습니다.

완료 기준: 증거와 원인·해결책을 분리해 기록하고 잘못 만든 미바인딩 PVC 정리.

### Day 29: 운영 읽기 보고서

다음 내용을 한 문서로 정리합니다: 현재 구조, MON/MGR/OSD/MDS 역할, Pool/PG 구성, 용량 해석, CephFS와 PVC 매핑, 권한 제한, 장애 대응 순서. 운영 주소와 식별자는 외부 공유본에서 제거합니다.

완료 기준: 다른 사람이 문서만 보고 이 환경의 저장소 경로를 이해.

### Day 30: 최종 점검과 후속 실습 설계

아래 질문을 자료 없이 설명하고 어려운 항목은 복습합니다.

- PVC/PV/StorageClass/CSI는 각각 무엇인가?
- RWO와 RWX는 무엇을 보장하고 무엇을 보장하지 않는가?
- CephFS·RBD·RGW는 RADOS와 어떻게 연결되는가?
- MON·MGR·OSD·MDS는 어떤 역할인가?
- Pool·PG·CRUSH는 어떻게 연결되는가?
- Pending과 Permission denied는 어디서부터 조사하는가?
- 스냅샷과 백업은 어떤 실패 범위를 보호하는가?

후속 과정은 별도 클러스터에서 구축 → Pool 생성 → 복제/CRUSH 변경 → OSD 장애 → recovery/backfill → 인증 관리 순으로 진행합니다. VM·디스크 등 환경 요건은 선택한 Ceph 배포 방식에 맞게 별도로 산정합니다. 60시간 과정 완료를 운영 변경 숙련과 동일시하지 않습니다.

## 실습 종료와 정리

데이터를 먼저 확인·백업하고 **실습 네임스페이스에 한해서** 정리합니다. 아래는 선택 실행하는 삭제 명령입니다.

```bash
kubectl get all,pvc -n ceph-lab
# 더 필요한 데이터가 없다는 것을 확인한 뒤 실행
kubectl delete namespace ceph-lab
```

현재 `qks-shared`는 Delete 정책이므로 PVC 삭제 시 연결된 CephFS 데이터도 삭제될 수 있습니다. snapshot을 만든 경우 VolumeSnapshotClass의 deletionPolicy와 잔여 VolumeSnapshotContent도 별도로 확인하세요. 기존 code-server와 시스템 리소스는 정리 대상이 아닙니다.

## 공식 참고 자료

- [Ceph 입문과 구성요소](https://docs.ceph.com/en/latest/start/beginners-guide/)
- [Ceph Storage Cluster 구조](https://docs.ceph.com/en/latest/architecture/storage-cluster/)
- [CephFS](https://docs.ceph.com/en/latest/cephfs/)
- [CephFS와 Ceph-CSI](https://docs.ceph.com/en/latest/csi/cephfs/)
- [Ceph-CSI 문서와 예제](https://github.com/ceph/ceph-csi/tree/devel/docs)
- [Ceph-CSI CephFS 예제](https://github.com/ceph/ceph-csi/tree/devel/examples/cephfs)
- [RBD](https://docs.ceph.com/en/latest/rbd/)
- [RGW](https://docs.ceph.com/en/latest/radosgw/)

`latest`/`devel` 문서는 설치 버전과 차이가 있을 수 있습니다. 실제 버전과 지원 명령을 먼저 확인하세요.

- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
