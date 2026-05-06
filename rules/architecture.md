# Architecture

> **FSD 위임**: FSD 레이어 구조, import 방향, slice 정의, widget/feature 정의, **entity slice 파일 구성**은 공식 [`feature-sliced-design`](https://skills.sh/feature-sliced/skills/feature-sliced-design) skill에 **전적 위임**한다. 이 파일은 그 위에 우리 harness가 추가로 고정한 컨벤션만 다룬다.
>
> **삭제된 규칙**:
> - PS-01 (FSD 레이어 구조), PS-02 (import 하향 방향), PS-06 (widget = 조합), PS-07 (feature = 사용자 액션) — `feature-sliced-design` skill에서 다룸 (v1.1.0).
> - **PS-03 (entity slice 파일 고정명) — v2.0.0에서 제거**. 기존에는 `api.ts`/`hooks.ts`/`types.ts` 같은 technical-role 파일명을 의도적으로 강제했으나, FSD v2.1 Rule 4-4(도메인 기반 네이밍)와의 정합을 우선하기로 결정 ([ADR-002 Superseded by ADR-021](../DECISIONS.md#adr-021-ps-03-제거--fsd-도메인-기반-네이밍-채택)).

---

## PS-05 Barrel Export 패턴 (🚫 MUST)

**적용 위치**: `entities/*/index.ts`, `features/*/index.ts`

**규칙**: FSD entity/feature 경계에서는 `index.ts`로 public API를 정의한다. `shared/ui`와 `packages/ui`에서는 barrel을 사용하지 않고 직접 파일 경로로 import한다.

**Default**: entity/feature 외부 노출은 `index.ts` barrel을 통해. 내부 직접 import 금지.

**Override policy** (Q4-B): 명시적 요청 시 경고 후 진행. 단, 일관성 유지.

**Do**:

```ts
// entities/{entity}/index.ts — entity의 public API 정의
export { productApi } from './api/product'
export { productKeys, productQueries } from './api/product-queries'
export {
  useCreateProduct,
  useUpdateProduct,
  useDeleteProduct,
} from './api/product-mutations'
export { productColumns } from './ui/product-columns'
export type { Product, ProductListParams, CreateProductInput } from './model/product'
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
import { productApi } from '@/entities/product/api/product'
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

**규칙**: entity별 `api/`, `model/`, `ui/` 세그먼트 구조가 비슷하더라도, 범용 추상화를 만들지 않는다. 3곳 이상에서 **정확히 동일한** 패턴이 반복될 때만 유틸리티로 추출한다.

**Default**: 같은 모양의 코드가 2~3개 entity에 반복되어도 그대로 둔다. 추출 trigger는 "3곳 이상 + 정확히 동일 + 10줄 이상".

**Why this choice**: 과도한 추상화는 설정 파라미터가 증가하고, 한 곳의 변경이 모든 entity에 영향을 미친다. entity별 독립적 코드는 수정 범위가 명확하고 이해하기 쉽다.

**Do**:

```ts
// 각 entity의 api/{entity}.ts가 비슷한 구조를 반복 — 이것은 의도적 설계
// entities/product/api/product.ts
export const productApi = {
  getList: (params) => { const query = new URLSearchParams(); ... return clientFetch(...) },
  create: (input) => clientFetch(...),
}

// entities/category/api/category.ts
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
