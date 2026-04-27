# State Management

상태 관리 전략 규칙. 4-카테고리 분류 + 각 도구별 BP.

---

## SM-01 상태 4-카테고리 분류 (🚫 MUST) — 최강 규칙

**규칙**: 상태는 4가지 카테고리로 나누고, 각 카테고리에 정해진 도구만 사용한다.

| 카테고리 | 도구 | 예시 |
|---------|------|------|
| **서버 상태** | TanStack Query | API 응답 (목록/상세/CRUD 결과) |
| **URL 상태** | nuqs | 검색어, 페이지, 정렬, 필터, 탭 |
| **로컬 UI 상태** | useState | 다이얼로그 open/close, 토글, 일시 입력 |
| **클라이언트 전역 상태** | Zustand | auth session, theme, locale, UI flags, wizard 진행상태 |

**Default**: 위 매트릭스. 카테고리가 모호하면 결정 트리:
1. 서버에서 오는 데이터인가? → **TanStack Query**
2. URL에 보여야 하는가 (북마크/공유/새로고침 보존)? → **nuqs**
3. 단일 컴포넌트만 쓰는가? → **useState**
4. 2개 이상 컴포넌트가 공유하는 비서버·비URL 데이터인가? → **Zustand**

**Override policy** (Q4-B): 카테고리 위반 요청 시 경고 후 진행:
> "SM-01에 따르면 [데이터 X]는 [카테고리 Y]로 [도구 Z]를 사용해야 합니다. 요청대로 [대안] 사용합니다."

**🚨 AI 실수 #2 차단용**: "임의로 Context/Zustand 도입"이 가장 흔한 실수 중 2위. 이 규칙은 그 실수를 카테고리 기반 결정 트리로 막는다.

**Do**:

```ts
// 서버 상태 → TanStack Query
const { data } = useQuery(productQueries.list(params))

// URL 상태 → nuqs
const [search, setSearch] = useQueryState('q', parseAsString.withDefault(''))

// 로컬 UI 상태 → useState
const [isOpen, setIsOpen] = useState(false)

// 클라이언트 전역 상태 → Zustand (custom hook export, SM-10 참조)
const user = useAuth()
```

**Don't**:

```ts
// ❌ 서버 데이터를 Zustand에 캐싱 (TanStack Query가 이미 함)
const useProductStore = create((set) => ({
  products: [],
  fetchProducts: async () => { ... },
}))

// ❌ Context로 데이터 공유 (TanStack Query가 이미 캐시 공유)
const ProductContext = createContext<Product[] | null>(null)

// ❌ URL 상태를 useState로 관리 (새로고침 시 유실)
const [search, setSearch] = useState('')  // → nuqs 사용

// ❌ 단일 컴포넌트만 쓰는 상태를 Zustand에 (overengineering)
const useDialogStore = create((set) => ({ isOpen: false, ... }))  // → useState
```

**Why**: 잘못된 카테고리 선택은 sync 버그·복잡도 폭발·디버깅 난해. 4-카테고리 + 결정 트리는 AI/사람 모두 일관된 판단 가능.

---

## SM-02 URL 상태는 nuqs로 관리 (🚫 MUST)

**규칙**: 사용자에게 보이는 상태(검색·정렬·페이지·필터)는 URL 쿼리 파라미터로 관리하며, `nuqs`를 사용한다.

> 라이브러리 선택은 [LIB-03](library-choices.md#lib-03-url-상태-must)에서 lock-in. 이 규칙은 옵션 가이드.

**Default**: nuqs `parseAs*` parser + `useQueryState` / `useQueryStates`.

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

// ❌ useSearchParams 직접 사용 (파싱 로직 반복) — LIB-03 위반
const page = parseInt(searchParams.get('page') ?? '1')
```

**고빈도 업데이트 예외**: 드래그·슬라이더·실시간 동기화는 LIB-03 예외 패턴 적용 (useState mirror + debounce).

**Why**: URL 상태는 새로고침, 북마크, 링크 공유 시 보존된다.

---

## SM-03 useTableState 훅 패턴 (⚠️ SHOULD)

**규칙**: 테이블의 URL 상태를 관리하는 공유 훅 `useTableState`를 사용한다.

**적용 위치**: `shared/lib/table/use-table-state.ts`

**Default**: 모든 server-side 테이블은 `useTableState()` 훅으로 q/page/limit/sort 통합 관리.

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

> Cross-reference: Vercel `rerender-lazy-state-init`. 이 규칙은 `vercel-react-best-practices` skill에도 있음 — 우리는 동일하게 prescribe.

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

> Cross-reference: Vercel `rerender-functional-setstate`. `vercel-react-best-practices` skill과 동일.

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

> Cross-reference: Vercel `rerender-dependencies`. `vercel-react-best-practices` skill과 동일.

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

---

# Zustand 사용 BP (SM-10 ~ SM-13)

> Zustand는 SM-01의 "클라이언트 전역 상태" 카테고리에서만 사용. 사용 시 아래 4개 규칙을 따른다.
>
> 출처: [TkDodo — Working with Zustand](https://tkdodo.eu/blog/working-with-zustand) (커뮤니티 표준 BP).
>
> Store 위치/크기 결정은 FSD 아키텍처 영역 → `feature-sliced-design` skill에 위임.

---

## SM-10 Zustand: custom hook만 export (🚫 MUST)

**규칙**: Zustand store 자체를 export하지 않는다. 컴포넌트는 항상 custom hook을 통해 접근.

**Default**: store는 module 내부에서만 보관, 외부로는 atomic custom hook만 export.

**Override policy** (Q4-B): 명시적 요청 시 경고 후 진행. 단 다른 컴포넌트의 일관성 위해 default 유지.

**Do**:

```ts
// shared/store/auth-store.ts (위치는 FSD skill 가이드 따름)
import { create } from 'zustand'

interface AuthState {
  user: User | null
  actions: {
    signIn: (user: User) => void
    signOut: () => void
  }
}

// store는 module 내부에만 보관 — export하지 않음
const authStore = create<AuthState>((set) => ({
  user: null,
  actions: {
    signIn: (user) => set({ user }),
    signOut: () => set({ user: null }),
  },
}))

// custom hook으로만 export
export const useAuth = () => authStore((s) => s.user)
export const useAuthActions = () => authStore((s) => s.actions)
```

**Don't**:

```ts
// ❌ store 자체를 export
export const useAuthStore = create<AuthState>((set) => ({ ... }))

// 컴포넌트에서 사용 시
const { user, actions } = useAuthStore()  // 전체 store 구독 → 어떤 변경에도 리렌더
```

**Why**: store 자체를 노출하면 컴포넌트가 전체 store를 구독하게 되어, 무관한 state 변경에도 리렌더가 발생한다. Custom hook으로 atomic selector를 강제한다.

---

## SM-11 Zustand: Atomic Selector (🚫 MUST)

**규칙**: selector는 단일 값(primitive 또는 안정적 참조)을 반환한다. 객체/배열을 새로 만드는 selector 금지.

**Default**: 한 번에 하나의 값만 selector. 여러 값이 필요하면 hook을 분리.

**Override policy** (Q4-B): 명시적 요청 시 경고. 객체 selector를 정말 써야 하면 `shallow` 비교 미들웨어 사용.

**Do**:

```ts
// 값 단위 atomic selector
export const useUser = () => authStore((s) => s.user)
export const useToken = () => authStore((s) => s.token)
export const useIsAuthenticated = () => authStore((s) => s.user !== null)
```

**Don't**:

```ts
// ❌ 객체 selector — 매 렌더마다 새 객체 → 항상 리렌더
export const useAuth = () => authStore((s) => ({
  user: s.user,
  token: s.token,
}))

// 컴포넌트에서 destructure 해도 같은 문제
const { user, token } = useAuth()  // 어떤 state 변경이든 리렌더
```

**Why**: Zustand는 strict equality(`===`) 비교로 리렌더 결정. 매번 새 객체를 반환하면 reference가 달라져서 항상 리렌더. Atomic selector는 값이 실제로 바뀔 때만 리렌더.

---

## SM-12 Zustand: Actions Namespace 분리 (⚠️ SHOULD)

**규칙**: state와 actions를 분리. Actions는 `actions` namespace에 모은다.

**Default**: `state.actions = { signIn, signOut, ... }` 패턴. Actions는 절대 안 바뀌므로 destructure 자유.

**Do**:

```ts
const authStore = create<AuthState>((set) => ({
  // state
  user: null,
  token: null,

  // actions — 한 namespace로
  actions: {
    signIn: (user, token) => set({ user, token }),
    signOut: () => set({ user: null, token: null }),
    refreshToken: async () => { ... },
  },
}))

// actions hook — 한 번에 가져와도 OK (actions는 안 바뀜)
export const useAuthActions = () => authStore((s) => s.actions)

// 컴포넌트에서 destructure 자유
const { signIn, signOut } = useAuthActions()
```

**Don't**:

```ts
// ❌ state와 actions 평면 배치
const authStore = create((set) => ({
  user: null,
  signIn: (user) => set({ user }),  // actions이 state와 섞임
  signOut: () => set({ user: null }),
}))

// 사용 시 atomic selector 강제됨 (보일러플레이트)
export const useSignIn = () => authStore((s) => s.signIn)
export const useSignOut = () => authStore((s) => s.signOut)
```

**Why**: Actions는 reference가 안정적(setter는 절대 안 바뀜). `actions` namespace를 한 hook으로 가져와도 리렌더 없음. 보일러플레이트 ↓.

---

## SM-13 Zustand: Action을 Domain Event로 명명 (⚠️ SHOULD)

**규칙**: Actions는 generic setter가 아닌 도메인 이벤트로 명명한다.

**Default**: `signIn`, `signOut`, `openSidebar`, `advanceWizardStep` 같은 동작 중심 명명.

**Do**:

```ts
actions: {
  signIn: (user) => set({ user }),
  signOut: () => set({ user: null }),
  openSidebar: () => set({ sidebarOpen: true }),
  closeSidebar: () => set({ sidebarOpen: false }),
  advanceWizardStep: () => set((s) => ({ step: s.step + 1 })),
  resetWizard: () => set({ step: 0, formData: {} }),
}
```

**Don't**:

```ts
// ❌ generic setter — 비즈니스 의미 없음
actions: {
  setUser: (user) => set({ user }),
  setSidebarOpen: (open) => set({ sidebarOpen: open }),
  setStep: (step) => set({ step }),
}
```

**Why**: Generic setter는 store가 단순한 데이터 백이 됨. Domain event로 명명하면 비즈니스 로직이 store 안에 명확히 표현되고, 호출 측 코드가 자연어처럼 읽힌다.
