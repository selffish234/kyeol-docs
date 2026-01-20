# KYEOL Platform Runbook

> **대상**: 플랫폼 레이어 (Kubernetes & GitOps)
> **관리**: EKS Cluster, AWS Load Balancer Controller, ArgoCD, Kustomize, ExternalDNS, Fluent Bit
> **작성일**: 2026-01-20
> **버전**: 3.0

---

## 📌 핵심 원칙

### ❌ 절대 금지 사항
1. **kubectl apply로 직접 리소스 수정 금지**
   - GitOps 원칙 위반
   - Git 리포지토리가 Single Source of Truth
   - 예외: 긴급 장애 대응 시에만 허용 (사후 Git에 반영 필수)

2. **ArgoCD 동기화 강제 비활성화 금지**
   - Sync Policy 우회 금지
   - 자동 동기화 비활성화 금지 (특별한 이유 없이)

3. **Production Ingress 직접 수정 금지**
   - ALB 설정 변경은 반드시 Git을 통해
   - Annotation 변경도 GitOps 프로세스 준수

### ✅ 필수 준수 사항
- 모든 변경은 Git → ArgoCD Sync 흐름으로
- 환경별 context 분리 (dev/stage/prod)
- Ingress annotation은 [ALB Controller 공식 문서](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) 기준 준수

---

## 🏗️ 플랫폼 구성

### Kubernetes Add-ons

| Add-on | 관리 방식 | 용도 | 네임스페이스 |
|--------|----------|------|------------|
| **AWS Load Balancer Controller** | Helm (GitOps) | ALB/NLB 자동 생성 | `kube-system` |
| **ExternalDNS** | Helm (GitOps) | Route53 자동 관리 | `kube-system` |
| **Fluent Bit** | Helm (GitOps) | 로그 수집 → CloudWatch | `kube-system` |
| **ArgoCD** | Helm (MGMT only) | GitOps CD | `argocd` |
| **Metrics Server** | Manifest | HPA 지원 | `kube-system` |

### EKS 클러스터 구성

| 구성 요소 | 설명 |
|----------|------|
| **Control Plane** | AWS 관리형 (버전: 1.28+) |
| **Node Group** | Managed Node Group (EC2 Auto Scaling) |
| **OIDC Provider** | IRSA 지원 활성화 |
| **CNI** | AWS VPC CNI |
| **CSI** | EBS CSI Driver |

---

## 🚀 초기 설정

### 1.1 kubeconfig 설정

**모든 환경 추가**:
```powershell
# DEV
aws eks update-kubeconfig `
  --region ap-northeast-2 `
  --name min-kyeol-dev-eks `
  --alias dev

# STAGE
aws eks update-kubeconfig `
  --region ap-northeast-2 `
  --name min-kyeol-stage-eks `
  --alias stage

# PROD
aws eks update-kubeconfig `
  --region ap-northeast-2 `
  --name min-kyeol-prod-eks `
  --alias prod

# MGMT (ArgoCD)
aws eks update-kubeconfig `
  --region ap-northeast-2 `
  --name min-kyeol-mgmt-eks `
  --alias mgmt
```

**Context 전환**:
```powershell
kubectl config use-context dev
kubectl config use-context prod
kubectl config current-context  # 현재 context 확인
```

**Namespace 기본 설정** (선택사항):
```powershell
kubectl config set-context --current --namespace=saleor
```

### 1.2 클러스터 접근 확인

```powershell
# 클러스터 정보
kubectl cluster-info

# 노드 확인
kubectl get nodes -o wide

# 시스템 Pod 확인
kubectl get pods -n kube-system
```

---

## 🔧 AWS Load Balancer Controller

### 배포 (kyeol-platform-gitops)

**리포지토리**: `kyeol-platform-gitops/clusters/[env]/values/aws-load-balancer-controller.yaml`

**주요 설정**:
```yaml
clusterName: min-kyeol-[env]-eks
region: ap-northeast-2
vpcId: vpc-xxxxx

serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::[account]:role/min-kyeol-[env]-alb-controller-role

replicaCount: 2
resources:
  limits:
    cpu: 200m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi
```

### 배포 확인

```powershell
# Pod 상태 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# 로그 확인
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50

# Webhook 확인
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

### ALB 생성 확인 (Ingress 적용 후)

```powershell
# Ingress 상태
kubectl get ingress -n saleor

# ALB 생성 확인 (AWS)
aws elbv2 describe-load-balancers `
  --query "LoadBalancers[?contains(LoadBalancerName,'k8s-saleor')]"

# Target Group Health Check
$TG_ARN = (kubectl get ingress saleor-ingress -n saleor `
  -o jsonpath='{.metadata.annotations.alb\.ingress\.kubernetes\.io/target-group-arns}')

aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

### 문제 해결

**Ingress ADDRESS가 비어있음**:
```powershell
# Controller 로그 확인
kubectl logs -n kube-system deploy/aws-load-balancer-controller --tail=100

# 일반적인 원인:
# 1. IAM Role 권한 부족
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml

# 2. Subnet Tag 누락 (Terraform에서 자동 설정됨)
# Public Subnet: kubernetes.io/role/elb = 1
# Private Subnet: kubernetes.io/role/internal-elb = 1
```

**ALB가 생성되지 않음**:
```powershell
# Ingress Events 확인
kubectl describe ingress saleor-ingress -n saleor

# Controller Pod 재시작
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
```

---

## 🌐 ExternalDNS

### 배포 (kyeol-platform-gitops)

**주요 설정**:
```yaml
provider: aws
sources:
  - ingress
  - service

domainFilters:
  - msp-g1.click  # 또는 kyeol.click

txtOwnerId: min-kyeol-[env]-externaldns  # 환경별 고유값

serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::[account]:role/min-kyeol-[env]-external-dns-role

policy: sync  # sync | upsert-only

resources:
  limits:
    cpu: 50m
    memory: 128Mi
```

### 배포 확인

```powershell
# Pod 상태
kubectl get pods -n kube-system -l app.kubernetes.io/name=external-dns

# 로그 확인 (DNS 레코드 생성 로그)
kubectl logs -n kube-system -l app.kubernetes.io/name=external-dns --tail=50
```

### DNS 레코드 확인

```powershell
# Route53에서 확인
aws route53 list-resource-record-sets `
  --hosted-zone-id Z0XXXXXXXXXXXX `
  --query "ResourceRecordSets[?contains(Name,'origin')]"

# TXT 레코드 확인 (ExternalDNS 소유권)
nslookup -type=TXT origin-dev-kyeol.click
```

### Ingress Annotation 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: saleor-ingress
  annotations:
    # ALB Controller
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:...

    # ExternalDNS (자동 DNS 생성)
    external-dns.alpha.kubernetes.io/hostname: origin-dev-kyeol.click
spec:
  ingressClassName: alb
  rules:
  - host: origin-dev-kyeol.click
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: saleor-api
            port:
              number: 8000
```

**적용 후 자동 동작**:
1. ALB Controller가 ALB 생성
2. ExternalDNS가 Route53에 `origin-dev-kyeol.click` A 레코드 생성 (ALB를 가리킴)

---

## 📊 Fluent Bit (로그 수집)

### 배포 (kyeol-platform-gitops)

**주요 설정**:
```yaml
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::[account]:role/min-kyeol-[env]-fluent-bit-role

config:
  outputs: |
    [OUTPUT]
        Name cloudwatch_logs
        Match *
        region ap-northeast-2
        log_group_name /aws/eks/min-kyeol-[env]-cluster/logs
        log_stream_prefix from-fluent-bit-
        auto_create_group true

  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On
```

### 배포 확인

```powershell
# DaemonSet 확인 (모든 노드에 1개씩)
kubectl get daemonset fluent-bit -n kube-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=fluent-bit

# 로그 수집 확인
kubectl logs -n kube-system -l app.kubernetes.io/name=fluent-bit --tail=50
```

### CloudWatch Logs 확인

```powershell
# Log Group 확인
aws logs describe-log-groups `
  --log-group-name-prefix /aws/eks/min-kyeol

# Log Stream 목록
aws logs describe-log-streams `
  --log-group-name /aws/eks/min-kyeol-dev-cluster/logs `
  --max-items 10

# 로그 조회
aws logs tail /aws/eks/min-kyeol-dev-cluster/logs --follow
```

---

## 🔄 ArgoCD (MGMT 클러스터)

### 초기 설정

**ArgoCD 설치** (MGMT 클러스터에만):
```powershell
kubectl config use-context mgmt

# Helm으로 설치
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd `
  --namespace argocd `
  --create-namespace `
  --set server.service.type=LoadBalancer
```

**Admin 비밀번호 확인**:
```powershell
kubectl get secret argocd-initial-admin-secret `
  -n argocd `
  -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

**ArgoCD UI 접근**:
```powershell
# LoadBalancer 주소 확인
kubectl get svc argocd-server -n argocd

# Port Forward (로컬 접근)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# 브라우저: https://localhost:8080
# ID: admin, PW: (위에서 확인한 비밀번호)
```

### 타겟 클러스터 등록

**DEV 클러스터 등록**:
```powershell
# Context 확인
kubectl config get-contexts

# ArgoCD CLI 로그인
argocd login localhost:8080 --username admin --password [password] --insecure

# 클러스터 추가
argocd cluster add dev --name dev-cluster
argocd cluster add stage --name stage-cluster
argocd cluster add prod --name prod-cluster
```

### Application 생성

**Saleor API (DEV) 예시**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: saleor-api-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mandoofu/kyeol-app-gitops
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://[dev-cluster-api]
    namespace: saleor
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**적용**:
```powershell
kubectl apply -f argocd-app-saleor-dev.yaml
```

### 동기화 확인

```powershell
# Application 상태
argocd app list

# 특정 App 상태
argocd app get saleor-api-dev

# 수동 동기화
argocd app sync saleor-api-dev

# 로그 확인
argocd app logs saleor-api-dev --tail=50
```

---

## 🛠️ 일반 운영 작업

### Namespace 생성

```powershell
kubectl create namespace saleor
kubectl label namespace saleor environment=dev
```

### ConfigMap / Secret 관리

**ConfigMap 생성** (예시):
```powershell
kubectl create configmap saleor-config `
  --from-literal=ALLOWED_HOSTS=api-dev.kyeol.click `
  --from-literal=DATABASE_URL=postgres://... `
  -n saleor
```

**Secret 생성** (Base64 인코딩):
```powershell
kubectl create secret generic saleor-secrets `
  --from-literal=SECRET_KEY=your-secret-key `
  --from-literal=DB_PASSWORD=your-db-password `
  -n saleor
```

**Git에 반영** (GitOps):
- ❌ `kubectl create`로 직접 생성 → 임시
- ✅ `kyeol-app-gitops/overlays/dev/` 디렉토리에 YAML 파일 추가 → ArgoCD Sync

### 리소스 상태 확인

```powershell
# Pod 상태
kubectl get pods -n saleor -o wide

# Pod 로그
kubectl logs -n saleor saleor-api-[pod-id] --tail=100 -f

# Pod 접속 (디버깅)
kubectl exec -it -n saleor saleor-api-[pod-id] -- /bin/bash

# 리소스 사용량
kubectl top nodes
kubectl top pods -n saleor
```

### Rolling Update / Rollback

**이미지 태그 변경** (GitOps):
```powershell
# 1. kyeol-app-gitops 리포지토리 클론
git clone https://github.com/mandoofu/kyeol-app-gitops
cd kyeol-app-gitops

# 2. overlays/dev/kustomization.yaml 편집
# images:
#   - name: [account].dkr.ecr.ap-northeast-2.amazonaws.com/min-kyeol-dev-saleor-api
#     newTag: v1.2.0  # 변경

# 3. Git Push
git add .
git commit -m "Update Saleor API to v1.2.0"
git push

# 4. ArgoCD 자동 Sync (또는 수동)
argocd app sync saleor-api-dev
```

**Rollback**:
```powershell
# ArgoCD UI 또는 CLI에서 이전 버전으로 롤백
argocd app rollback saleor-api-dev [revision-id]

# Git에서 Revert
git revert HEAD
git push
```

### HPA (Horizontal Pod Autoscaler)

**HPA 생성**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: saleor-api-hpa
  namespace: saleor
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: saleor-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**HPA 확인**:
```powershell
kubectl get hpa -n saleor
kubectl describe hpa saleor-api-hpa -n saleor
```

---

## 📊 모니터링 & 로깅

### Metrics Server 확인

```powershell
kubectl top nodes
kubectl top pods -A
```

**Metrics Server 미설치 시**:
```powershell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### CloudWatch Logs Insights 쿼리

**Saleor API 에러 로그**:
```sql
fields @timestamp, log
| filter kubernetes.namespace_name = "saleor"
| filter kubernetes.container_name = "saleor-api"
| filter log like /ERROR|CRITICAL/
| sort @timestamp desc
| limit 100
```

**요청 응답 시간 분석**:
```sql
fields @timestamp, log
| parse log /status=(?<status>\d+) duration=(?<duration>\d+)ms/
| filter kubernetes.namespace_name = "saleor"
| stats avg(duration), max(duration), count() by status
```

---

## ⚠️ 문제 해결

### Pod가 Pending 상태

**원인 1**: 노드 리소스 부족
```powershell
kubectl describe pod [pod-name] -n saleor
# Events에서 "Insufficient cpu" 또는 "Insufficient memory" 확인

# 해결: Auto Scaling 또는 노드 추가
```

**원인 2**: ImagePullBackOff
```powershell
kubectl describe pod [pod-name] -n saleor
# Events에서 "ErrImagePull" 확인

# ECR 권한 확인
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin [account].dkr.ecr.ap-northeast-2.amazonaws.com

# 이미지 태그 확인
aws ecr describe-images --repository-name min-kyeol-dev-saleor-api
```

### Ingress ADDRESS 없음

```powershell
kubectl get ingress -n saleor
# ADDRESS가 비어있으면 ALB Controller 문제

# ALB Controller 로그
kubectl logs -n kube-system deploy/aws-load-balancer-controller

# Subnet Tag 확인 (Terraform)
# kubernetes.io/role/elb = 1 (Public)
# kubernetes.io/cluster/min-kyeol-[env]-eks = owned
```

### DNS 레코드 생성 안 됨

```powershell
# ExternalDNS 로그
kubectl logs -n kube-system -l app.kubernetes.io/name=external-dns --tail=100

# 권한 확인
kubectl get sa external-dns -n kube-system -o yaml

# Ingress annotation 확인
kubectl get ingress saleor-ingress -n saleor -o yaml
# external-dns.alpha.kubernetes.io/hostname이 있는지 확인
```

---

## 📚 환경별 차이점

| 항목 | DEV | STAGE | PROD |
|------|-----|-------|------|
| **Node 수** | 2 | 2-4 | 3-5 |
| **HPA Min** | 1 | 2 | 2 |
| **HPA Max** | 3 | 5 | 10 |
| **ALB Scheme** | internet-facing | internet-facing | internet-facing |
| **ArgoCD Sync** | Auto (selfHeal: true) | Auto | Manual 권장 |
| **Log Retention** | 7일 | 14일 | 30일 |

---

**최종 업데이트**: 2026-01-20
**작성자**: KYEOL DevOps Team
**이전 문서**: [runbook-infra.md](./runbook-infra.md) - 인프라 레이어 운영 가이드
**다음 문서**: [runbook-ops.md](./runbook-ops.md) - 애플리케이션 레이어 운영 가이드
