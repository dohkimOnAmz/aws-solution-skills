# Eval: Hotel Scenario

## Input Prompt
```
호텔 체인 고객 통합 시스템을 만들어줘.
- 6개 채널: 웹, 앱, OTA, 워크인, 콜센터, 법인
- 예약/폴리오/선호도/로열티 데이터 관리
- OTA relay email 문제 해결
- Graph는 불필요
- 리전: ap-northeast-2
```

## Expected Output Checklist

### 아키텍처 결정
- [ ] 단일 CP Domain (멀티도메인 아님)
- [ ] Graph: disabled
- [ ] CSV ingestion (default)
- [ ] 비용 ~$50-100/month 범위

### 생성 파일
- [ ] `config/schema.yaml` — hotel 산업 맞춤
- [ ] `lib/main-stack.ts` — Graph/CrossDomain 없음
- [ ] 핵심 7개 스택만 (Foundation, Storage, Profiles, Matching, Ingestion, Auth, API)
- [ ] `backend/lambdas/` — graph-rag, graph-sync 없음

### ER 전략 (호텔 특화)
- [ ] LoyaltyNumber 우선
- [ ] NameAndPhone (OTA relay email 우회)
- [ ] OTA relay email 감지 로직 포함
- [ ] NameAndEmail (직접 예약 채널)

### Object Types
- [ ] Reservation (예약)
- [ ] Folio (부대시설 이용)
- [ ] GuestPreferences (선호도)
- [ ] Loyalty (멤버십)

### Calculated Attributes
- [ ] TotalHotelRevenueInAYear
- [ ] TotalNightsInAYear
- [ ] AverageDailyRate
- [ ] TotalFolioSpendInAYear
- [ ] MostRecentStayDate
