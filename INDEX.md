# Frontend Conventions Index

> AI 에이전트가 가장 먼저 읽는 파일. 모든 규칙의 요약과 상세 파일 링크를 제공한다.

## Entity 구조 템플릿

새 entity를 만들 때 아래 구조를 따른다:

```
entities/{entity-name}/
├── api.ts           # API 함수
├── queries.ts       # queryOptions factory + key factory (DF-01)
├── hooks.ts         # Mutation 훅 (useApiMutation)
├── types.ts         # 도메인 타입
├── schema.ts        # Zod 스키마 + FormValues (폼이 있을 때)
├── columns.tsx      # 테이블 컬럼 정의 (테이블이 있을 때)
└── index.ts         # Barrel export
```

## 작업별 참조 가이드

| 작업 | 읽어야 할 규칙 파일 |
|------|---------------------|
| 새 entity 추가 | architecture → api-layer → queries → mutations → naming |
| CRUD 폼 구현 | forms → mutations (DF-06, DF-07) → naming |
| 테이블 페이지 구현 | tables → state-management → queries |
| 새 페이지/라우트 추가 | nextjs-patterns → architecture (PS-02, PS-05) |
| 에러 처리 | error-handling |
| 성능 최적화 | code-quality → queries (DF-10~12) |
| 기존 코드 리팩토링 | 해당 영역의 규칙 파일 전체 |

---

## 전체 규칙 인덱스

### Architecture — [rules/architecture.md](rules/architecture.md)

FSD 레이어 구성, import 방향, 파일 배치, 추상화 수준.

| ID | 규칙 | 엄격도 |
|----|------|--------|
| PS-01 | FSD 레이어 구조 (app/widgets/features/entities/shared) | 🚫 MUST |
| PS-02 | Import 방향: 하향만 허용 | 🚫 MUST |
| PS-03 | Entity slice 파일 구성 | 🚫 MUST |
| PS-05 | Barrel export 패턴 (FSD 경계에서만) | 🚫 MUST |
| PS-06 | Widget = 테이블+상태+다이얼로그 조합 | ⚠️ SHOULD |
| PS-07 | Feature = 사용자 액션 단위 | ⚠️ SHOULD |
| PS-08 | @workspace/ import 스코프 | 🚫 MUST |
| PS-12 | 단순 중복 > 과도한 추상화 | ⚠️ SHOULD |

### Next.js Patterns — [rules/nextjs-patterns.md](rules/nextjs-patterns.md)

Next.js 16 App Router 특화 — Page, Providers, 'use client', 특수 파일.

| ID | 규칙 | 엄격도 |
|----|------|--------|
| PS-04 | Page는 thin shell (비즈니스 로직 금지) | 🚫 MUST |
| PS-11 | 'use client' 배치 기준 (FSD 레이어별) | ⚠️ SHOULD |
| PS-13 | App Providers 패턴 (중첩 순서, Toaster) | ⚠️ SHOULD |
| PS-14 | Next.js 특수 파일 (error, not-found, loading) | ⚠️ SHOULD |

### Code Quality — [rules/code-quality.md](rules/code-quality.md)

코드 품질 & 성능 — Dynamic Import, Conditional, Early Return, forwardRef.

| ID | 규칙 | 엄격도 |
|----|------|--------|
| PS-09 | Dynamic import (next/dynamic) | ⚠️ SHOULD |
| PS-10 | Conditional rendering (삼항 연산자) | ⚠️ SHOULD |
| PS-15 | Early return (guard clause) | ⚠️ SHOULD |
| PS-16 | forwardRef 금지 (React 19) | ⚠️ SHOULD |

### Queries — [rules/queries.md](rules/queries.md)

TanStack Query v5 읽기 쿼리 — queryOptions, query keys, staleTime, prefetch.

| ID | 규칙 | 엄격도 |
|----|------|--------|
| DF-01 | queryOptions factory (queries.ts) | 🚫 MUST |
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

쓰기 mutation — invalidate, useApiMutation, optimistic.

| ID | 규칙 | 엄격도 |
|----|------|--------|
| DF-06 | Mutation: invalidate + toast on success | 🚫 MUST |
| DF-07 | useApiMutation 래퍼 사용 | 🚫 MUST |
| DF-13 | Optimistic updates (예측 가능한 액션) | ✅ MAY |

### API Layer — [rules/api-layer.md](rules/api-layer.md)

API 함수 구조 + 응답 타입 + SSR.

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
| SM-01 | 상태 카테고리 분류 (서버/URL/로컬) | 🚫 MUST |
| SM-02 | URL 상태는 nuqs로 관리 | 🚫 MUST |
| SM-03 | useTableState 훅 패턴 | ⚠️ SHOULD |
| SM-04 | Dialog 상태: useDialogState | ⚠️ SHOULD |
| SM-05 | Lazy state initialization | ⚠️ SHOULD |
| SM-06 | Functional setState (stale closure 방지) | ⚠️ SHOULD |
| SM-07 | useEffect 의존성 최적화 (원시값 사용) | ⚠️ SHOULD |
| SM-08 | startTransition (비긴급 업데이트) | ✅ MAY |
| SM-09 | Deferred state reads | ✅ MAY |

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
| 총 규칙 수 | 67 |
| 🚫 MUST | 25 |
| ⚠️ SHOULD | 33 |
| ✅ MAY | 9 |
| 규칙 파일 | 11 |

## Cross-reference 출처

- [Vercel React Best Practices](https://vercel.com/blog/react-best-practices) — bundle, rendering, re-render 최적화
- [TanStack Query Best Practices](https://tanstack.com/query/latest/docs) — query key, caching, mutation 패턴

## 미결 사항

설계 결정 배경 및 Open Questions는 [DECISIONS.md](DECISIONS.md) 참조.
