---
name: frontend-conventions
description: Applies frontend coding conventions for Next.js 16 + TanStack Query + FSD architecture. Use when writing or reviewing React/TypeScript code, creating new entities/features/widgets, implementing CRUD forms or data tables, refactoring to TanStack Query v5, or auditing a project against coding standards. Covers FSD layers, queryOptions factory, react-hook-form + zod patterns, table instance pattern, nuqs URL state, error handling, and naming rules.
version: 1.0.0
---

# Frontend Conventions

Next.js 16 + React 19 + TanStack Query v5 + FSD architecture에서 **"좋은 코드이면서 일관된 코드"**를 생성하기 위한 컨벤션.

## When to Use This Skill

- 새로운 entity, feature, widget을 만들 때
- CRUD 폼이나 데이터 테이블을 구현할 때
- 기존 코드를 리팩토링할 때
- 프로젝트가 컨벤션을 준수하는지 검증할 때
- API layer, state management, error handling 패턴을 적용할 때

## Tech Stack

- **Next.js** 16.1.6 (App Router) + **React** 19.2.3 + **TypeScript** 5.9.2
- **@tanstack/react-query** ^5 + **@tanstack/react-table** ^8.21.3
- **react-hook-form** ^7.71.2 + **zod** ^3.25.76
- **nuqs** ^2.8.9 (URL state)
- **Tailwind CSS** ^4 + **shadcn/ui** (radix-nova) + **sonner** ^2.0.7 (toast)
- **Turborepo** ^2.8.12 + **pnpm** 9.0+ (internal: `@workspace/ui`, `@workspace/api`)

## Rule Categories

11개 카테고리, 67개 규칙. 작업에 맞는 파일을 참조:

| 영역 | 파일 | 언제 참조 |
|------|------|----------|
| **Architecture** | [rules/architecture.md](rules/architecture.md) | FSD 레이어, import 방향, entity slice, barrel export |
| **Next.js Patterns** | [rules/nextjs-patterns.md](rules/nextjs-patterns.md) | Page thin shell, Providers, 'use client', 특수 파일 |
| **Code Quality** | [rules/code-quality.md](rules/code-quality.md) | Dynamic import, conditional render, early return, forwardRef |
| **Queries** | [rules/queries.md](rules/queries.md) | queryOptions factory, query keys, staleTime, prefetch, useQueries |
| **Mutations** | [rules/mutations.md](rules/mutations.md) | invalidate, useApiMutation, optimistic |
| **API Layer** | [rules/api-layer.md](rules/api-layer.md) | API 객체, 응답 타입, SSR prefetch |
| **Forms** | [rules/forms.md](rules/forms.md) | zod 스키마, useForm, FormDialog, edit 패턴 |
| **Tables** | [rules/tables.md](rules/tables.md) | DataTable, columns, server-side pagination |
| **State Management** | [rules/state-management.md](rules/state-management.md) | nuqs, useState, 전역 상태 금지 |
| **Error Handling** | [rules/error-handling.md](rules/error-handling.md) | ApiError, toast, Error Boundary |
| **Naming** | [rules/naming.md](rules/naming.md) | 파일/훅/타입/상수 네이밍 |

전체 인덱스는 [INDEX.md](INDEX.md) 참조.

## Severity Levels

각 규칙은 세 가지 엄격도 중 하나:

| 레벨 | 의미 | AI 행동 |
|------|------|---------|
| 🚫 **MUST** | 절대 위반 불가 | 항상 이 패턴을 따른다 |
| ⚠️ **SHOULD** | 특별한 이유 없으면 따름 | 기본 적용, 예외 시 주석으로 이유 명시 |
| ✅ **MAY** | 권장하지만 유연 | 상황에 맞게 판단 |

## Validation Workflow

"이 프로젝트를 컨벤션 기준으로 검증해라" 요청 시 다음 순서로 수행:

1. **Tech stack 확인** — package.json에서 Next.js/React/TanStack Query 버전 확인
2. **FSD 구조 확인** (PS-01~05) — apps/*/entities, features, widgets, shared 존재 여부
3. **Import 방향 검증** (PS-02) — entity가 features/widgets를 import하는 곳 검색
4. **Entity 완전성** (PS-03) — 각 entity에 api.ts, queries.ts, hooks.ts, types.ts, index.ts 존재 여부
5. **Mutation 패턴** (DF-06, DF-07) — useApiMutation 사용률, raw useMutation 위반 검색
6. **폼 패턴** (FM-01~07) — zod schema 위치, useForm+zodResolver, form hook 추출 여부
7. **테이블 패턴** (TB-01~05) — useReactTable instance 방식, 서버사이드 manual 모드
8. **상태 관리** (SM-01) — Context/zustand 사용 여부 (금지)
9. **파일명 규칙** (NM-01) — kebab-case 준수율

**출력 형식**:
- 위반 목록 (규칙 ID + `file:line` + 수정 제안)
- 준수율 요약 (카테고리별)
- 우선순위별 권고 (🚫 먼저, ⚠️ 다음)

## Quick Start by Task

**새 entity 추가**: architecture → api-layer → queries → mutations → naming
**CRUD 폼 구현**: forms → mutations (DF-06, DF-07) → naming
**테이블 페이지 구현**: tables → state-management → queries
**새 페이지/라우트 추가**: nextjs-patterns → architecture (PS-02, PS-05)
**에러 처리**: error-handling
**성능 최적화**: code-quality → queries (DF-10~12)
**기존 코드 리팩토링**: 해당 영역의 규칙 파일 전체

## References

- Anthropic Skill authoring best practices: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- TanStack Query v5 docs: https://tanstack.com/query/latest
- Feature-Sliced Design: https://feature-sliced.design
