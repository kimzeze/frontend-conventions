# Changelog

모든 주요 변경사항은 이 파일에 기록됩니다.
이 프로젝트는 [Semantic Versioning](https://semver.org/)을 따릅니다.

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

- 🚫 **MUST** — 25개 (절대 위반 불가)
- ⚠️ **SHOULD** — 33개 (특별한 이유 없으면 따름)
- ✅ **MAY** — 9개 (권장, 상황에 따라)
