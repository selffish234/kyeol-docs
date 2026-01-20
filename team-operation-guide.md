# KYEOL 팀 운영 가이드 v1.0

> **Organization**: kyeol-team  
> **작성일**: 2026-01-07  
> **팀 규모**: 4명

---

## 1. GitHub Organization 운영 설계

### 1.1 Organization 권한 구조

| 역할 | 권한 | 담당자 수 |
|-----|------|:--------:|
| **Owner** | 전체 관리, Billing, 설정 | 1명 |
| **Maintainer** | 레포 관리, PR 승인, 브랜치 보호 | 1명 |
| **Member** | Push, PR 생성, 코드 리뷰 | 2명 |

### 1.2 필수 설정

#### Branch Protection Rules (main, develop)

```yaml
# Settings > Branches > Branch protection rules
main:
  require_pull_request_reviews: true
  required_approving_review_count: 1
  dismiss_stale_reviews: true
  require_status_checks: true
  require_branches_up_to_date: true
  include_administrators: true
  allow_force_pushes: false
  allow_deletions: false

develop:
  require_pull_request_reviews: true
  required_approving_review_count: 1
  require_status_checks: false  # CI 구축 전까지
```

#### CODEOWNERS 파일

```
# .github/CODEOWNERS
# 인프라 담당
/kyeol-infra-terraform/modules/vpc/       @network-owner
/kyeol-infra-terraform/modules/eks/       @compute-owner
/kyeol-infra-terraform/modules/rds*/      @data-owner
/kyeol-infra-terraform/modules/log*/      @security-owner

# 플랫폼 담당
/kyeol-platform-gitops/                   @platform-owner

# 전체 리뷰 필요
/kyeol-infra-terraform/envs/prod/         @all-maintainers
```

---

## 2. Repository 구조 전략

### 2.1 권장 레포 구조 (현행 유지 + 문서 통합)

```
kyeol-team/                     # GitHub Organization
├── kyeol-infra-terraform/      # IaC (Terraform)
├── kyeol-platform-gitops/      # Platform addons (ArgoCD/Helm)
├── kyeol-app-gitops/           # App manifests (Kustomize)
├── kyeol-storefront/           # Frontend 앱
├── kyeol-dashboard/            # Admin 앱
└── kyeol-docs/                 # 통합 문서 (신규)
```

### 2.2 레포 네이밍 컨벤션

```
[프로젝트]-[도메인]-[타입]

예시:
kyeol-infra-terraform      # 인프라 IaC
kyeol-platform-gitops      # 플랫폼 GitOps
kyeol-app-gitops           # 앱 GitOps
kyeol-monitoring-config    # 모니터링 설정 (필요 시)
```

---

## 3. GitHub Projects 칸반 운영

### 3.1 칸반 보드 구조

```
┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│  Backlog    │    Todo     │ In Progress │   Review    │    Done     │
├─────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 아이디어/   │ 이번 스프린트│ 현재 작업 중│ PR 오픈 상태│ 머지 완료   │
│ 미래 작업   │ 대기 작업   │ 브랜치 생성 │ 리뷰 대기   │ 배포 완료   │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

### 3.2 Issue → PR 흐름

```
1. Issue 생성 (Backlog)
   └─ 제목: [INFRA] VPC Peering 추가
   └─ 라벨: infra, network, priority:high
   └─ 담당자: @network-owner

2. Todo로 이동 (스프린트 시작)
   └─ 마일스톤: Sprint-01

3. 브랜치 생성 + In Progress
   └─ git checkout -b infra/vpc-peering

4. PR 생성 + Review
   └─ PR 제목: feat(infra): MGMT-DEV VPC Peering 추가
   └─ Issue #123 자동 연결

5. 승인 + 머지 + Done
   └─ Squash Merge 권장
```

### 3.3 마일스톤 운영

| 마일스톤 | 기간 | 목표 |
|---------|-----|------|
| Sprint-01 | 1주차 | Phase 3 인프라 완성 |
| Sprint-02 | 2주차 | Phase 4 로그 파이프라인 |
| Sprint-03 | 3주차 | 앱 배포 + 테스트 |

---

## 4. 4인 업무 분담안

### 4.1 파트 분담 (인프라 중심)

| 파트 | 담당 | 범위 | 주요 파일 |
|------|-----|------|----------|
| **Network** | A | VPC, Subnet, NAT, Peering, 라우팅 | `modules/vpc/*`, `modules/vpc_peering/*` | 
| **Compute** | B | EKS, Node Group, IRSA, 보안그룹 | `modules/eks/*` | 
| **Data** | C | RDS, Valkey, S3, 백업 정책 | `modules/rds_postgres/*`, `modules/valkey/*`, `modules/s3/*` |
| **Security/Obs** | D | CloudTrail, WAF, 로그분석, 알람 | `modules/cloudtrail/*`, `modules/waf*/*`, `modules/log_analytics/*` |

### 4.2 파트별 상세 범위

#### A. Network (네트워크)

```
담당 범위:
- VPC CIDR 설계
- Subnet 분리 (Public/App/Data/Cache/Payment)
- NAT Gateway (일반/결제)
- VPC Peering (MGMT ↔ DEV/STAGE/PROD)
- Route Table 관리
- VPC Endpoints

변경 대상:
modules/vpc/main.tf
modules/vpc/nat.tf
modules/vpc/route_tables.tf
modules/vpc_peering/*
envs/*/main.tf (VPC 모듈 호출부)

충돌 주의:
- CIDR 중복 (다른 파트와 협의)
- EKS 서브넷 태그 (Compute 파트와)

리뷰 포인트:
☐ CIDR 충돌 없음
☐ 라우팅 정합성
☐ NAT 비용 최적화
```

#### B. Compute (컴퓨팅)

```
담당 범위:
- EKS 클러스터 설정
- Node Group (일반/결제)
- IRSA (ALB Controller, External DNS)
- EKS 보안그룹
- Private Endpoint 설정

변경 대상:
modules/eks/main.tf
modules/eks/nodegroups.tf
modules/eks/iam_irsa.tf
envs/*/main.tf (EKS 모듈 호출부)

충돌 주의:
- 서브넷 ID (Network 파트와)
- 보안그룹 (Network 파트와)

리뷰 포인트:
☐ EKS 버전 1.31+
☐ Extended Support 방지
☐ Node 비용 최적화
```

#### C. Data (데이터)

```
담당 범위:
- RDS PostgreSQL 설정
- Valkey (Redis) 설정
- S3 버킷 (로그, 리포트)
- 백업/복구 정책

변경 대상:
modules/rds_postgres/*
modules/valkey/*
modules/s3/*
envs/*/main.tf (RDS/Valkey 호출부)

충돌 주의:
- 서브넷 ID (Network 파트와)
- 보안그룹 (Network 파트와)

리뷰 포인트:
☐ Multi-AZ 설정 (PROD만)
☐ 암호화 활성화
☐ 백업 보존 기간
```

#### D. Security/Observability (보안/관제)

```
담당 범위:
- CloudTrail 중앙 수집
- WAF (Regional/Global)
- 로그 분석 파이프라인
- EventBridge/Lambda/Bedrock
- Slack 알림

변경 대상:
modules/cloudtrail/*
modules/waf/*
modules/waf_global/*
modules/log_analytics/*
envs/mgmt/main.tf (보안 모듈 호출부)

충돌 주의:
- S3 버킷 (Data 파트와)
- IAM 정책 (Compute 파트와)

리뷰 포인트:
☐ ISMS-P 준수
☐ 로그 보존 기간 1년+
☐ 비용 최적화
```

### 4.3 환경별 분담 (교차 리뷰)

| 환경 | 주 담당 | 리뷰어 |
|------|--------|-------|
| DEV | A + B | C + D |
| STAGE | C + D | A + B |
| PROD | 전원 | Owner 승인 필수 |
| MGMT | D | 전원 |

---

## 5. 브랜치 전략

### 5.1 기본 구조 (Trunk-based 변형)

```
main ─────────────────────────────────── Production
  │
  └── develop ────────────────────────── Integration
        │
        ├── infra/vpc-peering ────────── Feature
        │     └── infra/vpc-peering-routing
        │
        ├── compute/eks-private-endpoint
        │
        ├── data/rds-backup-policy
        │
        └── obs/cloudtrail-central
```

### 5.2 브랜치 타입 정의

| 타입 | 용도 | 예시 |
|------|-----|------|
| `infra/` | 인프라 변경 (VPC, NAT, Peering) | `infra/payment-nat` |
| `compute/` | EKS, Node Group 변경 | `compute/eks-nodegroup` |
| `data/` | RDS, Valkey, S3 변경 | `data/rds-encryption` |
| `obs/` | 보안, 로그, 모니터링 | `obs/cloudtrail-logs` |
| `platform/` | GitOps, ArgoCD 변경 | `platform/argocd-config` |
| `app/` | 앱 매니페스트 변경 | `app/storefront-deploy` |
| `fix/` | 버그 수정 | `fix/nat-routing` |
| `hotfix/` | 긴급 수정 (main 직접) | `hotfix/prod-security` |
| `docs/` | 문서 변경 | `docs/runbook-update` |
| `chore/` | 설정, 의존성 업데이트 | `chore/terraform-upgrade` |

### 5.3 브랜치 네이밍 규칙

```
[타입]/[기능]-[상세]

예시:
infra/payment-nat               # 결제 NAT 추가
infra/payment-nat-routing       # 하위: 라우팅 추가
infra/payment-nat-sg            # 하위: 보안그룹 추가

compute/eks-private-endpoint    # EKS Private Endpoint
compute/eks-pe-security         # 하위: 보안그룹

obs/central-cloudtrail          # 중앙 CloudTrail
obs/central-cloudtrail-kms      # 하위: KMS 암호화

platform/argocd-private-endpoint
platform/argocd-peering-route
```

### 5.4 하위 브랜치 생성 규칙

```bash
# 메인 작업 브랜치에서 하위 브랜치 생성
git checkout infra/payment-nat
git checkout -b infra/payment-nat-routing

# 하위 작업 완료 후 상위 브랜치로 머지
git checkout infra/payment-nat
git merge infra/payment-nat-routing

# 최종적으로 develop으로 PR
```

---

## 6. Git 운영 규칙

### 6.1 커밋 메시지 규칙 (Conventional Commits)

```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

#### 타입 정의

| 타입 | 설명 |
|------|-----|
| `feat` | 새 기능 추가 |
| `fix` | 버그 수정 |
| `refactor` | 리팩토링 (기능 변경 없음) |
| `docs` | 문서 변경 |
| `chore` | 빌드, 설정 변경 |
| `perf` | 성능 개선 |
| `security` | 보안 관련 변경 |

#### 예시

```
feat(infra): MGMT-DEV VPC Peering 추가

- VPC Peering 연결 생성
- 양방향 라우팅 규칙 추가
- DNS 해석 옵션 활성화

Closes #123
```

```
fix(compute): EKS Private Endpoint 보안그룹 수정

MGMT VPC CIDR에서 443 포트 접근 허용 추가

Fixes #124
```

```
security(obs): CloudTrail KMS 암호화 활성화

ISMS-P 2.7.2 암호화 요구사항 충족

Refs #125
```

### 6.2 PR 템플릿

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## 📋 변경 요약
<!-- 이 PR에서 변경된 내용을 간단히 설명하세요 -->

## 🔗 관련 이슈
- Closes #

## 📂 변경된 파일
- [ ] modules/
- [ ] envs/
- [ ] docs/

## ✅ 체크리스트
### 일반
- [ ] 코드 리뷰 요청함
- [ ] 커밋 메시지 규칙 준수
- [ ] 관련 이슈 연결함

### 인프라 (해당 시)
- [ ] `terraform validate` 통과
- [ ] `terraform plan` 결과 확인
- [ ] 비용 영향 검토 (월 예상 추가 비용: $XX)
- [ ] 보안 영향 검토
- [ ] 충돌 가능성 검토 (다른 파트와)

### 운영 (해당 시)
- [ ] ISMS-P 준수 여부 확인
- [ ] 로그 보존 정책 확인
- [ ] 백업 정책 확인

## 📸 스크린샷/증거 (선택)
<!-- terraform plan 출력, 아키텍처 다이어그램 등 -->

## 🔍 리뷰어 가이드
<!-- 리뷰어가 특히 봐야 할 부분 -->
```

### 6.3 Issue 템플릿

```markdown
<!-- .github/ISSUE_TEMPLATE/infra-task.md -->
---
name: 인프라 작업
about: 인프라 변경 작업 요청
labels: infra
---

## 📋 작업 설명
<!-- 무엇을 변경해야 하는지 설명 -->

## 🎯 목표
<!-- 이 작업의 목표 -->

## 📂 변경 예상 파일
- [ ] `modules/xxx/`
- [ ] `envs/xxx/`

## ⚠️ 주의사항
<!-- 비용, 보안, 충돌 관련 -->

## 📊 우선순위
- [ ] High
- [ ] Medium
- [ ] Low

## 👤 담당 파트
- [ ] Network (A)
- [ ] Compute (B)
- [ ] Data (C)
- [ ] Security/Obs (D)
```

---

## 7. 팀 프로세스

### 7.1 작업 시작 시 (필수)

```
1. Issue 확인 → 담당 할당
2. 칸반에서 Todo → In Progress 이동
3. 브랜치 생성
   git checkout develop
   git pull origin develop
   git checkout -b [타입]/[기능]
4. Slack/팀채널에 공유:
   "[시작] #123 VPC Peering 작업 시작합니다"
```

### 7.2 작업 완료 시 (필수)

```
1. terraform validate + plan 실행
2. PR 생성 (템플릿 작성)
3. 칸반에서 In Progress → Review 이동
4. 리뷰어 지정 (CODEOWNERS 자동)
5. Slack/팀채널에 공유:
   "[리뷰요청] #123 VPC Peering PR 올렸습니다"
```

### 7.3 리뷰 완료 후

```
1. 승인 후 Squash Merge
2. 브랜치 삭제
3. 칸반에서 Review → Done 이동
4. Issue 자동 클로즈 확인
5. Slack/팀채널에 공유:
   "[완료] #123 VPC Peering 머지 완료"
```

### 7.4 검토 체크리스트

| 항목 | 확인 사항 |
|------|----------|
| **필요성** | 이 변경이 정말 필요한가? |
| **정합성** | 기존 아키텍처와 일관성 있는가? |
| **비용** | 비용 영향은? (월 $XX 추가) |
| **보안** | ISMS-P 요구사항 충족? |
| **충돌** | 다른 파트 작업과 충돌 가능성? |
| **롤백** | 문제 시 롤백 가능한가? |

---

## 8. 빠른 참조

### Git 명령어

```bash
# 브랜치 생성
git checkout develop && git pull
git checkout -b infra/my-feature

# 작업 후 커밋
git add .
git commit -m "feat(infra): 기능 추가"

# PR 전 최신화
git fetch origin
git rebase origin/develop

# Push
git push -u origin infra/my-feature
```

### Terraform 검증

```bash
cd envs/dev
terraform init
terraform validate
terraform plan
```

### 충돌 방지

```bash
# 작업 시작 전 항상
git fetch origin
git rebase origin/develop

# 충돌 발생 시
git rebase --continue  # 또는
git rebase --abort     # 취소
```

---

**이 문서를 기준으로 팀 운영을 시작하세요.**
