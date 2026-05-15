# 소스 데이터 분석

---

## 메타 정보

- 소스 경로: `/Users/dohkim/Data/Code/tnh-demo/new/unified-customer-profile-skill/` + LLM Wiki vault (14 source summaries)
- 분석 일시: 2026-05-14
- 문서 수: 25+ (UCP Skill 패키지 14 문서 + Wiki 관련 page 11+)

---

## 문서별 요약

### `unified-customer-profile-skill/README.md`
- 주제: UCP Skill 패키지 개요 — "템플릿 배포가 아닌 지식 배포"
- 핵심 키워드: Skill, 멀티-도구(Quick/Kiro/Claude Code), shared/, MCP, Gate 패턴
- 데이터 포인트: 3개 도구 어댑터, ~96 KB shared/patterns, 5단계 Gate, 9 레이어
- 분량: ~80줄

### `shared/reference/architecture.md`
- 주제: UCP 9-레이어 아키텍처 + WHY
- 핵심 키워드: Foundation, Storage, Profiles, Matching, Ingestion, ETL, Auth, API, Graph, Cross-Domain
- 데이터 포인트: 9개 레이어, IaC=CDK, 데이터 교환=S3, 결과=DynamoDB, AI=Bedrock, Auth=Cognito
- 분량: ~120줄

### `shared/reference/decision-tree.md`
- 주제: 6개 결정 분기 (Ingestion / ER / Graph / Cross-Domain / 프론트 / 비용)
- 데이터 포인트: CSV/Parquet/Kinesis/Glue/Hybrid 5종, ER 3 타입(Simple/Advanced/ML), 비용 티어 3종 ($50-100 / $100-200 / $400-600)

### `shared/reference/aws-services.md`
- 주제: 서비스별 quota·비용·리전 가용성
- 데이터 포인트: Connect Instance quota=2(default), ER ML 리전 제약, Neptune 최소 ~$300/월, Bedrock `apac.` 접두사

### `shared/patterns/` (6개 파일, ~96 KB)
- cdk-stacks.md (14 KB) — Foundation/Storage/Ingestion/Profiles/Matching CDK 패턴
- lambda-handlers.md (17 KB) — Matching/Accuracy/AI-Agent 핸들러 템플릿
- frontend-pages.md (18 KB) — React + Cloudscape 페이지 + API 연결 자동화
- er-strategies.md (19 KB) — Bedrock 자동 ER 규칙 생성 + HITL
- bedrock-prompts.md (12 KB) — 검증된 프롬프트 템플릿
- etl-transforms.md (18 KB) — PII 정규화 (Lightweight/Glue ETL)

### `shared/examples/` (Travel/Hotel/Retail 골든 레퍼런스)
- Travel: 3 도메인 + Cross-Domain + Graph (Ecosystem CLV 시너지 1.2~1.5x)
- Hotel: 단일 도메인 + OTA relay email 처리
- Retail: 게스트 구매 + RFM 세그먼트

### Wiki: `summary-claude-code-skills.md` (Anthropic 공식)
- 핵심 키워드: SKILL.md, Agent Skills 표준, agentskills.io, lazy-loaded, allowed-tools, disable-model-invocation
- 데이터 포인트: 4단계 위치 (Enterprise/Personal/Project/Plugin), 3-state 호출 제어 매트릭스

### Wiki: `summary-unified-customer-profile-skill.md`
- 한 지식베이스 + 3 어댑터 패턴, 9-레이어, Gate 패턴, Bedrock 자동 ER + HITL

### Wiki: `summary-woongjin-ai-dlc-bookcurator.md` (검증 사례)
- 데이터 포인트: 2일 MVP, 4배 속도, 70% 비용 절감 — Skill 채택 효과의 실증

---

## 시각화 가능 요소

| #   | 유형         | 설명                                                          | 소스 파일                       | 시각화 방식 제안                |
| --- | ------------ | ------------------------------------------------------------- | ------------------------------- | -------------------------------- |
| 1   | 아키텍처     | UCP 9-레이어 다이어그램                                       | architecture.md                 | 좌→우 레이어 플로우 다이어그램   |
| 2   | 비교         | CLAUDE.md(상시) vs SKILL.md(lazy)                             | summary-claude-code-skills      | 2컬럼 비교 표                    |
| 3   | 비교         | 3-도구 어댑터 (Quick/Kiro/Claude Code)                        | unified-customer-profile-skill  | 3컬럼 카드                        |
| 4   | 프로세스     | Gate 패턴 5단계 (Discovery→Design→Generate→Validate→Deploy)   | UCP README                      | 5단계 가로 플로우                 |
| 5   | 아키텍처     | AWS 배포 아키텍처 (Travel/Hotel/Retail 변형 1개 선택)         | architecture.md + examples      | AWS 아이콘 다이어그램             |
| 6   | 통계         | 비용 티어 3종 (Minimal/Standard/Full)                         | decision-tree.md                | 비교 표 + 강조 숫자               |
| 7   | 통계         | AI-DLC 적용 효과 (4배 속도, 70% 비용)                          | summary-woongjin               | 강조 숫자 + 비포-애프터           |
| 8   | 프로세스     | Bedrock ER 자동 규칙 생성 + HITL 루프                         | er-strategies.md                | 사이클 다이어그램                 |
| 9   | 비교         | Vibe Coding vs AI-DLC vs UCP Skill (구조 비교)                 | summary-aws-ai-dlc + UCP        | 3행 비교 표                       |
| 10  | 데이터       | 산업별 ER 매칭 규칙 조합                                      | examples/ + er-strategies.md    | 산업별 매칭 규칙 표               |
| 11  | 아키텍처     | shared/ + 어댑터 분리 구조 (Skill 패키지 내부)                 | UCP README                      | 트리 다이어그램                   |
| 12  | 통계         | shared/ 무게중심 (~96 KB patterns vs ~14 KB 어댑터)            | UCP analysis                    | 막대 차트 또는 도넛               |

---

## 핵심 주제 구조

- **Part 1: Skill이란 무엇인가**
  - Skill의 정의 (Claude/Kiro 공통)
  - CLAUDE.md vs SKILL.md (lazy-loaded)
  - Agent Skills 개방형 표준 (agentskills.io)
  - Multi-Tool Skill Package 패턴
- **Part 2: UCP Skill 구현 방식**
  - UCP 시스템이란 (AWS Connect CP + ER + Bedrock)
  - 패키지 구조 (shared/ + 3-도구 어댑터)
  - 9-레이어 아키텍처
  - Gate 패턴 + Bedrock 자동 ER
- **Part 3: UCP Skill 사용 방법**
  - 도구별 import (Quick / Kiro / Claude Code)
  - Discovery → Deploy 워크플로우
  - 산업별 골든 레퍼런스 (Travel/Hotel/Retail)
- **Part 4: AWS 프로덕션 배포 아키텍처**
  - 9-레이어 → AWS 서비스 매핑
  - 비용 티어 (Minimal/Standard/Full)
  - 하드 제약 (Connect quota, ER ML 리전, Bedrock apac.)
- **Part 5: 시작하기**
  - 다음 단계 + 참고 자료

---

## 소스 간 관계

- **UCP Skill 패키지(README + shared/)** = 1차 자산. 모든 구체 정보의 출처.
- **Wiki source summaries** = 2차 메타 자산. 패턴 추상화와 cross-domain 연결.
- **Claude Code Skills 공식 문서** = Skill 자체의 표준 명세. Part 1의 근거.
- **웅진씽크빅 AI-DLC 사례** = 효과 증명 (4배 속도, 70% 비용). 설득력 보강.
- **Travel/Hotel/Retail examples** = AWS 배포 아키텍처의 변형 자료. Part 4용.

세 자산은 보완 관계 — 표준(Claude Code 문서) → 구현(UCP Skill) → 효과 증명(웅진 사례) → 패턴 추상(Wiki).

---

## 누락/보완 필요 사항

- **실제 배포 시간/비용 사례 부재**: UCP를 Skill로 생성한 *고객 사례*는 위키에 없음. AI-DLC 사례(웅진)를 *비유*로 사용.
- **고객 측 사전 준비 체크리스트 부재**: AWS 계정, Connect quota 증가 신청, 데이터 소스 인벤토리 등 — 슬라이드에서 추가 작성.
- **장기 운영(Day 2)**: Skill로 1차 생성 후 운영·확장 가이드는 source 자체에서도 얇음. "Day 2는 후속 자료" 정도로만 언급.
