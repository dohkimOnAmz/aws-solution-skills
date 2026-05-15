# 슬라이드 09: UCP가 풀어야 할 문제 — 분산된 고객 데이터

---

## 원본 소스 참조

- 소스: shared/reference/architecture.md, concepts/unified-customer-profile.md
- 원본 내용:
  "다채널 고객 데이터를 수집하고, Entity Resolution으로 동일 고객을 식별하며, 통합된 고객 프로필을 관리하는 시스템."
  "[채널 데이터] → [Ingestion] → [S3/Glue Table] → [Entity Resolution] → [매칭 결과] → [Customer Profiles Import] → [360° Unified Profile]"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.FULL

---

## 레이아웃

```text
┌───────────────────────────────────────────────┐
│  좌측 6채널 → 중앙 ER → 우측 360° 프로필     │
└───────────────────────────────────────────────┘
```

배치전략: 단일 풀폭 다이어그램 — 좌측 입력 → 중앙 처리 → 우측 출력

---

## 구현 내용

### 제목

UCP 출발점 : 분산 채널 → 동일 고객 통합

#### Content — 다채널 → ER → 360° 프로필 플로우

> 🔧 구현: svg

```text
[ Web      ] ─┐
[ Mobile   ] ─┤
[ Call Ctr ] ─┼─▶ [ Entity      ] ─▶ [ Customer    ] ─▶ [ 360° Profile  ]
[ OTA      ] ─┤    Resolution         Profiles            + Calculated
[ POS      ] ─┤    (Simple/            Domain               Attributes
[ Corp     ] ─┘     Advanced/                              (CLV, RFM,
                    ML)                                     LastVisit ...)

  채널마다 동일 고객을        매칭 규칙 자동 생성        Golden Record           자동 집계 지표
  다른 ID·이름 변형으로       (Bedrock + HITL 검증)      (도메인당 50KB         (SUM/AVG/COUNT/
  기록                                                   max, 100 Object Type)  LAST_OCCURRENCE)
```

시각화 의도: 좌측 분산 채널 → 중앙 처리 단계 → 우측 통합 프로필. 각 박스 아래에 핵심 사실 캡션
