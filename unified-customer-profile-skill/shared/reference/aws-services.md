# AWS Services — Constraints & Quotas

## Amazon Connect Customer Profiles

| 항목 | 값 | 비고 |
|------|------|------|
| Instance quota (default) | 2 per account | 할당량 증가 요청 가능 (4~5) |
| Domain names | Account+Region 내 유닉 | |
| Object Types per domain | 100 | |
| Keys per Object Type | 10 | |
| Fields per Object Type | 200 | |
| Calculated Attributes per domain | 20 | |
| Profile size max | 50KB | |
| PutProfileObject rate | 500 TPS | 데모에서는 문제 없음 |

## AWS Entity Resolution

| 항목 | 값 | 비고 |
|------|------|------|
| ML Matching regions | us-east-1, us-west-2, eu-west-1, ap-southeast-1 등 | **반드시 확인!** |
| Rule-based matching | 대부분 리전 사용 가능 | |
| Max match keys per rule | 15 | |
| Max rules per workflow | 25 | |
| Input: Glue Table 필수 | | S3 직접 불가 |
| Processing time | ~5분 (10K records) | 배치 job, 실시간 아님 |

### 비용 (ER)
- Rule-based: $0.25 per 1,000 records processed
- ML-based: $1.00 per 1,000 records processed

## Amazon Neptune

| 항목 | 값 | 비고 |
|------|------|------|
| 최소 인스턴스 | db.r5.large | ~$0.58/hr = ~$420/month |
| Serverless 옵션 | Neptune Serverless | 최소 2.5 NCU = ~$200/month |
| VPC 필수 | YES | NAT Gateway 추가 비용 |
| 쿼리 언어 | openCypher, Gremlin, SPARQL | |

## Amazon Bedrock — Claude 모델 카탈로그 (최신, 2026-Q2 기준)

> 📌 **ID format**: 단일 region 호출은 `<vendor>.<model>`, 크로스 리전 inference profile은 `us./eu./apac.` 접두사. 예: `us.anthropic.claude-opus-4-7`. AP region에서 호출할 때는 `apac.` 접두사 사용. 사용자 region에 맞는 inference profile을 IAM에서 명시적으로 허용해야 한다.

| 모델 | Bedrock model ID | Context | Max output | 강점 | 비고 |
|---|---|---|---|---|---|
| **Claude Opus 4.7** | `us.anthropic.claude-opus-4-7` (cross-region) / `anthropic.claude-opus-4-7` | **1M** tokens | 128K | 최고 추론·코딩, knowledge cutoff 2026-01 | `thinking.type: "adaptive"`만 지원 (enabled 모드 없음). 가장 비싸지만 정밀도 최고. ER 규칙 생성·복잡 분석에 권장 |
| **Claude Sonnet 4** | `anthropic.claude-sonnet-4-20250514-v1:0` | 200K | 64K | 균형 모델, 코딩+추론 강함 | thinking + tool use 지원. 대부분 워크로드의 default |
| **Claude Opus 4.6** | `us.anthropic.claude-opus-4-6` | **1M** tokens | 128K | flagship 직전 세대, 대용량 RAG | knowledge cutoff 2025-05. `thinking.type: "enabled"` 지원 |
| **Claude Opus 4.1** | `anthropic.claude-opus-4-1-20250805-v1:0` | 200K | 32K | 안정적 reasoning | knowledge bases / agents 미지원 |
| **Claude Haiku 4.5** | `anthropic.claude-haiku-4-5-20251001` | 200K | 8K | 가장 저렴·빠름 | 대량 단순 분류·요약 워크로드, 1차 필터에 적합 |

### 가격 (대략, 1M token 기준)

| 모델 | Input | Output | 권장 용도 |
|---|---|---|---|
| Opus 4.7 | $15 | $75 | ER 규칙 자동개선 (Bedrock 호출 빈도 낮음, 정확도 중요) |
| Sonnet 4 | $3 | $15 | 일반 챗·요약·분석 — UCP의 default |
| Haiku 4.5 | $1 | $5 | 마케팅 메시지 생성, 1차 분류, 대량 personalization |

### 모델 선택 가이드 (Skill에서 사용자에게 제안)

```
Q: "AI 기능에 어떤 trade-off를 원하나요?"
├─ 정확도 최우선 (예: ER 규칙 생성, 복잡 추론)
│   → Opus 4.7   (`us.anthropic.claude-opus-4-7`)
├─ 균형 (대부분의 케이스)
│   → Sonnet 4   (`anthropic.claude-sonnet-4-20250514-v1:0`)
├─ 비용 최소화 (대량 personalization, 단순 분류)
│   → Haiku 4.5  (`anthropic.claude-haiku-4-5-20251001`)
└─ 1M context 필요 (대용량 프로필 + 거래내역 한꺼번에)
    → Opus 4.7 또는 Opus 4.6
```

### IAM 정책 — 두 가지 ARN 모두 필요

```json
{
  "actions": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
  "resources": [
    "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-opus-4-7",
    "arn:aws:bedrock:*::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0",
    "arn:aws:bedrock:us-east-1:<acct>:inference-profile/us.anthropic.claude-opus-4-7"
  ]
}
```

### 주의

- **`apac.` 접두사**: ap-northeast-2 등 AP region에서 cross-region inference profile 호출 시 필수
- **Opus 4.7 thinking**: `thinking.type: "enabled"` 사용하면 BadRequest. `"adaptive"`만 허용
- **모델 ID 진화**: 6~12개월마다 새 세대 출시. 이 카탈로그는 cutoff 2026-01 기준이며, 새로 빌드 시 AWS Knowledge MCP의 `aws___search_documentation`으로 최신 ID를 한번 더 확인할 것

## Amazon Kinesis Data Streams

| 항목 | 값 | 비고 |
|------|------|------|
| Shard 단위 비용 | ~$0.015/hr/shard | On-demand: 자동 스케일링 |
| PutRecord rate | 1,000 records/sec/shard | |
| Data retention | 24h (default) ~ 365일 | |

## Amazon Cognito

| 항목 | 값 | 비고 |
|------|------|------|
| MAU 무료 | 50,000 | 데모에서는 비용 0 |
| Hosted UI | 포함 | 커스텀 도메인 가능 |

## AWS Glue

| 항목 | 값 | 비고 |
|------|------|------|
| Database/Table 생성 | 무료 | 메타데이터만 (정적 정의 시) |
| Crawler 실행 | $0.44/DPU-hour | 최소 10분 과금, 보통 1~2 DPU 사용 |
| Connection (JDBC) | 무료 | 자격증명은 Secrets Manager ($0.40/secret/month) |
| ETL Job | $0.44/DPU-hour | 데이터 변환 필요 시 |

### Glue Connection 지원 소스
| 소스 | Connection Type | 비고 |
|------|----------------|------|
| Amazon RDS (MySQL, PostgreSQL) | JDBC | VPC 내 연결 |
| Amazon Aurora | JDBC | VPC 내 연결 |
| Amazon Redshift | JDBC/Redshift | 네이티브 지원 |
| Amazon DynamoDB | DynamoDB | 직접 지원 |
| On-premise DB | JDBC + VPN/DX | Site-to-Site VPN 또는 Direct Connect 필요 |
| S3 (CSV/Parquet/JSON) | S3 | Crawler 또는 직접 Table 정의 |

### Parquet vs CSV (S3 입력)
| 항목 | CSV | Parquet |
|------|-----|---------|
| 압축률 | 낮음 (원본 크기) | 높음 (50~80% 절감) |
| 스키마 | 외부 정의 필요 (Glue Table) | 파일 내 내장 |
| 쿼리 성능 (Athena) | 전체 스캔 | 컬럼 푸시다운 (10x+ 빠름) |
| ER 호환성 | ✅ Glue Table 정의 필요 | ✅ Glue Table 자동 인식 |
| 생성 용이성 | Excel/스크립트 즉시 가능 | pandas/Spark 등 필요 |

## 종합 비용 예시

### Minimal (데모/PoC)
```
Connect CP Domain: ~$0 (프로필 수 적을 때)
Entity Resolution: ~$5/month (20K records × 1회/주)
S3: ~$1
DynamoDB: ~$5 (on-demand)
Lambda: ~$0 (프리티어)
Cognito: ~$0 (50K MAU 이내)
API Gateway: ~$5
─────────────────
합계: ~$15-50/month
```

### Full (Neptune + Cross-Domain)
```
위 + Neptune: ~$300-420
위 + 추가 Connect Instances: ~$0 (인스턴스 자체는 무료)
위 + Kinesis (optional): ~$30
─────────────────
합계: ~$350-500/month
```
