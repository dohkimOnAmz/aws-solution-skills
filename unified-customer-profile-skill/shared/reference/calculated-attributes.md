# Calculated Attributes — 정의에서 화면 표시까지

Customer Profiles의 Calculated Attribute는 **Object Type instance를 기반으로 계산되는 동적 집계값**이다 (예: 연 매출, 누적 박수, 가장 최근 방문일). 정의만 만든다고 값이 나오는 게 아니라, **올바른 child instance가 attach된 뒤에야 채워진다**. 이 흐름을 안내하지 않으면 사용자는 "왜 다 0이 뜨지?"로 혼란을 겪는다.

## 동작 흐름 (반드시 사용자에게 가이드)

```
1) ObjectType 정의 (schema.yaml + Custom Resource)
2) Calculated Attribute 정의 (CreateCalculatedAttributeDefinition)
3) GuestProfile import (profile-import handler)              ← parent profile
4) Reservation/Folio import (cp-data-import handler)         ← child instances
5) CP indexing 진행 (Status: PREPARING)
6) Status가 COMPLETED + Readiness.ProgressPercentage=100 → 사용 가능
7) GetCalculatedAttributeForProfile 호출 → Value 반환
```

**Step 4를 빠뜨리면 Step 7에서 항상 0/null/empty.** 사용자가 가장 많이 막히는 지점.

## 정의 — `config/schema.yaml`

```yaml
calculated_attributes:
  - name: TotalHotelRevenueInAYear
    object_type: Reservation              # ← 하나의 ObjectType만 참조 가능
    field: TotalAmount                     # ← 그 ObjectType의 Fields map의 키 이름
    aggregation: SUM                       # SUM | COUNT | AVERAGE | MIN | MAX | FIRST_OCCURRENCE | LAST_OCCURRENCE | MAX_OCCURRENCE
    threshold_days: 365                    # 시간 윈도우 (0 = 전체 기간)
  - name: TotalNightsInAYear
    object_type: Reservation
    field: NumberOfNights
    aggregation: SUM
    threshold_days: 365
  - name: AverageDailyRateLast365
    object_type: Reservation
    field: AverageDailyRate
    aggregation: AVERAGE
    threshold_days: 365
  - name: TotalFolioSpendInAYear
    object_type: Folio
    field: Amount
    aggregation: SUM
    threshold_days: 365
  - name: MostRecentStayDate
    object_type: Reservation
    field: CheckInDate
    aggregation: MAX_OCCURRENCE            # 가장 최근 ingest된 instance의 해당 필드값
    threshold_days: 0
```

### 제약 (AWS 공식)

- **하나의 ObjectType + 최대 2개 필드**까지만 참조 가능 (수식: `{Type.Field1} + {Type.Field2}`)
- **Expression 형식: `{ObjectTypeName.AttributeName}`** — Target 경로 아니다. Fields map의 **key 이름**으로 참조
- **AttributeName**은 Object Type의 Fields map에 선언되어 있어야 함. 선언 안 한 필드는 참조 불가
- **도메인당 최대 20개** (할당량 증가 신청 가능)
- **Conditions.Range.Unit**은 `DAYS`만 지원
- **UseHistoricalData=true**: 정의 생성 이전에 이미 ingest된 instance도 계산에 포함

## Custom Resource — 정의 생성/업데이트

```typescript
// backend/custom-resources/create-calculated-attributes/handler.ts
import {
  CustomerProfilesClient,
  CreateCalculatedAttributeDefinitionCommand,
  UpdateCalculatedAttributeDefinitionCommand,
  DeleteCalculatedAttributeDefinitionCommand,
} from '@aws-sdk/client-customer-profiles';

const cp = new CustomerProfilesClient({});
const DOMAIN = process.env.CP_DOMAIN_NAME!;

export async function handler(event) {
  const props = event.ResourceProperties;
  const name = props.CalculatedAttributeName;

  if (event.RequestType === 'Delete') {
    await cp.send(new DeleteCalculatedAttributeDefinitionCommand({
      DomainName: DOMAIN, CalculatedAttributeName: name,
    })).catch(e => { if (e.name !== 'ResourceNotFoundException') console.error(e); });
    return { PhysicalResourceId: `calc-${name}` };
  }

  const conditions = props.ThresholdDays && Number(props.ThresholdDays) > 0
    ? { Range: { Value: Number(props.ThresholdDays), Unit: 'DAYS' } }
    : undefined;

  const objectTypeName = props.ObjectTypeName;
  const attrs = (props.AttributeDetails.Attributes as any[]).map(a => ({ Name: a.Name }));
  const expression = attrs.map(a => `{${objectTypeName}.${a.Name}}`).join(' + ');

  const createParams = {
    DomainName: DOMAIN,
    CalculatedAttributeName: name,
    DisplayName: name,
    Description: `Calculated attribute: ${name}`,
    AttributeDetails: { Attributes: attrs, Expression: expression },
    Statistic: props.Statistic,
    UseHistoricalData: true,                       // ← important
    ...(conditions && { Conditions: conditions }),
  };

  // Domain wipe 후 재배포 시 Update 이벤트가 와도 정의는 없을 수 있다 — 안전망
  if (event.RequestType === 'Create') {
    try {
      await cp.send(new CreateCalculatedAttributeDefinitionCommand(createParams));
    } catch (e: any) {
      if (e.name !== 'ConflictException' && e.name !== 'BadRequestException') throw e;
      await updateOnly(name, conditions);
    }
  } else {
    try {
      await updateOnly(name, conditions);
    } catch (e: any) {
      if (e.name !== 'ResourceNotFoundException') throw e;
      await cp.send(new CreateCalculatedAttributeDefinitionCommand(createParams));
    }
  }
  return { PhysicalResourceId: `calc-${name}`, Data: { Name: name } };
}

async function updateOnly(name: string, conditions: any) {
  await cp.send(new UpdateCalculatedAttributeDefinitionCommand({
    DomainName: DOMAIN, CalculatedAttributeName: name,
    DisplayName: name, Description: `Calculated attribute: ${name}`,
    ...(conditions && { Conditions: conditions }),
  }));
}
```

## CDK — Custom Resource 등록 + cache buster

```typescript
// lib/profiles-stack.ts (요지)
const SCHEMA_REV = 'rev3-guestkey-profile-link';
let prevCalc: cdk.CustomResource | undefined;
for (const calc of schemaConfig.calculated_attributes) {
  const calcResource = new cdk.CustomResource(this, `CalcAttr${calc.name}`, {
    serviceToken: new cr.Provider(this, `CalcAttrProvider${calc.name}`, {
      onEventHandler: calcAttrFn,
    }).serviceToken,
    properties: {
      CalculatedAttributeName: calc.name,
      ObjectTypeName: calc.object_type,
      AttributeDetails: { Attributes: [{ Name: calc.field }] },
      Statistic: calc.aggregation,
      ThresholdDays: calc.threshold_days,
      SchemaRev: SCHEMA_REV,                       // bump triggers re-run
    },
  });
  // ObjectType이 먼저 만들어진 뒤 calc를 만들어야 함
  calcResource.node.addDependency(prevCalc ?? cpDomain);
  prevCalc = calcResource;
}
```

## 사용 — `GetCalculatedAttributeForProfile`

```typescript
import { GetCalculatedAttributeForProfileCommand, ListCalculatedAttributeDefinitionsCommand } from '@aws-sdk/client-customer-profiles';

async function loadCalcAttrs(profileId: string) {
  const { Items: defs } = await cp.send(new ListCalculatedAttributeDefinitionsCommand({
    DomainName: DOMAIN, MaxResults: 30,
  }));

  const calculated: Record<string, string | undefined> = {};
  for (const ca of defs ?? []) {
    if (!ca.CalculatedAttributeName) continue;
    if (ca.CalculatedAttributeName.startsWith('_')) continue;     // skip system-defined
    try {
      const r = await cp.send(new GetCalculatedAttributeForProfileCommand({
        DomainName: DOMAIN, ProfileId: profileId,
        CalculatedAttributeName: ca.CalculatedAttributeName,
      }));
      calculated[ca.CalculatedAttributeName] = r.Value;
    } catch { /* skip */ }
  }
  return calculated;
}
```

응답 예 (READY 상태):
```json
{
  "CalculatedAttributeName": "TotalHotelRevenueInAYear",
  "Value": "1123950",
  "IsDataPartial": "true",                 // 윈도우 안 instance만 사용
  "LastObjectTimestamp": "2026-05-15T00:23:09Z"
}
```

## Status 추적

```bash
aws customer-profiles get-calculated-attribute-definition \
  --domain-name <domain> --calculated-attribute-name TotalHotelRevenueInAYear \
  --query '[Status,Readiness]'
```

```
PREPARING → COMPLETED (Readiness.ProgressPercentage = 100)
```

`PREPARING`이면 instance ingestion 직후 indexing 중이므로, **사용자에게 "수 분 기다리세요"가 아니라 "Step 4 (Send to CP) 끝났는지 먼저 확인하세요"** 라고 가이드한다 (Step 4 누락이 가장 흔한 실수).

## UI 가이드 — 사용자에게 보여줘야 할 단계

`ProfileViewPage`의 Calculated Attributes 섹션에 다음 안내를 항상 표시:

> 📌 Calculated Attributes는 다음 4단계가 모두 완료되어야 값이 채워집니다:
>
> 1. ER 매칭 실행 (Matching 페이지)
> 2. **Send to CP — Step 1: Golden Profile Import** (Profile Import 페이지)
> 3. **Send to CP — Step 2: Reservation/Folio Import** ← 누락 시 항상 0/null
> 4. CP가 인덱싱 완료 (수 분~수십 분, 정의의 Status가 `COMPLETED`로 바뀔 때까지)

빈 값이면 단계 4 안내 + Status 조회 버튼을 함께 노출하는 게 사용자 경험에 좋다.

## 자주 틀리는 패턴 (디버깅 체크리스트)

| 증상 | 원인 | 수정 |
|---|---|---|
| 모든 calc attr가 0/empty | Step 4 (cp-data-import) 미실행 또는 child instance가 부모에 attach 안 됨 | `ListProfileObjects(profileId, ObjectTypeName=Reservation)` 호출해서 instance가 있는지 먼저 확인 |
| Reservation 들어왔는데 calc는 0 | `field`가 ObjectType.Fields map에 없거나 ContentType이 STRING (NUMBER aggregation 불가) | schema.yaml의 ObjectType field에 `type: NUMBER` 명시 |
| ProfileCount 폭증 (inferred profiles) | child Object Type이 `_profileId` 키를 가졌거나 `AllowProfileCreation: true` | `_profileId` 제거 + child는 `AllowProfileCreation: false` (`shared/patterns/lambda-handlers.md` 참조) |
| Status가 계속 PREPARING | 아직 indexing 중 또는 ObjectType과 Calc Attr 정의가 비어있는 도메인 | 1-3분 대기 후 재확인. ObjectType instance가 0개면 절대 COMPLETED 안 됨 |
| Expression 에러 | `{Reservation.totalAmount}`처럼 case 다름 | Fields map key는 case-sensitive. `{Reservation.TotalAmount}` 정확히 일치해야 함 |
| MAX_OCCURRENCE인데 빈 값 | Conditions.Range가 너무 짧아 instance가 없음 | `threshold_days: 0` (전체 기간) 또는 늘림 |
