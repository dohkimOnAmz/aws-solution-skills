# Eval: Travel & Hospitality Scenario

## Input Prompt
```
여행 산업(항공+호텔+여행사) 고객 통합 시스템을 만들어줘.
- 3개 도메인(항공, 호텔, 여행사) 각각 독립 프로필 관리
- 도메인 간 동일 고객 식별 (Cross-Domain)
- Knowledge Graph로 고객 관계 분석
- AI로 매칭 규칙 자동 개선
- 리전: ap-northeast-2
```

## Expected Output Checklist

### 아키텍처 결정
- [ ] 3개 Connect Instance + 3개 CP Domain (airline, hotel, agency)
- [ ] Platform-level CP Domain (cross-domain 통합)
- [ ] Neptune Graph (enabled)
- [ ] Kinesis or CSV 선택 제시
- [ ] 비용 ~$400-600/month 경고

### 생성 파일
- [ ] `lib/main-stack.ts` — 도메인별 스택 + Platform 스택 + Graph 스택
- [ ] `lib/domain-profiles-stack.ts` — 반복 생성 (airline, hotel, agency)
- [ ] `lib/cross-domain-matching-stack.ts`
- [ ] `lib/neptune-stack.ts`
- [ ] `backend/lambdas/matching/cross-domain-handler.ts`
- [ ] `backend/lambdas/graph-rag/handler.ts`
- [ ] `backend/lambdas/graph-sync/handler.ts`
- [ ] `frontend/src/pages/CrossDomainMatching.tsx`
- [ ] `frontend/src/pages/EcosystemView.tsx`
- [ ] `frontend/src/pages/GraphBuilder.tsx`

### 코드 품질
- [ ] `cdk synth` 통과 가능한 구조
- [ ] Connect Instance 순차 생성 (addDependency)
- [ ] Connect Instance quota 경고 (4개 필요)
- [ ] Bedrock model ID에 `apac.` 접두사
- [ ] KMS 암호화 적용
- [ ] DLQ 설정

### ER 전략
- [ ] Simple: FFN/LoyaltyNumber 정확 일치
- [ ] Advanced: NameAndEmail, NameAndPhone
- [ ] ML: 옵션으로 제시 (리전 확인 포함)
- [ ] Cross-Domain ER: 도메인 간 매칭
