# Example: Retail & E-commerce

소매/이커머스 산업 고객 통합 참조.

## 산업 특성

- 온/오프라인 채널 통합이 핵심 (O2O)
- 게스트 구매 (비회원) 비율이 높음 → 식별 난이도 높음
- 이메일이 가장 강력한 식별자 (1인 1계정 패턴)
- 배송 주소가 보조 식별자로 유용
- RFM 세그멘테이션이 핵심 분석

## 채널

```yaml
channels:
  - WEB            # 공식 웹 쇼핑몰
  - MOBILE_APP     # 모바일 앱
  - POS            # 오프라인 매장 POS
  - CALL_CENTER    # 고객센터 전화 주문
  - MARKETPLACE    # 외부 마켓플레이스 (쿠팡, 네이버 등)
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
  - name: postalcode
    type: ADDRESS_POSTALCODE
    match_key: Address
    group: ShippingAddress
  - name: country
    type: ADDRESS_COUNTRY
    match_key: Address
    group: ShippingAddress
  - name: sourcechannel
    type: STRING
```

## Object Types

### Order
주문 헤더
- OrderId, CustomerId, OrderDate, OrderStatus
- TotalAmount, Currency, Channel, PaymentMethod
- ShippingCity, ShippingCountry

### OrderLineItem
주문 상세 (아이템별)
- LineItemId, OrderId, ProductId, ProductName
- Category, Quantity, UnitPrice, LineAmount

### LoyaltyMembership
멤버십 프로그램
- LoyaltyId, CustomerId, Tier (Bronze/Silver/Gold/VIP)
- PointsBalance, LifetimeSpend, EnrollDate

### Preferences
마케팅/상품 선호도
- PreferredCategories, PreferredBrands
- MarketingOptIn, PreferredLanguage

## Calculated Attributes

| 이름 | 집계 | 기간 |
|------|------|------|
| TotalSpendInAYear | SUM(Order.TotalAmount) | 365일 |
| OrderCountInAYear | COUNT(Order) | 365일 |
| AverageOrderValue | AVG(Order.TotalAmount) | 365일 |
| MostRecentOrderDate | LAST_OCCURRENCE(OrderDate) | — |
| LifetimeItemCount | SUM(OrderLineItem.Quantity) | 전체 |

## ER 전략 (소매 특화)

**주요 도전**:
- 게스트 구매: 이메일만 수집, 이름 없을 수 있음
- POS 현금 결제: 식별 정보 거의 없음
- 마켓플레이스: 실 이메일 대신 relay 주소 사용
- 가족이 같은 계정 공유

**권장 규칙 조합**:
1. `LoyaltyNumber` — 회원이면 확정
2. `EmailOnly` — 이커머스에서 가장 강력 (1인1메일 가정)
3. `NameAndPhone` — POS 멤버십 등록 시
4. `NameAndEmail` — 마켓플레이스 (relay email 제외 후)
5. `PhoneOnly` — 모바일 앱 인증 기반

**게스트 구매 처리**:
```typescript
// 이메일만 있고 이름이 없는 경우
if (!record.firstname && record.email) {
  // EmailOnly 규칙으로 매칭 시도
  // 나중에 회원가입하면 프로필 병합
}
```

**RFM 세그먼트 계산** (프론트엔드에서):
```typescript
interface RfmSegment {
  recency: number;    // days since last order
  frequency: number;  // orders per year
  monetary: number;   // total spend per year
  segment: 'Champion' | 'Loyal' | 'AtRisk' | 'Lost' | 'New';
}

function calculateRfm(profile: any): RfmSegment {
  const recency = daysSince(profile.MostRecentOrderDate);
  const frequency = profile.OrderCountInAYear;
  const monetary = profile.TotalSpendInAYear;

  // Simple scoring (1-5)
  const rScore = recency < 30 ? 5 : recency < 90 ? 4 : recency < 180 ? 3 : recency < 365 ? 2 : 1;
  const fScore = frequency > 12 ? 5 : frequency > 6 ? 4 : frequency > 3 ? 3 : frequency > 1 ? 2 : 1;
  const mScore = monetary > 1000000 ? 5 : monetary > 500000 ? 4 : monetary > 200000 ? 3 : monetary > 50000 ? 2 : 1;

  // Segment mapping
  if (rScore >= 4 && fScore >= 4) return { ...base, segment: 'Champion' };
  if (fScore >= 3) return { ...base, segment: 'Loyal' };
  if (rScore <= 2 && fScore >= 3) return { ...base, segment: 'AtRisk' };
  if (rScore <= 1) return { ...base, segment: 'Lost' };
  return { ...base, segment: 'New' };
}
```

## 프론트엔드 특화

- **구매 퍼널**: 장바구니 → 주문 → 배송 → 완료 시각화
- **RFM 세그먼트 맵**: 3D scatter 또는 heatmap
- **카테고리 선호도**: 구매 카테고리 트리맵
- **재구매율**: 코호트 분석 차트
- **AOV 트렌드**: 월별 평균 주문 금액 라인차트
