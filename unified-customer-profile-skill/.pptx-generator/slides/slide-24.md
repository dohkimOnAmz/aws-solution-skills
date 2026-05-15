# 슬라이드 24: 비용 티어 — Minimal / Standard / Full

---

## 원본 소스 참조

- 소스: shared/reference/decision-tree.md "6. 비용 티어", shared/reference/aws-services.md "종합 비용 예시"
- 원본 내용:
  "Minimal: CP + ER(Rule) + CSV + Cognito ~$50-100"
  "Standard: + Kinesis + ML Matching ~$100-200"
  "Full: + Neptune + Cross-Domain ~$400-600"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.C3

---

## 레이아웃

```text
┌──────────┬──────────┬──────────┐
│ Minimal  │ Standard │  Full    │
│ Card     │ Card     │  Card    │
└──────────┴──────────┴──────────┘
```

배치전략: 3개 비용 티어 카드 (균등 분할)

---

## 구현 내용

### 제목

비용 티어 : Minimal에서 시작 → 단계적 확장

#### Col1 — Minimal ($50-100/월)

> 🔧 구현: svg

```text
┌──────────────────────────┐
│  💰 Minimal              │
│   $50-100 /월             │
│  ────────────────────    │
│                          │
│  포함:                    │
│  ✓ Foundation (KMS+SQS)  │
│  ✓ Storage (S3)          │
│  ✓ Profiles (Connect+CP) │
│  ✓ Matching (ER Rule)    │
│  ✓ Ingestion (CSV)       │
│  ✓ Auth (Cognito)        │
│  ✓ API (API GW + Lambda) │
│                          │
│  미포함:                  │
│  ✗ ETL Transform          │
│  ✗ Graph (Neptune)        │
│  ✗ Cross-Domain           │
│  ✗ ML Matching            │
│  ✗ Kinesis 실시간         │
│                          │
│  적합 시나리오:            │
│  - PoC, 데모              │
│  - 단일 도메인 단순 사례  │
│  - 프로필 < 10K           │
└──────────────────────────┘
```

#### Col2 — Standard ($100-200/월)

```text
┌──────────────────────────┐
│  💰💰 Standard           │
│   $100-200 /월            │
│  ────────────────────    │
│                          │
│  Minimal 포함 +          │
│  ✓ Kinesis (실시간)      │
│  ✓ ER ML Matching        │
│  ✓ ETL Transform         │
│    (Glue ETL Job)        │
│                          │
│  미포함:                  │
│  ✗ Graph (Neptune)        │
│  ✗ Cross-Domain           │
│                          │
│  적합 시나리오:            │
│  - 프로덕션 단일 도메인    │
│  - 실시간 이벤트 수집      │
│  - 한영 혼용 등 변형 큰   │
│    데이터                  │
│  - 프로필 10K-100K        │
│                          │
│  대표 산업:                │
│  - Hotel (변형 큰 데이터) │
│  - Retail (실시간 주문)   │
└──────────────────────────┘
```

> 🔧 구현: svg

#### Col3 — Full ($400-600/월)

```text
┌──────────────────────────┐
│  💰💰💰 Full             │
│   $400-600 /월            │
│  ────────────────────    │
│                          │
│  Standard 포함 +         │
│  ✓ Graph (Neptune        │
│    db.r5.large +          │
│    GraphRAG +             │
│    VPC NAT GW)            │
│  ✓ Cross-Domain           │
│    (멀티 Connect Instance │
│     + Platform Domain)    │
│  ✓ Ecosystem CLV          │
│                          │
│  Connect quota 신청 필수: │
│  → 4-5개 인스턴스          │
│                          │
│  적합 시나리오:            │
│  - 사업부 통합 (그룹사)    │
│  - 관계 분석 핵심 가치    │
│    (가족, 동행자, 추천)   │
│  - GraphRAG AI 어시스턴트 │
│  - 프로필 100K+           │
│                          │
│  대표 산업:                │
│  - Travel (3 도메인)       │
│  - 금융 그룹               │
└──────────────────────────┘
```

> 🔧 구현: svg

핵심: **Minimal로 시작 → 데이터 쌓이면 Standard → 사업부 통합 시점에 Full. Skill의 옵션 분기로 단계적 전환 가능.**
