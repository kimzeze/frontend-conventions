# State Management

상태 관리 전략 규칙. 서버 상태 + URL 상태 + 로컬 상태로 운영한다.

---

## SM-01 상태 카테고리 분류 (🚫 MUST)

**규칙**: 상태를 세 가지 카테고리로 분류하고, 각 카테고리에 맞는 도구를 사용한다. 범용 전역 상태 라이브러리(zustand, Redux)는 사용하지 않는다.

| 카테고리 | 도구 | 예시 |
|----------|------|------|
| 서버 상태 | TanStack Query | 목록, 상세, CRUD 결과 |
| URL 상태 | nuqs | 검색어, 정렬, 페이지, 필터 |
| 로컬 UI 상태 | useState | 다이얼로그 open/close, 토글, 폼 입력 |

**Do**:

```ts
// 서버 상태 → TanStack Query
const { data } = useProductList(params)

// URL 상태 → nuqs
const [search, setSearch] = useQueryState('q', parseAsString.withDefault(''))

// 로컬 상태 → useState
const [isOpen, setIsOpen] = useState(false)
```

**Don't**:

```ts
// ❌ zustand로 서버 데이터 캐싱
const useProductStore = create((set) => ({
  products: [],
  fetchProducts: async () => { ... },
}))

// ❌ Context로 데이터 공유 (TanStack Query가 이미 캐시 공유)
const ProductContext = createContext<Product[] | null>(null)

// ❌ 전역 상태로 URL 상태 관리
const useSearchStore = create((set) => ({
  searchQuery: '',
  setSearchQuery: (q: string) => set({ searchQuery: q }),
}))
```

> **예외**: React Context는 theme, locale, auth session 등 **앱 설정**에는 사용 가능. 데이터 공유 목적으로는 사용 금지.

**Why**: TanStack Query가 캐싱/리페치/무효화를 자동 관리하고, nuqs가 URL 동기화를 처리한다. 이 두 가지가 90% 이상의 상태를 커버하므로 전역 상태 라이브러리는 복잡도만 증가시킨다.

---

## SM-02 URL 상태는 nuqs로 관리 (🚫 MUST)

**규칙**: 사용자에게 보이는 상태(검색, 정렬, 페이지, 필터)는 URL 쿼리 파라미터로 관리하며, nuqs를 사용한다.

**Do**:

```ts
import { parseAsInteger, parseAsString } from 'nuqs'

export const tableSearchParams = {
  q: parseAsString
    .withDefault('')
    .withOptions({ history: 'replace', shallow: true, clearOnDefault: true }),
  page: parseAsInteger
    .withDefault(1)
    .withOptions({ history: 'push', shallow: true, clearOnDefault: true }),
  limit: parseAsInteger
    .withDefault(10)
    .withOptions({ history: 'replace', shallow: true, clearOnDefault: true }),
  sort: parseAsString
    .withDefault('')
    .withOptions({ history: 'replace', shallow: true, clearOnDefault: true }),
}
```

**옵션 가이드**:

| 옵션 | 값 | 이유 |
|------|-----|------|
| `history` | `'push'` (page) | 페이지 변경은 뒤로가기 가능 |
| `history` | `'replace'` (나머지) | 검색/정렬 변경은 히스토리 쌓지 않음 |
| `shallow` | `true` | 서버 컴포넌트 리렌더 방지 |
| `clearOnDefault` | `true` | 기본값이면 URL에서 제거 (깨끗한 URL) |

**Don't**:

```ts
// ❌ useState로 검색/페이지 관리 (새로고침/공유 시 상태 유실)
const [search, setSearch] = useState('')
const [page, setPage] = useState(1)

// ❌ useSearchParams 직접 사용 (파싱 로직 반복)
const page = parseInt(searchParams.get('page') ?? '1')
```

**Why**: URL 상태는 새로고침, 북마크, 링크 공유 시 보존된다.

---

## SM-03 useTableState 훅 패턴 (⚠️ SHOULD)

**규칙**: 테이블의 URL 상태를 관리하는 공유 훅 `useTableState`를 사용한다.

**적용 위치**: `shared/lib/table/use-table-state.ts`

**Do**:

```ts
// shared/lib/table/use-table-state.ts — 핵심 기능
// - nuqs로 q, page, limit, sort 상태 관리
// - 검색 debounce (300ms)
// - TanStack Table SortingState ↔ API ordering 자동 변환
// - 검색/정렬 변경 시 page를 1로 자동 리셋

const {
  state: { q, page, setPage, limit, sort },
  debouncedQ,
  offset,
  sorting,
  handleSearchChange,
  handlePageSizeChange,
  handleSortingChange,
} = useTableState()
```

**Don't**:

```tsx
// ❌ widget에서 각 상태를 개별 관리 + debounce + sorting 변환 반복
function ProductTable() {
  const [q, setQ] = useQueryState('q', ...)
  const [page, setPage] = useQueryState('page', ...)
  // + debounce 로직 + sorting 변환 반복...
}
```

**Why**: 테이블 상태 관리 로직을 한 곳에서 관리하여 모든 테이블 페이지에서 일관된 동작을 보장한다.

---

## SM-04 Dialog 상태: useDialogState (⚠️ SHOULD)

**규칙**: create/edit 다이얼로그의 상태를 관리하는 공유 훅을 사용한다.

**Do**:

```ts
const dialog = useDialogState<Product>()

dialog.create.open()           // 등록 다이얼로그 열기
dialog.edit.open(product)      // 수정 다이얼로그 열기 (데이터 전달)
dialog.create.isOpen           // 열림 상태 확인
dialog.edit.selected           // 선택된 항목
```

**Don't**:

```tsx
// ❌ 여러 useState로 직접 관리
const [createOpen, setCreateOpen] = useState(false)
const [editOpen, setEditOpen] = useState(false)
const [selected, setSelected] = useState<Product | null>(null)
```

**Why**: 다이얼로그 상태 패턴은 모든 CRUD 페이지에서 반복된다. 공유 훅으로 보일러플레이트를 줄인다.

---

## SM-05 Lazy State Initialization (⚠️ SHOULD)

> Cross-reference: Vercel `rerender-lazy-state-init`

**규칙**: useState의 초기값이 비용이 큰 연산일 때, 값 대신 함수를 전달한다.

**Do**:

```ts
// 함수 전달 — 최초 렌더에서 한 번만 실행
const [data, setData] = useState(() => expensiveComputation(rawData))
const [items, setItems] = useState(() => JSON.parse(localStorage.getItem('items') ?? '[]'))
```

**Don't**:

```ts
// ❌ 값 전달 — 매 렌더마다 연산 실행 (결과는 버려짐)
const [data, setData] = useState(expensiveComputation(rawData))
const [items, setItems] = useState(JSON.parse(localStorage.getItem('items') ?? '[]'))
```

**Why**: 값으로 전달하면 매 렌더마다 초기값 연산이 실행된다(결과는 첫 번째만 사용). 함수로 전달하면 최초 렌더에서만 실행된다.

---

## SM-06 Functional setState (⚠️ SHOULD)

> Cross-reference: Vercel `rerender-functional-setstate`

**규칙**: 이전 state에 의존하는 setState는 함수형 업데이트를 사용한다. stale closure를 방지하고 stable callback을 만든다.

**Do**:

```ts
// 함수형 업데이트 — 항상 최신 state 참조
setItems(curr => [...curr, newItem])
setCount(prev => prev + 1)
setTodos(curr => curr.filter(t => t.id !== id))

// useCallback과 조합 — deps 불필요, stable reference
const addItem = useCallback((item: Item) => {
  setItems(curr => [...curr, item])
}, [])  // deps 비움 — stale closure 없음
```

**Don't**:

```ts
// ❌ 직접 참조 — stale closure 위험
setItems([...items, newItem])
setCount(count + 1)

// ❌ deps에 state 포함 — 매번 callback 재생성
const addItem = useCallback((item: Item) => {
  setItems([...items, item])
}, [items])  // items 변경마다 재생성
```

**적용 기준**:

| 상황 | 함수형? |
|------|--------|
| 이전 state 기반 업데이트 (`prev + 1`) | O |
| useCallback 내부에서 state 업데이트 | O |
| 정적 값 설정 (`setCount(0)`) | X — 직접 설정 OK |
| props/인자에서 값 설정 (`setName(name)`) | X — 직접 설정 OK |

**Why**: 함수형 업데이트는 closure가 생성된 시점이 아닌, 실행 시점의 최신 state를 보장한다.

---

## SM-07 useEffect 의존성 최적화 (⚠️ SHOULD)

> Cross-reference: Vercel `rerender-dependencies`

**규칙**: useEffect 의존성에 객체 대신 원시값(primitive)을 사용한다. 파생 상태는 미리 계산한다.

**Do**:

```ts
// 원시값 의존성 — id가 바뀔 때만 실행
useEffect(() => {
  fetchData(user.id)
}, [user.id])

// 파생 상태 미리 계산 — boolean 전환 시에만 실행
const isMobile = width < 768
useEffect(() => {
  if (isMobile) enableMobileMode()
}, [isMobile])
```

**Don't**:

```ts
// ❌ 객체 의존성 — user 참조 변경마다 실행
useEffect(() => {
  fetchData(user.id)
}, [user])

// ❌ 연속 값 의존성 — 매 픽셀마다 실행
useEffect(() => {
  if (width < 768) enableMobileMode()
}, [width])
```

**Why**: 객체는 참조 비교이므로, 값이 같아도 새 참조면 effect가 재실행된다. 원시값은 값 비교이므로 실제 변경 시에만 실행된다.

---

## SM-08 startTransition (✅ MAY)

> Cross-reference: Vercel `rerender-transitions`. 성능 이슈 발생 시 도입.

**규칙**: 빈번한 비긴급 state 업데이트에 `startTransition`을 사용하여 UI 반응성을 유지한다.

```ts
import { startTransition } from 'react'

// 스크롤/리사이즈 같은 빈번한 업데이트
const handler = () => {
  startTransition(() => setScrollY(window.scrollY))
}

// 필터 변경 같은 비긴급 업데이트
const handleFilterChange = (value: string) => {
  startTransition(() => setFilter(value))
}
```

**적용 시점**: 사용자 입력에 대한 즉각 반응이 필요하면서, 동시에 무거운 렌더가 발생하는 경우.

---

## SM-09 Deferred State Reads (✅ MAY)

> Cross-reference: Vercel `rerender-defer-reads`. 성능 이슈 발생 시 도입.

**규칙**: 콜백 안에서만 읽는 상태는 hook으로 구독하지 않고 필요 시점에 직접 읽는다.

```ts
// Do — 클릭 시에만 읽기 (구독 없음)
const handleShare = () => {
  const params = new URLSearchParams(window.location.search)
  const ref = params.get('ref')
  shareItem({ ref })
}

// Don't — searchParams 변경마다 리렌더
const searchParams = useSearchParams()
const handleShare = () => {
  const ref = searchParams.get('ref')
  shareItem({ ref })
}
```
