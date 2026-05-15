# 프레젠테이션 시나리오

---

## 출력 설정

- output_path: `.pptx-generator/ucp-skill-customer-introduction.pptx`
- 발표자: Dohyung Kim, Solutions Architect, AWS
- 발표 시간: 30분
- 청중: 혼합 (의사결정자 + 엔지니어)
- 언어: 한국어 본문 + 영어 기술 용어

---

## 내러티브 구조

### 선택된 구조

- 구조명: **문제 → 솔루션 (Skill 도입을 결정시키는 설득 흐름)**
- 선택 이유: 발표 목적이 "고객이 UCP Skill로 프로덕션 구축을 결정"하는 것. 단순 교육이 아니라 **현재의 어려움을 직시하고 → Skill이 그 어려움을 어떻게 제거하는지 보여주고 → 단계적 시작 경로를 제시**하는 흐름이 가장 효과적.
- 기술 깊이 그라디언트: 개념(Part 1) → 구조(Part 2) → 흐름(Part 3) → AWS 매핑·옵션 비교(Part 4) → 행동(Part 5)
- Accent 색상 조합 (콘텐츠 성격 기반):
  - Part 1 "Skill이란?" → TEAL (지식·흐름)
  - Part 2 "UCP Skill 구현" → DEEP_BLUE (구조·기술)
  - Part 3 "사용 방법" → ACCENT_ORANGE (행동·에너지)
  - Part 4 "AWS 배포" → AWS_ORANGE 계열 (관습)
  - Part 5 "시작하기" → LIME (성장·시작)

### 검토한 후보

| 후보 | 구조명           | 장점                                          | 단점                                  | 적합도 |
| ---- | ---------------- | ---------------------------------------------- | ------------------------------------- | ------ |
| A    | 문제 → 솔루션    | 설득력 강함, 청중 공감 형성                     | 도입부 길어질 수 있음                  | ★★★    |
| B    | 큰 → 작은        | 정보 전달 효율적                                | "왜 도입해야 하나"의 동기 약함         | ★★☆    |
| C    | 시간 순서        | 구현 단계 명확                                  | 청중이 결정 단계가 아님                | ★☆☆    |

---

## 슬라이드 목차 (총 26장)

### 오프닝

#### 슬라이드 1: Unified Customer Profile을 Skill로 구축하기

- 유형: Cover
- 핵심 메시지: —
- 시각화 의도: —

#### 슬라이드 2: 어젠다

- 유형: Agenda
- 핵심 메시지: 5 파트 미리보기
- 시각화 의도: —

### Section 1: Skill이란 무엇인가

#### 슬라이드 3: Skill — AI 도구의 휴대용 능력 확장

- 유형: Section
- 핵심 메시지: AI 도구가 공유하는 표준 + 도구별 고유 확장으로 도메인 작업을 모듈화한다
- 시각화 의도: —

#### 슬라이드 4: 채팅 반복은 한계, Skill로 절차를 외부화

- 유형: Content
- 핵심 메시지: 같은 다단계 절차를 채팅에 반복 붙여넣지 말고 SKILL.md로 외부화하면 lazy-load되어 컨텍스트 비용은 0에 가깝다
- 시각화 의도: 비포(반복 붙여넣기) vs 애프터(SKILL.md 호출) 2컬럼 비교

#### 슬라이드 5: CLAUDE.md(상시) vs SKILL.md(호출 시) 분업

- 유형: Content
- 핵심 메시지: CLAUDE.md는 사실·컨벤션을 항상 로드, SKILL.md는 절차·다단계 작업을 호출 시에만 로드 — 토큰 예산을 효율적으로 분배
- 시각화 의도: 2컬럼 비교 표 (로드 시점 / 토큰 비용 / 적합 콘텐츠)

#### 슬라이드 6: Agent Skills — 도구 무관 개방형 표준

- 유형: Content
- 핵심 메시지: agentskills.io 표준은 SKILL.md 명세를 정의하고, Claude Code는 자기 고유 기능(호출 제어, subagent, 동적 컨텍스트)으로 확장한다
- 시각화 의도: 동심원 다이어그램 (표준 코어 + 도구별 확장 링)

#### 슬라이드 7: 한 지식베이스 + 여러 도구 어댑터

- 유형: Content
- 핵심 메시지: shared/ 지식 한 번 작성하고 Quick / Kiro / Claude Code 어댑터를 얇게 얹으면 도구를 옮겨도 출력 품질이 일관된다
- 시각화 의도: 중앙 shared/ + 3 도구 어댑터로 분기되는 트리

### Section 2: UCP Skill 어떻게 구현되었나

#### 슬라이드 8: UCP — 360° 통합 고객 프로필

- 유형: Section
- 핵심 메시지: AWS Connect Customer Profiles + Entity Resolution + Bedrock 조합의 산업 무관 표준 패턴
- 시각화 의도: —

#### 슬라이드 9: UCP가 풀어야 할 문제 — 분산된 고객 데이터

- 유형: Content
- 핵심 메시지: 웹·앱·콜센터·OTA·POS 등 채널마다 흩어진 고객 데이터를 동일 고객으로 식별·통합·집계해야 한다
- 시각화 의도: 좌측 다채널 입력 → 중앙 ER → 우측 360° 프로필 플로우

#### 슬라이드 10: 직접 구축의 함정 — 처음 만든 사람만 아는 14개 제약

- 유형: Content
- 핵심 메시지: Connect Instance quota 2개·ER ML 리전·Bedrock apac. 접두사·OTA relay email·CP↔Domain 수동 association 등 처음 만든 사람만 아는 함정
- 시각화 의도: 함정 카드 그리드 (4-5개 대표 함정 강조)

#### 슬라이드 11: UCP Skill 패키지 구조 — 한 지식베이스, 세 어댑터

- 유형: Content
- 핵심 메시지: shared/(reference + patterns + examples) ~96 KB가 무게중심, quick/ kiro/ claude-code/ 어댑터는 얇은 진입점
- 시각화 의도: 디렉토리 트리 + 사이즈 표시 (shared/ 강조)

#### 슬라이드 12: 9-레이어 + Gate 5단계로 안전한 생성

- 유형: Content
- 핵심 메시지: Foundation→Storage→Profiles→Matching→Ingestion→ETL→Auth→API→[Graph]→[Cross-Domain]을 Discovery→Design→Generate→Validate→Deploy 5게이트로 점진 생성
- 시각화 의도: 9-레이어 좌측 스택 + 5-게이트 우측 가로 플로우

#### 슬라이드 13: Bedrock이 ER 규칙을 만들고, 사람은 검증한다

- 유형: Content
- 핵심 메시지: 데이터 프로파일을 Bedrock이 분석해 매칭 규칙을 자동 생성하고, 매칭 쌍 미리보기·precision/recall·false positive 경고로 사용자가 HITL 검증
- 시각화 의도: 사이클 다이어그램 (데이터 → AI 생성 → HITL 검증 → 적용 → 피드백)

### Section 3: 어떻게 사용하는가

#### 슬라이드 14: 사용 방법 — 도구 선택 후 한 줄 트리거

- 유형: Section
- 핵심 메시지: 워크스페이스에 Skill을 import하고 한 줄 자연어 명령으로 시작
- 시각화 의도: —

#### 슬라이드 15: 도구별 import 방법

- 유형: Content
- 핵심 메시지: Quick은 Settings에서 import, Kiro는 .kiro/steering/에 복사, Claude Code는 CLAUDE.md + commands 복사 — 3가지 모두 1분 이내 셋업
- 시각화 의도: 3컬럼 카드 (도구·명령·소요시간)

#### 슬라이드 16: Discovery → Deploy 워크플로우

- 유형: Content
- 핵심 메시지: 산업·채널·PII·KPI 10개 질문에 답하면 Skill이 아키텍처를 결정하고 코드를 생성하며, cdk synth 검증 후 배포까지 안내
- 시각화 의도: 5-게이트 가로 플로우 + 각 게이트의 사용자 행동 캡션

#### 슬라이드 17: 산업별 골든 레퍼런스 3종

- 유형: Content
- 핵심 메시지: Travel(3 도메인 + Cross-Domain + Graph), Hotel(OTA relay email + 단일 도메인), Retail(게스트 구매 + RFM + Kinesis 실시간) — 도메인 추가는 examples/ 한 장으로
- 시각화 의도: 3컬럼 산업 카드 (특징·매칭 전략·핵심 도전)

### Section 4: AWS에 어떻게 배포되나

#### 슬라이드 18: AWS 배포 — 베이스 + 옵션 조합

- 유형: Section
- 핵심 메시지: Skill이 생성하는 AWS 아키텍처는 9-레이어 베이스 + Pipeline / Graph / Cross-Domain 옵션의 조합
- 시각화 의도: —

#### 슬라이드 19: 베이스 아키텍처 — 9-레이어 → AWS 서비스 매핑

- 유형: Content
- 핵심 메시지: KMS·SQS·S3·Connect·CP·ER·Glue·DynamoDB·Cognito·API GW·Lambda·Bedrock — 모든 레이어가 관리형 서비스, 모든 옵션 OFF 시 Minimal 구성($50-100/월)
- 시각화 의도: AWS 아이콘 다이어그램 (9-레이어 베이스 전체 — Pipeline=CSV, Graph=off, Cross-Domain=off 가정)

#### 슬라이드 20: 옵션 1 — Ingestion Pipeline 5종 비교

- 유형: Content
- 핵심 메시지: CSV(데모/PoC) / Parquet(대용량 배치) / Kinesis(실시간) / Glue Connection(기존 DB 연동) / Hybrid(초기 마이그레이션 + 실시간) — 데이터 위치와 빈도로 결정
- 시각화 의도: 5개 미니 다이어그램을 가로/그리드 배치 (각 모드의 데이터 흐름 + 핵심 사용 케이스 캡션)

#### 슬라이드 21: 옵션 2 — Knowledge Graph on/off 비교

- 유형: Content
- 핵심 메시지: Graph OFF는 개별 프로필만, Graph ON은 Neptune + GraphRAG로 관계 분석 추가(가족/동행자, 법인-개인 연결, 영향력) — +$300/월
- 시각화 의도: 좌우 2-up 비교 다이어그램 (좌: Graph OFF, 우: Graph ON에 Neptune·GraphRAG·VPC 추가 강조)

#### 슬라이드 22: 옵션 3 — Cross-Domain on/off 비교

- 유형: Content
- 핵심 메시지: Single Domain은 단일 CP Domain, Cross-Domain ON은 사업부별 독립 CP Domain + Platform-level 통합 도메인 + Cross-Domain ER — Connect Instance quota 증가 신청 필요
- 시각화 의도: 좌우 2-up 비교 다이어그램 (좌: 단일 도메인, 우: 멀티 도메인 + Platform 통합 레이어)

#### 슬라이드 23: 산업별 권장 옵션 조합

- 유형: Content
- 핵심 메시지: Travel = Pipeline:Hybrid + Graph:ON + CrossDomain:ON ($400-600). Hotel = Pipeline:CSV + Graph:OFF + CrossDomain:OFF ($50-100). Retail = Pipeline:Kinesis + Graph:OFF + CrossDomain:OFF ($100-200)
- 시각화 의도: 3행 비교 표 (산업 × 3 옵션 + 월 비용 + 핵심 도전 컬럼)

#### 슬라이드 24: 비용 티어 — Minimal / Standard / Full

- 유형: Content
- 핵심 메시지: Minimal $50-100(베이스) → Standard $100-200(+Kinesis +ML) → Full $400-600(+Neptune +Cross-Domain). 시작은 Minimal, 단계적 확장
- 시각화 의도: 3개 카드 + 강조 숫자 (월 비용 + 포함 옵션 + 적합 시나리오)

### Section 5: 시작하기

#### 슬라이드 25: 다음 단계 — PoC(1주) → MVP(2주) → 프로덕션(4-8주)

- 유형: Content
- 핵심 메시지: 1주 안에 Skill 도입 검증 PoC, 2주 안에 1개 산업 도메인 MVP, 4-8주 안에 프로덕션 — 단계마다 명확한 산출물과 게이트
- 시각화 의도: 3단계 가로 타임라인 + 각 단계의 산출물·검증 항목

#### 슬라이드 26: Thank You

- 유형: ThankYou
- 핵심 메시지: —
- 시각화 의도: —

---

## 전체 흐름 요약

청중을 Skill 개념(Part 1)으로 끌어들이고, UCP Skill의 *구체적 구조*(Part 2)에서 이 Skill이 단순한 템플릿이 아니라 *지식 + 안전장치 + AI 협업*임을 보이고, *사용법*(Part 3)으로 진입 비용이 1분임을 증명한다. *AWS 배포*(Part 4)에서 베이스 아키텍처(슬라이드 19) 위에 **세 가지 선택 옵션(Pipeline 5종 / Graph on-off / Cross-Domain on-off)이 어떻게 다른 아키텍처를 만들어내는지**를 시각적 비교로 보여주고(20-22), 산업별 권장 조합(23)과 비용 스펙트럼(24)으로 구체화한다. 마지막 *행동*(Part 5)으로 PoC부터 시작하는 단계적 경로를 제시한다.

---

## 레이아웃 다양성 검증

레이아웃 패턴은 Phase 3(slide-NN.md)에서 결정한다.

- 연속 3장 이상 같은 패턴: Phase 3 완료 후 검증
- 텍스트 전용 슬라이드: 회피 — 모든 Content는 시각화 요소 포함
- 섹션 수: **5개** (최대 7개 이내)
- Part 4(슬라이드 18-24, 7장)은 옵션 비교에 다양한 시각화 패턴 사용:
  - 슬라이드 19: 베이스 9-레이어 아키텍처 (단일 큰 다이어그램)
  - 슬라이드 20: Pipeline 5종 그리드 비교 (작은 다이어그램 5개)
  - 슬라이드 21: Graph 2-up 좌우 비교 (전/후 다이어그램)
  - 슬라이드 22: Cross-Domain 2-up 좌우 비교 (전/후 다이어그램)
  - 슬라이드 23: 3-행 비교 표
  - 슬라이드 24: 3-컬럼 카드 (비용 티어)
