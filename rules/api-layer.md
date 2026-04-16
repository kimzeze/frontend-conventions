# API Layer

API 함수 구조, 응답 타입, SSR prefetch 규칙. `entities/*/api.ts` 파일의 구조.

> Cross-reference: [Next.js App Router](https://nextjs.org/docs/app), TanStack Query BP (`ssr-dehydration`)

---

## DF-08 API 객체 구조 (🚫 MUST)

**적용 위치**: `entities/*/api.ts`

**규칙**: 각 entity의 `api.ts`는 메서드를 가진 객체를 export한다. 쿼리 파라미터는 `URLSearchParams`로 빌드한다. 메서드명은 표준을 따른다.

**표준 메서드명**:

| 메서드 | HTTP | 설명 |
|--------|------|------|
| `getList` | GET | 목록 조회 (paginated) |
| `getDetail` | GET | 단건 조회 |
| `create` | POST | 생성 |
| `update` | PATCH | 부분 수정 |
| `delete` | DELETE | 삭제 |

특수 엔드포인트는 동사+명사로: `bulkCreate`, `exportCsv`, `toggleActive` 등.

**Do**:

```ts
// entities/{entity}/api.ts
export const productApi = {
  getList: (params: ProductListParams) => {
    const query = new URLSearchParams()
    query.set('limit', String(params.limit))
    query.set('offset', String(params.offset))
    if (params.ordering) query.set('ordering', params.ordering)
    if (params.search) query.set('name_contains', params.search)
    return clientFetch<PaginatedResponse<Product>>(`/product/?${query}`)
  },

  getDetail: (id: number) =>
    clientFetch<Product>(`/product/${id}/`),

  create: (input: CreateProductInput) =>
    clientFetch<Product>('/product/', {
      method: 'POST',
      body: input,
    }),

  update: (id: number, input: UpdateProductInput) =>
    clientFetch<Product>(`/product/${id}/`, {
      method: 'PATCH',
      body: input,
    }),

  delete: (id: number) =>
    clientFetch<void>(`/product/${id}/`, { method: 'DELETE' }),
}
```

**Don't**:

```ts
// ❌ 템플릿 리터럴로 쿼리 빌딩 (인코딩 누락 위험)
clientFetch(`/product/?limit=${params.limit}&offset=${params.offset}`)

// ❌ 개별 함수로 export
export function getProductList() { ... }
export function createProduct() { ... }

// ❌ fetch 직접 사용 (토큰 주입 누락)
fetch('/api/product/').then(r => r.json())
```

**Why**: `URLSearchParams`는 URL 인코딩을 자동 처리. 객체 형태는 IDE 자동완성과 일관된 네이밍을 보장. `clientFetch`는 토큰 주입, 에러 변환, 타임아웃을 자동 처리.

---

---

## DF-09 PaginatedResponse 타입 (🚫 MUST)

**규칙**: 모든 페이지네이션 API 응답은 `PaginatedResponse<T>` 제네릭 타입을 사용한다.

```ts
type PaginatedResponse<T> = {
  count: number
  next: string | null
  previous: string | null
  results: T[]
}
```

**Why**: 백엔드 페이지네이션 응답 형식과 일치하며, 제네릭으로 모든 entity에 재사용 가능.

---

---

## DF-16 SSR Prefetch (✅ MAY)

**규칙**: Server Component에서 데이터를 prefetch하여 초기 로딩 시 즉시 표시.

```tsx
// page.tsx (Server Component)
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query'

export default async function ProductsPage() {
  const queryClient = new QueryClient()
  await queryClient.prefetchQuery(productQueries.list({ page: 1, limit: 10 }))

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ProductTable />
    </HydrationBoundary>
  )
}
```

> **주의**: SSR prefetch 사용 시 PS-04(thin shell)과의 조율 필요. prefetch 로직은 wrapper 컴포넌트로 분리하여 page.tsx의 thin shell을 유지한다.
