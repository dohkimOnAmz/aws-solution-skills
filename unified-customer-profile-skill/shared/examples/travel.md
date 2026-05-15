# Example: Travel & Hospitality (Golden Reference)

여행 산업(항공·호텔·여행사) 고객 통합의 완전한 참조 구현.
소스 코드: `demo-for-customer-profile-travel-hospitality/`

## 특징

- **3개 도메인**: Airline, Hotel, Agency — 각각 독립 CP Domain
- **Cross-Domain 통합**: Platform-level CP로 도메인 간 동일 고객 식별
- **Knowledge Graph**: Neptune + GraphRAG로 관계 분석 + AI 어시스턴트
- **Ecosystem CLV**: 도메인 간 고객 생애 가치 합산

## 채널 구성

| 도메인 | 채널 |
|--------|------|
| Airline | WEB, MOBILE, OTA, CORPORATE, CALL_CENTER |
| Hotel | HOTEL_WEB, HOTEL_APP, HOTEL_OTA, WALK_IN, CORPORATE |
| Agency | AGENCY_WEB, AGENCY_APP, B2B |

## PII 필드

```yaml
- variantid (UNIQUE_ID)
- firstname (NAME_FIRST) → match_key: Name, group: FullName
- lastname (NAME_LAST) → match_key: Name, group: FullName
- email (EMAIL_ADDRESS) → match_key: Email
- phone (PHONE_NUMBER) → match_key: Phone
- dateofbirth (DATE) → match_key: DateOfBirth
- passportnumber (STRING) → match_key: PassportNumber  # 항공 특화
- frequentflyernumber (STRING) → match_key: FFN  # 항공 특화
- loyaltynumber (STRING) → match_key: LoyaltyNumber
- sourcechannel (STRING)
```

## Object Types

### Airline Domain
- **Booking**: PNR, 구간, 클래스, 금액, 좌석
- **FFP** (Frequent Flyer Program): 티어, 마일리지, 상태
- **Ancillary**: 부가 서비스 구매 (수하물, 좌석 업그레이드, 라운지)
- **ServiceCase**: CS 이력

### Hotel Domain
- **Reservation**: 체크인/아웃, 객실타입, ADR
- **Folio**: 부대시설 이용 (F&B, 스파, 미니바)
- **GuestPreferences**: 객실 선호, 베개, 온도
- **Loyalty**: 멤버십 티어, 포인트

### Agency Domain
- **Package**: 패키지 상품 예약 (항공+호텔 결합)
- **Inquiry**: 상담 이력
- **Preferences**: 여행 스타일, 선호 목적지

## ER 전략

3가지 타입 모두 사용:
1. **Simple**: FFN/LoyaltyNumber 정확 일치
2. **Advanced**: NameAndEmail, NameAndPhone (퍼지)
3. **ML**: 소스 간 변형이 큰 경우

## Calculated Attributes 예시

- TotalAirlineRevenueInAYear (Booking → SUM Amount)
- FlightCountInAYear (Booking → COUNT)
- TotalHotelNightsInAYear (Reservation → SUM NumberOfNights)
- AverageDailyRate (Reservation → AVG ADR)
- LastFlightDate (Booking → LAST_OCCURRENCE DepartureDate)
- EcosystemCLV (Cross-Domain 합산 — 별도 Lambda 계산)

## Cross-Domain Ecosystem CLV 계산

```typescript
interface EcosystemCLV {
  airlineCLV: number;   // 항공 1년 매출 × 잔존율
  hotelCLV: number;     // 호텔 1년 매출 × 잔존율
  agencyCLV: number;    // 여행사 1년 매출 × 잔존율
  ecosystemCLV: number; // 합산 + 시너지 계수
  synergyBonus: number; // 멀티도메인 사용 시 가중치
}

// 시너지 계수: 2개 도메인 = 1.2x, 3개 도메인 = 1.5x
```

## 핵심 설계 결정

| 결정 | 이유 |
|------|------|
| 도메인별 Connect Instance | 데이터 격리 + 독립 Object Type 관리 |
| Platform-level 별도 Domain | Cross-Domain 매칭 결과를 별도 저장 |
| 순차 Instance 생성 | Connect "pending" 상태 충돌 방지 |
| Neptune for Graph | 관계 깊이 3+ hop 쿼리에 최적 |
| GraphRAG (Bedrock + Cypher) | 자연어 → 그래프 쿼리 변환 |
