# GitHub Organization 이관 가이드

> **작성일**: 2026-01-07  
> **목적**: 인프라 코드를 GitHub Organization으로 이관하기 위한 파트별 상세 가이드

---

## 1. 현재 레포 구조

```
kyeol-infra-terraform/
├── .gitignore
├── README.md
├── trust-policy.json
│
├── modules/                    # Terraform 모듈
│   ├── vpc/                   # Network 파트
│   ├── vpc_peering/           # Network 파트
│   ├── eks/                   # Compute 파트
│   ├── rds_postgres/          # Data 파트
│   ├── valkey/                # Data 파트
│   ├── s3/                    # Data 파트
│   ├── ecr/                   # Data 파트
│   ├── cloudtrail/            # Security/Obs 파트
│   ├── waf/                   # Security/Obs 파트
│   ├── waf_global/            # Security/Obs 파트
│   ├── log_analytics/         # Security/Obs 파트
│   ├── cloudfront/            # Security/Obs 파트
│   └── lambda_edge/           # Security/Obs 파트
│
├── envs/                       # 환경별 루트
│   ├── bootstrap/
│   ├── dev/
│   ├── stage/
│   ├── prod/
│   └── mgmt/
│
└── global/                     # 글로벌 설정
```

---

## 2. 파트별 커밋 대상

### 🔵 A. Network 파트

**담당자**: A  
**범위**: VPC, Subnet, NAT, Peering, 라우팅

#### 커밋 대상 폴더/파일

```
modules/
├── vpc/
│   ├── main.tf
│   ├── nat.tf
│   ├── route_tables.tf
│   ├── igw.tf
│   ├── endpoints.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
└── vpc_peering/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── versions.tf
```

#### 환경별 파일 (Network 담당 부분만)

```
envs/dev/main.tf       → module "vpc" 블록, module "vpc_peering" 블록
envs/stage/main.tf     → module "vpc" 블록, module "vpc_peering" 블록
envs/prod/main.tf      → module "vpc" 블록, module "vpc_peering" 블록
envs/mgmt/main.tf      → module "vpc" 블록
envs/*/variables.tf    → VPC 관련 변수 (vpc_cidr, azs, subnet_cidrs 등)
envs/*/outputs.tf      → VPC 관련 출력 (vpc_id, subnet_ids 등)
```

#### 커밋 메시지 예시 (한글)

```
feat(network): VPC 모듈 추가 - 서브넷/라우팅/NAT 포함

feat(network): VPC Peering 모듈 추가 - MGMT-DEV/STAGE/PROD 연결

feat(network): DEV 환경 VPC 설정 추가

fix(network): NAT Gateway 라우팅 테이블 수정

chore(network): VPC 모듈 변수 기본값 정리
```

---

### 🟢 B. Compute 파트

**담당자**: B  
**범위**: EKS, Node Group, IRSA, 보안그룹

#### 커밋 대상 폴더/파일

```
modules/
└── eks/
    ├── main.tf
    ├── nodegroups.tf
    ├── iam_irsa.tf
    ├── oidc.tf
    ├── fluent_bit_irsa.tf
    ├── fluent_bit_outputs.tf
    ├── fluent_bit_variables.tf
    ├── variables.tf
    ├── outputs.tf
    └── versions.tf
```

#### 환경별 파일 (Compute 담당 부분만)

```
envs/dev/main.tf       → module "eks" 블록
envs/stage/main.tf     → module "eks" 블록
envs/prod/main.tf      → module "eks" 블록
envs/mgmt/main.tf      → module "eks" 블록
envs/*/variables.tf    → EKS 관련 변수 (cluster_version, node_instance_types 등)
envs/*/outputs.tf      → EKS 관련 출력 (cluster_name, cluster_endpoint 등)
```

#### 커밋 메시지 예시 (한글)

```
feat(compute): EKS 모듈 추가 - 클러스터 및 Node Group 설정

feat(compute): IRSA 설정 추가 - ALB Controller, External DNS

feat(compute): Fluent Bit IRSA 추가 - CloudWatch Logs 연동

fix(compute): EKS Private Endpoint 보안그룹 규칙 수정

chore(compute): EKS 버전 1.32로 업그레이드
```

---

### 🟠 C. Data 파트

**담당자**: C  
**범위**: RDS, Valkey, S3, ECR, 백업 정책

#### 커밋 대상 폴더/파일

```
modules/
├── rds_postgres/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
├── valkey/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
├── s3/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
└── ecr/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── versions.tf
```

#### 환경별 파일 (Data 담당 부분만)

```
envs/dev/main.tf       → module "rds", module "s3", module "ecr" 블록
envs/stage/main.tf     → module "rds", module "valkey", module "s3" 블록
envs/prod/main.tf      → module "rds", module "valkey", module "s3" 블록
envs/mgmt/main.tf      → module "s3" 블록 (로그 버킷)
envs/*/variables.tf    → RDS/Valkey/S3 관련 변수
envs/*/outputs.tf      → RDS/Valkey/S3 관련 출력
```

#### 커밋 메시지 예시 (한글)

```
feat(data): RDS PostgreSQL 모듈 추가 - Multi-AZ, 암호화 설정

feat(data): Valkey(Redis) 모듈 추가 - 클러스터 모드

feat(data): S3 모듈 추가 - 로그 버킷, 리포트 버킷

feat(data): ECR 레포지토리 모듈 추가

fix(data): RDS 백업 보존 기간 ISMS-P 기준으로 수정
```

---

### 🔴 D. Security/Obs 파트

**담당자**: D  
**범위**: CloudTrail, WAF, 로그분석, 알람, CloudFront, Lambda@Edge

#### 커밋 대상 폴더/파일

```
modules/
├── cloudtrail/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
├── waf/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
├── waf_global/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
├── log_analytics/
│   ├── main.tf
│   ├── iam.tf
│   ├── lambda.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── lambda_code/
│       ├── report_generator/handler.py
│       └── realtime_alert/handler.py
│
├── cloudfront/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
└── lambda_edge/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── src/
        ├── origin-response.js
        ├── viewer-request.js
        └── package.json
```

#### 환경별 파일 (Security/Obs 담당 부분만)

```
envs/mgmt/main.tf      → module "cloudtrail", module "waf_global", 
                         module "cloudfront", module "log_analytics" 블록
envs/prod/main.tf      → module "waf" 블록
envs/stage/main.tf     → module "waf" 블록
envs/dev/main.tf       → module "waf" 블록 (선택)
envs/*/variables.tf    → WAF/CloudTrail/Log Analytics 관련 변수
```

#### 커밋 메시지 예시 (한글)

```
feat(security): CloudTrail 모듈 추가 - 중앙 수집 구조

feat(security): WAF Regional 모듈 추가 - OWASP Top 10 규칙

feat(security): Global WAF + CloudFront 모듈 추가

feat(obs): 로그 분석 파이프라인 추가 - Bedrock AI 연동

feat(obs): Lambda@Edge 이미지 리사이징 추가
```

---

## 3. 공통 파일 오너십

### 루트 레벨 공통 파일

| 파일 | 오너 | 비고 |
|------|------|-----|
| `README.md` | **D (Security/Obs)** | 최종 통합 후 정리 |
| `.gitignore` | **D (Security/Obs)** | 최초 1회 커밋 |
| `trust-policy.json` | **B (Compute)** | IRSA 관련 |

### 환경별 공통 파일

| 파일 | 오너 | 비고 |
|------|------|-----|
| `envs/*/providers.tf` | **A (Network)** | VPC 먼저 생성하므로 |
| `envs/*/versions.tf` | **A (Network)** | 동일 이유 |
| `envs/*/backend.tf` | **A (Network)** | S3 백엔드 설정 |
| `envs/*/terraform.tfvars.example` | **각 파트 순차** | 자기 파트 변수만 추가 |

---

## 4. 이관 커밋 순서 (의존성 기준)

```
┌─────────────────────────────────────────────────────────────────┐
│                    이관 커밋 순서 (의존성)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1️⃣ A. Network (VPC/Subnet/NAT)                                 │
│         └─ 모든 리소스가 VPC에 의존                               │
│                                                                  │
│  2️⃣ B. Compute (EKS/Node Group/IRSA)                            │
│         └─ VPC, Subnet에 의존                                    │
│                                                                  │
│  3️⃣ C. Data (RDS/Valkey/S3)                                     │
│         └─ VPC, Subnet에 의존                                    │
│         └─ EKS IRSA에 일부 의존 (S3 접근)                        │
│                                                                  │
│  4️⃣ D. Security/Obs (CloudTrail/WAF/Log Analytics)              │
│         └─ S3 버킷에 의존 (로그 저장)                            │
│         └─ VPC Endpoint에 의존                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 상세 커밋 순서

#### Phase 1: 기반 설정 (A. Network 담당)

```bash
# 1-1. 루트 파일 커밋
git add .gitignore README.md trust-policy.json
git commit -m "chore: 루트 설정 파일 추가"

# 1-2. VPC 모듈 커밋
git add modules/vpc/
git commit -m "feat(network): VPC 모듈 추가 - 서브넷/라우팅/NAT 포함"

# 1-3. VPC Peering 모듈 커밋
git add modules/vpc_peering/
git commit -m "feat(network): VPC Peering 모듈 추가"

# 1-4. 환경별 VPC 설정 커밋
git add envs/*/providers.tf envs/*/versions.tf envs/*/backend.tf
git commit -m "chore(network): 환경별 기반 설정 추가"
```

#### Phase 2: 컴퓨팅 (B. Compute 담당)

```bash
# 2-1. EKS 모듈 커밋
git add modules/eks/
git commit -m "feat(compute): EKS 모듈 추가 - 클러스터/Node Group/IRSA"
```

#### Phase 3: 데이터 (C. Data 담당)

```bash
# 3-1. RDS 모듈 커밋
git add modules/rds_postgres/
git commit -m "feat(data): RDS PostgreSQL 모듈 추가"

# 3-2. Valkey 모듈 커밋
git add modules/valkey/
git commit -m "feat(data): Valkey(Redis) 모듈 추가"

# 3-3. S3/ECR 모듈 커밋
git add modules/s3/ modules/ecr/
git commit -m "feat(data): S3, ECR 모듈 추가"
```

#### Phase 4: 보안/관제 (D. Security/Obs 담당)

```bash
# 4-1. CloudTrail 모듈 커밋
git add modules/cloudtrail/
git commit -m "feat(security): CloudTrail 모듈 추가 - 중앙 수집"

# 4-2. WAF 모듈 커밋
git add modules/waf/ modules/waf_global/
git commit -m "feat(security): WAF Regional/Global 모듈 추가"

# 4-3. CloudFront/Lambda@Edge 커밋
git add modules/cloudfront/ modules/lambda_edge/
git commit -m "feat(security): CloudFront + Lambda@Edge 이미지 리사이징"

# 4-4. Log Analytics 커밋
git add modules/log_analytics/
git commit -m "feat(obs): 로그 분석 파이프라인 - Bedrock AI 연동"
```

#### Phase 5: 환경별 통합 (각 파트 순차)

```bash
# 5-1. A: 환경별 VPC 관련 main.tf
# 5-2. B: 환경별 EKS 관련 main.tf  
# 5-3. C: 환경별 RDS/S3 관련 main.tf
# 5-4. D: 환경별 보안 관련 main.tf
```

---

## 5. 충돌 방지 가이드

### 5.1 main.tf 분리 커밋 규칙

`envs/*/main.tf`는 **여러 파트가 공유**하므로 충돌 위험이 높습니다.

**규칙:**
1. **한 파트씩 순차적으로** main.tf 수정
2. 수정 전 반드시 `git pull` 실행
3. 자기 담당 모듈 블록만 수정

```hcl
# envs/dev/main.tf 구조

# ===== A. Network 담당 =====
module "vpc" { ... }
module "vpc_peering" { ... }

# ===== B. Compute 담당 =====
module "eks" { ... }

# ===== C. Data 담당 =====
module "rds" { ... }
module "s3" { ... }

# ===== D. Security/Obs 담당 =====
module "waf" { ... }
module "cloudtrail" { ... }
```

### 5.2 variables.tf 분리 규칙

각 파트가 자기 담당 변수만 추가:

```hcl
# A. Network 변수
variable "vpc_cidr" { ... }
variable "azs" { ... }

# B. Compute 변수  
variable "eks_cluster_version" { ... }
variable "eks_node_instance_types" { ... }

# C. Data 변수
variable "rds_instance_class" { ... }
variable "enable_valkey" { ... }

# D. Security/Obs 변수
variable "enable_cloudtrail" { ... }
variable "enable_waf" { ... }
```

### 5.3 PR 순서 규칙

```
1. A (Network) PR → 머지 완료
2. B (Compute) PR → A 머지 후 생성
3. C (Data) PR → B 머지 후 생성  
4. D (Security/Obs) PR → C 머지 후 생성
```

---

## 6. 검증 체크리스트

각 파트 커밋 후 반드시 확인:

- [ ] `terraform fmt -recursive` 실행
- [ ] `terraform validate` 통과
- [ ] `terraform plan` 에러 없음
- [ ] 다른 파트 파일 수정 없음
- [ ] 커밋 메시지 규칙 준수

---

**운영 반영 전 테스트 환경에서 먼저 검증하세요.**
