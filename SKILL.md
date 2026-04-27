---
name: frontend-conventions
description: Locks in opinionated frontend code patterns for Next.js 16 + TanStack Query v5 + FSD projects. ALWAYS invoke when writing data-fetching code (NEVER useEffect+fetch — always TanStack Query), creating forms (always react-hook-form + zod), introducing client state (Zustand for client-global only — NEVER for server data, NEVER Context for data sharing), implementing tables (TanStack Table v8 + manualPagination + nuqs URL state), or working in entities/features/widgets/app folders. Pairs with feature-sliced-design skill — defers all FSD architecture decisions there. Resolves ambiguity when multiple BP skills offer conflicting recommendations by locking in one direction.
version: 1.1.0
---

# Frontend Conventions (Personal Harness)

## Position

이 skill은 다른 BP skill들(`feature-sliced-design`, `tanstack-query-best-practices`, `next-best-practices`, `vercel-react-best-practices` 등)이 보여주는 **여러 가능한 방향들 중, 사용자가 선택한 한 방향만 고정**한다. AI가 "BP 따르라"는 모호한 지시를 받았을 때 흔들리지 않도록 결정 트리를 제공.

**우선순위**: 이 skill의 규칙이 다른 BP skill의 권고와 충돌하면 **이 skill이 우선**한다 (이 프로젝트의 의도적 선택).

**예외**: FSD 아키텍처 영역(레이어 구조, import 방향, slice/widget/feature 정의)은 `feature-sliced-design` skill에 **전적 위임**. 이 skill은 그 위에 추가로 고정한 컨벤션만 다룬다.

---

## AI가 가장 흔히 저지르는 실수 Top 3 — 이 skill의 1차 목적은 이를 차단

| 실수 | 왜 위험 | 차단 규칙 |
|---|---|---|
| **시키지 않은 `useEffect`로 데이터 페칭** | race condition, 캐시 없음, 중복 요청 | [LIB-02](rules/library-choices.md#lib-02-데이터-페칭-must--최강-규칙), [DF-01](rules/queries.md#df-01-queryoptions-factory-must), [DF-07](rules/mutations.md#df-07-useapimutation-래퍼-사용-must) |
| **임의로 Context/Zustand 도입** (TanStack Query/useState로 충분한데) | 복잡도 폭발, sync 버그, 디버깅 난해 | [SM-01](rules/state-management.md#sm-01-상태-4-카테고리-분류-must--최강-규칙) |
| **폼을 `useState`로 관리** (zod/RHF 안 씀) | validation 누락, 타입 동기화 깨짐 | [LIB-01](rules/library-choices.md#lib-01-폼-라이브러리-must), [FM-03](rules/forms.md) |

→ AI가 이 영역의 코드를 만들 때 항상 위 규칙을 먼저 확인.

---

## Override Policy (Q4-B)

모든 🚫 MUST 규칙은 사용자가 명시적으로 위반 요청 시 **경고 후 진행**한다. 형식:

> "[규칙 ID]에서는 [default]를 권장합니다. 요청대로 [대안] 사용합니다. 다른 코드는 일관성을 위해 default를 따릅니다."

예시:
- 사용자: "Zustand 말고 Redux로 바꿔줘"
- AI: "SM-01에서는 Zustand를 권장합니다. 요청대로 Redux 사용합니다. 다른 영역은 SM-01 default를 유지합니다." → Redux로 변경

⚠️ 단, 다음 두 경우는 override 시에도 추가 경고 필수:
1. **LIB-02 위반 (useEffect 데이터 페칭)** — race condition·중복 요청 위험
2. **SM-01 위반 (서버 데이터를 Zustand에 캐싱)** — sync 버그 위험

---

## Tech Stack (고정)

- **Next.js** 16.1.6 (App Router) + **React** 19.2.3 + **TypeScript** 5.9.2
- **@tanstack/react-query** ^5 + **@tanstack/react-table** ^8.21.3
- **react-hook-form** ^7.71.2 + **zod** ^3.25.76
- **nuqs** ^2.8.9
- **Tailwind CSS** ^4 + **shadcn/ui** (radix 기반) + **sonner** ^2.0.7
- **Turborepo** ^2.8.12 + **pnpm** 9.0+ (`@workspace/ui`, `@workspace/api`)

→ 라이브러리 선택은 [Library Choices](rules/library-choices.md)에 lock-in.

---

## Rule Categories

12개 카테고리:

| 영역 | 파일 | 언제 참조 |
|------|------|----------|
| **Library Choices** ⭐ | [rules/library-choices.md](rules/library-choices.md) | 어떤 라이브러리를 쓸지 결정 (LIB-01~06) |
| **Architecture** | [rules/architecture.md](rules/architecture.md) | entity slice 파일명, barrel export, @workspace import (FSD 일반은 feature-sliced-design skill) |
| **Next.js Patterns** | [rules/nextjs-patterns.md](rules/nextjs-patterns.md) | Page thin shell, Providers, 'use client', 특수 파일 |
| **Code Quality** | [rules/code-quality.md](rules/code-quality.md) | Dynamic import, conditional, early return, forwardRef |
| **Queries** | [rules/queries.md](rules/queries.md) | queryOptions factory, query keys, staleTime, prefetch |
| **Mutations** | [rules/mutations.md](rules/mutations.md) | invalidate, useApiMutation, optimistic |
| **API Layer** | [rules/api-layer.md](rules/api-layer.md) | API 객체, 응답 타입, SSR prefetch |
| **Forms** | [rules/forms.md](rules/forms.md) | zod schema, useForm, FormDialog, edit 패턴 |
| **Tables** | [rules/tables.md](rules/tables.md) | DataTable, columns, server-side pagination |
| **State Management** | [rules/state-management.md](rules/state-management.md) | 4-카테고리 분류, nuqs, useState, Zustand BP |
| **Error Handling** | [rules/error-handling.md](rules/error-handling.md) | ApiError, toast, Error Boundary |
| **Naming** | [rules/naming.md](rules/naming.md) | 파일/훅/타입/상수 네이밍 |

전체 인덱스: [INDEX.md](INDEX.md).

---

## Severity Levels

| 레벨 | 의미 | AI 행동 |
|------|------|---------|
| 🚫 **MUST** | 절대 권고 — 사용자 override만 허용 | 항상 이 패턴. 위반 요청 시 Override Policy 적용. |
| ⚠️ **SHOULD** | 강한 권고 | 기본 적용, 예외 시 주석으로 이유 명시 |
| ✅ **MAY** | 권장 | 상황에 맞게 판단 |

---

## Quick Start by Task

| 작업 | 참조 순서 |
|------|----------|
| 새 entity 추가 | architecture (PS-03) → api-layer → queries → mutations → naming |
| CRUD 폼 구현 | library-choices (LIB-01) → forms → mutations (DF-06, DF-07) |
| 테이블 페이지 | library-choices (LIB-03, LIB-04) → tables → state-management (SM-01,02,03) → queries |
| 새 페이지/라우트 | nextjs-patterns → architecture (PS-05) |
| 데이터 페칭 영역 | **library-choices (LIB-02)** → queries (DF-01) → mutations (DF-07) |
| 전역 상태 도입 | **state-management (SM-01)** → SM-10~13 (Zustand BP) |
| 에러 처리 | error-handling |
| 성능 최적화 | code-quality → queries (DF-10~12) → state-management (SM-08, 09) |

---

## Validation Workflow

"이 프로젝트를 컨벤션 기준으로 검증해라" 요청 시 다음 14단계를 순서대로 수행:

```
- [ ] Step 1: Tech Stack 확인 (package.json)
- [ ] Step 2: Library Choices 확인 (LIB-01~06 위반 grep)
- [ ] Step 3: FSD 구조 확인 → feature-sliced-design skill에 위임
- [ ] Step 4: Entity 파일 완전성 (PS-03 — types.ts/api.ts/queries.ts/hooks.ts/index.ts)
- [ ] Step 5: Barrel export 검증 (PS-05)
- [ ] Step 6: Page thin shell (PS-04)
- [ ] Step 7: Queries 패턴 (DF-01, DF-02 — queryOptions factory, key 의존성)
- [ ] Step 8: Mutation 패턴 (DF-06, DF-07 — useApiMutation 강제)
- [ ] Step 9: API Layer (DF-08, DF-09)
- [ ] Step 10: 폼 패턴 (FM-01~03 — zod + RHF 강제)
- [ ] Step 11: 테이블 (TB-01, TB-03 — manualPagination)
- [ ] Step 12: 상태 4-카테고리 (SM-01) + Zustand BP (SM-10, 11)
- [ ] Step 13: 에러 처리 (EH-01, EH-03)
- [ ] Step 14: 결과 요약 + 우선순위 권고
```

### 핵심 grep 명령어 (Step 2, 7, 8, 10, 11, 12)

```bash
# Step 2 (LIB-02 위반): useEffect로 데이터 페칭
grep -rn "useEffect.*fetch\|useEffect.*axios" apps/ --include="*.ts" --include="*.tsx"

# Step 7 (DF-01 위반): 인라인 query key
grep -rn "queryKey: \[" apps/ --include="*.ts" --include="*.tsx" | grep -v "queries.ts" | grep -v ".test."

# Step 8 (DF-07 위반): raw useMutation 사용
grep -rn "useMutation(" apps/ --include="*.ts" --include="*.tsx" | grep -v "useApiMutation"

# Step 10 (FM-03 위반): zodResolver 없는 useForm
grep -rn "useForm" apps/ | grep -v "zodResolver"

# Step 11 (TB-03 위반): manualPagination 미사용
grep -rn "useReactTable(" apps/ -A 10 | grep -vE "manualPagination: true|-- $"

# Step 12 (SM-01 위반): Context/zustand 임의 도입
grep -rn "createContext\|from 'zustand'" apps/ --include="*.ts" --include="*.tsx"
```

### 출력 형식

```markdown
## Validation 결과

### 준수율 요약
| 카테고리 | 🚫 MUST | ⚠️ SHOULD | ✅ MAY | 준수율 |
|---|---|---|---|---|
| Library Choices | 6/6 | - | - | 100% |
| Architecture | 3/3 | 1/1 | - | 100% |
...

### 🚫 MUST 위반 (최우선 수정)

**[DF-07] apps/admin/entities/product/hooks.ts:15**
- 위반: raw `useMutation` 사용
- 수정: `useApiMutation`으로 교체

### ⚠️ SHOULD 위반 / ✅ MAY 참고 (생략 가능)

### 권장 수정 순서
1. 🚫 MUST 위반 먼저 (특히 LIB-02, SM-01, FM-03 - AI 실수 Top 3 영역)
2. ⚠️ SHOULD 위반
3. ✅ MAY (최적화)
```

---

## References

- [feature-sliced-design skill](https://skills.sh/feature-sliced/skills/feature-sliced-design) — FSD 아키텍처 위임
- [TanStack Query v5](https://tanstack.com/query/latest)
- [TkDodo — Working with Zustand](https://tkdodo.eu/blog/working-with-zustand)
- [Anthropic Skill best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
