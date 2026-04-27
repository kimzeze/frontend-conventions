# Frontend Conventions Index

> AI 에이전트가 참조하는 73개 규칙의 요약 + 상세 파일 링크.

## Entity 구조 템플릿

새 entity를 만들 때 아래 구조 (PS-03):

```
entities/{entity-name}/
├── api.ts           # API 함수 (DF-08)
├── queries.ts       # queryOptions factory + key factory (DF-01)
├── hooks.ts         # Mutation 훅 (useApiMutation, DF-07)
├── types.ts         # 도메인 타입
├── schema.ts        # Zod 스키마 + FormValues (폼이 있을 때, FM-01)
├── columns.tsx      # 테이블 컬럼 정의 (테이블이 있을 때, TB-02)
└── index.ts         # Barrel export (PS-05)
```

## 작업별 참조 가이드

| 작업 | 읽어야 할 규칙 파일 |
|------|---------------------|
| 새 entity 추가 | architecture (PS-03) → api-layer → queries → mutations → naming |
| CRUD 폼 구현 | library-choices (LIB-01) → forms → mutations |
| 테이블 페이지 | library-choices (LIB-03, LIB-04) → tables → state-management → queries |
| 새 페이지/라우트 | nextjs-patterns → architecture (PS-05) |
| 데이터 페칭 영역 | **library-choices (LIB-02)** → queries → mutations |
| 전역 상태 도입 | **state-management (SM-01)** → SM-10~13 (Zustand BP) |
| 에러 처리 | error-handling |
| 성능 최적화 | code-quality → queries (DF-10~12) → state-management (SM-08, 09) |

---

## 전체 규칙 인덱스

### ⭐ Library Choices — [rules/library-choices.md](rules/library-choices.md)

라이브러리 선택 lock-in. 다른 BP skill이 보여주는 여러 옵션 중 하나만 고정.

| ID | 규칙 | 엄격도 |
|----|------|--------|
| LIB-01 | 폼: react-hook-form + zod | 🚫 MUST |
| LIB-02 | 데이터 페칭: TanStack Query v5 (useEffect+fetch 절대 금지) | 🚫 MUST |
| LIB-03 | URL 상태: nuqs (useSearchParams 직접 사용 금지) | 🚫 MUST |
| LIB-04 | 테이블: TanStack Table v8 + DataTable 래퍼 | 🚫 MUST |
| LIB-05 | Toast: sonner | 🚫 MUST |
| LIB-06 | UI: shadcn/ui + radix + Tailwind v4 | 🚫 MUST |

### Architecture — [rules/architecture.md](rules/architecture.md)

> FSD 일반(레이어/import/slice/widget/feature 정의)은 [`feature-sliced-design`](https://skills.sh/feature-sliced/skills/feature-sliced-design) skill에 전적 위임. 이 카테고리는 그 위에 우리 harness가 추가 고정한 컨벤션.

| ID | 규칙 | 엄격도 |
|----|------|--------|
| PS-03 | Entity slice 파일 고정명 (FSD 안티패턴 의도적 override) | 🚫 MUST |
| PS-05 | Barrel export 패턴 (FSD 경계에서만) | 🚫 MUST |
| PS-08 | @workspace/ import 스코프 | 🚫 MUST |
| PS-12 | 단순 중복 > 과도한 추상화 | ⚠️ SHOULD |

> **삭제됨** (`feature-sliced-design` skill 위임): PS-01 (FSD 5 layers), PS-02 (import 하향 방향), PS-06 (widget 정의), PS-07 (feature 정의).

### Next.js Patterns — [rules/nextjs-patterns.md](rules/nextjs-patterns.md)

| ID | 규칙 | 엄격도 |
|----|------|--------|
| PS-04 | Page는 thin shell (비즈니스 로직 금지) | 🚫 MUST |
| PS-11 | 'use client' 배치 기준 (FSD 레이어별) | ⚠️ SHOULD |
| PS-13 | App Providers 패턴 (중첩 순서, Toaster) | ⚠️ SHOULD |
| PS-14 | Next.js 특수 파일 (error, not-found, loading) | ⚠️ SHOULD |

### Code Quality — [rules/code-quality.md](rules/code-quality.md)

| ID | 규칙 | 엄격도 |
|----|------|--------|
| PS-09 | Dynamic import (next/dynamic) | ⚠️ SHOULD |
| PS-10 | Conditional rendering (삼항 연산자) | ⚠️ SHOULD |
| PS-15 | Early return (guard clause) | ⚠️ SHOULD |
| PS-16 | forwardRef 금지 (React 19) | ⚠️ SHOULD |

### Queries — [rules/queries.md](rules/queries.md)

TanStack Query v5 읽기 쿼리.

| ID | 규칙 | 엄격도 |
|----|------|--------|
| DF-01 | queryOptions factory (queries.ts 통합) | 🚫 MUST |
| DF-02 | Query key에 모든 의존성 포함 | 🚫 MUST |
| DF-03 | useQuery 훅 구조 (keepPreviousData) | ⚠️ SHOULD |
| DF-04 | placeholderData vs initialData 구분 | ⚠️ SHOULD |
| DF-05 | select 옵션으로 데이터 변환 | ⚠️ SHOULD |
| DF-10 | limit/offset 변환 유틸리티 | ⚠️ SHOULD |
| DF-11 | staleTime 데이터 변동성 기반 전략 | ⚠️ SHOULD |
| DF-12 | Intent prefetch (hover/focus) | ✅ MAY |
| DF-14 | useQueries 동적 병렬 쿼리 | ✅ MAY |
| DF-15 | Infinite query (무한 스크롤) | ✅ MAY |

### Mutations — [rules/mutations.md](rules/mutations.md)

| ID | 규칙 | 엄격도 |
|----|------|--------|
| DF-06 | Mutation: invalidate (entity) + toast (feature) | 🚫 MUST |
| DF-07 | useApiMutation 래퍼 강제 | 🚫 MUST |
| DF-13 | Optimistic updates (예측 가능한 액션) | ✅ MAY |

### API Layer — [rules/api-layer.md](rules/api-layer.md)

| ID | 규칙 | 엄격도 |
|----|------|--------|
| DF-08 | API 객체 구조 (entityApi = { ... }) | 🚫 MUST |
| DF-09 | PaginatedResponse\<T\> 타입 사용 | 🚫 MUST |
| DF-16 | SSR Prefetch (HydrationBoundary) | ✅ MAY |

### Forms — [rules/forms.md](rules/forms.md)

| ID | 규칙 | 엄격도 |
|----|------|--------|
| FM-01 | Zod 스키마는 schema.ts에 위치 | 🚫 MUST |
| FM-02 | FormValues = z.infer 타입 추출 | 🚫 MUST |
| FM-03 | useForm + zodResolver 패턴 | 🚫 MUST |
| FM-04 | FormDialog 컴포넌트 사용 | ⚠️ SHOULD |
| FM-05 | Form field 컴포넌트 분리 | ✅ MAY |
| FM-06 | Edit 폼: 기존 데이터로 defaultValues + PATCH | ⚠️ SHOULD |
| FM-07 | Create/Edit 스키마 통합 (as never 금지) | ⚠️ SHOULD |

### Tables — [rules/tables.md](rules/tables.md)

| ID | 규칙 | 엄격도 |
|----|------|--------|
| TB-01 | DataTable props 구조 (서버사이드) | 🚫 MUST |
| TB-02 | Column 정의: SortableHeader + Badge + size | ⚠️ SHOULD |
| TB-03 | 서버사이드 pagination/sorting (manual: true) | 🚫 MUST |
| TB-04 | DataTableToolbar (검색 + 액션) | ⚠️ SHOULD |
| TB-05 | Widget에서 테이블 오케스트레이션 | ⚠️ SHOULD |

### State Management — [rules/state-management.md](rules/state-management.md)

| ID | 규칙 | 엄격도 |
|----|------|--------|
| SM-01 | 상태 4-카테고리 분류 (서버/URL/로컬/Zustand) | 🚫 MUST |
| SM-02 | URL 상태는 nuqs로 관리 | 🚫 MUST |
| SM-03 | useTableState 훅 패턴 | ⚠️ SHOULD |
| SM-04 | Dialog 상태: useDialogState | ⚠️ SHOULD |
| SM-05 | Lazy state initialization | ⚠️ SHOULD |
| SM-06 | Functional setState (stale closure 방지) | ⚠️ SHOULD |
| SM-07 | useEffect 의존성 최적화 (원시값 사용) | ⚠️ SHOULD |
| SM-08 | startTransition (비긴급 업데이트) | ✅ MAY |
| SM-09 | Deferred state reads | ✅ MAY |
| SM-10 | Zustand: custom hook만 export | 🚫 MUST |
| SM-11 | Zustand: atomic selector | 🚫 MUST |
| SM-12 | Zustand: actions namespace 분리 | ⚠️ SHOULD |
| SM-13 | Zustand: action을 domain event로 명명 | ⚠️ SHOULD |

### Error Handling — [rules/error-handling.md](rules/error-handling.md)

| ID | 규칙 | 엄격도 |
|----|------|--------|
| EH-01 | ApiError 클래스 활용 | 🚫 MUST |
| EH-02 | Mutation 에러 표시 전략 (toast/inline/silent) | ⚠️ SHOULD |
| EH-03 | Auth 에러 cache level 자동 처리 | 🚫 MUST |
| EH-04 | 에러 메시지는 한국어 | ⚠️ SHOULD |
| EH-05 | Error Boundary + useQueryErrorResetBoundary | ⚠️ SHOULD |
| EH-06 | localStorage try-catch | ✅ MAY |

### Naming — [rules/naming.md](rules/naming.md)

| ID | 규칙 | 엄격도 |
|----|------|--------|
| NM-01 | 파일명: kebab-case + entity 고정명 | 🚫 MUST |
| NM-02 | Hook: use{Entity}{Operation} | 🚫 MUST |
| NM-03 | 타입: {Entity}, Create{Entity}Input | 🚫 MUST |
| NM-04 | 상수: UPPER_SNAKE_CASE + {ENTITY}\_{FIELD} 접두사 | ⚠️ SHOULD |
| NM-05 | Query key: {entity}Keys (camelCase 복수) | 🚫 MUST |
| NM-06 | 컴포넌트: {Action}{Entity}Dialog | ⚠️ SHOULD |
| NM-07 | Type-only import/export (import type) | ⚠️ SHOULD |
| NM-08 | Import 순서 (React→외부→@workspace→@/→상대→type) | ⚠️ SHOULD |

---

## 통계

| 항목 | 수 |
|------|-----|
| 총 규칙 수 | **73** |
| 🚫 MUST | **31** |
| ⚠️ SHOULD | **33** |
| ✅ MAY | **9** |
| 규칙 파일 | **12** |

## Cross-reference 출처

- [feature-sliced-design skill](https://skills.sh/feature-sliced/skills/feature-sliced-design) — 아키텍처 위임
- [Vercel React Best Practices](https://vercel.com/blog/react-best-practices) — bundle, rendering, re-render 최적화
- [TanStack Query Best Practices](https://tanstack.com/query/latest/docs) — query key, caching, mutation 패턴
- [TkDodo — Working with Zustand](https://tkdodo.eu/blog/working-with-zustand) — Zustand BP

## 미결 사항

설계 결정 배경 및 Open Questions: [DECISIONS.md](DECISIONS.md).
