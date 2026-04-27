# Library Choices

이 harness는 다른 BP skill들이 보여주는 **여러 가능한 라이브러리 선택지** 중 단 하나를 고정한다. 모든 코드 생성과 리뷰의 기반.

> **이 카테고리의 목적**: AI가 "Best Practice를 따르라"는 모호한 지시를 받았을 때 **여러 valid 옵션 사이에서 흔들리지 않도록** 한 방향만 박아둔다.

---

## LIB-01 폼 라이브러리 (🚫 MUST)

**규칙**: 모든 폼은 `react-hook-form` + `zod`로 구현한다.

**Default**: `react-hook-form` (form state) + `zod` (schema/validation) + `@hookform/resolvers/zod` (연결).

**차단되는 대안**:
- Formik / react-final-form / TanStack Form
- yup / valibot / joi / superstruct (validation)
- `useState`로 폼 상태 수동 관리 + 수동 validation
- HTML form `required` 속성에만 의존

**Why this choice**:
- `react-hook-form`은 uncontrolled 기반으로 리렌더 최소화 — Formik 대비 성능 우위
- `zod`는 타입 추론(`z.infer`)이 가장 강력 — yup/valibot 대비 TypeScript DX 우위
- `shadcn/ui`의 `Form` 컴포넌트가 RHF + zodResolver 전제

**Override policy** (Q4-B): 사용자가 명시적으로 다른 라이브러리 요청 시 경고 후 진행:
> "LIB-01은 react-hook-form + zod를 권장합니다. 요청대로 [대안] 사용합니다. 다른 폼은 default 따름."

**연결 규칙**: FM-01 (schema 위치), FM-02 (z.infer), FM-03 (useForm + zodResolver).

---

## LIB-02 데이터 페칭 (🚫 MUST) — 최강 규칙

**규칙**: 모든 클라이언트 사이드 서버 상태는 `@tanstack/react-query` v5로 관리한다. **`useEffect`로 데이터 페칭 절대 금지.**

**Default**: TanStack Query v5 (`useQuery`, `useMutation`, `queryOptions` factory).

**차단되는 대안**:
- SWR / RTK Query / Apollo Client (다른 server-state 라이브러리)
- `useEffect(() => { fetch(...).then(setState) }, [])` 패턴 ❌❌❌
- `useState` + `axios` 직접 조합
- Server Action만으로 처리 (우리 스택은 별도 Django backend → SA 미사용)

**Why this choice**:
- TanStack Query는 cache, refetch, invalidation, race condition 처리 모두 자동
- `useEffect` 패턴은 race condition·중복 요청·cache 부재로 근본적 결함
- Next.js App Router의 RSC와도 hydration 통해 자연스럽게 결합

**🚨 AI 실수 #1 차단용**: 이 규칙은 가장 흔한 AI 실수("시키지 않은 useEffect 데이터 페칭")를 막기 위해 가장 강하게 박힘. **Override policy 적용 시에도 반드시 사용자에게 race condition 위험을 경고.**

**연결 규칙**: DF-01 (queryOptions factory), DF-02 (queryKey 의존성), DF-07 (useApiMutation 래퍼).

---

## LIB-03 URL 상태 (🚫 MUST)

**규칙**: URL 동기화가 필요한 상태(검색·페이지·정렬·필터·탭 등)는 `nuqs`로 관리한다. `useSearchParams` 직접 사용 금지.

**Default**: `nuqs` v2+ (`useQueryState`, `useQueryStates`, `parseAs*` parser).

**차단되는 대안**:
- `useSearchParams` + `router.push(...)` 직접 사용 (보일러플레이트 + 타입 누락)
- `qs` / `query-string` 라이브러리
- URL state를 `useState`로 관리 (새로고침 시 유실)

**Why this choice**:
- Type-safe parser 시스템 (`parseAsInteger`, `parseAsString`, `parseAsArrayOf` 등)
- `useState`-like 선언적 API — 보일러플레이트 제거
- Multi-param batching (`useQueryStates`)
- Shallow routing + clearOnDefault + 6KB gzipped

**예외 (명문화된 단 하나)**:
**고빈도 업데이트** (드래그·슬라이더·실시간 동기화)는 nuqs 직접 호출 시 성능 저하. `useState` mirror + 디바운스 → nuqs 동기화 2-tier 패턴 사용:

```tsx
const [draftValue, setDraftValue] = useState('')
const [, setUrlValue] = useQueryState('q', parseAsString)
const debouncedSync = useDebouncedCallback((v: string) => setUrlValue(v), 300)

<Input
  value={draftValue}
  onChange={(e) => {
    setDraftValue(e.target.value)
    debouncedSync(e.target.value)
  }}
/>
```

**연결 규칙**: SM-02 (nuqs 옵션 가이드), SM-03 (useTableState 훅).

---

## LIB-04 테이블 (🚫 MUST)

**규칙**: 모든 데이터 테이블은 `@tanstack/react-table` v8 + 우리 `DataTable` 래퍼로 구현한다.

**Default**: TanStack Table v8 (headless) + `shared/ui/data-table` (스타일링 래퍼).

**차단되는 대안**:
- ag-grid / MUI DataGrid / react-table v7 / handsontable
- HTML `<table>` 직접 구현
- 페이지네이션을 클라이언트 사이드로만 처리 (우리는 server-side)

**Why this choice**:
- Headless — shadcn/ui 디자인 토큰과 자연스럽게 통합
- TypeScript 우위, ColumnDef 타입 추론 강력
- Server-side pagination/sorting/filtering 표준 지원 (`manualPagination: true`)

**연결 규칙**: TB-01 (DataTable props 구조), TB-03 (manual mode).

---

## LIB-05 Toast 알림 (🚫 MUST)

**규칙**: 사용자 피드백 toast는 `sonner` 사용.

**Default**: `sonner` v2+ (`toast.success`, `toast.error`, `toast.info`).

**차단되는 대안**:
- `react-hot-toast` / `react-toastify` / `notistack`
- 자체 toast 컴포넌트 직접 구현
- `alert()` / `confirm()` (UX 저열)

**Why this choice**:
- shadcn/ui 공식 권장 (Sonner adapter 제공)
- 가장 폴리시드된 디자인 + a11y 기본 지원
- Promise toast (로딩 → 성공/실패 자동 전환) 내장

**연결 규칙**: EH-01 (ApiError + toast 표시), DF-06 (mutation 성공 toast).

---

## LIB-06 UI 컴포넌트 + 스타일 (🚫 MUST)

**규칙**: UI 프레임워크는 `shadcn/ui` (radix 기반) + `Tailwind CSS` v4. 우리 모노레포에서는 `@workspace/ui` 패키지를 통해 공유.

**Default**:
- 컴포넌트 베이스: `radix-ui` (headless primitives)
- 스타일링: Tailwind CSS v4
- 컴포넌트 라이브러리: `shadcn/ui` 패턴 (복사-붙여넣기 + 커스터마이즈)
- 모노레포 공유 위치: `@workspace/ui/components/*`

**차단되는 대안**:
- MUI / Chakra UI / Mantine / Ant Design / Bootstrap
- Tailwind 외 CSS-in-JS (styled-components, emotion 등)
- CSS Modules 단독 사용

**Why this choice**:
- Headless + Tailwind 조합이 가장 유연 (디자인 토큰 직접 제어)
- `shadcn/ui`는 라이브러리 잠금이 약함 (코드를 직접 소유)
- Next.js + RSC 호환 (zero-runtime CSS)

**예외**:
- 차트는 `recharts` 또는 `tremor` 허용 (shadcn/ui 영역 밖)
- 애니메이션은 `framer-motion` 또는 `motion` 허용 (필요 시)

---

## 라이브러리 선택 매트릭스 (요약)

| 영역 | 우리 선택 | 차단되는 대안 |
|------|----------|-----------|
| 폼 + validation | react-hook-form + zod | Formik, yup, valibot |
| 데이터 페칭 | TanStack Query v5 | SWR, RTK Query, useEffect+fetch |
| URL 상태 | nuqs | useSearchParams 직접, qs |
| 테이블 | TanStack Table v8 + DataTable 래퍼 | ag-grid, MUI DataGrid |
| Toast | sonner | react-hot-toast, react-toastify |
| UI 베이스 | shadcn/ui + radix + Tailwind v4 | MUI, Chakra, Mantine, CSS-in-JS |

---

## 새 영역 추가 시 원칙

새로운 의존성이 필요할 때 (예: 차트, 드래그앤드롭, date 처리):

1. **이미 위 6개 LIB로 해결 가능한지 먼저 검토**
2. 해결 불가 시 1개 라이브러리만 선택 (대안 비교 + 선정 사유 기록)
3. 새 LIB-XX 규칙으로 추가 (이 파일에 append)
4. CHANGELOG에 minor version bump
