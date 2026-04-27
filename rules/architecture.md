# Architecture

> **FSD 위임**: FSD 레이어 구조, import 방향, slice 정의, widget/feature 정의는 공식 [`feature-sliced-design`](https://skills.sh/feature-sliced/skills/feature-sliced-design) skill에 **전적 위임**한다. 이 파일은 그 위에 우리 harness가 추가로 고정한 컨벤션만 다룬다.
>
> **삭제된 규칙**: PS-01 (FSD 레이어 구조), PS-02 (import 하향 방향), PS-06 (widget = 조합), PS-07 (feature = 사용자 액션) — 모두 `feature-sliced-design` skill에서 다룸.

---

## PS-03 Entity Slice 파일 고정명 (🚫 MUST)

**적용 위치**: `entities/*/`

**규칙**: 각 entity는 아래 고정된 **technical-role 파일명**으로 구성한다.

| 파일 | 역할 | 필수 |
|------|------|------|
| `types.ts` | 도메인 타입 (API 응답 shape) | O |
| `api.ts` | API 함수 (clientFetch 호출) | O |
| `queries.ts` | queryOptions factory + key factory (DF-01) | O |
| `hooks.ts` | Mutation 훅 (useApiMutation) | O |
| `index.ts` | Barrel re-export | O |
| `schema.ts` | Zod 스키마 + FormValues 타입 | 폼이 있을 때 |
| `columns.tsx` | TanStack Table 컬럼 정의 | 테이블이 있을 때 |

> **⚠️ FSD skill override (의도적)**: 공식 FSD 2.1은 `types.ts`/`api.ts` 같은 **technical-role file names를 안티패턴으로 규정**하고 도메인 기반 네이밍을 권고한다. 이 harness는 **AI 코드 생성 일관성을 최우선**으로 하기 위해 의도적으로 override한다. 이유: 모든 entity가 동일한 파일명 구조를 가져야 새 entity 추가 시 AI가 일관되게 파일 생성 가능. 두 skill이 같은 프로젝트에 설치되면 이 규칙이 FSD의 domain-based naming을 덮는다.

**Default**: 위 7개 고정명. queryOptions factory + key factory는 단일 파일 `queries.ts`에 통합 (DF-01).

**Override policy** (Q4-B): 사용자가 다른 파일명 요청 시 경고 후 진행. 단, 같은 entity 내에서는 일관성 유지.

**Do**:

```
entities/{entity-name}/
├── api.ts
├── queries.ts
├── hooks.ts
├── types.ts
├── schema.ts
├── columns.tsx
└── index.ts
```

**Don't**:

```
entities/{entity-name}/
├── productApi.ts          # ❌ entity명을 파일명에 반복
├── useProductQuery.ts     # ❌ 훅을 개별 파일로 분리
├── ProductTypes.ts        # ❌ PascalCase 파일명
├── query-keys.ts          # ❌ key를 별도 파일로 분리 — queries.ts에 통합 (DF-01)
└── index.ts
```

**Why**: 파일명이 고정되면 어떤 entity든 동일한 구조를 가지므로, 새 entity 추가 시 복사-수정이 단순해진다. AI 일관성 ↑.

---

## PS-05 Barrel Export 패턴 (🚫 MUST)

**적용 위치**: `entities/*/index.ts`, `features/*/index.ts`

**규칙**: FSD entity/feature 경계에서는 `index.ts`로 public API를 정의한다. `shared/ui`와 `packages/ui`에서는 barrel을 사용하지 않고 직접 파일 경로로 import한다.

**Default**: entity/feature 외부 노출은 `index.ts` barrel을 통해. 내부 직접 import 금지.

**Override policy** (Q4-B): 명시적 요청 시 경고 후 진행. 단, 일관성 유지.

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

## PS-08 @workspace/ Import 스코프 (🚫 MUST)

**규칙**: 모노레포 패키지는 반드시 `@workspace/` 스코프로 import한다. `@repo/`는 사용 금지.

**Default**: 모노레포 internal 패키지는 `@workspace/{package-name}/...`. 앱 내부는 `@/...` (path alias).

**Fallback to skill**: Turborepo skill이 `@workspace/` vs `@repo/` 선택지를 다룰 수 있음. 선택은 우리가 고정 (`@workspace/`).

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

## PS-12 단순 중복 > 과도한 추상화 (⚠️ SHOULD)

> Cross-reference: Clean Code React — Coupling: Handling Duplicate Code

**규칙**: entity별 `api.ts`, `hooks.ts`, `columns.tsx` 등의 구조가 비슷하더라도, 범용 추상화를 만들지 않는다. 3곳 이상에서 **정확히 동일한** 패턴이 반복될 때만 유틸리티로 추출한다.

**Default**: 같은 모양의 코드가 2~3개 entity에 반복되어도 그대로 둔다. 추출 trigger는 "3곳 이상 + 정확히 동일 + 10줄 이상".

**Why this choice**: 과도한 추상화는 설정 파라미터가 증가하고, 한 곳의 변경이 모든 entity에 영향을 미친다. entity별 독립적 코드는 수정 범위가 명확하고 이해하기 쉽다.

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
