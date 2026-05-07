# Changelog

모든 주요 변경사항은 이 파일에 기록됩니다.
이 프로젝트는 [Semantic Versioning](https://semver.org/)을 따릅니다.

## [2.1.0] - 2026-05-06

### 🎯 단일 사용 features 정책 명시

ADR-021(v2.0.0)에서 FSD에 entity 파일 구성을 위임한 결과, *"FSD 결정 모두 따라야 하는가?"* 후속 질문 발생. 특히 FSD Section 5-2 / Steiger `insignificant-slice`의 *"단일 사용 features는 view에 inline"* 권장에 대해 dashboard 사례 분석 후 **분리 유지를 의식적 선택**으로 결정.

### Added

- **ADR-022** — 단일 사용 features 분리 허용. FSD `insignificant-slice` 권장 대비 의식적 선택. 근거: AI 코드 생성 일관성 + 모노레포 재사용 + DF-06 책임 분리.
- **SKILL.md Position 보강** — *"Single-use feature 정책"* 섹션 추가. FSD insignificant-slice 권장과의 차이 명시.

### Why not just rule (PS-XX)?

- v1.1.0/v2.0.0의 *"FSD architecture는 feature-sliced-design skill에 전적 위임"* 정책과 일관성 유지를 위해 신규 규칙(PS-XX) 추가하지 않음
- 정책 명문화는 ADR + SKILL.md Position으로 충분
- AI는 SKILL.md(항상 로드) → ADR(필요 시 참조) 순으로 컨텍스트 확보

### Severity System (Unchanged)

- 🚫 MUST: 30 (변동 없음)
- ⚠️ SHOULD: 33 (변동 없음)
- ✅ MAY: 9 (변동 없음)
- 총: 72개 규칙 (변동 없음)

### Migration

기존 v2.0.0 코드 영향 없음. dashboard 앱에 이미 적용된 features 분리 패턴이 컨벤션으로 명문화된 형태.

---

## [2.0.0] - 2026-05-06

### 💥 Breaking Changes — FSD v2.1 정합 채택

**핵심**: PS-03 (entity 평탄 파일 고정명)을 v1.1.0의 *"FSD 안티패턴 의도적 override"* 결정에서 **FSD v2.1 정합**으로 전환. entity 파일 구성을 `feature-sliced-design` skill에 완전 위임.

이는 dashboard 앱(`apps/dashboard`)에 FSD v2.1 가이드를 적용해본 결과 ([PR #160](https://github.com/aptimizer-co/frontend-aptimizer/pull/160))를 토대로 한 검증된 결정. 자세한 근거는 [ADR-021](DECISIONS.md#adr-021-ps-03-제거--fsd-도메인-기반-네이밍-채택) 참조.

### Removed

- **PS-03 (Entity slice 파일 고정명) 규칙 삭제** — `feature-sliced-design` skill에 위임. v1.1.0의 *"FSD 안티패턴 의도적 override"* 명문 폐기.

### Changed

- **NM-01 재작성** — entity 내부 "고정명" 강제 폐기 → 모든 파일 도메인 기반 kebab-case (FSD Rule 4-4 정합). technical-role 파일명(`api.ts`, `hooks.ts`, `types.ts`, `utils.ts`) 명시적 금지.
- **INDEX.md "Entity 구조 템플릿" 재작성** — 평탄 7파일 → 세그먼트(`api/`, `model/`, `ui/`) + 도메인 기반 파일명. FSD v2.1 표준 세그먼트 채택.
- **적용 위치 경로 갱신**:
  - DF-01: `entities/*/queries.ts` → `entities/*/api/{entity}-queries.ts`
  - DF-06: `entities/*/hooks.ts` → `entities/*/api/{entity}-mutations.ts`
  - DF-07: 동일
  - DF-08: `entities/*/api.ts` → `entities/*/api/{entity}.ts`
  - FM-01: `entities/*/schema.ts` → `entities/*/model/{entity}-form.ts`
  - TB-02: `entities/*/columns.tsx` → `entities/*/ui/{entity}-columns.tsx`
- **PS-11 'use client' 배치표 갱신** — entity 세그먼트 경로 기준으로 재작성 (`entities/*/api/{entity}-mutations.ts` 항상, `entities/*/model/*` 금지 등).
- **DF-06 Why 보강** — entity/feature toast 분리의 두 번째 이유로 *"FSD Rule 4-3 cross-import 회피"* 추가. entity가 다른 entity의 query key를 import하면 같은 레이어 cross-import 발생.
- **ADR-002 Status: Superseded** by ADR-021.

### Added

- **ADR-021** — PS-03 제거 + FSD 도메인 기반 네이밍 채택. dashboard 앱 검증 결과(PR #160) 기반 결정 기록.

### Severity System (Updated)

- 🚫 **MUST** — **30개** (1.1.0의 31개 - PS-03 1 = 30)
- ⚠️ **SHOULD** — **33개** (변동 없음)
- ✅ **MAY** — **9개** (변동 없음)
- **총: 72개 규칙, 12개 카테고리** (이전 73개)

### Migration from 1.1.0

기존 v1.x 코드는 다음 마이그레이션이 필요:

```
# Before (v1.x — PS-03)
entities/product/
├── api.ts
├── queries.ts
├── hooks.ts
├── types.ts
├── schema.ts
├── columns.tsx
└── index.ts

# After (v2.0 — FSD 정합)
entities/product/
├── api/
│   ├── product.ts
│   ├── product-queries.ts
│   └── product-mutations.ts
├── model/
│   ├── product.ts
│   └── product-form.ts (있을 때)
├── ui/
│   └── product-columns.tsx (있을 때)
└── index.ts
```

**index.ts 외부 노출 인터페이스는 동일 유지** 가능하므로 외부 슬라이스 임포트 변경 불필요. 내부 파일 이동 + import 경로 갱신만 필요.

DF-06 cross-import 회피 패턴 적용 시 features 레이어로 cross-domain invalidation 이동 (PR #160, PR #164 사례 참조).

---

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
