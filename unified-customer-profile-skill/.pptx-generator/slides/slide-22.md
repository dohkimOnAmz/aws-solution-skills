# 슬라이드 22: 옵션 3 — Cross-Domain on/off 비교

---

## 원본 소스 참조

- 소스: shared/reference/decision-tree.md "4. Cross-Domain 필요 여부", shared/reference/architecture.md "Layer 9: Cross-Domain"
- 원본 내용:
  "Q: 서로 다른 사업부/브랜드 간 고객 통합이 필요한가? (여행+호텔+여행사 같은 생태계)"
  "여러 Connect Instance (도메인별), Platform-level CP Domain (통합 뷰), Cross-Domain ER Workflow"
  "Connect Instance quota 기본 2개 → 할당량 증가 필요"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.C2

---

## 레이아웃

```text
┌────────────────────┬──────────────────┐
│  Single Domain     │  Cross-Domain     │
│  (기본)            │  (+멀티 도메인 +   │
│                    │   Platform 통합)  │
└────────────────────┴──────────────────┘
```

배치전략: 좌우 50:50 비교 (전·후 다이어그램)

---

## 구현 내용

### 제목

옵션 3 — Cross-Domain on/off 비교

#### Left — Single Domain (기본)

> 🔧 구현: svg

```text
┌──────────────────────────────┐
│ Connect Instance × 1          │
│  ↓                           │
│ CP Domain × 1                 │
│  ├─ Object Types              │
│  │  (Reservation, Order,      │
│  │   ServiceCase ...)         │
│  ├─ Calculated Attributes     │
│  └─ ER Workflows × 3          │
│     (Simple/Advanced/ML)      │
│                              │
│  특징:                        │
│  ✓ 단일 사업·브랜드            │
│  ✗ 사업부 간 통합 불가         │
│                              │
│  Connect quota 사용: 1개      │
│  비용 추가: $0                 │
└──────────────────────────────┘

적합:
- 단일 호텔 체인
- 단일 이커머스 브랜드
- 단일 항공사 (예약만)
```

#### Right — Cross-Domain (+Platform 통합)

> 🔧 구현: svg

```text
┌──────────────────────────────┐
│ Connect Instance × N          │
│ (사업부별 — 항공/호텔/여행사)  │
│  ↓                           │
│ CP Domain × N (사업부별)      │
│  + Platform CP Domain ★       │
│    (통합 뷰)                  │
│                              │
│ Cross-Domain ER Workflow ★    │
│  ├─ 사업부 A 프로필 ↔ B       │
│  └─ Platform Domain에 통합    │
│                              │
│  특징:                        │
│  ✓ 사업부 간 동일 고객 식별    │
│  ✓ Ecosystem CLV 계산 가능    │
│    (사업부 매출 합 + 시너지)   │
│                              │
│  Connect quota: N+1개         │
│   → 할당량 증가 신청 필요      │
│  비용 추가: 도메인 수 비례     │
└──────────────────────────────┘

적합:
- 여행 그룹 (항공+호텔+여행사)
- 금융 그룹 (은행+카드+증권)
- 리테일 그룹 (백화점+할인점+H&B)
```

핵심: **Cross-Domain은 사업부 간 동일 고객을 통합한 360° 뷰가 필요할 때. Connect quota 증가 신청이 첫 번째 벽.**
