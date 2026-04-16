# Queries

TanStack Query v5 기반 **읽기 쿼리** 규칙. queryOptions factory, query key, 캐시, staleTime, prefetch, useQueries, infinite query.

> Cross-reference: TanStack Query BP (`qk-factory-pattern`, `cache-invalidation`, `perf-select-transform`, `cache-stale-time`, `cache-placeholder-vs-initial`, `pf-intent-prefetch`)

## 아키텍처 결정

- **별도 백엔드(Django) API 서버** — DB에서 Server Action으로 직접 가져오는 경우는 없음
- **SSR/CSR 혼용** 가능 — SSR prefetch는 필요하면 즉시 사용 가능하지만, 현재는 client-side TanStack Query + clientFetch 중심
- **mutation은 useApiMutation으로 통일** — Server Action 미사용 (mutations.md DF-07 참조)
- **queryOptions factory** — key + fn + staleTime을 `queries.ts`에서 통합 관리

---

## DF-01 queryOptions Factory (🚫 MUST)

**적용 위치**: `entities/*/queries.ts`

**규칙**: 모든 entity는 `queries.ts`에 key factory + queryOptions factory를 export한다. key, queryFn, staleTime을 한 곳에서 관리한다.

**Do**:

```ts
// entities/{entity}/queries.ts
import { queryOptions, keepPreviousData } from '@tanstack/react-query'
import { productApi } from './api'
import type { ProductListParams } from './types'

// Key factory — invalidation에 사용
export const productKeys = {
  all: ['products'] as const,
  lists: () => [...productKeys.all, 'list'] as const,
  list: (params: ProductListParams) => [...productKeys.lists(), params] as const,
  details: () => [...productKeys.all, 'detail'] as const,
  detail: (id: number) => [...productKeys.details(), id] as const,
}

// queryOptions factory — useQuery, prefetchQuery에 사용
export const productQueries = {
  list: (params: ProductListParams) => queryOptions({
    queryKey: productKeys.list(params),
    queryFn: () => productApi.getList(params),
    placeholderData: keepPreviousData,
    staleTime: 60 * 1000,
  }),
  detail: (id: number) => queryOptions({
    queryKey: productKeys.detail(id),
    queryFn: () => productApi.getDetail(id),
    staleTime: 5 * 60 * 1000,
  }),
}

// 사용
useQuery(productQueries.list(params))
queryClient.prefetchQuery(productQueries.list(params))
queryClient.invalidateQueries({ queryKey: productKeys.all })
```

**Don't**:

```ts
// ❌ 인라인 query key
useQuery({ queryKey: ['products', params], queryFn: ... })

// ❌ key와 fn을 별도 파일에서 분리 관리 (query-keys.ts + hooks.ts)
// → queries.ts 하나로 통합

// ❌ 불일치하는 키 이름
// file A: ['products'] / file B: ['product']
```

**Why**: queryOptions factory는 key + fn + staleTime을 한 곳에서 관리하여 DRY를 보장한다. prefetch에도 동일 객체를 사용할 수 있고, 타입 추론이 완벽하다. TanStack Query 공식 권장 패턴.

---

---

## DF-02 Query Key에 모든 의존성 포함 (🚫 MUST)

**규칙**: queryKey는 queryFn이 의존하는 모든 변수를 포함해야 한다.

**Do**:

```ts
export const useProductList = (params: ProductListParams) => {
  return useQuery({
    queryKey: productKeys.list(params), // { page, limit, search, sort } 전부 포함
    queryFn: () => productApi.getList(params),
  })
}
```

**Don't**:

```ts
// ❌ params를 key에서 누락 — 검색어가 바뀌어도 캐시가 안 바뀜
export const useProductList = (params: ProductListParams) => {
  return useQuery({
    queryKey: productKeys.all,  // params 누락!
    queryFn: () => productApi.getList(params),
  })
}
```

**Why**: queryKey가 의존성을 빠뜨리면, 해당 값이 변경되어도 TanStack Query가 refetch하지 않아 stale 데이터가 표시된다.

---

---

## DF-03 useQuery 훅 구조 (⚠️ SHOULD)

**규칙**: `useQuery(productQueries.list(params))`로 직접 사용한다. 커스텀 훅으로 래핑하지 않는다.

**Do**:

```ts
// widget 또는 feature에서 직접 사용
const { data, isLoading } = useQuery(productQueries.list(params))
const { data: product } = useQuery(productQueries.detail(id))

// 조건부 실행 — enabled 옵션 추가
const { data } = useQuery({
  ...productQueries.detail(id!),
  enabled: id !== null,
})
```

**Don't**:

```ts
// ❌ useQuery를 커스텀 훅으로 래핑 (queryOptions가 이미 모든 설정을 포함)
export const useProductList = (params) => useQuery({ queryKey: ..., queryFn: ..., ... })

// ❌ 인라인 queryKey + queryFn (DF-01 위반)
function ProductTable() {
  const { data } = useQuery({ queryKey: ['products', params], queryFn: () => fetch(...) })
}

// ❌ keepPreviousData 누락 (페이지 전환 시 깜박임)
useQuery({ queryKey: productKeys.list(params), queryFn: () => productApi.getList(params) })
```

**Why**: `keepPreviousData`는 페이지 전환 시 이전 데이터를 보여줘서 로딩 깜박임을 방지한다.

---

---

## DF-04 placeholderData vs initialData (⚠️ SHOULD)

> Cross-reference: TanStack `cache-placeholder-vs-initial`

**규칙**: `placeholderData`와 `initialData`의 차이를 이해하고 용도에 맞게 사용한다.

| 동작 | `initialData` | `placeholderData` |
|------|---------------|-------------------|
| 캐시에 저장 | O | X |
| staleTime 적용 | O | X (항상 fetch) |
| `isPlaceholderData` | `false` | `true` |
| 다른 컴포넌트에 공유 | O (캐시됨) | X |
| 용도 | SSR 데이터, 완전한 데이터 | 미리보기, 이전 페이지 |

**Do**:

```ts
// 목록 → 상세 전환 시 목록 캐시에서 미리보기
export const useProductDetail = (id: number) => {
  const queryClient = useQueryClient()

  return useQuery({
    queryKey: productKeys.detail(id),
    queryFn: () => productApi.getDetail(id),
    placeholderData: () => {
      const products = queryClient.getQueryData<PaginatedResponse<Product>>(
        productKeys.lists()
      )
      return products?.results.find(p => p.id === id)
    },
  })
}

// 페이지네이션 — 이전 페이지 데이터 유지
placeholderData: keepPreviousData
```

**Don't**:

```ts
// ❌ 불완전한 데이터를 initialData로 사용 (캐시 오염)
useQuery({
  queryKey: productKeys.detail(id),
  queryFn: () => productApi.getDetail(id),
  initialData: partialProductFromList,  // 필드가 누락된 데이터가 캐시에 저장됨
})
```

**Why**: 잘못된 선택은 캐시 오염(initialData) 또는 불필요한 refetch(placeholderData)를 유발한다.

---

---

## DF-05 select 옵션으로 데이터 변환 (⚠️ SHOULD)

> Cross-reference: TanStack `perf-select-transform`

**규칙**: 쿼리 데이터를 필터링, 정렬, 파생값 계산할 때는 `select` 옵션을 사용한다.

**Do**:

```ts
// select — 데이터가 변경될 때만 재실행 (memoization)
export const useActiveProducts = (params: ProductListParams) => {
  return useQuery({
    queryKey: productKeys.list(params),
    queryFn: () => productApi.getList(params),
    select: (data) => ({
      ...data,
      results: data.results.filter(p => p.is_active),
    }),
  })
}

// 외부 값에 의존하는 select — useCallback으로 안정화
const selectByCategory = useCallback(
  (data: PaginatedResponse<Product>) =>
    data.results.filter(p => p.category === category),
  [category]
)
```

**Don't**:

```ts
// ❌ 컴포넌트에서 매 렌더링마다 필터링
function ActiveProducts() {
  const { data } = useProductList(params)
  const active = data?.results.filter(p => p.is_active) ?? [] // 매 렌더마다 실행
}
```

**Why**: `select`는 structural sharing을 통해 데이터가 실제로 변경될 때만 재실행된다.

---

---

## DF-10 limit/offset 변환 (⚠️ SHOULD)

**규칙**: UI는 page/limit(1-indexed), API는 limit/offset(0-indexed). `toOffset()`, `toOrdering()` 유틸리티로 변환한다.

```ts
// shared/lib/table/table-utils.ts
export function toOffset(page: number, limit: number): number {
  return (page - 1) * limit
}

export function toOrdering(sorting: SortingState): string {
  return sorting.map((s) => (s.desc ? `-${s.id}` : s.id)).join(',')
}

export function fromOrdering(ordering: string): SortingState {
  if (!ordering) return []
  return ordering.split(',').map((s) => {
    const desc = s.startsWith('-')
    return { id: desc ? s.slice(1) : s, desc }
  })
}
```

**Why**: 변환 로직을 유틸리티로 추출하면 일관성을 보장하고, 오프바이원 에러를 방지한다.

---

---

## DF-11 staleTime 전략 (⚠️ SHOULD)

> Cross-reference: TanStack `cache-stale-time`

**규칙**: 데이터 변동성에 따라 staleTime을 차등 적용한다.

| 데이터 유형 | staleTime | 이유 |
|------------|-----------|------|
| 실시간 (모니터링, 알림) | 0 | 항상 최신이어야 함 |
| 사용자 생성 콘텐츠 (목록) | 60초 (기본) | 사용자 액션에 따라 변경 |
| 상세 정보 | 5분 | 변경 빈도 낮음 |
| 참조/설정 데이터 | 10-30분 | 거의 변경 안 됨 |
| 정적 데이터 | Infinity | 수동 무효화만 |

```ts
// 참조 데이터 — 자주 안 바뀜
useQuery({
  queryKey: configKeys.all,
  queryFn: configApi.getAll,
  staleTime: 10 * 60 * 1000, // 10분
})
```

**Why**: 적절한 staleTime은 불필요한 API 요청을 줄이면서 데이터 신선도를 유지한다.

---

---

## DF-12 Intent Prefetch (✅ MAY)

> Cross-reference: TanStack `pf-intent-prefetch`, Vercel `bundle-preload`

**규칙**: 사용자가 hover/focus할 때 다음 페이지 데이터를 미리 로드한다.

**Do**:

```tsx
function ProductRow({ product, onSelect }: { product: Product; onSelect: (p: Product) => void }) {
  const queryClient = useQueryClient()

  const handlePrefetch = () => {
    queryClient.prefetchQuery({
      queryKey: productKeys.detail(product.id),
      queryFn: () => productApi.getDetail(product.id),
      staleTime: 60 * 1000,
    })
  }

  return (
    <tr
      onMouseEnter={handlePrefetch}
      onClick={() => onSelect(product)}
    >
      <td>{product.name}</td>
    </tr>
  )
}
```

**적용 기준**:

| 트리거 | 용도 |
|--------|------|
| `onMouseEnter` | 데스크톱, 클릭 가능성 높은 링크 |
| `onFocus` | 키보드 탐색, 접근성 |
| 컴포넌트 mount | 다음 단계가 예측 가능한 위자드 |

**Why**: Prefetch는 체감 로딩 시간을 0에 가깝게 줄인다. 단, 모든 곳에 적용하지 말고 사용자가 실제로 이동할 가능성이 높은 경로에만.

---

---

## DF-14 useQueries — 동적 병렬 쿼리 (✅ MAY)

**규칙**: 동적 개수의 query를 병렬로 실행할 때 `useQueries`를 사용한다.

```ts
const queries = useQueries({
  queries: selectedIds.map(id => ({
    queryKey: productKeys.detail(id),
    queryFn: () => productApi.getDetail(id),
  })),
  combine: (results) => ({
    data: results.map(r => r.data).filter(Boolean),
    isPending: results.some(r => r.isPending),
  }),
})
```

**Don't**: `selectedIds.map(id => useQuery(...))` — hook을 루프 안에서 호출할 수 없음.

---

---

## DF-15 Infinite Query (✅ MAY)

**규칙**: 무한 스크롤이 필요할 때 `useInfiniteQuery`를 사용한다.

```ts
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
  queryKey: productKeys.infinite(filters),
  queryFn: ({ pageParam }) => productApi.getList({ page: pageParam, limit: 20 }),
  initialPageParam: 1,
  getNextPageParam: (lastPage, allPages) =>
    lastPage.results.length < 20 ? undefined : allPages.length + 1,
})

// 데이터 평탄화
const allProducts = data?.pages.flatMap(page => page.results) ?? []
```
