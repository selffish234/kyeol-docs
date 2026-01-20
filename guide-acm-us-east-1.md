# CloudFront용 us-east-1 ACM 인증서 발급 가이드 (kyeol.click)

CloudFront 배포를 위해서는 반드시 **us-east-1 (N. Virginia)** 리전에 ACM 인증서가 발급되어 있어야 합니다. 이 문서는 `kyeol.click` 도메인에 대한 와일드카드 인증서를 발급하는 절차를 설명합니다.

## 1. ACM 인증서 요청 (CLI)

아래 명령어를 통해 `us-east-1` 리전에 `kyeol.click` 및 `*.kyeol.click`에 대한 인증서를 요청합니다.

```powershell
aws acm request-certificate `
  --region us-east-1 `
  --domain-name "kyeol.click" `
  --subject-alternative-names "*.kyeol.click" `
  --validation-method DNS
```

## 2. DNS 검증 레코드 확인 및 등록

요청 후 생성된 CNAME 레코드 정보를 확인합니다.

```powershell
# 요청한 인증서의 ARN 확인
$CERT_ARN = aws acm list-certificates `
  --region us-east-1 `
  --query "CertificateSummaryList[?DomainName=='kyeol.click'].CertificateArn" `
  --output text

# 검증용 CNAME 레코드 정보 추출
aws acm describe-certificate `
  --region us-east-1 `
  --certificate-arn $CERT_ARN `
  --query "Certificate.DomainValidationOptions[0].ResourceRecord" `
  --output json
```

추출된 `Name`과 `Value` 값을 Route53의 `kyeol.click` Hosted Zone에 **CNAME** 레코드로 등록합니다. (ExternalDNS가 아닌 수동 등록 권장)

## 3. 발급 상태 확인

상태가 `ISSUED`로 변경될 때까지 대기합니다 (보통 5~10분 소요).

```powershell
aws acm describe-certificate `
  --region us-east-1 `
  --certificate-arn $CERT_ARN `
  --query "Certificate.Status" `
  --output text
```

## 4. Terraform 반영

발급된 ARN을 `envs/mgmt/terraform.tfvars` 또는 `envs/prod/terraform.tfvars`의 `cloudfront_acm_arn` 변수에 입력합니다.

```hcl
# envs/prod/terraform.tfvars
cloudfront_acm_arn = "arn:aws:acm:us-east-1:827913617839:certificate/..."
```

> [!IMPORTANT]
> CloudFront는 us-east-1 리전의 인증서만 인식합니다. ap-northeast-2(서울) 리전의 인증서를 사용하면 CloudFront 생성 시 오류가 발생합니다.
