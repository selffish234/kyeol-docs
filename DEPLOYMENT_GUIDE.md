# 🚀 KYEOL Storefront 배포 가이드

## 📦 1단계: 로컬 검증

```bash
# 프로젝트 디렉토리로 이동
cd kyeol-storefront-org

# 의존성 설치 (이미 되어있다면 스킵)
pnpm install

# TypeScript 타입 체크
pnpm tsc --noEmit

# 프로덕션 빌드
pnpm build

# 로컬에서 프로덕션 모드 실행
pnpm start
```

### ✅ 로컬 테스트 시나리오

1. **로그인 테스트**
   - http://localhost:3000/default-channel/login
   - 잘못된 credentials 입력 → 에러 메시지 표시 확인
   - 올바른 credentials 입력 → 리다이렉트 확인

2. **장바구니 테스트**
   - 상품 페이지에서 "Add to Cart" 클릭
   - 장바구니 페이지 (/default-channel/cart) 이동
   - "Remove" 버튼 클릭 → "Removing..." 상태 표시 확인

3. **비로그인 상태 테스트**
   - 로그아웃 상태에서 장바구니 동작 확인
   - 500 에러 없이 정상 동작 확인

---

## 🐳 2단계: Docker 빌드

```bash
# Docker 이미지 빌드
docker build -t kyeol-storefront:v1.0.0 .

# 로컬에서 Docker 컨테이너 실행
docker run -p 3000:3000 \
  -e NEXT_PUBLIC_SALEOR_API_URL=https://your-saleor-api.com/graphql/ \
  kyeol-storefront:v1.0.0

# 헬스체크
curl http://localhost:3000/default-channel
```

---

## ☸️ 3단계: EKS 배포

### ArgoCD 배포 (권장)

```bash
# ArgoCD에 변경사항 반영
git add .
git commit -m "fix: resolve Server Actions runtime crashes

- Move login logic to API Route Handler
- Add executeServerAction wrapper for startTransition
- Fix TypeScript type errors with React 18.2.0
- Stabilize cart actions with proper error handling

Closes: KYEOL-XXX"

git push origin main

# ArgoCD 자동 배포 대기 또는 수동 sync
argocd app sync kyeol-storefront
```

### 수동 kubectl 배포

```bash
# 새 이미지로 배포 업데이트
kubectl set image deployment/kyeol-storefront \
  kyeol-storefront=your-registry/kyeol-storefront:v1.0.0 \
  -n production

# Rollout 상태 확인
kubectl rollout status deployment/kyeol-storefront -n production

# Pod 로그 확인
kubectl logs -f deployment/kyeol-storefront -n production
```

---

## 🔍 4단계: 배포 후 검증

### Pod 상태 확인

```bash
# Pod가 Running 상태인지 확인
kubectl get pods -n production -l app=kyeol-storefront

# Pod 로그에서 에러 확인
kubectl logs -n production -l app=kyeol-storefront --tail=100

# ✅ 확인 사항:
# - "s is not a function" 에러 없어야 함
# - "[POST /api/auth/login]" 로그 있어야 함
# - GraphQL 호출 로그 정상
```

### 헬스체크

```bash
# ALB/Ingress를 통한 헬스체크
curl https://your-storefront.com/default-channel

# 예상 응답: HTTP 200
```

### 기능 테스트 (프로덕션)

1. **로그인**
   - https://your-storefront.com/default-channel/login
   - 정상 로그인 → 리다이렉트 확인

2. **장바구니**
   - 상품 추가/삭제 정상 동작 확인

3. **에러 모니터링**
   ```bash
   # CloudWatch 또는 Datadog에서 500 에러율 확인
   # 목표: 0% (로그인/장바구니 관련)
   ```

---

## 🚨 롤백 절차 (문제 발생 시)

### ArgoCD 롤백

```bash
# 이전 버전으로 롤백
argocd app rollback kyeol-storefront
```

### kubectl 롤백

```bash
# 이전 Revision으로 롤백
kubectl rollout undo deployment/kyeol-storefront -n production

# 특정 Revision으로 롤백
kubectl rollout undo deployment/kyeol-storefront -n production --to-revision=3
```

---

## 📊 모니터링 대시보드

### 주요 메트릭

1. **HTTP 500 에러율**
   - 목표: 0%
   - 경로: `/default-channel/login`, `/api/auth/login`

2. **API 응답 시간**
   - `/api/auth/login` < 500ms
   - Cart actions < 200ms

3. **Pod 재시작 횟수**
   - 목표: 0 (CrashLoopBackOff 없어야 함)

### CloudWatch Logs Insights 쿼리

```sql
# Server Actions 에러 검색
fields @timestamp, @message
| filter @message like /s is not a function/
| sort @timestamp desc
| limit 20

# 로그인 성공/실패 분석
fields @timestamp, @message
| filter @message like /POST \/api\/auth\/login/
| stats count() by bin(5m)
```

---

## 🎯 성공 기준

- [ ] TypeScript 빌드 에러 0개
- [ ] Next.js 프로덕션 빌드 성공
- [ ] Docker 이미지 빌드 성공
- [ ] EKS Pod Running 상태
- [ ] 로그인 기능 정상 동작 (HTTP 200)
- [ ] 장바구니 추가/삭제 정상 동작
- [ ] 런타임 에러 로그 없음
- [ ] HTTP 500 에러율 0%

---

**배포 책임자:** Platform Team
**긴급 연락처:** DevOps Slack Channel
**Rollback 권한자:** Tech Lead
