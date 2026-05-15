# Unified Customer Profile Builder — AI Skill

> **템플릿 배포가 아닌, 지식 배포.** 사용자에게 코드를 주는 게 아니라 코드를 만드는 능력을 준다.

이 디렉토리는 Unified Customer Profile (AWS Connect CP + Entity Resolution + Bedrock AI)
아키텍처를 **대화형으로 생성**할 수 있는 AI Skill을 3가지 도구 포맷으로 제공합니다.

## 지원 도구

| 도구 | 포맷 | 위치 | 사용 맥락 |
|------|------|------|-----------|
| **Amazon Quick** | `SKILL.md` | `quick/` | 대화형 생성 + 백그라운드 태스크 + MCP 연동 |
| **Kiro** | Steering Rules + Spec | `kiro/` | IDE 내 프로젝트 생성 + 코드 수정 |
| **Claude Code** | `CLAUDE.md` + commands | `claude-code/` | CLI 에이전트로 전체 프로젝트 생성 |

## 디렉토리 구조

```
unified-customer-profile-skill/
├── README.md                    ← 이 파일
├── quick/
│   └── SKILL.md                 ← Amazon Quick Skill 정의
├── kiro/
│   ├── steering.md              ← Kiro Steering Rules
│   └── specs/
│       └── generate-ucp.md      ← Kiro Spec (요구사항 → 구현)
├── claude-code/
│   ├── CLAUDE.md                ← Claude Code 프로젝트 지침
│   └── commands/
│       └── generate-ucp.md      ← /generate-ucp 슬래시 커맨드
├── shared/                      ← 세 도구가 공유하는 지식 (핵심)
│   ├── reference/
│   │   ├── architecture.md      ← 아키텍처 결정 + WHY
│   │   ├── aws-services.md      ← 서비스별 제약/할당량/비용
│   │   ├── decision-tree.md     ← 조건부 선택 로직
│   │   └── constraints.md       ← 알려진 제한사항
│   ├── patterns/
│   │   ├── cdk-stacks.md        ← CDK 스택 패턴 (코드 블록)
│   │   ├── lambda-handlers.md   ← Lambda 핸들러 패턴
│   │   ├── frontend-pages.md    ← React/Cloudscape 페이지 패턴
│   │   └── er-strategies.md     ← Entity Resolution 매칭 전략
│   └── examples/
│       ├── travel.md            ← 여행 산업 참조 (golden example)
│       ├── hotel.md             ← 호텔 참조
│       └── retail.md            ← 소매 참조
└── evals/                       ← Skill 검증용 테스트 시나리오
    ├── travel-scenario.md
    ├── hotel-scenario.md
    └── retail-scenario.md
```

## 핵심 설계 원칙

1. **Shared Knowledge**: 실제 지식은 `shared/`에 한 번만 작성. 각 도구별 Skill은 이를 참조.
2. **MCP 활용**: AWS Knowledge MCP, CloudFormation MCP 등으로 실시간 검증.
3. **Gate 패턴**: Discovery → Design → Generate → Validate → Deploy (각 단계 승인).
4. **점진적 생성**: Core → Matching → Enrichment → Graph (레이어별 추가).
5. **Golden Examples**: Travel Demo 코드를 정답 참조로 활용.

## 사용법

### Amazon Quick
```
Settings → Capabilities → Skills → Import → quick/SKILL.md
```

### Kiro
```bash
cp -r kiro/.kiro/ <your-project>/.kiro/
# 또는 workspace에 steering.md를 Steering Rules로 등록
```

### Claude Code
```bash
cp claude-code/CLAUDE.md <your-project>/CLAUDE.md
cp -r claude-code/commands/ <your-project>/.claude/commands/
```

## MCP 요구사항

| MCP | 용도 | 필수 여부 |
|-----|------|-----------|
| AWS Knowledge MCP | 서비스 문서/가용성 조회 | 권장 |
| CloudFormation MCP | 스택 검증/배포 | Level 2+ |
| Bedrock MCP | AI 규칙 개선 테스트 | 선택 |
