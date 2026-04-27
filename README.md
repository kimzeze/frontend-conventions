# Frontend Conventions

> **Next.js 16 + React 19 + TanStack Query v5 + FSD** 기반 개인 프론트엔드 harness.
> 다른 BP skill들이 보여주는 여러 방향 중 **사용자가 선택한 한 방향만 고정**하여 AI 코드 생성·리뷰 일관성을 보장하는 **Claude Code skill**.

[![Version](https://img.shields.io/badge/version-1.1.0-blue.svg)](./CHANGELOG.md)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-purple.svg)](https://code.claude.com/docs/en/skills)

## 왜 이 harness인가

- 🎯 **"BP 따르라"는 모호한 지시를 한 방향 결정으로 변환** — Best Practice에도 여러 valid 방향이 있는데, 그중 하나만 고정
- 🤖 **AI 세션 간 동일한 코드 스타일** — 누가 쓰든, 어떤 세션이든 같은 결과
- 📐 **73개 규칙, 12개 영역** — Library Choices, FSD 보완, TanStack Query, Forms, Tables, State, Naming 등
- 🚫/⚠️/✅ **3단계 경계** — MUST / SHOULD / MAY로 엄격도 명시
- 🔒 **Override Policy 명문화** — 사용자 명시 요청 시 경고 후 진행

## 다른 skill과의 관계

| 이 skill의 역할 | 위임 또는 보완 |
|---|---|
| **Library Choices** lock-in (LIB-01~06) | 단독 — 라이브러리 선택은 우리만 결정 |
| **TanStack Query 패턴 강제** | `tanstack-query-best-practices` 위에 우리 선택 (queries.ts 위치, useApiMutation 래퍼 등) |
| **FSD 아키텍처** | **`feature-sliced-design` skill에 전적 위임** — entity slice 파일 고정명만 override |
| **Next.js 패턴** | `next-best-practices` 위에 우리 선택 (page thin shell 등) |
| **React 성능 최적화** | `vercel-react-best-practices` 위에 우리 선택 |

## Installation

```bash
npx skills add kimzeze/frontend-conventions
```

→ `.claude/skills/frontend-conventions/`에 설치되어 Claude Code에서 자동 활성.

**권장 동시 설치**:

```bash
npx skills add feature-sliced/skills          # FSD 아키텍처
npx skills add tanstack-query-best-practices   # TanStack Query 패턴
npx skills add next-best-practices              # Next.js 16
npx skills add vercel-react-best-practices     # React 성능
```

→ 다른 skill들이 가르치는 패턴 위에 이 harness가 한 방향 고정.

## Usage

### 1. 새 코드 생성

```
"frontend-conventions를 따라 새 product entity를 만들어줘"
```

→ Library Choices(LIB-01~06) + entity 파일 고정명(PS-03) + 4-카테고리 상태(SM-01) + 폼 패턴(FM-03) 등 자동 적용.

### 2. 프로젝트 검증

```
"이 프로젝트를 frontend-conventions 기준으로 검증해줘"
```

→ SKILL.md의 14단계 Validation Workflow 자동 실행.

### 3. 리팩토링

```
"entities/product가 컨벤션에 맞지 않는 부분 리팩토링해줘"
```

## AI가 가장 흔히 저지르는 실수 Top 3 (이 skill이 차단)

1. **시키지 않은 `useEffect`로 데이터 페칭** → LIB-02 차단
2. **임의로 Context/Zustand 도입** → SM-01 4-카테고리 차단
3. **폼을 `useState`로 관리** → LIB-01 + FM-03 차단

## Documentation Structure

```
frontend-conventions/
├── README.md           ← 이 파일 (공개 진입점)
├── SKILL.md            ← Claude Code skill 진입점 (canonical)
├── INDEX.md            ← 73개 규칙 전체 인덱스
├── DECISIONS.md        ← ADR 형식 설계 결정 (19 ADRs + Open Questions)
├── CHANGELOG.md        ← 버전별 변경사항
└── rules/              ← 12개 카테고리별 규칙
    ├── library-choices.md     ⭐ NEW (LIB-01~06)
    ├── architecture.md        (FSD skill 위임 후 PS-03,05,08,12만)
    ├── nextjs-patterns.md
    ├── code-quality.md
    ├── queries.md
    ├── mutations.md
    ├── api-layer.md
    ├── forms.md
    ├── tables.md
    ├── state-management.md    (SM-01 재작성, SM-10~13 신규 Zustand BP)
    ├── error-handling.md
    └── naming.md
```

## Severity Levels

| 레벨 | 의미 | AI 행동 |
|------|------|---------|
| 🚫 **MUST** | 절대 권고 | 항상 이 패턴, 사용자 override만 허용 (경고 후 진행) |
| ⚠️ **SHOULD** | 강한 권고 | 기본 적용, 예외 시 주석으로 이유 명시 |
| ✅ **MAY** | 권장 | 상황에 맞게 판단 |

## Contributing

이 repo는 개인 harness이지만 PR/이슈 환영합니다.

### 새 규칙 제안

1. 이슈로 제안 (왜 필요한지, 어떤 문제 해결)
2. Rule format 따라 작성 — 모든 규칙에 `Default / Why this choice / Override policy` 구조 필수
3. 해당 카테고리 파일에 추가 또는 새 파일 생성
4. SKILL.md, INDEX.md, CHANGELOG.md 업데이트
5. PR

### 규칙 수정

- 기존 규칙 ID(예: DF-07)는 유지 (참조 보존)
- Severity 변경(SHOULD → MUST)은 breaking change → 메이저 버전

### Versioning

- `major` — 규칙 삭제, severity 강화 (SHOULD → MUST)
- `minor` — 규칙 추가, library 추가
- `patch` — 예시·문구 개선, 오타 수정

## References

- [Anthropic Claude Skills](https://code.claude.com/docs/en/skills)
- [Anthropic Skill best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Feature-Sliced Design](https://feature-sliced.design)
- [TanStack Query v5](https://tanstack.com/query/latest)
- [TanStack Table v8](https://tanstack.com/table/latest)
- [TkDodo — Working with Zustand](https://tkdodo.eu/blog/working-with-zustand)

## Version

- **v1.1.0** (현재) — Library Choices 카테고리 신설, Zustand BP 추가, FSD 위임, Override Policy 명문화
- v1.0.0 — 초기 release (67 rules, 11 categories)

## License

MIT
