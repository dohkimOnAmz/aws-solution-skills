# 슬라이드 20: 옵션 1 — Ingestion Pipeline 5종 비교

---

## 원본 소스 참조

- 소스: shared/reference/decision-tree.md "1. Ingestion Mode 선택"
- 원본 내용:
  "기존 DB → glue_connection (RDS, Aurora, Redshift, DynamoDB, on-prem)"
  "파일 작음 → csv (default, S3 + Lambda)"
  "파일 큼 → parquet (S3 + Glue Table 직접 참조, 압축률 높음)"
  "실시간 → kinesis (Kinesis Stream + EventBridge Pipe + CP 직접 적재)"
  "하이브리드 → glue_connection (초기) + kinesis (신규 이벤트)"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.G3X2

---

## 레이아웃

```text
┌────────┬────────┬────────┐
│ CSV    │ Parquet│ Kinesis│
├────────┼────────┼────────┤
│ Glue   │ Hybrid │ 결정   │
│ Conn.  │        │ 가이드 │
└────────┴────────┴────────┘
```

배치전략: 6셀 — 5개 모드 + 1 결정 가이드 셀 (3열×2행)

---

## 구현 내용

### 제목

옵션 1 — Ingestion Pipeline 5종 비교

#### TL — CSV (default, 데모/PoC)

> 🔧 구현: svg

```text
[CSV File] ──▶ S3 ──▶ Lambda ──▶ Glue Table

  ✓ 데모/PoC, 소량 (<100MB)
  ✓ 빈도 낮음 (주1회~)
  ✓ 가장 빠른 셋업
  비용: ~$1-5/월
```

#### TM — Parquet (대용량 배치)

```text
[Parquet] ──▶ S3 ──▶ Glue Table (직접 참조)

  ✓ 대용량 (>100MB)
  ✓ 압축률 50-80%
  ✓ Athena 컬럼 푸시다운
  ✓ 스키마 내장
  비용: ~$5-20/월
```

> 🔧 구현: svg

#### TR — Kinesis (실시간)

```text
[Event] ──▶ Kinesis ──▶ EventBridge Pipe ──▶ CP

  ✓ 실시간 (<1초 지연)
  ✓ 자동 스케일 (on-demand)
  ✓ 24h~365일 보존
  비용: ~$30-100/월 (shard 수 비례)
```

> 🔧 구현: svg

#### BL — Glue Connection (기존 DB 연동)

```text
[RDS / Aurora / DynamoDB] ──▶ Glue JDBC ──▶ Glue Crawler ──▶ Glue Table

  ✓ 기존 DB 직접 연동
  ✓ 주기적 동기화
  ✓ MySQL / PostgreSQL / Oracle / SQL Server / DynamoDB 지원
  비용: ~$10-30/월 (Crawler DPU-hour 기준)
```

> 🔧 구현: svg

#### BM — Hybrid (마이그레이션 + 실시간)

```text
[기존 DB] ──▶ Glue Connection (초기 일괄)
[신규 이벤트] ──▶ Kinesis (지속)

  ✓ 마이그레이션 단계
  ✓ 점진적 전환
  ✓ 두 모드 병행
  비용: 위 두 비용 합 ~$40-130/월
```

> 🔧 구현: svg

#### BR — 결정 가이드

```text
┌────────────────────────────┐
│  데이터 어디에 있나?       │
│  ──────────────            │
│  → 기존 DB:                │
│      Glue Connection       │
│  → 파일 (소량):            │
│      CSV                   │
│  → 파일 (대용량):          │
│      Parquet               │
│  → 실시간 이벤트:          │
│      Kinesis               │
│  → 마이그레이션 중:        │
│      Hybrid                │
└────────────────────────────┘
```

> 🔧 구현: svg
