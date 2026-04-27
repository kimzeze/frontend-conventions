# Mutations

TanStack Query v5 기반 **쓰기 mutation** 규칙. invalidate, useApiMutation, optimistic updates. 라이브러리 선택은 [LIB-02](library-choices.md#lib-02-데이터-페칭-must--최강-규칙) lock-in.

> Cross-reference: TanStack Query BP (`mut-invalidate-queries`, `mut-optimistic-updates`)
>
> **Override Policy**: 이 파일의 모든 🚫 MUST 규칙은 [SKILL.md Override Policy (Q4-B)](../SKILL.md#override-policy-q4-b) 적용 — 사용자 명시 요청 시 경고 후 진행.

---

## DF-06 Mutation: invalidate + toast (🚫 MUST)

**규칙**: entity hooks.ts의 mutation은 **쿼리 무효화만** 담당한다. **toast는 feature hooks에서 호출**한다 (관심사 분리).

**Do**:

```ts
// entities/{entity}/hooks.ts — mutation + invalidation만
export const useCreateProduct = () => {
  const queryClient = useQueryClient()
  return useApiMutation({
    mutationFn: (input: CreateProductInput) => productApi.create(input),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: productKeys.all })
      // ⚠️ toast는 여기서 호출하지 않는다
    },
  })
}

// features/{entity}/hooks/useCreateProductForm.ts — toast + form reset
export const useCreateProductForm = () => {
  const { mutate: createProduct, isPending } = useCreateProduct()
  const form = useForm(...)

  const submit = (values, onSuccess: () => void) => {
    createProduct(values, {
      onSuccess: () => {
        toast.success('상품이 등록되었습니다.')
        form.reset()
        onSuccess()
      },
    })
  }
  return { form, submit, isPending }
}
```

> **Note**: TanStack Query 공식 문서는 mutation hook 안에서 invalidation + side effects를 함께 넣는 패턴을 보여준다. 우리는 **관심사 분리**를 위해 entity(invalidation) / feature(toast) 분리를 채택. 이유: 같은 mutation hook을 다른 feature에서 다른 toast로 재사용할 수 있음.

**무효화 전략**:

```ts
// 기본: broad invalidation — 단순하고 안전
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: productKeys.all })
}

// 최적화 (성능 이슈 시): targeted invalidation
onSuccess: (data, { id }) => {
  queryClient.invalidateQueries({ queryKey: productKeys.detail(id) })
  queryClient.invalidateQueries({ queryKey: productKeys.lists() })
}
```

**Don't**:

```ts
// ❌ 무효화 누락 + raw useMutation 사용 (DF-07 위반)
useMutation({ mutationFn: productApi.create })

// ❌ 전체 캐시 무효화
queryClient.invalidateQueries()  // 모든 쿼리!

// ❌ entity hook에서 toast 호출 (feature에서 호출해야 함)
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: productKeys.all })
  toast.success('등록되었습니다.')  // entity에서 호출하지 않는다
}
```

**Why**: 무효화 없이는 UI가 서버 상태와 동기화되지 않는다. 토스트는 사용자에게 액션 결과를 즉시 알린다.

---

---

## DF-07 useApiMutation 래퍼 사용 (🚫 MUST)

**적용 위치**: `entities/*/hooks.ts`

**규칙**: 모든 mutation은 raw `useMutation` 대신 `useApiMutation` 래퍼를 사용한다.

**Do**:

```ts
import { useApiMutation } from '@workspace/api/query/useApiMutation'

// 기본: 에러 시 toast 자동 표시
useApiMutation({ mutationFn: productApi.create })

// 폼 인라인 에러 표시
useApiMutation({ mutationFn: productApi.create, displayType: 'inline' })

// 백그라운드 작업 (에러 무시)
useApiMutation({ mutationFn: syncApi.push, displayType: 'silent' })
```

| displayType | 동작 | 용도 |
|-------------|------|------|
| `'toast'` (기본) | 토스트 알림 | 일반 CRUD |
| `'inline'` | 컴포넌트에 에러 전달 | 서버 validation |
| `'silent'` | 에러 무시 | 백그라운드 동기화 |

**Why**: 에러 처리 로직을 중복 작성하지 않고, 모든 mutation에서 일관된 에러 경험을 제공한다.

---

---

## DF-13 Optimistic Updates (✅ MAY)

**규칙**: 결과가 예측 가능한 사용자 액션(토글, 상태 변경)에는 optimistic update를 고려한다.

**패턴**:

```ts
const useToggleProductActive = (listParams: ProductListParams) => {
  const queryClient = useQueryClient()
  return useApiMutation({
    mutationFn: (id: number) => productApi.toggleActive(id),
    displayType: 'silent', // optimistic이므로 에러 toast 수동 제어
    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: productKeys.lists() })
      const previous = queryClient.getQueryData(productKeys.list(listParams))
      queryClient.setQueryData(productKeys.list(listParams), (old: PaginatedResponse<Product> | undefined) => {
        if (!old) return old
        return {
          ...old,
          results: old.results.map(p =>
            p.id === id ? { ...p, is_active: !p.is_active } : p
          ),
        }
      })
      return { previous }
    },
    onError: (err, id, context) => {
      if (context?.previous) {
        queryClient.setQueryData(productKeys.list(listParams), context.previous)
      }
      toast.error('변경에 실패했습니다.')
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: productKeys.lists() })
    },
  })
}
```

| 상황 | Optimistic? |
|------|------------|
| CRUD 폼 (다이얼로그 닫히며 대기) | 아니오 |
| 체크박스 토글 | 예 |
| 상태 드롭다운 변경 | 예 |
| 드래그 앤 드롭 순서 변경 | 예 |

**Why**: 서버 응답 대기 없이 UI를 즉시 반영하여 체감 성능을 높인다.
