# 5주차 — 운영 관찰과 진단 능력

[전체 계획](../README.md) · Day 21~25 · 10시간

## 시작 전

명령은 저장소 루트 `클론한 ceph-learning 디렉터리`에서 실행합니다. `kubectl`은 `k`로 줄여도 됩니다. `ceph` 명령은 인증된 전용 CLI에서 실행합니다. 인증 키와 Secret 원문은 학습 노트에 넣지 않습니다. 준비 방법과 현재 권한은 [전체 가이드](../README.md#준비-kubernetes와-ceph-cli)를 먼저 읽으세요.

매일 0~25분은 개념, 25~45분은 상태 확인과 실행 준비, 45~80분은 실습, 80~100분은 해석과 문제 조사, 100~120분은 질문 답변과 기록에 사용합니다. 실습이 일찍 끝나면 출력의 각 필드를 설명하고 재현 절차를 정리하세요.

이번 주 참고: [Ceph 운영](https://docs.ceph.com/en/latest/rados/operations/). 설치 버전과 문서 버전 차이를 확인하세요.

## Day 21 — 3회 관측으로 변화 읽기

### 0~25분: 개념 이해

측정값은 시각과 맥락이 있어야 비교할 수 있습니다. 용량은 시점의 상태이고 I/O rate는 구간 또는 최근 관측을 요약합니다. 하나의 status 결과로 장기간 추세를 판단할 수 없습니다.

### 25~45분: 실행 전 확인

시작·약 40분 뒤·약 80분 뒤에 같은 명령을 실행하세요. 시간 사이에는 출력 해석과 문서 읽기를 진행하세요. 시각·raw used·stored·PG 상태·I/O rate를 표로 적으세요.

### 45~80분: 실습

실습 시작·중간·종료에 `ceph -s`, `ceph df`, `ceph pg stat`을 실행해 시각과 함께 기록합니다. 관측치가 변해도 자신의 실습 때문이라고 단정하지 않습니다.

완료 기준: 누적 용량과 순간 I/O 지표를 구분하고 관측의 한계를 적기.

### 80~100분: 결과 해석

공유 클러스터의 변화에는 다른 워크로드가 섞입니다. 자신의 작은 파일 실습과 전체 I/O 변화가 동시에 발생했다고 인과관계로 쓰지 않습니다. 값이 변하지 않아도 유효한 관측입니다.

### 100~120분: 확인 질문과 기록

**질문:** 세 번 측정으로 장기 용량 증가율을 확정할 수 있나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

관측 간격과 기간이 부족합니다. 단기 예시이며 운영 추세에는 더 긴 수집이 필요합니다.

</details>

`notes/day-21.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 22 — 작은 파일 I/O와 메타데이터

### 0~25분: 개념 이해

파일 내용 쓰기와 파일명·디렉터리 항목 생성은 다른 종류의 작업입니다. dd의 짧은 실행시간은 캐시, 파일 크기, 네트워크와 다른 부하의 영향을 받으므로 장치 최대 성능과 동일하지 않습니다.

### 25~45분: 실행 전 확인

한 Pod에서 1MiB 파일을 쓰고 다른 Pod에서 cksum과 wc -c를 확인하세요. 작은 텍스트 파일 한 개를 추가 생성하고 ls -l로 메타데이터를 관찰하세요. 부하 테스트를 위해 반복 횟수를 늘리지 않습니다.

### 45~80분: 실습

실습 PVC에서만 1MiB 파일 하나를 쓰고 읽습니다. 성능 벤치마크가 아니라 I/O 경로 확인입니다.

```bash
kubectl exec -n ceph-lab deployment/shared-reader -- sh -c 'dd if=/dev/zero of=/data/io-sample.bin bs=1024 count=1024; sync; cksum /data/io-sample.bin'
```

다른 Pod에서도 cksum을 비교합니다. 캐시·부하·파일 크기를 통제하지 않았으므로 이 결과로 Ceph 성능을 평가하지 않습니다.

완료 기준: 데이터 I/O와 파일 생성·목록 같은 메타데이터 작업의 차이 설명.

추가 확인:

```bash
kubectl get pods -n ceph-lab -l app=shared-reader -o wide
POD_A=$(kubectl get pods -n ceph-lab -l app=shared-reader -o jsonpath='{.items[0].metadata.name}')
POD_B=$(kubectl get pods -n ceph-lab -l app=shared-reader -o jsonpath='{.items[1].metadata.name}')
kubectl exec -n ceph-lab "$POD_A" -- cksum /data/io-sample.bin
kubectl exec -n ceph-lab "$POD_B" -- cksum /data/io-sample.bin
kubectl exec -n ceph-lab "$POD_B" -- wc -c /data/io-sample.bin
```

Deployment 지정 exec는 어느 한 Pod를 선택하므로, 반대편 검증은 위처럼 Pod 이름을 명시합니다.

### 80~100분: 결과 해석

예상 바이트 수는 1048576입니다. 양쪽 cksum이 같으면 그 시점의 파일 내용이 같은지 확인한 것입니다. cksum은 암호학적 무결성이나 저장소 내구성 검증을 대신하지 않습니다.

### 100~120분: 확인 질문과 기록

**질문:** dd 출력이 빨랐으니 운영 DB도 빠르다고 결론 내릴 수 있나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

아닙니다. 접근 패턴과 동기 쓰기, 병렬도, 캐시 등 조건이 다릅니다.

</details>

`notes/day-22.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 23 — 장애 진단 표 만들기

### 0~25분: 개념 이해

같은 증상에도 서로 다른 원인이 있습니다. 진단은 오류가 발생한 층을 좁히는 과정입니다. 운영 변경 전에 관측 사실과 가설을 분리하면 불필요한 재시작을 줄일 수 있습니다.

### 25~45분: 실행 전 확인

아래 진단 표를 노트에 복사하고 각 행에 확인 명령과 실제 또는 가상의 증거를 추가하세요. 실제로 재현하는 오류는 존재하지 않는 Class를 사용하는 작은 PVC로 제한합니다.

### 45~80분: 실습

다음 증상마다 확인 명령과 가능한 원인을 2개 이상 작성합니다: PVC Pending, Pod Pending, ContainerCreating, Permission denied, 용량 부족, 파일이 안 보임.

완료 기준: 증상 → Kubernetes 이벤트 → CSI → Ceph 상태 → 애플리케이션 순서의 조사표 작성. 장애 재현은 Day 8의 실습 PVC 범위로 제한.

#### 진단표 시작점

| 증상 | 첫 확인 | 가능한 원인 | 다음 증거 |
|---|---|---|---|
| PVC Pending | describe pvc, get sc | Class 누락, 프로비저닝 실패, 바인딩 대기 | PVC 이벤트와 provisioner 로그 |
| Pod Pending | describe pod | anti-affinity, 자원 부족, 미바인딩 PVC | Scheduling 이벤트 |
| ContainerCreating | describe pod | FailedMount, 이미지 다운로드 | Pod 이벤트와 해당 노드 nodeplugin |
| Permission denied | exec id, ls -ldn | UID/GID 불일치, 디렉터리 mode, Ceph 권한 | 실패한 경로와 관련 로그 |
| 용량 부족 | PVC spec/status, df, ceph df | subvolume quota, 전체/Pool 공간 문제 | 확장 이벤트와 Ceph health |
| 파일 안 보임 | Pod PVC 참조와 mountPath | 다른 PVC, 다른 경로, 쓰기 실패 | PV 매핑과 양쪽 파일 목록 |

`describe pod -n ceph-lab <실제 Pod 이름>`처럼 대상 이름을 확인한 뒤 실행합니다. 실패 명령의 원문과 종료 상태도 함께 기록하세요.

### 80~100분: 결과 해석

운영 중인 CSI나 MON을 재시작하는 것은 원인 조사 시작점이 아닙니다. 이벤트에 원인이 명확하면 더 넓은 구성요소를 건드리지 않고 실습 리소스만 수정합니다.

### 100~120분: 확인 질문과 기록

**질문:** 로그에 에러가 있으면 그것이 현재 PVC의 원인인가요?

<details>
<summary>생각한 뒤 해설 보기</summary>

시간·PVC UID·volumeHandle 등으로 관련성을 확인해야 합니다.

</details>

`notes/day-23.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 24 — 운영 구성과 orchestrator

### 0~25분: 개념 이해

orchestrator는 Ceph 데몬 배치와 수명주기 관리에 관여합니다. CSI는 애플리케이션 볼륨 요청과 연동합니다. config dump는 설정 DB의 항목을 보여주며 데몬에 실제 적용된 모든 유효값을 한 번에 증명하는 것은 아닙니다.

### 25~45분: 실행 전 확인

orch status에서 backend와 가용 여부를 적으세요. config dump에서 범위가 global인지 특정 daemon인지 구분해 항목 세 개만 조사하세요. 설정값이나 주소를 공개 노트에 그대로 옮길 필요는 없습니다.

### 45~80분: 실습

```bash
ceph orch status
ceph config dump
```

orchestrator의 역할과 설정 계층을 읽습니다. config dump는 기본값까지 전부 보여주는 명령이 아님을 확인합니다. 설정 변경과 서비스 재배포는 하지 않습니다.

완료 기준: CSI가 Ceph 클러스터 자체를 설치·운영하는 도구와 다른 이유 설명.

### 80~100분: 결과 해석

명령이 성공했다는 사실은 조회 권한만 검증합니다. orchestrator로 서비스 배포나 제거가 가능하다는 뜻이 아닙니다. 값이 목록에 없다고 기본값 자체가 없다는 뜻도 아닙니다.

### 100~120분: 확인 질문과 기록

**질문:** config dump와 실제 daemon의 유효 설정은 항상 같나요?

<details>
<summary>생각한 뒤 해설 보기</summary>

설정 계층과 런타임 적용 등에 따라 별도 확인이 필요합니다.

</details>

`notes/day-24.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## Day 25 — 복구·백업과 삭제 정책

### 0~25분: 개념 이해

백업은 복원 가능성을 검증해야 가치가 있습니다. 스냅샷과 원본이 같은 저장 시스템에 있으면 공통 장애를 공유할 수 있습니다. RPO는 허용 데이터 손실 시점, RTO는 복구 목표 시간입니다.

### 25~45분: 실행 전 확인

Day 9 성공 시 복원 파일과 원본의 checksum을 다시 비교하세요. 미지원이면 아래 로컬 복사 실습을 사용하세요. 복사 시각과 복원 검증 시간을 기록하세요.

### 45~80분: 실습

Day 9 복원 결과를 복습합니다. 스냅샷이 없다면 실습 텍스트 파일 하나를 로컬 임시 폴더로 복사해 원본과 cksum을 비교합니다. 스냅샷은 같은 클러스터의 장애까지 보호하는 독립 백업과 같지 않습니다.

완료 기준: Pod 삭제, PVC 삭제, Ceph 클러스터 장애 각각에서 필요한 복구 수단 정리. 현재 StorageClass가 Delete임을 다시 확인.

#### 스냅샷 미지원 시: 파일 백업과 새 이름으로 복원

로컬에서 아래를 실행합니다. kubectl cp의 tar 의존성을 피하기 위해 단일 텍스트 파일을 stdout으로 읽습니다.

```bash
BACKUP_DIR=$(mktemp -d /tmp/ceph-lab-backup.XXXXXX)
kubectl exec -n ceph-lab deployment/shared-reader -- sh -c 'echo backup-proof > /data/backup-proof.txt; sync'
kubectl exec -n ceph-lab deployment/shared-reader -- cat /data/backup-proof.txt > "$BACKUP_DIR/backup-proof.txt"
cksum "$BACKUP_DIR/backup-proof.txt"
kubectl exec -i -n ceph-lab deployment/shared-reader -- sh -c 'cat > /data/backup-restored.txt' < "$BACKUP_DIR/backup-proof.txt"
kubectl exec -n ceph-lab deployment/shared-reader -- cksum /data/backup-proof.txt /data/backup-restored.txt
```

두 파일의 checksum과 바이트 수가 같은지 비교합니다. 파일 이름은 달라도 됩니다. 각 명령이 성공했는지 확인한 뒤 다음 줄을 실행하세요. exec 실패 시 로컬 파일이 비어 있을 수 있으므로 로컬 파일 존재만으로 성공을 판정하지 않습니다. 원본을 삭제하지 않는 비파괴 복원 검증입니다.

### 80~100분: 결과 해석

텍스트 파일 하나의 복사는 개념 실습입니다. DB 백업, 권한·확장 속성 보존, 전체 디렉터리 일관성을 검증한 것은 아닙니다. 실습 파일 삭제 없이 복원본을 새 이름으로 검증하세요.

### 100~120분: 확인 질문과 기록

**질문:** 스냅샷이 10개면 백업은 충분한가요?

<details>
<summary>생각한 뒤 해설 보기</summary>

개수보다 장애 범위, 보관 정책, 외부 사본과 실제 복원 검증이 중요합니다.

</details>

`notes/day-25.md`에 관측 시각, 핵심 결과, 해석, 남은 질문을 기록하세요. 실습 본문의 완료 기준을 충족하지 못했다면 다음 날 시작 전 보완할 항목 한 개를 정합니다.

## 주말 복습

이번 주 완료 기준 중 설명이 막히는 두 항목을 다시 실행하세요. 결과가 달라졌다면 환경 변화와 실행 순서 차이를 먼저 조사합니다. 운영 설정 변경으로 결과를 맞추지 않습니다.
