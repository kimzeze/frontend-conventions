# Design Decisions

> 이 컨벤션의 주요 설계 결정과 그 근거를 기록한 **Architecture Decision Records (ADR)**.
> 각 결정은 [Michael Nygard ADR 템플릿](https://github.com/joelparkerhenderson/architecture-decision-record)을 따릅니다.

## 목적

이 문서는 다음 질문에 답하기 위해 작성되었습니다:

- **왜 이렇게 정했는가?** — 각 규칙의 선택 근거
- **어떤 대안을 고려했는가?** — 거부된 옵션과 그 이유
- **어떤 트레이드오프가 있는가?** — 선택의 결과 (긍정적/부정적)

이 문서는 **추가 전용(append-only)**입니다. 결정이 변경되면 새 ADR을 추가하고 이전 ADR은 "Superseded" 상태로 남깁니다.

---

## 결정 원칙

모든 결정은 다음 메타 원칙을 따릅니다:

1. **좋은 코드 + 우리 코드** — 업계 BP를 따르되, 팀의 일관된 패턴으로 통일
2. **AI 친화적** — AI 에이전트가 즉시 이해하고 적용 가능한 수준의 구체성
3. **단순 중복 > 과도한 추상화** — 3곳 미만 반복은 추상화하지 않음
4. **강제성 계층화** — 🚫 MUST / ⚠️ SHOULD / ✅ MAY로 엄격도 명시
5. **실제 검증 우선** — BP보다 실제 코드베이스에서 동작하는 패턴 우선

---

# Architecture

## ADR-001: FSD (Feature-Sliced Design) 레이어 구조 채택

**Status**: Accepted (v1.0.0)

**Context**
중대형 Next.js 프로젝트에서 관심사 분리와 모듈 경계가 필요. 파일/폴더 구조가 일관되지 않으면 AI 에이전트와 개발자 모두 탐색 비용 증가.

**Decision**
FSD 5레이어 채택: `app` → `widgets` → `features` → `entities` → `shared` (하향 의존성). `src/` 없이 앱 루트에 직접 배치.

**Consequences**
- ✓ 단방향 의존성으로 순환 참조 방지
- ✓ 새 entity 추가 시 예측 가능한 구조 (파일명 고정)
- ✓ AI가 레이어만 보고 역할 판단 가능
- ✗ 작은 프로젝트에서는 오버엔지니어링
- ✗ 초기 학습 곡선 존재

**Alternatives considered**
- **Atomic Design**: UI 중심, 도메인 모델 표현 부족
- **Feature-based flat**: 초기엔 간단하지만 확장성 제한
- **Next.js 기본 (app/ + components/)**: 관심사 분리 부족

**Related**: [PS-01~08, PS-12](./rules/architecture.md)

---

## ADR-002: Entity 파일 구성 고정

**Status**: Accepted (v1.0.0)

**Context**
Entity마다 파일명과 구조가 다르면 AI가 패턴 학습 불가. 새 entity 추가 시 개발자마다 다른 구조 생성.

**Decision**
각 entity는 고정된 7개 파일로 구성: `api.ts`, `queries.ts`, `hooks.ts`, `types.ts`, `schema.ts`(폼 있을 때), `columns.tsx`(테이블 있을 때), `index.ts`.

**Consequences**
- ✓ 한 entity를 다른 entity로 복사-수정이 가능 (AI 친화적)
- ✓ barrel export(`index.ts`)로 외부 API 정의
- ✗ 파일 수가 적은 entity도 항상 5-7개 파일 유지 필요

**Alternatives considered**
- **단일 파일**: 작은 entity에 유리하지만 파일이 커지면 분리 필요
- **자유 구조**: AI 일관성 확보 불가

**Related**: [PS-03, PS-05](./rules/architecture.md)

---

## ADR-003: Entity 간 Cross-Import 금지 (Type-only 포함)

**Status**: Accepted (v1.0.0)

**Context**
FSD 원칙은 "같은 레이어 내 모듈 간 import 금지"이지만, 현실적으로 한 entity가 다른 entity의 타입을 참조하는 경우가 자주 발생 (예: `Product`에서 `Category` 참조).

**Decision**
Entity 간 **type-only import도 금지**. 외부 데이터가 필요하면 widget에서 factory 함수로 주입한다.

**Consequences**
- ✓ 완전한 entity 독립성 (테스트/리팩토링 용이)
- ✓ 순환 의존성 원천 차단
- ✗ `createProductColumns(categoryMap)` 같은 주입 패턴 필요 (boilerplate 증가)

**Alternatives considered**
- **Type-only 허용**: 런타임 영향 없으므로 허용 → FSD 원칙 약화
- **공통 타입 shared/types/로 이동**: 실제로는 도메인 타입이므로 부적절

**Related**: [PS-02](./rules/architecture.md)

---

## ADR-004: Barrel Export (`index.ts`) — FSD 경계에서만

**Status**: Accepted (v1.0.0)

**Context**
Vercel의 `bundle-barrel-imports` BP는 "barrel file 피하라" (번들 크기). 하지만 FSD에서는 entity의 public API를 정의하는 수단으로 필수.

**Decision**
- FSD entity/feature 경계에서는 **barrel export 사용** (캡슐화 우선)
- `shared/ui`, `packages/ui`에서는 **barrel 미사용** (직접 파일 경로)
- Next.js `optimizePackageImports`로 번들 영향 완화

**Consequences**
- ✓ Entity의 public API 명시적 정의 (내부 구현 보호)
- ✓ 외부 코드가 entity 내부에 직접 접근 방지
- ✗ Vercel BP와 부분적 충돌 (경미)

**Alternatives considered**
- **Barrel 완전 제거**: FSD 캡슐화 원칙 위배
- **모든 곳에 barrel 사용**: 번들 크기 영향

**Related**: [PS-05](./rules/architecture.md)

---

## ADR-005: 전역 상태 라이브러리 미사용

**Status**: Accepted (v1.0.0)

**Context**
상태 관리 라이브러리(Redux, zustand, Jotai) 선택은 프로젝트 초기에 중요한 결정. 잘못 선택하면 전역 상태에 서버 데이터 캐싱 같은 안티패턴 발생.

**Decision**
세 가지 상태 카테고리로 분류하고, 각각 다른 도구 사용:

| 카테고리 | 도구 | 예시 |
|---------|------|------|
| 서버 상태 | TanStack Query | 목록, 상세, CRUD |
| URL 상태 | nuqs | 검색, 정렬, 페이지 |
| 로컬 UI 상태 | useState | 다이얼로그, 토글 |

**범용 전역 상태 라이브러리(zustand/Redux)는 사용하지 않음**. Context는 theme/locale 등 앱 설정에만 허용.

**Consequences**
- ✓ 서버 상태는 TanStack Query가 자동 관리 (캐싱/리페치/무효화)
- ✓ URL 상태는 브라우저 히스토리와 자연스럽게 동기화
- ✓ 90%의 상태가 이 3개 카테고리로 커버됨
- ✗ "복잡한 UI 상태"가 필요한 케이스가 생기면 재검토 필요

**Alternatives considered**
- **zustand**: 서버 데이터 캐싱 안티패턴 위험
- **Redux Toolkit**: 과도한 boilerplate
- **Jotai**: 원자 단위 상태가 좋지만 학습 곡선

**Related**: [SM-01, SM-02](./rules/state-management.md)

---

# Data Fetching

## ADR-006: queryOptions Factory 전면 채택

**Status**: Accepted (v1.0.0)

**Context**
TanStack Query v5에서 `queryOptions()` 공식 권장. 기존 key factory + 별도 hooks.ts 분리 방식은 DRY 원칙 위배 (key와 fn이 여러 파일에 분산).

**Decision**
모든 entity에서 **queryOptions factory 패턴** 전면 사용. `entities/*/queries.ts`에 key factory + queryOptions factory 통합.

```ts
export const productQueries = {
  list: (params) => queryOptions({ queryKey, queryFn, staleTime, ... }),
  detail: (id) => queryOptions({ ... }),
}

// 사용: useQuery(productQueries.list(params))
// prefetch에도 동일 객체 사용 가능
```

**Consequences**
- ✓ key + fn + options 한 곳에서 관리 (DRY)
- ✓ prefetch에서 동일 객체 재사용
- ✓ TypeScript 타입 추론 완벽
- ✗ 기존 `query-keys.ts` + `hooks.ts` 분리 구조에서 마이그레이션 필요

**Alternatives considered**
- **v1: key factory만, v2에서 queryOptions**: 점진적 전환의 이점 없음
- **새 entity부터 queryOptions**: 일관성 저하

**Related**: [DF-01~05, DF-10~12](./rules/queries.md)

---

## ADR-007: useApiMutation 강제 (useMutation 금지)

**Status**: Accepted (v1.0.0)

**Context**
Raw `useMutation`을 쓰면 개별 mutation마다 에러 처리, auth 에러 스킵, toast 표시 로직을 중복 작성해야 함.

**Decision**
**모든 mutation은 `useApiMutation` 래퍼 필수 사용** (🚫 MUST). raw `useMutation` 금지.

래퍼는 세 가지 `displayType` 제공:
- `'toast'` (기본): 에러 toast 자동
- `'inline'`: 컴포넌트에 에러 전달 (폼 validation)
- `'silent'`: 에러 무시 (백그라운드 동기화)

**Consequences**
- ✓ 에러 처리 일관성 (모든 mutation이 동일한 방식)
- ✓ auth 에러는 cache level에서 자동 처리, UI 표시 스킵
- ✓ 에러 toast 중복 제거
- ✗ 특수한 에러 처리가 필요한 경우 `displayType: 'silent'`로 우회

**Alternatives considered**
- **SHOULD로 설정**: 일부 entity가 raw `useMutation` 계속 사용 → 일관성 저하
- **에러 boundary로 처리**: Mutation 실패는 에러 boundary까지 전파 안 되므로 부적합

**Related**: [DF-07](./rules/mutations.md)

---

## ADR-008: Toast 호출 위치 — entity/feature 분리

**Status**: Accepted (v1.0.0)

**Context**
Mutation 성공 시 `toast.success()`를 어디서 호출할지 두 가지 패턴 존재:
- **Pattern A**: entity hooks.ts의 `onSuccess`에서 호출 (간단)
- **Pattern B**: entity는 invalidation만, feature hooks가 toast 호출 (관심사 분리)

**Decision**
**Pattern B 채택**: entity hooks는 mutation + invalidation만, feature hooks는 toast + form reset.

**Consequences**
- ✓ 같은 mutation hook을 여러 feature에서 재사용 가능 (각자 다른 toast 메시지)
- ✓ entity는 데이터 계층, feature는 UX 계층 명확히 분리
- ✗ TanStack Query 공식 예제와 다름 (공식은 mutation hook 안에서 모두 처리)

**Alternatives considered**
- **Pattern A (entity에서 모두)**: 재사용성 저하, Mutation hook이 UX 결정에 묶임

**Related**: [DF-06](./rules/mutations.md)

---

## ADR-009: 외부 백엔드 API — Server Action 미사용

**Status**: Accepted (v1.0.0)

**Context**
Next.js 14+에서 Server Action이 도입되었지만, 이는 DB에 직접 접근하는 상황 가정. 외부 REST API 백엔드 서버를 사용하는 경우 패턴이 다름.

**Decision**
**외부 백엔드 API 서버가 있다면 Server Action 미사용**. Client-side `useQuery` + `clientFetch` 중심으로 사용.

SSR prefetch(`HydrationBoundary`)는 필요하면 즉시 사용 가능하되, 현재는 client-side 패턴 유지.

**Consequences**
- ✓ AI가 "Server Action으로 DB 직접 접근" 안티패턴을 피함
- ✓ 기존 REST API 호출 패턴 그대로 활용
- ✗ Progressive enhancement 혜택 미활용 (폼이 JS 없이도 동작하는 패턴)

**Alternatives considered**
- **Server Action으로 REST API 프록시**: 추가 네트워크 hop, 복잡도 증가

**Related**: [queries.md 아키텍처 결정](./rules/queries.md), [DF-16](./rules/api-layer.md)

---

# Forms & Tables

## ADR-010: react-hook-form + zod + shadcn Form 조합

**Status**: Accepted (v1.0.0)

**Context**
폼 라이브러리 선택은 개발자 경험과 타입 안전성에 큰 영향. 각 라이브러리가 validation, state 관리, UI 렌더링을 다르게 접근.

**Decision**
세 가지 라이브러리 조합 사용:
- **zod** (schema + validation, `entities/*/schema.ts`)
- **react-hook-form** (state 관리, `zodResolver` 연결)
- **shadcn Form** (UI, `FormField`/`FormItem`/`FormMessage`)

Schema에서 `z.infer<typeof schema>`로 타입 추출. Form 로직은 `features/*/hooks/use{Action}{Entity}Form.ts`로 추출.

**Consequences**
- ✓ 스키마가 validation + 타입의 single source of truth
- ✓ 재렌더 최소화 (react-hook-form uncontrolled 기본)
- ✓ 타입 안전한 폼 상태
- ✗ 3개 라이브러리 학습 곡선

**Alternatives considered**
- **Formik**: 구식, 성능 이슈
- **react-hook-form 단독**: 스키마 기반 validation 부족
- **TanStack Form**: 생태계 성숙도 낮음

**Related**: [FM-01~07](./rules/forms.md)

---

## ADR-011: Create/Edit 스키마 통합 — `as never` 금지

**Status**: Accepted (v1.0.0)

**Context**
Create 폼과 Edit 폼에서 동일한 form field를 공유할 때, 두 스키마 타입이 달라서 `<FormField name={'field' as never} />` 같은 타입 단언 발생 → 타입 안전성 우회.

**Decision**
**base 스키마 → partial + extend로 edit 스키마 파생**. `as never` 금지.

```ts
const baseProductSchema = z.object({ ... })
export const createProductSchema = baseProductSchema
export const editProductSchema = baseProductSchema.partial().extend({
  name: z.string().min(1), // 필수 유지
})
```

**Consequences**
- ✓ 타입 안전성 완전 보존
- ✓ form field 컴포넌트 공유 가능
- ✗ edit 시 일부 필드가 optional이 되는 것을 명시적으로 처리

**Alternatives considered**
- **Create/Edit 완전 분리**: form field 중복
- **as never 허용**: 타입 단언으로 안전성 우회 → 런타임 에러 위험

**Related**: [FM-07](./rules/forms.md)

---

## ADR-012: DataTable — Table Instance 방식

**Status**: Accepted (v1.0.0)

**Context**
TanStack Table을 사용하는 두 가지 방식:
- **Pattern A**: `<DataTable columns={...} data={...} sorting={...} ... />` (separate props)
- **Pattern B**: widget에서 `useReactTable()`로 instance 생성 → `<DataTable table={table} />` (instance)

공식 문서는 Pattern B를 권장.

**Decision**
**Pattern B (table instance) 채택**. 서버사이드 처리 전용 (`manualPagination: true`).

**Consequences**
- ✓ 공식 권장 패턴
- ✓ DataTable 컴포넌트가 table 설정을 몰라도 됨 (유연성)
- ✓ Row selection, virtualization 등 고급 기능 확장 용이
- ✗ widget마다 `useReactTable` 호출 boilerplate

**Alternatives considered**
- **Pattern A (separate props)**: DataTable이 모든 옵션을 props로 받아 복잡화
- **DataTable이 내부에서 useReactTable**: 테이블 인스턴스에 접근 불가

**Related**: [TB-01, TB-03, TB-05](./rules/tables.md)

---

# Code Style

## ADR-013: 파일명 — FSD 레이어별 규칙

**Status**: Accepted (v1.0.0)

**Context**
파일명 규칙이 일관되지 않으면 AI가 파일 위치 예측 불가.

**Decision**
레이어별 파일명 규칙:
- `entities/*/` — 고정명: `api.ts`, `queries.ts`, `hooks.ts`, `types.ts`, `schema.ts`, `columns.tsx`, `index.ts`
- `shared/lib`, `shared/ui`, `widgets/` — **kebab-case**
- `features/*/ui/` 다이얼로그 — **PascalCase** (`Create{Entity}Dialog.tsx`)
- `features/*/ui/` sub 컴포넌트 — kebab-case (`{entity}-form-fields.tsx`)
- `features/*/hooks/` — camelCase + `use` 접두사

**Consequences**
- ✓ AI가 파일명만 보고 레이어/역할 추론 가능
- ✗ 혼합 규칙 (kebab-case/PascalCase) 학습 필요

**Alternatives considered**
- **모든 파일 kebab-case**: React 컨벤션과 충돌 (PascalCase 일반적)
- **모든 파일 PascalCase**: `use-table-state.ts` 같은 hook 파일엔 어색

**Related**: [NM-01, NM-06](./rules/naming.md)

---

## ADR-014: `'use client'` 배치 — FSD 레이어별 기준

**Status**: Accepted (v1.0.0)

**Context**
Next.js App Router에서 `'use client'` 지시어 배치 기준이 일관되지 않으면 Server/Client 경계가 모호해짐.

**Decision**
FSD 레이어별 `'use client'` 배치 고정:

| 위치 | `'use client'` |
|------|--------------|
| `app/page.tsx`, `layout.tsx` | **금지** (Server Component) |
| `widgets/`, `features/ui/`, `features/hooks/` | **항상** |
| `entities/hooks.ts`, `columns.tsx` | **항상** |
| `entities/types.ts`, `api.ts`, `schema.ts`, `queries.ts` | **금지** (서버 호환 유지) |
| `shared/ui/` | 인터랙티브만 |

**Consequences**
- ✓ types/api/schema가 서버에서도 사용 가능 → SSR prefetch 도입 용이
- ✓ AI가 파일만 보고 지시어 판단 가능
- ✗ 규칙이 세분화되어 초기 학습 필요

**Alternatives considered**
- **모든 client 파일에 추가**: 과도, SSR 활용 기회 상실

**Related**: [PS-11](./rules/nextjs-patterns.md)

---

## ADR-015: Suspense + fallback 필수 (nuqs CSR bailout)

**Status**: Accepted (v1.0.0)

**Context**
nuqs의 `useQueryState`가 내부적으로 `useSearchParams`를 사용 → Next.js가 CSR bailout 경고. Suspense boundary로 감싸지 않으면 빌드 경고.

**Decision**
`useQueryState`를 사용하는 widget은 **page.tsx에서 `<Suspense fallback={<Skeleton />}>` 필수**로 감싼다.

**Consequences**
- ✓ CSR bailout 방지
- ✓ 초기 로딩 시 skeleton UI 제공
- ✗ 모든 테이블 페이지에 Suspense 래퍼 필요

**Alternatives considered**
- **Suspense 없이**: 빌드 경고 + 초기 로딩 시 빈 화면
- **widget 내부에 Suspense**: boundary 위치가 부적절

**Related**: [PS-04, PS-14](./rules/nextjs-patterns.md)

---

## ADR-016: Conditional Rendering — boolean만 `&&` 허용

**Status**: Accepted (v1.0.0)

**Context**
React 공식 문서 경고: `{count && <Component />}`에서 `count === 0`이면 `0`이 렌더링됨. 하지만 boolean이면 `false`는 렌더링 안 되므로 안전.

**Decision**
- **boolean 조건**: `&&` 허용 (`{isLoading && <Spinner />}`)
- **number/string 조건**: **삼항 연산자 필수** (`{count > 0 ? <Badge count={count} /> : null}`)
- **3개 이상 분기**: object lookup 사용 (`STATUS_COMPONENTS[status]`)

**Consequences**
- ✓ `0`/`''` 렌더링 버그 방지
- ✓ boolean 조건은 간결하게 `&&` 사용
- ✗ 개발자가 좌항 타입 판단 필요

**Alternatives considered**
- **모든 조건을 삼항으로**: 과도한 verbosity
- **모든 곳에 `&&`**: `0` 렌더링 버그 발생

**Related**: [PS-10](./rules/code-quality.md)

---

# AI Integration

## ADR-017: MUST / SHOULD / MAY 3단계 엄격도 (RFC 2119)

**Status**: Accepted (v1.0.0)

**Context**
"이 규칙을 반드시 따라야 하는가?"가 모호하면 AI가 일관되게 적용하지 못함. RFC 2119가 IETF 표준 용어.

**Decision**
RFC 2119 용어 채택:
- **MUST** — 절대 위반 불가
- **SHOULD** — 특별한 이유 없으면 따름 (예외 시 주석으로 이유 명시)
- **MAY** — 권장하지만 상황에 따라 유연

**Consequences**
- ✓ AI가 위반 중요도 즉시 판단
- ✓ 표준 용어로 누구나 이해
- ✗ 세 단계 이상 세분화 불가

**Related**: 모든 규칙의 헤더

---

## ADR-018: 🚫 / ⚠️ / ✅ 이모지 병기 (RFC 2119와 함께)

**Status**: Accepted (v1.0.0)

**Context**
ETH 연구 및 Vercel BP 권장: AI에게 이모지 경계가 단어보다 빠른 패턴 매칭 제공.

**Decision**
RFC 2119 단어에 이모지 병기:
- 🚫 **MUST** — 금지/필수
- ⚠️ **SHOULD** — 주의/권장
- ✅ **MAY** — 선택 가능

단어는 유지하여 RFC 2119 표준 준수, 이모지로 빠른 인식 추가.

**Consequences**
- ✓ 시각적 패턴 매칭 (AI + 인간 모두)
- ✓ 표준 용어 유지로 다른 도구/문서와 호환
- ✗ 이모지는 환경에 따라 렌더링 차이 (일부 터미널)

**Related**: 모든 규칙 헤더

---

## ADR-019: Validation Workflow — 체크리스트 + bash 명령어

**Status**: Accepted (v1.0.0)

**Context**
"이 프로젝트를 컨벤션 기준으로 검증해달라"는 요청에 AI가 일관된 방식으로 응답해야 함. 공식 BP (Anthropic): "체크리스트를 복사하여 response에 포함, 단계별 체크".

**Decision**
`AGENTS.md`의 Validation Workflow에 두 모드 제공:
- **Mode 1** (전체 검증): 15단계 체크리스트 + bash 명령어 + 출력 형식 예시
- **Mode 2** (파일 단위 검증): 수정 중인 파일별 확인할 규칙 테이블

**Consequences**
- ✓ 검증 결과 일관성 (다른 AI 세션에서 같은 출력 형식)
- ✓ 자동화 친화적 (bash 명령어를 CI에서도 사용 가능)
- ✗ AGENTS.md 길이 증가 (200줄 → 250줄)

**Alternatives considered**
- **요약만 제공**: AI 응답 일관성 저하
- **완전 자동화 스크립트**: 규칙 변경 시 스크립트도 업데이트 필요

**Related**: [AGENTS.md Validation Workflow](./AGENTS.md)

---

# Open Questions

아직 결정되지 않았거나, 프로젝트 성장에 따라 재검토가 필요한 항목들.

## OQ-001: Optimistic Updates 도입 범위

**현황**: DF-13으로 MAY 문서화만 된 상태.

**질문**: 어떤 mutation에 optimistic update를 적용할지 구체적 가이드가 필요한가?

**후보 기준**:
- 토글 (체크박스, 상태 변경): 적합
- 삭제: 롤백 UX 고려 필요
- Create: 임시 ID 처리 복잡

**결정 시점**: 실제 UX 피드백이 모이면 구체화.

---

## OQ-002: Memoization 자동화 (React Compiler)

**현황**: `useCallback`/`useMemo`를 SHOULD로 명시적 사용 권장.

**질문**: React Compiler가 안정화되면 자동 memoization이 되는데, 그때 규칙을 어떻게 바꿀까?

**예상 전환**:
- React Compiler 채택 시 대부분의 수동 memoization 불필요
- 규칙을 "React Compiler 사용 시 생략 가능"으로 업데이트

**결정 시점**: React Compiler가 stable 릴리스되고 Next.js가 공식 지원하면.

---

## OQ-003: 테스트 컨벤션

**현황**: 테스트 관련 규칙 없음.

**질문**: Vitest + Playwright 테스트 작성 규칙을 추가할 것인가?

**고려사항**:
- Unit test 위치 (`*.test.ts` vs `__tests__/`)
- E2E test 시나리오 구조
- Mock 패턴 (특히 TanStack Query)

**결정 시점**: 테스트 커버리지가 충분히 올라가면.

---

## OQ-004: 스타일링 / 디자인 시스템

**현황**: Tailwind + shadcn/ui + radix-nova 사용. 스타일링 규칙은 별도 디자인 시스템 문서에서 다룰 예정.

**질문**: 스타일링 규칙을 이 컨벤션에 포함시킬 것인가?

**분리 근거**: 디자인 시스템은 별도 repo/문서에서 관리하는 게 일반적 (Material-UI, Chakra 등).

**결정 시점**: 디자인 시스템 문서가 작성되면 링크만 추가.

---

## OQ-005: Infinite Query / Cursor Pagination

**현황**: DF-15로 MAY 문서화 (offset 페이지네이션 중심).

**질문**: 언제 infinite query를 적극 도입할 것인가?

**후보 상황**:
- 타임라인 스타일 UI (피드, 로그)
- 검색 결과 (관련성 순)
- 대용량 목록 (수천~수만 건)

**결정 시점**: 해당 UX가 실제 요구되면.

---

# References

### Architecture Decision Records (ADR)
- [Michael Nygard — Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [joelparkerhenderson/architecture-decision-record](https://github.com/joelparkerhenderson/architecture-decision-record)
- [ADR Templates (adr.github.io)](https://adr.github.io/adr-templates/)

### 프레임워크 BP
- [TanStack Query v5](https://tanstack.com/query/latest)
- [TanStack Table v8](https://tanstack.com/table/latest)
- [Next.js App Router](https://nextjs.org/docs/app)
- [React 19 Release](https://react.dev/blog/2024/12/05/react-19)

### 아키텍처
- [Feature-Sliced Design](https://feature-sliced.design)
- [Vercel React Best Practices](https://vercel.com/blog/introducing-react-best-practices)
- [Anthropic Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

### RFC 2119
- [RFC 2119 — Key words for use in RFCs](https://datatracker.ietf.org/doc/html/rfc2119)

---

## Change Log

| 날짜 | 변경 | 관련 ADR |
|------|------|---------|
| 2026-04-16 | Initial release (v1.0.0) — 19 ADRs + 5 Open Questions | All |
