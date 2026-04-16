# Next.js Patterns

Next.js 16 App Router 특화 규칙. Page 구조, Providers, 'use client' 배치, 특수 파일.

> Cross-reference: [Next.js App Router docs](https://nextjs.org/docs/app), [Claude Code Skills BP](https://platform.claude.com/docs/en/agents-and-tools/agent-skills)

---

## PS-04 Page는 Thin Shell (🚫 MUST)

**적용 위치**: `app/**/page.tsx`

**규칙**: `app/**/page.tsx`는 레이아웃 조합과 메타데이터만 포함한다. 비즈니스 로직, hook 호출, 복잡한 상태 관리를 금지.

**Do**:

```tsx
// app/(dashboard)/products/page.tsx
import { Suspense } from 'react'
import { PageHeader } from '@/shared/ui/page-header'
import { ProductTable } from '@/widgets/product-table'
import { Skeleton } from '@workspace/ui/components/skeleton'

export default function ProductsPage() {
  return (
    <div className="space-y-6">
      <PageHeader>상품 관리</PageHeader>
      <Suspense fallback={<Skeleton className="h-96 w-full" />}>
        <ProductTable />
      </Suspense>
    </div>
  )
}
```

**Don't**:

```tsx
// ❌ page.tsx에서 직접 hook 호출
export default function ProductsPage() {
  const { page, limit } = useTableState()
  const { data } = useProductList({ page, limit })
  return <DataTable data={data?.results ?? []} />
}
```

**Why**: Page를 thin shell로 유지하면 비즈니스 로직이 widget/feature에 캡슐화되어 재사용성과 테스트 용이성이 높아진다.

---

---

## PS-11 'use client' 배치 기준 (⚠️ SHOULD)

**규칙**: FSD 레이어별로 `'use client'` 지시어 배치 기준을 따른다.

| 위치 | 'use client' | 이유 |
|------|-------------|------|
| `app/page.tsx`, `app/layout.tsx` | **금지** | Server Component 유지 (PS-04) |
| `widgets/` | **항상** | hook 사용, 인터랙티브 |
| `features/ui/` | **항상** | hook 사용, 이벤트 핸들러 |
| `features/hooks/` | **항상** | React hook |
| `entities/hooks.ts` | **항상** | useQuery, useApiMutation |
| `entities/columns.tsx` | **항상** | JSX 렌더링 함수 |
| `entities/types.ts` | **금지** | 타입 전용, 서버 호환 |
| `entities/api.ts` | **금지** | 서버/클라이언트 양쪽에서 사용 가능 |
| `entities/schema.ts` | **금지** | 타입/스키마 전용 |
| `entities/query-keys.ts` | **금지** | 데이터 전용 |
| `shared/ui/` | **인터랙티브만** | useState/onClick 사용 컴포넌트만 |
| `shared/lib/` | **hook만** | use 접두사 파일만 |

**Do**:

```tsx
// widgets/product-table.tsx — 항상 'use client'
'use client'
export function ProductTable() { ... }

// entities/product/types.ts — 'use client' 없음
export type Product = { id: number; name: string; ... }

// entities/product/api.ts — 'use client' 없음
export const productApi = { getList: ... }
```

**Don't**:

```tsx
// ❌ page.tsx에 'use client' (PS-04 위반)
'use client'
export default function ProductsPage() { ... }

// ❌ types.ts에 'use client' (불필요)
'use client'
export type Product = { ... }
```

**Why**: `'use client'`가 없는 파일은 Server Component에서도 import 가능. types, api, schema를 서버 호환으로 유지하면 SSR prefetch(향후) 도입이 용이하다.

---

---

## PS-13 App Providers 패턴 (⚠️ SHOULD)

**적용 위치**: `app/providers.tsx`

**규칙**: 각 앱의 `app/providers.tsx`에 Provider 중첩 순서와 설정을 통일한다.

**Do**:

```tsx
// app/providers.tsx
'use client'

import { isServer, QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { toast } from 'sonner'
import { Toaster } from '@workspace/ui/components/sonner'
import { configureApi } from '@workspace/api/configure'
import { createQueryClient } from '@workspace/api/query/createQueryClient'

// 1. 모듈 레벨에서 configureApi 1회 호출
configureApi({ toast: { error: (msg) => toast.error(msg) } })

// 2. QueryClient: server=매번 생성, client=싱글톤
let browserQueryClient: QueryClient | undefined
const getQueryClient = (): QueryClient => {
  if (isServer) return createQueryClient()
  if (!browserQueryClient) browserQueryClient = createQueryClient()
  return browserQueryClient
}

// 3. Provider 중첩 순서: QueryClientProvider > (기타 Provider) > children + Toaster
export function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient()
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <Toaster position="bottom-right" />
    </QueryClientProvider>
  )
}
```

**체크리스트**:
- [ ] `configureApi()`는 Provider 외부 모듈 레벨에서 1회 호출
- [ ] QueryClient: server에서는 매번 새로 생성 (데이터 공유 방지)
- [ ] Toaster position: `bottom-right` 통일
- [ ] Provider 추가 시 QueryClientProvider 안쪽에 중첩

---

---

## PS-14 Next.js 특수 파일 (⚠️ SHOULD)

**적용 위치**: `app/error.tsx`, `app/not-found.tsx`, `app/loading.tsx`

**규칙**: 루트에 `error.tsx`, `not-found.tsx`, `loading.tsx`를 반드시 배치한다.

**Do**:

```
app/
├── error.tsx          # 라우트 에러 경계 (MUST be 'use client')
├── not-found.tsx      # 404 페이지
├── loading.tsx        # 라우트 전환 로딩 UI
├── layout.tsx
└── (dashboard)/
    └── ...
```

```tsx
// app/error.tsx
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen gap-4">
      <h2 className="text-xl font-semibold">문제가 발생했습니다</h2>
      <p className="text-muted-foreground">{error.message}</p>
      <Button onClick={reset}>다시 시도</Button>
    </div>
  )
}
```

```tsx
// app/not-found.tsx
export default function NotFound() {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen gap-4">
      <h2 className="text-xl font-semibold">페이지를 찾을 수 없습니다</h2>
      <p className="text-muted-foreground">요청한 페이지가 존재하지 않습니다.</p>
    </div>
  )
}
```

**Why**: 특수 파일이 없으면 에러/404 시 빈 화면이 표시되어 사용자 경험이 나빠진다.
