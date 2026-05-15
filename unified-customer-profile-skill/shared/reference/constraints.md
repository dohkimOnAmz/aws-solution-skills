# Constraints — Known Limitations & Gotchas

## 배포 시 수동 설정 필요 사항

1. **Connect Instance → Customer Profiles 활성화**
   - CDK로 Connect Instance + CP Domain 생성은 가능
   - 하지만 Instance와 Domain의 **연결(association)**은 Console에서 수동
   - `Profile creation: "Create inferred profiles only"` 선택
   - CDK Custom Resource로 자동화 가능 (associate-connect-domain.ts 참조)

2. **Connect → Data Source Integration (Kinesis mode)**
   - Kinesis → CP 연동은 Console에서 설정
   - EventBridge Pipe 생성 → CP integration에서 해당 Pipe 선택 → Object Type Mapping 지정

3. **ER ML Matching Training**
   - 첫 실행 시 학습 데이터 필요 (최소 ~100개 labeled pairs)
   - Rule-based는 즉시 사용 가능

## CDK 관련 제약

1. **Connect Instance 생성 순서**
   - 여러 인스턴스를 동시 생성하면 "pending" 상태에서 실패
   - 반드시 `addDependency()`로 직렬화
   ```typescript
   domainStack.addDependency(previousDomainStack);
   ```

2. **CP Domain Custom Resources**
   - Object Type 생성은 CDK 네이티브 미지원 → Custom Resource Lambda 필요
   - Calculated Attributes도 마찬가지
   - 참조: `custom-resources/upsert-object-type.ts`, `create-calculated-attributes.ts`

3. **ER Schema Mapping**
   - Glue Table의 컬럼명이 ER Schema Mapping의 InputSourceConfig와 정확히 일치해야 함
   - 대소문자 주의 (모두 lowercase 권장)

## Entity Resolution 제약

1. **비동기 실행**
   - ER Job은 비동기 배치. 실시간 매칭 아님.
   - StartMatchingJob → GetMatchingJob (polling) → 결과 S3에 저장
   - 데모에서는 Lambda가 polling + 결과 파싱

2. **Input 형식**
   - Glue Table 필수 (S3 직접 참조 불가)
   - 컬럼: `variantid`(UNIQUE_ID), PII 필드들, `sourcechannel`
   - 파티셔닝: 불필요 (ER이 전체 테이블 스캔)

3. **Rule 조합**
   - 하나의 Workflow에 여러 Rule 가능 (OR 관계)
   - Rule 내 Match Key는 AND 관계
   - 예: NameAndEmail = (Name AND Email match)

## Customer Profiles 제약

1. **PutProfileObject 호출 규칙**
   - Object Type과 정확히 일치하는 JSON 필요
   - Keys에 정의된 필드가 반드시 값이 있어야 함
   - `_profileObjectType` 헤더 필수

2. **SearchProfiles 한계**
   - Key 기반 검색만 가능 (free-text search 없음)
   - 프로필 목록 조회: ListProfileObjects 사용

3. **Calculated Attributes**
   - 기존 Object Type의 필드만 집계 가능
   - 생성 후 변경 불가 → 삭제 후 재생성
   - 20개 제한 (도메인당)

## 프론트엔드 제약

1. **Cognito Hosted UI Callback URL**
   - localhost:3000 (개발) + Amplify URL (프로덕션) 둘 다 등록 필요
   - CDK에서 미리 두 URL 등록하는 것 권장

2. **CORS**
   - API Gateway에서 CORS 설정 필수
   - Cognito token refresh 시 preflight 요청 고려

## 보안 고려사항

1. PII 데이터 → KMS 암호화 + S3 bucket policy
2. API → Cognito 토큰 필수 (public endpoint 없음)
3. Lambda → VPC 내 배치 시 NAT Gateway 비용 증가
4. Neptune → VPC 격리 필수 (public 접근 불가)
