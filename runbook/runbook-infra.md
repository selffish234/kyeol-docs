# KYEOL Infrastructure Runbook

> **대상**: 인프라 레이어 (Terraform 관리 리소스)
> **관리**: VPC, EKS, RDS, Redis(Valkey), S3, ECR, IAM, WAF, CloudFront
> **작성일**: 2026-01-20
> **버전**: 3.0

---

## 📌 핵심 원칙

### ❌ 절대 금지 사항
1. **Terraform으로 ALB 직접 생성 금지**
   - ALB는 EKS + AWS Load Balancer Controller + Ingress로만 생성
   - 이유: Ingress와 생명주기 동기화, GitOps 친화적

2. **프로덕션 직접 배포 금지**
   - 반드시 STAGE 환경 검증 후 PROD 배포
   - Terraform plan 없이 apply 금지

3. **tfstate 파일 수동 편집 금지**
   - S3 백엔드를 사용하여 팀 협업
   - DynamoDB로 동시 실행 방지

### ✅ 필수 준수 사항
- 모든 리소스에 `min-kyeol-[env]-` prefix 사용
- 환경별 변수 파일(`terraform.tfvars`) 분리
- 변경 전 `terraform plan` 리뷰 필수
- 민감 정보는 Secrets Manager 사용

---

## 🏗️ 환경별 구성

### 리전 & VPC CIDR

| 환경 | 리전 | VPC CIDR | 용도 |
|------|------|----------|------|
| **MGMT** | ap-northeast-2 | 10.0.0.0/16 | ArgoCD, 관리 도구 |
| **DEV** | ap-northeast-2 | 10.10.0.0/16 | 개발/테스트 |
| **STAGE** | ap-northeast-2 | 10.20.0.0/16 | 스테이징/QA |
| **PROD** | ap-northeast-2 | 10.30.0.0/16 | 프로덕션 |

### 리소스 사이징

#### EKS 노드 (Managed Node Group)
| 환경 | Instance Type | Desired | Min | Max |
|------|--------------|---------|-----|-----|
| DEV | t3.medium | 2 | 1 | 2 |
| STAGE | t3.medium | 2 | 2 | 4 |
| PROD | t3.medium | 3 | 2 | 5 |
| MGMT | t3.medium | 2 | 1 | 3 |

#### RDS PostgreSQL
| 환경 | Instance Class | Multi-AZ | 백업 보관 기간 |
|------|---------------|----------|--------------|
| DEV | db.t3.small | ❌ | 7일 |
| STAGE | db.t3.medium | ✅ | 14일 |
| PROD | db.t3.medium | ✅ | 30일 |

#### ElastiCache (Valkey/Redis)
| 환경 | Node Type | 사용 여부 | 구성 |
|------|----------|---------|------|
| DEV | - | ❌ | 사용 안 함 |
| STAGE | cache.t3.small | ✅ | 단일 노드 |
| PROD | cache.t3.small | ✅ | 단일 노드 |

---

## 📂 디렉토리 구조

```
kyeol-infra-terraform/
├── modules/
│   ├── vpc/              # VPC, Subnet, NAT, Endpoints
│   ├── eks/              # EKS Cluster, Node Group, IRSA
│   ├── rds_postgres/     # RDS PostgreSQL
│   ├── valkey/           # ElastiCache Valkey
│   ├── ecr/              # ECR Repository
│   ├── s3/               # S3 Buckets
│   ├── waf/              # WAF Web ACL
│   ├── cloudfront/       # CloudFront Distribution
│   └── cloudtrail/       # CloudTrail
├── envs/
│   ├── bootstrap/        # tfstate S3 bucket 생성
│   ├── dev/              # DEV 환경
│   ├── stage/            # STAGE 환경
│   └── prod/             # PROD 환경
└── README.md
```

---

## 🚀 Phase 1: Bootstrap & 기본 인프라

### 1.1 사전 준비

**필수 도구 설치 확인**:
```powershell
aws --version        # AWS CLI 2.x
terraform --version  # Terraform 1.5.0+
kubectl version --client
```

**AWS 자격증명 설정**:
```powershell
aws configure
# AWS Access Key ID: [your-access-key]
# AWS Secret Access Key: [your-secret-key]
# Default region name: ap-northeast-2
```

**확인**:
```powershell
aws sts get-caller-identity
```

### 1.2 Bootstrap (tfstate 저장소 생성)

**목적**: Terraform 상태 파일을 저장할 S3 bucket과 DynamoDB lock 테이블 생성

```powershell
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project\kyeol-infra-terraform\envs\bootstrap

# terraform.tfvars 설정
Copy-Item terraform.tfvars.example terraform.tfvars
# 편집: aws_account_id, aws_region 입력

# 적용
terraform init
terraform plan
terraform apply
```

**생성 리소스**:
- S3 bucket: `min-kyeol-tfstate-[account-id]`
- DynamoDB table: `min-kyeol-terraform-locks`

**검증**:
```powershell
aws s3 ls | Select-String "kyeol-tfstate"
aws dynamodb list-tables | Select-String "terraform-locks"
```

### 1.3 DEV 환경 배포

```powershell
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project\kyeol-infra-terraform\envs\dev

# terraform.tfvars 설정
Copy-Item terraform.tfvars.example terraform.tfvars
# 필수 변수 입력:
#   - aws_account_id
#   - hosted_zone_id (Route53)
#   - acm_certificate_arn (ap-northeast-2)

# 초기화 (S3 백엔드 설정)
terraform init

# 계획 확인
terraform plan

# 적용
terraform apply
```

**생성 리소스** (약 15-20분 소요):
- VPC: `min-kyeol-dev-vpc`
- EKS Cluster: `min-kyeol-dev-eks`
- RDS Instance: `min-kyeol-dev-rds`
- ECR Repository: `min-kyeol-dev-*`
- NAT Gateway: `min-kyeol-dev-natgw`
- IAM Roles: IRSA for ALB Controller, ExternalDNS, Fluent Bit

**검증**:
```powershell
# VPC 확인
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=min-kyeol-dev-vpc"

# EKS 클러스터 확인
aws eks describe-cluster --name min-kyeol-dev-eks --region ap-northeast-2

# RDS 인스턴스 확인
aws rds describe-db-instances --db-instance-identifier min-kyeol-dev-rds
```

### 1.4 kubeconfig 설정

```powershell
aws eks update-kubeconfig `
  --region ap-northeast-2 `
  --name min-kyeol-dev-eks `
  --alias dev

# 확인
kubectl config get-contexts
kubectl cluster-info
kubectl get nodes
```

---

## 🔧 Phase 2: STAGE/PROD 환경 확장

### 2.1 STAGE 환경 배포

```powershell
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project\kyeol-infra-terraform\envs\stage

Copy-Item terraform.tfvars.example terraform.tfvars
# 변수 설정 (DEV와 동일한 패턴)

terraform init
terraform plan
terraform apply
```

**주요 차이점**:
- RDS Multi-AZ 활성화
- ElastiCache Valkey 추가
- 백업 보관 기간 14일
- Auto Scaling 최대값 증가

**kubeconfig 추가**:
```powershell
aws eks update-kubeconfig `
  --region ap-northeast-2 `
  --name min-kyeol-stage-eks `
  --alias stage
```

### 2.2 PROD 환경 배포

**⚠️ STAGE 검증 완료 후에만 진행**

```powershell
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project\kyeol-infra-terraform\envs\prod

Copy-Item terraform.tfvars.example terraform.tfvars
# PROD 변수 설정

terraform init
terraform plan > plan.out  # 계획 저장 및 리뷰
# 팀원과 plan.out 공유 및 승인 후 진행

terraform apply
```

**kubeconfig 추가**:
```powershell
aws eks update-kubeconfig `
  --region ap-northeast-2 `
  --name min-kyeol-prod-eks `
  --alias prod
```

---

## 📊 Phase 3: 보안 및 추가 서비스

### 3.1 S3 Buckets

**용도별 S3 버킷**:
| 버킷 이름 | 용도 | 환경 |
|----------|------|------|
| `min-kyeol-[env]-media-[account]-ap-northeast-2` | 미디어/이미지 저장 | 전체 |
| `min-kyeol-[env]-logs-[account]-ap-northeast-2` | 서비스 로그 | 전체 |
| `min-kyeol-prod-audit-[account]-ap-northeast-2` | CloudTrail 감사 로그 | PROD만 |

**배포**:
```powershell
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project\kyeol-infra-terraform\envs\[env]

# terraform.tfvars에 s3_enabled = true 추가
terraform plan
terraform apply
```

**S3 버킷 정책 확인**:
```powershell
aws s3api get-bucket-policy --bucket min-kyeol-prod-media-[account]-ap-northeast-2
```

### 3.2 WAF (Web Application Firewall)

**적용 대상**: ALB (Ingress 생성 후)

**WAF Rule Sets**:
- AWS Managed Rules - Core Rule Set
- AWS Managed Rules - Known Bad Inputs
- Rate Limiting (2000 req/5min per IP)

**배포**:
```powershell
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project\kyeol-infra-terraform\envs\[env]

# terraform.tfvars에 waf_enabled = true 추가
terraform plan
terraform apply
```

**WAF 연결** (ALB 생성 후 수동 연결 필요):
```powershell
# ALB ARN 확인
aws elbv2 describe-load-balancers `
  --query "LoadBalancers[?contains(LoadBalancerName,'k8s-saleor')].LoadBalancerArn" `
  --output text

# WAF ACL ARN 확인
$WAF_ARN = (aws wafv2 list-web-acls `
  --scope REGIONAL `
  --region ap-northeast-2 `
  --query "WebACLs[?Name=='min-kyeol-prod-waf'].ARN" `
  --output text)

# ALB에 WAF 연결
aws wafv2 associate-web-acl `
  --web-acl-arn $WAF_ARN `
  --resource-arn [ALB_ARN] `
  --region ap-northeast-2
```

### 3.3 CloudFront

**환경별 CloudFront 배포판**:
| 환경 | CNAME | Origin |
|------|-------|--------|
| DEV | `dev-kyeol.click` | `origin-dev-kyeol.click` (ALB) |
| STAGE | `stage-kyeol.click` | `origin-stage-kyeol.click` (ALB) |
| PROD | `kyeol.click` | `origin-prod-kyeol.click` (ALB) |

**사전 요구사항**:
- **us-east-1** 리전의 ACM 인증서 발급 (CloudFront용)
- ALB가 생성되어 Origin 도메인이 존재해야 함

**배포**:
```powershell
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project\kyeol-infra-terraform\envs\[env]

# terraform.tfvars에 추가:
#   cloudfront_enabled = true
#   acm_certificate_arn_us_east_1 = "arn:aws:acm:us-east-1:..."

terraform plan
terraform apply
```

**CloudFront 배포 확인**:
```powershell
aws cloudfront list-distributions `
  --query "DistributionList.Items[?contains(Comment,'kyeol-prod')].DomainName"
```

### 3.4 CloudTrail (PROD만)

**목적**: API 감사 로그 기록 (ISMS-P 준수)

**배포**:
```powershell
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project\kyeol-infra-terraform\envs\prod

# terraform.tfvars에 cloudtrail_enabled = true 추가
terraform plan
terraform apply
```

**CloudTrail 이벤트 조회**:
```powershell
aws cloudtrail lookup-events `
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances `
  --max-items 10
```

---

## 🔐 IAM & IRSA (IAM Roles for Service Accounts)

### IRSA 역할 목록

| Service Account | IAM Role | 권한 |
|----------------|----------|------|
| `aws-load-balancer-controller` | `min-kyeol-[env]-alb-controller-role` | ELB 관리 (ALB/NLB 생성/삭제) |
| `external-dns` | `min-kyeol-[env]-external-dns-role` | Route53 관리 (DNS 레코드 자동 생성) |
| `fluent-bit` | `min-kyeol-[env]-fluent-bit-role` | CloudWatch Logs 쓰기 권한 |
| `saleor-api` | `min-kyeol-[env]-saleor-api-role` | S3 미디어 버킷 읽기/쓰기 |

### IRSA 동작 확인

```powershell
# 1. OIDC Provider 확인
aws eks describe-cluster `
  --name min-kyeol-[env]-eks `
  --query "cluster.identity.oidc.issuer" `
  --output text

# 2. IAM Role 확인
aws iam get-role --role-name min-kyeol-[env]-alb-controller-role

# 3. Service Account Annotation 확인
kubectl get sa aws-load-balancer-controller `
  -n kube-system `
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'
```

---

## 🛠️ 운영 절차

### RDS 백업/복원

#### 수동 스냅샷 생성
```powershell
aws rds create-db-snapshot `
  --db-instance-identifier min-kyeol-[env]-rds `
  --db-snapshot-identifier min-kyeol-[env]-manual-$(Get-Date -Format "yyyyMMdd-HHmmss")
```

#### 스냅샷 목록 확인
```powershell
aws rds describe-db-snapshots `
  --db-instance-identifier min-kyeol-[env]-rds `
  --query "DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime,Status]" `
  --output table
```

#### 스냅샷에서 복원
```powershell
aws rds restore-db-instance-from-db-snapshot `
  --db-instance-identifier min-kyeol-[env]-rds-restored `
  --db-snapshot-identifier [snapshot-id] `
  --db-instance-class db.t3.medium
```

### CloudFront 캐시 무효화

```powershell
# Distribution ID 확인
$DIST_ID = (aws cloudfront list-distributions `
  --query "DistributionList.Items[?contains(Comment,'kyeol-prod')].Id" `
  --output text)

# 전체 캐시 무효화
aws cloudfront create-invalidation `
  --distribution-id $DIST_ID `
  --paths "/*"

# 특정 경로만 무효화
aws cloudfront create-invalidation `
  --distribution-id $DIST_ID `
  --paths "/static/*" "/media/*"
```

### Terraform 상태 관리

#### 상태 목록 조회
```powershell
terraform state list
```

#### 특정 리소스 상태 확인
```powershell
terraform state show module.vpc.aws_vpc.main
```

#### 리소스 import (수동 생성된 리소스를 Terraform 관리로 전환)
```powershell
terraform import aws_security_group.example sg-0123456789abcdef
```

### 비용 모니터링

#### 환경별 비용 확인
```powershell
# Cost Explorer API 사용
aws ce get-cost-and-usage `
  --time-period Start=2026-01-01,End=2026-01-31 `
  --granularity MONTHLY `
  --metrics "UnblendedCost" `
  --group-by Type=TAG,Key=Environment `
  --filter file://filter.json
```

**filter.json**:
```json
{
  "Tags": {
    "Key": "Environment",
    "Values": ["dev", "stage", "prod"]
  }
}
```

---

## ⚠️ 문제 해결

### Terraform Lock 충돌

**증상**: `Error acquiring the state lock`

**원인**: 이전 terraform 실행이 비정상 종료되어 DynamoDB lock이 남아있음

**해결**:
```powershell
# Lock ID 확인 (에러 메시지에서 복사)
aws dynamodb get-item `
  --table-name min-kyeol-terraform-locks `
  --key '{"LockID":{"S":"[env]/[resource]/terraform.tfstate-md5"}}'

# Lock 강제 해제 (주의: 다른 사람이 실행 중이 아닌지 확인 후)
terraform force-unlock [LOCK_ID]
```

### EKS 노드 연결 실패

**증상**: `kubectl get nodes` 시 노드가 NotReady 상태

**확인**:
```powershell
# 노드 상세 정보
kubectl describe node [node-name]

# kubelet 로그 (EC2 인스턴스 접속 필요)
# Systems Manager Session Manager 사용
aws ssm start-session --target [instance-id]
sudo journalctl -u kubelet -f
```

**일반적인 원인**:
- IAM Role이 EKS 클러스터에 매핑되지 않음
- VPC CNI Plugin 문제
- Security Group 규칙 누락

**해결**:
```powershell
# aws-auth ConfigMap 확인
kubectl get configmap aws-auth -n kube-system -o yaml

# VPC CNI 재시작
kubectl rollout restart daemonset aws-node -n kube-system
```

### RDS 연결 실패

**증상**: 애플리케이션에서 DB 연결 에러

**확인**:
```powershell
# RDS 엔드포인트 확인
aws rds describe-db-instances `
  --db-instance-identifier min-kyeol-[env]-rds `
  --query "DBInstances[0].Endpoint.Address" `
  --output text

# Security Group 규칙 확인
aws ec2 describe-security-groups `
  --group-ids [rds-sg-id] `
  --query "SecurityGroups[0].IpPermissions"
```

**해결**:
1. RDS Security Group에 EKS Node Security Group 추가
2. Secrets Manager에서 DB 비밀번호 확인
3. RDS 파라미터 그룹 확인 (`max_connections` 등)

---

## 📚 참고

### 환경별 Terraform 변수 예시

**dev/terraform.tfvars**:
```hcl
aws_account_id = "123456789012"
aws_region = "ap-northeast-2"
environment = "dev"
hosted_zone_id = "Z0XXXXXXXXXXXX"
acm_certificate_arn = "arn:aws:acm:ap-northeast-2:..."

# VPC
vpc_cidr = "10.10.0.0/16"

# EKS
eks_node_instance_type = "t3.medium"
eks_node_desired_size = 2
eks_node_min_size = 1
eks_node_max_size = 2

# RDS
rds_instance_class = "db.t3.small"
rds_multi_az = false
rds_backup_retention_period = 7

# Valkey
valkey_enabled = false
```

**prod/terraform.tfvars**:
```hcl
aws_account_id = "123456789012"
aws_region = "ap-northeast-2"
environment = "prod"
hosted_zone_id = "Z0XXXXXXXXXXXX"
acm_certificate_arn = "arn:aws:acm:ap-northeast-2:..."
acm_certificate_arn_us_east_1 = "arn:aws:acm:us-east-1:..."  # CloudFront용

# VPC
vpc_cidr = "10.30.0.0/16"

# EKS
eks_node_instance_type = "t3.medium"
eks_node_desired_size = 3
eks_node_min_size = 2
eks_node_max_size = 5

# RDS
rds_instance_class = "db.t3.medium"
rds_multi_az = true
rds_backup_retention_period = 30

# Valkey
valkey_enabled = true
valkey_node_type = "cache.t3.small"

# 보안
waf_enabled = true
cloudtrail_enabled = true
cloudfront_enabled = true
```

---

**최종 업데이트**: 2026-01-20
**작성자**: KYEOL DevOps Team
**다음 문서**: [runbook-platform.md](./runbook-platform.md) - 플랫폼 레이어 운영 가이드
