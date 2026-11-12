# 0주차 — 프로비저닝과 Kubernetes 스토리지 인터페이스

[전체 계획](../README.md) · 사전 학습 2시간 · [다음: 1주차](week-01.md)

이 문서는 6주 본 과정 전에 읽는 사전 학습입니다. 별도로 진행하면 전체 학습 시간은 62시간입니다. 총 60시간을 유지하려면 첫 주의 보충 시간에 나누어 읽거나 Day 1의 개념 학습과 묶어 진행하세요.

목표는 **PVC 요청이 실제 저장소로 연결되는 과정**을 설명하는 것입니다. 이번에는 조회만 실행하며 프로비저닝 예제는 읽고 해석합니다. 실제 PVC 생성은 Day 2에서 진행합니다.

## 2시간 진행 순서

| 시간 | 학습 내용 | 남길 결과 |
|---|---|---|
| 0~25분 | StorageClass·PVC·PV·CSI 역할 | 객체별 역할 표 |
| 25~50분 | 동적·정적 프로비저닝 | 두 방식의 순서도 |
| 50~80분 | API·CSI RPC·파일 I/O 경로 | 관리 요청과 데이터 경로 구분 |
| 80~100분 | 현재 클러스터 읽기 전용 확인 | 실제 설정과 개념 연결 |
| 100~120분 | 확인 질문과 기록 | `notes/day-00.md` |

## 1. StorageClass·PVC·PV·CSI는 각각 무엇인가?

컨테이너가 `/data/hello.txt`에 파일을 쓰는 상황을 생각해 보세요. 애플리케이션은 파일 경로를 사용하지만, 그 경로 뒤의 저장소를 준비하는 역할은 여러 구성요소에 나뉘어 있습니다.

| 구성요소 | 역할 | 범위 | 예시 |
|---|---|---|---|
| StorageClass | 저장소를 어떤 드라이버·파라미터·정책으로 제공할지 정의 | 클러스터 | `qks-shared` |
| PVC | 사용자가 필요한 용량·접근 모드 등을 요청 | 네임스페이스 | `ceph-lab/shared-workspace` |
| PV | 할당된 실제 볼륨과 Kubernetes의 연결 정보를 표현 | 클러스터 | 동적으로 생성되는 `pvc-...` 형태 이름 등 |
| CSI | Kubernetes와 저장소 드라이버가 통신하는 표준 인터페이스 | API 객체가 아닌 인터페이스 규격 | `CreateVolume`, `NodePublishVolume` |
| CSI 드라이버 | CSI 규격을 구현하고 실제 저장소를 조작·연결 | controller/node 구성으로 배포 | `cephfs.csi.ceph.com` |
| Pod의 volumes | Pod가 사용할 PVC를 참조 | Pod 내부 선언 | `claimName: shared-workspace` |
| volumeMounts | 컨테이너의 어느 경로에 볼륨을 보일지 지정 | 컨테이너 내부 선언 | `mountPath: /data` |

PV 자체가 데이터 파일이나 디스크는 아닙니다. Kubernetes API에 저장되는 **볼륨 설명 객체**입니다. PVC도 데이터를 저장하지 않습니다. 실제 바이트는 Ceph 등의 백엔드에 저장됩니다.

또한 `CSIDriver`라는 Kubernetes 객체는 드라이버의 특성을 알리는 등록 정보입니다. CSI라는 규격, 실제 드라이버 프로세스, CSIDriver 객체는 서로 구분하세요.

### PVC가 요청하는 주요 조건

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-workspace
  namespace: ceph-lab
spec:
  storageClassName: qks-shared
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
```

- `storageClassName`: 어떤 종류의 저장소를 요청할지 선택합니다.
- `accessModes`: 필요한 마운트 접근 모드를 요청합니다. RWX는 여러 노드의 읽기·쓰기 사용에 해당합니다.
- `volumeMode`: Filesystem은 컨테이너에 파일시스템으로 제공합니다. Block이면 raw block 장치를 제공하며 컨테이너의 `volumeDevices`로 연결합니다.
- `requests.storage`: 필요한 용량을 요청합니다. 실제 데이터 사용량과 같은 값은 아닙니다.

`volumeMode: Filesystem`이라고 CephFS인 것은 아닙니다. RBD 같은 블록 장치 위에 ext4/xfs를 구성해 Filesystem으로 제공할 수도 있습니다. RWO/RWX는 블록/파일 분류와 별개의 축입니다.

## 2. 동적 프로비저닝

**사용자가 PVC를 요청하면 설정된 provisioner가 필요한 백엔드 볼륨을 만들고 PV 객체를 생성하는 방식**입니다. 사용자가 PVC별 PV YAML을 미리 작성할 필요가 없습니다.

개념적 순서는 다음과 같습니다. 조건에 맞는 기존 PV가 없는 일반적인 CSI 동적 생성 사례입니다.

```text
사용자: PVC 생성
    ↓ Kubernetes API에 요청 저장
external-provisioner: PVC와 StorageClass 관찰
    ↓ CSI CreateVolume RPC
CSI controller: 백엔드 볼륨 생성 요청
    ↓
CephFS: subvolume 생성
    ↓ volume 식별 정보 반환
external-provisioner: PV 객체 생성
    ↓ Kubernetes 바인딩 컨트롤러가 연결
PVC ↔ PV: Bound
```

`qks-shared`에서는 CephFS CSI가 지정된 파일시스템과 subvolume group 안에서 볼륨을 관리합니다. **PVC마다 Ceph 클러스터 전체나 새 물리 디스크를 설치하는 것은 아닙니다.**

external-provisioner는 Kubernetes API와 CSI RPC 사이를 이어 주는 sidecar입니다. 드라이버 자체와 역할이 다릅니다. 실제 백엔드 처리는 CSI controller가 담당합니다. [external-provisioner 공식 설명](https://kubernetes-csi.github.io/docs/external-provisioner.html)

### 언제 생성되는가?

| 바인딩 모드 | 동작 | 현재 예시 |
|---|---|---|
| Immediate | PVC 요청 시 바인딩/프로비저닝을 진행 | `qks-shared` |
| WaitForFirstConsumer | 소비 Pod의 스케줄링 요구를 고려할 수 있도록 대기 | `qks-block` |

따라서 Pending 상태가 항상 오류는 아닙니다. WaitForFirstConsumer는 소비 Pod를 기다리는 정상 상태일 수 있습니다.

### StorageClass를 생략하면?

`storageClassName`을 생략하면 기본 StorageClass가 적용될 수 있습니다. 현재 환경의 기본값은 반드시 `kubectl get sc`로 확인하세요. 앞선 환경에서는 `qks-block`이 기본값이므로 CephFS를 실습하려면 `qks-shared`를 명시하는 것이 좋습니다.

반면 **`storageClassName: ""`는 기본 Class를 선택하지 않으며 해당 PVC의 동적 프로비저닝을 비활성화**합니다. 생략과 빈 문자열은 다른 의미입니다. [Kubernetes PV 수명주기](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

## 3. 정적 프로비저닝

**관리자가 기존 또는 미리 준비한 실제 저장소를 PV 객체로 등록하고, PVC가 그 PV에 바인딩되는 방식**입니다.

```text
관리자: 실제 저장소와 접근 경로 준비
    ↓
관리자: PV 생성 — 볼륨 식별자, 용량, 드라이버, 접근 설정 등록
    ↓
사용자: 조건에 맞는 PVC 생성
    ↓
Kubernetes 바인딩: PVC ↔ 기존 PV
    ↓
CSI node 등: Pod에서 볼륨 사용 준비
```

정적 프로비저닝에서도 **CSI 드라이버를 사용할 수 있습니다.** 새 볼륨 생성을 자동화하는지와, 볼륨을 연결·마운트하는 인터페이스가 무엇인지는 별개의 문제입니다.

예를 들어 이미 존재하는 CephFS 경로를 CSI PV로 등록할 수 있지만, Ceph-CSI 버전에 맞는 static volume 설정, rootPath, 인증 범위와 식별자가 필요합니다. 임의의 운영 경로를 넣어 적용하는 실습은 하지 않습니다.

### 정적 PVC의 형태 읽기

다음은 **학습용 가상 예제**입니다. 실제 PV가 준비되지 않았으므로 적용하지 않습니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: existing-data
  namespace: ceph-lab
spec:
  storageClassName: ""
  volumeName: existing-cephfs-pv
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
```

이 예시는 관리자가 `existing-cephfs-pv`를 Class 없는 PV로 미리 등록했다는 전제입니다. `volumeName`은 특정 PV를 요청합니다. 실제 PV의 용량·Class·accessModes·volumeMode 및 기존 바인딩 상태 등도 맞아야 합니다. 이름만 지정했다고 조건 검사가 사라지지는 않습니다.

정적 PV에 StorageClass를 지정하는 구성도 가능합니다. 여기서 빈 Class를 쓴 것은 기본 Class와 동적 생성을 배제해 예제를 단순화하기 위한 선택입니다.

### 두 방식 비교

| 관점 | 동적 | 정적 |
|---|---|---|
| 백엔드 준비 | PVC 요청에 맞춰 자동 생성 | 관리자가 사전 준비 또는 기존 저장소 사용 |
| PV 등록 | provisioner가 생성 | 관리자가 작성 |
| 사용자 작업 | 보통 Class를 지정한 PVC 작성 | 준비된 PV 조건에 맞는 PVC 작성 |
| 주요 용도 | 반복적인 신규 볼륨 할당 | 기존 데이터 연결, 사전 통제된 볼륨 제공 |
| CSI 사용 | 가능 | 가능 |
| 삭제 정책 | 설정 확인 필요 | 설정 확인 필요 |

**정적이면 무조건 Retain, 동적이면 무조건 Delete인 것은 아닙니다.** 동적 PV는 일반적으로 StorageClass의 reclaimPolicy를 물려받습니다. 실제 적용 정책은 PV에서 확인합니다. Retain도 자동 백업이나 재사용 절차까지 제공하지는 않습니다.

## 4. “인터페이스 방식”을 세 층으로 구분하기

### 4-1. 사용자 ↔ Kubernetes: 선언형 API

사용자는 YAML을 `kubectl apply`로 제출합니다. API 서버에 원하는 상태가 저장되고 컨트롤러가 실제 상태를 맞춥니다. `kubectl apply`는 Ceph에 직접 파일을 만드는 명령이 아닙니다.

PVC/PV/StorageClass는 Kubernetes REST API로 읽고 쓰는 리소스입니다. PVC가 프로그램처럼 CSI에 RPC를 직접 보내는 것은 아닙니다.

### 4-2. Kubernetes 구성요소 ↔ CSI 드라이버: gRPC

CSI는 Protocol Buffers로 정의된 gRPC 인터페이스를 사용합니다. Kubernetes의 sidecar나 kubelet이 드라이버 endpoint에 요청하며, 일반적인 배포에서는 Unix domain socket을 사용합니다.

| 호출 계열 | 대표 메서드 | 역할 |
|---|---|---|
| Identity | GetPluginInfo, GetPluginCapabilities, Probe | 드라이버 정보·기능·상태 확인 |
| Controller | CreateVolume, DeleteVolume | 백엔드 볼륨 생성·삭제 |
| Controller | ControllerPublishVolume | 필요한 드라이버에서 노드에 attach 준비 |
| Controller | ControllerExpandVolume | 백엔드 볼륨 확장 |
| Node | NodeStageVolume | 지원하는 드라이버에서 노드의 staging 경로 준비 |
| Node | NodePublishVolume | Pod가 사용할 경로로 볼륨 제공 |
| Node | NodeUnpublishVolume, NodeUnstageVolume | Pod/노드 사용 경로 해제 |

모든 드라이버가 모든 메서드나 단계를 요구하는 것은 아닙니다. 특히 CephFS에 블록 디스크 attach 흐름을 그대로 대입하지 마세요. 세부 호출은 드라이버의 capability와 배포 구성에 따라 달라집니다. [CSI 규격](https://github.com/container-storage-interface/spec/blob/master/spec.md)

### 4-3. 애플리케이션 ↔ 실제 데이터: 파일 또는 블록 I/O

Pod에 마운트가 끝난 뒤 애플리케이션은 `open/read/write` 같은 파일 연산을 사용합니다. CephFS에서는 커널 CephFS 클라이언트 또는 FUSE 클라이언트가 백엔드와 통신합니다. MDS는 메타데이터와 capability에 관여하고 파일 데이터 I/O는 OSD와 이루어집니다.

```text
관리·연결 요청 경로
PVC → Kubernetes controller/sidecar → CSI RPC → CSI 드라이버 → Ceph
Pod 배치 → kubelet → CSI node RPC → 볼륨 사용 경로 준비

마운트 후 파일 데이터 경로
애플리케이션 → /data → CephFS 클라이언트 → OSD
                             ↕ 메타데이터·capability
                            MDS
```

일반적인 파일 read/write 바이트가 API 서버나 external-provisioner를 매번 경유하는 것은 아닙니다. CSI는 볼륨의 생성·연결·수명주기를 위한 인터페이스이고 데이터 경로 자체와 구분됩니다.

참고: [CephFS 구조](https://docs.ceph.com/en/latest/cephfs/).

## 5. Pod YAML에서 연결 경로 읽기

아래 발췌는 [실습 YAML](../labs/01-shared.yaml)의 관계를 설명합니다. 독립적으로 apply할 완전한 Pod 정의는 아닙니다.

```yaml
spec:
  containers:
    - name: reader
      # image와 command 등 다른 필드는 생략
      volumeMounts:
        - name: shared
          mountPath: /data
  volumes:
    - name: shared
      persistentVolumeClaim:
        claimName: shared-workspace
```

1. 컨테이너의 `volumeMounts.name: shared`가 Pod의 `volumes.name: shared`를 참조합니다.
2. `claimName: shared-workspace`가 **같은 네임스페이스**의 PVC를 참조합니다.
3. PVC의 `spec.volumeName`이 Bound 상태의 PV를 가리킵니다.
4. PV의 `spec.csi`에 드라이버와 volumeHandle 등 연결 정보가 있습니다.
5. 애플리케이션은 `/data` 경로를 사용합니다.

여러 Pod가 RWX PVC 하나를 사용할 수 있어도 PVC↔PV 바인딩은 기본적으로 일대일입니다. 공유한다고 PVC 하나에 PV가 여러 개 붙는 것은 아닙니다.

## 6. 읽기 전용 관찰 실습

저장소 루트에서 현재 컨텍스트를 먼저 확인합니다. 명령은 기본 kubeconfig를 사용합니다.

```bash
kubectl config current-context
kubectl get sc
kubectl get sc qks-shared -o yaml
kubectl get csidrivers
kubectl get pods -n kube-system -o wide | rg ceph
```

다음 항목을 노트에 채우세요.

| 확인 대상 | 기록할 필드 | 해석할 질문 |
|---|---|---|
| qks-shared | provisioner | 어느 CSI 드라이버가 담당하나? |
| qks-shared | parameters.fsName | 어떤 CephFS를 사용하나? |
| qks-shared | volumeBindingMode | 소비 Pod 없이 프로비저닝을 시작하나? |
| qks-shared | reclaimPolicy | PVC 삭제 후 데이터 수명에 어떤 영향이 있나? |
| qks-shared | allowVolumeExpansion | 확장을 요청할 수 있도록 설정되어 있나? |
| CSI Pod | controller와 nodeplugin의 위치 | 관리 요청과 노드 마운트 역할이 어떻게 나뉘나? |

기존 code-server PVC로 바인딩을 읽어 봅니다. 리소스를 변경하지 않습니다.

```bash
kubectl get pvc -n code-server
PV_NAME=$(kubectl get pvc code-server-workspace -n code-server -o jsonpath='{.spec.volumeName}')
# Bound 상태이고 PV_NAME이 비어 있지 않은 경우에만 다음 명령 실행
kubectl get pv "$PV_NAME" -o yaml
```

PVC가 없으면 이 부분은 건너뛰고 Day 2 이후 실습 PVC로 확인합니다. `qks-block`의 provisioner가 KubeVirt라는 사실만으로 Ceph RBD 백엔드를 확정하지 않습니다.

**해석 순서:** PVC 요청 → 실제 PV 바인딩 → PV의 CSI driver → 백엔드 연결 정보. 동적으로 생성된 PV의 annotation은 생성 경위를 판단하는 단서가 될 수 있지만 이름이나 annotation 하나로 모든 이력을 확정하지 않습니다.

## 7. 자주 생기는 오해

| 오해 | 정확한 구분 |
|---|---|
| PVC가 파일을 담고 있다 | PVC는 요청 객체이며 데이터는 백엔드에 존재 |
| PV는 언제나 디스크 하나다 | CephFS subvolume 등 구현에 따라 다름 |
| CSI를 설치하면 Ceph도 생긴다 | CSI 드라이버와 실제 Ceph 클러스터는 별도 |
| 정적 PV는 CSI를 사용하지 않는다 | 정적 CSI PV도 가능 |
| RWX면 여러 앱의 동시 쓰기가 안전하다 | 마운트 모드와 애플리케이션 정합성은 별개 |
| Bound면 앱에서 파일 쓰기가 반드시 성공한다 | 스케줄링·마운트·POSIX 권한·앱 경로 등 추가 확인 필요 |
| PVC 삭제는 이름만 지우는 일이다 | Delete 정책에서는 백엔드 데이터 삭제로 이어질 수 있음 |

## 8. 확인 질문과 해설

먼저 답을 기록한 뒤 해설을 읽으세요.

1. StorageClass와 PVC는 각각 무엇을 정하나요?
2. 동적 프로비저닝에서 PV 객체를 만드는 구성요소는 무엇인가요?
3. 정적 프로비저닝에서도 CSI를 사용할 수 있나요?
4. PVC를 생성했는데 Pending이면 어떤 정보를 먼저 확인하나요?
5. `storageClassName` 생략과 `""`는 어떻게 다른가요?
6. 파일을 읽을 때 CSI controller와 API 서버를 항상 거치나요?
7. Filesystem 볼륨이면 반드시 CephFS인가요?
8. PVC 하나를 Pod 두 개가 공유하면 PV도 두 개 필요한가요?

<details>
<summary>해설 보기</summary>

1. Class는 제공 방식과 정책, PVC는 필요한 용량·접근 모드·Class 등의 요청을 정합니다.
2. 일반적인 CSI 동적 프로비저닝에서는 external-provisioner가 드라이버의 CreateVolume 결과를 기반으로 PV를 만듭니다. 이후 바인딩 컨트롤러가 PVC와 연결합니다.
3. 가능합니다. 기존 볼륨을 가리키는 CSI PV를 등록하고 노드 사용 준비에 CSI를 이용할 수 있습니다.
4. PVC 이벤트, Class 존재, 바인딩 모드, 소비 Pod 유무를 먼저 확인합니다. Pending만으로 Ceph 장애라고 단정하지 않습니다.
5. 생략하면 기본 Class가 적용될 수 있고 빈 문자열은 기본 Class를 사용하지 않으며 동적 프로비저닝을 비활성화합니다.
6. 아닙니다. 마운트 후 실제 I/O는 클라이언트와 저장소 사이의 데이터 경로를 사용합니다.
7. 아닙니다. 블록 장치 위에 파일시스템을 만들어 제공할 수도 있습니다.
8. 아닙니다. PVC↔PV 바인딩은 일대일이고 여러 Pod가 해당 PVC를 사용할 수 있습니다. 접근 모드와 애플리케이션 지원을 별도로 확인합니다.

</details>

## 완료 기준과 다음 학습

- [ ] StorageClass·PVC·PV·CSI 드라이버를 각각 한 문장으로 설명한다.
- [ ] 동적·정적 프로비저닝의 백엔드 준비와 PV 생성 주체를 구분한다.
- [ ] Kubernetes API, CSI gRPC, 파일 데이터 I/O를 서로 다른 경로로 그린다.
- [ ] 현재 qks-shared 설정에서 개념에 대응되는 필드를 찾는다.
- [ ] PVC 삭제 전에 reclaimPolicy와 데이터 보존 필요성을 확인해야 하는 이유를 설명한다.

[노트 양식](../notes/TEMPLATE.md)을 `day-00.md`로 복사해 작성한 뒤 [1주차 Day 1](week-01.md)로 진행하세요.

## 공식 참고 자료

- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [동적 프로비저닝](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [CSI external-provisioner](https://kubernetes-csi.github.io/docs/external-provisioner.html)
- [CSI 인터페이스 규격](https://github.com/container-storage-interface/spec/blob/master/spec.md)
- [CephFS](https://docs.ceph.com/en/latest/cephfs/)
