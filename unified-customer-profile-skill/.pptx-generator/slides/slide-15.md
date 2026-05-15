# 슬라이드 15: 도구별 import 방법

---

## 원본 소스 참조

- 소스: unified-customer-profile-skill/README.md "사용법" 섹션
- 원본 내용:
  "Amazon Quick: Settings → Capabilities → Skills → Import → quick/SKILL.md"
  "Kiro: cp -r kiro/.kiro/ <your-project>/.kiro/"
  "Claude Code: cp claude-code/CLAUDE.md <your-project>/CLAUDE.md && cp -r claude-code/commands/ <your-project>/.claude/commands/"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.C3

---

## 레이아웃

```text
┌──────────┬──────────┬──────────┐
│  Quick   │   Kiro    │  Claude  │
│  Card    │   Card    │  Code    │
│          │           │  Card    │
└──────────┴──────────┴──────────┘
```

배치전략: 3개 도구 카드 (균등 분할)

---

## 구현 내용

### 제목

도구별 import : 1분 내 셋업, 한 줄 자연어로 시작

#### Col1 — Amazon Quick (Q)

> 🔧 구현: svg

```text
┌──────────────────────────┐
│  Amazon Quick (Q)        │
│  ────────────────────    │
│  Settings →              │
│   Capabilities →         │
│    Skills →              │
│     Import →             │
│      quick/SKILL.md      │
│                          │
│  소요: ~30초             │
│  ────────────────────    │
│  트리거 (자연어):         │
│  "고객 프로필 통합        │
│   시스템 만들어줘"        │
└──────────────────────────┘
```

#### Col2 — Kiro IDE

```text
┌──────────────────────────┐
│  Kiro IDE                │
│  ────────────────────    │
│  $ cp -r kiro/.kiro/     │
│      <project>/.kiro/    │
│                          │
│  또는 workspace에         │
│  steering.md를 Steering   │
│  Rules로 등록             │
│                          │
│  소요: ~30초             │
│  ────────────────────    │
│  트리거:                  │
│  "Using AI-DLC, UCP를     │
│   만들어주세요"            │
└──────────────────────────┘
```

> 🔧 구현: svg

#### Col3 — Claude Code (CLI)

```text
┌──────────────────────────┐
│  Claude Code (CLI)       │
│  ────────────────────    │
│  $ cp claude-code/        │
│      CLAUDE.md            │
│      <project>/           │
│                          │
│  $ cp -r claude-code/     │
│      commands/            │
│      .claude/commands/    │
│                          │
│  소요: ~1분              │
│  ────────────────────    │
│  트리거 (슬래시):          │
│  /generate-ucp            │
└──────────────────────────┘
```

> 🔧 구현: svg
