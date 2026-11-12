# 1주차 — 공유 스토리지를 직접 사용하기

[전체 계획](../README.md) · Day 1~5 · 10시간

먼저 [0주차: 동적·정적 프로비저닝과 CSI/PVC/PV](week-00.md)를 읽으면 이번 주 리소스 연결을 이해하기 쉽습니다.

## 시작 전

명령은 저장소 루트 `클론한 ceph-learning 디렉터리`에서 실행합니다. `kubectl`은 `k`로 줄여도 됩니다. `ceph` 명령은 인증된 전용 CLI에서 실행합니다. 인증 키와 Secret 원문은 학습 노트에 넣지 않습니다. 준비 방법과 현재 권한은 [전체 가이드](../README.md#준비-kubernetes와-ceph-cli)를 먼저 읽으세요.

매일 0~25분은 개념, 25~45분은 상태 확인과 실행 준비, 45~80분은 실습, 80~100분은 해석과 문제 조사, 100~120분은 질문 답변과 기록에 사용합니다. 실습이 일찍 끝나면 출력의 각 필드를 설명하고 재현 절차를 정리하세요.

이번 주 참고: [Kubernetes PV](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). 설치 버전과 문서 버전 차이를 확인하세요.

## Day 01 — 전체 구조와 현재 환경

### 0~25분: 개념 이해

Pod가 파일을 저장한다고 해서 항상 영속 저장소에 저장되는 것은 아닙니다. 컨테이너의 쓰기 계층은 컨테이너 교체 시 사라질 수 있습니다. PVC는 저장소를 요청하는 네임스페이스 객체이고 PV는 실제 할당을 표현하는 클러스터 객체입니다. StorageClass는 어떤 드라이버와 정책으로 이를 제공할지 정합니다.

### 25~45분: 실행 전 확인

SC 두 개의 provisioner와 바인딩 방식을 나란히 기록하세요. qks-shared의 parameters에서 fsName과 Secret 참조 이름을 찾으세요. Secret 값은 읽을 필요가 없습니다.

### 45~80분: 실습

개념: StorageClass는 저장소 제공 방식, PVC는 요청, PV는 할당된 볼륨, CSI는 Kubernetes와 스토리지 간 연결입니다.

```bash
kubectl get sc qks-shared -o yaml
kubectl get csidrivers
kubectl get pods -n kube-system | rg ceph
```

`PROVISIONER`, `reclaimPolicy`, `volumeBindingMode`, `allowVolumeExpansion`을 찾아 설명하세요. `qks-block`과 `qks-shared`의 차이를 기록합니다.

완료 기준: Pod → PVC → PV → CSI → CephFS 흐름을 말로 설명할 수 있습니다.

### 80~100분: 결과 해석

Immediate는 소비 Pod가 없어도 프로비저닝을 시작한다는 뜻입니다. WaitForFirstConsumer는 스케줄링 제약을 고려할 수 있도록 소비자를 기다립니다. 어느 쪽도 실제 파일 공유 가능 여부를 단독으로 결정하지 않습니다.

### 100~120분: 확인 질문과 기록

**질문:** PVC를 만들면 언제나 새 디스크가 생길까요?

<details>
<summary>생각한 뒤 해설 보기</summary>

아닙니다. 구현에 따라 CephFS subvolume, RBD image 등 서로 다른 백엔드 객체가 만들어질 수 있습니다.

</details>

`notes/day-01.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 02 — 1Gi RWX PVC와 Pod 두 개 생성

### 0~25분: 개념 이해

이 실습에서 Deployment는 두 Pod를 유지하고 두 Pod는 하나의 RWX PVC를 마운트합니다. Pod를 두 개 만든 것과 실제로 서로 다른 노드에서 공유한 것은 다른 검증입니다. required anti-affinity가 노드 분산을 강제합니다.

### 25~45분: 실행 전 확인

적용 전 YAML에서 claimName, mountPath, replicas, selector, securityContext를 찾으세요. 적용 후 PVC Bound와 Pod Running/Ready를 따로 확인하고 NODE 열을 비교하세요.

### 45~80분: 실습

저장소 루트에서 실행합니다. 이 명령부터 실제 실습 리소스를 생성합니다.

```bash
kubectl apply -f labs/01-shared.yaml
kubectl rollout status deployment/shared-reader -n ceph-lab --timeout=180s
kubectl get pods,pvc -n ceph-lab -o wide
```

Pod는 anti-affinity로 서로 다른 노드에 배치됩니다. 두 노드 환경에서도 재시작이 진행되도록 Recreate 전략을 사용하며, 재시작 중에는 두 Pod가 잠시 중단됩니다. 사용 가능한 노드가 2개 미만이면 한 Pod는 Pending이 됩니다. 이는 스토리지 오류가 아닐 수 있으므로 `kubectl describe pod`의 스케줄링 이벤트를 확인하세요.

완료 기준: PVC Bound, Pod 두 개 Ready, 서로 다른 노드에서 같은 PVC 사용.

### 80~100분: 결과 해석

PVC Bound여도 Pod는 이미지 다운로드나 스케줄링 문제로 시작하지 못할 수 있습니다. rollout 타임아웃이 나면 재적용을 반복하기 전에 describe의 Events를 읽으세요.

### 100~120분: 확인 질문과 기록

**질문:** 왜 Deployment와 Service 없이도 파일 공유를 확인할 수 있나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

파일 공유는 PVC와 CSI 마운트 경로로 이루어집니다. Deployment는 Pod 관리를 돕고 Service는 네트워크 서비스 노출에 쓰입니다. 이 실습에는 Service가 필요 없습니다.

</details>

`notes/day-02.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 03 — 서로 다른 노드에서 파일 공유

### 0~25분: 개념 이해

CephFS는 두 Pod에 같은 파일시스템 영역을 제공합니다. A의 파일을 B가 읽는 것은 파일을 Pod 사이에 복사해서가 아니라 같은 저장소를 보기 때문입니다. 단, 동시 수정의 의미는 애플리케이션의 파일 잠금과 쓰기 방식에 달려 있습니다.

### 25~45분: 실행 전 확인

Pod가 두 개 모두 Ready인지 확인한 뒤 변수를 설정하세요. A가 쓴 뒤 B에서 읽고 B가 추가한 뒤 A에서 읽는 순서를 지키세요. 마지막 출력 두 줄을 기록하세요.

### 45~80분: 실습

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

### 80~100분: 결과 해석

예상 출력은 hello-ceph와 from-pod-b입니다. 안 보이면 Pod 이름, namespace, claimName, mountPath를 먼저 비교하세요. sync는 대기 중인 쓰기의 반영을 요청하며 백업을 만드는 명령은 아닙니다.

### 100~120분: 확인 질문과 기록

**질문:** RWX PVC를 붙이면 데이터베이스 프로세스 두 개가 같은 파일을 안전하게 수정할까요?

<details>
<summary>생각한 뒤 해설 보기</summary>

보장되지 않습니다. 애플리케이션이 그 공유·동시성 모델을 지원해야 합니다.

</details>

`notes/day-03.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 04 — Pod 수명과 데이터 수명

### 0~25분: 개념 이해

Deployment의 Pod는 교체 가능한 실행 단위이고 PVC는 별도 객체입니다. Pod가 사라져도 PVC가 남으면 새 Pod가 동일한 저장소에 다시 접근할 수 있습니다. 다른 노드로 재배치되는지는 이번 재시작만으로 보장되지 않습니다.

### 25~45분: 실행 전 확인

재시작 전 Pod 이름과 PVC의 volumeName을 기록하세요. 재시작 후 이름을 비교하고 파일 내용과 volumeName이 유지되는지 확인하세요. Day 3의 이전 Pod 변수는 재사용하지 마세요.

### 45~80분: 실습

```bash
kubectl rollout restart deployment/shared-reader -n ceph-lab
kubectl rollout status deployment/shared-reader -n ceph-lab --timeout=180s
kubectl exec -n ceph-lab deployment/shared-reader -- cat /data/day03.txt
kubectl get pvc -n ceph-lab
```

완료 기준: Pod가 교체되어도 파일이 남는 이유와 PVC를 삭제했을 때의 차이를 설명합니다.

### 80~100분: 결과 해석

Recreate 동안 Pod가 일시적으로 0개일 수 있습니다. rollout이 끝난 뒤 파일이 유지되면 실행 수명과 저장소 수명을 구분한 것입니다. 이것은 스토리지 전체 장애 복구 검증은 아닙니다.

### 100~120분: 확인 질문과 기록

**질문:** Pod 재시작과 PVC 삭제가 왜 다른가요?

<details>
<summary>생각한 뒤 해설 보기</summary>

Pod 재시작은 저장소 요청을 유지합니다. PVC 삭제는 Delete 정책에서 실제 저장소 삭제로 이어질 수 있습니다.

</details>

`notes/day-04.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 05 — 권한과 주간 복습

### 0~25분: 개념 이해

리눅스 파일 권한은 사용자 UID, 그룹 GID, 소유자·그룹·기타의 mode 비트로 판단합니다. runAsUser/runAsGroup은 프로세스 신원이고 fsGroup은 볼륨 접근을 위한 그룹 설정에 관여합니다. 실제 적용은 CSI 동작에도 영향을 받습니다.

### 25~45분: 실행 전 확인

id의 uid/gid/groups와 ls -n의 숫자를 대조하세요. 실습 파일에만 chmod 600을 적용하고 읽기 결과를 관찰한 뒤 chmod 644로 되돌리세요. 공유 디렉터리 전체에 재귀 chmod/chown은 하지 마세요.

### 45~80분: 실습

```bash
kubectl exec -n ceph-lab deployment/shared-reader -- id
kubectl exec -n ceph-lab deployment/shared-reader -- ls -ldn /data
kubectl exec -n ceph-lab deployment/shared-reader -- sh -c 'umask; touch /data/permissions.txt; ls -ln /data/permissions.txt'
```

UID, GID, fsGroup, 파일 mode를 비교하세요. fsGroup은 모든 기존 파일의 소유 UID를 바꾸는 기능이 아닙니다.

완료 기준: YAML을 보지 않고 PVC와 Pod 관계를 설명하고 1주차 결과를 한 페이지로 정리.

### 80~100분: 결과 해석

동일 UID의 두 Pod가 파일을 읽는다고 그룹 권한까지 검증한 것은 아닙니다. umask는 새 파일의 기본 권한에서 비트를 제거하며 기존 파일 권한을 바꾸지 않습니다.

### 100~120분: 확인 질문과 기록

**질문:** 쓰기 오류를 해결하려고 항상 root로 실행하면 될까요?

<details>
<summary>생각한 뒤 해설 보기</summary>

원인 계층을 숨길 수 있습니다. 프로세스 신원, 디렉터리 권한, 마운트 및 Ceph 권한을 구분해 확인해야 합니다.

</details>

`notes/day-05.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## 주말 복습

이번 주 완료 기준 중 설명이 막히는 두 항목을 다시 실행하세요. 결과가 달라졌다면 환경 변화와 실행 순서 차이를 먼저 조사합니다. 운영 설정 변경으로 결과를 맞추지 않습니다.
