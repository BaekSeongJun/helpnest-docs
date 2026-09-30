# 01. 협업 규칙 (Git · 파일 소유권 · Claude Code)

> **이 문서는 PRD의 모든 기능보다 우선한다.** Claude Code에게 작업을 시킬 때도 이 문서를 먼저 읽히고 시작한다.

---

## 1. 최우선 규칙 요약

| 순위 | 규칙 | 한 줄 요약 |
|:-:|---|---|
| 1 | **push 전 pull 확인** | `fetch → status(behind?) → pull → 충돌해결·빌드 → push` |
| 2 | **본인 파일만 수정** | 남의 파일은 읽기만. 변경은 Issue로 요청 |
| 3 | **feature → dev → main** | PR로만 머지, dev/main 직접 push·force push 금지 |
| 4 | **로컬 DB** | AWS 배포 전까지 로컬 PostgreSQL |
| 5 | **디자인 토큰·공용 UI** | 색·간격 하드코딩 금지, `components/ui`·`common` 재사용 ([08](08_design-system.md)) |

---

## 2. 저장소와 브랜치

### 2.1 저장소 (3개, 각자 독립된 main/dev 보유)
| Repo | 내용 |
|---|---|
| `helpnest-front` | Next.js |
| `helpnest-back` | Spring Boot |
| `helpnest-docs` | PRD 및 설계 문서 |

### 2.2 브랜치 전략
```
main  ← 검증 완료본(마일스톤 단위). 배포 기준
 └─ dev  ← 통합/테스트 브랜치
     ├─ feature/BSJ-login
     ├─ feature/PMJ-ticket-state
     └─ feature/SSJ-ai-classify
```
- **이름 규칙:** `feature/{이니셜}-{기능}` (이니셜은 대문자 3자, 기능은 소문자 kebab-case)

  | 담당자 | 이니셜 |
  |---|---|
  | 백성준 | `BSJ` |
  | 박민재 | `PMJ` |
  | 신수진 | `SSJ` |

  예) `feature/BSJ-inquiry-form`, `feature/PMJ-sla-scheduler`, `feature/SSJ-dashboard`
- feature 브랜치는 **항상 최신 `dev`에서 생성**한다.
- 버그 수정도 예외 없이 `feature/{이니셜}-fix-{내용}` → dev → main.
- 하나의 feature 브랜치 = 한 사람 소유. 다른 사람의 feature 브랜치에 커밋하지 않는다.

### 2.3 GitHub 저장소 설정 (3개 repo 모두, Sprint 0에 **백성준**이 설정)

저장소 위치: **https://github.com/BaekSeongJun** (개인 계정, **Public**)

| Repo | URL |
|---|---|
| helpnest-docs | https://github.com/BaekSeongJun/helpnest-docs |
| helpnest-front | https://github.com/BaekSeongJun/helpnest-front |
| helpnest-back | https://github.com/BaekSeongJun/helpnest-back |

> 개인 계정 repo는 **소유자(백성준)만** Settings(협업자 초대, 브랜치 보호, Actions 설정)를 바꿀 수 있다. 협업자는 Write 권한(push·PR·리뷰·Issue)만 가진다.
> Public repo라서 GitHub Free에서도 브랜치 보호와 Actions를 무료로 쓸 수 있다. 대신 **누구나 코드를 볼 수 있으므로 비밀값(.env, API 키, AWS 키) 커밋은 절대 금지**.

**① 협업자 초대 (백성준)** — 각 repo(3개 모두) → Settings → Collaborators → Add people → **`totot03`(박민재)**, **`s-sujin-99`(신수진)** 초대 → 두 사람이 메일/알림에서 초대 수락

**② 브랜치 만들기 (백성준)** — 각 repo 첫 커밋(README 등)을 `main`에 올린 뒤 `dev` 생성·push, Settings → General → Default branch를 **`dev`**로 변경 (PR 기본 대상이 dev가 되도록)

**③ 브랜치 보호 (백성준)** — Settings → Branches → Add branch protection rule (또는 Rulesets)

| 브랜치 | 설정 |
|---|---|
| `main` | Require PR, 직접 push 금지, force push 금지, 삭제 금지, **dev에서 온 PR만 허용**(리뷰 시 확인) |
| `dev` | Require PR, 승인 1명 이상(작성자 외), force push 금지, **Required status checks: CI**(신수진의 워크플로가 첫 실행된 후 백성준이 등록) |
| 참고 | `helpnest-docs`는 CI가 없으므로 Required status checks 없이 PR·승인 규칙만 적용 |
| 공통 | **Do not allow bypassing the above settings**(관리자 포함) 체크 — 소유자인 백성준도 같은 규칙을 따르도록 |

> CODEOWNERS(§4.4)는 **리뷰어 자동 지정과 소유권 표시** 용도로 쓴다. "Require review from Code Owners"는 켜지 않는다 — 자기 파일만 수정한 PR은 작성자 본인이 유일한 코드 오너라 승인이 불가능해지기 때문.

---

## 3. push 전 pull 확인 절차 (규칙 1)

### 3.1 매 작업 시작 시
```bash
git checkout dev
git pull origin dev                    # 최신 dev
git checkout -b feature/BSJ-inquiry-form # 새 기능이면 생성
# 이미 있는 브랜치라면
git checkout feature/BSJ-inquiry-form
git pull origin feature/BSJ-inquiry-form
git pull origin dev                    # dev 변경사항 흡수 (merge 방식)
```

### 3.2 push 직전 (매번)
```bash
git fetch origin
git status                                         # "Your branch is behind" 확인
git log --oneline HEAD..origin/$(git branch --show-current)   # 원격 내 브랜치에 새 커밋?
git log --oneline HEAD..origin/dev                 # dev에 새 커밋?

# 하나라도 출력되면
git pull origin $(git branch --show-current)
git pull origin dev
# 충돌 해결 → 빌드/테스트 통과 확인 (front: npm run build / back: ./mvnw verify, Windows cmd는 mvnw.cmd verify)
git push origin $(git branch --show-current)
```
- `git pull`은 **merge 방식**을 사용한다(공유 브랜치 이력 재작성 방지). `git push --force` 금지.

### 3.3 자동 검사 훅 (각 repo에 백성준이 추가, 각자 1회 활성화)
`.githooks/pre-push`
```sh
#!/bin/sh
branch=$(git rev-parse --abbrev-ref HEAD)

if [ "$branch" = "main" ] || [ "$branch" = "dev" ]; then
  echo "[ERROR] $branch 브랜치에 직접 push 할 수 없습니다. PR을 사용하세요."
  exit 1
fi

git fetch origin --quiet

if git rev-parse --verify --quiet "origin/$branch" >/dev/null; then
  behind=$(git rev-list --count "HEAD..origin/$branch")
  if [ "$behind" -gt 0 ]; then
    echo "[ERROR] origin/$branch 에 새 커밋 ${behind}개. 먼저 git pull origin $branch"
    exit 1
  fi
fi

behind_dev=$(git rev-list --count "HEAD..origin/dev")
if [ "$behind_dev" -gt 0 ]; then
  echo "[ERROR] origin/dev 에 새 커밋 ${behind_dev}개. 먼저 git pull origin dev 후 빌드 확인"
  exit 1
fi
echo "[OK] pull 할 내용 없음. push 진행"
```
```bash
chmod +x .githooks/pre-push
git config core.hooksPath .githooks   # 각자 clone 후 1회
```

### 3.4 충돌 해결 원칙
- 충돌 파일이 **내 소유** → 내가 해결.
- 충돌 파일이 **남의 소유** → 내 변경을 버리고 원격 버전 유지(`git checkout --theirs <file>` 또는 수동 확인) 후, 필요한 변경은 소유자에게 Issue 요청. **남의 코드를 내가 합쳐서 고치지 않는다.**
- 공용 파일(백성준 소유) 충돌은 백성준에게 알리고 백성준이 해결.

---

## 4. 파일 소유권

### 4.1 원칙
1. 파일을 **처음 만든 사람이 소유자**다. 소유자만 수정한다.
2. 공용 파일(레이아웃, 보안, 공통 응답/예외, 설정, 초기 세팅)은 **백성준 소유**.
3. 남의 파일 변경이 필요하면 → GitHub Issue 생성
   - 라벨: `change-request`, 담당자: 소유자
   - 제목: `[CR][PMJ→BSJ] Sidebar에 '채팅' 메뉴 추가 요청`
   - 본문: 필요한 이유, 원하는 변경, 급한 정도, 기한
4. 소유자는 24시간 내 응답(수락/거절/대안). 급하면 메신저로도 알린다.
5. 남의 코드는 **import·호출·읽기만** 가능하다. 확장이 필요하면 소유자에게 포트/이벤트 추가를 요청한다.
6. 새 파일 상단에 소유자 주석을 단다.
   - TS/TSX: `// @owner BSJ`
   - Java: `// @owner PMJ` (package 선언 위)
   - SQL: `-- @owner SSJ`

### 4.2 공용 파일 목록 (백성준 소유 — 다른 사람 수정 금지)
| Repo | 경로 |
|---|---|
| front | `src/app/layout.tsx`, `src/app/(main)/layout.tsx`, `src/components/layout/**`(Header, Sidebar, Footer), `src/components/ui/**`(shadcn), `src/components/common/**`(PageHeader, DataTable, 상태·우선순위 배지 등), `src/lib/api/client.ts`, `src/lib/auth/**`, `src/lib/format.ts`, `src/lib/utils.ts`, `src/middleware.ts`, `src/config/menu.ts`, `src/config/badge.ts`, `src/app/globals.css`(디자인 토큰), `components.json`, `package.json`, `tsconfig.json`, `eslint.config.*`, `.prettierrc`, `next.config.*`, `.env.example`, `.gitignore`, `.github/`(workflows 제외) |
| back | `pom.xml`, `HelpNestApplication.java`, `global/**`(config, security, common, error, util — 단 `global/websocket/`은 박민재 소유), `application.yml`, `application-local.yml`, `docker-compose.yml`, `.githooks/**`, `.editorconfig`, `.gitattributes`, `.gitignore`, `mvnw`·`.mvn/`, `.github/`(workflows 제외) |
| docs | `README.md`, `PRD.md`, `docs/01_collaboration-rules.md`, `claude/**`(공용 CLAUDE.md·agent·skill 원본), `.githooks/**`, `.github/` |

> **패키지·설정 추가 요청:** `package.json` 의존성, `pom.xml` 의존성, 사이드바 메뉴 추가는 모두 백성준에게 CR Issue로 요청한다.
> **설정 분리:** 박민재·신수진 전용 설정은 각자 파일(`application-ws.yml`(박민재), `application-ai.yml`, `application-aws.yml`(신수진))로 두고 백성준의 `application.yml`에서 `spring.config.import`로 가져온다. → 설정 파일 충돌 방지.
> **에러 코드 분리:** 공통 `ErrorCode` 인터페이스(백성준)를 각 도메인이 자기 enum(`TicketErrorCode`(박민재), `AiErrorCode`(신수진))으로 구현한다.

### 4.3 docs repo 소유권
| 파일 | 소유 |
|---|---|
| README.md, PRD.md, 01_collaboration-rules.md | 백성준 (내용 변경은 3인 합의 후 백성준 반영) |
| 02_architecture.md, 03_database.md, 04_api-spec.md | 섹션별 소유 — 각 섹션 제목의 `(백성준)`/`(박민재)`/`(신수진)` 표기 담당자만 수정 |
| 05_ai-automation.md | 신수진 |
| 06_roadmap.md | 섹션별 — **자기 이름 섹션의 체크박스만** 수정 |
| 07_shrimp-task-guide.md | 백성준 |
| 08_design-system.md | 백성준 |
| 09_screen-spec.md | 행 단위 — 각 화면 행의 "담당"만 수정, 문서 틀은 백성준 |
| 10_coding-conventions.md | 백성준 (§4 CI는 신수진) |
| claude/ | 백성준 (Claude Code 공용 설정 원본) |

### 4.4 CODEOWNERS (각 repo `.github/CODEOWNERS`, 백성준이 생성)
> GitHub 아이디: 백성준 `@BaekSeongJun` · 박민재 `@totot03` · 신수진 `@s-sujin-99` (두 사람은 Collaborator 초대를 수락해야 CODEOWNERS 지정이 유효함)

**helpnest-front**
```
*                                   @BaekSeongJun
/src/app/(main)/console/tickets/    @totot03
/src/app/(main)/console/chat/       @totot03
/src/app/(main)/chat/               @totot03
/src/app/(main)/admin/sla/          @totot03
/src/components/ticket/             @totot03
/src/components/notification/       @totot03
/src/components/chat/               @totot03
/src/lib/api/ticket.ts              @totot03
/src/lib/api/notification.ts        @totot03
/src/lib/api/chat.ts                @totot03
/src/lib/ws/                        @totot03
/src/types/ticket.ts                @totot03
/src/types/chat.ts                  @totot03
/src/app/(main)/console/dashboard/  @s-sujin-99
/src/app/(main)/console/reports/    @s-sujin-99
/src/components/ai/                 @s-sujin-99
/src/components/dashboard/          @s-sujin-99
/src/lib/api/ai.ts                  @s-sujin-99
/src/lib/api/dashboard.ts           @s-sujin-99
/src/lib/api/report.ts              @s-sujin-99
/src/types/ai.ts                    @s-sujin-99
/.github/workflows/                 @s-sujin-99
```

**helpnest-back**
```
*                                               @BaekSeongJun
/src/main/java/com/helpnest/domain/ticket/      @totot03
/src/main/java/com/helpnest/domain/assignment/  @totot03
/src/main/java/com/helpnest/domain/sla/         @totot03
/src/main/java/com/helpnest/domain/notification/ @totot03
/src/main/java/com/helpnest/domain/chat/        @totot03
/src/main/java/com/helpnest/global/websocket/   @totot03
/src/main/resources/application-ws.yml          @totot03
/src/main/java/com/helpnest/domain/ai/          @s-sujin-99
/src/main/java/com/helpnest/domain/dashboard/   @s-sujin-99
/src/main/java/com/helpnest/domain/report/      @s-sujin-99
/src/main/java/com/helpnest/infra/              @s-sujin-99
/src/main/resources/application-ai.yml          @s-sujin-99
/src/main/resources/application-aws.yml         @s-sujin-99
/src/main/resources/application-prod.yml        @s-sujin-99
/deploy/                                        @s-sujin-99
/src/main/resources/templates/mail/             @s-sujin-99
/.github/workflows/                             @s-sujin-99
```
Flyway 마이그레이션 파일은 파일명에 이니셜을 넣어 소유를 구분한다(`V202610011030__PMJ_create_ticket.sql`).

---

## 5. 커밋 · PR 규칙

### 5.1 커밋 메시지
```
<type>(<scope>): <요약> [<이니셜: BSJ|PMJ|SSJ>]

type: feat | fix | refactor | test | docs | style | chore
scope: auth, inquiry, faq, survey, ticket, sla, noti, chat, ai, dashboard, report, infra, layout ...
예) feat(ticket): 상태 전이 서비스 검증 추가 [PMJ]
    fix(ai): LLM 타임아웃 fallback 누락 수정 [SSJ]
```
- 한 커밋 = 한 가지 의미. Shrimp 태스크 1개 완료 = 커밋 1개 이상.

### 5.2 PR (feature → dev)
- 제목: `[BSJ] 문의 접수 폼 + 첨부 업로드`
- 본문 템플릿(`.github/pull_request_template.md`, 백성준 생성)
```md
## 작업 내용
-
## 관련 Shrimp 태스크 / Issue
-
## 체크리스트
- [ ] push 전 fetch/pull 확인 (origin/dev 반영 완료)
- [ ] 내 소유 파일만 수정함 (남의 파일 diff 없음)
- [ ] 로컬 빌드/테스트 통과 (CI 통과)
- [ ] (프론트) 08 디자인 시스템 §14 셀프 체크 통과 — 하드코딩 색상 없음, 로딩/빈/에러 상태 있음
- [ ] 포맷은 내가 수정한 파일에만 적용
- [ ] API 변경 시 docs/04_api-spec.md 반영 PR 생성
## 스크린샷 / 테스트 결과
```
- 리뷰어는 **"남의 파일 diff 없음"**을 가장 먼저 확인한다. 위반 시 Request changes.
- 머지 방식: **Squash and merge**(feature → dev), **Create a merge commit**(dev → main).
- 머지 후 feature 브랜치는 삭제.

### 5.3 dev → main (마일스톤)
| 마일스톤 | 날짜 | 조건 |
|---|---|---|
| M1 | 10/8(목) | 핵심 흐름(접수→분류→배정→답변→해결) dev에서 시연 성공 |
| M2 | 10/14(수) | 필수 기능 전체 통합 테스트 통과 |
| M3 | 10/18(일) | 선택 기능 포함 |
| Final | 10/21(수) | AWS 배포 검증 완료 |

절차: 3인이 dev 브랜치를 각자 로컬에서 실행해 시나리오 확인 → PR `dev → main` → 전원 확인 → 머지.

---

## 6. 로컬 개발 환경 (규칙 4)
- DB: `helpnest-back/docker-compose.yml`(백성준)로 PostgreSQL 실행. 각자 로컬 데이터는 공유하지 않는다.
- 시드 데이터: `src/main/resources/db/seed/`에 도메인별 파일(`seed_BSJ_member.sql` 등) — 각자 소유.
- 비밀값(JWT 키, LLM 키, AWS 키)은 **절대 커밋 금지**. `.env` / `application-local-secret.yml`은 `.gitignore`.
- AWS 자원(S3/SES/RDS)은 Sprint 4에 신수진이 연결. 그 전까지 로컬 프로필 구현체(로컬 디스크 저장, 메일 로그 출력)를 사용.

---

## 7. Claude Code 작업 규칙
Claude Code는 메인 디렉토리 `helpnest/`에서 실행하고, 공용 규칙은 `helpnest-docs/claude/CLAUDE.md`를 메인 디렉토리로 복사해 쓴다([README](../README.md#claude-code-설정-claude-백성준)). 담당자는 각자 `CLAUDE.local.md`에 적는다. 아래를 지킨다.
1. 작업 시작 프롬프트에 **"나는 X 담당, 01_collaboration-rules 준수"**를 포함한다.
2. Claude가 남의 파일 수정이 필요하다고 판단하면 → 코드를 고치지 말고 **CR Issue 초안**을 출력하게 한다.
3. Claude가 `git push`를 실행하기 전 반드시 3.2 절차를 수행하게 한다. git 명령은 `git -C <repo>`로 대상 repo를 지정한다.
4. 대규모 리팩터링, 포맷터 일괄 적용(남의 파일까지 바뀜) 금지.
5. 의존성 추가가 필요하면 CR Issue로 백성준에게 요청.
