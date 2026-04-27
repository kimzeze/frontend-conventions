# Changelog

모든 주요 변경사항은 이 파일에 기록됩니다.
이 프로젝트는 [Semantic Versioning](https://semver.org/)을 따릅니다.

## [1.1.0] - 2026-04-27

### 🎯 Position 재정립

이 harness는 다른 BP skill들이 보여주는 **여러 valid 방향들 중 사용자가 선택한 한 방향만 고정**하는 도구로 명문화. Claude Code 전용으로 단순화.

### Added

- **⭐ Library Choices 카테고리 신설** — `rules/library-choices.md` (LIB-01~06, 모두 MUST):
  - LIB-01: 폼 = react-hook-form + zod (Formik/yup/valibot 차단)
  - LIB-02: 데이터 페칭 = TanStack Query v5 (useEffect+fetch 절대 금지) — **최강 규칙**
  - LIB-03: URL 상태 = nuqs (useSearchParams 직접 사용 금지)
  - LIB-04: 테이블 = TanStack Table v8 + DataTable 래퍼
  - LIB-05: Toast = sonner
  - LIB-06: UI = shadcn/ui + radix + Tailwind v4
- **Zustand BP 4개 추가** (`rules/state-management.md`, TkDodo 기반):
  - SM-10: custom hook만 export (🚫 MUST)
  - SM-11: atomic selector (🚫 MUST)
  - SM-12: actions namespace 분리 (⚠️ SHOULD)
  - SM-13: action을 domain event로 명명 (⚠️ SHOULD)
- **Override Policy** — Q4-B 정책 명문화. 모든 MUST 규칙은 사용자 명시 요청 시 경고 후 진행.
- **AI 실수 Top 3** — SKILL.md description과 본문에 명시. (1) useEffect 데이터 페칭, (2) Context/Zustand 임의 도입, (3) 폼 useState.

### Changed

- **SM-01 재작성** — Zustand 금지 → 4-카테고리 분류 (서버=TQ, URL=nuqs, 로컬=useState, 클라이언트 전역=Zustand)
- **`SKILL.md` description rewrite** — "pushy" 형식으로 강화. ALWAYS/NEVER 명시. Top 3 trigger 키워드 포함.
- **모든 MUST 규칙에 Default + Override Policy 구조 추가**
- **PS-03 `entities/*` 파일 고정명**에 "FSD 안티패턴 의도적 override" 명문화 (FSD 2.1은 domain-based naming 권고하나 AI 일관성 위해 override).

### Removed

- **Architecture 4 규칙** (`feature-sliced-design` skill에 위임):
  - PS-01 (FSD 5 layers 구조)
  - PS-02 (import 하향 방향)
  - PS-06 (widget = 조합 정의)
  - PS-07 (feature = 사용자 액션 정의)
- **`AGENTS.md` 삭제** — Claude Code 전용 결정에 따라 도구 중립 진입점 불필요.
- **`CLAUDE.md` 삭제** — 스킬 설치 시 dead file. 개인 harness이므로 dev 가이드도 제거.

### Fixed

- `rules/queries.md` DF-01과 `rules/architecture.md` PS-03 / `rules/naming.md` NM-01의 **`queries.ts` vs `query-keys.ts` 모순** 해결. `queries.ts`로 통일.
- `rules/mutations.md:47` orphan code fence (` ``` `) 제거.
- `rules/architecture.md` 중복 `---` separator 정리.
- `rules/nextjs-patterns.md` 'use client' 정책 표의 `entities/query-keys.ts` → `entities/queries.ts`.
- `.gitignore`에 `.context/` 추가 + `.context/notes.md`, `.context/todos.md` 빈 파일 제거.

### Documentation

- `README.md` — Claude Code 전용 명시, AGENTS.md 참조 제거, "다른 skill과의 관계" 섹션 추가.
- `INDEX.md` — Library Choices 카테고리 추가, 통계 갱신 (67 → 73 rules).
- `SKILL.md` — Position 섹션, AI 실수 Top 3, Override Policy 섹션 추가. Validation Workflow 14단계 통일.

### Severity System (Updated)

- 🚫 **MUST** — **31개** (1.0.0의 25개 + LIB 6 + SM-10,11 2 - PS-01,02 2 = 31)
- ⚠️ **SHOULD** — **33개** (1.0.0의 33개 + SM-12,13 2 - PS-06,07 2 = 33)
- ✅ **MAY** — **9개** (변동 없음)
- **총: 73개 규칙, 12개 카테고리** (이전 67개, 11개 카테고리)

### Migration from 1.0.0

기존 코드 영향:
- ❌ Breaking 없음. 모든 변경은 add 또는 same-direction strengthen.
- ✅ `query-keys.ts` 파일이 있던 entity는 `queries.ts`로 통합 (DF-01 권고이며 1.0.0 시점에도 권고).
- ✅ Zustand 사용 코드는 SM-10~13 BP 적용 권장.

---

## [1.0.0] - 2026-04-16

### 🎉 Initial Release

67개 규칙, 11개 카테고리로 구성된 첫 번째 정식 버전.

### Added

- **Architecture** (8 규칙) — FSD 레이어, import 방향, entity slice
- **Next.js Patterns** (4 규칙) — Page, Providers, 'use client', 특수 파일
- **Code Quality** (4 규칙) — Dynamic import, Conditional render, Early return, forwardRef
- **Queries** (10 규칙) — queryOptions factory, staleTime, prefetch, useQueries
- **Mutations** (3 규칙) — invalidate, useApiMutation, optimistic
- **API Layer** (3 규칙) — API 객체, 응답 타입, SSR prefetch
- **Forms** (7 규칙) — zod + react-hook-form + shadcn Form
- **Tables** (5 규칙) — TanStack Table instance, server-side
- **State Management** (9 규칙) — nuqs, useState, 전역 상태 금지
- **Error Handling** (6 규칙) — ApiError, toast, Error Boundary
- **Naming** (8 규칙) — 파일/훅/타입/상수/컴포넌트

### Documentation

- `README.md` — 공개 진입점 (Installation, Usage, Contributing)
- `AGENTS.md` — AI 에이전트 진입점 (Tech Stack, Validation Workflow)
- `CLAUDE.md` — Claude Code 진입점 (`@AGENTS.md` import)
- `SKILL.md` — Skills CLI 진입점 (YAML frontmatter)
- `INDEX.md` — 67개 규칙 전체 인덱스

### Severity System

- 🚫 **MUST** — 25개
- ⚠️ **SHOULD** — 33개
- ✅ **MAY** — 9개
