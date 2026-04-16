# Code Quality

코드 품질 & 성능 최적화 규칙. Dynamic Import, Conditional Rendering, Early Return, React 19 forwardRef.

> Cross-reference: [Vercel React BP](https://vercel.com/blog/introducing-react-best-practices), [React 19 Release](https://react.dev/blog/2024/12/05/react-19)

---

## PS-09 Dynamic Import (⚠️ SHOULD)

> Cross-reference: Vercel `bundle-dynamic-imports`, `bundle-defer-third-party`

**규칙**: 초기 렌더링에 필요하지 않은 무거운 컴포넌트는 `next/dynamic`으로 지연 로드한다.

**Do**:

```tsx
import dynamic from 'next/dynamic'

// 무거운 에디터 컴포넌트 — 클릭 시에만 로드
const RichTextEditor = dynamic(
  () => import('@/shared/ui/rich-text-editor').then(m => m.RichTextEditor),
  { ssr: false }
)

// 분석/로깅 — hydration 이후 로드
const Analytics = dynamic(
  () => import('@vercel/analytics/react').then(m => m.Analytics),
  { ssr: false }
)
```

**Don't**:

```tsx
// ❌ 무거운 컴포넌트를 정적 import (초기 번들에 포함)
import { RichTextEditor } from '@/shared/ui/rich-text-editor'
import { Analytics } from '@vercel/analytics/react'
```

**적용 기준**:

| 컴포넌트 | Dynamic Import? |
|----------|----------------|
| 에디터 (Monaco, TipTap 등) | 예 — 300KB+ |
| 차트 라이브러리 | 예 — 200KB+ |
| 모달/다이얼로그 내부 복잡 UI | 상황에 따라 |
| 분석/로깅 SDK | 예 — ssr: false |
| 기본 UI 컴포넌트 (Button, Input) | 아니오 |

**Why**: Dynamic import은 초기 번들 크기를 줄여 TTI(Time to Interactive)와 LCP를 개선한다.

---

---

## PS-10 Conditional Rendering (⚠️ SHOULD)

> Cross-reference: Vercel `rendering-conditional-render`

**규칙**: 조건부 렌더링에 `&&` 대신 삼항 연산자를 사용한다.

**Do**:

```tsx
// 삼항 — 0, '', NaN이 렌더링될 위험 없음
{items.length > 0 ? <ItemList items={items} /> : null}

{isLoading ? <Skeleton /> : <Content data={data} />}
```

**Don't**:

```tsx
// ❌ && 연산자 — items.length가 0이면 "0"이 화면에 렌더링됨
{items.length && <ItemList items={items} />}

// ❌ 빈 문자열이 렌더링될 수 있음
{errorMessage && <ErrorBanner message={errorMessage} />}
```

**예외 — boolean 조건은 `&&` 허용**:

```tsx
// ✓ boolean은 JSX로 렌더링되지 않으므로 안전
{isLoading && <Spinner />}
{hasError && <ErrorBanner />}
{isAdmin && <AdminPanel />}
```

**판단 기준**:

| 좌항 타입 | `&&` 사용 | 이유 |
|----------|----------|------|
| `boolean` | ✓ 허용 | `false`는 렌더링 안 됨 |
| `number` | ✗ 삼항 사용 | `0`이 렌더링됨 |
| `string` | ✗ 삼항 사용 | `''`이 렌더링됨 |
| `object \| null` | ✓ 허용 | `null`은 렌더링 안 됨 |

**3개 이상 분기 — Object Lookup 사용**:

```tsx
// Do — object lookup (3+ 분기)
const STATUS_COMPONENTS: Record<Status, ReactNode> = {
  pending: <PendingBadge />,
  active: <ActiveBadge />,
  inactive: <InactiveBadge />,
}
return STATUS_COMPONENTS[status] ?? <UnknownBadge />

// Don't — 중첩 삼항
{status === 'pending' ? <Pending /> : status === 'active' ? <Active /> : <Inactive />}
```

**Why**: `&&`의 왼쪽이 falsy 값(0, '', NaN)이면 그 값이 JSX로 렌더링된다. boolean 조건은 안전하지만, number/string은 위험. 3개 이상 분기는 object lookup이 가독성이 높다.

---

---

## PS-15 Early Return (⚠️ SHOULD)

> Cross-reference: Vercel `js-early-exit`

**규칙**: 조건 불일치 시 즉시 return하여 중첩을 줄인다 (guard clause 패턴).

**Do**:

```ts
function processOrder(order: Order) {
  if (!order.items.length) return null
  if (order.status === 'cancelled') return null

  // 핵심 로직 — 중첩 없이 flat
  const total = calculateTotal(order.items)
  return { ...order, total }
}
```

```tsx
function UserProfile({ user }: { user: User | null }) {
  if (!user) return <EmptyState />

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  )
}
```

**Don't**:

```ts
// ❌ 깊은 중첩
function processOrder(order: Order) {
  if (order.items.length) {
    if (order.status !== 'cancelled') {
      const total = calculateTotal(order.items)
      return { ...order, total }
    }
  }
  return null
}
```

**Why**: Early return은 "이 조건이면 더 볼 것 없다"를 명시하여 인지 부하를 줄인다.

---

---

## PS-16 forwardRef 금지 (⚠️ SHOULD)

> React 19에서는 ref를 일반 prop으로 전달 가능. forwardRef는 불필요.

**규칙**: `forwardRef`를 사용하지 않는다. ref가 필요하면 props로 직접 받는다.

**Do**:

```tsx
// React 19 — ComponentProps를 사용하면 ref가 자동 포함
function Input({ className, type, ...props }: React.ComponentProps<'input'>) {
  return <input type={type} className={cn('...', className)} {...props} />
}

// 또는 커스텀 props가 필요한 경우
function CustomInput({ label, ref, ...props }: { label: string } & React.ComponentProps<'input'>) {
  return (
    <div>
      <label>{label}</label>
      <input ref={ref} {...props} />
    </div>
  )
}
```

**Don't**:

```tsx
// ❌ forwardRef (React 18 패턴)
const Input = forwardRef<HTMLInputElement, InputProps>((props, ref) => {
  return <input ref={ref} {...props} />
})
```

**Why**: React 19에서 forwardRef는 deprecated. ref를 일반 prop으로 받으면 코드가 단순해지고 타입 추론이 개선된다.
