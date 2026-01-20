# KYEOL Troubleshooting Guide

> **작성일**: 2026-01-20
> **버전**: 3.0
> **목적**: 실제 발생 사례 기반 장애 대응 가이드

**구조**: 증상 → 원인 → 확인 → 해결 → 재발 방지

---

## 📋 목차

1. [인프라 레이어](#1-인프라-레이어)
   - [1.1 Terraform 오류](#11-terraform-오류)
   - [1.2 RDS 연결 실패](#12-rds-연결-실패)
   - [1.3 NAT Gateway 장애](#13-nat-gateway-장애)
   - [1.4 CloudFront 503 에러](#14-cloudfront-503-에러)

2. [플랫폼 레이어](#2-플랫폼-레이어)
   - [2.1 ALB가 생성되지 않음](#21-alb가-생성되지-않음)
   - [2.2 Ingress ADDRESS 없음](#22-ingress-address-없음)
   - [2.3 ExternalDNS 미작동](#23-externaldns-미작동)
   - [2.4 Pod ImagePullBackOff](#24-pod-imagepullbackoff)
   - [2.5 Pod CrashLoopBackOff](#25-pod-crashloopbackoff)
   - [2.6 Pod Pending 상태](#26-pod-pending-상태)

3. [애플리케이션 레이어](#3-애플리케이션-레이어)
   - [3.1 API 500 Internal Server Error](#31-api-500-internal-server-error)
   - [3.2 Storefront가 API에 연결 안 됨](#32-storefront가-api에-연결-안-됨)
   - [3.3 Dashboard 로그인 실패](#33-dashboard-로그인-실패)
   - [3.4 이미지 업로드 실패](#34-이미지-업로드-실패)
   - [3.5 GraphQL 쿼리 타임아웃](#35-graphql-쿼리-타임아웃)

4. [네트워크 & DNS](#4-네트워크--dns)
   - [4.1 ALB 502 Bad Gateway](#41-alb-502-bad-gateway)
   - [4.2 Route53 레코드 미생성](#42-route53-레코드-미생성)
   - [4.3 HTTPS 인증서 오류](#43-https-인증서-오류)
   - [4.4 CORS 에러](#44-cors-에러)

---

## 1. 인프라 레이어

### 1.1 Terraform 오류

#### 사례 1: State Lock 충돌

**증상**:
```
Error acquiring the state lock
Lock Info:
  ID:        abc123-def456-ghi789
  Path:      min-kyeol-tfstate-123456789012/dev/terraform.tfstate
```

**원인**: 이전 terraform 실행이 비정상 종료되어 DynamoDB lock이 남아있음

**확인**:
```powershell
# DynamoDB Lock 테이블 확인
aws dynamodb scan --table-name min-kyeol-terraform-locks --max-items 10
```

**해결**:
```powershell
# 1. 다른 사람이 실행 중인지 팀원 확인 (필수!)
# 2. Lock 강제 해제
terraform force-unlock abc123-def456-ghi789

# 3. 재실행
terraform plan
terraform apply
```

**재발 방지**:
- Terraform 실행 시 중단하지 말고 완료 대기
- CI/CD 파이프라인에서 timeout 설정
- 팀원과 Terraform 실행 전 커뮤니케이션

---

#### 사례 2: Resource Already Exists

**증상**:
```
Error: Error creating EKS Cluster: ResourceInUseException:
Cluster already exists: min-kyeol-dev-eks
```

**원인**: Terraform 상태 파일이 실제 AWS 리소스와 불일치

**확인**:
```powershell
# AWS에서 리소스 확인
aws eks describe-cluster --name min-kyeol-dev-eks --region ap-northeast-2

# Terraform 상태 확인
terraform state list | Select-String "eks_cluster"
```

**해결**:
```powershell
# Option 1: Import (수동 생성된 리소스를 Terraform 관리로)
terraform import module.eks.aws_eks_cluster.main min-kyeol-dev-eks

# Option 2: 상태 제거 후 재생성 (위험 - 백업 필수)
terraform state rm module.eks.aws_eks_cluster.main
terraform plan
terraform apply
```

**재발 방지**:
- 수동으로 AWS 리소스 생성 금지
- Terraform으로만 인프라 변경
- 상태 파일 주기적 백업

---

### 1.2 RDS 연결 실패

#### 사례 1: Security Group 차단

**증상**: Saleor API Pod에서 DB 연결 에러
```
django.db.utils.OperationalError: could not connect to server: Connection timed out
```

**원인**: RDS Security Group이 EKS Node Security Group을 허용하지 않음

**확인**:
```powershell
# RDS Security Group 확인
$RDS_SG = (aws rds describe-db-instances `
  --db-instance-identifier min-kyeol-dev-rds `
  --query "DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId" `
  --output text)

aws ec2 describe-security-groups --group-ids $RDS_SG --query "SecurityGroups[0].IpPermissions"

# EKS Node Security Group ID 확인
kubectl get nodes -o jsonpath='{.items[0].spec.providerID}'
# 출력: aws:///ap-northeast-2a/i-0123456789abcdef
aws ec2 describe-instances --instance-ids i-0123456789abcdef --query "Reservations[0].Instances[0].SecurityGroups"
```

**해결**:
```powershell
# Terraform에서 수정 (modules/rds_postgres/main.tf)
# ingress {
#   from_port       = 5432
#   to_port         = 5432
#   protocol        = "tcp"
#   security_groups = [var.eks_node_security_group_id]  # 추가
# }

# 또는 AWS CLI (임시)
aws ec2 authorize-security-group-ingress `
  --group-id $RDS_SG `
  --protocol tcp `
  --port 5432 `
  --source-group [EKS_NODE_SG_ID]
```

**재발 방지**:
- Terraform 모듈에 EKS Node SG를 RDS Ingress에 자동 추가
- 배포 후 연결 테스트 자동화

---

#### 사례 2: Endpoint 변경

**증상**: DB 연결 에러 (RDS 재생성 후)

**원인**: Terraform Apply로 RDS를 재생성했으나 Kubernetes Secret이 이전 Endpoint를 가리킴

**확인**:
```powershell
# 현재 RDS Endpoint
aws rds describe-db-instances `
  --db-instance-identifier min-kyeol-dev-rds `
  --query "DBInstances[0].Endpoint.Address" `
  --output text

# Kubernetes Secret 확인
kubectl get secret saleor-db-secret -n saleor --context=dev -o jsonpath='{.data.DATABASE_URL}' | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

**해결**:
```powershell
# Secret 업데이트
kubectl delete secret saleor-db-secret -n saleor --context=dev

kubectl create secret generic saleor-db-secret `
  --from-literal=DATABASE_URL="postgresql://admin:[password]@[NEW_ENDPOINT]:5432/saleordb" `
  -n saleor `
  --context=dev

# Pod 재시작
kubectl rollout restart deployment/saleor-api -n saleor --context=dev
```

**재발 방지**:
- Terraform Output을 CI/CD에서 읽어 자동으로 Secret 업데이트
- Secrets Manager 사용 (DB 자격증명 중앙 관리)

---

### 1.3 NAT Gateway 장애

**증상**: Private Subnet의 Pod에서 외부 API 호출 실패
```
curl: (7) Failed to connect to api.example.com port 443: Connection timed out
```

**원인**: NAT Gateway 장애 또는 Route Table 설정 오류

**확인**:
```powershell
# NAT Gateway 상태
aws ec2 describe-nat-gateways `
  --filter "Name=tag:Name,Values=min-kyeol-dev-natgw" `
  --query "NatGateways[0].State"

# Route Table 확인
aws ec2 describe-route-tables `
  --filters "Name=tag:Name,Values=min-kyeol-dev-private-rt" `
  --query "RouteTables[0].Routes"
```

**해결**:
```powershell
# NAT Gateway 재생성 (Terraform)
cd kyeol-infra-terraform/envs/dev
terraform taint module.vpc.aws_nat_gateway.main
terraform apply

# 또는 수동 생성 후 Route Table 수정
aws ec2 create-route `
  --route-table-id rtb-xxxxx `
  --destination-cidr-block 0.0.0.0/0 `
  --nat-gateway-id nat-xxxxx
```

**재발 방지**:
- NAT Gateway는 Regional NAT 1개로 비용 절감하되, PROD는 AZ별 NAT 권장
- CloudWatch Alarm으로 NAT Gateway 모니터링

---

### 1.4 CloudFront 503 에러

**증상**: CloudFront URL 접근 시 503 Service Unavailable

**원인**: Origin (ALB) 응답 없음 또는 보안 그룹 차단

**확인**:
```powershell
# CloudFront Origin 확인
aws cloudfront get-distribution --id E1234567890ABC --query "Distribution.DistributionConfig.Origins"

# ALB Health Check
$ALB_DNS = (kubectl get ingress -n saleor -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' --context=dev)
nslookup $ALB_DNS

# ALB Target Group Health
aws elbv2 describe-target-health --target-group-arn [TG_ARN]
```

**해결**:
```powershell
# 1. ALB Security Group에 CloudFront IP 대역 허용
# CloudFront Managed Prefix List 사용
aws ec2 describe-managed-prefix-lists --filters "Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing"

# 2. Target Group에 정상 타겟 있는지 확인
kubectl get pods -n saleor --context=dev -o wide

# 3. CloudFront Cache 무효화
aws cloudfront create-invalidation --distribution-id E1234567890ABC --paths "/*"
```

**재발 방지**:
- Terraform에서 ALB SG에 CloudFront Prefix List 자동 추가
- ALB Health Check 설정 최적화 (interval: 30s, timeout: 5s, healthy threshold: 2)

---

## 2. 플랫폼 레이어

### 2.1 ALB가 생성되지 않음

**증상**: Ingress 적용 후 수 분이 지나도 ALB가 AWS 콘솔에 나타나지 않음

**원인**: AWS Load Balancer Controller가 실행되지 않음 또는 권한 부족

**확인**:
```powershell
# Controller Pod 상태
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --context=dev

# Controller 로그
kubectl logs -n kube-system deploy/aws-load-balancer-controller --tail=100 --context=dev

# IRSA 확인
kubectl get sa aws-load-balancer-controller -n kube-system --context=dev -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'
```

**해결**:
```powershell
# 1. Controller 재배포
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system --context=dev

# 2. IAM Role Policy 확인
aws iam get-role-policy `
  --role-name min-kyeol-dev-alb-controller-role `
  --policy-name ALBControllerPolicy

# 3. Ingress Annotation 확인
kubectl get ingress -n saleor --context=dev -o yaml
# alb.ingress.kubernetes.io/scheme: internet-facing (필수)
# alb.ingress.kubernetes.io/target-type: ip (필수)
```

**재발 방지**:
- kyeol-platform-gitops에서 ALB Controller Helm Chart 버전 고정
- Health Check Probe 설정으로 Controller 가용성 모니터링

---

### 2.2 Ingress ADDRESS 없음

**증상**: `kubectl get ingress` 시 ADDRESS 필드가 비어있음

**원인**: Subnet Tag 누락

**확인**:
```powershell
# Public Subnet Tag 확인
aws ec2 describe-subnets --filters "Name=tag:Name,Values=min-kyeol-dev-public-*" --query "Subnets[*].Tags"
# 필수 Tag: kubernetes.io/role/elb = 1
# 필수 Tag: kubernetes.io/cluster/min-kyeol-dev-eks = owned

# Private Subnet Tag 확인
aws ec2 describe-subnets --filters "Name=tag:Name,Values=min-kyeol-dev-private-*" --query "Subnets[*].Tags"
# 필수 Tag: kubernetes.io/role/internal-elb = 1
```

**해결**:
```powershell
# Terraform에서 자동 설정됨 (modules/vpc/main.tf 확인)
# Tag가 없다면 수동 추가:
aws ec2 create-tags `
  --resources subnet-xxxxx `
  --tags Key=kubernetes.io/role/elb,Value=1 Key=kubernetes.io/cluster/min-kyeol-dev-eks,Value=owned
```

**재발 방지**:
- VPC 모듈에 Subnet Tag 자동 설정
- 배포 후 Tag 검증 스크립트 실행

---

### 2.3 ExternalDNS 미작동

**증상**: Ingress 생성 후 Route53 레코드가 자동으로 생성되지 않음

**원인**: ExternalDNS IRSA 권한 부족 또는 txtOwnerId 충돌

**확인**:
```powershell
# ExternalDNS 로그
kubectl logs -n kube-system -l app.kubernetes.io/name=external-dns --tail=100 --context=dev

# 권한 확인
kubectl get sa external-dns -n kube-system --context=dev -o yaml

# Ingress Annotation 확인
kubectl get ingress -n saleor --context=dev -o yaml | Select-String "external-dns"
# external-dns.alpha.kubernetes.io/hostname: origin-dev-kyeol.click
```

**해결**:
```powershell
# 1. ExternalDNS Pod 재시작
kubectl rollout restart deployment external-dns -n kube-system --context=dev

# 2. IAM Policy 확인
aws iam get-role-policy `
  --role-name min-kyeol-dev-external-dns-role `
  --policy-name ExternalDNSPolicy

# 3. txtOwnerId 확인 (환경별로 달라야 함)
# dev: min-kyeol-dev-externaldns
# stage: min-kyeol-stage-externaldns
```

**재발 방지**:
- 환경별 txtOwnerId 고유값 사용
- ExternalDNS Helm Chart values에 domainFilters 명확히 설정

---

### 2.4 Pod ImagePullBackOff

**증상**: Pod가 `ImagePullBackOff` 또는 `ErrImagePull` 상태

**원인**: ECR 이미지 없음, 태그 불일치, ECR 권한 부족

**확인**:
```powershell
# Pod 이벤트 확인
kubectl describe pod -n saleor saleor-api-[pod-id] --context=dev | Select-String "Failed to pull image"

# ECR 이미지 확인
aws ecr describe-images `
  --repository-name min-kyeol-dev-saleor-api `
  --query "imageDetails[*].imageTags" `
  --output table

# kustomization.yaml 이미지 태그 확인
cat kyeol-app-gitops/overlays/dev/kustomization.yaml
```

**해결**:
```powershell
# 1. 이미지가 없다면 빌드 & Push
cd kyeol-dashboard
docker build -t 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/min-kyeol-dev-saleor-dashboard:latest .
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com
docker push 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/min-kyeol-dev-saleor-dashboard:latest

# 2. kustomization.yaml 이미지 태그 수정
# images:
#   - name: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/min-kyeol-dev-saleor-api
#     newTag: latest  # 또는 특정 SHA

# 3. Git Push → ArgoCD Sync
```

**재발 방지**:
- GitHub Actions CI/CD에서 이미지 빌드 자동화
- ECR Lifecycle Policy로 오래된 이미지 자동 삭제

---

### 2.5 Pod CrashLoopBackOff

**증상**: Pod가 반복적으로 재시작

**원인**: 애플리케이션 시작 실패 (환경 변수 누락, DB 연결 실패 등)

**확인**:
```powershell
# 이전 Pod 로그 확인 (재시작 전)
kubectl logs -n saleor saleor-api-[pod-id] --previous --context=dev --tail=100

# Pod 환경 변수 확인
kubectl describe pod -n saleor saleor-api-[pod-id] --context=dev | Select-String "Environment:"

# Pod Events
kubectl get events -n saleor --context=dev --sort-by='.lastTimestamp'
```

**해결**:
```powershell
# 1. 누락된 환경 변수 확인
# ConfigMap/Secret에 필수 변수가 있는지 확인

# 2. DB 연결 테스트
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- env | grep DATABASE_URL
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- python manage.py dbshell

# 3. Liveness/Readiness Probe 조정
# initialDelaySeconds를 늘려 시작 시간 확보
```

**재발 방지**:
- Deployment에 Init Container 추가 (DB 연결 대기)
- Liveness Probe `failureThreshold` 증가

---

### 2.6 Pod Pending 상태

**증상**: Pod가 Pending 상태로 유지되며 스케줄링되지 않음

**원인**: 노드 리소스 부족, Affinity 규칙 불일치

**확인**:
```powershell
# Pod Events
kubectl describe pod -n saleor saleor-api-[pod-id] --context=dev | Select-String "Events:" -Context 0,10
# "Insufficient cpu" 또는 "Insufficient memory" 메시지 확인

# 노드 리소스 확인
kubectl top nodes --context=dev
kubectl describe nodes --context=dev | Select-String "Allocated resources:" -Context 0,10
```

**해결**:
```powershell
# Option 1: 노드 Auto Scaling (Terraform)
# eks_node_max_size 증가

# Option 2: Pod Resource Request 감소
# resources:
#   requests:
#     cpu: 100m  # 500m → 100m
#     memory: 256Mi  # 512Mi → 256Mi

# Option 3: 기존 Pod 정리
kubectl delete pod -n saleor [불필요한-pod] --context=dev
```

**재발 방지**:
- Cluster Autoscaler 활성화
- Pod Resource Request/Limit 최적화
- Horizontal Pod Autoscaler (HPA) 설정

---

## 3. 애플리케이션 레이어

### 3.1 API 500 Internal Server Error

**증상**: Saleor API 호출 시 500 에러

**원인**: Django 애플리케이션 에러, DB 연결 실패, Secret Key 누락

**확인**:
```powershell
# API Pod 로그
kubectl logs -n saleor -l app=saleor-api --tail=100 --context=dev | Select-String "ERROR|Traceback"

# Health Endpoint 확인
kubectl exec -n saleor saleor-api-[pod-id] --context=dev -- curl -s http://localhost:8000/health/
```

**해결**:
```powershell
# 1. 환경 변수 확인
kubectl get configmap saleor-api-config -n saleor --context=dev -o yaml
kubectl get secret saleor-app-secret -n saleor --context=dev -o jsonpath='{.data.SECRET_KEY}' | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# 2. DB 연결 테스트
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- python manage.py check --database default

# 3. Django Debug 모드 임시 활성화 (DEV 환경만)
# DEBUG=True 환경 변수 추가 → Pod 재시작 → 에러 상세 확인
```

**재발 방지**:
- Sentry 또는 CloudWatch Logs Insights로 에러 모니터링
- Health Check Endpoint 개선 (DB 연결 체크 포함)

---

### 3.2 Storefront가 API에 연결 안 됨

**증상**: Storefront에서 상품 목록이 표시되지 않음, GraphQL 에러

**원인**: `NEXT_PUBLIC_API_URI` 환경 변수 오류, CORS 설정 누락

**확인**:
```powershell
# Storefront 환경 변수 확인
kubectl describe pod -n saleor saleor-storefront-[pod-id] --context=dev | Select-String "NEXT_PUBLIC_API_URI"

# API Ingress 주소 확인
kubectl get ingress -n saleor --context=dev
# origin-dev-kyeol.click 확인

# CORS 설정 확인 (API ConfigMap)
kubectl get configmap saleor-api-config -n saleor --context=dev -o yaml | Select-String "ALLOWED_CLIENT_HOSTS"
```

**해결**:
```powershell
# 1. Storefront ConfigMap 수정
# NEXT_PUBLIC_API_URI=https://origin-dev-kyeol.click/graphql/

# 2. API CORS 설정 추가
# ALLOWED_CLIENT_HOSTS=https://dev-kyeol.click

# 3. Pod 재시작
kubectl rollout restart deployment/saleor-storefront -n saleor --context=dev
kubectl rollout restart deployment/saleor-api -n saleor --context=dev
```

**재발 방지**:
- 환경별 ConfigMap에 올바른 도메인 하드코딩
- 배포 후 연결 테스트 자동화

---

### 3.3 Dashboard 로그인 실패

**증상**: Dashboard에서 로그인 시 "Invalid credentials" 에러

**원인**: Superuser 미생성, 비밀번호 불일치

**확인**:
```powershell
# Django Shell에서 사용자 확인
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- python manage.py shell
```

**Django Shell**:
```python
from django.contrib.auth import get_user_model
User = get_user_model()
User.objects.filter(is_superuser=True)
# 결과가 없으면 Superuser 미생성
```

**해결**:
```powershell
# Superuser 생성
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- python manage.py createsuperuser
# Email: admin@kyeol.click
# Password: [입력]
```

**재발 방지**:
- 초기 배포 시 Superuser 생성 Job 추가
- 비밀번호를 Secrets Manager에 저장

---

### 3.4 이미지 업로드 실패

**증상**: Dashboard에서 상품 이미지 업로드 시 에러

**원인**: S3 버킷 정책 누락, CORS 설정 없음, IAM 권한 부족

**확인**:
```powershell
# S3 버킷 정책
aws s3api get-bucket-policy --bucket min-kyeol-dev-media-123456789012-ap-northeast-2

# CORS 설정
aws s3api get-bucket-cors --bucket min-kyeol-dev-media-123456789012-ap-northeast-2

# IRSA 확인
kubectl get sa saleor-api -n saleor --context=dev -o yaml
```

**해결**:
```powershell
# 1. CORS 설정 추가
aws s3api put-bucket-cors --bucket min-kyeol-dev-media-123456789012-ap-northeast-2 --cors-configuration file://cors.json

# cors.json:
# {
#   "CORSRules": [
#     {
#       "AllowedOrigins": ["https://dev-kyeol.click", "https://dashboard-dev.kyeol.click"],
#       "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
#       "AllowedHeaders": ["*"],
#       "MaxAgeSeconds": 3000
#     }
#   ]
# }

# 2. IAM Policy 확인
aws iam get-role-policy --role-name min-kyeol-dev-saleor-api-role --policy-name S3MediaAccess
```

**재발 방지**:
- Terraform S3 모듈에 CORS 자동 설정
- 배포 후 S3 업로드 테스트 스크립트 실행

---

### 3.5 GraphQL 쿼리 타임아웃

**증상**: Storefront에서 상품 목록 로딩 시간이 10초 이상

**원인**: DB 인덱스 누락, N+1 쿼리, Valkey 캐시 미사용

**확인**:
```powershell
# API Pod 로그 (쿼리 실행 시간)
kubectl logs -n saleor -l app=saleor-api --tail=100 --context=dev | Select-String "duration|ms"

# Valkey 연결 확인
kubectl exec -it -n saleor saleor-api-[pod-id] --context=dev -- env | grep REDIS_URL
```

**해결**:
```powershell
# 1. Valkey 캐시 활성화 확인 (DEV는 미사용, STAGE/PROD만)
# REDIS_URL 환경 변수 설정

# 2. DB 인덱스 추가 (Django Migration)
# python manage.py makemigrations
# python manage.py migrate

# 3. API 쿼리 최적화 (DataLoader 사용)
# Saleor 코드 수정 필요
```

**재발 방지**:
- STAGE/PROD에서 Valkey 필수 사용
- Django Debug Toolbar로 쿼리 분석
- GraphQL 쿼리 깊이 제한 (최대 5단계)

---

## 4. 네트워크 & DNS

### 4.1 ALB 502 Bad Gateway

**증상**: ALB에서 502 에러 반환

**원인**: Target Group Health Check 실패, Pod 미준비

**확인**:
```powershell
# Target Group Health
$TG_ARN = (kubectl get ingress -n saleor --context=dev -o jsonpath='{.metadata.annotations.alb\.ingress\.kubernetes\.io/target-group-arns}')
aws elbv2 describe-target-health --target-group-arn $TG_ARN

# Pod 상태
kubectl get pods -n saleor --context=dev -o wide
```

**해결**:
```powershell
# 1. Health Check 엔드포인트 확인
kubectl exec -n saleor saleor-api-[pod-id] --context=dev -- curl -s http://localhost:8000/health/

# 2. Readiness Probe 조정
# readinessProbe:
#   httpGet:
#     path: /health/
#     port: 8000
#   initialDelaySeconds: 30
#   periodSeconds: 10

# 3. Pod 재배포
kubectl rollout restart deployment/saleor-api -n saleor --context=dev
```

**재발 방지**:
- Liveness/Readiness Probe 최적화
- Target Group Deregistration Delay 조정 (기본: 300초)

---

### 4.2 Route53 레코드 미생성

**증상**: ExternalDNS가 Route53 레코드를 생성하지 않음

**원인**: Hosted Zone ID 불일치, ExternalDNS 권한 부족

**확인**:
```powershell
# ExternalDNS 로그
kubectl logs -n kube-system -l app.kubernetes.io/name=external-dns --tail=100 --context=dev | Select-String "error|failed"

# Ingress Annotation 확인
kubectl get ingress -n saleor --context=dev -o yaml | Select-String "external-dns"
```

**해결**:
```powershell
# 1. ExternalDNS Helm Values 확인
# domainFilters: ["msp-g1.click"]  # 또는 "kyeol.click"

# 2. IAM Policy 확인
aws iam get-role-policy --role-name min-kyeol-dev-external-dns-role --policy-name ExternalDNSPolicy

# 3. 수동 레코드 생성 (임시)
aws route53 change-resource-record-sets --hosted-zone-id Z0XXXXXXXXXXXX --change-batch file://record.json
```

**재발 방지**:
- ExternalDNS domainFilters 명확히 설정
- txtOwnerId 환경별 고유값 사용

---

### 4.3 HTTPS 인증서 오류

**증상**: HTTPS 접속 시 "Your connection is not private" 에러

**원인**: ACM 인증서 검증 미완료, Ingress Annotation 오류

**확인**:
```powershell
# ACM 인증서 상태
aws acm describe-certificate --certificate-arn arn:aws:acm:ap-northeast-2:... --query "Certificate.Status"
# "ISSUED" 여야 함

# Ingress Annotation 확인
kubectl get ingress -n saleor --context=dev -o yaml | Select-String "certificate-arn"
```

**해결**:
```powershell
# 1. ACM 인증서 DNS 검증 레코드 추가
aws acm describe-certificate --certificate-arn [ARN] --query "Certificate.DomainValidationOptions"
# CNAME 레코드를 Route53에 추가

# 2. Ingress Annotation 수정
# alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:...

# 3. ALB Listener 확인
aws elbv2 describe-listeners --load-balancer-arn [ALB_ARN]
```

**재발 방지**:
- ACM 인증서 자동 갱신 (DNS 검증)
- Ingress Template에 ACM ARN 하드코딩

---

### 4.4 CORS 에러

**증상**: 브라우저 콘솔에 "CORS policy: No 'Access-Control-Allow-Origin' header" 에러

**원인**: Saleor API의 `ALLOWED_CLIENT_HOSTS` 설정 누락

**확인**:
```powershell
# API ConfigMap 확인
kubectl get configmap saleor-api-config -n saleor --context=dev -o yaml | grep ALLOWED_CLIENT_HOSTS
```

**해결**:
```powershell
# ConfigMap 수정 (Git)
# ALLOWED_CLIENT_HOSTS: "https://dev-kyeol.click,https://dashboard-dev.kyeol.click"

# Git Push → ArgoCD Sync → Pod 재시작
kubectl rollout restart deployment/saleor-api -n saleor --context=dev
```

**재발 방지**:
- 환경별 ConfigMap에 모든 허용 도메인 명시
- API Health Check에 CORS 테스트 포함

---

## 📚 빠른 참조

### 로그 확인 명령어 모음

```powershell
# Terraform
terraform show
terraform state list

# Kubernetes
kubectl get pods -A --context=[env] -o wide
kubectl logs -n [namespace] [pod-name] --tail=100 -f --context=[env]
kubectl describe pod -n [namespace] [pod-name] --context=[env]
kubectl get events -n [namespace] --context=[env] --sort-by='.lastTimestamp'

# AWS
aws logs tail /aws/eks/min-kyeol-[env]-cluster/logs --follow
aws elbv2 describe-target-health --target-group-arn [TG_ARN]
aws rds describe-db-instances --db-instance-identifier min-kyeol-[env]-rds
```

### 긴급 복구 체크리스트

- [ ] ALB Health Check 확인
- [ ] Pod 상태 확인 (Running, Ready)
- [ ] RDS 연결 테스트
- [ ] S3 버킷 접근 권한 확인
- [ ] CloudWatch Logs 확인
- [ ] Terraform State Lock 해제
- [ ] ArgoCD Sync 상태 확인

---

**최종 업데이트**: 2026-01-20
**작성자**: KYEOL DevOps Team
**관련 문서**:
- [runbook/runbook-infra.md](./runbook/runbook-infra.md)
- [runbook/runbook-platform.md](./runbook/runbook-platform.md)
- [runbook/runbook-ops.md](./runbook/runbook-ops.md)
