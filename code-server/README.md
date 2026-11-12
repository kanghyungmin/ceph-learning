# code-server 배포

공개 예제입니다. 도메인 `vscode.example.com`, StorageClass, Gateway 이름과 namespace는 실제 환경에 맞게 수정하세요. 아래 배포 명령은 이 디렉터리에서 실행합니다. 기존 Secret과 Gateway가 있는지는 자신의 환경에서 확인해야 합니다.

현재 클러스터의 `qks-block` StorageClass와 `kube-system/qks-workload-gw` Gateway의 `https` listener를 사용합니다. Gateway의 TLS 인증서와 다른 네임스페이스의 Route 허용 설정은 이미 존재합니다.

## 배포

기존 `code-server/code-server-auth` Secret을 그대로 사용합니다. 비밀번호는 파일에 저장하지 않습니다.

```bash
kubectl apply -f resources.yaml
kubectl rollout status deployment/code-server -n code-server --timeout=180s
kubectl get pods,pvc,svc,httproute -n code-server
```

접속: https://vscode.example.com

새 클러스터에서 Secret이 없다면 먼저 아래 명령으로 생성하세요. 비밀번호 입력은 화면과 셸 히스토리에 남지 않습니다(zsh 기준).

```zsh
kubectl create namespace code-server --dry-run=client -o yaml | kubectl apply -f -
read -rs 'CODE_SERVER_PASSWORD?code-server 비밀번호: '; echo
printf '%s' "$CODE_SERVER_PASSWORD" | kubectl create secret generic code-server-auth \
  -n code-server --from-file=PASSWORD=/dev/stdin
unset CODE_SERVER_PASSWORD
```

## 반영한 수정

- 지원하지 않는 `--password` 인자를 제거하고 Secret의 `PASSWORD` 환경변수 사용.
- 공식 Docker 이미지 경로인 `codercom/code-server` 사용. 현재 클러스터에서 받은 이미지 digest로 고정.
- UID/GID 및 fsGroup을 1000으로 설정해 PVC 쓰기 권한 제공.
- `/home/coder` 전체를 20Gi PVC에 보관해 코드, 확장, 사용자 설정 유지.
- `Recreate` 전략으로 기존 Pod 종료 후 새 Pod 시작. 업데이트 중 잠시 중단됨.
- startup/readiness/liveness probe 추가.
- Gateway의 실제 HTTPS listener를 명시.

## 확인 및 문제 해결

```bash
kubectl logs -n code-server deployment/code-server --tail=100
kubectl describe pod -n code-server -l app=code-server
kubectl describe httproute -n code-server code-server
```

HTTPRoute의 `Accepted`와 `ResolvedRefs`가 True인지 확인하세요. Secret 비밀번호 변경 후에는 `kubectl rollout restart deployment/code-server -n code-server`가 필요합니다.

`fsGroup`은 기존 파일의 소유 UID까지 변경하지 않습니다. 기존 볼륨에서 권한 오류가 지속되면 파일 권한을 별도로 확인해야 합니다. PVC를 삭제하면 StorageClass의 Delete 정책에 따라 데이터도 삭제될 수 있습니다.

참고: https://coder.com/docs/code-server/install
