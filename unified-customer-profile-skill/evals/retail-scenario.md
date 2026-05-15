# Eval: Retail Scenario

## Input Prompt
```
이커머스 고객 통합 시스템을 만들어줘.
- 5개 채널: 웹, 앱, POS, 콜센터, 마켓플레이스
- 주문/아이템/멤버십/선호도 데이터
- 게스트 구매(비회원) 처리 필요
- RFM 세그먼트 분석
- Graph 불필요, Kinesis 실시간 수집 필요
- 리전: ap-northeast-2
```

## Expected Output Checklist

### 아키텍처 결정
- [ ] 단일 CP Domain
- [ ] Graph: disabled
- [ ] Kinesis ingestion mode (실시간 요구)
- [ ] 비용 ~$100-200/month 범위

### 생성 파일
- [ ] `config/schema.yaml` — retail 맞춤
- [ ] `lib/ingestion-stack.ts` — Kinesis Stream + EventBridge Pipe 포함
- [ ] `frontend/src/pages/` — RFM 세그먼트 페이지 포함

### ER 전략 (소매 특화)
- [ ] EmailOnly 규칙 포함 (이커머스 핵심)
- [ ] LoyaltyNumber 규칙
- [ ] NameAndPhone (POS 대응)
- [ ] 게스트 구매 처리 로직 (이메일만 있을 때)
- [ ] 마켓플레이스 relay email 감지

### Object Types
- [ ] Order (주문 헤더)
- [ ] OrderLineItem (주문 상세)
- [ ] LoyaltyMembership (멤버십)
- [ ] Preferences (선호도)

### Calculated Attributes
- [ ] TotalSpendInAYear
- [ ] OrderCountInAYear
- [ ] AverageOrderValue
- [ ] MostRecentOrderDate
- [ ] LifetimeItemCount

### 프론트엔드 특화
- [ ] RFM 세그먼트 시각화
- [ ] 구매 카테고리 분석
- [ ] AOV 트렌드 차트
