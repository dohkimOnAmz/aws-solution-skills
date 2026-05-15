# 슬라이드 12: 9-레이어 + Gate 5단계로 안전한 생성

---

## 원본 소스 참조

- 소스: shared/reference/architecture.md, concepts/gate-pattern.md
- 원본 내용:
  "Layer 1-9: Foundation → Storage → Profiles → Matching → Ingestion → ETL Transform(선택) → Auth → API → Graph(선택) → Cross-Domain(선택)"
  "Discovery → Design → Generate → Validate → Deploy 5단계 Gate"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.C2

---

## 레이아웃

```text
┌────────────────────┬──────────────────┐
│  9-레이어 스택     │  Gate 5단계       │
│  (수직)            │  (수평 플로우)    │
└────────────────────┴──────────────────┘
```

배치전략: 좌(레이어 스택) 50 / 우(Gate 플로우) 50

---

## 구현 내용

### 제목

생성의 두 축 : 9-레이어 + 5-게이트

#### Left — 9-레이어 아키텍처 스택

> 🔧 구현: svg

```text
┌────────────────────────────┐
│ 9. Cross-Domain (선택)      │ ← 멀티 사업부 통합
├────────────────────────────┤
│ 8. Graph (선택)             │ ← Neptune + GraphRAG
├────────────────────────────┤
│ 7. API                      │ ← API GW + Lambda
├────────────────────────────┤
│ 6. Auth                     │ ← Cognito
├────────────────────────────┤
│ 5.5. ETL Transform (선택)   │ ← PII 정규화
├────────────────────────────┤
│ 5. Ingestion                │ ← CSV/Kinesis/Glue
├────────────────────────────┤
│ 4. Matching                 │ ← Entity Resolution
├────────────────────────────┤
│ 3. Profiles                 │ ← Connect CP
├────────────────────────────┤
│ 2. Storage                  │ ← S3
├────────────────────────────┤
│ 1. Foundation               │ ← KMS + DLQ
└────────────────────────────┘
```

시각화 의도: 레이어 스택을 위에서 아래로 (인프라 → 옵션). [선택] 표시로 옵션 레이어 구분

#### Right — 5-Gate 가로 플로우

> 🔧 구현: svg

```text
[Discovery]  [Design]   [Generate]   [Validate]   [Deploy]
    │           │           │            │            │
    ▼           ▼           ▼            ▼            ▼
요구사항      아키텍처      코드        cdk synth     배포
10 질문       결정          점진 생성    + MCP        가이드
              비용 추정     레이어별     검증
              리전 확인     5단계
    │           │           │            │            │
    ▼           ▼           ▼            ▼            ▼
 ⛔ GATE 1  ⛔ GATE 2  ⛔ GATE 3   ⛔ GATE 4   ⛔ GATE 5

각 게이트 = 사용자 명시적 승인
환각·과잉 생성 차단 지점
```

시각화 의도: 5단계가 좌→우로 흐르며 각 단계 아래에 ⛔ Gate 명시. 게이트마다 사용자 승인이 필요함을 강조
