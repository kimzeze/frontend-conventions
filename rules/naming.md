# Naming

파일, 변수, 타입, 상수, 훅, 컴포넌트의 네이밍 규칙.

> **Override Policy**: 이 파일의 모든 🚫 MUST 규칙은 [SKILL.md Override Policy (Q4-B)](../SKILL.md#override-policy-q4-b) 적용 — 사용자 명시 요청 시 경고 후 진행.

---

## NM-01 파일명 규칙 (🚫 MUST)

**규칙**: 일반 파일은 kebab-case, entity 폴더 내 파일은 고정명.

| 위치 | 규칙 | 예시 |
|------|------|------|
| entity 내부 | 고정명 | `api.ts`, `queries.ts`, `hooks.ts`, `types.ts`, `schema.ts`, `columns.tsx`, `index.ts` |
| shared/lib | kebab-case | `use-table-state.ts`, `table-utils.ts` |
| shared/ui | kebab-case | `data-table.tsx`, `sortable-header.tsx`, `form-dialog.tsx` |
| features/ui (다이얼로그) | PascalCase | `Create{Entity}Dialog.tsx`, `Edit{Entity}Dialog.tsx` |
| features/ui (sub) | kebab-case | `{entity}-form-fields.tsx` |
| features/hooks | camelCase + use | `useCreate{Entity}Form.ts` |

**Don't**:

```
// ❌ entity명을 파일명에 반복
entities/product/productApi.ts

// ❌ PascalCase 타입 파일
entities/product/ProductTypes.ts

// ❌ snake_case
entities/product/product_hooks.ts
```

**Why**: 일관된 파일명은 어떤 entity든 동일한 구조를 보장한다.

---

## NM-02 Hook 네이밍 (🚫 MUST)

**규칙**: Hook은 `use` 접두사 + 역할을 나타내는 이름.

| 종류 | 패턴 | 예시 |
|------|------|------|
| Query 훅 | `use{Entity}{Operation}` | `useProductList`, `useProductDetail` |
| Mutation 훅 | `use{Action}{Entity}` | `useCreateProduct`, `useUpdateProduct`, `useDeleteProduct` |
| Form 훅 | `use{Action}{Entity}Form` | `useCreateProductForm`, `useEditProductForm` |
| 상태 훅 | `use{Feature}State` | `useTableState`, `useDialogState` |

**Don't**:

```ts
useGetProducts()       // ❌ "get" 불필요 (useQuery 자체가 get)
useProductMutation()   // ❌ 어떤 mutation인지 불명확
useProducts()          // ❌ 목록인지 상세인지 불명확
useProductsList()      // ❌ "s" + "List" 중복
```

---

## NM-03 타입 네이밍 (🚫 MUST)

**규칙**: 타입은 PascalCase, 목적에 따라 접미사를 붙인다.

| 종류 | 패턴 | 예시 |
|------|------|------|
| 도메인 타입 | `{Entity}` (단수) | `Product`, `Category`, `Order` |
| API Input | `{Action}{Entity}Input` | `CreateProductInput`, `UpdateProductInput` |
| Form Values | `{Action}{Entity}FormValues` | `CreateProductFormValues` |
| 목록 파라미터 | `{Entity}ListParams` | `ProductListParams` |
| Enum/Union | `{Entity}{Field}` | `ProductStatus`, `OrderType` |

**Don't**:

```ts
interface IProduct { ... }        // ❌ "I" 접두사 (Java/C# 스타일)
type ProductType = 'A' | 'B'      // ❌ "Type" 모호 — Status, Category 등 구체적 이름
type Products = Product[]          // ❌ 복수형 타입 불필요 — Product[] 직접 사용
```

---

## NM-04 상수 네이밍 (⚠️ SHOULD)

**규칙**: 상수는 UPPER_SNAKE_CASE. Label/variant 맵은 `{ENTITY}_{FIELD}_{TYPE}` 패턴.

**Do**:

```ts
// Enum 값 배열
export const PRODUCT_STATUS_VALUES = ['ACTIVE', 'INACTIVE', 'DRAFT'] as const

// Label 맵 (UI 표시용)
export const PRODUCT_STATUS_LABELS: Record<ProductStatus, string> = {
  ACTIVE: '활성',
  INACTIVE: '비활성',
  DRAFT: '초안',
}

// Badge variant 맵
export const PRODUCT_STATUS_BADGE_VARIANTS: Record<ProductStatus, BadgeVariant> = {
  ACTIVE: 'default',
  INACTIVE: 'secondary',
  DRAFT: 'outline',
}

// 일반 상수
export const MAX_FILE_SIZE = 5 * 1024 * 1024
export const DEFAULT_PAGE_SIZE = 10
```

**Don't**:

```ts
export const productStatusLabels = { ... }  // ❌ camelCase
export const STATUS_LABELS = { ... }        // ❌ entity 누락 — 어느 entity?
```

**Magic Number 금지**: 함수/컴포넌트 외부의 숫자 리터럴은 상수로 추출한다.

```ts
// Do
const DEBOUNCE_DELAY_MS = 300
const DEFAULT_PAGE_SIZE = 10
const MAX_FILE_SIZE = 5 * 1024 * 1024

useDebouncedCallback(handler, DEBOUNCE_DELAY_MS)
if (file.size > MAX_FILE_SIZE) { ... }

// Don't
useDebouncedCallback(handler, 300)    // ❌ 300이 뭘 의미?
if (file.size > 5242880) { ... }       // ❌ 매직 넘버
```

---

## NM-05 Query Key 네이밍 (🚫 MUST)

**규칙**: Factory 객체는 `{entity}Keys` (camelCase 복수형), base key는 복수형 소문자 문자열.

**Do**:

```ts
export const productKeys = {
  all: ['products'] as const,      // 복수형 소문자
  lists: () => [...productKeys.all, 'list'] as const,
  list: (params) => [...productKeys.lists(), params] as const,
  details: () => [...productKeys.all, 'detail'] as const,
  detail: (id: number) => [...productKeys.details(), id] as const,
}
```

**Don't**:

```ts
export const productQueryKeys = { ... }  // ❌ "QueryKeys" 불필요
export const ProductKeys = { ... }       // ❌ PascalCase
all: ['product'] as const               // ❌ 단수형
all: ['Products'] as const              // ❌ 대문자
```

---

## NM-06 컴포넌트 네이밍 (⚠️ SHOULD)

**규칙**: 컴포넌트는 PascalCase, 역할에 따른 접미사 사용.

| 종류 | 패턴 | 예시 |
|------|------|------|
| 다이얼로그 | `{Action}{Entity}Dialog` | `CreateProductDialog`, `EditProductDialog` |
| 테이블 위젯 | `{Entity}Table` | `ProductTable` |
| 폼 필드 | `{Entity}FormFields` | `ProductFormFields` |
| 공유 UI | 기능 설명적 | `DataTable`, `SortableHeader`, `FormDialog` |

**Don't**:

```tsx
function ProductForm() { ... }     // ❌ Create? Edit? 불명확
function ProductModal() { ... }    // ❌ "Dialog"로 통일
function ProductManager() { ... }  // ❌ 너무 포괄적
function Product() { ... }         // ❌ 컴포넌트인지 타입인지 불명확
```

---

## NM-07 Type-Only Import/Export (⚠️ SHOULD)

**규칙**: 타입만 import/export할 때는 `import type` / `export type` 키워드를 사용한다.

**Do**:

```ts
// type-only import — 런타임 번들에 포함되지 않음
import type { Product, ProductListParams } from './types'
import type { ColumnDef } from '@tanstack/react-table'
import type { Control } from 'react-hook-form'

// type-only export
export type { Product, CreateProductInput } from './types'

// 값과 타입을 함께 import할 때
import { productApi } from './api'
import type { Product } from './types'
```

**Don't**:

```ts
// ❌ 값 import로 타입 가져오기 (tree-shaking에 불리)
import { Product } from './types'  // Product가 type-only라면 import type 사용

// ❌ 혼합 import에서 타입을 구분하지 않음
import { productApi, Product, ProductListParams } from './api'
// → import { productApi } from './api'
// → import type { Product, ProductListParams } from './types'
```

**Why**: `import type`은 컴파일 시 완전히 제거되어 런타임 번들에 포함되지 않는다. 의도를 명확히 하고 tree-shaking을 돕는다.

> **자동 강제**: `tsconfig.json`에 `verbatimModuleSyntax: true` + ESLint `@typescript-eslint/consistent-type-imports`로 자동 강제.

---

## NM-08 Import 순서 (⚠️ SHOULD)

**규칙**: import 문은 아래 순서를 따른다. `eslint-plugin-import`로 자동 정렬.

```ts
// 1. React / Next.js
import { useState, useMemo } from 'react'
import dynamic from 'next/dynamic'

// 2. 외부 라이브러리
import { useQuery } from '@tanstack/react-query'
import { z } from 'zod'
import { Plus } from 'lucide-react'

// 3. @workspace 패키지
import { Button } from '@workspace/ui/components/button'
import { clientFetch } from '@workspace/api/fetch/clientFetch'

// 4. @/ 내부 모듈 (shared → entities → features → widgets 순)
import { DataTable } from '@/shared/ui/data-table'
import { useProductList } from '@/entities/product'
import { CreateProductDialog } from '@/features/product/ui/CreateProductDialog'

// 5. 상대 경로
import { ProductFormFields } from './product-form-fields'

// 6. 타입 import (각 그룹 끝 또는 별도 그룹)
import type { Product } from '@/entities/product'
```

**그룹 간 빈 줄**: 각 그룹 사이에 빈 줄 1개로 구분.

**Why**: 일관된 import 순서는 코드 탐색성을 높이고, 의존성 방향을 시각적으로 보여준다.
