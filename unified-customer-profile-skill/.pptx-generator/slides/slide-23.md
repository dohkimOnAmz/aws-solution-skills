# 슬라이드 23: 산업별 권장 옵션 조합

---

## 원본 소스 참조

- 소스: shared/examples/travel.md, hotel.md, retail.md + shared/reference/decision-tree.md
- 원본 내용:
  Travel: "3개 도메인 + Cross-Domain + Knowledge Graph + Ecosystem CLV"
  Hotel: "단일 도메인 + LoyaltyNumber 우선 + OTA relay email 처리 + CSV ingestion"
  Retail: "단일 도메인 + EmailOnly 핵심 + Kinesis 실시간 + RFM 세그먼트"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.FULL

---

## 레이아웃

```text
┌───────────────────────────────────────────────┐
│  3행 × 6열 비교 표                            │
│  (산업 / Pipeline / Graph / Cross / 비용 / 도전) │
└───────────────────────────────────────────────┘
```

배치전략: 단일 콘텐츠 100% — 비교 표

---

## 구현 내용

### 제목

산업별 권장 옵션 조합

#### Content — 산업별 옵션 조합 비교 표

| 산업 | Pipeline | Graph | Cross-Domain | 핵심 ER | 월 비용 | 핵심 도전 |
|------|----------|-------|--------------|---------|---------|----------|
| **Travel** ✈ | **Hybrid** (Glue + Kinesis) | **ON** (Neptune + GraphRAG) | **ON** (Airline+Hotel+Agency+Platform) | FFN, Loyalty + Name+Email | **$400-600** | Connect quota 4개+, Ecosystem CLV 시너지 계수 |
| **Hotel** 🏨 | **CSV** (default) | OFF | OFF (단일) | Loyalty, Name+Phone (OTA relay 우회) | **$50-100** | OTA relay email 감지, 법인 대리예약 |
| **Retail** 🛒 | **Kinesis** (실시간 주문) | OFF | OFF (단일) | Loyalty, EmailOnly (1인1메일), Name+Phone (POS) | **$100-200** | 게스트 구매(이름 없음), 마켓플레이스 relay |

> 🔧 구현: native

#### Content — 핵심 메시지

**같은 9-레이어 베이스 위에 옵션 조합만 다르다.**

→ Skill의 Discovery Phase에서 사용자가 산업·데이터·관계 분석 필요 여부를 답하면, decision-tree.md 6개 분기를 거쳐 **위 표의 한 행이 자동으로 선택**된다.

→ 새 산업(금융, 헬스케어, 텔코)은 같은 표에 한 행을 추가하는 것과 같다 — `shared/examples/<industry>.md` 한 장.
