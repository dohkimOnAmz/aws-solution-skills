# 슬라이드 16: Discovery → Deploy 워크플로우

---

## 원본 소스 참조

- 소스: quick/SKILL.md "워크플로우" 섹션, kiro/specs/generate-ucp.md, concepts/gate-pattern.md
- 원본 내용:
  "Phase 1: Discovery (대화형 요구사항 수집) — 산업, 채널, 식별 정보, 거래 데이터, KPI, 데이터 소스, 매칭 전략, 추가 기능, 리전/비용, 데이터 정규화"
  "Phase 2: Architecture Design"
  "Phase 3: Code Generation"
  "Phase 4: Deploy & Iterate"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.FULL

---

## 레이아웃

```text
┌───────────────────────────────────────────────┐
│  5단계 가로 플로우 + 각 단계의 사용자 행동     │
└───────────────────────────────────────────────┘
```

배치전략: 단일 풀폭 5단계 가로 타임라인

---

## 구현 내용

### 제목

Discovery → Deploy : 답·결정만, 나머지는 Skill

#### Content — 5단계 가로 타임라인

> 🔧 구현: svg

```text
[ Discovery ]   [ Design ]    [ Generate ]   [ Validate ]   [ Deploy ]
   ⏱ ~10분       ⏱ ~5분        ⏱ ~10분        ⏱ ~3분          ⏱ ~30분
      │             │              │              │              │
      ▼             ▼              ▼              ▼              ▼
┌──────────┐ ┌──────────┐  ┌──────────┐  ┌──────────┐   ┌──────────┐
│ 사용자   │ │ Skill   │  │ Skill   │  │ Skill   │   │ Skill   │
│ 10 질문  │ │ 자동     │  │ CDK +    │  │ cdk synth│   │ 배포 가이드│
│ 응답:    │ │ 결정:    │  │ Lambda + │  │ MCP 검증 │   │ + 콘솔    │
│          │ │          │  │ Frontend │  │          │   │ 수동 단계 │
│ 산업·채널│ │ 9 레이어 │  │ + 배포   │  │          │   │ 안내      │
│ PII·KPI  │ │ Pipeline │  │ 스크립트 │  │          │   │           │
│ 데이터   │ │ ER 전략  │  │          │  │          │   │           │
│ 소스·    │ │ 비용 티어│  │ 점진     │  │          │   │           │
│ 매칭·    │ │          │  │ 생성     │  │          │   │           │
│ Graph·   │ │          │  │ Core →   │  │          │   │           │
│ Cross·   │ │          │  │ Match →  │  │          │   │           │
│ 리전     │ │          │  │ Enrich   │  │          │   │           │
└──────────┘ └──────────┘  └──────────┘  └──────────┘   └──────────┘
      │             │              │              │              │
      ▼             ▼              ▼              ▼              ▼
   ⛔ GATE 1     ⛔ GATE 2      ⛔ GATE 3      ⛔ GATE 4      ⛔ GATE 5
   요구사항       설계 확정       코드 검토       검증 통과       배포 결정
   확정
```

시각화 의도: 5단계가 좌→우로 시간 흐름. 각 박스에 사용자가 하는 일과 Skill이 하는 일을 분리. 게이트가 시간 기준 분리선 역할
