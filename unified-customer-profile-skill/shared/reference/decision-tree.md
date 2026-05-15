# Decision Tree — Feature & Strategy Selection

이 문서는 사용자 요구사항에 따라 어떤 기능/전략을 선택해야 하는지의 판단 로직.

## 1. Ingestion Mode 선택

```
Q: 고객 데이터가 현재 어디에 있는가?
│
├─ 기존 DB에 있음 (RDS, Aurora, Redshift, DynamoDB, on-prem DB)
│   └─ mode: glue_connection
│       └─ Glue Connection + Crawler → Glue Table → ER 입력
│       └─ JDBC (MySQL, PostgreSQL, Oracle, SQL Server) 또는 DynamoDB
│
├─ 파일로 추출 가능 (배치)
│   ├─ 데이터 사이즈 작음 (<100MB), 빈도 낮음
│   │   └─ mode: csv (default)
│   │       └─ S3 업로드 (CSV) + Lambda 처리 + Glue Table
│   │
│   └─ 데이터 사이즈 큼 (>100MB), 컬럼형 최적화 필요
│       └─ mode: parquet
│           └─ S3 업로드 (Parquet) + Glue Table 직접 참조 (변환 불필요)
│           └─ 장점: 압축률 높음, Athena/ER 쿼리 성능 좋음, 스키마 내장
│
├─ 실시간 이벤트 스트림
│   └─ mode: kinesis
│       └─ Kinesis Stream + EventBridge Pipe + CP 직접 적재
│
└─ 하이브리드 (배치 + 실시간)
    └─ mode: hybrid
        └─ Glue Connection (초기 마이그레이션) + Kinesis (신규 이벤트)
```

**판단 기준**:
- 데모/PoC, 소량 → CSV (단순, 빠른 셋업)
- 대용량 배치, 분석 병행 → Parquet (Athena 쿼리 가능, 압축 효율)
- 기존 DB 연동, 주기적 동기화 → Glue Connection + Crawler
- 프로덕션 실시간 → Kinesis
- 초기 마이그레이션 + 이후 실시간 → Hybrid (Glue + Kinesis)

## 2. Entity Resolution 매칭 전략

### 핵심 원칙

1. **3가지 매칭은 항상 전부 포함**: Simple Rule + Advanced Rule + ML Matching
   사용자에게 "어떤 걸 쓸지" 묻지 않는다. 전부 실행 가능하게 만든다.

2. **비교 대시보드**: 3가지 매칭 결과를 나란히 비교하는 페이지 필수 포함
   - 각 타입별 매칭 그룹 수, 미매칭 수, 의심 그룹 수
   - Precision/Recall 비교 차트
   - 샘플 매칭 쌍 비교 (같은 데이터에 대해 각 타입이 어떻게 판정했는지)

3. **Bedrock 자동 생성**: AI가 데이터를 보고 규칙을 만들어주고, HITL로 검증

```

### 매칭 타입 자동 결정 (AI가 판단)

```
Q: 데이터에 고유 식별자가 있는가? (Bedrock이 데이터 프로파일링 후 판단)
│
├─ 고유 식별자 존재 (loyaltyNumber, membershipId 등 uniqueRate > 0.95)
│   └─ Bedrock이 Simple Rule 1개 자동 생성 (정확 일치)
│       + 추가로 Advanced Rule도 생성 (고유 ID 없는 레코드용)
│
├─ 고유 식별자 없음, PII 필드 품질 양호 (nullRate < 0.3)
│   └─ Bedrock이 Advanced Rule 2~4개 자동 생성
│       - 필드 조합: 데이터 프로파일 기반 최적 조합
│       - 한국어 감지 시: ETL 정규화 단계 추가 권장
│
└─ 데이터 품질 낮음 (nullRate > 0.5, 변형 패턴 다수)
    └─ Bedrock이 판단:
        ├─ ETL 정규화 후 재분석 권장 (먼저 데이터 품질 개선)
        └─ 또는 ML Matching 보조 사용 제안 (리전 가용성 확인 필수)
```

### HITL 검증 필수 포인트

Bedrock이 규칙을 생성한 후, 반드시 사용자가 확인해야 하는 것:
1. **매칭 쌍 미리보기**: "이 두 레코드가 동일 고객으로 판정됩니다" — 맞나요?
2. **예상 precision/recall**: 규칙별 기대 정확도 표시
3. **주의사항**: false positive 위험이 높은 규칙 강조
4. **테스트 실행**: 소규모 데이터로 실제 매칭 → 결과 확인 → 피드백 → 개선 루프

### Advanced Rule 조합 전략

| 규칙 이름 | Match Keys | 신뢰도 | 사용 조건 |
|-----------|-----------|--------|-----------|
| NameAndEmail | Name + Email | 높음 | 이메일이 있는 경우 |
| NameAndPhone | Name + Phone | 높음 | 전화번호가 있는 경우 |
| NameAndAddress | Name + Address | 중간 | 주소 데이터가 정형화된 경우 |
| EmailOnly | Email | 중간 | 1인 1이메일 보장 시 |
| PhoneOnly | Phone | 중간 | 전화번호 고유성 보장 시 |
| LoyaltyNumber | LoyaltyNumber | 최고 | 멤버십 시스템 있을 때 |

## 3. Knowledge Graph 필요 여부

```
Q: 고객 간 관계 분석이 필요한가?
├─ YES → graph.enabled: true
│   │
│   ├─ 가족/동행자 관계 파악 필요
│   ├─ 법인 고객-개인 연결
│   ├─ 영향력/네트워크 분석
│   └─ AI 어시스턴트 (GraphRAG) 필요
│   │
│   └─ ⚠️ 비용 경고: Neptune db.r5.large + NAT = ~$300/month 최소
│
└─ NO → graph.enabled: false (default)
    └─ 개별 고객 프로필만으로 충분한 경우
```

## 4. Cross-Domain 필요 여부

```
Q: 서로 다른 사업부/브랜드 간 고객 통합이 필요한가?
├─ YES → Cross-Domain 스택 추가
│   │
│   ├─ 여러 Connect Instance (도메인별)
│   ├─ Platform-level CP Domain (통합 뷰)
│   └─ Cross-Domain ER Workflow
│   │
│   └─ ⚠️ Connect Instance quota: 기본 2개 → 할당량 증가 필요
│
└─ NO → 단일 도메인 (default)
    └─ 하나의 CP Domain으로 모든 채널 통합
```

## 5. 프론트엔드 페이지 선택

| 기본 (항상) | 매칭 관련 | 보강/통합 | 고급 |
|------------|----------|----------|------|
| Dashboard | Matching Results | Profile Enrichment | Cross-Domain View |
| Data Ingestion | Matching Dashboard | Unified Profile View | Graph Builder |
| | AI Rule Improvement | | GraphRAG Chat |
| | Rule History | | Ecosystem View |

**선택 로직**:
- Core pages (항상): Dashboard, Ingestion, Matching, AI Rules, Profile View
- Cross-Domain pages: Cross-Domain View, Ecosystem View → 멀티도메인일 때만
- Graph pages: Graph Builder, GraphRAG → graph.enabled일 때만
- Domain-specific: 산업별 맞춤 시각화 (여정 맵, 구매 퍼널 등)

## 6. 비용 티어

| 티어 | 구성 | 예상 월비용 |
|------|------|------------|
| Minimal | CP + ER (Rule) + CSV + Cognito | ~$50-100 |
| Standard | + Kinesis + ML Matching | ~$100-200 |
| Full | + Neptune + Cross-Domain | ~$400-600 |

*비용은 프로필 수, 매칭 빈도, 리전에 따라 변동*
