# AGENTS.md

Next.js 16 + React 19 + TanStack Query v5 + FSD 아키텍처 프론트엔드 코딩 컨벤션.
AI 코딩 에이전트(Claude Code, Cursor, Codex, GitHub Copilot 등)가 이 컨벤션을 기반으로 코드를 생성·검토·리팩토링하기 위한 진입점.

## Documentation Precedence

- `AGENTS.md` (this file) — 도구 중립 진입점, 모든 AI 에이전트가 먼저 읽음
- `SKILL.md` — `npx skills add`로 설치 시 진입점 (AGENTS.md와 동일 내용을 skill 형식으로)
- `CLAUDE.md` — Claude Code 진입점 (→ AGENTS.md 참조)
- `INDEX.md` — 67개 규칙 전체 인덱스
- `DECISIONS.md` — 설계 결정 배경 (ADR 형식, 19 ADRs)
- `rules/*.md` — 카테고리별 상세 규칙

**충돌 시 우선순위**: 이 파일 > `INDEX.md` > `rules/*.md`. `rules/*.md`의 내용이 최종 규범이며, 이 파일은 라우터 역할.

## Agent Entry Points

- **Codex / Copilot / Cursor / 기타**: `AGENTS.md` (this file)
- **Claude Code**: `CLAUDE.md` → `AGENTS.md`
- **Skills CLI 설치**: `SKILL.md` (`.claude/skills/frontend-conventions/`)

## Tech Stack

### Runtime
- **Node.js** ≥ 18
- **pnpm** 9.0+

### Framework
- **Next.js** 16.1.6 (App Router)
- **React** 19.2.3
- **TypeScript** 5.9.2

### Data & State
- **@tanstack/react-query** ^5 (queryOptions factory)
- **@tanstack/react-table** ^8.21.3 (table instance 패턴)
- **nuqs** ^2.8.9 (URL state)

### Forms & Validation
- **react-hook-form** ^7.71.2
- **zod** ^3.25.76

### UI
- **Tailwind CSS** ^4
- **shadcn/ui** + **radix-nova**
- **sonner** ^2.0.7 (toast 알림)

### Monorepo
- **Turborepo** ^2.8.12
- **pnpm workspace** (internal packages: `@workspace/ui`, `@workspace/api`)

## Rule Categories

67개 규칙, 11개 영역:

| 영역 | 파일 | 규칙 수 |
|------|------|---------|
| Architecture (FSD 레이어, import, 파일 구성) | [rules/architecture.md](rules/architecture.md) | 8 (PS-01,02,03,05,06,07,08,12) |
| Next.js Patterns (Page, Providers, use client, 특수 파일) | [rules/nextjs-patterns.md](rules/nextjs-patterns.md) | 4 (PS-04,11,13,14) |
| Code Quality (Dynamic Import, Conditional, Early Return, forwardRef) | [rules/code-quality.md](rules/code-quality.md) | 4 (PS-09,10,15,16) |
| Queries (queryOptions, query keys, staleTime, prefetch, useQueries) | [rules/queries.md](rules/queries.md) | 10 (DF-01~05,10~12,14,15) |
| Mutations (invalidate, useApiMutation, optimistic) | [rules/mutations.md](rules/mutations.md) | 3 (DF-06,07,13) |
| API Layer (API 객체, 응답 타입, SSR) | [rules/api-layer.md](rules/api-layer.md) | 3 (DF-08,09,16) |
| Forms (zod, react-hook-form) | [rules/forms.md](rules/forms.md) | 7 |
| Tables (DataTable, server-side) | [rules/tables.md](rules/tables.md) | 5 |
| State Management (nuqs, useState) | [rules/state-management.md](rules/state-management.md) | 9 |
| Error Handling (ApiError, toast) | [rules/error-handling.md](rules/error-handling.md) | 6 |
| Naming (파일/훅/타입/상수) | [rules/naming.md](rules/naming.md) | 8 |

전체 목록: [INDEX.md](INDEX.md)

## Severity Levels

각 규칙은 세 가지 엄격도 중 하나:

- 🚫 **MUST** — 절대 위반 불가
- ⚠️ **SHOULD** — 특별한 이유 없으면 따름 (예외 시 주석으로 이유 명시)
- ✅ **MAY** — 권장하지만 상황에 따라 판단

## Validation Workflow

### Mode 1: 프로젝트 전체 검증

"이 프로젝트를 컨벤션 기준으로 검증해라" 요청 시, 아래 체크리스트를 **응답에 복사**하고 단계별로 체크해나간다:

```
Validation Progress:
- [ ] Step 1: Tech Stack 대조 (package.json 버전)
- [ ] Step 2: FSD 구조 확인 (architecture.md - PS-01)
- [ ] Step 3: Import 방향 검증 (architecture.md - PS-02)
- [ ] Step 4: Entity 파일 완전성 (architecture.md - PS-03)
- [ ] Step 5: Barrel export 검증 (architecture.md - PS-05)
- [ ] Step 6: Page thin shell (nextjs-patterns.md - PS-04)
- [ ] Step 7: Queries 패턴 (queries.md - DF-01, DF-02)
- [ ] Step 8: Mutation 패턴 (mutations.md - DF-06, DF-07)
- [ ] Step 9: API Layer (api-layer.md - DF-08, DF-09)
- [ ] Step 10: 폼 패턴 (forms.md - FM-01~03)
- [ ] Step 11: 테이블 패턴 (tables.md - TB-01, TB-03)
- [ ] Step 12: 상태 관리 (state-management.md - SM-01, SM-02)
- [ ] Step 13: 에러 처리 (error-handling.md - EH-01, EH-03)
- [ ] Step 14: 파일명 규칙 (naming.md - NM-01, NM-02, NM-05)
- [ ] Step 15: 결과 요약 + 우선순위 권고
```

각 단계별 검증 명령어:

**Step 1** — Tech Stack
```bash
cat package.json apps/*/package.json | jq '.dependencies, .devDependencies'
```

**Step 2** — FSD 구조
```bash
ls apps/*/entities apps/*/features apps/*/widgets apps/*/shared 2>/dev/null
# 각 앱에 4개 디렉토리 모두 존재해야 함
```

**Step 3** — Import 방향 (하향만 허용)
```bash
# entity가 feature/widget을 import하는지 (위반)
grep -rn "from '@/features\|from '@/widgets" apps/*/entities/ 2>/dev/null
# entity 간 cross-import (위반)
grep -rn "from '@/entities/" apps/*/entities/ 2>/dev/null | grep -v "// same-entity"
```

**Step 4** — Entity 파일 완전성
```bash
for e in apps/*/entities/*/; do
  for required in api.ts queries.ts hooks.ts types.ts index.ts; do
    [ -f "$e$required" ] || echo "MISSING: $e$required"
  done
done
```

**Step 5** — Barrel export
```bash
# entity 내부 파일에 직접 접근 (위반 — index.ts 우회)
grep -rn "from '@/entities/[^']*/" apps/*/{widgets,features}/ | grep -vE "'@/entities/[^/]+'$"
```

**Step 7-8** — TanStack Query
```bash
# queries.ts가 없는 entity (DF-01 위반)
find apps/*/entities -maxdepth 2 -name "queries.ts" -o -name "query-keys.ts" | sort
# raw useMutation 사용 (DF-07 위반)
grep -rn "useMutation(" apps/ --include="*.ts" --include="*.tsx" | grep -v "useApiMutation"
# 인라인 query key (DF-01 위반)
grep -rn "queryKey: \[" apps/ --include="*.ts" --include="*.tsx" | grep -v "queries.ts" | grep -v ".test."
```

**Step 10** — 폼
```bash
# useForm이 zodResolver 없이 사용 (FM-03 위반)
grep -rn "useForm" apps/ | grep -v "zodResolver"
```

**Step 11** — 테이블
```bash
# manualPagination 미사용 (TB-03 위반)
grep -rn "useReactTable(" apps/ -A 10 | grep -vE "manualPagination: true|-- $"
```

**Step 12** — 상태
```bash
# Context/zustand 사용 (SM-01 위반)
grep -rn "createContext\|from 'zustand'" apps/ --include="*.ts" --include="*.tsx"
```

### Mode 2: 파일 단위 검증 (수정 중)

특정 파일을 수정할 때 해당 영역의 규칙만 대조:

| 수정 중인 파일 | 확인할 규칙 |
|---------------|-------------|
| `entities/*/api.ts` | DF-08, DF-09 (api-layer.md) |
| `entities/*/queries.ts` | DF-01, DF-02 (queries.md) |
| `entities/*/hooks.ts` | DF-06, DF-07 (mutations.md) |
| `entities/*/schema.ts` | FM-01, FM-02 (forms.md) |
| `entities/*/columns.tsx` | TB-02 (tables.md) |
| `features/*/hooks/*.ts` | FM-03 (forms.md) |
| `features/*/ui/*Dialog.tsx` | FM-04, FM-05 (forms.md) |
| `widgets/*.tsx` | TB-05, PS-06 (tables.md, architecture.md) |
| `app/**/page.tsx` | PS-04, PS-14 (nextjs-patterns.md) |
| `app/providers.tsx` | PS-13 (nextjs-patterns.md) |

### 출력 형식

**응답 구조**:

```markdown
## Validation 결과

### 준수율 요약
| 카테고리 | 🚫 MUST | ⚠️ SHOULD | ✅ MAY | 준수율 |
|---------|---------|----------|-------|--------|
| Architecture | 5/5 | 2/3 | - | 88% |
| Queries | 2/2 | 4/5 | 1/3 | 70% |
...

### 🚫 MUST 위반 (최우선 수정)

**[DF-07] apps/admin/entities/product/hooks.ts:15**
- 위반: raw `useMutation` 사용
- 수정: `useApiMutation`으로 교체
- 영향: mutation 에러 자동 처리 누락

### ⚠️ SHOULD 위반

**[NM-01] apps/admin/widgets/ProductTable.tsx**
- 위반: PascalCase 파일명
- 수정: `product-table.tsx`로 rename

### ✅ MAY 참고

(생략 가능 — MUST/SHOULD가 많으면)

### 권장 수정 순서
1. 🚫 MUST 위반 먼저 (기능적 문제)
2. ⚠️ SHOULD 위반 (일관성)
3. ✅ MAY (최적화)
```

## Quick Start by Task

| 작업 | 참조 순서 |
|------|----------|
| 새 entity 추가 | `architecture` → `api-layer` → `queries` → `mutations` → `naming` |
| CRUD 폼 구현 | `forms` → `mutations` (DF-06, DF-07) → `naming` |
| 테이블 페이지 구현 | `tables` → `state-management` → `queries` |
| 새 페이지/라우트 추가 | `nextjs-patterns` → `architecture` (PS-02, PS-05) |
| 에러 처리 | `error-handling` |
| 성능 최적화 | `code-quality` → `queries` (DF-10~12) |
| 기존 코드 리팩토링 | 해당 영역 규칙 파일 전체 |

## Continuous Improvement

규칙이 모호하거나 실제 작업과 맞지 않을 때, 다음 형식으로 제안:

- **무엇이 실패/모호했는가**
- **왜 재작업/리스크가 발생했는가**
- **추가/수정할 규칙**
- **적용 범위** (AGENTS.md vs rules/*.md)

저장소 특화 & 반복 적용 가능한 규칙만 제안.

## References

- [Anthropic Skill best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [AGENTS.md spec](https://agents.md/)
- [TanStack Query v5](https://tanstack.com/query/latest)
- [Feature-Sliced Design](https://feature-sliced.design)
