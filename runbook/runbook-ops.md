# KYEOL Operations Runbook

> **대상**: 애플리케이션 레이어
> **관리**: Saleor Core, Saleor Dashboard, Saleor Storefront
> **작성일**: 2026-01-20
> **버전**: 3.0

---

## 📌 핵심 원칙

### ❌ 절대 금지 사항
1. **프로덕션 DB 직접 수정 금지**
   - 반드시 Django 마이그레이션 스크립트 사용
   - SQL 직접 실행 금지 (읽기 전용 쿼리 제외)

2. **민감 정보 평문 저장 금지**
   - Kubernetes Secret 암호화 필수
   - Git에 비밀번호/API 키 커밋 금지
   - 환경 변수에 민감 정보 하드코딩 금지

3. **Production 환경 직접 배포 금지**
   - 반드시 STAGE 환경 검증 후 PROD 배포
   - GitOps 프로세스 우회 금지

### ✅ 필수 준수 사항
- 모든 배포는 Git → ArgoCD Sync 흐름
- 환경별 도메인 명확히 구분
- 데이터베이스 마이그레이션 전 백업 필수

---

## 🏗️ 애플리케이션 구성

### Saleor 컴포넌트

| 컴포넌트 | 기술 스택 | 포트 | 설명 |
|----------|----------|------|------|
| **Saleor API** | Python/Django/GraphQL | 8000 | 백엔드 API 서버 |
| **Storefront** | Next.js/React (SSR) | 3000 | 프론트엔드 (고객용) |
| **Dashboard** | React (SPA) | 9000 | 관리자 대시보드 |

### 데이터 저장소

| 용도 | 서비스 | 연결 정보 |
|------|--------|----------|
| **메인 DB** | RDS PostgreSQL | Secrets Manager: `min-kyeol-[env]-db-secret` |
| **세션/캐시** | Valkey (ElastiCache) | 환경 변수: `REDIS_URL` |
| **미디어** | S3 | 환경 변수: `AWS_MEDIA_BUCKET_NAME` |

---

## 📂 디렉토리 구조

```
kyeol-app-gitops/
├── base/
│   ├── api/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── secret.yaml
│   ├── storefront/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── dashboard/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── api-patch.yaml
    │   ├── storefront-patch.yaml
    │   ├── dashboard-patch.yaml
    │   └── ingress.yaml
    ├── stage/
    │   └── ...
    └── prod/
        └── ...
```

---

## 🚀 초기 배포

### 1.1 사전 준비

**필수 인프라 확인**:
- ✅ EKS 클러스터 Running
- ✅ RDS PostgreSQL Running
- ✅ S3 Media Bucket 생성
- ✅ ACM 인증서 발급 완료
- ✅ Secrets Manager에 DB 자격증명 저장

**Namespace 생성**:
```powershell
kubectl create namespace saleor --context=dev
kubectl label namespace saleor environment=dev
```

### 1.2 Secret 생성

**DB 연결 정보** (Secrets Manager → Kubernetes Secret):
```powershell
# Secrets Manager에서 값 가져오기
$DB_SECRET = (aws secretsmanager get-secret-value `
  --secret-id min-kyeol-dev-db-secret `
  --query SecretString `
  --output text | ConvertFrom-Json)

# Kubernetes Secret 생성
kubectl create secret generic saleor-db-secret `
  --from-literal=DATABASE_URL="postgresql://$($DB_SECRET.username):$($DB_SECRET.password)@$($DB_SECRET.host):$($DB_SECRET.port)/$($DB_SECRET.dbname)" `
  -n saleor `
  --context=dev
```

**Saleor Secret Key**:
```powershell
# Django SECRET_KEY 생성 (Python)
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"

# Kubernetes Secret 생성
kubectl create secret generic saleor-app-secret `
  --from-literal=SECRET_KEY="[generated-secret-key]" `
  -n saleor `
  --context=dev
```

### 1.3 ConfigMap 생성

**Saleor API ConfigMap**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: saleor-api-config
  namespace: saleor
data:
  ALLOWED_HOSTS: "api-dev.kyeol.click,origin-dev-kyeol.click"
  ALLOWED_CLIENT_HOSTS: "dev-kyeol.click"
  DEFAULT_CHANNEL_SLUG: "default-channel"
  AWS_MEDIA_BUCKET_NAME: "min-kyeol-dev-media-123456789012-ap-northeast-2"
  AWS_STORAGE_BUCKET_NAME: "min-kyeol-dev-media-123456789012-ap-northeast-2"
  AWS_REGION: "ap-northeast-2"
  REDIS_URL: "redis://min-kyeol-dev-valkey.xxxxx.cache.amazonaws.com:6379/0"
```

**적용**:
```powershell
kubectl apply -f configmap-saleor-api.yaml --context=dev
```

### 1.4 데이터베이스 마이그레이션

**초기 마이그레이션**:
```powershell
# Saleor API Pod 확인
kubectl get pods -n saleor -l app=saleor-api --context=dev

# Pod에 접속하여 마이그레이션 실행
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- python manage.py migrate

# Superuser 생성
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- python manage.py createsuperuser
```

**마이그레이션 Job 방식** (권장):
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: saleor-migrate
  namespace: saleor
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: [account].dkr.ecr.ap-northeast-2.amazonaws.com/min-kyeol-dev-saleor-api:latest
        command: ["python", "manage.py", "migrate"]
        envFrom:
        - configMapRef:
            name: saleor-api-config
        - secretRef:
            name: saleor-db-secret
        - secretRef:
            name: saleor-app-secret
      restartPolicy: OnFailure
```

**적용 및 확인**:
```powershell
kubectl apply -f job-migrate.yaml --context=dev
kubectl get jobs -n saleor --context=dev
kubectl logs job/saleor-migrate -n saleor --context=dev
```

### 1.5 카탈로그 초기 데이터 주입

**스크립트 위치**: `D:\4th_Parkminwook\WORKSPACE\kyeol-project\scripts\seed_kyeol_catalog.py`

**데이터 파일 위치**: `D:\4th_Parkminwook\WORKSPACE\kyeol-project\archive\seed-data\`

**실행 순서**:
```powershell
# 1. 이미지 S3 업로드 (선행 작업)
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project
$env:S3_BUCKET="min-kyeol-dev-media-123456789012-ap-northeast-2"
$env:AWS_REGION="ap-northeast-2"
python scripts\upload_images_to_s3.py

# 2. API 토큰 생성 (Dashboard 또는 API)
# Dashboard → Apps → Create App → Generate Token

# 3. 카탈로그 시딩
$env:SALEOR_API_URL="https://api-dev.kyeol.click/graphql/"
$env:SALEOR_TOKEN="your-api-token"
python scripts\seed_kyeol_catalog.py --token $env:SALEOR_TOKEN
```

**예상 결과**:
```
✓ 카테고리 4개 생성
✓ 상품 100개 생성
✓ Variant 260개 생성
✓ 이미지 연결 완료
```

---

## 🔄 배포 및 업데이트

### GitOps 배포 흐름

```
Developer → GitHub Push → GitHub Actions (Build & Push to ECR)
                                ↓
                      Update kyeol-app-gitops (image tag)
                                ↓
                            ArgoCD Sync
                                ↓
                          EKS Deployment
```

### 이미지 빌드 & Push

**GitHub Actions Workflow** (`.github/workflows/build-push.yaml`):
```yaml
name: Build and Push to ECR

on:
  push:
    branches: [main]
    paths:
      - 'saleor/**'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
          aws-region: ap-northeast-2

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/min-kyeol-dev-saleor-api:$IMAGE_TAG .
          docker push $ECR_REGISTRY/min-kyeol-dev-saleor-api:$IMAGE_TAG
          docker tag $ECR_REGISTRY/min-kyeol-dev-saleor-api:$IMAGE_TAG $ECR_REGISTRY/min-kyeol-dev-saleor-api:latest
          docker push $ECR_REGISTRY/min-kyeol-dev-saleor-api:latest
```

### Manifest 업데이트

**kustomization.yaml 수정**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base/api

images:
  - name: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/min-kyeol-dev-saleor-api
    newTag: abc123def  # GitHub SHA

patches:
  - path: api-patch.yaml
```

**Git Push**:
```powershell
cd kyeol-app-gitops
git add overlays/dev/kustomization.yaml
git commit -m "Update Saleor API image to abc123def"
git push
```

**ArgoCD Sync** (자동 또는 수동):
```powershell
# 수동 Sync
argocd app sync saleor-api-dev

# 자동 Sync 확인
argocd app get saleor-api-dev
```

### Rolling Update 확인

```powershell
# Deployment Rollout 상태
kubectl rollout status deployment/saleor-api -n saleor --context=dev

# Pod 변경 확인
kubectl get pods -n saleor -l app=saleor-api --context=dev -w

# 새 이미지 확인
kubectl describe pod -n saleor saleor-api-[new-pod-id] --context=dev | Select-String "Image:"
```

---

## 🛠️ 운영 작업

### 환경 변수 업데이트

**ConfigMap 수정**:
```powershell
# 1. Git에서 수정
cd kyeol-app-gitops
# overlays/dev/configmap-patch.yaml 편집

# 2. Git Push
git add .
git commit -m "Update ALLOWED_HOSTS"
git push

# 3. ArgoCD Sync 후 Pod 재시작
argocd app sync saleor-api-dev
kubectl rollout restart deployment/saleor-api -n saleor --context=dev
```

### 로그 확인

**실시간 로그**:
```powershell
# Saleor API 로그
kubectl logs -n saleor -l app=saleor-api --tail=100 -f --context=dev

# Storefront 로그
kubectl logs -n saleor -l app=saleor-storefront --tail=100 -f --context=dev

# 특정 Pod 로그
kubectl logs -n saleor saleor-api-[pod-id] --context=dev
```

**CloudWatch Logs Insights 쿼리**:
```sql
fields @timestamp, log
| filter kubernetes.namespace_name = "saleor"
| filter kubernetes.pod_name like /saleor-api/
| filter log like /ERROR|CRITICAL|Exception/
| sort @timestamp desc
| limit 100
```

### Health Check

**API Health Endpoint**:
```powershell
# Kubernetes Liveness/Readiness Probe
kubectl describe pod -n saleor saleor-api-[pod-id] --context=dev | Select-String "Liveness|Readiness"

# 직접 Health Check
$POD_IP = (kubectl get pod -n saleor saleor-api-[pod-id] -o jsonpath='{.status.podIP}' --context=dev)
Invoke-WebRequest -Uri "http://$POD_IP:8000/health/"
```

### 데이터베이스 마이그레이션

**새 마이그레이션 파일 확인**:
```powershell
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- python manage.py showmigrations
```

**마이그레이션 적용**:
```powershell
# 1. RDS 백업 생성 (필수)
aws rds create-db-snapshot `
  --db-instance-identifier min-kyeol-dev-rds `
  --db-snapshot-identifier min-kyeol-dev-pre-migration-$(Get-Date -Format "yyyyMMdd-HHmmss")

# 2. 마이그레이션 실행
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- python manage.py migrate

# 3. 확인
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- python manage.py showmigrations
```

### Scale Out/In

**수동 스케일링**:
```powershell
# Replica 수 변경
kubectl scale deployment saleor-api --replicas=5 -n saleor --context=dev

# 확인
kubectl get deployment saleor-api -n saleor --context=dev
```

**HPA 설정** (권장):
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
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 🔐 보안 관리

### API 토큰 생성

**Dashboard에서 생성**:
1. Dashboard 접속: `https://dashboard-dev.kyeol.click`
2. Apps → Create App
3. Permissions 설정:
   - `MANAGE_PRODUCTS`
   - `MANAGE_ORDERS`
   - `MANAGE_USERS`
4. Generate Token → 복사

**토큰 저장** (Kubernetes Secret):
```powershell
kubectl create secret generic saleor-external-api-token `
  --from-literal=API_TOKEN="[token]" `
  -n saleor `
  --context=dev
```

### S3 접근 권한 (IRSA)

**IAM Role 확인**:
```powershell
# ServiceAccount Annotation 확인
kubectl get sa saleor-api -n saleor --context=dev -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'

# IAM Role Policy 확인
aws iam get-role-policy `
  --role-name min-kyeol-dev-saleor-api-role `
  --policy-name S3MediaAccess
```

**S3 접근 테스트**:
```powershell
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- aws s3 ls s3://min-kyeol-dev-media-123456789012-ap-northeast-2/
```

---

## 📊 모니터링

### Grafana 대시보드 (Phase 4)

**메트릭 항목**:
- API 요청 수 (RPS)
- API 응답 시간 (p50, p95, p99)
- 에러율
- Pod CPU/Memory 사용률
- DB 연결 수

### CloudWatch Alarms

**API Pod CPU 사용률**:
```powershell
aws cloudwatch put-metric-alarm `
  --alarm-name "kyeol-dev-api-cpu-high" `
  --alarm-description "Saleor API CPU > 80%" `
  --namespace "ContainerInsights" `
  --metric-name "pod_cpu_utilization" `
  --dimensions Name=ClusterName,Value=min-kyeol-dev-eks Name=Namespace,Value=saleor `
  --statistic Average `
  --period 300 `
  --evaluation-periods 2 `
  --threshold 80 `
  --comparison-operator GreaterThanThreshold
```

---

## 🌐 환경별 차이점

| 항목 | DEV | STAGE | PROD |
|------|-----|-------|------|
| **도메인 (Storefront)** | `dev-kyeol.click` | `stage-kyeol.click` | `kyeol.click` |
| **도메인 (API)** | `api-dev.kyeol.click` | `api-stage.kyeol.click` | `api.kyeol.click` |
| **도메인 (Dashboard)** | `dashboard-dev.kyeol.click` | `dashboard-stage.kyeol.click` | `dashboard.kyeol.click` |
| **Replica (API)** | 2 | 2-4 | 3-10 |
| **Replica (Storefront)** | 2 | 2-4 | 3-10 |
| **Replica (Dashboard)** | 1 | 2 | 2 |
| **RDS** | db.t3.small (단일 AZ) | db.t3.medium (Multi-AZ) | db.t3.medium (Multi-AZ) |
| **Valkey** | ❌ 사용 안 함 | cache.t3.small | cache.t3.small |
| **S3 Bucket** | `min-kyeol-dev-media-*` | `min-kyeol-stage-media-*` | `min-kyeol-prod-media-*` |

---

## ⚠️ 문제 해결

### API 500 에러

**확인**:
```powershell
# Pod 로그
kubectl logs -n saleor -l app=saleor-api --tail=100 --context=dev

# Django 에러 로그
kubectl logs -n saleor saleor-api-[pod-id] --context=dev | Select-String "ERROR|Traceback"
```

**일반적인 원인**:
1. DB 연결 실패 → `DATABASE_URL` 확인
2. Secret Key 누락 → `SECRET_KEY` 확인
3. S3 권한 부족 → IRSA Role Policy 확인

### Storefront가 API에 연결 안 됨

**확인**:
```powershell
# Storefront 환경 변수
kubectl describe pod -n saleor saleor-storefront-[pod-id] --context=dev | Select-String "NEXT_PUBLIC_API_URI"

# API Ingress 주소
kubectl get ingress -n saleor --context=dev
```

**해결**:
- `NEXT_PUBLIC_API_URI` 환경 변수가 올바른 API 도메인을 가리키는지 확인
- CORS 설정 확인: `ALLOWED_CLIENT_HOSTS`

### 이미지 업로드 실패

**확인**:
```powershell
# S3 버킷 정책
aws s3api get-bucket-policy --bucket min-kyeol-dev-media-123456789012-ap-northeast-2

# CORS 설정
aws s3api get-bucket-cors --bucket min-kyeol-dev-media-123456789012-ap-northeast-2
```

**해결**:
```powershell
# CORS 설정 추가
aws s3api put-bucket-cors --bucket min-kyeol-dev-media-123456789012-ap-northeast-2 --cors-configuration file://cors.json
```

**cors.json**:
```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["https://dev-kyeol.click", "https://dashboard-dev.kyeol.click"],
      "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3000
    }
  ]
}
```

---

## 📚 참고 문서

- [Saleor 공식 문서](https://docs.saleor.io/)
- [Saleor GraphQL API](https://docs.saleor.io/docs/3.x/api-reference/overview)
- [scripts/README.md](D:\4th_Parkminwook\WORKSPACE\kyeol-project\scripts\README.md) - 유틸리티 스크립트 사용법
- [troubleshooting.md](../troubleshooting.md) - 실제 발생 기반 장애 대응

---

**최종 업데이트**: 2026-01-20
**작성자**: KYEOL DevOps Team
**이전 문서**: [runbook-platform.md](./runbook-platform.md) - 플랫폼 레이어 운영 가이드
**다음 문서**: [../troubleshooting.md](../troubleshooting.md) - 장애 대응 가이드
