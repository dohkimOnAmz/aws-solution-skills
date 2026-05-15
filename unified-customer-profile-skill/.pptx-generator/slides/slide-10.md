# 슬라이드 10: 직접 구축의 함정 — 처음 만든 사람만 아는 14개 제약

---

## 원본 소스 참조

- 소스: shared/reference/aws-services.md, shared/reference/constraints.md
- 원본 내용:
  "Connect Instance quota (default) 2 per account · ER ML Matching regions us-east-1, us-west-2, eu-west-1, ap-southeast-1 등 — 반드시 확인! · Neptune 최소 인스턴스 db.r5.large (~$420/month) · Bedrock AP regions apac. 접두사! · OTA relay email guest-xxx@booking.com — 매칭 부적합"
  "Connect Instance ↔ CP Domain association은 Console에서 수동"

---

## 메타

- 유형: Content
- 레이아웃: LAYOUT.G3X2

---

## 레이아웃

```text
┌────────┬────────┬────────┐
│ 함정 1  │ 함정 2  │ 함정 3  │
├────────┼────────┼────────┤
│ 함정 4  │ 함정 5  │ 함정 6  │
└────────┴────────┴────────┘
```

배치전략: 6개 함정 카드 그리드 (3열 × 2행)

---

## 구현 내용

### 제목

직접 구축의 함정 : 14개 제약 중 대표 6개

#### TL — Connect Instance Quota

> 🔧 구현: svg

```text
┌────────────────────────┐
│  ⚠ Quota               │
│                        │
│  계정당 기본 2개        │
│  최대 4-5 (증가 신청)   │
│                        │
│  Cross-Domain의         │
│  첫 번째 벽             │
└────────────────────────┘
```

#### TM — ER ML Matching 리전 제약

```text
┌────────────────────────┐
│  ⚠ Region              │
│                        │
│  us-east-1, us-west-2, │
│  eu-west-1,            │
│  ap-southeast-1 등     │
│                        │
│  추천 전 반드시 확인     │
└────────────────────────┘
```

#### TR — Bedrock 모델 ID 접두사

```text
┌────────────────────────┐
│  ⚠ apac. Prefix        │
│                        │
│  AP 리전:               │
│  apac.anthropic.       │
│  claude-sonnet-4-...   │
│                        │
│  접두사 누락 시 호출 실패│
└────────────────────────┘
```

#### BL — Connect ↔ CP Domain 수동 연결

```text
┌────────────────────────┐
│  ⚠ Manual Setup        │
│                        │
│  CDK로 둘 다 생성 가능  │
│  연결(association)은    │
│  Console에서 수동        │
│                        │
│  CDK Custom Resource    │
│  로 자동화 가능          │
└────────────────────────┘
```

#### BM — OTA Relay Email

```text
┌────────────────────────┐
│  ⚠ Relay Email         │
│                        │
│  guest-xxx@           │
│  booking.com           │
│                        │
│  Email 단독 매칭 불가    │
│  → Phone/Name 폴백 필요 │
└────────────────────────┘
```

#### BR — Neptune 최소 비용

```text
┌────────────────────────┐
│  ⚠ Neptune Cost        │
│                        │
│  db.r5.large 최소       │
│  ~$420/month            │
│  + NAT Gateway 추가     │
│                        │
│  Graph 옵션 = +$300/월 │
└────────────────────────┘
```

> Skill이 이 14개 제약을 모두 *외워두고* 사용자 결정에 반영. 사용자는 함정을 모를 권리.
