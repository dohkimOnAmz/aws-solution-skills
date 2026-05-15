# 슬라이드 19: 베이스 아키텍처 — 9-레이어 → AWS 서비스 매핑

---

## 원본 소스 참조

- 소스: shared/reference/architecture.md, shared/reference/aws-services.md
- 원본 내용:
  "Layer 1 Foundation: KMS Key + SQS DLQ"
  "Layer 2 Storage: S3 Bucket"
  "Layer 3 Profiles: Connect Instance + Customer Profiles Domain + Object Types + Calculated Attributes"
  "Layer 4 Matching: Glue DB/Table + Entity Resolution Workflows + DynamoDB"
  "Layer 5 Ingestion: CSV / Parquet / Kinesis / Glue Connection / Hybrid"
  "Layer 6 Auth: Cognito User Pool + Hosted UI"
  "Layer 7 API: API Gateway REST API + Lambda Handlers"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.FULL

---

## 레이아웃

```text
┌───────────────────────────────────────────────┐
│  베이스 AWS 아키텍처 (Minimal: 옵션 모두 OFF) │
└───────────────────────────────────────────────┘
```

배치전략: 단일 풀폭 — 좌→우 데이터 흐름 + 9개 레이어를 AWS 아이콘으로

---

## 구현 내용

### 제목

베이스 아키텍처 : 9-레이어 → AWS 관리형 서비스

#### Content — 9-레이어 AWS 매핑 다이어그램

> 🔧 구현: svg

```text
                         AWS Cloud (Region: ap-northeast-2)
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  L1 Foundation                                                         │
│  ┌────────────┐  ┌────────────┐                                        │
│  │   KMS      │  │  SQS DLQ   │      ← 모든 레이어 암호화·실패 큐      │
│  │   Key      │  │            │                                        │
│  └────────────┘  └────────────┘                                        │
│                                                                        │
│  L5 Ingest →  L2 Storage →  L4 Matching →  L3 Profiles →  L7 API       │
│  ┌────────┐   ┌────────┐    ┌──────────┐   ┌──────────┐   ┌─────────┐  │
│  │ CSV    │──▶│   S3   │──▶│  Glue    │──▶│  Connect │──▶│  API    │  │
│  │ Upload │   │ Bucket │    │  DB/Tbl  │   │ Instance │   │ Gateway │  │
│  │ Lambda │   │ (KMS)  │    │ + ER WF  │   │ + CP     │   │         │  │
│  └────────┘   └────────┘    │ + DDB    │   │ Domain   │   └────┬────┘  │
│                              └──────────┘   │ + Object │        │      │
│                                             │ Types +  │        ▼      │
│                              L6 Auth        │ Calc Attr│   ┌─────────┐ │
│                              ┌──────────┐   └──────────┘   │ Lambda  │ │
│                              │ Cognito  │                  │ Handlers│ │
│                              │ User Pool│                  │ (5개+)  │ │
│                              │ + Hosted │                  └─────────┘ │
│                              │ UI       │                              │
│                              └──────────┘                              │
│                                                                        │
│              + Bedrock (apac.anthropic.claude-sonnet-4-...)            │
│                ER 규칙 자동 생성·개선 + AI 어시스턴트                  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
                  ▲                                              │
                  │ React + Cloudscape                           ▼
                  │ (S3 + CloudFront)                       Cognito 토큰
                  └─────────────────────────────────────────────┘

  옵션 OFF (Minimal):  ETL · Graph · Cross-Domain 미포함
  비용 추정: ~$50-100/월 (프로필 < 10K, ER 주1회)
```

시각화 의도: 9-레이어를 AWS 아이콘과 함께 좌→우 데이터 흐름으로. KMS/Cognito/Bedrock은 보조 위치. 베이스 = 옵션 모두 OFF임을 명시
