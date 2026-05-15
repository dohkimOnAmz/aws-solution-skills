# ETL Transform Patterns

데이터 수집 후 Entity Resolution 실행 전에 적용하는 **Raw → ER input** 파이프라인 패턴.

## End-to-End 파이프라인 구조 (필수 이해)

```
PostgreSQL/CSV/JDBC (Raw)
   │  guests / reservations / folios / loyalty_members / guest_preferences
   │  └─ 정규화 안 된 PII (한영 혼용, 전화 포맷 다양, OTA relay email...)
   ▼
[Glue Crawler] — Glue Catalog Table 등록 (src_*)
   │
   ▼
[Glue ETL Job (PySpark)] — 핵심: 한 명의 guest를 N개 채널 variant로 폭발
   │   1) PII 정규화 (이름/전화/이메일)
   │   2) 채널별 variant 생성 (HOTEL_WEB, HOTEL_OTA, WALK_IN…)
   │   3) variantId = `V-{guest_id}-{n}` (ER가 그룹핑할 단위)
   │   4) ER input 스키마로 컬럼 정렬
   ▼
S3: er-input/unified/*.csv  ← ER이 직접 읽는 입력 (Glue Table로 등록)
   │
   ▼
[Entity Resolution Workflow] — Simple / Advanced / ML 매칭 실행
   ▼
DynamoDB matching_results (matchId 그룹)
   ▼
[profile-import Lambda] — 골든 레코드 → CP GuestProfile
   ▼
[cp-data-import Lambda] — PostgreSQL → CP Reservation/Folio (child instances)
   ▼
Calculated Attributes 자동 계산
```

**왜 variant 확장이 필요한가**: 한 명의 guest_id가 한 행만 있으면 ER이 그룹핑할 게 없다. 채널별로 살짝 다르게 표현된 행을 만들어줘야(예: phone format 3종, name locale 2종) ER 매칭의 효과가 가시화된다. 데모/PoC용. 운영 환경에선 이미 채널별로 분산된 실 데이터를 unify하면 됨.

## 왜 ETL이 필요한가

```
Raw Data (DB/CSV/Parquet)
  │
  │  이름: "김민호" / "KIM MINHO" / "Kim, Min-Ho" / "MINHO KIM"
  │  전화: "+82 10-1234-5678" / "010-1234-5678" / "01012345678"
  │  이메일: "Minho.Kim@Gmail.COM" / "guest-abc123@booking.com"
  │
  ▼ [ETL Transform]
  │
  │  이름: "MINHO" + "KIM" (정규화)
  │  전화: "01012345678" (숫자만)
  │  이메일: "minho.kim@gmail.com" (소문자, relay 태깅)
  │
  ▼
Entity Resolution (정규화된 데이터로 정확도 향상)
```

**ETL 없이 ER을 돌리면**: Advanced Rule의 fuzzy matching이 일부 커버하지만,
극단적 변형(한영 혼용, 국가코드 유무)은 놓칠 수 있음.

## ETL 실행 전략 (Decision)

```
Q: 데이터 품질/변형 수준은?
│
├─ 이미 정형화됨 (단일 시스템에서 export, 포맷 일관)
│   └─ ETL 불필요 → 바로 ER 실행
│
├─ 경미한 변형 (대소문자, 공백, 하이픈 정도)
│   └─ Lightweight ETL (Lambda 내 인라인 변환)
│       └─ 비용: 거의 0 (Lambda 실행 시간에 포함)
│
├─ 심각한 변형 (한영 혼용, 다국어, 주소 비정형)
│   └─ Full ETL (Glue ETL Job)
│       └─ 비용: $0.44/DPU-hour (보통 2 DPU × 5~10분 = ~$0.07/회)
│
└─ 대용량 + 복잡한 변환 (수백만 레코드, 조인 필요)
    └─ Glue ETL Job (Spark) 또는 Step Functions 파이프라인
        └─ 비용: DPU × 시간 비례
```

## Lightweight ETL (Lambda 인라인)

소량 데이터, 단순 변환 시 Lambda 핸들러 내에서 직접 처리.

```typescript
// backend/lambdas/ingestion/transforms.ts

/**
 * PII 정규화 파이프라인
 * 순서: 공백정리 → 이름정규화 → 전화정규화 → 이메일정규화 → 주소정규화
 */
export function normalizeRecord(record: Record<string, string>): Record<string, string> {
  const normalized = { ...record };

  // 1. 공통: 전후 공백 제거
  for (const key of Object.keys(normalized)) {
    if (typeof normalized[key] === 'string') {
      normalized[key] = normalized[key].trim();
    }
  }

  // 2. 이름 정규화
  if (normalized.firstname) normalized.firstname = normalizeName(normalized.firstname);
  if (normalized.lastname) normalized.lastname = normalizeName(normalized.lastname);

  // 3. 전화 정규화
  if (normalized.phone) normalized.phone = normalizePhone(normalized.phone);

  // 4. 이메일 정규화
  if (normalized.email) normalized.email = normalizeEmail(normalized.email);

  // 5. 주소 정규화
  if (normalized.postalcode) normalized.postalcode = normalizePostalCode(normalized.postalcode);

  return normalized;
}

// ─── 이름 정규화 ──────────────────────────────────────────

function normalizeName(input: string): string {
  // 1. 특수문자 제거 (하이픈, 점 등)
  let name = input.replace(/[-.'·]/g, '');

  // 2. 한글인지 영문인지 판별
  const isKorean = /[\uAC00-\uD7AF]/.test(name);

  if (isKorean) {
    // 한글 이름: 공백 제거 후 그대로 유지
    name = name.replace(/\s/g, '');
  } else {
    // 영문 이름: 대문자 변환, 공백 제거
    name = name.replace(/\s/g, '').toUpperCase();
  }

  return name;
}

/**
 * 한국 이름 성/이름 분리
 * - 한글 2~4글자: 첫 글자 = 성, 나머지 = 이름
 * - 영문: "KIM MINHO" → { lastName: "KIM", firstName: "MINHO" }
 *         "MINHO KIM" → 감지하여 swap
 */
export function splitKoreanName(fullName: string): { firstName: string; lastName: string } {
  const trimmed = fullName.trim();
  const isKorean = /[\uAC00-\uD7AF]/.test(trimmed);

  if (isKorean) {
    // 한글: 첫 글자 = 성 (1~2자), 나머지 = 이름
    const noSpace = trimmed.replace(/\s/g, '');
    // 대부분 성이 1자 (90%+), 2자 성은 남궁/선우/사공/독고 등
    const twoCharSurnames = ['남궁', '선우', '사공', '독고', '황보', '제갈', '하동'];
    const isTwoChar = twoCharSurnames.some(s => noSpace.startsWith(s));
    const lastNameLen = isTwoChar ? 2 : 1;
    return {
      lastName: noSpace.slice(0, lastNameLen),
      firstName: noSpace.slice(lastNameLen),
    };
  } else {
    // 영문: 공백으로 split
    const parts = trimmed.toUpperCase().split(/\s+/);
    if (parts.length === 1) return { firstName: parts[0], lastName: '' };

    // 한국식 영문 순서 감지: "KIM MINHO" vs "MINHO KIM"
    // 한국 성 목록 (상위 10개)
    const koreanSurnames = ['KIM', 'LEE', 'PARK', 'CHOI', 'JUNG', 'KANG',
      'CHO', 'YUN', 'JANG', 'LIM', 'HAN', 'OH', 'SEO', 'SHIN', 'KWON',
      'HWANG', 'AHN', 'SONG', 'RYU', 'HONG', 'YOO', 'MOON', 'YANG', 'NOH'];

    if (koreanSurnames.includes(parts[0]) && !koreanSurnames.includes(parts[parts.length - 1])) {
      // "KIM MINHO" → 성이 앞에
      return { lastName: parts[0], firstName: parts.slice(1).join('') };
    } else {
      // "MINHO KIM" → 서양식 또는 성이 뒤에
      return { firstName: parts.slice(0, -1).join(''), lastName: parts[parts.length - 1] };
    }
  }
}

// ─── 전화번호 정규화 ──────────────────────────────────────

function normalizePhone(input: string): string {
  // 1. 숫자만 추출
  let digits = input.replace(/[^\d+]/g, '');

  // 2. 국가코드 제거
  if (digits.startsWith('+82')) digits = '0' + digits.slice(3);
  if (digits.startsWith('82') && digits.length > 10) digits = '0' + digits.slice(2);

  // 3. 앞에 0 없으면 추가
  if (!digits.startsWith('0') && digits.length === 10) digits = '0' + digits;

  return digits; // 결과: "01012345678"
}

// ─── 이메일 정규화 ──────────────────────────────────────

interface NormalizedEmail {
  email: string;
  isRelay: boolean;
  relayProvider?: string;
}

function normalizeEmail(input: string): string {
  // 소문자 변환
  return input.toLowerCase().trim();
}

export function detectRelayEmail(email: string): NormalizedEmail {
  const normalized = email.toLowerCase().trim();
  const relayPatterns: Record<string, RegExp> = {
    'booking.com': /guest-.*@booking\.com/,
    'expedia': /.*@guest\.expedia\.com/,
    'hotels.com': /.*@guest\.hotels\.com/,
    'coupang': /.*@buyer\.coupang\.com/,
    'naver': /.*@relay\.naver\.com/,
  };

  for (const [provider, pattern] of Object.entries(relayPatterns)) {
    if (pattern.test(normalized)) {
      return { email: normalized, isRelay: true, relayProvider: provider };
    }
  }

  return { email: normalized, isRelay: false };
}

// ─── 우편번호 정규화 ──────────────────────────────────────

function normalizePostalCode(input: string): string {
  // 한국: 5자리 숫자 (구: 6자리 → 5자리로 변환 불가하므로 그대로)
  const digits = input.replace(/[^\d]/g, '');
  return digits;
}

// ─── 데이터 품질 스코어 ───────────────────────────────────

export interface QualityScore {
  completeness: number;  // 0~1: 필수 필드 채워짐 비율
  validity: number;      // 0~1: 형식 유효성 비율
  overall: number;       // 가중 평균
}

export function calculateQuality(record: Record<string, string>, requiredFields: string[]): QualityScore {
  // Completeness: 필수 필드 중 비어있지 않은 것
  const filled = requiredFields.filter(f => record[f] && record[f].trim() !== '');
  const completeness = filled.length / requiredFields.length;

  // Validity: 형식 검증
  let validCount = 0;
  let checkCount = 0;
  if (record.email) { checkCount++; if (/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(record.email)) validCount++; }
  if (record.phone) { checkCount++; if (/^0\d{9,10}$/.test(normalizePhone(record.phone))) validCount++; }
  const validity = checkCount > 0 ? validCount / checkCount : 1;

  return { completeness, validity, overall: completeness * 0.7 + validity * 0.3 };
}
```

## Full ETL (Glue Job — PySpark)

대용량 데이터, 복잡한 변환 시 Glue ETL Job으로 처리.

### CDK 리소스

```typescript
// lib/etl-stack.ts (또는 ingestion-stack.ts에 포함)
import * as glue from 'aws-cdk-lib/aws-glue';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as s3assets from 'aws-cdk-lib/aws-s3-assets';

export class EtlConstruct extends Construct {
  constructor(scope: Construct, id: string, props: EtlProps) {
    super(scope, id);

    // ETL 스크립트를 S3에 업로드
    const etlScript = new s3assets.Asset(this, 'EtlScript', {
      path: 'backend/glue-scripts/normalize-pii.py',
    });

    // Glue ETL Job Role
    const etlRole = new iam.Role(this, 'EtlRole', {
      assumedBy: new iam.ServicePrincipal('glue.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSGlueServiceRole'),
      ],
    });
    props.dataBucket.grantReadWrite(etlRole);

    // Glue ETL Job
    new glue.CfnJob(this, 'NormalizePiiJob', {
      name: `${props.projectName}-normalize-pii`,
      role: etlRole.roleArn,
      command: {
        name: 'glueetl',
        scriptLocation: etlScript.s3ObjectUrl,
        pythonVersion: '3',
      },
      defaultArguments: {
        '--TempDir': `s3://${props.dataBucket.bucketName}/glue-temp/`,
        '--job-bookmark-option': 'job-bookmark-enable', // 증분 처리
        '--SOURCE_PATH': `s3://${props.dataBucket.bucketName}/er-input-raw/`,
        '--TARGET_PATH': `s3://${props.dataBucket.bucketName}/er-input/`, // 정규화된 결과
        '--DATABASE_NAME': props.glueDbName,
        '--TABLE_NAME': props.glueTableName,
      },
      glueVersion: '4.0',
      numberOfWorkers: 2,
      workerType: 'G.1X', // 4 vCPU, 16GB — 소규모 충분
      timeout: 30, // 30분
    });
  }
}
```

### Glue ETL Script (PySpark)

```python
# backend/glue-scripts/normalize-pii.py
import sys
import re
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql.functions import udf, col, lower, trim, regexp_replace, when, lit
from pyspark.sql.types import StringType, StructType, StructField, FloatType

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'SOURCE_PATH', 'TARGET_PATH', 'DATABASE_NAME', 'TABLE_NAME'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# ─── UDFs ─────────────────────────────────────────────

@udf(StringType())
def normalize_phone(phone):
    """전화번호 정규화: 숫자만 추출 + 국가코드 제거"""
    if not phone:
        return None
    digits = re.sub(r'[^\d]', '', phone)
    # +82 제거
    if digits.startswith('82') and len(digits) > 10:
        digits = '0' + digits[2:]
    # 앞에 0 없으면 추가
    if not digits.startswith('0') and len(digits) == 10:
        digits = '0' + digits
    return digits

@udf(StringType())
def normalize_name(name):
    """이름 정규화: 특수문자 제거, 공백 제거, 대문자"""
    if not name:
        return None
    # 한글 판별
    if re.search(r'[\uAC00-\uD7AF]', name):
        return re.sub(r'\s', '', name)
    else:
        return re.sub(r'[-.\'\s]', '', name).upper()

@udf(StringType())
def detect_relay_email(email):
    """릴레이 이메일 감지 → 태그 반환 (relay:provider 또는 None)"""
    if not email:
        return None
    e = email.lower().strip()
    patterns = {
        'booking.com': r'guest-.*@booking\.com',
        'expedia': r'.*@guest\.expedia\.com',
        'hotels.com': r'.*@guest\.hotels\.com',
    }
    for provider, pattern in patterns.items():
        if re.match(pattern, e):
            return f'relay:{provider}'
    return None

@udf(FloatType())
def quality_score(firstname, lastname, email, phone):
    """데이터 품질 스코어 (0~1)"""
    fields = [firstname, lastname, email, phone]
    filled = sum(1 for f in fields if f and f.strip())
    return filled / len(fields)

# ─── Main Transform ───────────────────────────────────

# 1. 원본 읽기 (CSV 또는 Parquet 자동 감지)
source_path = args['SOURCE_PATH']
if source_path.endswith('.parquet') or '/parquet/' in source_path:
    df = spark.read.parquet(source_path)
else:
    df = spark.read.option('header', 'true').csv(source_path)

# 2. 정규화 적용
df_normalized = df \
    .withColumn('firstname', normalize_name(col('firstname'))) \
    .withColumn('lastname', normalize_name(col('lastname'))) \
    .withColumn('phone', normalize_phone(col('phone'))) \
    .withColumn('email', lower(trim(col('email')))) \
    .withColumn('_relay_tag', detect_relay_email(col('email'))) \
    .withColumn('_quality_score', quality_score(col('firstname'), col('lastname'), col('email'), col('phone')))

# 3. 릴레이 이메일 처리: ER에서 email match key 제외할 수 있도록 태깅
# (실제 email 값은 유지하되, _relay_tag 컬럼으로 후처리 가능)

# 4. 품질 필터 (선택): 최소 품질 이하 레코드 경고
low_quality = df_normalized.filter(col('_quality_score') < 0.25)
if low_quality.count() > 0:
    print(f"⚠️ Low quality records: {low_quality.count()}")
    # DLQ 또는 별도 경로에 저장
    low_quality.write.mode('append').parquet(f"{args['TARGET_PATH']}_low_quality/")

# 5. 정규화된 결과 저장 (Parquet — ER 입력용)
df_clean = df_normalized.filter(col('_quality_score') >= 0.25) \
    .drop('_relay_tag', '_quality_score')  # 내부 컬럼 제거

df_clean.write.mode('overwrite').parquet(args['TARGET_PATH'])

# 6. Glue Data Catalog 업데이트 (Parquet 파티션 등록)
# → Crawler가 자동으로 해주거나, Catalog API로 직접 등록

job.commit()
```

## ETL 파이프라인 통합 (Step Functions)

복잡한 시나리오에서는 Step Functions로 오케스트레이션:

```
┌─────────────────────────────────────────────────────┐
│ Step Functions: UCP Data Pipeline                     │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ① Ingest ──→ ② ETL Normalize ──→ ③ ER Matching     │
│  (S3/DB)       (Glue Job)          (ER Job)          │
│                     │                    │            │
│                     ▼                    ▼            │
│              [품질 리포트]         ④ CP Import        │
│                                         │            │
│                                         ▼            │
│                                   ⑤ Graph Sync       │
│                                   (선택)              │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### CDK (Step Functions)

```typescript
import * as sfn from 'aws-cdk-lib/aws-stepfunctions';
import * as tasks from 'aws-cdk-lib/aws-stepfunctions-tasks';

// Glue ETL → ER → CP Import 파이프라인
const etlJob = new tasks.GlueStartJobRun(this, 'RunEtl', {
  glueJobName: `${projectName}-normalize-pii`,
  integrationPattern: sfn.IntegrationPattern.RUN_JOB, // 완료까지 대기
});

const erJob = new tasks.LambdaInvoke(this, 'RunMatching', {
  lambdaFunction: matchingLambda,
  payload: sfn.TaskInput.fromObject({ matchingType: 'advanced' }),
});

const cpImport = new tasks.LambdaInvoke(this, 'ImportToCP', {
  lambdaFunction: profilesLambda,
  payload: sfn.TaskInput.fromObject({ action: 'import-golden-records' }),
});

const pipeline = etlJob
  .next(erJob)
  .next(cpImport);

new sfn.StateMachine(this, 'DataPipeline', {
  stateMachineName: `${projectName}-data-pipeline`,
  definitionBody: sfn.DefinitionBody.fromChainable(pipeline),
  timeout: cdk.Duration.hours(1),
});
```

## Transform 규칙 설정 (Schema-Driven)

`config/schema.yaml`에 transform 규칙을 선언적으로 추가:

```yaml
# config/schema.yaml (확장)
features:
  ingestion:
    mode: glue_connection  # csv | parquet | glue_connection | kinesis | hybrid
  etl:
    enabled: true
    mode: glue_job         # inline (Lambda) | glue_job (PySpark)
    transforms:
      - field: firstname
        operations: [trim, remove_special, uppercase, remove_spaces]
      - field: lastname
        operations: [trim, remove_special, uppercase, remove_spaces]
      - field: phone
        operations: [digits_only, remove_country_code_kr, ensure_leading_zero]
      - field: email
        operations: [lowercase, trim, tag_relay]
      - field: postalcode
        operations: [digits_only]
    quality_filter:
      min_score: 0.25      # 이 이하는 DLQ로
      required_fields: [firstname, lastname]  # 하나라도 없으면 reject
    relay_email:
      action: tag          # tag | exclude | replace_with_null
      providers: [booking.com, expedia, hotels.com, coupang]
```

## Discovery 질문 (ETL 관련)

AI가 사용자에게 추가로 물어볼 것:

```
"데이터 정규화가 필요한가요?"
├─ 이미 깨끗함 → etl.enabled: false
├─ 이름/전화 포맷 불일치 → etl.mode: inline (Lambda 내 처리)
└─ 대용량 + 복잡한 변환 → etl.mode: glue_job

"한국어 이름이 포함되어 있나요?"
├─ YES → 한영 이름 정규화 + 성/이름 분리 로직 추가
└─ NO → 표준 영문 정규화만

"릴레이 이메일 (OTA/마켓플레이스) 처리는?"
├─ 태깅만 (ER에서 email match 가중치 낮춤) → tag
├─ ER 매칭에서 아예 제외 → exclude
└─ null로 치환 (phone/name으로 폴백) → replace_with_null
```

## Raw → ER Input — 풀 PySpark 스크립트 패턴 (`backend/glue-scripts/build-er-input.py`)

이 스크립트는 위 파이프라인 다이어그램의 **[Glue ETL Job]** 단계 전체를 구현. Glue Catalog의 `src_*` 테이블(Crawler가 PostgreSQL에서 등록)을 읽어 ER이 사용할 수 있는 단일 CSV로 변환한다.

```python
import sys, re, random
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.context import SparkContext
from pyspark.sql.functions import udf, col, lit, concat, explode, when
from pyspark.sql.types import StringType, ArrayType, StructType, StructField

args = getResolvedOptions(sys.argv, [
    'JOB_NAME', 'TARGET_S3_PATH', 'GLUE_DATABASE', 'GUESTS_TABLE',
])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext); job.init(args['JOB_NAME'], args)

# ─── Normalization UDFs ──────────────────────────────
@udf(StringType())
def normalize_name(name):
    if not name: return None
    if re.search(r'[가-힯]', name): return re.sub(r'\s', '', name)
    return re.sub(r"[-.'\s]", '', name).upper()

@udf(StringType())
def normalize_phone(phone):
    if not phone: return None
    digits = re.sub(r'[^\d]', '', phone)
    if digits.startswith('82') and len(digits) > 10: digits = '0' + digits[2:]
    if not digits.startswith('0') and len(digits) == 10: digits = '0' + digits
    return digits

@udf(StringType())
def mask_relay_email(email):
    if not email: return None
    e = email.lower().strip()
    for p in [r'guest-.*@booking\.com', r'.*@guest\.expedia\.com',
              r'.*@guest\.hotels\.com', r'.*@agoda\.com', r'.*@trip\.com']:
        if re.match(p, e): return ''
    return e

# ─── Variant expansion (KEY DEMO TRICK) ──────────────
# Each guest becomes 2-3 ER input rows representing different channels
# with realistic variations. This is what makes ER matching demonstrably useful.
variant_struct = ArrayType(StructType([
    StructField('suffix', StringType()),
    StructField('channel', StringType()),
    StructField('phone_format', StringType()),     # 'digits' | 'hyphen' | 'plus82'
    StructField('email_strategy', StringType()),   # 'normal' | 'relay' | 'empty'
    StructField('name_locale', StringType()),      # 'asis' | 'en' | 'ko'
]))

@udf(variant_struct)
def make_variants(seed):
    rng = random.Random(seed)
    n = rng.choice([2, 2, 3])
    channels = ['HOTEL_WEB', 'HOTEL_APP', 'HOTEL_OTA', 'WALK_IN', 'CALL_CENTER', 'CORPORATE']
    rng.shuffle(channels)
    out = []
    for i in range(n):
        ch = channels[i]
        phone_fmt = rng.choice(['digits', 'hyphen', 'plus82'])
        email_strat = (rng.choice(['relay', 'relay', 'normal']) if ch == 'HOTEL_OTA' else
                       rng.choice(['empty', 'empty', 'normal']) if ch == 'WALK_IN' else 'normal')
        name_loc = rng.choice(['asis', 'asis', 'asis', 'ko' if i == 1 else 'en'])
        out.append((str(i + 1), ch, phone_fmt, email_strat, name_loc))
    return out

@udf(StringType())
def reformat_phone(phone, fmt):
    if not phone: return None
    digits = re.sub(r'[^\d]', '', phone)
    if digits.startswith('82') and len(digits) > 10: digits = '0' + digits[2:]
    if fmt == 'digits': return digits
    if fmt == 'hyphen' and len(digits) == 11:
        return f'{digits[0:3]}-{digits[3:7]}-{digits[7:]}'
    if fmt == 'plus82' and digits.startswith('0'):
        return '+82 ' + digits[1:3] + '-' + digits[3:7] + '-' + digits[7:]
    return digits

# ─── Main ────────────────────────────────────────────
guests = glueContext.create_dynamic_frame.from_catalog(
    database=args['GLUE_DATABASE'], table_name=args['GUESTS_TABLE'],
).toDF()

expanded = guests.withColumn('_variants', make_variants(col('guest_id'))) \
    .withColumn('_v', explode(col('_variants'))).drop('_variants')

out = expanded.select(
    concat(lit('V-'), col('guest_id'), lit('-'), col('_v.suffix')).alias('variantid'),  # ⭐ ER 단위
    normalize_name(col('firstname')).alias('firstname'),
    normalize_name(col('lastname')).alias('lastname'),
    mask_relay_email(col('email')).alias('email'),
    reformat_phone(col('phone'), col('_v.phone_format')).alias('phone'),
    col('dateofbirth').cast('string').alias('dateofbirth'),
    when(col('loyaltynumber').isNull(), lit('')).otherwise(col('loyaltynumber')).alias('loyaltynumber'),
    when(col('street').isNull(), lit('')).otherwise(col('street')).alias('street'),
    when(col('city').isNull(), lit('')).otherwise(col('city')).alias('city'),
    when(col('state').isNull(), lit('')).otherwise(col('state')).alias('state'),
    when(col('postalcode').isNull(), lit('')).otherwise(col('postalcode')).alias('postalcode'),
    when(col('country').isNull(), lit('')).otherwise(col('country')).alias('country'),
    col('_v.channel').alias('sourcechannel'),
)

# 단일 CSV로 출력 (ER이 partition 인식 안 함)
out.coalesce(1).write.mode('overwrite').option('header', 'true') \
   .csv(args['TARGET_S3_PATH'])

job.commit()
```

### CDK — Crawler + ETL Job + Glue Table for ER input

```typescript
// lib/ingestion-stack.ts (요지)
const etlScript = new s3assets.Asset(this, 'BuildErInputScript', {
  path: 'backend/glue-scripts/build-er-input.py',
});

const etlJob = new glue.CfnJob(this, 'BuildErInput', {
  name: `${projectName}-build-er-input`,
  role: glueRole.roleArn,
  glueVersion: '4.0', numberOfWorkers: 2, workerType: 'G.1X',
  command: { name: 'glueetl', scriptLocation: etlScript.s3ObjectUrl, pythonVersion: '3' },
  defaultArguments: {
    '--JOB_NAME': `${projectName}-build-er-input`,
    '--TARGET_S3_PATH': `s3://${dataBucket.bucketName}/er-input/unified/`,
    '--GLUE_DATABASE': glueDb.databaseName,
    '--GUESTS_TABLE': 'src_<schema>_public_guests',
    '--enable-glue-datacatalog': 'true',
    '--TempDir': `s3://${dataBucket.bucketName}/glue-temp/`,
  },
});

// ER이 읽을 Glue Table을 정적 정의 (S3 prefix)
new glue.CfnTable(this, 'ErInputTable', {
  catalogId: cdk.Aws.ACCOUNT_ID,
  databaseName: glueDb.databaseName,
  tableInput: {
    name: `${projectName.replace(/-/g, '_')}_variants`,
    tableType: 'EXTERNAL_TABLE',
    parameters: { classification: 'csv', skipHeaderLineCount: '1' },
    storageDescriptor: {
      location: `s3://${dataBucket.bucketName}/er-input/unified/`,
      inputFormat: 'org.apache.hadoop.mapred.TextInputFormat',
      outputFormat: 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat',
      serdeInfo: {
        serializationLibrary: 'org.apache.hadoop.hive.serde2.OpenCSVSerde',
        parameters: { 'separatorChar': ',', 'quoteChar': '"' },
      },
      columns: [
        { name: 'variantid', type: 'string' },
        { name: 'firstname', type: 'string' },
        { name: 'lastname', type: 'string' },
        { name: 'email', type: 'string' },
        { name: 'phone', type: 'string' },
        { name: 'dateofbirth', type: 'string' },
        { name: 'loyaltynumber', type: 'string' },
        { name: 'street', type: 'string' },
        { name: 'city', type: 'string' },
        { name: 'state', type: 'string' },
        { name: 'postalcode', type: 'string' },
        { name: 'country', type: 'string' },
        { name: 'sourcechannel', type: 'string' },
      ],
    },
  },
});
```

### Lambda 트리거 (`/api/ingestion/build-er-input`)

ETL을 프론트의 IngestionPage 버튼 한번에 실행시키려면 ingestion handler에 `glue:StartJobRun` 호출 추가:

```typescript
// backend/lambdas/ingestion/handler.ts
if (path.endsWith('/build-er-input') && method === 'POST') {
  const r = await glue.send(new StartJobRunCommand({
    JobName: process.env.ETL_JOB_NAME!,
    Arguments: { '--SOURCE_REFRESH': new Date().toISOString() }, // bookmark force
  }));
  return ok({ jobRunId: r.JobRunId });
}
```

### ER input 스키마 — 도메인 무관 표준

ER이 가장 효율적으로 작동하려면 입력 스키마를 다음 12개 컬럼으로 정렬한다:

| 컬럼 | 의미 | match_key 가능 |
|---|---|---|
| `variantid` | ER 그룹핑 고유 ID (`V-{entity_id}-{n}`) | UNIQUE_ID (필수) |
| `firstname` / `lastname` | 정규화된 이름 | NAME_FIRST / NAME_LAST |
| `email` | lower + relay 마스킹 | EMAIL_ADDRESS |
| `phone` | 정규화된 번호 | PHONE_NUMBER |
| `dateofbirth` | YYYY-MM-DD | DATE |
| `loyaltynumber` | 멤버십 번호 | STRING |
| `street/city/state/postalcode/country` | 주소 5요소 | ADDRESS_* (group: FullAddress) |
| `sourcechannel` | 추적용 (매칭에는 미사용) | — |

