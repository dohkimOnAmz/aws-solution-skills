# 슬라이드 11: UCP Skill 패키지 구조 — 한 지식베이스, 세 어댑터

---

## 원본 소스 참조

- 소스: unified-customer-profile-skill/README.md
- 원본 내용:
  "이 디렉토리는 Unified Customer Profile (AWS Connect CP + Entity Resolution + Bedrock AI) 아키텍처를 대화형으로 생성할 수 있는 AI Skill을 3가지 도구 포맷으로 제공합니다."
  shared/patterns/ ~96 KB / shared/reference/ ~19 KB / shared/examples/ ~12 KB / 도구별 어댑터 합 ~14 KB

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.C2_3_R2R

---

## 레이아웃

```text
┌──────────────────────┬──────────┐
│                      │ shared/  │
│  디렉토리 트리       │ 무게     │
│  (전체 패키지)       │ 막대차트 │
│                      ├──────────┤
│                      │ 어댑터   │
│                      │ 비율     │
└──────────────────────┴──────────┘
```

배치전략: 좌(트리) 67 / 우(시각화 보조) 33

---

## 구현 내용

### 제목

패키지 구조 : shared/ 중심, 어댑터는 진입점

#### Left — 패키지 디렉토리 구조

> 🔧 구현: svg

```text
unified-customer-profile-skill/
│
├─ README.md
│
├─ quick/
│  └─ SKILL.md                ← Amazon Quick 어댑터
│
├─ kiro/
│  ├─ steering.md
│  └─ specs/generate-ucp.md   ← Kiro 어댑터
│
├─ claude-code/
│  ├─ CLAUDE.md
│  └─ commands/generate-ucp.md ← Claude Code 어댑터
│
├─ shared/                    ★ 진짜 자산 (96 KB)
│  ├─ reference/  (19 KB) — WHY: 결정 근거
│  │  └─ architecture / decision-tree / aws-services / constraints
│  ├─ patterns/   (58 KB) — HOW: 코드 템플릿
│  │  └─ cdk-stacks / lambda-handlers / frontend-pages / er-strategies / bedrock-prompts / etl-transforms
│  └─ examples/   (12 KB) — WHAT: 산업별 골든 레퍼런스
│     └─ travel / hotel / retail
│
└─ evals/                     ← 회귀 테스트 (Skill 수정 시 검증)
   └─ travel-scenario / hotel-scenario / retail-scenario
```

시각화 의도: 어댑터 3개 + shared/(★ 강조) + evals/ 4개 그룹을 트리로. shared/만 굵은 보더로 무게중심 강조

#### Right-Top — 사이즈 비교 막대 차트

> 🔧 구현: svg

```text
shared/patterns/    ███████████████  58 KB
shared/reference/   █████             19 KB
shared/examples/    ████              12 KB
─────────────────────────────────────
어댑터 3종 합       ███               14 KB

shared/ 합계: 89 KB (87%)
어댑터 합계: 14 KB (13%)
```

#### Right-Bottom — 핵심 메시지

**Skill 패키지의 가치는 어댑터가 아니라 shared/에 있다.**

지식 한 번 갱신 → 세 도구 출력 일관

