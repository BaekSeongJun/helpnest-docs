# 08. 디자인 시스템 가이드 (백성준)

> **목표: 지금은 "틀"만 맞춘다.** 3명이 같은 토큰·같은 컴포넌트·같은 화면 패턴을 쓰면, 마지막에 백성준이 **토큰 값과 `components/ui` 스타일만 바꿔서** 전체 디자인을 한 번에 정리할 수 있다. 페이지 코드는 건드리지 않는다.

---

## 1. 핵심 규칙

| # | 규칙 | ✅ 이렇게 | ❌ 이렇게 하지 않기 |
|---|---|---|---|
| D1 | 색은 **의미 토큰**만 사용 | `bg-primary`, `text-muted-foreground`, `border-border` | `bg-indigo-600`, `text-[#6b7280]`, `style={{color:'red'}}` |
| D2 | UI 기본 요소는 `components/ui`(shadcn) 사용 | `<Button variant="outline">` | 직접 만든 `<button className="...">` |
| D3 | 화면 상태는 공용 패턴 컴포넌트 사용 | `<EmptyState>`, `<ErrorState>`, `<LoadingSkeleton>` | 페이지마다 다른 "데이터 없음" 문구·모양 |
| D4 | 상태·우선순위·감정 표시는 공용 배지 | `<StatusBadge status="IN_PROGRESS" />` | 페이지에서 색 직접 지정 |
| D5 | 간격·크기는 Tailwind 기본 스케일 | `gap-4`, `p-6` | `p-[13px]`, `mt-[7px]` |
| D6 | 날짜·숫자는 공용 포맷 함수 | `formatDateTime(v)` | 컴포넌트마다 `toLocaleString()` |
| D7 | 새 UI 기본 요소가 필요하면 **백성준에게 CR** | `[CR][PMJ→BSJ] shadcn tabs 추가 요청` | 각자 `npx shadcn add` 실행 (→ `components/ui` 오염) |

> 도메인 전용 **조합 컴포넌트**(예: `TicketTable`, `AiDraftButton`)는 각자 자기 폴더에 자유롭게 만들되, 내부는 `components/ui` + 토큰만 쓴다.

---

## 2. 소유권

| 경로 | 소유 | 내용 |
|---|---|---|
| `src/app/globals.css` | 백성준 | 디자인 토큰(CSS 변수, 라이트/다크) |
| `components.json` | 백성준 | shadcn 설정 |
| `src/components/ui/**` | 백성준 | shadcn 원본 컴포넌트 (**직접 수정 금지**) |
| `src/components/common/**` | 백성준 | 공용 패턴·배지 컴포넌트 |
| `src/components/layout/**` | 백성준 | Header, Sidebar, Footer |
| `src/config/badge.ts` | 백성준 | 상태 → 배지 매핑 |
| `src/lib/format.ts`, `src/lib/utils.ts`(`cn`) | 백성준 | 포맷·클래스 유틸 |
| `helpnest-docs/claude/.claude/agents/**`, `…/skills/**` | 백성준 | 공용 Claude Code agent·skill (§12) |

---

## 3. 디자인 토큰 (틀 + 초기값)

> 초기값은 **퍼플/인디고(AI 강조)** 톤의 임시값이다. 최종값은 Sprint 3 이후 조정(PRD Q17).
> 형식은 shadcn/ui + Tailwind CSS 변수(`:root`, `.dark`). 아래 이름만 고정하고 값은 바뀔 수 있다.

### 3.1 색 (의미 토큰)
| 토큰 | Tailwind 클래스 | 용도 | 초기값(라이트, 참고) |
|---|---|---|---|
| `--background` / `--foreground` | `bg-background` `text-foreground` | 페이지 바탕/기본 글자 | white / slate-900 |
| `--card` / `--card-foreground` | `bg-card` | 카드·패널 | white |
| `--muted` / `--muted-foreground` | `bg-muted` `text-muted-foreground` | 보조 배경·설명 글자 | slate-100 / slate-500 |
| `--primary` / `--primary-foreground` | `bg-primary` `text-primary-foreground` | 주요 버튼·링크·선택 상태 | **indigo-600** / white |
| `--secondary` | `bg-secondary` | 보조 버튼 | indigo-50 |
| `--accent` | `bg-accent` | hover·선택 행 | indigo-50 |
| `--ai` / `--ai-foreground` | `bg-ai` `text-ai` | **AI 기능 전용 강조**(AI 패널, 초안 버튼, AI 배지) | violet-600 |
| `--success` | `bg-success` `text-success` | 해결·성공 | emerald-600 |
| `--warning` | `bg-warning` `text-warning` | SLA 임박·주의 | amber-500 |
| `--destructive` | `bg-destructive` | 삭제·SLA 초과·긴급·오류 | red-600 |
| `--info` | `bg-info` `text-info` | 배정·안내 | sky-600 |
| `--border` / `--input` / `--ring` | `border-border` `ring-ring` | 테두리·입력·포커스 | slate-200 / slate-200 / indigo-500 |

> `--ai`, `--success`, `--warning`, `--info`는 shadcn 기본에 없는 **커스텀 토큰** — 백성준이 S0에 `globals.css`에 추가한다.

### 3.2 타이포그래피
| 이름 | 클래스 | 용도 |
|---|---|---|
| 페이지 제목 | `text-2xl font-semibold` | `PageHeader` 제목 |
| 섹션 제목 | `text-lg font-semibold` | 카드·패널 제목 |
| 본문 | `text-sm` | 기본 (콘솔은 정보 밀도 위해 14px 기준) |
| 보조 | `text-xs text-muted-foreground` | 메타 정보(작성일, 티켓번호) |
| KPI 숫자 | `text-3xl font-bold tabular-nums` | 대시보드 카드 |
| 글꼴 | Pretendard (웹폰트) → 없으면 system-ui | 백성준이 `layout.tsx`에서 1회 설정 |

### 3.3 간격·모양
| 항목 | 규칙 |
|---|---|
| 페이지 패딩 | `p-6` (모바일 `p-4`) |
| 섹션 간격 | `space-y-6` / 카드 내부 `p-4`~`p-6` / 요소 간 `gap-2`·`gap-4` |
| 모서리 | `--radius` 토큰 (`rounded-md` 기본, 카드 `rounded-lg`) |
| 그림자 | 카드 `shadow-sm`만. 모달은 shadcn 기본 |
| 아이콘 | **lucide-react**만, 크기 `size-4`(본문) / `size-5`(헤더) |

---

## 4. 레이아웃

```
┌──────────────────────────── Header (h-14, 고정) ────────────────────────────┐
│ 로고 · (콘솔) 전역 검색 슬롯                    알림 벨(박민재) · 사용자 메뉴 │
├──────────────┬───────────────────────────────────────────────────────────────┤
│ Sidebar      │  PageHeader: 제목 + 설명 + 우측 액션 버튼                      │
│ (w-60,       │  ───────────────────────────────────────────                  │
│  역할별 메뉴, │  필터 바 (있을 때)                                             │
│  모바일은    │  본문 (카드 / 테이블 / 2단 레이아웃)                            │
│  Sheet)      │                                                               │
├──────────────┴───────────────────────────────────────────────────────────────┤
│ Footer (고객 화면만)                                                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

| 화면 유형 | 규칙 |
|---|---|
| 모든 페이지 | 최상단에 `<PageHeader title description actions />` |
| 고객 화면 | 본문 `max-w-3xl mx-auto` (폼·FAQ·설문은 좁게) |
| 콘솔 목록 | 전체 폭, 필터 바 + `DataTable` |
| 콘솔 상세(티켓) | 2단: 좌 `flex-1`(본문·타임라인·에디터) / 우 `w-80`(정보·AI 패널·고객 이력) — 1024px 미만은 1단 |
| 대시보드 | KPI 카드 4열 그리드(`grid-cols-2 lg:grid-cols-4`) → 차트 2열 → 표 |

---

## 5. 컴포넌트 목록

### 5.1 shadcn/ui (S0에 백성준이 일괄 설치)
`button`, `input`, `textarea`, `label`, `select`, `checkbox`, `radio-group`, `switch`, `form`, `card`, `badge`, `table`, `dialog`, `alert-dialog`, `sheet`, `dropdown-menu`, `popover`, `tooltip`, `tabs`, `separator`, `skeleton`, `avatar`, `scroll-area`, `pagination`, `sonner`(토스트), `calendar`/`date-picker`(기간 필터), `command`(검색 선택)

### 5.2 공용 조합 컴포넌트 (백성준, `components/common`)
| 컴포넌트 | 주요 props | 용도 |
|---|---|---|
| `PageHeader` | `title, description?, actions?` | 모든 페이지 상단 |
| `DataTable` | `columns, data, loading, emptyText, pagination, onRowClick` | 목록 화면 공통 |
| `FilterBar` | `children, onReset` | 목록 필터 영역 |
| `EmptyState` | `icon?, title, description?, action?` | 데이터 없음 |
| `ErrorState` | `message?, onRetry` | 조회 실패 |
| `LoadingSkeleton` | `variant: 'table' \| 'card' \| 'detail'` | 로딩 |
| `ConfirmDialog` | `title, description, confirmText, destructive?, onConfirm` | 삭제·상태 변경 확인 |
| `StatusBadge` | `status: TicketStatus` | 티켓 상태 |
| `PriorityBadge` | `priority` | 우선순위 |
| `SentimentBadge` | `sentiment` | 불만 표시 |
| `SlaBadge` | `dueAt, respondedAt, breached` | SLA 남은 시간/초과 |
| `AiBadge` | - | "AI" 표시 (분류 결과, 초안) |
| `PlainText` | `text` | 일반 텍스트 본문 출력 (이스케이프 + 줄바꿈 + URL 링크) |
| `FileList` / `FileUploader` | - | 첨부 표시·업로드 |
| `KpiCard` | 신수진 소유 (`components/dashboard`) — 단 `Card` + 토큰만 사용 | 대시보드 |

---

## 6. 배지 매핑 (`config/badge.ts`, 백성준)

| 구분 | 값 | 표시 | 토큰 |
|---|---|---|---|
| 상태 | RECEIVED | 접수 | muted |
| | ASSIGNED | 배정 | info |
| | IN_PROGRESS | 처리중 | primary |
| | RESOLVED | 해결 | success |
| | CLOSED | 종료 | muted (outline) |
| 우선순위 | URGENT | 긴급 | destructive |
| | HIGH | 높음 | warning |
| | NORMAL | 보통 | secondary |
| | LOW | 낮음 | muted (outline) |
| 감정 | NEGATIVE | 불만 | destructive (outline) |
| SLA | 여유 / 임박(80%) / 초과 | 남은 시간 / "임박" / "초과" | muted / warning / destructive |
| 채팅 | WAITING / OPEN / CLOSED | 대기 / 상담중 / 종료 | warning / primary / muted |

> 한글 라벨도 이 파일에서만 정의한다(각 페이지에서 `'처리중'` 문자열 하드코딩 금지).

---

## 7. 화면 상태 패턴

| 상황 | 패턴 |
|---|---|
| 첫 로딩 | `LoadingSkeleton` (스피너 단독 사용 금지) |
| 버튼 처리 중 | 버튼 `disabled` + 아이콘 `Loader2 animate-spin` + 문구 유지 |
| 데이터 없음 | `EmptyState` — 제목 1줄 + 다음 행동 버튼 (예: "문의가 없습니다" + [문의하기]) |
| 조회 실패 | `ErrorState` + 다시 시도 |
| 저장 성공 | 토스트(sonner) 우하단, 3초 |
| 저장 실패 | 폼 필드 에러는 필드 아래 빨간 글자, 서버 에러는 토스트(`error.message` 그대로) |
| 위험 동작 | `ConfirmDialog` (삭제, 해결 처리, 재배정, 채팅 종료) |
| 권한 없음 | 403 공용 페이지(백성준) |
| 실시간 갱신 | 목록 상단에 "새 티켓 N건" 배너 → 클릭 시 새로고침 (자동 재정렬로 행이 튀지 않게) |

---

## 8. 폼 규칙
- `react-hook-form` + `zod` + shadcn `Form` (PRD Q18)
- 라벨은 입력 위, 필수 항목은 라벨 옆 `*`
- 검증 메시지 톤: "제목을 입력해 주세요", "10MB 이하 파일만 올릴 수 있어요" (해요체)
- 제출 버튼은 폼 우측 하단, 주요 버튼 1개만 `primary`
- 텍스트 영역은 글자 수 카운터 표시 (본문 5,000자)

## 9. 테이블 규칙
- 기본 20행, 서버 페이지네이션
- 첫 열: 식별자(티켓번호 등) `font-mono text-xs`
- 상태/우선순위 열은 배지, 날짜 열은 상대시간("3분 전") + tooltip 절대시간
- 행 클릭 = 상세 이동, 행 안 버튼은 클릭 전파 막기

## 10. 텍스트·포맷 규칙 (`lib/format.ts`)
| 대상 | 형식 | 예 |
|---|---|---|
| 날짜시간 | `yyyy.MM.dd HH:mm` | 2026.10.02 14:05 |
| 날짜 | `yyyy.MM.dd` | 2026.10.02 |
| 상대시간(24시간 이내) | `n분 전`, `n시간 전` | 3분 전 |
| 소요시간 | `n시간 m분` | 3시간 12분 |
| 숫자 | 천 단위 콤마, 비율 소수 1자리 `%` | 1,204 / 12.5% |
| 시간대 | 표시는 `Asia/Seoul` 고정 | |
| 문구 톤 | 고객 화면 해요체, 콘솔 간결체("저장됨", "배정 완료") | |

## 11. 반응형·접근성
- 고객 화면: 모바일(360px) ~ 데스크톱 대응
- 콘솔: 1280px 기준 데스크톱 우선, 1024px 미만은 1단 레이아웃까지만 보장
- 색만으로 상태를 구분하지 않는다 (배지에 항상 글자)
- 모든 아이콘 버튼에 `aria-label`, 포커스 링(`ring`) 제거 금지

## 12. Claude Code agent·skill (`claude/`, 백성준)
| 항목 | 규칙 |
|---|---|
| 위치 | 원본 `helpnest-docs/claude/.claude/agents/`, `…/skills/` (docs repo에 커밋) → 각자 메인 디렉토리 `helpnest/.claude/`로 복사해 사용 ([README](../README.md#claude-code-설정-claude-백성준)) |
| 추가·수정 | 백성준만. 필요하면 CR Issue |
| 목록 | 아래 표에 기록 (받아오는 대로 백성준이 채움) |
| 사용 원칙 | agent·skill이 만든 코드도 D1~D7 규칙을 따라야 함. `components/ui`, `globals.css`를 바꾸려 하면 중단하고 CR 초안만 출력 |

| 이름 | 종류 | 출처 | 용도 | 사용자 |
|---|---|---|---|---|
| `git-commit` | skill | 백성준 작성 | 브랜치·소유권·비밀값 검사 후 01 §5.1 형식으로 repo별 커밋, push는 요청 시 R1 절차 | 전원 |

## 13. 최종 디자인 정리 절차 (PRD Q17)
1. Sprint 3 종료 시점에 3명 모두 D1~D7 위반 검사 (§14 명령어) → 각자 자기 파일 수정
2. 백성준이 `globals.css` 토큰 값, `components/ui` 스타일, 폰트, 로고 조정 (feature/BSJ-design-polish)
3. 3명이 각자 화면 스크린샷 확인 → 깨진 곳은 해당 소유자가 수정
4. 메일 템플릿(신수진)의 색·로고를 최종 토큰 값에 맞춤

## 14. PR 전 UI 셀프 체크
```bash
# 하드코딩 색상/임의 값 검사 (결과가 나오면 토큰으로 교체)
grep -rnE "#[0-9a-fA-F]{3,6}|(bg|text|border)-(red|blue|indigo|violet|purple|green|gray|slate|amber|emerald|sky)-[0-9]{2,3}|\[[0-9]+px\]" src/app src/components --include=*.tsx | grep -v "components/ui"
```
- [ ] 새 화면에 `PageHeader`, 로딩/빈/에러 상태 모두 있음
- [ ] 배지·날짜 포맷은 공용 사용
- [ ] 모바일 폭(고객 화면) 확인
