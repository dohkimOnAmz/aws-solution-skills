# Entity Resolution — Matching Strategies

## 개요

Entity Resolution은 3가지 매칭 타입을 제공하지만, 핵심 워크플로우는
**"사용자가 직접 규칙을 선택"하는 것이 아니라**:

```

## Bedrock 기반 초기 규칙 자동 생성 (핵심 구현)

위 "AI 규칙 개선"이 기존 규칙을 개선하는 것이라면,
이 섹션은 **데이터만 보고 처음부터 규칙을 만들어주는** 구현이다.

### Step 1: 샘플 데이터 분석 프롬프트

```typescript
// backend/lambdas/ai-agent/rule-generator.ts

interface DataProfile {
  totalRecords: number;
  fields: Array<{
    name: string;
    nullRate: number;        // 0~1: 비어있는 비율
    uniqueRate: number;      // 0~1: 고유값 비율 (1에 가까울수록 식별력 높음)
    variationPatterns: string[];  // 발견된 변형 패턴 예시
    sampleValues: string[];  // 대표 값 5~10개
  }>;
  channelDistribution: Record<string, number>;  // 채널별 레코드 수
}

async function analyzeDataAndGenerateRules(dataProfile: DataProfile): Promise<GeneratedRules> {
  const MODEL_ID = process.env.BEDROCK_MODEL_ID!;

  const prompt = `당신은 AWS Entity Resolution 매칭 규칙 전문가입니다.

아래 고객 데이터 프로파일을 분석하고, 최적의 Entity Resolution 퍼지 매칭 규칙 셋을 생성해주세요.

## 데이터 프로파일
- 전체 레코드 수: ${dataProfile.totalRecords}
- 채널 분포: ${JSON.stringify(dataProfile.channelDistribution)}

## 필드별 품질 분석
${dataProfile.fields.map(f => `
### ${f.name}
- Null 비율: ${(f.nullRate * 100).toFixed(1)}%
- 고유값 비율: ${(f.uniqueRate * 100).toFixed(1)}%
- 변형 패턴: ${f.variationPatterns.join(', ') || '없음'}
- 샘플 값: ${f.sampleValues.join(' | ')}
`).join('\n')}

## 규칙 생성 요구사항
1. ER Advanced Rule 형식의 규칙 셋을 JSON으로 생성
2. 각 규칙은 matchingKeys 배열을 포함 (필드명은 위 필드명과 동일)
3. 규칙 간 관계는 OR (하나라도 매칭되면 동일 고객)
4. 규칙 내 matchingKeys는 AND (모든 키가 매칭되어야 함)
5. 각 규칙에 대해:
   - 왜 이 조합을 선택했는지 근거
   - 예상 precision (정밀도)
   - 예상 recall (재현율)
   - 주의사항 (false positive 위험 등)

## 출력 형식 (JSON)
\`\`\`json
{
  "rules": [
    {
      "ruleName": "RuleName",
      "matchingKeys": ["field1", "field2"],
      "rationale": "선택 근거",
      "estimatedPrecision": 0.95,
      "estimatedRecall": 0.8,
      "warnings": ["주의사항"]
    }
  ],
  "overallStrategy": "전체 전략 설명",
  "recommendedOrder": ["가장 정밀한 규칙부터 순서"],
  "dataQualityNotes": "데이터 품질 관련 특이사항"
}
\`\`\``;

  const response = await bedrockClient.send(new InvokeModelCommand({
    modelId: MODEL_ID,
    contentType: 'application/json',
    accept: 'application/json',
    body: JSON.stringify({
      anthropic_version: 'bedrock-2023-05-31',
      max_tokens: 4096,
      messages: [{ role: 'user', content: prompt }],
    }),
  }));

  const result = JSON.parse(new TextDecoder().decode(response.body));
  return parseGeneratedRules(result.content[0].text);
}
```

### Step 2: 데이터 프로파일 생성 (샘플 데이터에서)

```typescript
// backend/lambdas/ai-agent/data-profiler.ts

async function profileData(s3Path: string): Promise<DataProfile> {
  // 1. S3에서 샘플 데이터 로드 (최대 1000 레코드)
  const records = await loadSampleRecords(s3Path, 1000);

  // 2. 필드별 분석
  const fields = Object.keys(records[0]).map(fieldName => {
    const values = records.map(r => r[fieldName]).filter(Boolean);
    const nullCount = records.length - values.length;
    const uniqueValues = new Set(values);

    // 변형 패턴 감지
    const patterns = detectVariationPatterns(fieldName, values);

    return {
      name: fieldName,
      nullRate: nullCount / records.length,
      uniqueRate: uniqueValues.size / values.length,
      variationPatterns: patterns,
      sampleValues: values.slice(0, 10),
    };
  });

  // 3. 채널 분포
  const channelDist: Record<string, number> = {};
  records.forEach(r => {
    const ch = r.sourcechannel || 'UNKNOWN';
    channelDist[ch] = (channelDist[ch] || 0) + 1;
  });

  return { totalRecords: records.length, fields, channelDistribution: channelDist };
}

function detectVariationPatterns(fieldName: string, values: string[]): string[] {
  const patterns: string[] = [];
  if (fieldName.includes('name')) {
    const hasKorean = values.some(v => /[\uAC00-\uD7AF]/.test(v));
    const hasEnglish = values.some(v => /^[A-Za-z\s-]+$/.test(v));
    if (hasKorean && hasEnglish) patterns.push('한영 혼용');
    if (values.some(v => v.includes('-'))) patterns.push('하이픈 포함');
    if (values.some(v => v !== v.toUpperCase() && v !== v.toLowerCase())) patterns.push('대소문자 혼용');
  }
  if (fieldName.includes('phone')) {
    if (values.some(v => v.startsWith('+82'))) patterns.push('국가코드 포함');
    if (values.some(v => v.includes('-'))) patterns.push('하이픈 포함');
  }
  if (fieldName.includes('email')) {
    if (values.some(v => /guest-.*@booking/i.test(v))) patterns.push('릴레이 이메일');
  }
  return patterns;
}
```

### Step 3: HITL 검증 API

```typescript
// POST /api/ai/generate-rules — 초기 규칙 생성
async function generateInitialRules(body: { dataPath: string }) {
  // 1. 데이터 프로파일링
  const profile = await profileData(body.dataPath);

  // 2. Bedrock으로 규칙 생성
  const generated = await analyzeDataAndGenerateRules(profile);

  // 3. DynamoDB에 PENDING 상태로 저장
  const suggestionId = crypto.randomUUID();
  await saveSuggestion(suggestionId, { ...generated, status: 'PENDING_REVIEW', dataProfile: profile });

  return { suggestionId, rules: generated.rules, dataProfile: profile };
}

// POST /api/ai/test-rules — 생성된 규칙으로 테스트 매칭 실행
async function testRules(body: { suggestionId: string }) {
  const suggestion = await getSuggestion(body.suggestionId);
  // 1. 테스트용 ER Workflow 생성 (임시)
  // 2. 소규모 데이터로 ER Job 실행
  // 3. 매칭 결과 파싱 → 샘플 매칭 쌍 추출
  // 4. 결과를 suggestion에 첨부
  return { matchedPairs: [], stats: { totalMatched: 0, groups: 0 } };
}

// POST /api/ai/review-rules — HITL 피드백 제출
async function reviewRules(body: {
  suggestionId: string;
  action: 'APPROVE' | 'MODIFY' | 'REJECT';
  feedback?: string;
  modifications?: Array<{ ruleName: string; matchingKeys: string[] }>;
}) {
  if (body.action === 'APPROVE') {
    // → 프로덕션 ER Workflow에 규칙 적용
    await applyRulesToWorkflow(body.suggestionId);
    await updateSuggestionStatus(body.suggestionId, 'APPROVED');
  } else if (body.action === 'MODIFY') {
    // → 수정된 규칙으로 Bedrock에 재분석 요청 (개선 루프)
    const improved = await requestImprovement(body.suggestionId, body.feedback!, body.modifications);
    return { newSuggestionId: improved.id, rules: improved.rules };
  } else {
    await updateSuggestionStatus(body.suggestionId, 'REJECTED');
  }
}
```

### 프론트엔드 HITL 플로우

```
[AI Rules 페이지]
  ├── "규칙 생성" 버튼 → POST /api/ai/generate-rules
  │     └── 데이터 프로파일 + 생성된 규칙 표시
  │
  ├── "테스트 실행" 버튼 → POST /api/ai/test-rules
  │     └── 매칭된 쌍 목록 (사용자가 맞는지/틀린지 표시)
  │
  ├── "승인" 버튼 → POST /api/ai/review-rules (APPROVE)
  │     └── ER Workflow에 적용 완료
  │
  ├── "수정 요청" → 피드백 입력 → POST /api/ai/review-rules (MODIFY)
  │     └── Bedrock이 피드백 반영하여 규칙 재생성 (루프)
  │
  └── "거절" → POST /api/ai/review-rules (REJECT)
```
```

즉, AI가 데이터 품질과 패턴을 보고 최적의 매칭 규칙을 제안하고,
사람이 검증/수정한 후 적용하는 **AI-assisted Rule Generation + HITL** 패턴이다.

## 핵심 워크플로우: AI Rule Generation + HITL

```
┌─────────────────────────────────────────────────────────────┐
│                                                               │
│  ① 샘플 데이터 업로드 (CSV/Parquet/DB에서 추출)             │
│       ↓                                                      │
│  ② Bedrock 분석                                             │
│     - PII 필드별 품질 분석 (null 비율, 변형 패턴)           │
│     - 필드 간 상관관계 분석 (email+name 조합 유일성)         │
│     - 최적 match key 조합 추천                               │
│       ↓                                                      │
│  ③ 퍼지 규칙 자동 생성                                      │
│     - ER Advanced Rule 형태로 규칙 셋 생성                   │
│     - 각 규칙의 예상 precision/recall 추정                    │
│       ↓                                                      │
│  ④ HITL 검증 (프론트엔드 UI)                                │
│     - 생성된 규칙 목록 표시                                   │
│     - 샘플 매칭 쌍 미리보기 ("이 두 레코드가 매칭됩니다")    │
│     - 사용자: 승인 / 수정 / 거절                            │
│       ↓                                                      │
│  ⑤ 테스트 매칭 실행 (소규모 데이터셋)                       │
│     - 실제 ER Job 실행 (테스트 모드)                         │
│     - 결과: matched pairs + confidence                       │
│       ↓                                                      │
│  ⑥ 정확도 리뷰                                              │
│     - False Positive/Negative 샘플 표시                       │
│     - 사용자 피드백 수집                                      │
│       ↓                                                      │
│  ⑦ 규칙 개선 루프 (반복)                                    │
│     - 피드백 기반으로 Bedrock이 규칙 개선 제안               │
│     - HITL 재검증 → 만족할 때까지 반복                       │
│       ↓                                                      │
│  ⑧ 최종 적용 (프로덕션 ER Workflow에 반영)                  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

## ER 매칭 타입 (참고)

위 워크플로우에서 Bedrock이 생성하는 규칙의 대상 타입:
1. **Simple Rule** — 정확 일치 (고유 식별자가 확실할 때)
2. **Advanced Rule** — 퍼지 매칭 (대부분의 경우 이것을 생성)
3. **ML Matching** — 규칙으로 커버 불가할 때 보조적 사용

## Simple Rule Matching

### 용도
- 고유 식별자가 있을 때 (회원번호, 주민번호, 사번)
- 정확 일치만으로 충분한 경우

### 패턴
```typescript
// ER Workflow 생성 (Lambda handler)
const createWorkflowParams = {
  workflowName: `${projectName}-simple`,
  roleArn: ER_ROLE_ARN,
  inputSourceConfig: [{
    inputSourceARN: `arn:aws:glue:${region}:${accountId}:table/${glueDbName}/${glueTableName}`,
    schemaName: `${projectName}-schema`,
  }],
  resolutionTechniques: {
    resolutionType: 'RULE_MATCHING',
    ruleBasedProperties: {
      attributeMatchingModel: 'ONE_TO_ONE',
      rules: [{
        ruleName: 'LoyaltyNumberMatch',
        matchingKeys: ['loyaltynumber'],
      }],
    },
  },
  outputSourceConfig: [{
    outputS3Path: `s3://${dataBucket}/er-output/simple/`,
    output: [
      { name: 'variantid', hashed: false },
      { name: 'firstname', hashed: false },
      { name: 'lastname', hashed: false },
      { name: 'email', hashed: false },
    ],
  }],
};
```

## Advanced Rule Matching

### 용도 (핵심 차이: 퍼지 비교 함수 사용)
- 이름 유사도(Cosine), 전화번호 편집거리(Levenshtein), 발음 매칭(Soundex) 등
- 한영 혼용, 오타, 형식 차이를 커버
- **Simple Rule과의 차이**: Simple은 정확 일치(Exact)만 가능, Advanced는 퍼지 함수 사용

### ⚠️ 중요: API 형식 차이

| 타입 | API 속성 | 규칙 형식 |
|------|---------|-----------|
| Simple | `ruleBasedProperties` | `matchingKeys: ['field1', 'field2']` (정확 일치만) |
| **Advanced** | **`ruleConditionProperties`** | **`conditionString: 'Cosine(Name, 0.7) AND Exact(Email)'`** |

**Simple과 Advanced는 서로 다른 API 속성을 사용한다!** `ruleBasedProperties`에는 퍼지 함수를 쓸 수 없다.

### 사용 가능한 비교 함수 (conditionString)

| 함수 | 용도 | 예시 |
|------|------|------|
| `Exact(matchKey)` | 정확 일치 | `Exact(Email)` |
| `Cosine(matchKey, threshold)` | 코사인 유사도 (0.0~1.0) | `Cosine(FirstName, 0.7)` |
| `Levenshtein(matchKey, threshold)` | 편집 거리 (정수) | `Levenshtein(Phone, 3)` |
| `Soundex(matchKey)` | 발음 유사 (영문) | `Soundex(LastName)` |
| `DateDifference(matchKey, days)` | 날짜 차이 범위 | `DateDifference(DOB, 30)` |

### 조건 문법
```
Exact(Email) AND Cosine(FirstName, 0.7) AND Cosine(LastName, 0.7)
(Exact(Phone) OR Levenshtein(Phone, 3)) AND Soundex(FirstName)
Cosine(FirstName, 0.6) AND Cosine(LastName, 0.6) AND Exact(DateOfBirth)
```

### 규칙 조합 전략 (퍼지)

| 우선순위 | 규칙 | 조건식 | 기대 정밀도 |
|---------|------|--------|------------|
| 1 | EmailAndNameFuzzy | `Exact(Email) AND Cosine(FirstName, 0.7) AND Cosine(LastName, 0.7)` | 95%+ |
| 2 | PhoneAndNameSoundex | `Exact(Phone) AND Soundex(FirstName) AND Soundex(LastName)` | 90%+ |
| 3 | NameFuzzyAndDOB | `Cosine(FirstName, 0.7) AND Cosine(LastName, 0.7) AND Exact(DateOfBirth)` | 85%+ |
| 4 | PhoneAndEmail | `Exact(Phone) AND Exact(Email)` | 95%+ |

### 패턴 (CDK/Lambda에서 Workflow 생성)
```typescript
const advancedWorkflowParams = {
  workflowName: `${projectName}-advanced`,
  roleArn: ER_ROLE_ARN,
  inputSourceConfig: [/* same as above */],
  resolutionTechniques: {
    resolutionType: 'RULE_MATCHING',
    // ⚠️ Advanced는 ruleConditionProperties를 사용! (ruleBasedProperties 아님)
    ruleConditionProperties: {
      matchPurpose: 'IDENTIFIER_GENERATION',
      rules: [
        {
          ruleName: 'EmailAndNameFuzzy',
          conditionString: 'Exact(Email) AND Cosine(FirstName, 0.7) AND Cosine(LastName, 0.7)',
        },
        {
          ruleName: 'PhoneAndNameSoundex',
          conditionString: 'Exact(Phone) AND Soundex(FirstName) AND Soundex(LastName)',
        },
        {
          ruleName: 'NameFuzzyAndDOB',
          conditionString: 'Cosine(FirstName, 0.7) AND Cosine(LastName, 0.7) AND Exact(DateOfBirth)',
        },
      ],
    },
  },
  outputSourceConfig: [/* ... */],
};
```

### 스키마 주의사항 (Advanced 퍼지 사용 시)

`ruleConditionProperties`에서 conditionString은 **매치키 이름**을 참조한다.
NAME 타입 필드는 `groupName`을 쓸 수 없으므로, FirstName/LastName을 분리된 매치키로 등록해야 한다:

```typescript
// 스키마 매핑에서:
schemaInputAttributes: [
  { fieldName: 'firstname', type: 'NAME_FIRST', matchKey: 'FirstName' },   // 별도 매치키
  { fieldName: 'lastname', type: 'NAME_LAST', matchKey: 'LastName' },      // 별도 매치키
  { fieldName: 'email', type: 'EMAIL_ADDRESS', matchKey: 'Email' },
  { fieldName: 'phone', type: 'PHONE_NUMBER', matchKey: 'Phone' },
  { fieldName: 'dateofbirth', type: 'DATE', matchKey: 'DateOfBirth' },
  // ...
]
// ❌ 틀림: groupName: 'FullName' (NAME 타입은 group 불가)
// ✅ 맞음: 각각 별도 matchKey
```

### 한국어 이름 처리 전략

한국어 환경에서는 이름 변형이 많음:
- 김민호 / KIM MINHO / Kim Min-Ho / MINHO KIM
- 010-1234-5678 / +82 10-1234-5678 / 01012345678

**권장 전처리**:
```typescript
function normalizeKoreanName(input: string): { firstName: string; lastName: string } {
  // 1. Remove spaces, hyphens
  // 2. Detect Korean vs English
  // 3. Korean: 첫 글자 = 성, 나머지 = 이름
  // 4. English: Last token = 성 (한국식 영문 순서 고려)
  // 5. Uppercase
}

function normalizePhone(input: string): string {
  // 1. Remove all non-digits
  // 2. Remove country code (+82 → 0)
  // 3. Standard format: 01012345678
}
```

## ML Matching

### 용도
- 데이터 소스가 5개 이상
- PII 품질이 소스마다 크게 다름
- Rule로는 놓치는 매치가 많을 때

### 리전 제약
⚠️ **ML Matching은 제한된 리전에서만 사용 가능**
- 반드시 AWS Knowledge MCP로 확인:
  `aws_get_regional_availability("entityresolution", "MLMatchingWorkflow")`

### 패턴
```typescript
const mlWorkflowParams = {
  workflowName: `${projectName}-ml`,
  roleArn: ER_ROLE_ARN,
  inputSourceConfig: [/* ... */],
  resolutionTechniques: {
    resolutionType: 'ML_MATCHING',
    // ML은 추가 설정 불필요 — 자동으로 학습
  },
  outputSourceConfig: [/* ... */],
};
```

### ML 매칭 비용 참고
- Rule: $0.25 / 1,000 records
- ML: $1.00 / 1,000 records (4배)
- 10만 레코드 기준: Rule $25 vs ML $100

## 매칭 결과 처리

ER 실행 후 결과 처리 플로우:
```
ER Job 완료 → S3 output (JSON Lines) → Lambda 파싱 → DynamoDB 저장
                                                    → CP Import (Golden Record)
```

### 결과 파싱 패턴
```typescript
interface ErOutputRecord {
  matchId: string;      // 동일 고객 그룹 ID
  variantId: string;    // 원본 레코드 ID
  confidenceScore?: number; // ML일 때만
}

// Golden Record 선정: matchId 그룹 내에서 가장 완전한 레코드 선택
function selectGoldenRecord(variants: ErOutputRecord[]): GoldenRecord {
  // 1. 필드 채워짐 비율 높은 것 우선
  // 2. 최신 데이터 우선
  // 3. 신뢰 채널 우선 (CRM > Web > OTA)
}
```

## AI 규칙 개선 (Human-in-the-Loop)

Bedrock Claude가 매칭 결과를 분석하고 규칙 개선 제안:

```
[현재 규칙] + [매칭 실패 사례] → Claude 분석 → [개선 제안]
                                                    ↓
                                            관리자 검토 (UI)
                                              ↙        ↘
                                         승인 → 적용     거절 → 기각
```

```typescript
// AI Agent Lambda에서 Bedrock 호출
const prompt = `
현재 Entity Resolution 규칙:
${JSON.stringify(currentRules)}

매칭 실패 사례 (같은 고객인데 매칭 안 됨):
${JSON.stringify(falsNegatives)}

잘못된 매칭 사례 (다른 고객인데 매칭 됨):
${JSON.stringify(falsePositives)}

위 사례를 분석하고 규칙 개선안을 제시해주세요.
`;
```
