# Architecture

FSD(Feature-Sliced Design) 기반 프로젝트 아키텍처 규칙. 레이어 구성, import 방향, 파일 배치, 추상화 수준.

> Cross-reference: [Feature-Sliced Design](https://feature-sliced.design), [TanStack Query BP](https://tanstack.com/query)

---

## PS-01 FSD 레이어 구조 (🚫 MUST)

**규칙**: 프로젝트는 FSD 5개 레이어로 구성하며, `src/` 디렉토리 없이 앱 루트에 직접 배치한다.

**Do**:

```
apps/{app-name}/
├── app/                    # Next.js App Router (라우트 전용)
│   ├── (dashboard)/        # 사이드바 있는 페이지 그룹
│   │   ├── products/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   └── login/              # 사이드바 없는 페이지
│       └── page.tsx
├── entities/               # 도메인 모델 + 쿼리
│   ├── product/
│   └── category/
├── features/               # 사용자 액션 (폼, 다이얼로그)
│   └── product/
├── widgets/                # 페이지 수준 조합 컴포넌트
│   └── product-table.tsx
└── shared/                 # 공유 유틸리티, UI, 타입
    ├── ui/
    ├── lib/
    ├── config/
    └── types/
```

**Don't**:

```
apps/{app-name}/
├── src/                    # ❌ src/ 디렉토리 사용 금지
│   ├── components/         # ❌ FSD 레이어가 아닌 기능별 분류
│   ├── hooks/              # ❌ 훅을 한 곳에 모으지 않음
│   └── utils/              # ❌ shared/lib 사용
```

**Why**: FSD는 관심사를 도메인 단위로 분리하여 코드 탐색성과 독립성을 높인다. `src/`를 생략하면 경로가 짧아지고 Next.js App Router와 자연스럽게 공존한다.

---

---

## PS-02 Import 방향 (🚫 MUST)

**규칙**: Import는 반드시 하향만 허용한다. 상위 레이어가 하위 레이어를 import하며, 역방향은 금지.

```
app → widgets → features → entities → shared
```

**Do**:

```ts
// widgets/product-table.tsx — feature와 entity를 import
import { CreateProductDialog } from '@/features/product/ui/CreateProductDialog'
import { useProductList, productColumns } from '@/entities/product'
import { DataTable } from '@/shared/ui/data-table'

// entities/product/hooks.ts — shared만 import
import { clientFetch } from '@workspace/api/fetch/clientFetch'
```

**Don't**:

```ts
// ❌ entity가 feature를 import
// entities/product/hooks.ts
import { CreateProductDialog } from '@/features/product/ui/CreateProductDialog'

// ❌ shared가 entity를 import
// shared/ui/data-table.tsx
import { Product } from '@/entities/product'

// ❌ entity가 다른 entity를 import (type-only도 금지)
// entities/product/columns.tsx
import type { Category } from '@/entities/category'
// → widget에서 categoryMap을 주입하는 팩토리 함수로 해결
```

**entity 간 데이터가 필요한 경우**: widget에서 주입 패턴 사용.

```ts
// Do — widget에서 주입
export function createProductColumns(categoryMap: Map<number, string>): ColumnDef<Product>[] { ... }

// widget에서 사용
const categoryMap = useCategoryMap()
const columns = useMemo(() => createProductColumns(categoryMap), [categoryMap])
```

**Why**: 단방향 의존성이 깨지면 순환 참조가 발생하고, 독립적인 테스트와 리팩토링이 불가능해진다. entity 간 type import도 금지하여 의존성을 완전히 차단한다.

---

---

## PS-03 Entity Slice 파일 구성 (🚫 MUST)

**적용 위치**: `entities/*/`

**규칙**: 각 entity는 아래 고정된 파일명으로 구성한다.

| 파일 | 역할 | 필수 |
|------|------|------|
| `types.ts` | 도메인 타입 (API 응답 shape) | O |
| `api.ts` | API 함수 (clientFetch 호출) | O |
| `queries.ts` | queryOptions factory + key factory (DF-01) | O |
| `hooks.ts` | Mutation 훅 (useApiMutation) | O |
| `index.ts` | Barrel re-export | O |
| `schema.ts` | Zod 스키마 + FormValues 타입 | 폼이 있을 때 |
| `columns.tsx` | TanStack Table 컬럼 정의 | 테이블이 있을 때 |

> `queries.ts`에는 queryOptions factory + key factory를 통합. `hooks.ts`에는 mutation 훅만. DF-01 참조.

**Do**:

```
entities/{entity-name}/
├── api.ts
├── hooks.ts
├── types.ts
├── schema.ts
├── query-keys.ts
├── columns.tsx
└── index.ts
```

**Don't**:

```
entities/{entity-name}/
├── productApi.ts          # ❌ entity명을 파일명에 반복
├── useProductQuery.ts     # ❌ 훅을 개별 파일로 분리
├── ProductTypes.ts        # ❌ PascalCase 파일명
└── index.ts
```

**Why**: 파일명이 고정되면 어떤 entity든 동일한 구조를 가지므로, 새 entity 추가 시 복사-수정이 단순해진다.

---

---

## PS-05 Barrel Export 패턴 (🚫 MUST)

**적용 위치**: `entities/*/index.ts`, `features/*/index.ts`

**규칙**: FSD entity/feature 경계에서는 `index.ts`로 public API를 정의한다. `shared/ui`와 `packages/ui`에서는 barrel을 사용하지 않고 직접 파일 경로로 import한다.

**Do**:

```ts
// entities/{entity}/index.ts — entity의 public API 정의
export { productApi } from './api'
export { productKeys, productQueries } from './queries'
export { useCreateProduct, useUpdateProduct, useDeleteProduct } from './hooks'
export { productColumns } from './columns'
export type { Product, ProductListParams, CreateProductInput } from './types'
```

```ts
// 사용측 — barrel을 통해 import
import { useProductList, productColumns, type Product } from '@/entities/product'
```

```ts
// shared/ui — 직접 파일 경로 import (barrel 미사용)
import { DataTable } from '@/shared/ui/data-table'
import { Button } from '@workspace/ui/components/button'
```

**Don't**:

```ts
// ❌ shared/ui에서 barrel import
import { DataTable, SortableHeader } from '@/shared/ui'

// ❌ entity 내부 파일에 직접 접근 (barrel 우회)
import { productApi } from '@/entities/product/api'
```

> **Note**: Vercel `bundle-barrel-imports` 규칙과의 충돌 — FSD 캡슐화가 우선. Next.js `optimizePackageImports` 옵션과 tree-shaking으로 완화. 상세 근거는 [ADR-004](../DECISIONS.md) 참조.

**Why**: Barrel export는 entity의 public API를 명시적으로 정의하여, 외부에서 내부 구현에 의존하지 않도록 보호한다.

---

---

## PS-06 Widget = 조합 컴포넌트 (⚠️ SHOULD)

**규칙**: Widget은 테이블 상태 + 쿼리 훅 + 다이얼로그 상태 + UI를 조합하는 단위이다. 페이지 섹션당 하나의 widget.

**Do**:

```tsx
// widgets/{entity}-table.tsx
export function ProductTable() {
  // 1. URL 상태
  const tableState = useTableState()
  // 2. 데이터 페칭
  const { data, isLoading } = useProductList({ ... })
  // 3. 다이얼로그 상태
  const dialog = useDialogState<Product>()

  return (
    <>
      <DataTableToolbar ...>
        <Button onClick={dialog.create.open}>등록</Button>
      </DataTableToolbar>
      <DataTable ... />
      <DataTablePagination ... />
      <CreateProductDialog ... />
    </>
  )
}
```

**패턴**: `useTableState() → useEntityList() → useDialogState() → UI 조합`

**Why**: Widget이 오케스트레이션을 담당하면, page는 thin shell을 유지하고, entity/feature는 독립적으로 남는다.

---

---

## PS-07 Feature = 사용자 액션 (⚠️ SHOULD)

**규칙**: Feature는 하나의 사용자 액션을 나타내며, entity + shared를 조합하여 구현한다.

**Do**:

```
features/{entity}/
├── ui/
│   ├── Create{Entity}Dialog.tsx
│   ├── Edit{Entity}Dialog.tsx
│   └── {entity}-form-fields.tsx
└── hooks/
    ├── useCreate{Entity}Form.ts
    └── useEdit{Entity}Form.ts
```

**Don't**:

```
features/{entity}/
├── create-{entity}/         # ❌ nested 구조 (flat으로 통일)
│   ├── hooks/
│   └── ui/
├── edit-{entity}/           # ❌ nested 구조
│   ├── hooks/
│   └── ui/

features/{entity}/
├── EntityManager.tsx        # ❌ 모호한 이름
├── EntityActions.tsx         # ❌ 여러 액션 합치기
```

**Why**: 액션 단위로 분리하면 각 feature가 독립적으로 수정/삭제 가능하고, 코드 리뷰 범위가 명확해진다.

---

---

## PS-08 @workspace/ Import 스코프 (🚫 MUST)

**규칙**: 모노레포 패키지는 반드시 `@workspace/` 스코프로 import한다. `@repo/`는 사용 금지.

**Do**:

```ts
// 패키지 import — @workspace/ 스코프
import { Button } from '@workspace/ui/components/button'
import { cn } from '@workspace/ui/lib/utils'
import { clientFetch } from '@workspace/api/fetch/clientFetch'
```

```ts
// 앱 내부 import — @/ alias
import { useProductList } from '@/entities/product'
import { DataTable } from '@/shared/ui/data-table'
```

**Don't**:

```ts
// ❌ @repo/ 스코프
import { Button } from '@repo/ui/components/button'

// ❌ 상대 경로로 패키지 접근
import { Button } from '../../../packages/ui/src/components/button'
```

**Why**: `@workspace/` 스코프는 shadcn CLI의 컴포넌트 자동 생성과 호환된다.

---

---

## PS-12 단순 중복 > 과도한 추상화 (⚠️ SHOULD)

> Cross-reference: Clean Code React — Coupling: Handling Duplicate Code

**규칙**: entity별 api.ts, hooks.ts, columns.tsx 등의 구조가 비슷하더라도, 범용 추상화를 만들지 않는다. 3곳 이상에서 **정확히 동일한** 패턴이 반복될 때만 유틸리티로 추출한다.

**Do**:

```ts
// 각 entity의 api.ts가 비슷한 구조를 반복 — 이것은 의도적 설계
export const productApi = {
  getList: (params) => { const query = new URLSearchParams(); ... return clientFetch(...) },
  create: (input) => clientFetch(...),
}

export const categoryApi = {
  getList: (params) => { const query = new URLSearchParams(); ... return clientFetch(...) },
  create: (input) => clientFetch(...),
}
```

**Don't**:

```ts
// ❌ 범용 API 팩토리 (설정 파라미터가 과도)
function createEntityApi<T>(config: {
  basePath: string
  searchField?: string
  hasDetail?: boolean
  hasBulkCreate?: boolean
}) { ... }

const productApi = createEntityApi<Product>({ basePath: '/product', searchField: 'name_contains', ... })
```

**판단 기준**:

| 상황 | 행동 |
|------|------|
| 2-3줄의 비슷한 코드 | 그대로 둔다 |
| 3곳 이상에서 **동일한** 10줄+ 로직 | 유틸리티로 추출 |
| 비슷하지만 **미묘하게 다른** 코드 | 그대로 둔다 (통합 시 if/else 증가) |

**Why**: 과도한 추상화는 설정 파라미터가 증가하고, 한 곳의 변경이 모든 entity에 영향을 미친다. entity별 독립적 코드는 수정 범위가 명확하고 이해하기 쉽다.
