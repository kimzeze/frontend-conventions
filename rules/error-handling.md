# Error Handling

API 에러 처리 규칙. ApiError 클래스 기반 계층적 에러 처리.

---

## EH-01 ApiError 클래스 활용 (🚫 MUST)

**규칙**: 모든 API 에러는 `ApiError` 인스턴스로 변환되며, convenience getter로 에러 종류를 판단한다.

**Do**:

```ts
// ApiError 구조
class ApiError extends Error {
  httpStatus: number
  errorCode?: number
  raw?: unknown

  get isAuth(): boolean     // 인증 에러
  get isServer(): boolean   // 5xx 서버 에러
  get isNetwork(): boolean  // 네트워크 실패

  static is(error: unknown): error is ApiError  // 타입 가드
}

// 에러 타입별 분기
if (ApiError.is(error)) {
  if (error.isAuth) { /* cache level 자동 처리 */ }
  else if (error.isServer) { /* 사용자에게 안내 */ }
  else if (error.isNetwork) { /* 재시도 안내 */ }
}
```

**Don't**:

```ts
// ❌ httpStatus로 직접 분기
if (error.status === 401 || error.status === 403) { ... }

// ❌ try/catch를 각 hook에서 반복
useMutation({
  mutationFn: async (input) => {
    try { return await api.create(input) }
    catch (e) { /* 개별 처리 */ }
  },
})
```

**Why**: `ApiError`는 에러 코드 → 한국어 메시지 매핑과 에러 종류별 분기를 일관되게 처리한다.

---

## EH-02 Mutation 에러 표시 전략 (⚠️ SHOULD)

**규칙**: `useApiMutation`의 `displayType`으로 에러 표시 방식을 선택한다.

| displayType | 동작 | 용도 |
|-------------|------|------|
| `'toast'` (기본) | 토스트 알림 | 일반 CRUD |
| `'inline'` | 컴포넌트에 에러 전달 | 서버 validation 에러를 폼에 표시 |
| `'silent'` | 에러 무시 | 백그라운드 동기화, 로깅 |

```ts
// 기본: toast
useApiMutation({ mutationFn: api.create })

// inline: 폼 validation
useApiMutation({ mutationFn: api.create, displayType: 'inline' })

// silent: 백그라운드
useApiMutation({ mutationFn: syncApi.push, displayType: 'silent' })
```

**Why**: 에러 표시를 옵션 하나로 제어하여 일관성과 유연성을 동시에 확보한다.

---

## EH-03 Auth 에러 자동 처리 (🚫 MUST)

**규칙**: 인증 에러는 `QueryCache`/`MutationCache` 레벨에서 자동 처리한다. 개별 컴포넌트에서 처리하지 않는다.

**Do**:

```ts
// QueryClient 설정 — 전역 에러 핸들러
const queryClient = new QueryClient({
  queryCache: new QueryCache({ onError: handleCacheError }),
  mutationCache: new MutationCache({ onError: handleCacheError }),
})

// handleCacheError 내부
if (error.isAuth) {
  handleAuthError()  // 토큰 리프레시 → 실패 시 로그아웃
  return
}
```

**Don't**:

```ts
// ❌ 개별 hook에서 auth 에러 처리
useQuery({
  onError: (error) => {
    if (error.isAuth) router.push('/login')  // cache level에서 자동 처리됨
  },
})
```

**Why**: Auth 에러는 앱 전체에서 동일하게 처리되어야 한다. Cache level에서 한 번 정의하면 모든 query/mutation에 자동 적용.

---

## EH-04 에러 메시지 한국어 (⚠️ SHOULD)

**규칙**: 사용자에게 표시되는 모든 에러 메시지는 한국어로 작성한다.

```ts
// HTTP 상태 기본 메시지
const DEFAULT_MESSAGES: Record<number, string> = {
  400: '잘못된 요청입니다.',
  401: '로그인이 필요합니다.',
  403: '접근 권한이 없습니다.',
  404: '요청한 정보를 찾을 수 없습니다.',
  500: '서버 오류가 발생했습니다.',
  502: '서버가 일시적으로 응답할 수 없습니다.',
  503: '서버 점검 중입니다.',
  504: '서버 응답이 지연되고 있습니다.',
}

// Mutation 성공 토스트 — 주어 포함 형식
toast.success('{entity}가 등록되었습니다.')
toast.success('{entity} 정보가 수정되었습니다.')
toast.success('{entity}가 삭제되었습니다.')
```

**Toast 종류별 사용 기준**:

| 종류 | 용도 | 예시 |
|------|------|------|
| `toast.success` | Mutation 성공 | `'상품이 등록되었습니다.'` |
| `toast.error` | Mutation 실패 (useApiMutation 자동) | `error.message` |
| `toast.info` | 정보 안내 (에러도 성공도 아닌) | `'이미 존재하는 항목입니다.'` |
| `toast.warning` | 주의/확인 필요 | `'변경사항이 저장되지 않았습니다.'` |

**메시지 형식**: `'{주어}가/이 {동사}되었습니다.'`

```ts
// Do — 주어 포함
toast.success('상품이 등록되었습니다.')
toast.success('사용자 정보가 수정되었습니다.')
toast.info('이미 존재하는 항목입니다.')

// Don't — 주어 없음
toast.success('등록되었습니다.')     // ❌ 무엇이 등록?
toast.success('등록 완료')          // ❌ 명사형
```

**Toast Position**: `bottom-right` 통일.

**Don't**:

```ts
toast.error('Failed to create item')       // ❌ 영문
toast.error('HTTP 500 Internal Server Error') // ❌ 기술적 메시지
```

---

## EH-05 Error Boundary (⚠️ SHOULD)

> Cross-reference: TanStack `err-error-boundaries`

**규칙**: 페이지 섹션별로 Error Boundary를 배치하여 부분 실패를 격리한다.

**Do**:

```tsx
import { useQueryErrorResetBoundary } from '@tanstack/react-query'
import { ErrorBoundary } from 'react-error-boundary'

function QueryErrorBoundary({ children }: { children: React.ReactNode }) {
  const { reset } = useQueryErrorResetBoundary()

  return (
    <ErrorBoundary
      onReset={reset}
      fallbackRender={({ error, resetErrorBoundary }) => (
        <div className="p-4 text-center">
          <p className="text-muted-foreground">{error.message}</p>
          <Button variant="outline" onClick={resetErrorBoundary}>
            다시 시도
          </Button>
        </div>
      )}
    >
      {children}
    </ErrorBoundary>
  )
}

// 사용 — 각 섹션이 독립적으로 실패/복구
function DashboardPage() {
  return (
    <div className="grid grid-cols-2 gap-4">
      <QueryErrorBoundary>
        <Suspense fallback={<Skeleton />}>
          <RecentActivity />
        </Suspense>
      </QueryErrorBoundary>
      <QueryErrorBoundary>
        <Suspense fallback={<Skeleton />}>
          <Statistics />
        </Suspense>
      </QueryErrorBoundary>
    </div>
  )
}
```

**Don't**:

```tsx
// ❌ 앱 전체를 하나의 Error Boundary로 감싸기 (한 곳 실패 → 전체 실패)
<ErrorBoundary><App /></ErrorBoundary>
```

**Why**: 섹션별 Error Boundary는 한 곳의 에러가 전체 페이지를 깨뜨리지 않게 격리한다. `useQueryErrorResetBoundary`는 retry 시 query 에러 상태도 함께 리셋한다.

---

## EH-06 localStorage/sessionStorage try-catch (✅ MAY)

**규칙**: `localStorage`/`sessionStorage` 접근은 반드시 try-catch로 감싼다.

**Do**:

```ts
function savePrefs(prefs: UserPrefs) {
  try {
    localStorage.setItem('prefs:v1', JSON.stringify(prefs))
  } catch {
    // incognito/private 모드, quota 초과, disabled
  }
}

function loadPrefs(): UserPrefs | null {
  try {
    const data = localStorage.getItem('prefs:v1')
    return data ? JSON.parse(data) : null
  } catch {
    return null
  }
}
```

**Don't**:

```ts
// ❌ try-catch 없음 — Safari incognito에서 런타임 크래시
const data = localStorage.getItem('prefs')
localStorage.setItem('prefs', JSON.stringify(prefs))
```

**Why**: `localStorage`는 incognito/private 모드(Safari, Firefox), quota 초과, 비활성화 상태에서 throw한다.
