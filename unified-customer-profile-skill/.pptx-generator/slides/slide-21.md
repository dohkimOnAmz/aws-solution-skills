# 슬라이드 21: 옵션 2 — Knowledge Graph on/off 비교

---

## 원본 소스 참조

- 소스: shared/reference/decision-tree.md "3. Knowledge Graph 필요 여부", shared/reference/architecture.md "Layer 8: Graph"
- 원본 내용:
  "Q: 고객 간 관계 분석이 필요한가? (가족/동행자, 법인-개인, 영향력 분석, GraphRAG)"
  "Neptune Cluster (openCypher/Gremlin), Graph Sync Lambda, GraphRAG Lambda"
  "비용: db.r5.large 최소 ~$420/월 또는 Serverless 2.5 NCU ~$200/월. + NAT Gateway 추가 비용"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.C2

---

## 레이아웃

```text
┌────────────────────┬──────────────────┐
│  Graph OFF (기본)  │  Graph ON         │
│  단일 프로필만     │  + Neptune        │
│                    │  + GraphRAG       │
└────────────────────┴──────────────────┘
```

배치전략: 좌우 50:50 비교 (전·후 다이어그램)

---

## 구현 내용

### 제목

옵션 2 — Knowledge Graph on/off 비교

#### Left — Graph OFF (기본 / Minimal)

> 🔧 구현: svg

```text
┌──────────────────────────────┐
│ 베이스 9-레이어 (그대로)      │
│  ─────────────────            │
│  Connect CP Domain            │
│  └─ 360° 개별 프로필           │
│     - Calculated Attributes    │
│       (CLV, 방문 빈도)         │
│     - Object Types             │
│       (Booking, Order)         │
│                              │
│  특징:                        │
│  ✓ 개별 고객만 분석            │
│  ✗ 관계 정보 없음              │
│                              │
│  비용 추가: $0                 │
│  복잡도 추가: 0                │
└──────────────────────────────┘

분석 가능:
- "이 고객의 1년 매출은?"
- "이 고객의 마지막 방문일?"
- "이 고객의 RFM 세그먼트?"
```

#### Right — Graph ON (+Neptune + GraphRAG)

> 🔧 구현: svg

```text
┌──────────────────────────────┐
│ 베이스 + Layer 8 추가          │
│  ─────────────────            │
│  Connect CP Domain (그대로)   │
│  + Neptune Cluster   ★        │
│    (openCypher, db.r5.large)  │
│  + Graph Sync Lambda  ★        │
│    (CP 프로필 → Neptune        │
│     노드/엣지 동기화)          │
│  + GraphRAG Lambda    ★        │
│    (자연어 → Cypher 변환)      │
│  + VPC + NAT Gateway  ★        │
│    (Neptune 격리)              │
│                              │
│  비용 추가: ~+$300/월         │
│  복잡도 추가: VPC 설계 필요   │
└──────────────────────────────┘

추가 분석:
- "이 고객의 가족 구성원은?"
- "법인 고객이 추천한 개인은?"
- "여행 동반자 네트워크는?"
- "AI: '서울 출장 잦은 VIP 그룹의
       선호 호텔은?'" (GraphRAG)
```

핵심: **Graph는 선택 — 비용·복잡도가 +$300/월·VPC 설계로 큼. 관계 분석이 비즈니스 가치를 만들 때만.**
