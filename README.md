# Frontend Conventions

> **Next.js 16 + React 19 + TanStack Query v5 + FSD 아키텍처**를 위한 코딩 컨벤션.
> AI 코딩 에이전트(Claude Code, Cursor, Codex, Copilot)가 자동으로 로드하여 일관된 코드를 생성하도록 돕는 **Skill**입니다.

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](./CHANGELOG.md)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)
[![Skills Compatible](https://img.shields.io/badge/skills-compatible-purple.svg)](https://github.com/vercel-labs/skills)

## 왜 이 컨벤션인가

- 🎯 **좋은 코드 + 우리 코드** — 업계 BP를 따르되, 팀의 일관된 패턴으로 통일
- 🤖 **AI 세션 간 동일한 코드 스타일** — 누가 쓰든, 어떤 세션이든 같은 결과
- 📐 **67개 규칙, 11개 영역** — FSD, TanStack Query, Forms, Tables, State 등 핵심 영역 커버
- 🚫/⚠️/✅ **3단계 경계** — MUST / SHOULD / MAY로 엄격도 명시

## Installation

[Vercel Labs Skills CLI](https://github.com/vercel-labs/skills)로 설치:

```bash
npx skills add kimzeze/frontend-conventions
```

지원 AI:
- **Claude Code** — `.claude/skills/frontend-conventions/` 자동 생성
- **Cursor** — `.cursor/skills/` 자동 생성
- **Codex, Copilot, 기타** — AGENTS.md 표준 자동 지원

## Usage

### 1. 프로젝트에 설치 후

```bash
cd your-project
npx skills add kimzeze/frontend-conventions
```

### 2. AI에게 작업 지시

```
# 컨벤션 기반 코드 생성
"frontend-conventions를 따라 새 product entity를 만들어줘"

# 프로젝트 검증
"이 프로젝트를 frontend-conventions 기준으로 검증해줘"
# → AGENTS.md의 Validation Workflow 15단계 체크리스트 실행

# 리팩토링
"entities/product가 컨벤션에 맞지 않는 부분 리팩토링해줘"
```

### 3. 수동 참조 (AI 없이)

- **[AGENTS.md](./AGENTS.md)** — 진입점 (Tech Stack, Validation Workflow)
- **[INDEX.md](./INDEX.md)** — 67개 규칙 전체 인덱스
- **[rules/](./rules)** — 카테고리별 상세 규칙 (11파일)
- **[DECISIONS.md](./DECISIONS.md)** — ADR 형식의 설계 결정 (19개 ADR + 5개 Open Questions)

## Rule Categories

| 영역 | 파일 | 규칙 수 | 주제 |
|------|------|---------|------|
| Architecture | [rules/architecture.md](./rules/architecture.md) | 8 | FSD 레이어, import 방향, entity slice |
| Next.js Patterns | [rules/nextjs-patterns.md](./rules/nextjs-patterns.md) | 4 | Page, Providers, 'use client', 특수 파일 |
| Code Quality | [rules/code-quality.md](./rules/code-quality.md) | 4 | Dynamic import, Conditional, Early Return |
| Queries | [rules/queries.md](./rules/queries.md) | 10 | queryOptions factory, staleTime, prefetch |
| Mutations | [rules/mutations.md](./rules/mutations.md) | 3 | invalidate, useApiMutation, optimistic |
| API Layer | [rules/api-layer.md](./rules/api-layer.md) | 3 | API 객체, 응답 타입, SSR |
| Forms | [rules/forms.md](./rules/forms.md) | 7 | zod + react-hook-form + shadcn Form |
| Tables | [rules/tables.md](./rules/tables.md) | 5 | TanStack Table instance, server-side |
| State Management | [rules/state-management.md](./rules/state-management.md) | 9 | nuqs, useState, 전역 상태 금지 |
| Error Handling | [rules/error-handling.md](./rules/error-handling.md) | 6 | ApiError, toast, Error Boundary |
| Naming | [rules/naming.md](./rules/naming.md) | 8 | 파일/훅/타입/상수/컴포넌트 네이밍 |

## Tech Stack

이 컨벤션은 아래 스택을 전제로 합니다:

- **Next.js** 16.1.6 (App Router) + **React** 19.2.3 + **TypeScript** 5.9.2
- **@tanstack/react-query** ^5 + **@tanstack/react-table** ^8
- **react-hook-form** + **zod**
- **nuqs** (URL state)
- **Tailwind CSS** + **shadcn/ui** + **sonner**
- **Turborepo** + **pnpm workspace**

다른 스택에서도 참고 가능하지만, 일부 규칙(queryOptions, useApiMutation 등)은 직접 적용되지 않을 수 있습니다.

## Rule Format

각 규칙은 다음 구조를 따릅니다:

```markdown
## [ID] 규칙 이름 (🚫 MUST)

**적용 위치**: `entities/*/queries.ts`

**규칙**: 한 문장 명령형.

**Do**: 올바른 코드 예제

**Don't**: 잘못된 코드 예제

**Why**: 이유 설명.
```

### 엄격도 레벨

| 레벨 | 의미 | AI 행동 |
|------|------|---------|
| 🚫 **MUST** | 절대 위반 불가 | 항상 이 패턴을 따름 |
| ⚠️ **SHOULD** | 특별한 이유 없으면 따름 | 기본 적용, 예외 시 주석으로 이유 명시 |
| ✅ **MAY** | 권장하지만 유연 | 상황에 맞게 판단 |

## Documentation Structure

```
frontend-conventions/
├── README.md           ← 이 파일 (공개 진입점)
├── SKILL.md            ← Skills CLI 진입점 (npx skills add)
├── AGENTS.md           ← AI 에이전트 진입점 (도구 중립)
├── CLAUDE.md           ← Claude Code 진입점 (→ AGENTS.md)
├── INDEX.md            ← 67개 규칙 전체 인덱스
├── DECISIONS.md        ← 19 ADRs + 5 Open Questions (설계 결정 배경)
├── CHANGELOG.md        ← 버전별 변경사항
└── rules/              ← 11개 카테고리별 규칙
    ├── architecture.md
    ├── nextjs-patterns.md
    ├── code-quality.md
    ├── queries.md
    ├── mutations.md
    ├── api-layer.md
    ├── forms.md
    ├── tables.md
    ├── state-management.md
    ├── error-handling.md
    └── naming.md
```

## Contributing

### 새 규칙 제안
1. 이슈를 열어 제안 내용 공유 (왜 필요한지, 어떤 문제를 해결하는지)
2. [Rule Format](#rule-format)을 따라 규칙 작성
3. 해당 카테고리 파일에 추가 또는 새 파일 생성
4. `AGENTS.md` + `INDEX.md` + `SKILL.md`의 관련 테이블 업데이트
5. PR 생성

### 규칙 수정
- 기존 규칙 ID(예: `DF-07`)는 유지 (기존 코드/리뷰 참조 보존)
- Severity 변경(SHOULD → MUST) 시 breaking change — 메이저 버전 업

### Breaking Changes
- Semver를 따릅니다:
  - `major` — 규칙 삭제, severity 강화 (SHOULD → MUST)
  - `minor` — 규칙 추가
  - `patch` — 예시 개선, 오타 수정

## References

이 컨벤션은 다음 출처의 BP를 기반으로 합니다:

- [Anthropic Skill best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [AGENTS.md spec](https://agents.md/)
- [TanStack Query v5](https://tanstack.com/query/latest)
- [TanStack Table v8](https://tanstack.com/table/latest)
- [Feature-Sliced Design](https://feature-sliced.design)
- [Vercel React Best Practices](https://vercel.com/blog/introducing-react-best-practices)
- [Next.js App Router docs](https://nextjs.org/docs/app)

## Version

- **v1.0.0** (현재) — 67개 규칙, 11개 영역

## License

MIT — 자유롭게 사용, 수정, 배포 가능합니다. [LICENSE](./LICENSE) 참조.
