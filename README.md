# homelab-gitOps

Argo CD를 통해 Kubernetes 클러스터에 자기소개 웹 애플리케이션을 배포하는 GitOps 매니페스트 저장소입니다. 애플리케이션 이미지는 GitHub Container Registry의 `ghcr.io/woonkim0413/self-introduction:initial`을 사용합니다.

## 저장소 구성

```text
.
├── argocd/
│   └── self-Intruduction.yaml       # 앱을 등록하는 Argo CD Application
└── apps/
    └── self-Intruduction/
        ├── deployment.yaml          # 웹 애플리케이션 Deployment
        └── service.yaml             # 외부 접근용 NodePort Service
```

## 배포 설정

`argocd/self-Intruduction.yaml`의 Application은 `main` 브랜치의 `apps/self-Intruduction` 경로를 추적합니다. Argo CD 자동 동기화가 켜져 있어 Git 변경 사항을 클러스터에 반영하고, Git에서 제거된 리소스를 정리하며, 클러스터의 변경 사항을 선언된 상태로 되돌립니다. 대상 네임스페이스는 `self-Intruduction`이며 없으면 자동 생성합니다.

Deployment는 애플리케이션을 2개 복제본으로 실행하고 컨테이너 포트 `8080`을 사용합니다. Service는 NodePort `30080`을 통해 포트 `8080`으로 트래픽을 전달하도록 설정되어 있습니다.

## 클러스터에 등록

Argo CD가 설치된 클러스터에서 Application 매니페스트를 적용합니다.

```bash
kubectl apply -f argocd/self-Intruduction.yaml
```

동기화 상태와 생성된 리소스를 확인할 수 있습니다.

```bash
kubectl get application self-introduction -n argocd
kubectl get deployment,service,pods -n self-Intruduction
```

노드의 주소와 NodePort `30080`을 사용해 애플리케이션에 접근할 수 있습니다. 클러스터의 네트워크 정책과 방화벽에서 해당 포트 접근을 허용해야 합니다.

## 확인할 설정

- `apps/self-Intruduction/service.yaml`의 Service selector는 `app: spring-demo`지만 Deployment Pod 라벨은 `app: self-Intruduction`입니다. Service가 Pod를 선택하도록 두 값이 일치해야 합니다.
- `argocd/self-Intruduction.yaml`의 `source.repoURL`과 `targetRevision`은 현재 저장소 주소와 배포에 사용할 브랜치에 맞게 설정해야 합니다.
