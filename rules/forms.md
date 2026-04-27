# Forms

zod + react-hook-form + shadcn Form 기반 폼 처리 규칙. 라이브러리 선택은 [LIB-01](library-choices.md#lib-01-폼-라이브러리-must) lock-in.

> **Override Policy**: 이 파일의 모든 🚫 MUST 규칙은 [SKILL.md Override Policy (Q4-B)](../SKILL.md#override-policy-q4-b) 적용 — 사용자 명시 요청 시 경고 후 진행.

---

## FM-01 Zod 스키마 위치와 구조 (🚫 MUST)

**적용 위치**: `entities/*/schema.ts`

**규칙**: Zod 스키마는 `entities/{entity}/schema.ts`에 정의한다. 한 파일에 해당 entity의 모든 스키마를 모은다.

**Do**:

```ts
// entities/{entity}/schema.ts
import { z } from 'zod'

export const createProductSchema = z.object({
  name: z.string().min(1, '이름을 입력하세요'),
  price: z.coerce.number({ required_error: '가격을 입력하세요' }).min(0),
  category_id: z.coerce.number({ required_error: '카테고리를 선택하세요' }).min(1),
  description: z.string().optional(),
})

export type CreateProductFormValues = z.infer<typeof createProductSchema>

// edit 스키마 — create 기반으로 확장/수정
export const editProductSchema = createProductSchema.partial().extend({
  name: z.string().min(1, '이름을 입력하세요'), // 필수 필드 유지
})

export type EditProductFormValues = z.infer<typeof editProductSchema>
```

**Don't**:

```ts
// ❌ feature 파일에 스키마 정의
// features/{entity}/ui/CreateDialog.tsx 안에 두지 않는다

// ❌ 영문 validation 메시지
z.string().min(1, 'Name is required')

// ❌ HTML input에서 z.number() 직접 사용
z.number().min(1)  // HTML input은 항상 string 반환 → z.coerce.number() 사용
```

**Tips**:
- HTML form input은 항상 string → 숫자 필드에는 `z.coerce.number()`
- Validation 메시지는 한국어
- `required_error`로 빈 값 에러 메시지 커스텀

**Why**: 스키마를 entity에 위치시키면 도메인 규칙이 한 곳에 모인다.

---

## FM-02 FormValues 타입 추출 (🚫 MUST)

**규칙**: 폼 값 타입은 반드시 `z.infer<typeof schema>`로 추출한다. 수동 타입 정의를 금지.

**Do**:

```ts
export const createProductSchema = z.object({ name: z.string(), price: z.coerce.number() })
export type CreateProductFormValues = z.infer<typeof createProductSchema>
```

**Don't**:

```ts
// ❌ 수동 타입 — 스키마와 동기화 안 될 위험
export type CreateProductFormValues = { name: string; price: number }
```

**네이밍**: `{Action}{Entity}FormValues` — API input 타입(`{Action}{Entity}Input`)과 구분.

**Why**: 스키마와 타입이 항상 동기화되어 런타임 validation과 컴파일 타임 타입 검사가 일치한다.

---

## FM-03 useForm + zodResolver 패턴 (🚫 MUST)

**적용 위치**: `features/*/hooks/useCreate*Form.ts`, `features/*/hooks/useEdit*Form.ts`

**규칙**: 모든 폼은 `zodResolver`를 사용하며 `defaultValues`를 반드시 제공한다. 폼 로직은 `features/{entity}/hooks/`에 커스텀 훅으로 추출.

**Do**:

```ts
// features/{entity}/hooks/useCreate{Entity}Form.ts
export const useCreateProductForm = () => {
  const { mutate: createProduct, isPending } = useCreateProduct()

  const form = useForm<CreateProductFormValues>({
    resolver: zodResolver(createProductSchema),
    defaultValues: {
      name: '',
      price: 0,
      description: '',
    },
  })

  const submit = (values: CreateProductFormValues, onSuccess: () => void) => {
    createProduct(values, {
      onSuccess: () => {
        form.reset()
        onSuccess()
      },
    })
  }

  return { form, submit, isPending }
}
```

**Don't**:

```ts
// ❌ defaultValues 누락 → uncontrolled→controlled 경고
useForm<CreateProductFormValues>({ resolver: zodResolver(createProductSchema) })

// ❌ 컴포넌트에 useForm 직접 호출
function CreateProductDialog() {
  const form = useForm({ ... })  // 여기 두지 않는다
}
```

**Why**: `zodResolver`는 스키마 기반 validation을 연결. `defaultValues`는 controlled component 원칙. 커스텀 훅으로 UI와 로직 분리.

---

## FM-04 FormDialog 컴포넌트 사용 (⚠️ SHOULD)

**규칙**: create/edit 다이얼로그는 공유 `FormDialog` 컴포넌트를 사용한다.

**Do**:

```tsx
// features/{entity}/ui/Create{Entity}Dialog.tsx
export function CreateProductDialog({ open, onOpenChange }: DialogProps) {
  const { form, submit, isPending } = useCreateProductForm()

  return (
    <FormDialog
      open={open}
      onOpenChange={onOpenChange}
      title="상품 등록"
      description="새로운 상품 정보를 입력하세요."
      form={form}
      onSubmit={(values) => submit(values, () => onOpenChange(false))}
      isPending={isPending}
    >
      <ProductFormFields control={form.control} />
    </FormDialog>
  )
}
```

**Don't**:

```tsx
// ❌ Dialog + Form 보일러플레이트 매번 반복
<Dialog><DialogContent><Form><form onSubmit={...}>...</form></Form></DialogContent></Dialog>
```

**Why**: FormDialog는 Dialog + Form 조합의 보일러플레이트를 제거하고 일관된 레이아웃을 보장한다.

---

## FM-05 Form Field 컴포넌트 분리 (✅ MAY)

**규칙**: create와 edit에서 동일한 필드를 사용할 때, form field를 별도 컴포넌트로 추출한다.

**Do**:

```tsx
// features/{entity}/ui/{entity}-form-fields.tsx
interface ProductFormFieldsProps {
  control: Control<CreateProductFormValues>
}

export function ProductFormFields({ control }: ProductFormFieldsProps) {
  return (
    <div className="space-y-4">
      <FormField
        control={control}
        name="name"
        render={({ field }) => (
          <FormItem>
            <FormLabel>이름</FormLabel>
            <FormControl><Input placeholder="이름을 입력하세요" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )}
      />
    </div>
  )
}
```

**When**: create와 edit가 같은 필드를 공유할 때만 추출. 한 곳에서만 사용하면 분리 불필요.

---

## FM-06 Edit 폼 패턴 (⚠️ SHOULD)

**규칙**: Edit 폼은 기존 entity 데이터를 `defaultValues`로 주입하며, PATCH로 부분 업데이트한다.

**Do**:

```ts
export const useEditProductForm = (product: Product) => {
  const { mutate: updateProduct, isPending } = useUpdateProduct()

  const form = useForm<EditProductFormValues>({
    resolver: zodResolver(editProductSchema),
    defaultValues: {
      name: product.name,
      price: product.price,
      description: product.description ?? '',
    },
  })

  const submit = (values: EditProductFormValues, onSuccess: () => void) => {
    updateProduct({ id: product.id, ...values }, { onSuccess })
  }

  return { form, submit, isPending }
}
```

**Don't**:

```ts
// ❌ PUT으로 전체 교체 (변경 안 된 필드까지 전송)
clientFetch(`/product/${id}/`, { method: 'PUT', body: values })
```

**Why**: PATCH는 변경된 필드만 전송하여 의도치 않은 데이터 덮어쓰기를 방지한다.

---

## FM-07 Create/Edit 스키마 통합 (⚠️ SHOULD)

**규칙**: create와 edit에서 동일한 form field를 공유할 때, 스키마를 통합하여 `as never` 타입 단언을 방지한다.

**Do**:

```ts
// schema.ts — 기본 스키마 + action별 확장
const baseProductSchema = z.object({
  name: z.string().min(1, '이름을 입력하세요'),
  price: z.coerce.number().min(0),
  description: z.string().optional(),
})

export const createProductSchema = baseProductSchema
export type CreateProductFormValues = z.infer<typeof createProductSchema>

export const editProductSchema = baseProductSchema.partial().extend({
  name: z.string().min(1, '이름을 입력하세요'), // 필수 필드 유지
})
export type EditProductFormValues = z.infer<typeof editProductSchema>
```

```tsx
// form field — 공유 가능 (타입 호환)
interface ProductFormFieldsProps<T extends CreateProductFormValues> {
  control: Control<T>
}
```

**Don't**:

```tsx
// ❌ as never로 타입 안전성 우회
<FormField control={control} name={'name' as never} render={...} />
```

**Why**: `as never`는 타입 시스템을 완전히 우회하여 name 오타도 감지하지 못한다. 기본 스키마를 공유하면 create/edit의 form field가 타입 안전하게 공유된다.
