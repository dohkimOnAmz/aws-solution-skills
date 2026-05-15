---
name: unified-customer-profile-builder
display_name: Unified Customer Profile Builder
description: >
  대화형으로 요구사항을 수집하여 AWS Connect Customer Profiles + Entity Resolution
  기반의 고객 통합 프로필 시스템을 맞춤 생성합니다. 산업/도메인 무관.
trigger: "고객 프로필 통합 시스템 만들어줘"
icon: 🧬
inputs:
  - industry (산업/도메인)
  - channels (고객 데이터 채널)
  - pii_fields (매칭에 사용할 PII 필드)
tools:
  - aws_knowledge_mcp_server__aws_search_documentation
  - aws_knowledge_mcp_server__aws_read_documentation
  - aws_knowledge_mcp_server__aws_get_regional_availability
  - run_python
  - file_write
  - file_read
  - start_task
---

# Unified Customer Profile Builder

## 목적
사용자와 대화를 통해 요구사항을 수집하고, AWS Connect Customer Profiles + Entity Resolution
기반의 고객 통합 프로필 시스템을 맞춤 생성한다.

## 참조 지식
이 Skill의 실행에 필요한 아키텍처 지식, 패턴, 예시는 아래에서 참조:
- `shared/reference/architecture.md` — 아키텍처 결정과 이유
- `shared/reference/decision-tree.md` — 조건부 선택 로직
- `shared/reference/aws-services.md` — 서비스/모델 카탈로그 (Bedrock Claude 모델 ID 포함)
- `shared/reference/constraints.md` — 한계와 제약
- **`shared/reference/calculated-attributes.md` — Calc Attribute 정의·동작·디버깅 가이드 (필독)**
- `shared/patterns/cdk-stacks.md` — CDK 스택
- `shared/patterns/lambda-handlers.md` — Lambda 핸들러 (CP 전송, ObjectType 정의 함정 포함)
- `shared/patterns/frontend-pages.md` — React + Tailwind + shadcn/ui 페이지
- `shared/patterns/etl-transforms.md` — Raw → ER input 파이프라인
- `shared/patterns/bedrock-prompts.md` — 모델 선택 + 프롬프트
- `shared/patterns/er-strategies.md` — ER 매칭 전략
- `shared/examples/` — 산업별 golden example

## 워크플로우

### Phase 1: Discovery (대화형 요구사항 수집)

사용자에게 아래 질문을 순서대로 수집한다. 이미 알고 있는 정보는 건너뛴다.

```
1. 산업/도메인: "어떤 산업인가요?" (항공, 호텔, 소매, 금융, 헬스케어, 기타)
2. 채널: "고객 데이터가 들어오는 채널은?" (웹, 앱, 콜센터, OTA, POS, 법인...)
3. 식별 정보: "고객을 식별하는 주요 정보는?" (이름, 이메일, 전화, 생년월일, 멤버십번호...)
4. 거래 데이터: "어떤 거래/이벤트 데이터를 쌓나요?" (예약, 주문, 방문, 청구...)
5. KPI: "핵심 고객 지표는?" (연간 매출, 방문 빈도, AOV, CLV...)
6. 데이터 소스: "고객 데이터가 현재 어디에 있나요?" (기존 DB / CSV파일 / 실시간 이벤트)
   - 기존 DB → Glue Connection (JDBC 타입, 호스트, DB명)
   - 파일 형태 → CSV (소량) 또는 Parquet (대용량, 이미 정형화된 경우)
   - 실시간 → Kinesis
7. 매칭 전략: "데이터 품질은 어떤가요?" (정형화 높음 → Rule / 다양한 변형 → ML)
8. 추가 기능: "Knowledge Graph, Cross-Domain 통합 필요한가요?"
9. 리전/비용: "배포 리전과 비용 제약은?"
10. 데이터 정규화: "PII 정규화가 필요한가요?"
    - 이미 깨끗함 → ETL 불필요
    - 이름/전화 포맷 불일치 → Inline ETL (Lambda 내 처리)
    - 대용량 + 복잡한 변환 (한영 혼용, 릴레이 이메일 등) → Glue ETL Job
    - 릴레이 이메일 처리 방식: tag / exclude / replace_with_null
11. **LLM 모델 선택**: "AI 기능에 어떤 trade-off를 원하나요?" — 카탈로그는 `shared/reference/aws-services.md`
    - 정확도 최우선 (ER 규칙 생성, 복잡 추론) → **Claude Opus 4.7** (`us.anthropic.claude-opus-4-7`, 1M ctx, thinking=adaptive)
    - 균형 (대부분 케이스) → **Claude Sonnet 4** (`anthropic.claude-sonnet-4-20250514-v1:0`)
    - 비용 최소화 (대량 personalization) → **Claude Haiku 4.5** (`anthropic.claude-haiku-4-5-20251001`)
    - **두 개 사용 권장**: ER 규칙 생성 = Opus 4.7, personalization = Haiku 4.5
    - 항상 AWS Knowledge MCP `aws___search_documentation`으로 최신 ID 한 번 더 확인
12. **CP 전송 워크플로우 확인**: "ER 결과를 CP로 보낼 화면이 필요합니다 — 자동 포함됩니다 (Send to CP 페이지). 추가 질문 있나요?"
13. **Calculated Attribute**: "고객 단위 집계 지표가 필요한가요?" (예: 연매출, 누적 박수, ADR 평균, 가장 최근 방문일) — 정의는 `config/schema.yaml`의 `calculated_attributes`
```

⛔ **GATE 1**: 수집된 요구사항을 요약하여 사용자에게 확인 요청.
   승인 시 Phase 2로 진행.

### Phase 2: Architecture Design (설계 결정)

수집된 요구사항 기반으로 아키텍처를 결정한다:

1. **스택 구성 결정** (`shared/reference/decision-tree.md` 참조)
   - Foundation (KMS, DLQ) — 항상 포함
   - Storage (S3) — 항상 포함
   - Profiles (Connect + CP Domain) — 항상 포함
   - Matching (ER + Glue + DynamoDB) — 항상 포함
   - Ingestion (CSV or Kinesis) — mode 선택
   - Auth (Cognito) — 항상 포함
   - Graph (Neptune) — 선택적
   - Cross-Domain — 멀티 도메인일 때만

2. **ER 매칭 전략 결정** (`shared/patterns/er-strategies.md` 참조)
   - Simple Rule: 정확 일치 (loyaltyNumber 등)
   - Advanced Rule: 퍼지 매칭 (이름+이메일, 이름+전화)
   - ML Matching: 데이터 품질 낮을 때

3. **비용 추정** (`shared/reference/aws-services.md` 참조)

4. **리전별 서비스 가용성 확인** (AWS Knowledge MCP 사용)

⛔ **GATE 2**: 설계 결과를 다이어그램/표로 제시. 사용자 승인.

### Phase 3: Code Generation (코드 생성)

승인된 설계를 기반으로 코드를 점진적으로 생성:

**Step 3.1**: 프로젝트 scaffolding
```
bin/app.ts, package.json, tsconfig.json, cdk.json
```

**Step 3.2**: CDK 스택 (`shared/patterns/cdk-stacks.md` 참조)
```
lib/foundation-stack.ts
lib/storage-stack.ts
lib/profiles-stack.ts
lib/matching-stack.ts
lib/ingestion-stack.ts
lib/auth-stack.ts
lib/api-stack.ts
lib/main-stack.ts
[선택] lib/graph-stack.ts
[선택] lib/cross-domain-stack.ts
```

**Step 3.3**: Lambda 핸들러 (`shared/patterns/lambda-handlers.md` 참조)
```
backend/lambdas/matching/handler.ts
backend/lambdas/accuracy/handler.ts
backend/lambdas/ai-agent/handler.ts            ← 모델 ID 동적 (env로 주입)
backend/lambdas/profiles/handler.ts
backend/lambdas/ingestion/handler.ts            ← /build-er-input 엔드포인트로 ETL 트리거
backend/lambdas/profile-import/handler.ts      ← Send to CP - Step 1 (golden record)
backend/lambdas/cp-data-import/handler.ts      ← Send to CP - Step 2 (Reservation/Folio)
backend/lambdas/personalization/handler.ts     ← assembleProfile + assembleFromGolden 모두 ListProfileObjects 호출
backend/custom-resources/upsert-object-type/handler.ts          ← Target rule + delete-recreate
backend/custom-resources/create-calculated-attributes/handler.ts ← Create/Update fallback
backend/glue-scripts/build-er-input.py         ← raw DB → ER input
[선택] backend/lambdas/graph-rag/handler.ts
[선택] backend/lambdas/graph-sync/handler.ts
```

**Step 3.4**: Frontend (`shared/patterns/frontend-pages.md` 참조 — React + Vite + Tailwind + shadcn/ui)
```
frontend/src/pages/
  DashboardPage.tsx
  IngestionPage.tsx               ← Crawl + Run ETL 버튼
  MatchingComparisonPage.tsx
  AccuracyPage.tsx
  AiRulesPage.tsx                 ← LLM 모델 Select
  ProfileImportPage.tsx           ← ★ Send to CP (Step 1 + Step 2 두 카드)
  ProfileViewPage.tsx             ← Calculated Attributes 섹션 + 안내 alert
frontend/src/components/ui/       ← shadcn 컴포넌트 (card, button, badge, alert, table, select, tabs, dialog, skeleton, ...)
frontend/src/components/          ← Layout, PageHeader, StatCard
frontend/src/api/                 ← apiCall + auto-refresh
frontend/src/lib/utils.ts         ← cn() helper
```

**Step 3.5**: 배포 스크립트
```
scripts/deploy.sh
scripts/destroy.sh
scripts/check-prerequisites.sh
```

⛔ **GATE 3**: 생성된 코드가 `cdk synth` 통과하는지 검증.
   AWS Knowledge MCP로 사용된 API/리소스가 유효한지 확인.

### Phase 4: Deploy & Iterate

1. 배포 가이드 제공 (또는 CloudFormation MCP로 직접 배포)
2. 사용자 피드백 수집 → 수정 반영
3. 추가 기능 점진적 추가 (Graph, Cross-Domain 등)

## 생성 규칙

- CDK는 TypeScript, aws-cdk-lib v2 사용
- Lambda는 Node.js 20+ TypeScript, esbuild 번들링 (`@aws-sdk/client-*` v3)
- Frontend는 **React 18 + Vite + Tailwind v3 + shadcn/ui** (Cloudscape 사용 금지)
  - 컴포넌트는 `frontend/src/components/ui/*`에 shadcn 형태로 배치 (Card, Button, Badge, Alert, Skeleton, Table, Select, Tabs, Dialog 등)
  - 아이콘 `lucide-react`, 차트 `recharts`, 토스트 `sonner`
- 도메인 용어는 사용자가 제공한 언어(한/영) 따름
- ER 규칙 이름은 `{MatchKey1}And{MatchKey2}` 형식 (예: NameAndEmail)
- LLM 모델은 Bedrock 최신 Claude 사용 — 자세한 카탈로그는 `shared/reference/aws-services.md` 참조

## MCP 활용 시점

| 시점 | MCP | 호출 |
|------|-----|------|
| 리전 가용성 확인 | AWS Knowledge | `aws_get_regional_availability` |
| 서비스 제약 조회 | AWS Knowledge | `aws_search_documentation` |
| CDK 패턴 확인 | AWS Knowledge | `aws_read_documentation` |
| 생성 코드 검증 | CloudFormation | validate-template |
| 실배포 | CloudFormation | create-stack |
