# Tables

TanStack Table v8 기반 데이터 테이블 규칙. 서버사이드 처리 전용.

---

## TB-01 DataTable — Table Instance 방식 (🚫 MUST)

**규칙**: DataTable은 `useReactTable`로 생성한 table instance를 받는다. 서버사이드 모드(manual)로만 운영. 이는 TanStack Table 공식 문서 권장 패턴.

**Do**:

```tsx
// widget에서 table instance 생성
const table = useReactTable({
  data: data?.results ?? [],
  columns: productColumns,
  pageCount: Math.ceil((data?.count ?? 0) / limit),
  state: { sorting },
  onSortingChange: handleSortingChange,
  manualPagination: true,
  manualSorting: true,
  manualFiltering: true,
  getCoreRowModel: getCoreRowModel(),
})

// DataTable은 table instance를 받음
<DataTable table={table} isLoading={isLoading} onRowClick={(row) => dialog.edit.open(row.original)} />
```

**DataTable Props**:

| Prop | 타입 | 설명 |
|------|------|------|
| `table` | `Table<T>` | useReactTable 인스턴스 |
| `isLoading` | `boolean` | 로딩 상태 |
| `onRowClick?` | `(row: Row<T>) => void` | 행 클릭 핸들러 |
| `emptyMessage?` | `string` | 데이터 없을 때 메시지 |

**Don't**:

```tsx
// ❌ 클라이언트 사이드 처리
const table = useReactTable({
  data: allData,
  getSortedRowModel: getSortedRowModel(),     // ❌
  getPaginationRowModel: getPaginationRowModel(), // ❌
})
```

**Why**: table instance 방식은 TanStack Table 공식 권장. DataTable 컴포넌트가 table 설정을 몰라도 되므로 유연성이 높고, row selection 등 고급 기능 추가가 용이.

---

## TB-02 Column 정의 패턴 (⚠️ SHOULD)

**적용 위치**: `entities/*/columns.tsx`

**규칙**: 컬럼은 `entities/{entity}/columns.tsx`에 정의. 정렬 가능한 컬럼은 `SortableHeader`, enum은 `Badge`, 날짜는 포맷팅.

**Do**:

```tsx
// entities/{entity}/columns.tsx
export const productColumns: ColumnDef<Product>[] = [
  {
    accessorKey: 'id',
    header: ({ column }) => <SortableHeader column={column}>ID</SortableHeader>,
    size: 70,
  },
  {
    accessorKey: 'name',
    header: ({ column }) => <SortableHeader column={column}>이름</SortableHeader>,
    size: 200,
  },
  {
    accessorKey: 'status',
    header: '상태',
    cell: ({ row }) => {
      const status = row.getValue<ProductStatus>('status')
      return (
        <Badge variant={STATUS_BADGE_VARIANTS[status]}>
          {STATUS_LABELS[status]}
        </Badge>
      )
    },
    size: 100,
  },
  {
    accessorKey: 'created_at',
    header: ({ column }) => <SortableHeader column={column}>등록일</SortableHeader>,
    cell: ({ row }) => new Date(row.getValue('created_at')).toLocaleDateString('ko-KR'),
    size: 120,
  },
]
```

**팩토리 함수 패턴** (외부 데이터 의존 시):

```tsx
// 외부 맵이 필요한 경우 — 팩토리 함수로 정의
export function createProductColumns(
  categoryMap: Map<number, string>
): ColumnDef<Product>[] {
  return [
    { accessorKey: 'id', header: ({ column }) => <SortableHeader column={column}>ID</SortableHeader>, size: 70 },
    {
      accessorKey: 'category_id',
      header: '카테고리',
      cell: ({ row }) => categoryMap.get(row.original.category_id) ?? '-',
      size: 150,
    },
    // ...
  ]
}

// widget에서 useMemo와 조합
const columns = useMemo(() => createProductColumns(categoryMap), [categoryMap])
```

**네이밍**: 상수는 `{entity}Columns`, 함수는 `create{Entity}Columns`

**Cell 패턴 가이드**:

| 패턴 | 코드 예시 | 사용 시점 |
|------|----------|----------|
| SortableHeader | `<SortableHeader column={column}>이름</SortableHeader>` | 정렬 가능 컬럼 |
| Badge + Label 맵 | `<Badge variant={VARIANTS[status]}>{LABELS[status]}</Badge>` | enum/status |
| 날짜 포맷 | `new Date(value).toLocaleDateString('ko-KR')` | 날짜 |
| 모노스페이스 | `<span className="font-mono text-xs">{code}</span>` | 코드/ID |
| null 대체 | `value ?? <span className="text-muted-foreground">없음</span>` | 선택적 필드 |
| 조건부 포맷 | `type === 'PERCENT' ? \`${v}%\` : \`${v}원\`` | 동적 포맷 |

**체크리스트**:
- [ ] 정렬 가능한 컬럼에 `SortableHeader` 사용
- [ ] enum/status 값에 `Badge` + label 맵
- [ ] 날짜에 `toLocaleDateString('ko-KR')` 포맷
- [ ] 모든 컬럼에 `size` 지정
- [ ] null/undefined 필드에 대체 표시

---

## TB-03 서버사이드 Pagination/Sorting (🚫 MUST)

**규칙**: TanStack Table을 `manual` 모드로 사용한다.

```tsx
const table = useReactTable({
  data,
  columns,
  pageCount,
  state: { sorting },
  onSortingChange,
  manualPagination: true,
  manualSorting: true,
  manualFiltering: true,
  getCoreRowModel: getCoreRowModel(),
})
```

**Don't**:

```tsx
// ❌ manual 모드에서 클라이언트 모델은 불필요 (포함해도 무시되지만, 혼동 방지를 위해 제거)
getSortedRowModel: getSortedRowModel()       // manualSorting: true이면 불필요
getPaginationRowModel: getPaginationRowModel() // manualPagination: true이면 불필요
getFilteredRowModel: getFilteredRowModel()     // manualFiltering: true이면 불필요
```

---

## TB-04 DataTableToolbar (⚠️ SHOULD)

**규칙**: 테이블 상단에 검색 + 액션 버튼을 배치한다.

```tsx
<DataTableToolbar
  searchValue={searchValue}
  onSearchChange={handleSearchChange}
  searchPlaceholder="검색..."
>
  <Button onClick={dialog.create.open}>등록</Button>
</DataTableToolbar>
```

**구성**: 좌측 = 검색 (debounced 300ms), 우측 = 액션 버튼 (children)

---

## TB-05 Widget 테이블 오케스트레이션 (⚠️ SHOULD)

**규칙**: Widget은 테이블의 모든 상태를 조합하는 컨테이너이다.

**패턴**:

```tsx
// widgets/{entity}-table.tsx
'use client'

export function ProductTable() {
  // 1. URL 상태
  const {
    state: { q, page, setPage, limit },
    debouncedQ, offset, sorting,
    handleSearchChange, handlePageSizeChange, handleSortingChange,
  } = useTableState()

  // 2. 데이터
  const { data, isLoading } = useQuery(productQueries.list({ limit, offset, search: debouncedQ, ordering: sorting }))
  const pageCount = Math.ceil((data?.count ?? 0) / limit)

  // 3. Table instance (TB-01)
  const table = useReactTable({
    data: data?.results ?? [],
    columns: productColumns,
    pageCount,
    state: { sorting },
    onSortingChange: handleSortingChange,
    manualPagination: true,
    manualSorting: true,
    getCoreRowModel: getCoreRowModel(),
  })

  // 4. 다이얼로그
  const dialog = useDialogState<Product>()

  return (
    <>
      <DataTableToolbar searchValue={q} onSearchChange={handleSearchChange} searchPlaceholder="검색...">
        <Button onClick={dialog.create.open}>등록</Button>
      </DataTableToolbar>
      <DataTable table={table} isLoading={isLoading} onRowClick={(row) => dialog.edit.open(row.original)} />
      <DataTablePagination page={page} pageCount={pageCount} totalCount={data?.count ?? 0} pageSize={limit} onPageChange={setPage} onPageSizeChange={handlePageSizeChange} />
      <CreateProductDialog open={dialog.create.isOpen} onOpenChange={dialog.create.setOpen} />
      {dialog.edit.selected && <EditProductDialog open={dialog.edit.isOpen} onOpenChange={dialog.edit.setOpen} item={dialog.edit.selected} />}
    </>
  )
}
```

**흐름**: `useTableState() → useEntityList() → useDialogState() → UI 조합`
