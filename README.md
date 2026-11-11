# homelab-gitops

Argo CD로 자기소개 웹 앱과 Cloudflare Tunnel Connector를 배포하는 Kubernetes GitOps 저장소입니다.

```text
argocd/
├── self-Intruduction.yaml       # 웹 앱 Argo CD Application
├── traefik.yaml                  # Traefik Helm chart Argo CD Application
└── cloudflared.yaml              # Tunnel Connector Argo CD Application
apps/self-Intruduction/
├── deployment.yaml               # 웹 앱 Deployment
├── service.yaml                  # 내부 ClusterIP Service
└── ingress.yaml                  # intro.woonapp.com → 웹 앱
infrastructure/cloudflared/
├── kustomization.yaml
└── deployment.yaml               # 원격 관리 Tunnel Connector
```

## 트래픽 흐름

```text
브라우저 HTTPS
  → Cloudflare DNS / Tunnel
  → cloudflared (Kubernetes)
  → Traefik Service (HTTP, 클러스터 내부)
  → Traefik Ingress: intro.woonapp.com
  → self-introduction Service:8080
  → Spring Boot Pod
```

앱 Service는 `ClusterIP`로만 노출됩니다. Tunnel Connector 토큰은 Git에 저장하지 않고, `cloudflare` 네임스페이스의 `cloudflared-token` Secret으로 별도 생성합니다. Cloudflare Dashboard의 Tunnel 및 Published Application 설정도 이 저장소에서 관리하지 않습니다.

## 최초 연결 순서

### 1. Argo CD 설치

클러스터에는 현재 Argo CD가 없으므로 최초 1회 홈서버 터미널에서 설치합니다. 비고가용성(single replica) 공식 매니페스트를 버전 `v3.6.1`로 고정합니다.

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=5m
```

Argo CD 서비스는 기본 ClusterIP로 유지합니다. 저장소가 비공개라면 Argo CD에 GitHub read-only 저장소 인증을 별도로 등록해야 합니다.

### 2. Cloudflare Tunnel 사용

Cloudflare Dashboard의 `Networking → Tunnels`에 원격 관리 Tunnel `homeServer-tunnel`을 생성했습니다. Connector 두 개가 Kubernetes에서 실행 중이며 Cloudflare에 연결된 상태입니다. 토큰은 `cloudflare` 네임스페이스의 `cloudflared-token` Secret에만 저장하고 Git에는 넣지 않습니다. Docker 안내 명령은 로컬에서 실행하지 않습니다.

### 3. Traefik 등록 및 실제 내부 주소 확인

저장소 변경 사항을 `main`에 반영한 뒤 Traefik Application을 등록합니다. Argo CD가 공식 Helm chart `42.3.1`을 `traefik` 네임스페이스에 설치하고, Service를 `ClusterIP`로 유지합니다.

```bash
kubectl apply -f argocd/traefik.yaml
kubectl rollout status deployment/traefik -n traefik --timeout=5m
kubectl get ingressclass
kubectl get svc -n traefik
```

### 4. 홈서버의 실제 Traefik 주소 확인

홈서버에 연결된 VS Code 터미널에서 확인합니다.

```bash
kubectl get nodes
kubectl get ingressclass
kubectl get svc -A | grep -i traefik
```

IngressClass가 `traefik`인지 확인합니다. Tunnel Published Application의 Service URL에는 조회 결과에 맞춰 다음 형식의 Traefik 내부 주소를 사용합니다.

```text
http://<service-name>.<namespace>.svc.cluster.local:<service-port>
```

예를 들어 Service가 `traefik` 네임스페이스의 `traefik`이고 포트가 80이면 `http://traefik.traefik.svc.cluster.local:80`입니다. 이 예시를 실제 조회 없이 그대로 사용하지 않습니다.

### 5. 토큰을 Kubernetes Secret으로 저장

토큰을 교체해야 하면 홈서버 터미널에서 아래 명령을 실행하고, 프롬프트에 Tunnel 토큰을 붙여 넣습니다. 입력값은 화면에 표시되지 않고 Git에도 기록되지 않습니다. Secret을 갱신한 뒤 Connector Pod를 재시작합니다.

```bash
kubectl create namespace cloudflare --dry-run=client -o yaml | kubectl apply -f -
read -rsp "Cloudflare Tunnel token: " TUNNEL_TOKEN
echo
printf '%s' "$TUNNEL_TOKEN" | kubectl create secret generic cloudflared-token \
  --namespace cloudflare \
  --from-file=token=/dev/stdin \
  --dry-run=client -o yaml | kubectl apply -f -
unset TUNNEL_TOKEN
kubectl rollout restart deployment/cloudflared -n cloudflare
```

Kubernetes Secret은 기본적으로 etcd에서 암호화되지 않을 수 있으므로 클러스터의 etcd 암호화와 RBAC 접근 제한을 확인합니다.

### 6. 앱을 Argo CD에 등록

저장소 변경 사항을 `main`에 반영한 다음 홈서버 터미널에서 새 Connector Application을 등록합니다. 웹 앱 Application이 아직 등록되지 않았다면 함께 적용합니다.

```bash
kubectl apply -f argocd/cloudflared.yaml
kubectl apply -f argocd/self-Intruduction.yaml
```

세 Application(Traefik, Connector, 웹 앱)은 자동 동기화됩니다. Connector Deployment는 `cloudflare` 네임스페이스의 Secret을 읽고 `cloudflare/cloudflared:2026.10.7`을 2개 고정 replica로 실행합니다. `/ready` probe가 Cloudflare 연결 상태를 확인하며, 자동 스케일링은 사용하지 않습니다.

### 7. 공개 호스트 연결

Cloudflare Dashboard에서 `Networking → Tunnels → homeServer-tunnel → Routes → Add route → Published application`을 선택합니다. 현재 `intro.woonapp.com` 경로를 생성했고, Cloudflare가 Tunnel 대상 CNAME DNS 레코드를 자동으로 만들었습니다. Cloudflare Dashboard의 Tunnel과 Published Application 설정은 이 저장소가 관리하지 않습니다.

| 항목 | 값 |
| --- | --- |
| Subdomain | `intro` |
| Domain | `woonapp.com` |
| Service type | `HTTP` |
| Service URL | 위에서 확인한 Traefik 내부 Service 주소 |

저장 후 `intro.woonapp.com` DNS 레코드가 만들어졌는지 확인합니다. 자동 생성되지 않았다면 Tunnel UUID를 대상으로 한 Proxied CNAME을 추가합니다. 첫 연결은 `intro.woonapp.com`만 설정하고, 와일드카드 호스트는 별도 검증 후 추가합니다.

## 상태 확인

```bash
kubectl get pods -n cloudflare
kubectl logs -n cloudflare deployment/cloudflared
kubectl get svc,ingress -n self-introduction
curl -I https://intro.woonapp.com
```

Cloudflare Tunnel을 통한 외부 요청은 Cloudflare에서 HTTPS로 받고, Tunnel 내부에서 Traefik까지는 HTTP로 전달합니다. 앱 경로는 `apps/self-Intruduction/ingress.yaml`에서 선언적으로 관리합니다.
