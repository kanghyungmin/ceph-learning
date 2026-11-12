# 2주차 — PVC 수명주기와 CSI 문제 해결

[전체 계획](../README.md) · Day 6~10 · 10시간

## 시작 전

명령은 저장소 루트 `클론한 ceph-learning 디렉터리`에서 실행합니다. `kubectl`은 `k`로 줄여도 됩니다. `ceph` 명령은 인증된 전용 CLI에서 실행합니다. 인증 키와 Secret 원문은 학습 노트에 넣지 않습니다. 준비 방법과 현재 권한은 [전체 가이드](../README.md#준비-kubernetes와-ceph-cli)를 먼저 읽으세요.

매일 0~25분은 개념, 25~45분은 상태 확인과 실행 준비, 45~80분은 실습, 80~100분은 해석과 문제 조사, 100~120분은 질문 답변과 기록에 사용합니다. 실습이 일찍 끝나면 출력의 각 필드를 설명하고 재현 절차를 정리하세요.

이번 주 참고: [Kubernetes 스냅샷](https://kubernetes.io/docs/concepts/storage/volume-snapshots/). 설치 버전과 문서 버전 차이를 확인하세요.

## Day 06 — PVC에서 PV까지 추적

### 0~25분: 개념 이해

PVC는 어떤 PV에 바인딩되었는지 volumeName으로 가리킵니다. PV의 CSI 필드는 드라이버와 볼륨 식별자를 보관합니다. Kubernetes 객체의 UID와 CSI volumeHandle, Ceph subvolume 이름은 서로 다른 식별자입니다.

### 25~45분: 실행 전 확인

PVC의 UID·volumeName, PV의 claimRef·driver·volumeHandle을 표로 옮기세요. PV는 클러스터 객체이므로 -n이 필요 없는 점을 확인하세요.

### 45~80분: 실습

```bash
PV_NAME=$(kubectl get pvc shared-workspace -n ceph-lab -o jsonpath='{.spec.volumeName}')
kubectl get pv "$PV_NAME" -o yaml
kubectl describe pvc shared-workspace -n ceph-lab
```

`claimRef`, `csi.driver`, `volumeHandle`, `volumeAttributes`를 확인합니다. volumeHandle은 드라이버 식별자이므로 사람이 읽을 수 있는 subvolume 이름과 동일하다고 가정하지 않습니다.

완료 기준: Kubernetes 객체와 Ceph 내부 객체를 구분.

### 80~100분: 결과 해석

Bound 관계를 찾았다는 것과 Ceph의 실제 경로까지 찾았다는 것은 다릅니다. 아직 모르는 매핑은 추정 대신 미확인으로 기록하고 Day 17에서 이어갑니다.

### 100~120분: 확인 질문과 기록

**질문:** PVC와 PV의 이름이 같아야 하나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

아닙니다. 바인딩 관계는 이름 일치가 아니라 volumeName과 claimRef 등으로 확인합니다.

</details>

`notes/day-06.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 07 — 1Gi에서 2Gi로 확장

### 0~25분: 개념 이해

확장은 요청 변경, CSI controller의 백엔드 확장, 필요 시 노드 측 처리의 단계로 진행됩니다. CephFS는 블록 파일시스템과 구현이 다르므로 모든 스토리지에 동일한 동작을 가정하면 안 됩니다.

### 25~45분: 실행 전 확인

수정 전 요청량과 capacity를 기록하세요. YAML에서 PVC requests.storage만 2Gi로 바꾸고 적용하세요. spec.resources.requests와 status.capacity, conditions를 비교하세요.

### 45~80분: 실습

먼저 `labs/01-shared.yaml`의 요청 용량을 `2Gi`로 수정한 뒤 적용합니다. 원본 YAML에도 변경을 남겨 이후 재적용 시 축소 요청이 생기지 않게 합니다.

```bash
kubectl apply -f labs/01-shared.yaml
kubectl get pvc shared-workspace -n ceph-lab -w
# 용량 반영을 확인한 뒤 Ctrl-C
kubectl exec -n ceph-lab deployment/shared-reader -- df -h /data
kubectl describe pvc shared-workspace -n ceph-lab
```

완료 기준: 요청량·PVC capacity·Pod에서 보이는 용량을 비교. CephFS 클라이언트에서 보이는 값은 quota/클라이언트 설정에 영향을 받을 수 있으므로 df 하나만으로 성공을 판정하지 않습니다. PVC 축소는 하지 않습니다.

### 80~100분: 결과 해석

spec만 2Gi이고 status가 1Gi면 완료되지 않은 것입니다. 이벤트와 resizer 로그를 확인하세요. df는 quota와 클라이언트 설정에 따라 다를 수 있습니다. 재적용할 원본 파일을 1Gi로 되돌리지 마세요.

### 100~120분: 확인 질문과 기록

**질문:** 2Gi로 늘린 후 파일을 지우면 PVC도 자동으로 1Gi가 되나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

아닙니다. 사용량과 할당·요청 용량은 다르며 PVC 축소는 지원되는 일반 절차가 아닙니다.

</details>

`notes/day-07.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 08 — 의도적으로 Pending 만들고 진단

### 0~25분: 개념 이해

Pending은 아직 바인딩되지 않았다는 상태이며 원인 이름이 아닙니다. 존재하지 않는 StorageClass 요청은 CSI 장애와 구분할 수 있는 작은 재현 사례입니다.

### 25~45분: 실행 전 확인

보조 YAML을 읽고 실습 PVC 이름과 namespace를 확인하세요. 적용 시각을 적고 describe의 이벤트 Reason과 Message를 그대로 짧게 기록하세요. cleanup은 해당 미바인딩 PVC 하나만 대상으로 합니다.

### 45~80분: 실습

새 파일 `labs/02-pending.yaml`에 namespace가 `ceph-lab`, 이름이 `missing-class`, StorageClass가 `does-not-exist`, 용량이 `1Gi`, accessModes가 `ReadWriteMany`인 PVC를 작성합니다.

```bash
kubectl apply -f labs/02-pending.yaml
kubectl describe pvc missing-class -n ceph-lab
kubectl get events -n ceph-lab --sort-by=.metadata.creationTimestamp
kubectl delete pvc missing-class -n ceph-lab
```

완료 기준: 상태만 보고 판단하지 않고 이벤트 메시지로 원인을 설명. 기존 StorageClass와 CSI는 수정하지 않습니다.

보조 파일은 [02-pending.yaml](../labs/02-pending.yaml)에 제공되어 있습니다. 먼저 직접 작성해 보고 파일과 비교해도 됩니다. 동일 이름의 다른 PVC가 이미 있다면 덮어쓰지 말고 실습용 이름을 바꾸세요.

### 80~100분: 결과 해석

원인은 StorageClass 부재입니다. 운영 CSI 재시작이나 Ceph 설정 수정은 필요 없습니다. 이벤트 문구는 버전에 따라 달라질 수 있으며 이벤트가 누적되기까지 시간이 걸릴 수 있습니다.

### 100~120분: 확인 질문과 기록

**질문:** PVC가 Pending이면 Ceph 클러스터가 고장 난 것인가요?

<details>
<summary>생각한 뒤 해설 보기</summary>

아닙니다. Class 누락, 바인딩 대기, quota, 인증 등 여러 원인이 가능합니다.

</details>

`notes/day-08.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 09 — 스냅샷 가능 여부 판단

### 0~25분: 개념 이해

VolumeSnapshot은 네임스페이스의 요청 객체이고 VolumeSnapshotContent는 실제 스냅샷과의 연결을 표현합니다. SnapshotClass는 드라이버별 정책을 제공합니다. 복원은 스냅샷을 dataSource로 참조하는 새 PVC 생성입니다.

### 25~45분: 실행 전 확인

아래 추가 실습의 분기 순서를 따르세요. API와 적절한 Class가 없다면 새 controller나 Class를 임의 설치하지 말고 지원 조건 조사로 마칩니다. 사용할 Class의 deletionPolicy도 기록하세요.

### 45~80분: 실습

```bash
kubectl api-resources --api-group=snapshot.storage.k8s.io
kubectl get volumesnapshotclass
```

VolumeSnapshot API와 `driver: cephfs.csi.ceph.com`인 SnapshotClass가 있는지 확인합니다. CSI에 snapshotter가 있다는 것만으로 사용 가능하다고 판단하지 않습니다.

지원된다면 해당 Class를 사용하는 VolumeSnapshot YAML을 공식 예제로 작성해 실습 PVC만 스냅샷합니다. `readyToUse: true` 확인 후 새로운 이름의 PVC로 복원하고 파일을 비교합니다. 지원되지 않으면 객체 간 관계와 필요한 구성요소를 문서로 정리합니다. 애플리케이션 일관성은 별도 문제이므로 실습 동안 파일 쓰기를 멈춥니다.

완료 기준: VolumeSnapshot → VolumeSnapshotContent → 새 PVC의 관계 또는 지원되지 않는 구체적 이유 기록.

#### 지원되는 경우: 스냅샷과 복원을 끝까지 실행

아래는 파일을 셸에서 생성하는 명령입니다. 먼저 `kubectl get volumesnapshotclass -o yaml`에서 driver와 삭제 정책을 확인하고, **실제로 존재하는 CephFS SnapshotClass 이름**을 입력하세요. 이 명령은 SnapshotClass 자체를 생성하지 않습니다.

```bash
kubectl exec -n ceph-lab deployment/shared-reader -- sh -c 'echo snapshot-before > /data/snapshot-proof.txt; sync'
read -r 'SNAPSHOT_CLASS?CephFS VolumeSnapshotClass 이름: '
cat > labs/day09-snapshot.yaml <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: workspace-day09
  namespace: ceph-lab
spec:
  volumeSnapshotClassName: ${SNAPSHOT_CLASS}
  source:
    persistentVolumeClaimName: shared-workspace
EOF
kubectl apply -f labs/day09-snapshot.yaml
kubectl wait -n ceph-lab volumesnapshot/workspace-day09 --for=jsonpath='{.status.readyToUse}'=true --timeout=180s
kubectl get volumesnapshot workspace-day09 -n ceph-lab -o yaml
```

`read`는 Mac의 zsh 문법입니다. wait가 실패하면 다음 단계로 넘어가지 말고 describe로 error를 확인합니다. 원본 PVC는 Day 7에서 2Gi로 확장했다고 가정합니다. 복원 크기는 snapshot의 restoreSize 이상이어야 합니다.

```bash
cat > labs/day09-restore.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: workspace-restored
  namespace: ceph-lab
spec:
  storageClassName: qks-shared
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 2Gi
  dataSource:
    name: workspace-day09
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: restore-reader
  namespace: ceph-lab
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
    - name: reader
      image: busybox:1.37.0
      command: [sh, -c, 'while true; do sleep 3600; done']
      resources:
        requests: {cpu: 10m, memory: 16Mi}
        limits: {cpu: 100m, memory: 64Mi}
      volumeMounts:
        - name: restored
          mountPath: /data
  volumes:
    - name: restored
      persistentVolumeClaim:
        claimName: workspace-restored
EOF
kubectl apply -f labs/day09-restore.yaml
kubectl wait pod/restore-reader -n ceph-lab --for=condition=Ready --timeout=180s
kubectl exec -n ceph-lab restore-reader -- cat /data/snapshot-proof.txt
kubectl exec -n ceph-lab restore-reader -- cksum /data/snapshot-proof.txt
kubectl exec -n ceph-lab deployment/shared-reader -- cksum /data/snapshot-proof.txt
```

예상 결과는 복원본에 `snapshot-before`가 있고 두 checksum이 같은 것입니다. 이 단계를 마칠 때까지 원본 파일은 수정하지 않습니다. 타임아웃은 데이터 삭제로 해결하지 말고 snapshot/PVC/Pod 이벤트에서 조사합니다.

#### 지원되지 않는 경우: 75분 대체 실습

25분은 snapshot CRD/controller/sidecar/Class/백엔드 역할을 표로 정리하고, 25분은 공식 문서의 복원 YAML을 읽으며 필드 의미를 주석으로 달고, 25분은 위 YAML을 적용하지 않고 검토합니다. 기록에는 **실제 복원 미검증**이라고 명시합니다. 외부 지원 설치를 위해 운영 클러스터 구성을 바꾸지는 않습니다.

참고: [Kubernetes VolumeSnapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/).

### 80~100분: 결과 해석

readyToUse가 true여야 복원 단계로 갑니다. 스냅샷 전 파일과 복원된 파일의 내용 비교가 실제 검증입니다. readyToUse만으로 애플리케이션 일관성까지 검증되지는 않습니다.

### 100~120분: 확인 질문과 기록

**질문:** 스냅샷을 만들면 원본 PVC가 없어져도 항상 복구할 수 있나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

정책과 백엔드 동작에 달려 있습니다. 스냅샷 삭제 정책, 원본과의 의존성, 장애 범위를 확인해야 합니다.

</details>

`notes/day-09.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 10 — CSI 로그로 요청 추적

### 0~25분: 개념 이해

CSI controller 쪽은 CreateVolume/확장 같은 관리 요청을 처리하고 nodeplugin은 각 노드의 마운트 경로를 담당합니다. 여러 sidecar의 로그는 같은 사건을 다른 관점에서 보여줍니다.

### 25~45분: 실행 전 확인

학습 시작 시 시각을 적고 Day 7 확장 이후 로그가 남아 있는지 확인하세요. PVC UID 또는 volumeHandle이 포함된 줄을 찾되 자격 증명이나 운영 요청 전문을 노트에 복사하지 마세요.

### 45~80분: 실습

```bash
kubectl logs -n kube-system deployment/ceph-csi-cephfs-provisioner -c csi-cephfsplugin --since=15m --tail=200
kubectl logs -n kube-system deployment/ceph-csi-cephfs-provisioner -c csi-resizer --since=15m --tail=100
```

PVC UID·PV volumeHandle·타임스탬프를 바탕으로 실습 요청과 로그를 연결합니다. 로그가 없으면 시간 범위를 조정하되 전체 운영 로그를 대량 수집하지 않습니다.

완료 기준: Pending 진단 절차를 이벤트 → CSI controller → nodeplugin 순서로 정리.

### 80~100분: 결과 해석

과거 요청이 since=15m에 없다고 실패한 것이 아닙니다. 새 파일을 쓰는 것만으로 CreateVolume 로그가 생기지도 않습니다. 요청 종류와 관측 시간을 맞추세요.

### 100~120분: 확인 질문과 기록

**질문:** PVC가 Bound인데 Pod mount가 실패하면 controller 로그만 보면 충분할까요?

<details>
<summary>생각한 뒤 해설 보기</summary>

아닙니다. Pod 이벤트와 해당 노드의 nodeplugin을 함께 조사해야 합니다.

</details>

`notes/day-10.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## 주말 복습

이번 주 완료 기준 중 설명이 막히는 두 항목을 다시 실행하세요. 결과가 달라졌다면 환경 변화와 실행 순서 차이를 먼저 조사합니다. 운영 설정 변경으로 결과를 맞추지 않습니다.
