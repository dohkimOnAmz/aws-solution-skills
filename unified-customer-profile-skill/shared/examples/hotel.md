# Example: Hotel & Hospitality

호텔/리조트 산업 고객 통합 참조.

## 산업 특성

- 동일 게스트가 OTA, 직접예약, 법인 등 다양한 경로로 예약
- 이름/이메일 변형이 특히 많음 (OTA relay email: guest-abc123@booking.com)
- 로열티 프로그램이 강력한 식별자
- 객실 선호도/식이제한 등 개인화 데이터가 핵심 가치

## 채널

```yaml
channels:
  - HOTEL_WEB       # 호텔 공식 웹사이트
  - HOTEL_APP       # 호텔 공식 앱
  - HOTEL_OTA       # OTA (Booking.com, Expedia 등)
  - WALK_IN         # 현장 직접 체크인
  - CALL_CENTER     # 전화 예약
  - CORPORATE       # 법인 계약 예약
```

## PII 필드

```yaml
pii_fields:
  - name: variantid
    type: UNIQUE_ID
    required: true
  - name: firstname
    type: NAME_FIRST
    match_key: Name
    group: FullName
    required: true
  - name: lastname
    type: NAME_LAST
    match_key: Name
    group: FullName
    required: true
  - name: email
    type: EMAIL_ADDRESS
    match_key: Email
  - name: phone
    type: PHONE_NUMBER
    match_key: Phone
  - name: dateofbirth
    type: DATE
    match_key: DateOfBirth
  - name: loyaltynumber
    type: STRING
    match_key: LoyaltyNumber
  - name: street / city / state / postalcode / country
    type: ADDRESS_*
    match_key: Address
    group: FullAddress
  - name: sourcechannel
    type: STRING
```

## Object Types

### Reservation
호텔 예약 + 체류 기록
- ReservationId, GuestProfileId, PropertyCode, PropertyName
- CheckInDate, CheckOutDate, RoomType, RateCode
- AverageDailyRate, TotalAmount, NumberOfNights
- BookingChannel, Status

### Folio
체류 기간 부대시설 이용 내역
- FolioId, ReservationId, ItemType (ROOM/FNB/SPA/MINIBAR/LAUNDRY)
- Description, Amount, Currency, ChargeDate

### GuestPreferences
게스트 개인화 데이터
- RoomPreference (high floor, ocean view, quiet)
- PillowType, TemperaturePreference, AmenityPreferences
- DietaryRestrictions, SmokingPreference

### Loyalty
멤버십 프로그램
- LoyaltyId, MembershipTier (Silver/Gold/Platinum/Diamond)
- PointsBalance, LifetimeNights, LifetimeSpend, EnrollDate

## Calculated Attributes

| 이름 | 집계 | 기간 |
|------|------|------|
| TotalHotelRevenueInAYear | SUM(TotalAmount) | 365일 |
| TotalNightsInAYear | SUM(NumberOfNights) | 365일 |
| AverageDailyRate | AVG(AverageDailyRate) | 365일 |
| TotalFolioSpendInAYear | SUM(Folio.Amount) | 365일 |
| MostRecentStayDate | LAST_OCCURRENCE(CheckInDate) | — |

## ER 전략 (호텔 특화)

**주요 도전**:
- OTA relay email (guest-xxx@booking.com) → Email 단독 매칭 불가
- 법인 예약 시 비서가 대리 예약 → 이름이 다를 수 있음
- Walk-in 시 최소 정보만 수집

**권장 규칙 조합**:
1. `LoyaltyNumber` — 최고 신뢰도 (있으면 확정)
2. `NameAndPhone` — OTA relay email 우회
3. `NameAndEmail` — 직접 예약 시 강력
4. `NameAndDOB` — 전화/이메일 없을 때 보조

**OTA relay email 전처리**:
```typescript
function isRelayEmail(email: string): boolean {
  return /guest-.*@(booking|expedia|hotels)\.com/i.test(email);
}
// relay email은 매칭에서 제외하고 phone/name으로 폴백
```

## 프론트엔드 특화

- **체류 타임라인**: 체크인/아웃 히스토리를 시각적 타임라인으로
- **선호도 카드**: 객실/식이/온도 선호를 한눈에
- **Revenue per Guest**: 객실비 + 부대시설 합산
- **Property 분포**: 어떤 호텔을 주로 이용하는지 파이차트
