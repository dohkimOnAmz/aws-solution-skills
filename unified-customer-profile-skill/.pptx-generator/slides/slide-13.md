# 슬라이드 13: Bedrock이 ER 규칙을 만들고, 사람은 검증한다

---

## 원본 소스 참조

- 소스: shared/patterns/er-strategies.md, shared/patterns/bedrock-prompts.md, concepts/entity-resolution.md
- 원본 내용:
  "Bedrock 자동 생성: AI가 데이터를 보고 규칙을 만들어주고, HITL로 검증"
  "HITL 검증 필수: 매칭 쌍 미리보기 / 예상 precision/recall / false positive 경고 / 테스트 실행"
  "3가지 매칭은 항상 전부 포함: Simple Rule + Advanced Rule + ML Matching"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.FULL

---

## 레이아웃

```text
┌───────────────────────────────────────────────┐
│  AI 자동 생성 + HITL 검증 사이클 다이어그램   │
└───────────────────────────────────────────────┘
```

배치전략: 단일 풀폭 — 5단계 사이클 다이어그램

---

## 구현 내용

### 제목

ER 규칙 : Bedrock이 만들고, 사람은 검증한다 (HITL)

#### Content — 5단계 사이클 다이어그램

> 🔧 구현: svg

```text
                     [1. 데이터 프로파일링]
                           Bedrock 분석:
                           - nullRate, uniqueRate
                           - 변형 패턴 (한영, OTA relay)
                           - 채널 분포
                              │
                              ▼
       ┌──────────────[2. AI 규칙 자동 생성]
       │              Bedrock 출력 (JSON):
       │              { ruleName, matchingKeys,
       │                rationale, estPrecision,
       │                estRecall, warnings }
       │                    │
       │                    ▼
       │         [3. HITL 검증 — 사용자 게이트]
       │              ┌──────────────┐
       │              │ 매칭 쌍       │
       │              │ 미리보기       │
       │              │ + precision    │
       │              │ + recall       │
       │              │ + false pos.   │
       │              │   경고         │
       │              └──────────────┘
       │                    │
       │     ┌──────────────┴──────────────┐
       │     ▼                              ▼
       │  ✓ 승인                          ✗ 거부 / 수정
       │     │                              │
       │     ▼                              │
       │ [4. 3-타입 동시 적용]               │
       │  Simple + Advanced + ML            │
       │  (사용자가 고르지 않음 — 모두 생성)  │
       │     │                              │
       │     ▼                              │
       │ [5. 비교 대시보드]                  │
       │  타입별 매칭 그룹/미매칭/precision  │
       │     │                              │
       └─────┴──────────[피드백 루프]────────┘
                    "이 규칙은 false positive
                     너무 많아" 등 사용자 피드백
                     → Bedrock 재생성
```

시각화 의도: 데이터 → AI → HITL → 적용 → 피드백의 사이클을 명확히. HITL 게이트(3번)가 시각적 중심에 위치
