# Lambda Handler Patterns

재사용 가능한 Lambda 핸들러 패턴. 도메인별로 필드/로직만 바꿔서 생성.

## 공통 구조

모든 핸들러의 기본 뼈대:

```typescript
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';

const HEADERS = {
  'Content-Type': 'application/json',
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'Content-Type,Authorization',
  'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
};

function ok(body: unknown): APIGatewayProxyResult {
  return { statusCode: 200, headers: HEADERS, body: JSON.stringify(body) };
}

function error(statusCode: number, message: string): APIGatewayProxyResult {
  return { statusCode, headers: HEADERS, body: JSON.stringify({ error: message }) };
}

export async function handler(event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> {
  try {
    const path = event.path;
    const method = event.httpMethod;

    if (method === 'OPTIONS') return ok({});

    // Route to sub-handlers
    // ...

    return error(404, 'Not found');
  } catch (err: any) {
    console.error('Handler error:', err);
    return error(500, err.message || 'Internal server error');
  }
}
```

## Matching Handler

Entity Resolution 워크플로우 실행 + 결과 조회

```typescript
import { EntityResolutionClient, StartMatchingJobCommand, GetMatchingJobCommand, ListMatchingJobsCommand } from '@aws-sdk/client-entityresolution';
import { DynamoDBDocumentClient, PutCommand, QueryCommand } from '@aws-sdk/lib-dynamodb';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';

const erClient = new EntityResolutionClient({});
const ddbClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const s3Client = new S3Client({});

const WORKFLOW_NAME = process.env.WORKFLOW_NAME!;
const DATA_BUCKET = process.env.DATA_BUCKET!;
const RESULTS_TABLE = process.env.RESULTS_TABLE!;
const GLUE_DB_NAME = process.env.GLUE_DB_NAME!;
const GLUE_TABLE_NAME = process.env.GLUE_TABLE_NAME!;

// POST /api/matching/run — 매칭 실행
async function runMatching(body: { matchingType: 'simple' | 'advanced' | 'ml' }) {
  const workflowName = `${WORKFLOW_NAME}-${body.matchingType}`;

  // 1. ER Job 시작
  const { jobId } = await erClient.send(new StartMatchingJobCommand({
    workflowName,
  }));

  // 2. Polling (실제로는 Step Functions 또는 EventBridge 권장)
  let status = 'RUNNING';
  while (status === 'RUNNING') {
    await new Promise(r => setTimeout(r, 5000));
    const job = await erClient.send(new GetMatchingJobCommand({
      workflowName,
      jobId: jobId!,
    }));
    status = job.status || 'UNKNOWN';
  }

  // 3. 결과 파싱 (S3 output)
  const results = await parseErOutput(workflowName, jobId!);

  // 4. DynamoDB에 저장
  for (const result of results) {
    await ddbClient.send(new PutCommand({
      TableName: RESULTS_TABLE,
      Item: {
        pk: `MATCH#${body.matchingType}`,
        sk: `${result.matchId}#${result.variantId}`,
        ...result,
        timestamp: new Date().toISOString(),
      },
    }));
  }

  return { jobId, matchCount: results.length, matchingType: body.matchingType };
}

// GET /api/matching/results — 결과 조회
async function getResults(matchingType: string) {
  const { Items } = await ddbClient.send(new QueryCommand({
    TableName: RESULTS_TABLE,
    KeyConditionExpression: 'pk = :pk',
    ExpressionAttributeValues: { ':pk': `MATCH#${matchingType}` },
  }));
  return Items || [];
}

// S3 ER 출력 파싱
async function parseErOutput(workflowName: string, jobId: string) {
  // ER output format: s3://{bucket}/er-output/{workflowName}/{jobId}/
  // JSON Lines: { matchId, variantId, confidenceScore? }
  // ... S3 list + get + parse
  return [];
}
```

## Ingestion Handler (Multi-Mode)

CSV, Parquet 업로드 + Glue Crawler 트리거

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { GlueClient, StartCrawlerCommand, GetCrawlerCommand } from '@aws-sdk/client-glue';
import { parse } from 'csv-parse/sync';

const s3Client = new S3Client({});
const glueClient = new GlueClient({});
const DATA_BUCKET = process.env.DATA_BUCKET!;
const GLUE_DB_NAME = process.env.GLUE_DB_NAME!;
const CRAWLER_NAME = process.env.CRAWLER_NAME; // optional (glue_connection mode)

// POST /api/ingestion/upload-csv — CSV 업로드 + 검증
async function uploadCsv(body: { fileName: string; content: string; channel: string }) {
  const records = parse(body.content, { columns: true, skip_empty_lines: true });

  // Validate required fields
  for (const record of records) {
    if (!record.variantid) return error(400, 'variantid is required');
    if (!record.sourcechannel) record.sourcechannel = body.channel;
  }

  // S3에 CSV로 저장
  const key = `er-input/${body.channel}/${body.fileName}`;
  const csvContent = [
    Object.keys(records[0]).join(','),
    ...records.map((r: any) => Object.values(r).map(v => `"${v}"`).join(',')),
  ].join('\n');

  await s3Client.send(new PutObjectCommand({
    Bucket: DATA_BUCKET, Key: key, Body: csvContent, ContentType: 'text/csv',
  }));

  return { recordCount: records.length, format: 'csv', path: key };
}

// POST /api/ingestion/upload-parquet — Parquet 업로드 (바이너리)
async function uploadParquet(body: { fileName: string; channel: string }, fileBuffer: Buffer) {
  // Parquet은 스키마가 내장되어 있으므로 별도 파싱 불필요
  // Glue Table이 Parquet SerDe로 설정되어 있으면 바로 참조 가능
  const key = `er-input/${body.channel}/${body.fileName}`;

  await s3Client.send(new PutObjectCommand({
    Bucket: DATA_BUCKET,
    Key: key,
    Body: fileBuffer,
    ContentType: 'application/x-parquet',
  }));

  // Parquet 업로드 후 Glue Table 파티션 갱신 (선택)
  // → MSCK REPAIR TABLE 또는 Crawler 재실행

  return { format: 'parquet', path: key, message: 'Schema auto-detected from Parquet metadata' };
}

// POST /api/ingestion/crawl — Glue Crawler 실행 (DB 소스에서 가져오기)
async function triggerCrawler() {
  if (!CRAWLER_NAME) return error(400, 'Glue Crawler not configured (ingestion mode != glue_connection)');

  await glueClient.send(new StartCrawlerCommand({ Name: CRAWLER_NAME }));

  // Polling (또는 EventBridge로 완료 이벤트 수신)
  let status = 'RUNNING';
  while (status === 'RUNNING') {
    await new Promise(r => setTimeout(r, 10000));
    const { Crawler } = await glueClient.send(new GetCrawlerCommand({ Name: CRAWLER_NAME }));
    status = Crawler?.State || 'UNKNOWN';
    if (status === 'READY') break; // 완료
  }

  return { crawlerName: CRAWLER_NAME, status: 'COMPLETED' };
}

// GET /api/ingestion/status — 현재 데이터 상태
async function getIngestionStatus() {
  // S3 파일 목록 (er-input/ prefix)
  // Glue Table 파티션 정보
  // 마지막 Crawler 실행 시간
  return {
    totalFiles: 0,
    totalRecords: 0,
    lastCrawlTime: null,
    formats: ['csv', 'parquet'], // 현재 존재하는 포맷
  };
}
```

## Accuracy Handler

매칭 정확도 평가 (Precision / Recall / F1)

```typescript
const ACCURACY_TABLE = process.env.ACCURACY_TABLE!;

interface AccuracyMetrics {
  matchingType: string;
  totalPairs: number;
  truePositives: number;
  falsePositives: number;
  falseNegatives: number;
  precision: number;
  recall: number;
  f1Score: number;
}

// POST /api/accuracy/evaluate — 정확도 계산
async function evaluateAccuracy(body: {
  matchingType: string;
  samplePairs: Array<{ id1: string; id2: string; expectedMatch: boolean }>;
}) {
  const results = await getResults(body.matchingType);
  const matchedPairs = buildMatchedPairSet(results);

  let tp = 0, fp = 0, fn = 0;
  for (const pair of body.samplePairs) {
    const actuallyMatched = matchedPairs.has(`${pair.id1}|${pair.id2}`);
    if (pair.expectedMatch && actuallyMatched) tp++;
    else if (!pair.expectedMatch && actuallyMatched) fp++;
    else if (pair.expectedMatch && !actuallyMatched) fn++;
  }

  const precision = tp / (tp + fp) || 0;
  const recall = tp / (tp + fn) || 0;
  const f1Score = 2 * (precision * recall) / (precision + recall) || 0;

  const metrics: AccuracyMetrics = {
    matchingType: body.matchingType,
    totalPairs: body.samplePairs.length,
    truePositives: tp, falsePositives: fp, falseNegatives: fn,
    precision, recall, f1Score,
  };

  await ddbClient.send(new PutCommand({
    TableName: ACCURACY_TABLE,
    Item: { pk: 'ACCURACY', sk: `${body.matchingType}#${Date.now()}`, ...metrics },
  }));

  return metrics;
}
```

## AI Agent Handler (Bedrock)

AI 규칙 개선 제안 생성

```typescript
import { BedrockRuntimeClient, InvokeModelCommand } from '@aws-sdk/client-bedrock-runtime';

const bedrockClient = new BedrockRuntimeClient({});
const MODEL_ID = process.env.BEDROCK_MODEL_ID!; // e.g. apac.anthropic.claude-sonnet-4-20250514-v1:0
const SUGGESTIONS_TABLE = process.env.SUGGESTIONS_TABLE!;

// POST /api/ai/suggest — AI 규칙 개선 요청
async function suggestRuleImprovement(body: {
  currentRules: Array<{ ruleName: string; matchingKeys: string[] }>;
  falseNegatives: Array<{ pair: [any, any]; reason: string }>;
  falsePositives: Array<{ pair: [any, any]; reason: string }>;
}) {
  const prompt = buildPrompt(body);

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
  const suggestion = parseSuggestion(result.content[0].text);

  // 저장 (pending 상태)
  const suggestionId = crypto.randomUUID();
  await ddbClient.send(new PutCommand({
    TableName: SUGGESTIONS_TABLE,
    Item: {
      pk: 'SUGGESTION',
      sk: suggestionId,
      status: 'PENDING', // PENDING → APPROVED / REJECTED
      suggestion,
      createdAt: new Date().toISOString(),
    },
  }));

  return { suggestionId, suggestion };
}

// POST /api/ai/approve — 제안 승인
async function approveSuggestion(suggestionId: string) {
  // 1. Update status → APPROVED
  // 2. Apply rule changes to ER workflow
  // 3. Record in rule change history
}

// POST /api/ai/reject — 제안 거절
async function rejectSuggestion(suggestionId: string, reason: string) {
  // 1. Update status → REJECTED
  // 2. Record reason in history
}

function buildPrompt(body: any): string {
  return `당신은 Entity Resolution 매칭 규칙 전문가입니다.

## 현재 규칙
${JSON.stringify(body.currentRules, null, 2)}

## 매칭 실패 사례 (False Negatives - 같은 고객인데 매칭 안 됨)
${JSON.stringify(body.falseNegatives, null, 2)}

## 잘못된 매칭 (False Positives - 다른 고객인데 매칭 됨)
${JSON.stringify(body.falsePositives, null, 2)}

위 사례를 분석하고 다음 형식으로 규칙 개선안을 제시하세요:

1. 문제 분석 (왜 실패/오매칭 발생했는지)
2. 제안 규칙 (JSON 형식으로)
3. 예상 효과 (precision/recall 변화 예측)
4. 주의사항`;
}
```

## Profiles Handler

Customer Profiles 조회/관리

```typescript
import { CustomerProfilesClient, SearchProfilesCommand, GetProfileCommand, ListProfileObjectsCommand } from '@aws-sdk/client-customer-profiles';

const cpClient = new CustomerProfilesClient({});
const DOMAIN_NAME = process.env.CP_DOMAIN_NAME!;

// GET /api/profiles/search?key=email&value=test@example.com
async function searchProfiles(keyName: string, values: string[]) {
  const { Items } = await cpClient.send(new SearchProfilesCommand({
    DomainName: DOMAIN_NAME,
    KeyName: `_${keyName}`, // CP uses underscore prefix for key search
    Values: values,
  }));
  return Items || [];
}

// GET /api/profiles/:profileId
async function getProfile(profileId: string) {
  const profile = await cpClient.send(new GetProfileCommand({
    DomainName: DOMAIN_NAME,
    ProfileId: profileId,
  }));
  return profile;
}

// GET /api/profiles/:profileId/objects?objectType=Booking
async function getProfileObjects(profileId: string, objectTypeName: string) {
  const { Items } = await cpClient.send(new ListProfileObjectsCommand({
    DomainName: DOMAIN_NAME,
    ProfileId: profileId,
    ObjectTypeName: objectTypeName,
  }));
  return Items || [];
}
```

## ⭐ CP Object Type 정의 — 정확한 Key/Field 매핑 (Critical)

**가장 자주 틀리는 부분.** AWS Customer Profiles의 Object Type 정의는 매우 엄격하다. 잘못 정의하면 PutProfileObject가 200 OK를 반환해도 child instance가 profile에 attach되지 않아, ListProfileObjects/Calculated Attributes가 모두 비어있게 된다. 디버깅이 어렵다.

### 절대 규칙 (AWS 공식 문서 기반)

1. **Target은 `_profile`만 지원한다.** `_hotelReservation` / `_loyalty` / `_hotelStayRevenue` 등은 **TemplateId**이지 Target namespace가 아니다.
   > "The format of this field is always a JSON accessor. The only supported target object is `_profile`." — [AWS docs](https://docs.aws.amazon.com/connect/latest/adminguide/object-type-mapping-definition-details.html)

2. **Target은 optional이다.** 자식 instance object(Reservation, Folio 등)는 **Target을 omit**한다. Target을 주면 standard profile field를 덮어쓰게 되어 instance가 안 생긴다.

3. **`_profileId` 키를 정의하지 마라.** `_profileId`는 CP 시스템 예약 키 — 우리가 넣은 값이 아니라 CP가 자동 생성한 UUID로 채워진다. 그래서 자식 object가 `GuestProfileId=golden-...`로 들어와도 `_profileId=golden-...`인 profile은 존재하지 않아 매칭이 실패하고 inferred profile이 폭증한다.

4. **Cross-object 링크는 같은 이름의 Key + 같은 값**으로만 일어난다. CP key namespace는 도메인 전역.

### 정확한 패턴

```yaml
# config/schema.yaml
object_types:
  - name: GuestProfile           # 부모 (golden record)
    keys:
      # 모든 자식 ObjectType이 같은 이름·같은 값으로 이 profile에 attach될 link key
      - name: GuestKey
        fields: [GuestProfileId]
        standard_identifiers: [PROFILE, UNIQUE]   # ← PROFILE+UNIQUE
      - name: EmailKey
        fields: [EmailAddress]
        standard_identifiers: [LOOKUP_ONLY]       # 검색용
      - name: PhoneKey
        fields: [PhoneNumber]
        standard_identifiers: [LOOKUP_ONLY]
    fields:
      - { name: GuestProfileId, type: STRING, required: true }
      - { name: FirstName, type: STRING }
      - { name: LastName, type: STRING }
      - { name: EmailAddress, type: STRING }
      - { name: PhoneNumber, type: STRING }
      - { name: BirthDate, type: STRING }
      - { name: Address1, type: STRING }
      - { name: City, type: STRING }
      - { name: State, type: STRING }
      - { name: PostalCode, type: STRING }
      - { name: Country, type: STRING }
      # 그 외 custom 메타: SourceChannels, MatchId, ImportedAt, ...

  - name: Reservation             # 자식 (instance object)
    keys:
      - name: ReservationKey
        fields: [ReservationId]
        standard_identifiers: [UNIQUE]            # 같은 ReservationId면 upsert
      - name: GuestKey                            # 부모와 같은 이름·같은 필드값
        fields: [GuestProfileId]
        standard_identifiers: [PROFILE]           # PROFILE만 (UNIQUE 빼라)
    fields:
      # required + 비즈니스 필드들
      - { name: ReservationId, type: STRING, required: true }
      - { name: GuestProfileId, type: STRING, required: true }
      - { name: TotalAmount, type: NUMBER }
      - { name: NumberOfNights, type: NUMBER }
      # ... CalculatedAttribute가 참조할 필드 모두 선언
```

### Custom Resource (ObjectType upsert) — 정확한 매핑 핸들러

```typescript
// backend/custom-resources/upsert-object-type/handler.ts
import { CustomerProfilesClient, PutProfileObjectTypeCommand, DeleteProfileObjectTypeCommand } from '@aws-sdk/client-customer-profiles';
import type { CdkCustomResourceEvent, CdkCustomResourceResponse } from 'aws-lambda';

const cp = new CustomerProfilesClient({});
const DOMAIN = process.env.CP_DOMAIN_NAME!;

export async function handler(event: CdkCustomResourceEvent): Promise<CdkCustomResourceResponse> {
  const props = event.ResourceProperties as any;
  const objectTypeName = props.ObjectTypeName as string;

  if (event.RequestType === 'Delete') {
    await cp.send(new DeleteProfileObjectTypeCommand({ DomainName: DOMAIN, ObjectTypeName: objectTypeName }))
      .catch(e => { if (e.name !== 'ResourceNotFoundException') console.error(e); });
    return { PhysicalResourceId: `objtype-${objectTypeName}` };
  }

  // Keys: pass through schema.yaml verbatim. DO NOT auto-add LOOKUP_ONLY.
  const keys: Record<string, any[]> = {};
  for (const k of props.Keys as any[]) {
    keys[k.name] = [{
      StandardIdentifiers: k.standard_identifiers ?? k.StandardIdentifiers ?? [],
      FieldNames: k.fields ?? k.FieldNames,
    }];
  }

  // Fields:
  //   - GuestProfile: standard fields → _profile.X, address subs → _profile.Address.X, custom → _profile.Attributes.X
  //   - Child types (Reservation/Folio/...): Target OMITTED — each PutProfileObject creates an instance
  const STANDARD = new Set(['FirstName','LastName','MiddleName','BirthDate','Gender',
    'EmailAddress','PersonalEmailAddress','BusinessEmailAddress',
    'PhoneNumber','HomePhoneNumber','MobilePhoneNumber','BusinessPhoneNumber',
    'AccountNumber','PartyType','BusinessName']);
  const ADDRESS: Record<string,string> = {
    Address1:'Address.Address1', Address2:'Address.Address2', Address3:'Address.Address3', Address4:'Address.Address4',
    City:'Address.City', State:'Address.State', County:'Address.County', Country:'Address.Country',
    Province:'Address.Province', PostalCode:'Address.PostalCode',
  };

  const fields: Record<string, any> = {};
  for (const f of props.Fields as any[]) {
    const entry: any = { Source: `_source.${f.name}`, ContentType: f.type === 'NUMBER' ? 'NUMBER' : 'STRING' };
    if (objectTypeName === 'GuestProfile') {
      if (STANDARD.has(f.name))      entry.Target = `_profile.${f.name}`;
      else if (ADDRESS[f.name])      entry.Target = `_profile.${ADDRESS[f.name]}`;
      else                           entry.Target = `_profile.Attributes.${f.name}`;
    }
    // child types: NO target
    fields[f.name] = entry;
  }

  // Only the parent type may create profiles. Children link or skip.
  const allowProfileCreation = objectTypeName === 'GuestProfile';

  // Object Type Keys are immutable after creation. PutProfileObjectType cannot
  // change Keys/StandardIdentifiers in place. Schema rev → delete-recreate.
  if (event.RequestType === 'Update') {
    await cp.send(new DeleteProfileObjectTypeCommand({ DomainName: DOMAIN, ObjectTypeName: objectTypeName }))
      .catch(e => { if (e.name !== 'ResourceNotFoundException') console.error(e); });
  }

  await cp.send(new PutProfileObjectTypeCommand({
    DomainName: DOMAIN, ObjectTypeName: objectTypeName,
    Description: props.Description, Keys: keys, Fields: fields,
    AllowProfileCreation: allowProfileCreation,
    ExpirationDays: 365,
  }));

  return { PhysicalResourceId: `objtype-${objectTypeName}`, Data: { ObjectTypeName: objectTypeName } };
}
```

CDK 측에서는 schema bump 시 CR re-run을 강제하기 위해 `properties.SchemaRev` 같은 cache-buster를 항상 포함:

```typescript
// lib/profiles-stack.ts
const SCHEMA_REV = 'rev3-guestkey-profile-link';   // bump on any Keys/Fields change
for (const objType of schemaConfig.object_types) {
  new cdk.CustomResource(this, `ObjType${objType.name}`, {
    serviceToken: provider.serviceToken,
    properties: {
      ObjectTypeName: objType.name,
      Description: objType.description,
      Keys: objType.keys,
      Fields: objType.fields,
      SchemaRev: SCHEMA_REV,
    },
  });
}
```

## Profile Import Handler — Golden Record → CP

ER 매칭 결과를 GuestProfile object로 PUT.

```typescript
// backend/lambdas/profile-import/handler.ts
import { CustomerProfilesClient, PutProfileObjectCommand, SearchProfilesCommand, DeleteProfileCommand } from '@aws-sdk/client-customer-profiles';

const cp = new CustomerProfilesClient({});
const DOMAIN = process.env.CP_DOMAIN_NAME!;

async function run(matchingType: 'simple'|'advanced'|'ml', replaceExisting: boolean) {
  const variants = await loadVariants();              // S3 ER input CSV
  const groups = await loadGroupings(matchingType);   // DynamoDB matching results: matchId → [variantId,...]

  if (replaceExisting) await deleteExistingGoldenProfiles(groups);

  let imported = 0;
  for (const [matchId, members] of groups) {
    const golden = buildGolden(matchId, members, variants);
    if (!golden) continue;

    // ⚠️ FIELD NAMES MUST MATCH ObjectType.Fields exactly.
    //    Unknown fields are silently dropped.
    //    Do NOT include `ProfileId` (CP-reserved) — use `GuestProfileId` only.
    const objectPayload: Record<string, string> = {
      GuestProfileId: golden.goldenProfileId,         // ← link key (same name as Reservation.GuestKey)
      MatchId: matchId,
      MatchingType: matchingType,
      VariantCount: String(golden.variantCount),
      FirstName: golden.fields.firstname ?? '',
      LastName: golden.fields.lastname ?? '',
      EmailAddress: golden.fields.email ?? '',
      PhoneNumber: golden.fields.phone ?? '',
      BirthDate: golden.fields.dateofbirth ?? '',
      LoyaltyNumber: golden.fields.loyaltynumber ?? '',
      Address1: golden.fields.street ?? '',           // flat — NOT nested { Address: { ... } }
      City: golden.fields.city ?? '',
      State: golden.fields.state ?? '',
      PostalCode: golden.fields.postalcode ?? '',
      Country: golden.fields.country ?? '',
      SourceChannels: golden.fields.sourcechannel ?? '',
      SourceVariantIds: golden.sources.join(','),
      ImportedAt: new Date().toISOString(),
    };

    await cp.send(new PutProfileObjectCommand({
      DomainName: DOMAIN, ObjectTypeName: 'GuestProfile',
      Object: JSON.stringify(objectPayload),
    }));
    imported++;
  }
  return { importedCount: imported };
}
```

## CP Data Import Handler — Reservation/Folio → CP

PostgreSQL의 거래성 데이터를 자식 ObjectType의 instance로 PUT. **`AllowProfileCreation: false`**(자식 type에서)이라 GuestProfileId가 매칭되는 부모가 있어야만 attach 된다.

```typescript
// backend/lambdas/cp-data-import/handler.ts (요지)
for (const row of reservationRows) {
  const goldenId = guestToGolden.get(row.guest_id);
  if (!goldenId) { unmatchedGuestIds++; continue; }   // skip — no parent profile

  await cp.send(new PutProfileObjectCommand({
    DomainName: DOMAIN, ObjectTypeName: 'Reservation',
    Object: JSON.stringify({
      // GuestProfileId is the link key. NO `ProfileId` field.
      GuestProfileId: goldenId,
      ReservationId: row.reservation_id,
      PropertyCode: row.property_code,
      CheckInDate: row.check_in_date,
      CheckOutDate: row.check_out_date,
      NumberOfNights: Number(row.number_of_nights ?? 0),
      AverageDailyRate: Number(row.average_daily_rate ?? 0),
      TotalAmount: Number(row.total_amount ?? 0),
      // ... 모든 필드는 schema.yaml ObjectType.fields에 선언되어야 함
    }),
  }));
}
```

### Fire-and-forget 패턴 (대용량 import)

API Gateway는 29초 타임아웃이라 수백~수천 PutProfileObject는 동기 처리 불가. Lambda self-invoke로 분리:

```typescript
// /api/cp-data-import/run (API path)
if (path.endsWith('/run') && method === 'POST') {
  await lambdaClient.send(new InvokeCommand({
    FunctionName: process.env.AWS_LAMBDA_FUNCTION_NAME!,
    InvocationType: 'Event',                             // async
    Payload: Buffer.from(JSON.stringify({ __mode: 'WORKER' })),
  }));
  return ok({ status: 'STARTED', message: 'Background import 3-10 min' });
}

// Worker mode in same handler
if ((event as any).__mode === 'WORKER') {
  await runImport();   // long-running PutProfileObject loop
  return;
}
```

CDK에서 self-invoke IAM policy 필수:

```typescript
fn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['lambda:InvokeFunction'],
  resources: [`arn:aws:lambda:${cdk.Aws.REGION}:${cdk.Aws.ACCOUNT_ID}:function:${projectName}-cp-data-import`],
}));
```

## Cleanup Helper (도메인 wipe 없이 profile 정리)

ObjectType이 delete-recreate되면 child instance는 함께 사라지지만, profile 자체는 남아 stale state. SearchProfiles로 알려진 키 값들을 sweep해서 삭제:

```typescript
async function cleanupAllProfiles() {
  const seen = new Set<string>();
  for (const goldenId of allGoldenIds) {
    const r = await cp.send(new SearchProfilesCommand({
      DomainName: DOMAIN, KeyName: 'GuestKey', Values: [goldenId], MaxResults: 10,
    })).catch(() => ({ Items: [] }));
    for (const item of r.Items ?? []) {
      const pid = item.ProfileId;
      if (!pid || seen.has(pid)) continue;
      seen.add(pid);
      await cp.send(new DeleteProfileCommand({ DomainName: DOMAIN, ProfileId: pid }))
        .catch(e => console.warn(e.message));
    }
  }
  return { deleted: seen.size };
}
```

## Ingestion Handler

데이터 수집 (CSV 업로드 + Glue 테이블 적재)

```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { parse } from 'csv-parse/sync';

const DATA_BUCKET = process.env.DATA_BUCKET!;

// POST /api/ingestion/upload — CSV 업로드
async function uploadCsv(body: { fileName: string; content: string; channel: string }) {
  const records = parse(body.content, { columns: true, skip_empty_lines: true });

  // Validate required fields (variantid must exist)
  for (const record of records) {
    if (!record.variantid) {
      return error(400, 'Each record must have a variantid field');
    }
    // Add source channel if not present
    if (!record.sourcechannel) {
      record.sourcechannel = body.channel;
    }
  }

  // Write to S3 in Glue-compatible format (CSV with headers)
  const csvContent = [
    Object.keys(records[0]).join(','),
    ...records.map(r => Object.values(r).join(',')),
  ].join('\n');

  await s3Client.send(new PutObjectCommand({
    Bucket: DATA_BUCKET,
    Key: `er-input/${body.channel}/${body.fileName}`,
    Body: csvContent,
    ContentType: 'text/csv',
  }));

  return { recordCount: records.length, channel: body.channel, path: `er-input/${body.channel}/${body.fileName}` };
}

// POST /api/ingestion/generate — 샘플 데이터 생성 (데모용)
async function generateSampleData(body: { count: number; channels: string[] }) {
  // Use faker or custom logic to generate PII variants
  // Each "person" gets multiple variants across channels with realistic variations
  // → See data-generator patterns in travel demo
}
```

## Rule Management Handler

ER 규칙 CRUD + 변경 이력

```typescript
const RULE_HISTORY_TABLE = process.env.RULE_HISTORY_TABLE!;

interface RuleChange {
  changeId: string;
  timestamp: string;
  action: 'CREATE' | 'UPDATE' | 'DELETE';
  ruleName: string;
  previousRule?: { matchingKeys: string[] };
  newRule?: { matchingKeys: string[] };
  source: 'MANUAL' | 'AI_APPROVED';
  approvedBy?: string;
}

// GET /api/rules — 현재 규칙 목록
async function listRules() {
  // ER GetMatchingWorkflow → extract rules
}

// POST /api/rules — 규칙 추가
async function createRule(body: { ruleName: string; matchingKeys: string[] }) {
  // 1. Update ER Workflow with new rule
  // 2. Record change history
}

// PUT /api/rules/:ruleName — 규칙 수정
async function updateRule(ruleName: string, body: { matchingKeys: string[] }) {
  // 1. Get current rule (for history)
  // 2. Update ER Workflow
  // 3. Record change history
}

// GET /api/rules/history — 변경 이력
async function getRuleHistory() {
  const { Items } = await ddbClient.send(new QueryCommand({
    TableName: RULE_HISTORY_TABLE,
    KeyConditionExpression: 'pk = :pk',
    ExpressionAttributeValues: { ':pk': 'HISTORY' },
    ScanIndexForward: false, // newest first
    Limit: 50,
  }));
  return Items || [];
}
```

## Graph RAG Handler (선택)

Neptune + Bedrock를 활용한 자연어 그래프 쿼리

```typescript
import { NeptuneClient } from './shared/neptune-client';

const neptuneClient = new NeptuneClient(process.env.NEPTUNE_ENDPOINT!);

// POST /api/graph/query — 자연어 질의
async function graphQuery(body: { question: string; context?: string }) {
  // 1. Bedrock로 질문 → openCypher 쿼리 변환
  const cypherQuery = await generateCypher(body.question);

  // 2. Neptune에서 쿼리 실행
  const graphResult = await neptuneClient.executeOpenCypher(cypherQuery);

  // 3. Bedrock로 결과 → 자연어 답변 생성
  const answer = await generateAnswer(body.question, graphResult);

  return { question: body.question, cypher: cypherQuery, answer, rawResult: graphResult };
}

async function generateCypher(question: string): Promise<string> {
  // Prompt includes graph schema (node labels, relationship types, properties)
  // Returns valid openCypher query
  const prompt = `Given this graph schema:
Nodes: Customer(id, name, email, segment), Booking(id, date, amount), Hotel(name, city)
Relationships: BOOKED(Customer→Booking), STAYED_AT(Booking→Hotel), KNOWS(Customer→Customer)

Convert this question to an openCypher query:
"${question}"`;

  // Call Bedrock...
  return ''; // parsed cypher
}
```

## Graph Sync Handler (선택)

Customer Profiles → Neptune 동기화

```typescript
// POST /api/graph/sync — 프로필을 그래프로 동기화
async function syncProfilesToGraph(body: { profileIds?: string[] }) {
  // 1. CP에서 프로필 조회
  // 2. 프로필 → Neptune 노드 (Customer, with properties)
  // 3. Object 데이터 → Neptune 노드 + 엣지 (Booking→Hotel 등)
  // 4. 매칭 결과 → SAME_AS 관계

  // Upsert pattern (idempotent):
  const upsertCypher = `
    MERGE (c:Customer {id: $profileId})
    SET c.name = $name, c.email = $email, c.segment = $segment, c.updatedAt = datetime()
    RETURN c
  `;
}
```
