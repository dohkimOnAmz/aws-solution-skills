# Bedrock Prompts — AI Rule Generation & Improvement

기존 데모에서 검증된 Bedrock 프롬프트 패턴. 규칙 생성/개선 시 그대로 활용.

## 모델 선택 (런타임 동적)

**모델 ID는 하드코딩하지 않는다.** 사용자가 UI에서 모델을 선택하거나(`AiRulesPage`의 Select 컴포넌트), 환경변수로 default를 지정. 카탈로그·가격은 `shared/reference/aws-services.md` 참조.

```typescript
// backend/lambdas/_shared/bedrock.ts
import { BedrockRuntimeClient, InvokeModelCommand } from '@aws-sdk/client-bedrock-runtime';

const br = new BedrockRuntimeClient({});

const DEFAULT_MODEL = process.env.BEDROCK_MODEL_ID ?? 'anthropic.claude-sonnet-4-20250514-v1:0';
const PERSONALIZATION_MODEL = process.env.PERSONALIZATION_MODEL_ID ?? 'anthropic.claude-haiku-4-5-20251001';

export async function invokeClaude(
  prompt: string,
  opts: { modelId?: string; maxTokens?: number; thinking?: 'adaptive' | 'enabled' | 'off' } = {}
) {
  const modelId = opts.modelId ?? DEFAULT_MODEL;

  // Opus 4.7 only supports adaptive thinking — guardrail
  let thinking: any = undefined;
  if (opts.thinking && opts.thinking !== 'off') {
    if (modelId.includes('opus-4-7')) thinking = { type: 'adaptive' };
    else thinking = { type: 'enabled', budget_tokens: 4000 };
  }

  const body = {
    anthropic_version: 'bedrock-2023-05-31',
    max_tokens: opts.maxTokens ?? 8192,
    messages: [{ role: 'user', content: prompt }],
    ...(thinking && { thinking }),
  };

  const r = await br.send(new InvokeModelCommand({
    modelId, body: JSON.stringify(body),
    contentType: 'application/json', accept: 'application/json',
  }));

  const parsed = JSON.parse(new TextDecoder().decode(r.body));
  return parsed.content?.find((c: any) => c.type === 'text')?.text ?? '';
}
```

### Lambda 환경변수 (CDK common-env)

```typescript
const commonEnv = {
  // ...
  BEDROCK_MODEL_ID: schemaConfig.features.ai.bedrock.model_id,                              // ER 규칙 생성용
  BEDROCK_THINKING_TYPE: schemaConfig.features.ai.bedrock.thinking_type ?? 'adaptive',
  PERSONALIZATION_MODEL_ID: schemaConfig.features.ai.bedrock.personalization_model_id
                            ?? schemaConfig.features.ai.bedrock.model_id,                   // 마케팅용 (저비용)
};
```

### `config/schema.yaml` — 모델 선택 가능하게

```yaml
features:
  ai:
    bedrock:
      # 사용자가 Discovery Phase에서 선택. 카탈로그는 reference/aws-services.md 참조.
      model_id: us.anthropic.claude-opus-4-7              # 정확도 우선 (ER 규칙)
      thinking_type: adaptive                              # Opus 4.7은 'adaptive'만
      max_tokens: 8192
      personalization_model_id: anthropic.claude-haiku-4-5-20251001   # 비용 우선 (대량 호출)
```

### Frontend — 모델 선택 UI (선택적)

`frontend/src/pages/AiRulesPage.tsx`의 헤더에 Select를 두어, 사용자가 호출 직전 모델을 바꿀 수 있게 한다 (`shared/patterns/frontend-pages.md`의 AiRulesPage 섹션 참조). API에는 `body.modelId`로 전달, Lambda는 `invokeClaude(prompt, { modelId: body.modelId })`.



## 프롬프트 구성 원칙

Bedrock에 전달하는 프롬프트는 아래 섹션들을 조합하여 구성:

```
[ER 동작 원리 설명] + [실제 데이터 품질 분석] + [스키마 정보] +
[현재 규칙] + [사용자 피드백] + [현재 매칭 메트릭] + [미매칭 패턴 분석] +
[출력 형식 지정]
```

**핵심**: 프롬프트에 실제 데이터 컨텍스트를 최대한 넣어야 정확한 규칙이 나옴.

---

## 1. 데이터 품질 분석 섹션

Lambda에서 S3의 실제 데이터를 분석하여 동적으로 생성하는 부분:

```
데이터 품질 분석 (총 {totalRecords}개 레코드):

Column: firstname
  빈 값 비율: 2.5%
  상위 빈도: "KIM" (8.3%), "LEE" (5.1%), "PARK" (3.2%)

Column: email
  빈 값 비율: 15.0%
  상위 빈도: "guest-abc@booking.com" (1.2%), ...

Column: phone
  빈 값 비율: 5.0%
  상위 빈도: ...

실제 데이터 샘플 (처음 10개):
Record 1: {"variantid":"V001","firstname":"KIM","lastname":"MINHO","email":"minho@gmail.com","phone":"01012345678","sourcechannel":"WEB"}
Record 2: {"variantid":"V002","firstname":"김민호","lastname":"","email":"minho.kim@naver.com","phone":"010-1234-5678","sourcechannel":"OTA"}
...
```

## 2. 스키마 정보 섹션

ER Schema Mapping에서 가져온 필드 정의:

```
Entity Resolution Schema Mapping:
- variantid (type: UNIQUE_ID)
- firstname (type: NAME_FIRST, matchKey: Name, group: FullName)
- lastname (type: NAME_LAST, matchKey: Name, group: FullName)
- email (type: EMAIL_ADDRESS, matchKey: Email)
- phone (type: PHONE_NUMBER, matchKey: Phone)
- dateofbirth (type: DATE, matchKey: DateOfBirth)
- loyaltynumber (type: STRING, matchKey: LoyaltyNumber)
- sourcechannel (type: STRING)
```

## 3. Advanced Rule 생성 프롬프트 (전체 템플릿)

```
Entity Resolution Advanced Rule 매칭 규칙을 추천해주세요.

## ER Advanced Rule 동작 원리

### 기본 특성
- Advanced Rule은 조건부 퍼지 매칭을 지원합니다.
- 각 규칙은 conditionString으로 정의되며, 다양한 비교 함수를 사용할 수 있습니다.
- 규칙 간 관계는 OR: 하나라도 매치하면 동일 고객으로 판정.
- 조건 내부는 AND, OR, 괄호를 사용하여 복합 조건 구성 가능.

### 사용 가능한 비교 함수
- Exact(field): 정확 일치
- Cosine(field, threshold): 코사인 유사도 (0.0~1.0, 이름에 효과적)
- Levenshtein(field, threshold): 편집 거리 (전화번호/이메일 오타에 효과적)
- Soundex(field): 발음 유사 매칭 (영문 이름)
- DateDifference(field, days): 날짜 차이 범위

### 조건 문법
```
Exact(Email) AND Cosine(Name, 0.8)
(Exact(Phone) OR Levenshtein(Phone, 2)) AND Cosine(Name, 0.7)
Exact(LoyaltyNumber)
```

## 아래는 실제 데이터 기반으로 자동 생성된 컨텍스트입니다

{schemaSection}
{currentRulesSection}
{dataQualitySection}
{sampleSection}
{feedbackSection}
{metricsSection}
{unmatchedSection}

## 출력 형식

추천 규칙을 아래 JSON 형식으로 제공하세요:

```json
{
  "rules": [
    {
      "ruleName": "NameAndEmail",
      "conditionString": "Cosine(Name, 0.8) AND Exact(Email)",
      "description": "이름 유사 + 이메일 정확 일치",
      "estimatedPrecision": 0.95,
      "estimatedRecall": 0.7,
      "rationale": "이메일이 있는 대부분의 레코드를 커버. 이름은 코사인으로 한영 변형 허용."
    },
    {
      "ruleName": "NameAndPhone",
      "conditionString": "Cosine(Name, 0.7) AND Levenshtein(Phone, 2)",
      "description": "이름 유사 + 전화번호 유사",
      "estimatedPrecision": 0.9,
      "estimatedRecall": 0.6,
      "rationale": "OTA relay email 우회. 전화번호 형식 차이 허용 (하이픈/국가코드)."
    }
  ],
  "reasoning": "전체 전략 설명",
  "dataQualityAnalysis": "데이터 특성 분석 요약"
}
```

## 규칙 설계 가이드라인

1. **빈값 필드 주의**: 빈 값 비율이 높은 필드(>30%)는 단독 조건으로 부적합
   - Email이 15% 비어있으면 → Email 단독 규칙은 85% 레코드만 커버
   - 이름이 빈 경우 → DOB+Phone 또는 DOB+Email 조합 필요

2. **코사인 vs 정확 일치**:
   - 이름: Cosine(Name, 0.7~0.8) 권장 (한영 혼용, 성이름 순서 차이)
   - 이메일: Exact(Email) 권장 (도메인 다르면 다른 사람)
   - 전화: Levenshtein(Phone, 2) 권장 (형식 차이)

3. **릴레이 이메일 처리**:
   - OTA relay (guest-xxx@booking.com) → Email 조건 의미 없음
   - 이 채널은 Name+Phone으로 폴백

4. **False Positive 방지**:
   - "김민호"는 한국에 매우 많음 → Name 단독 절대 불가
   - 반드시 Name + (Email OR Phone OR DOB) 조합

5. **기존 매칭에 영향 금지**:
   - 새 규칙 추가 시 기존 매칭 그룹이 깨지면 안 됨
   - 미매칭 커버 규칙은 OR로 추가하거나 별도 규칙으로 구성
```

## 4. Simple Rule 생성 프롬프트 (핵심 부분)

```
Entity Resolution Simple Rule 매칭 규칙을 추천해주세요.

## ER Simple Rule 동작 원리 (공식 문서 기반)

### 기본 특성
- Simple Rule은 AND + ExactMatch만 지원합니다. OR, 괄호, Fuzzy 매칭은 사용할 수 없습니다.
- 각 규칙은 matchingKeys 배열로 정의되며, 지정된 필드들이 모두 정확히 일치해야 매칭됩니다.
- 데이터 정규화(applyNormalization)가 활성화되어 대소문자, 전화번호 형식, 주소 약어 등은 자동 처리됩니다.

### Waterfall 매칭 (규칙 순서가 중요)
- 규칙은 위에서 아래로 순서대로 적용됩니다 (hierarchical waterfall).
- 규칙 1로 A↔B가 매칭되고, 규칙 2로 B↔C가 매칭되면, A,B,C는 하나의 그룹으로 전이적 병합됩니다.
- 엄격한 규칙(키가 많은)을 먼저 배치하면 높은 Precision으로 기본 매칭을 확보하고, 느슨한 규칙(키가 적은)이 나머지를 보완합니다.

### 전이적 병합의 핵심
- Record A: LoyaltyNumber="L001", Email="a@gmail.com", Phone="01012345678"
- Record B: LoyaltyNumber="L001" (규칙1: LoyaltyNumber → A↔B 매칭)
- Record C: Email="a@gmail.com" (규칙2: Email → A↔C 매칭)
- 결과: A, B, C 모두 같은 그룹 (A를 통해 전이적 연결)

## 출력 형식

```json
{
  "rules": [
    {
      "ruleName": "LoyaltyNumberMatch",
      "matchingKeys": ["loyaltynumber"],
      "rationale": "고유 식별자. 빈값 비율 낮고 유니크 비율 높음.",
      "estimatedPrecision": 0.99,
      "estimatedRecall": 0.4
    },
    {
      "ruleName": "NameAndEmail",
      "matchingKeys": ["firstname", "lastname", "email"],
      "rationale": "이름+이메일 조합. 정규화로 대소문자 처리됨.",
      "estimatedPrecision": 0.95,
      "estimatedRecall": 0.6
    }
  ],
  "ruleOrder": ["LoyaltyNumberMatch가 먼저 (가장 엄격)", "NameAndEmail이 그 다음"]
}
```
```

## 5. 사용자 검증 피드백 섹션

HITL에서 사용자가 매칭 결과를 검증한 후의 피드백:

```
## 사용자 검증 피드백 (중요 — 규칙 개선에 반영하세요)

### 잘못된 매칭 (False Positive) — 3건
현재 규칙이 다른 고객을 같은 고객으로 잘못 매칭한 케이스입니다. 해당 규칙을 강화(조건 추가)해야 합니다.
- Record A: {"firstname":"KIM","lastname":"MINHO","email":"minho1@gmail.com","phone":"01012345678"}
  Record B: {"firstname":"KIM","lastname":"MINHO","email":"minho2@gmail.com","phone":"01087654321"}
  → 동명이인! Name만으로는 부족, Email 또는 Phone도 일치해야 함

### 놓친 매칭 (False Negative) — 2건
현재 규칙이 같은 고객을 매칭하지 못한 케이스입니다. 이 쌍들의 공통 필드 패턴을 분석하여 새로운 규칙을 추가하세요.
- Record A: {"firstname":"MINHO","lastname":"KIM","email":"minho@gmail.com","phone":""}
  Record B: {"firstname":"KIM","lastname":"MINHO","email":"minho@gmail.com","phone":"01012345678"}
  → 이름/성 스왑! Cosine(Name) 사용하면 해결됨
```

## 6. 미매칭 레코드 분석 섹션

ER 실행 후 단독으로 남은 레코드와 가장 유사한 그룹의 비교:

```
## 미매칭 레코드 분석 (중요 — 이 패턴을 커버하는 규칙을 추가하세요)
총 15건의 미매칭 레코드가 있습니다. 각 미매칭 레코드와 가장 유사한 매칭 그룹 레코드를 비교한 결과:

미매칭 레코드: {"firstname":"","lastname":"","email":"minho@gmail.com","phone":"01012345678","sourcechannel":"OTA"}
유사 그룹 레코드: {"firstname":"KIM","lastname":"MINHO","email":"minho@gmail.com","phone":"01012345678","sourcechannel":"WEB"}
일치 필드: email, phone
불일치 원인: firstname: 빈값 (원본: KIM), lastname: 빈값 (원본: MINHO)
빈값 필드: firstname, lastname — 이 필드들은 이 레코드에서 빈값이므로 매칭 조건에 사용해도 매칭되지 않습니다

위 패턴을 분석하여 미매칭 레코드를 커버하는 규칙을 추가하세요:
- 미매칭 레코드의 빈값 필드는 해당 레코드의 매칭 조건으로 사용할 수 없습니다. 빈값이 아닌 필드만으로 조건을 구성하세요.
- 이름이 빈값인 경우 → Name/Cosine(Name)/Soundex(Name) 조건 없이 DOB+Phone 또는 DOB+Email만으로 매칭하는 규칙 필요
- 이름이 스왑된 경우 → Cosine(Name) 사용 (Exact(Name) 대신)
- 전화번호 형식 차이 → Levenshtein(Phone,threshold) 사용
- 이메일 도메인 차이 → Levenshtein(Email,threshold) 사용
```

## 7. 현재 매칭 메트릭 섹션 (가드레일)

```
## 현재 매칭 결과 (중요 — 이 지표를 악화시키지 마세요)
- 총 레코드: 500개
- 매칭 그룹 (2개+ 레코드): 120개
- 미매칭 프로필 (단독): 80개
- 의심 그룹 (채널 수 초과): 3개

추천 규칙의 목표:
1. 미매칭 프로필을 줄이면서 (Recall 개선)
2. 의심 그룹을 늘리지 않아야 합니다 (Precision 유지)
3. 기존 매칭 그룹 수를 유지하거나 늘려야 합니다
4. 미매칭 프로필이 현재보다 증가하는 규칙은 추천하지 마세요
```

---

## 구현 참고사항

### 프롬프트 조합 순서
```typescript
function buildPrompt(...): string {
  // 1. ER 동작 원리 (정적 — Rule 타입에 따라 Simple/Advanced)
  // 2. 스키마 정보 (ER GetSchemaMapping에서 동적 로드)
  // 3. 현재 규칙 (있으면 — 개선 모드)
  // 4. 데이터 품질 분석 (S3 데이터에서 동적 생성)
  // 5. 샘플 레코드 (S3에서 10개 추출)
  // 6. 사용자 검증 피드백 (DynamoDB에서 로드 — 있으면)
  // 7. 현재 매칭 메트릭 (S3 ER 결과에서 계산 — 있으면)
  // 8. 미매칭 패턴 분석 (S3 ER 결과에서 분석 — 있으면)
  // 9. 출력 형식 지정 (JSON)
}
```

### 모델 설정
```typescript
const MODEL_ID = process.env.BEDROCK_MODEL_ID ?? 'apac.anthropic.claude-sonnet-4-20250514-v1:0';
// AP 리전은 apac. 접두사 필수!

const invokeParams = {
  modelId: MODEL_ID,
  contentType: 'application/json',
  accept: 'application/json',
  body: JSON.stringify({
    anthropic_version: 'bedrock-2023-05-31',
    max_tokens: 8192,  // 규칙 생성은 긴 출력 필요
    messages: [{ role: 'user', content: prompt }],
  }),
};
```

### 커스텀 프롬프트 지원
사용자가 UI에서 프롬프트 템플릿을 직접 수정할 수 있도록:
```typescript
// 사용자 커스텀 프롬프트가 있으면 그것을 기본으로 하고,
// 데이터 컨텍스트는 자동으로 하단에 추가
if (customPromptTemplate) {
  return `${customPromptTemplate}

## 아래는 실제 데이터 기반으로 자동 생성된 컨텍스트입니다
${schemaSection}
${currentRulesSection}
${dataQualitySection}
${sampleSection}
${feedbackSection}
${metricsSection}
${unmatchedSection}`;
}
```
