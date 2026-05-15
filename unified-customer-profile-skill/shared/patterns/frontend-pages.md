# Frontend Page Patterns

**Stack**: React 18 + Vite + TypeScript + Tailwind v3 + shadcn/ui + Radix primitives + lucide-react + recharts + sonner. **Cloudscape는 사용하지 않는다.**

## ⚠️ 필수: 백엔드 API 연결 (빠뜨리면 UI가 빈 화면)

프론트엔드는 반드시 배포된 API Gateway endpoint에 연결되어야 한다. **이 연결이 빠지면 페이지는 렌더링되지만 데이터가 안 나옴.**

### 1. CDK가 노출해야 할 Output

```typescript
// lib/api-stack.ts (또는 main-stack.ts) 하단에 반드시
new cdk.CfnOutput(this, 'ApiUrl', { value: api.url });
new cdk.CfnOutput(this, 'CognitoUserPoolId', { value: userPool.userPoolId });
new cdk.CfnOutput(this, 'CognitoClientId', { value: userPoolClient.userPoolClientId });
new cdk.CfnOutput(this, 'CognitoDomain', { value: `${projectName}-${cdk.Aws.ACCOUNT_ID}.auth.${cdk.Aws.REGION}.amazoncognito.com` });
new cdk.CfnOutput(this, 'Region', { value: cdk.Aws.REGION });
```

### 2. 환경 변수 자동 설정 (`scripts/update-frontend-env.sh`)

```bash
#!/bin/bash
STACK="${1:-${PROJECT_NAME:-app}-stack}"
REGION="${2:-${AWS_REGION:-us-east-1}}"
get() { aws cloudformation describe-stacks --stack-name "$STACK" --region "$REGION" \
  --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }

cat > frontend/.env.local <<EOF
VITE_API_URL=$(get ApiUrl)
VITE_COGNITO_USER_POOL_ID=$(get CognitoUserPoolId)
VITE_COGNITO_CLIENT_ID=$(get CognitoClientId)
VITE_COGNITO_DOMAIN=$(get CognitoDomain)
VITE_REGION=$REGION
EOF
echo "✅ frontend/.env.local updated"
```

`scripts/deploy.sh` 마지막에 이 스크립트를 호출하여 자동화.

## 프로젝트 구조

```
frontend/
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── postcss.config.js
├── components.json           ← shadcn registry
├── package.json
├── src/
│   ├── main.tsx
│   ├── App.tsx               ← Router + AuthProvider
│   ├── index.css             ← Tailwind directives + CSS vars
│   ├── lib/utils.ts          ← `cn()` (twMerge + clsx)
│   ├── api/client.ts         ← apiCall + auto-refresh
│   ├── api/auth.ts           ← OIDC userManager
│   ├── pages/                ← 라우트 컴포넌트
│   ├── components/
│   │   ├── ui/               ← shadcn 컴포넌트 (button, card, badge, alert, table, select, tabs, dialog, skeleton, ...)
│   │   ├── Layout.tsx
│   │   ├── PageHeader.tsx
│   │   └── StatCard.tsx
│   └── hooks/
```

## package.json 핵심 의존성

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-dropdown-menu": "^2.0.6",
    "@radix-ui/react-label": "^2.0.2",
    "@radix-ui/react-select": "^2.0.0",
    "@radix-ui/react-slot": "^1.0.2",
    "@radix-ui/react-tabs": "^1.0.4",
    "@radix-ui/react-tooltip": "^1.0.7",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0",
    "tailwindcss-animate": "^1.0.7",
    "lucide-react": "^0.469.0",
    "recharts": "^2.10.4",
    "sonner": "^1.4.0",
    "oidc-client-ts": "^3.0.1",
    "react-oidc-context": "^3.1.1"
  },
  "devDependencies": {
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.16",
    "postcss": "^8.4.32",
    "@vitejs/plugin-react": "^4.2.1",
    "vite": "^5.0.0",
    "typescript": "^5.3.0"
  }
}
```

## tailwind.config.ts

```typescript
import type { Config } from 'tailwindcss';
export default {
  darkMode: 'class',
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        // shadcn 표준 CSS variables 그대로 사용
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: { DEFAULT: 'hsl(var(--primary))', foreground: 'hsl(var(--primary-foreground))' },
        muted: { DEFAULT: 'hsl(var(--muted))', foreground: 'hsl(var(--muted-foreground))' },
        // ... destructive, accent, card, popover, border, input, ring
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
} satisfies Config;
```

## App.tsx — Router + Layout (shadcn)

```tsx
import { BrowserRouter, Routes, Route, Navigate, Link, useLocation } from 'react-router-dom';
import { AuthProvider, useAuth } from 'react-oidc-context';
import { Toaster } from 'sonner';
import { LayoutDashboard, Database, Network, GaugeCircle, Sparkles, Users2, GitGraph, Send } from 'lucide-react';
import Layout from './components/Layout';
import DashboardPage from './pages/DashboardPage';
// ... other pages

const NAV = [
  { to: '/', label: 'Dashboard', icon: LayoutDashboard },
  { to: '/ingestion', label: 'Data Ingestion', icon: Database },
  { to: '/matching', label: 'Entity Matching', icon: Network },
  { to: '/accuracy', label: 'Accuracy', icon: GaugeCircle },
  { to: '/ai-rules', label: 'AI Rules', icon: Sparkles },
  { to: '/profile-import', label: 'Send to CP', icon: Send },     // 5번 메모: CP 전송 화면
  { to: '/profiles', label: 'Unified Profile', icon: Users2 },
  { to: '/graph', label: 'Knowledge Graph', icon: GitGraph },     // optional
];

export default function App() {
  return (
    <AuthProvider {...oidcConfig}>
      <BrowserRouter>
        <Layout nav={NAV}>
          <Routes>
            <Route path="/" element={<DashboardPage />} />
            {/* ... */}
            <Route path="*" element={<Navigate to="/" />} />
          </Routes>
        </Layout>
        <Toaster position="top-right" richColors />
      </BrowserRouter>
    </AuthProvider>
  );
}
```

## Layout.tsx — sidebar + topbar (shadcn 패턴)

```tsx
import { Link, useLocation } from 'react-router-dom';
import { cn } from '../lib/utils';

export default function Layout({ nav, children }: { nav: NavItem[]; children: React.ReactNode }) {
  const { pathname } = useLocation();
  return (
    <div className="min-h-screen bg-background text-foreground">
      <aside className="fixed inset-y-0 left-0 w-60 border-r bg-card">
        <div className="px-6 py-5 border-b">
          <h1 className="font-semibold tracking-tight">Unified Customer Profile</h1>
        </div>
        <nav className="p-3 space-y-1">
          {nav.map(({ to, label, icon: Icon }) => (
            <Link key={to} to={to}
              className={cn(
                'flex items-center gap-3 rounded-md px-3 py-2 text-sm transition-colors',
                pathname === to ? 'bg-primary/10 text-primary font-medium' : 'text-muted-foreground hover:bg-muted hover:text-foreground'
              )}>
              <Icon className="h-4 w-4" />
              {label}
            </Link>
          ))}
        </nav>
      </aside>
      <main className="ml-60 p-6">{children}</main>
    </div>
  );
}
```

## API Client — Token 자동 첨부 + 401 재시도

```typescript
// src/api/client.ts
import { userManager } from './auth';
const API_URL = import.meta.env.VITE_API_URL;

async function token(): Promise<string> {
  const u = await userManager.getUser();
  if (!u || u.expired) {
    const refreshed = await userManager.signinSilent().catch(() => null);
    return refreshed?.id_token ?? '';
  }
  return u.id_token ?? '';
}

export async function apiCall<T>(path: string, init: RequestInit = {}): Promise<T> {
  const doFetch = async (t: string) =>
    fetch(`${API_URL}${path}`, {
      ...init,
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${t}`, ...(init.headers ?? {}) },
    });
  let res = await doFetch(await token());
  if (res.status === 401) {
    // force one refresh + retry
    const refreshed = await userManager.signinSilent().catch(() => null);
    res = await doFetch(refreshed?.id_token ?? '');
  }
  if (!res.ok) {
    const err = await res.json().catch(() => ({ error: res.statusText }));
    throw new Error(err.error || `API ${res.status}`);
  }
  return res.json();
}
```

## 페이지 패턴 — shadcn 표준 컴포넌트만 사용

모든 페이지는 같은 시각적 골격을 따른다:

```tsx
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from '../components/ui/card';
import { Button } from '../components/ui/button';
import { Badge } from '../components/ui/badge';
import { Alert, AlertDescription, AlertTitle } from '../components/ui/alert';
import { Skeleton } from '../components/ui/skeleton';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '../components/ui/table';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '../components/ui/select';
import { Tabs, TabsList, TabsTrigger, TabsContent } from '../components/ui/tabs';
import PageHeader from '../components/PageHeader';
```

## DashboardPage — StatCard + Recharts

```tsx
export default function DashboardPage() {
  const [stats, setStats] = useState<any>(null);
  useEffect(() => { apiCall('/api/dashboard-summary').then(setStats); }, []);

  return (
    <>
      <PageHeader title="Dashboard" description="고객 통합 프로필 현황" />
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        <StatCard label="Total Profiles" value={stats?.totalProfiles} icon={Users2} />
        <StatCard label="Matched Groups" value={stats?.matchedGroups} icon={Network} />
        <StatCard label="Match Rate" value={`${stats?.matchRate ?? 0}%`} icon={GaugeCircle} />
        <StatCard label="Data Sources" value={stats?.dataSources} icon={Database} />
      </div>
      <Card className="mt-6">
        <CardHeader><CardTitle>Daily Trend</CardTitle></CardHeader>
        <CardContent>
          <ResponsiveContainer width="100%" height={300}>
            <LineChart data={stats?.dailyTrend ?? []}>
              <XAxis dataKey="date" /><YAxis /><Tooltip />
              <Line dataKey="profiles" stroke="hsl(var(--primary))" />
            </LineChart>
          </ResponsiveContainer>
        </CardContent>
      </Card>
    </>
  );
}
```

## IngestionPage — 데이터 수집 (3가지 mode)

```tsx
export default function IngestionPage() {
  const [mode, setMode] = useState<'csv'|'glue_connection'|'kinesis'>('glue_connection');
  const [running, setRunning] = useState(false);

  return (
    <>
      <PageHeader title="Data Ingestion" />
      <Tabs value={mode} onValueChange={v => setMode(v as any)}>
        <TabsList>
          <TabsTrigger value="csv">CSV Upload</TabsTrigger>
          <TabsTrigger value="glue_connection">Glue Connection (DB)</TabsTrigger>
          <TabsTrigger value="kinesis">Kinesis Stream</TabsTrigger>
        </TabsList>
        <TabsContent value="csv">
          {/* file picker → POST /api/ingestion/upload-csv */}
        </TabsContent>
        <TabsContent value="glue_connection">
          <Button onClick={async () => {
            setRunning(true);
            await apiCall('/api/ingestion/crawl', { method: 'POST' });
            toast.success('Crawler started');
            setRunning(false);
          }} disabled={running}>Start Crawler</Button>

          {/* ★ ETL pipeline 트리거: raw → ER input */}
          <Button variant="secondary" onClick={async () => {
            await apiCall('/api/ingestion/build-er-input', { method: 'POST' });
            toast.success('ETL job started — converting raw → ER input');
          }}>Run ETL (Raw → ER Input)</Button>
        </TabsContent>
      </Tabs>
    </>
  );
}
```

## MatchingComparisonPage — 3가지 동시 실행 + 비교

```tsx
import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, Legend } from 'recharts';

export default function MatchingComparisonPage() {
  const [results, setResults] = useState<any[]>([]);
  const [running, setRunning] = useState(false);

  async function runAll() {
    setRunning(true);
    try {
      const [s, a, m] = await Promise.all([
        apiCall('/api/matching/run', { method: 'POST', body: JSON.stringify({ matchingType: 'simple' }) }),
        apiCall('/api/matching/run', { method: 'POST', body: JSON.stringify({ matchingType: 'advanced' }) }),
        apiCall('/api/matching/run', { method: 'POST', body: JSON.stringify({ matchingType: 'ml' }) }),
      ]);
      setResults([s, a, m]);
      toast.success('3가지 매칭 완료');
    } finally { setRunning(false); }
  }

  return (
    <>
      <PageHeader title="Matching Comparison"
        actions={<Button onClick={runAll} disabled={running}>3가지 매칭 모두 실행</Button>} />
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-4">
        {results.map(r => (
          <Card key={r.type}>
            <CardHeader>
              <Badge variant={r.type === 'ml' ? 'destructive' : 'secondary'}>{r.label}</Badge>
              <CardTitle>{r.matchedGroups} groups</CardTitle>
            </CardHeader>
            <CardContent className="text-sm space-y-1">
              <p>미매칭: <b>{r.unmatchedProfiles}</b></p>
              <p>Precision: <b>{(r.precision*100).toFixed(1)}%</b></p>
              <p>Recall: <b>{(r.recall*100).toFixed(1)}%</b></p>
              <p className="text-muted-foreground">{r.executionTime} · {r.cost}</p>
            </CardContent>
          </Card>
        ))}
      </div>
    </>
  );
}
```

## ProfileImportPage — **CP로 전송하는 화면** (필수)

매칭 결과(golden record)를 Customer Profiles로 전송하는 두 단계 화면. **이 페이지는 반드시 포함된다** — 그렇지 않으면 ER 결과가 CP에 도달하지 못한다.

### 단계
1. **Step 1: Preview & Import GuestProfile** — 활성 매칭 타입(simple/advanced/ml)을 선택, preview 후 `POST /api/profile-import/run`
2. **Step 2: Import Reservation/Folio** — 1단계가 끝난 뒤 `POST /api/cp-data-import/run` (PostgreSQL → CP child instance)

```tsx
export default function ProfileImportPage() {
  const [matchingType, setMatchingType] = useState<'simple'|'advanced'|'ml'>('ml');
  const [preview, setPreview] = useState<any>(null);
  const [importResult, setImportResult] = useState<any>(null);
  const [cpDataStatus, setCpDataStatus] = useState<any>(null);

  return (
    <>
      <PageHeader title="Send to Customer Profiles"
        description="ER 매칭 결과 → CP GuestProfile, 그리고 PostgreSQL → CP Reservation/Folio" />

      {/* STEP 1 */}
      <Card>
        <CardHeader>
          <CardTitle>Step 1 · Golden Profile Import</CardTitle>
          <CardDescription>매칭 결과를 합쳐 CP 도메인의 GuestProfile로 전송</CardDescription>
        </CardHeader>
        <CardContent className="space-y-4">
          <Select value={matchingType} onValueChange={v => setMatchingType(v as any)}>
            <SelectTrigger className="w-64"><SelectValue /></SelectTrigger>
            <SelectContent>
              <SelectItem value="simple">Simple Rule</SelectItem>
              <SelectItem value="advanced">Advanced Rule</SelectItem>
              <SelectItem value="ml">ML Matching</SelectItem>
            </SelectContent>
          </Select>
          <div className="flex gap-2">
            <Button variant="secondary" onClick={async () => setPreview(
              await apiCall('/api/profile-import/preview', { method:'POST', body: JSON.stringify({ matchingType, limit: 8 }) })
            )}>Preview</Button>
            <Button onClick={async () => {
              if (!confirm(`${matchingType} 매칭 결과를 CP에 전송합니다. 계속할까요?`)) return;
              setImportResult(await apiCall('/api/profile-import/run', {
                method: 'POST',
                body: JSON.stringify({ matchingType, replaceExisting: true })
              }));
              toast.success('Golden profiles imported');
            }}>
              <Send className="h-4 w-4 mr-2" /> Send Golden Profiles to CP
            </Button>
          </div>
          {importResult && (
            <Alert>
              <CheckCircle2 className="h-4 w-4" />
              <AlertTitle>{importResult.importedCount} profiles imported</AlertTitle>
              <AlertDescription>{importResult.durationMs}ms · errors: {importResult.errorCount}</AlertDescription>
            </Alert>
          )}
        </CardContent>
      </Card>

      {/* STEP 2 — required for Calculated Attributes to populate */}
      <Card className="mt-6">
        <CardHeader>
          <CardTitle>Step 2 · Reservation / Folio (transactional data)</CardTitle>
          <CardDescription>PostgreSQL → CP child instances. Calculated Attribute 값은 이 데이터가 들어와야 채워집니다.</CardDescription>
        </CardHeader>
        <CardContent>
          <Button onClick={async () => {
            await apiCall('/api/cp-data-import/run', { method: 'POST' });
            toast.info('Background import started (3-10 min)');
          }}>Send Reservation/Folio to CP</Button>
          {cpDataStatus && (
            <p className="mt-3 text-sm text-muted-foreground">
              Last run: {cpDataStatus.reservationCount} reservations + {cpDataStatus.folioCount} folios
              ({cpDataStatus.unmatchedGuestIds} unmatched)
            </p>
          )}
        </CardContent>
      </Card>
    </>
  );
}
```

## ProfileViewPage — Calculated Attributes 표시

Calculated Attribute 값을 GuestProfile detail 페이지에 별도 섹션으로 보여준다. **단, Step 2 (Reservation/Folio import)가 끝나야 값이 채워진다는 안내를 함께 표시.**

```tsx
{calcAttrs && Object.keys(calcAttrs).length > 0 ? (
  <Card>
    <CardHeader>
      <CardTitle>Calculated Attributes</CardTitle>
      <CardDescription>
        Reservation/Folio instance 기반 집계값. 빈 값이면 Step 2 (Send to CP) 미실행 상태.
      </CardDescription>
    </CardHeader>
    <CardContent className="grid grid-cols-2 md:grid-cols-3 gap-3">
      {Object.entries(calcAttrs).map(([k, v]) => (
        <div key={k} className="rounded-md border p-3">
          <div className="text-xs text-muted-foreground">{ATTR_LABEL[k] ?? k}</div>
          <div className="text-lg font-semibold">{formatValue(v)}</div>
        </div>
      ))}
    </CardContent>
  </Card>
) : (
  <Alert>
    <AlertTriangle className="h-4 w-4" />
    <AlertTitle>Calculated Attribute 값 없음</AlertTitle>
    <AlertDescription>
      "Send to CP" 화면에서 Step 2 (Reservation/Folio import)를 실행한 뒤
      CP가 인덱싱을 마칠 때까지 (수 분~수십 분) 기다리세요. 정의된 attribute의 Status가
      `COMPLETED`가 되면 값이 표시됩니다.
    </AlertDescription>
  </Alert>
)}
```

## AiRulesPage — HITL (shadcn Dialog)

Generate → 미리보기 → Approve/Reject 흐름. AI 모델 선택 셀렉터를 헤더에 둔다 (사용자가 비용/품질 trade-off를 보면서 모델을 바꿀 수 있게):

```tsx
<Select value={modelId} onValueChange={setModelId}>
  <SelectTrigger className="w-72"><SelectValue /></SelectTrigger>
  <SelectContent>
    <SelectItem value="us.anthropic.claude-opus-4-7">Claude Opus 4.7 (best, 1M ctx)</SelectItem>
    <SelectItem value="anthropic.claude-sonnet-4-20250514-v1:0">Claude Sonnet 4 (balanced)</SelectItem>
    <SelectItem value="us.anthropic.claude-opus-4-6">Claude Opus 4.6 (1M ctx)</SelectItem>
    <SelectItem value="anthropic.claude-haiku-4-5-20251001">Claude Haiku 4.5 (cheap)</SelectItem>
  </SelectContent>
</Select>
```

요청 본문에 `modelId`를 함께 보내고, Lambda는 그 값을 Bedrock InvokeModel에 전달.

## 도메인별 커스터마이징 포인트

| 산업 | 대시보드 특화 | 프로필 특화 | 추가 페이지 |
|------|-------------|-----------|-----------|
| 항공 | 노선별 매출, FFP 분포 | 여행 여정 타임라인 | 마일리지 대시보드 |
| 호텔 | 객실 점유율, ADR 트렌드 | 체류 히스토리 + Calculated Attributes | 로열티 |
| 소매 | GMV, 카테고리 분포 | 구매 퍼널, RFM | 장바구니 분석 |
| 금융 | AUM, 거래 빈도 | 상품 포트폴리오 | 리스크 스코어 |
| 통신 | ARPU, 이탈률 | 요금제 히스토리 | 네트워크 품질 |

## shadcn 컴포넌트 추가 방법

각 컴포넌트는 `components/ui/*.tsx`에 직접 생성한다. shadcn CLI가 있으면 `npx shadcn-ui@latest add card button badge alert table select tabs dialog skeleton` 한 번에 가능.

수동 생성이 필요한 경우 `https://ui.shadcn.com/docs/components/<name>`의 코드를 그대로 복사. (이 skill은 외부 fetch 없이도 hotel-1 프로젝트의 `frontend/src/components/ui/*` 를 참조 템플릿으로 쓸 수 있다.)
