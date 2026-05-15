# 슬라이드 06: Agent Skills — 도구 무관 개방형 표준

---

## 원본 소스 참조

- 소스: summary-claude-code-skills.md, concepts/agent-skills-standard.md
- 원본 내용:
  "Claude Code skills는 Agent Skills 개방형 표준(agentskills.io)을 따르며, 이는 여러 AI 도구에서 작동합니다. Claude Code는 호출 제어, subagent 실행, 동적 컨텍스트 주입과 같은 추가 기능으로 표준을 확장합니다."

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.C2

---

## 레이아웃

```text
┌────────────────────┬──────────────────┐
│  표준 동심원        │  표준 vs 확장    │
│  (코어 + 확장링)   │   비교 표        │
└────────────────────┴──────────────────┘
```

배치전략: 좌(다이어그램) 50 / 우(표) 50

---

## 구현 내용

### 제목

Agent Skills 개방형 표준 : 표준 코어 + 도구별 확장

#### Left — 표준 + 확장 동심원

> 🔧 구현: svg

```text
       ┌──────────────────────┐
       │  Tool Extensions      │
       │ ┌──────────────────┐ │
       │ │ Standard Core    │ │
       │ │  (agentskills.io)│ │
       │ │   SKILL.md       │ │
       │ │   - name         │ │
       │ │   - description  │ │
       │ │   - body         │ │
       │ └──────────────────┘ │
       │  + Claude Code:       │
       │    - allowed-tools    │
       │    - context: fork    │
       │    - !`<command>`    │
       └──────────────────────┘
```

시각화 의도: 표준이 작은 코어이고 도구별 확장이 바깥 링으로 둘러싸는 동심원 (확장이 비표준은 아님 — 표준 위에 얹는 구조)

#### Right — 표준 vs Claude Code 확장

| 기능 | 표준 (코어) | Claude Code 확장 |
|------|-------------|------------------|
| `name`, `description` | ✓ | ✓ |
| markdown 본문 | ✓ | ✓ |
| `disable-model-invocation` | — | ✓ |
| `allowed-tools` | — | ✓ |
| `context: fork` + subagent | — | ✓ |
| `` !`<command>` `` 동적 주입 | — | ✓ |
| 4-tier 위치 (Enterprise/Personal/Project/Plugin) | — | ✓ |

> 🔧 구현: native

핵심: **다른 AI 도구가 같은 표준을 채택하면 SKILL.md의 핵심 부분(`name`, `description`, body)은 호환된다.**
