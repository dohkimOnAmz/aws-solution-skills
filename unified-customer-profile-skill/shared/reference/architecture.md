# Architecture — Unified Customer Profile

## 시스템 개요

다채널 고객 데이터를 수집하고, Entity Resolution으로 동일 고객을 식별하며,
통합된 고객 프로필을 관리하는 시스템.

## 핵심 데이터 흐름

```
[채널 데이터] → [Ingestion] → [S3/Glue Table]
       ↓                                     ↑
[기존 DB (RDS/Aurora)] → [Glue Connection/Crawler]
                                     │
                              [ETL Transform]
                         (정규화/클렌징/품질필터)
                                     │
                                     ▼
                            [Entity Resolution]
                              ↙     ↓     ↘
                         Simple  Advanced   ML
                              ↘     ↓     ↙
                         [매칭 결과 (Golden Record)]
                                     ↓
                      [Customer Profiles Import]
                                     ↓
                         [360° Unified Profile]
                            ↙          ↘
                     [Calculated        [Knowledge
                      Attributes]        Graph (선택)]
```

## 레이어별 역할

### Layer 1: Foundation
- **KMS Key**: 모든 데이터 암호화의 단일 키
- **SQS DLQ**: 모든 비동기 실패의 중앙 큐
- **WHY**: 보안 기본값 + 운영 가시성을 인프라 레벨에서 보장

### Layer 2: Storage
- **S3 Bucket**: CSV 업로드, Glue 크롤링 대상, ER 출력 저장
- **WHY**: 모든 데이터 교환의 중간 저장소. Glue + ER이 S3 기반

### Layer 3: Profiles
- **Connect Instance**: Customer Profiles 기능의 컨테이너 (주의: 인스턴스 quota)
- **CP Domain**: 프로필 데이터의 논리적 분리 단위
- **Object Types**: 거래/이벤트 데이터의 스키마 정의
- **Calculated Attributes**: 집계 지표 자동 계산 (SUM, AVG, COUNT, MIN, MAX 등)
- **WHY**: AWS 관리형 서비스로 프로필 CRUD + 자동 집계 제공

### Layer 4: Matching
- **Glue Database/Table**: ER 입력 데이터의 스키마 정의
- **Entity Resolution Workflows**: 매칭 실행 단위
  - Simple: 정확 일치 (1개 match key)
  - Advanced: 퍼지 매칭 (복합 rule)
  - ML: 학습 기반 확률적 매칭
- **DynamoDB Tables**: 매칭 결과, 정확도 메트릭, AI 제안, 규칙 변경 이력
- **WHY**: ER은 stateless batch job → 결과를 DDB에 캐싱하여 API에서 즉시 조회

### Layer 5: Ingestion
- **CSV Mode**: S3 업로드 → Lambda 처리 → Glue Table 적재
- **Kinesis Mode**: 실시간 스트림 → EventBridge Pipe → CP 직접 적재
- **Parquet Mode**: S3 Parquet 업로드 → Glue Table 직접 참조 (변환 불필요, 압축·스키마 내장)
- **Glue Connection Mode**: 기존 DB(RDS, Aurora, DynamoDB) → JDBC Connection + Crawler → Glue Table 자동 생성
- **Hybrid Mode**: Glue Connection (초기 마이그레이션) + Kinesis (신규 이벤트)
- **WHY**: 고객 데이터 소스가 다양하므로 모든 입력 경로 지원. 데모는 CSV, 프로덕션은 Glue+Kinesis

### Layer 5.5: ETL Transform (선택)
- **Inline (Lambda)**: 소량·단순 변환 시 Lambda 핸들러 내에서 직접 처리
- **Glue Job (PySpark)**: 대용량·복잡한 변환 (한영 이름 정규화, 전화번호 표준화, 릴레이 이메일 처리)
- **Step Functions Pipeline**: Ingest → ETL → ER → CP Import 전체 오케스트레이션
- **WHY**: Raw 데이터 품질이 ER 정확도를 직접 좌우. 정규화 후 ER precision이 10~30% 향상 가능
- **Cognito User Pool**: 사용자 관리 + Hosted UI
- **Cognito Authorizer**: API Gateway에서 토큰 검증
- **WHY**: AWS 네이티브 OIDC. 추가 인프라 불필요

### Layer 6: Auth
- **Cognito User Pool**: 사용자 관리 + Hosted UI
- **Cognito Authorizer**: API Gateway에서 토큰 검증
- **WHY**: AWS 네이티브 OIDC. 추가 인프라 불필요

### Layer 7: API
- **API Gateway REST API**: 모든 백엔드 기능 노출
- **Lambda Handlers**: 각 도메인별 핸들러 (matching, profiles, accuracy, ai-agent, etc.)
- **WHY**: Serverless + 자동 스케일링 + Cognito 통합

### Layer 8: Graph (선택)
- **Neptune Cluster**: 그래프 DB (openCypher/Gremlin)
- **Graph Sync Lambda**: CP 프로필 → Neptune 노드/엣지 동기화
- **GraphRAG Lambda**: 자연어 질의 → Cypher 변환 → 결과 해석
- **WHY**: 관계 기반 인사이트 (인맥 네트워크, 영향력 분석). 비용 높으므로 선택적

### Layer 9: Cross-Domain (선택)
- **멀티 도메인 CP**: 산업/사업부별 독립 프로필 도메인
- **Platform CP**: 도메인 간 통합 뷰
- **Cross-Domain ER**: 서로 다른 도메인의 프로필을 매칭
- **WHY**: 항공+호텔+여행사 같은 생태계 통합. 복잡도 높으므로 선택적

## 설계 결정 (Design Decisions)

| 결정 | 선택 | 대안 | 이유 |
|------|------|------|------|
| IaC | CDK (TypeScript) | CloudFormation, Terraform | Type safety + CDK Custom Resources로 CP/ER 세부 설정 |
| 데이터 교환 | S3 | DynamoDB, RDS | Glue+ER의 네이티브 데이터 소스 |
| 매칭 결과 저장 | DynamoDB | S3+Athena | 밀리초 조회, API 직접 서빙 |
| 프론트엔드 | React+Cloudscape | Amplify Studio | AWS 스타일 일관성 + 커스터마이징 자유도 |
| AI | Bedrock Claude | SageMaker | Serverless, 별도 모델 관리 불필요 |
| Auth | Cognito | 없음/Custom | 관리형 + OIDC 표준 + API GW 네이티브 통합 |
| 배포 | CDK Deploy | CI/CD Pipeline | 데모 단순성 우선. 프로덕션은 Pipeline 추가 |

## CDK 스택 의존성

```
Foundation ─┐
            ├─→ Profiles ─┐
Storage ────┤             │
            ├─→ Matching  ├─→ API
            ├─→ Ingestion │
            └─→ Auth ─────┘
                          │
                [Optional]│
                Graph ────┘
```
