# 슬라이드 17: 산업별 골든 레퍼런스 3종

---

## 원본 소스 참조

- 소스: shared/examples/travel.md, shared/examples/hotel.md, shared/examples/retail.md
- 원본 내용:
  "Travel: 3 도메인(Airline + Hotel + Agency) + Cross-Domain + Knowledge Graph + Ecosystem CLV (시너지 1.2~1.5x)"
  "Hotel: 단일 도메인 + OTA relay email 처리 + LoyaltyNumber 우선"
  "Retail: 게스트 구매(이메일만) + 마켓플레이스 relay + RFM 세그먼트 + Kinesis 실시간"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.C3

---

## 레이아웃

```text
┌──────────┬──────────┬──────────┐
│ Travel   │  Hotel    │  Retail  │
│ Card     │  Card     │  Card    │
└──────────┴──────────┴──────────┘
```

배치전략: 3개 산업 카드 (균등 분할)

---

## 구현 내용

### 제목

산업별 레퍼런스 : 신규 도메인 = examples/ 한 장

#### Col1 — Travel & Hospitality

> 🔧 구현: svg

```text
┌──────────────────────────┐
│  ✈ Travel & Hospitality  │
│  ────────────────────    │
│  도메인: 3개 +            │
│  Platform 통합            │
│  (Airline / Hotel /       │
│   Agency)                 │
│                          │
│  핵심 ER 규칙:            │
│  - FFN (정확)             │
│  - Loyalty (정확)         │
│  - Name + Email (퍼지)    │
│                          │
│  옵션:                    │
│  ✓ Cross-Domain ON        │
│  ✓ Graph ON (Neptune)     │
│  ✓ Ecosystem CLV          │
│                          │
│  비용: ~$400-600/월       │
│  난이도: ★★★             │
└──────────────────────────┘
```

#### Col2 — Hotel

```text
┌──────────────────────────┐
│  🏨 Hotel & Hospitality  │
│  ────────────────────    │
│  도메인: 단일             │
│                          │
│  채널: 6개                │
│  Web / App / OTA /        │
│  Walk-in / Call / Corp    │
│                          │
│  핵심 ER 규칙:            │
│  - Loyalty (정확) 우선    │
│  - Name + Phone (퍼지)    │
│    ← OTA relay email      │
│      우회                 │
│                          │
│  옵션:                    │
│  - Cross-Domain OFF       │
│  - Graph OFF              │
│  - Pipeline: CSV          │
│                          │
│  비용: ~$50-100/월        │
│  난이도: ★                │
└──────────────────────────┘
```

> 🔧 구현: svg

#### Col3 — Retail / E-commerce

```text
┌──────────────────────────┐
│  🛒 Retail / E-commerce │
│  ────────────────────    │
│  도메인: 단일             │
│                          │
│  채널: 5개                │
│  Web / App / POS /        │
│  Call / Marketplace       │
│                          │
│  핵심 ER 규칙:            │
│  - Loyalty (정확)         │
│  - Email Only (퍼지)      │
│    ← 1인1메일 가정         │
│  - Name + Phone (POS)     │
│                          │
│  옵션:                    │
│  - Cross-Domain OFF       │
│  - Graph OFF              │
│  - Pipeline: Kinesis      │
│    (실시간 주문)           │
│  - RFM 세그먼트            │
│                          │
│  비용: ~$100-200/월       │
│  난이도: ★★               │
└──────────────────────────┘
```

> 🔧 구현: svg

핵심: **새 산업(금융·헬스케어·텔코)은 `shared/examples/<industry>.md` 한 장 추가로 지원**
