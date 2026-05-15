# 슬라이드 07: 한 지식베이스 + 여러 도구 어댑터

---

## 원본 소스 참조

- 소스: summary-unified-customer-profile-skill.md, concepts/multi-tool-skill-package.md
- 원본 내용:
  "Shared Knowledge: 실제 지식은 shared/에 한 번만 작성. 각 도구별 Skill은 이를 참조."
  "shared/patterns/ ~96 KB가 무게중심. 도구별 어댑터 합계 ~14 KB는 얇은 진입점."

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.C1_3_R3R

---

## 레이아웃

```text
┌──────────┬─────────────────────────┐
│          │  Quick — Settings 1줄    │
│ shared/  ├─────────────────────────┤
│ (트리)   │  Kiro — .kiro/steering/ │
│          ├─────────────────────────┤
│          │  Claude Code — CLAUDE.md │
└──────────┴─────────────────────────┘
```

배치전략: 좌(중앙 지식베이스 트리) 33 / 우(3개 도구 어댑터, 행 분할) 67

---

## 구현 내용

### 제목

Skill Package : 1 지식베이스 + 어댑터

#### Left — shared/ 무게중심 트리

> 🔧 구현: svg

```text
ucp-skill/
│
├─ shared/  ★ 96 KB (실제 지식)
│  ├─ reference/  19 KB
│  │   ├─ architecture.md
│  │   ├─ decision-tree.md
│  │   ├─ aws-services.md
│  │   └─ constraints.md
│  ├─ patterns/   58 KB
│  │   ├─ cdk-stacks.md
│  │   ├─ lambda-handlers.md
│  │   ├─ frontend-pages.md
│  │   ├─ er-strategies.md
│  │   ├─ bedrock-prompts.md
│  │   └─ etl-transforms.md
│  └─ examples/   12 KB
│      ├─ travel.md
│      ├─ hotel.md
│      └─ retail.md
│
└─ 어댑터 (~14 KB, 얇은 진입점) →
```

시각화 의도: shared/가 두꺼운 박스 / 어댑터들은 그 옆 작은 박스로 무게 차이를 시각적으로 표현

#### Right-Top — Quick (Amazon Q): 자연어 트리거

```text
$ Settings → Skills → Import quick/SKILL.md

사용자: "고객 프로필 통합 시스템 만들어줘"
       ↓
       Skill 자동 발화 → 같은 shared/ 참조
```

> 🔧 구현: svg

#### Right-Middle — Kiro: 워크스페이스 Steering

```text
$ cp -r kiro/.kiro/  <project>/.kiro/

Kiro IDE: Steering Rules + Spec 인식
       ↓
       같은 shared/ 참조
```

> 🔧 구현: svg

#### Right-Bottom — Claude Code: CLAUDE.md + 슬래시 커맨드

```text
$ cp claude-code/CLAUDE.md  <project>/
$ cp -r claude-code/commands/  .claude/commands/

사용자: /generate-ucp
       ↓
       같은 shared/ 참조
```

> 🔧 구현: svg
