# 4주차 — CephFS와 Kubernetes 연결하기

[전체 계획](../README.md) · Day 16~20 · 10시간

## 시작 전

명령은 저장소 루트 `클론한 ceph-learning 디렉터리`에서 실행합니다. `kubectl`은 `k`로 줄여도 됩니다. `ceph` 명령은 인증된 전용 CLI에서 실행합니다. 인증 키와 Secret 원문은 학습 노트에 넣지 않습니다. 준비 방법과 현재 권한은 [전체 가이드](../README.md#준비-kubernetes와-ceph-cli)를 먼저 읽으세요.

매일 0~25분은 개념, 25~45분은 상태 확인과 실행 준비, 45~80분은 실습, 80~100분은 해석과 문제 조사, 100~120분은 질문 답변과 기록에 사용합니다. 실습이 일찍 끝나면 출력의 각 필드를 설명하고 재현 절차를 정리하세요.

이번 주 참고: [CephFS](https://docs.ceph.com/en/latest/cephfs/). 설치 버전과 문서 버전 차이를 확인하세요.

## Day 16 — CephFS 메타데이터와 데이터

### 0~25분: 개념 이해

CephFS는 디렉터리·파일명·권한 등 메타데이터와 실제 파일 데이터를 구분합니다. MDS는 메타데이터와 클라이언트 capability 조정을 담당하고, 클라이언트는 맵과 권한을 바탕으로 OSD와 데이터 I/O를 수행합니다.

### 25~45분: 실행 전 확인

qks-ceph의 active MDS와 standby 표시를 확인하세요. fs status의 표를 읽고 파일 생성/열기/내용 읽기에서 어떤 서비스가 관여하는지 순서도로 정리하세요.

### 45~80분: 실습

```bash
ceph fs status qks-ceph
ceph osd pool ls detail
```

MDS가 다루는 메타데이터와 OSD에 저장되는 데이터를 구분합니다. 파일 내용 전체가 MDS를 통과한다고 가정하지 않습니다.

완료 기준: 파일 열기와 데이터 읽기의 개념적 흐름 설명.

### 80~100분: 결과 해석

MDS 수가 파일 복제본 수는 아닙니다. active rank 수와 standby는 역할이 다릅니다. 실제 경로에는 캐시와 capability가 관여하므로 매 read마다 같은 네트워크 왕복이 생긴다고 가정하지 않습니다.

### 100~120분: 확인 질문과 기록

**질문:** MDS가 파일 내용 전체를 중계하나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

일반적인 CephFS 데이터 경로에서는 클라이언트가 OSD와 직접 데이터 I/O를 수행합니다.

</details>

`notes/day-16.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 17 — PVC와 subvolume 연결

### 0~25분: 개념 이해

CephFS subvolume은 파일시스템 내에서 CSI 볼륨을 구분·관리하는 단위입니다. 여러 PVC가 같은 CephFS에 속해도 각각 다른 subvolume을 사용할 수 있습니다. group은 관리상 묶음이며 별도 파일시스템과 같지 않습니다.

### 25~45분: 실행 전 확인

PVC UID, PV volumeHandle, volumeAttributes, subvolume 목록을 비교하세요. 명시적 식별 근거가 없다면 이름이 비슷한 후보만으로 확정하지 말고 미확인으로 표시하세요. CSI 로그가 보조 증거입니다.

### 45~80분: 실습

```bash
ceph fs subvolumegroup ls qks-ceph
ceph fs subvolume ls qks-ceph --group_name csi-example-cluster
```

Day 6의 PV 정보와 CSI 로그를 함께 사용해 실습 볼륨을 추적합니다. 이름만 보고 운영 subvolume을 삭제하거나 변경하지 않습니다.

완료 기준: PVC 하나가 CephFS 파일시스템 전체와 동일한 것이 아님을 설명.

### 80~100분: 결과 해석

목록에 여러 항목이 보이는 것은 다른 볼륨이 있다는 뜻일 수 있습니다. 학습용 항목이 어느 것인지 모르는 상태에서 info나 rm 대상으로 추정하지 않습니다.

### 100~120분: 확인 질문과 기록

**질문:** PVC 하나가 qks-ceph 전체를 소유하나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

아닙니다. 일반적인 CSI 동적 프로비저닝에서는 지정된 파일시스템의 subvolume을 사용합니다.

</details>

`notes/day-17.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 18 — 권한의 여러 계층

### 0~25분: 개념 이해

권한은 여러 층에 존재합니다. RBAC은 Kubernetes API, CephX는 Ceph 명령과 서비스 접근, POSIX는 파일·디렉터리 접근, securityContext는 프로세스 신원 등에 영향을 줍니다. 한 층의 성공은 다른 층 전체의 권한 증명이 아닙니다.

### 25~45분: 실행 전 확인

표의 행을 RBAC/CephX/POSIX/securityContext로 만들고 통제 대상·관측 명령·실제 확인 결과를 적으세요. 이전 EACCES와 Pod 파일 쓰기 성공을 서로 다른 행에 기록하세요.

### 45~80분: 실습

Kubernetes RBAC, CephX caps, CephFS POSIX 권한, Pod securityContext를 표로 비교합니다. 이전 확인에서 `ceph auth get`과 `auth ls`는 거부되었습니다. caps를 모르는 상태에서 자신의 쓰기 권한을 추측하지 않습니다.

완료 기준: `kubectl exec` 성공, `ceph -s` 성공, 파일 쓰기 성공이 각각 어떤 권한을 검증하는지 설명.

### 80~100분: 결과 해석

현재 계정은 여러 조회가 가능해도 auth 조회가 거부됩니다. 정확한 caps를 모르면 Pool 변경 가능 여부를 단정할 수 없습니다. caps 확인을 위해 변경 명령을 테스트하지 않습니다.

### 100~120분: 확인 질문과 기록

**질문:** Kubernetes 관리자면 Ceph 관리자도 자동으로 되나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

아닙니다. Ceph에 사용되는 신원과 caps를 별도로 확인해야 합니다.

</details>

`notes/day-18.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 19 — CSI controller와 nodeplugin

### 0~25분: 개념 이해

CSI는 표준 호출 경계이며 driver 구현과 여러 sidecar가 함께 동작합니다. controller가 Kubernetes 요청을 받아 백엔드 객체를 만들고 nodeplugin은 kubelet 요청으로 노드에 볼륨을 연결·마운트합니다.

### 25~45분: 실행 전 확인

Pod의 NODE를 확인하고 같은 노드의 ceph nodeplugin을 찾으세요. CSIDriver 객체의 fsGroupPolicy 등 실제 존재하는 필드를 읽고 뜻을 조사하세요. controller/sidecar 이름은 실제 배포에서 확인합니다.

### 45~80분: 실습

```bash
kubectl get pods -n kube-system -o wide | rg ceph
kubectl get csidriver cephfs.csi.ceph.com -o yaml
kubectl describe pod -n ceph-lab -l app=shared-reader
```

생성·확장 등 controller 동작과 노드의 mount 동작을 구분합니다. secret 값 대신 StorageClass의 참조 이름만 확인합니다.

완료 기준: PVC 생성 실패와 Pod 마운트 실패의 조사 경로를 따로 작성.

### 80~100분: 결과 해석

모든 CSI가 같은 attach 흐름을 요구하는 것은 아닙니다. CephFS에 블록 디바이스 attach 단계를 그대로 대입하지 않습니다. FailedMount와 provisioning 실패는 조사 위치가 다릅니다.

### 100~120분: 확인 질문과 기록

**질문:** nodeplugin 한 개가 모든 노드의 mount를 담당하나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

일반적으로 각 대상 노드의 nodeplugin이 해당 노드의 작업을 처리합니다.

</details>

`notes/day-19.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 20 — 스토리지 선택 비교

### 0~25분: 개념 이해

RWO/RWX는 접근 모드이고 블록/파일은 저장 인터페이스 분류입니다. 같은 층의 개념이 아닙니다. volumeMode Filesystem이라고 해서 반드시 CephFS인 것도 아닙니다. 블록 장치를 ext4로 포맷해 제공할 수도 있습니다.

### 25~45분: 실행 전 확인

기존 code-server YAML은 읽기만 하세요. 사용 목적, accessMode, CSI provisioner, 배포 전략, 데이터 이전 절차를 비교표로 작성하세요.

### 45~80분: 실습

code-server YAML을 읽고 RWO/RWX, 블록/파일 저장소, Immediate/WaitForFirstConsumer 차이를 표로 정리합니다. RWO는 일반적으로 단일 노드의 읽기·쓰기 마운트를 뜻하며 단일 Pod 제한과 같지 않습니다.

완료 기준: code-server를 공유 저장소로 옮길 때 새 PVC와 데이터 복사가 필요한 이유, RWX여도 애플리케이션 복제 안전성이 보장되지 않는 이유 설명. 실제 기존 PVC는 변경하지 않습니다.

### 80~100분: 결과 해석

qks-block은 KubeVirt CSI라는 사실만 확인되었습니다. RBD 여부는 상위 환경 근거가 추가로 필요합니다. RWX로 바꾸어도 VS Code 서버 다중 실행의 잠금·설정 충돌이 자동 해결되지는 않습니다.

### 100~120분: 확인 질문과 기록

**질문:** StorageClass 이름만 바꾸면 파일이 새 저장소로 옮겨지나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

아닙니다. 새 PVC를 만들고 데이터 복사·검증·전환 계획을 세워야 합니다.

</details>

`notes/day-20.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## 주말 복습

이번 주 완료 기준 중 설명이 막히는 두 항목을 다시 실행하세요. 결과가 달라졌다면 환경 변화와 실행 순서 차이를 먼저 조사합니다. 운영 설정 변경으로 결과를 맞추지 않습니다.
