@AGENTS.md

## Claude Code Specific

### Skills Installation

이 레포는 `npx skills add kimzeze/frontend-conventions`로 설치 가능한 Claude Code skill입니다.
설치 시 `.claude/skills/frontend-conventions/`에 배치되며, AI가 자동으로 로드합니다.

### Validation Command

프로젝트를 이 컨벤션 기준으로 검증할 때:

> "이 프로젝트를 frontend-conventions 기준으로 검증해줘"

AGENTS.md의 **Validation Workflow** 섹션에 정의된 9단계를 순서대로 수행하고,
결과는 위반 목록 + 카테고리별 준수율 + 우선순위 권고 형식으로 출력.

### Scoped Rules

파일 작업 시 해당 영역 규칙을 우선 참조:

- `entities/*` 수정 → `rules/architecture.md`, `rules/queries.md`, `rules/mutations.md`, `rules/api-layer.md`, `rules/naming.md`
- `features/*` 수정 → `rules/forms.md`, `rules/architecture.md` (PS-07)
- `widgets/*` 수정 → `rules/tables.md`, `rules/state-management.md`
- `app/*` 수정 → `rules/nextjs-patterns.md` (PS-04, PS-11, PS-14)

### Plan Mode

컨벤션 규칙을 위반할 가능성이 있는 리팩토링 작업(3+ 파일 수정)은 plan mode로 먼저 계획 수립 후 진행.
