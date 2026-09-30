# HelpNest Docs

> AI 고객상담(CS) 헬프데스크 · 티켓 관리 시스템 **HelpNest**의 기획/설계 문서 저장소 (`helpnest-docs`)

## ⚠️ 작업 전 반드시 읽을 것 (최우선 규칙)

1. **push 하기 전에 pull 할 것이 있는지 먼저 확인한다.** (`git fetch` → 뒤처졌으면 `git pull` → 그 다음 `push`)
2. **내가 만들지 않은 파일/코드는 수정하지 않는다.** 필요하면 소유자에게 Issue(`change-request`)로 요청한다.
3. **머지 순서는 항상 `feature/*` → `dev` → `main`.** `dev`/`main` 직접 push·force push 금지.
4. **AWS 배포 전까지 DB는 각자 로컬 PostgreSQL**(docker-compose)로 테스트한다.
5. **디자인 토큰과 공용 UI(`components/ui`)만 사용**한다. 색 hex·임의 Tailwind 팔레트 직접 사용 금지.

자세한 절차는 [01_collaboration-rules.md](docs/01_collaboration-rules.md) 참고.

## 문서 맵

| 문서 | 내용 | 주 독자 |
|---|---|---|
| [PRD.md](PRD.md) | 제품 요구사항 전체(목표, 사용자, 기능, 상태 전이, SLA, 범위, 가정 사항) | 전원 |
| [docs/01_collaboration-rules.md](docs/01_collaboration-rules.md) | Git 브랜치·머지·pull 확인·파일 소유권·커밋/PR 규칙, CODEOWNERS | 전원 (필독) |
| [docs/02_architecture.md](docs/02_architecture.md) | 기술 스택, repo 구조(파일 소유 표시), 도메인 간 이벤트/포트 계약, 로컬 환경 | 전원 |
| [docs/03_database.md](docs/03_database.md) | ERD, 테이블 명세(소유자), PostgreSQL DDL, SLA/처리시간 쿼리 | 전원 |
| [docs/04_api-spec.md](docs/04_api-spec.md) | REST API 목록, 공통 응답, WebSocket(STOMP) 목적지 | 전원 |
| [docs/05_ai-automation.md](docs/05_ai-automation.md) | LLM 분류·답변 초안·자동 메일 설계, 프롬프트, 실패 처리 | 신수진 (백성준·박민재 참고) |
| [docs/06_roadmap.md](docs/06_roadmap.md) | 9/29 ~ 10/21 스프린트 일정, 담당자별 체크리스트 | 전원 |
| [docs/07_shrimp-task-guide.md](docs/07_shrimp-task-guide.md) | Shrimp Task Manager 설정·사용 순서·프롬프트 예시 | 전원 |
| [docs/08_design-system.md](docs/08_design-system.md) | 디자인 토큰·공용 UI(shadcn)·배지·화면 상태 패턴·agent/skill 운영 | 전원 (프론트 필독) |
| [docs/09_screen-spec.md](docs/09_screen-spec.md) | 화면 목록(ID·경로·권한·담당·구성·API), 사이트맵, 사용자 흐름 | 전원 |
| [docs/10_coding-conventions.md](docs/10_coding-conventions.md) | Front/Back 코딩 컨벤션, 테스트, GitHub Actions CI | 전원 |

## 저장소 구성

> GitHub: **https://github.com/BaekSeongJun** (개인 계정, Public) — 박민재·신수진은 Collaborator로 초대받아 참여. 저장소 설정 방법은 [01 §2.3](docs/01_collaboration-rules.md#23-github-저장소-설정-3개-repo-모두-sprint-0에-백성준이-설정)

| Repo | 용도 | 브랜치 |
|---|---|---|
| `helpnest-front` | Next.js 프론트엔드 | `main`, `dev`, `feature/{이니셜}-{기능}` |
| `helpnest-back` | Spring Boot 백엔드 | `main`, `dev`, `feature/{이니셜}-{기능}` |
| `helpnest-docs` | 본 문서 저장소 | `main`, `dev`, `feature/{이니셜}-{기능}` |

## 로컬 폴더 구성

메인 디렉토리 아래에 3개 repo를 **나란히** clone한다. 메인 디렉토리는 git 저장소가 아니다.

```
helpnest/                  ← 메인 디렉토리 (git init 하지 않음)
├─ helpnest-docs/          ← git repo ① 문서
├─ helpnest-front/         ← git repo ② Next.js
└─ helpnest-back/          ← git repo ③ Spring Boot
```

```bash
mkdir helpnest && cd helpnest
git clone https://github.com/BaekSeongJun/helpnest-docs.git
git clone https://github.com/BaekSeongJun/helpnest-front.git
git clone https://github.com/BaekSeongJun/helpnest-back.git
```

**이름 규칙**
| 구분 | 이름 | 비고 |
|---|---|---|
| 메인 디렉토리 | `helpnest` | 소문자 |
| repo / 폴더 | `helpnest-docs`, `helpnest-front`, `helpnest-back` | **소문자 kebab-case** — `create-next-app`은 폴더 이름을 npm 패키지명으로 쓰는데, 대문자가 있으면 생성 실패 |
| front `package.json` name | `helpnest-front` | |
| back Maven | `groupId: com.helpnest`, `artifactId: helpnest-back` | Java 패키지 `com.helpnest` (소문자) |

> 대문자 제한은 프로젝트(패키지) 이름에만 해당한다. 컴포넌트 파일(`TicketTable.tsx`)·Java 클래스(`TicketService.java`)는 PascalCase 그대로 쓴다 ([10 컨벤션](docs/10_coding-conventions.md)).

**프론트 생성 (백성준, S0)**
```bash
cd helpnest/helpnest-front
npx create-next-app@latest .     # "." = 현재 폴더에 생성
```

**clone 후 1회 설정**
```bash
cd helpnest
cp -r helpnest-docs/claude/. .                 # 공용 CLAUDE.md + .claude/(skills·agents) 복사
echo "- 나는 HelpNest 박민재(PMJ) 담당이다." > CLAUDE.local.md   # 본인 이름·이니셜로
for r in helpnest-docs helpnest-front helpnest-back; do git -C $r config core.hooksPath .githooks; done
```
- `helpnest-docs/claude/`가 바뀌면(pull 후) 위 `cp` 명령을 다시 실행한다. 메인 디렉토리의 사본은 직접 고치지 않는다.

**주의**
- 메인 디렉토리는 git 저장소가 아니다. git 명령은 **repo마다 따로** 실행한다 (`git -C helpnest-back status` 또는 해당 폴더 안에서).
- Claude Code는 **메인 디렉토리 `helpnest/`에서 실행**한다. 3개 repo가 모두 보이므로 남의 파일 수정 위험이 커진다 → 공용 `CLAUDE.md`의 소유권 규칙과 `/git-commit` 스킬의 소유권 검사를 따른다.
- Shrimp 데이터는 메인 디렉토리 한 곳(`helpnest/.shrimp`) — [07 가이드](docs/07_shrimp-task-guide.md)
- 백엔드 IDE는 IntelliJ(백성준)·Eclipse/STS(박민재·신수진) 혼용 → 초기 설정은 [10 §3.5](docs/10_coding-conventions.md#35-ide-환경-통일-intellij-1명--eclipsests-2명)

## 팀 역할 요약

| 담당 | 영역 |
|---|---|
| **백성준** (BSJ · `@BaekSeongJun`) | 고객 문의 접수 화면 · 첨부 · FAQ · 답변 템플릿 · 만족도 설문·설문 결과 조회 · 로그인/권한·비밀번호 찾기/변경 · 스팸 방지 · 고객 이력 묶음 · **공용 파일(레이아웃/보안/공통/디자인 토큰·공용 UI/.claude) 전담** |
| **박민재** (PMJ · `@totot03`) | 티켓 CRUD·상태 전이 · 자동/수동 배정 · SLA · 알림(WebSocket, 고객 답변 알림) · 실시간 1:1 채팅(대기열) |
| **신수진** (SSJ · `@s-sujin-99`) | AI 분류·답변 초안 · 처리현황 대시보드 · 월간 리포트(CSV) · S3/SES/LLM·메일 인프라 · CI · AWS 배포 |

## Claude Code 설정 (`claude/`, 백성준)

Claude 관련 공용 설정의 **원본은 이 repo의 `claude/`** 에 둔다. 메인 디렉토리는 git 저장소가 아니라서 여기에 커밋해 공유하고, 각자 메인 디렉토리로 복사해 쓴다 (위 "clone 후 1회 설정").

| 경로 | 내용 |
|---|---|
| `claude/CLAUDE.md` | 공용 작업 규칙 (→ `helpnest/CLAUDE.md`) |
| `claude/.claude/skills/` | 공용 skill (→ `helpnest/.claude/skills/`) — 목록은 [08 §12](docs/08_design-system.md#12-claude-code-agentskill-claude-백성준) |
| `claude/.claude/agents/` | 공용 agent (→ `helpnest/.claude/agents/`) |
| `helpnest/CLAUDE.local.md` | **개인용**(복사 대상 아님): "나는 X 담당이다" |
