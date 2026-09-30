<!-- @owner BSJ — 원본은 helpnest-docs/claude/CLAUDE.md. 메인 디렉토리의 사본은 직접 수정하지 말고 원본 수정 후 다시 복사 -->
# HelpNest 작업 규칙 (최우선)

Claude Code는 메인 디렉토리 `helpnest/`에서 실행한다. 이 폴더는 git 저장소가 아니고, 아래 3개 repo가 나란히 있다.

| 폴더 | 내용 |
|---|---|
| `helpnest-docs/` | 문서 (PRD.md, docs/01~10) + Claude 공용 설정 원본(`claude/`) |
| `helpnest-front/` | Next.js |
| `helpnest-back/` | Spring Boot |

- 담당자는 `CLAUDE.local.md`(각자 작성, 공유 안 함)의 "나는 X 담당이다"를 따른다. 없으면 먼저 물어본다.
- 작업 전 `helpnest-docs/docs/01_collaboration-rules.md`를 기준으로 한다.
- git 명령은 항상 `git -C <repo>`로 repo를 지정한다. 한 커밋에 여러 repo를 섞지 않는다. 커밋은 `/git-commit` 스킬 사용.
- push 전 반드시: `git fetch origin` → behind 확인 → behind면 `git pull` → 충돌 해결·빌드 → push. `--force`·`--no-verify` 금지.
- 3개 repo가 모두 보이므로 **소유권(R2)을 먼저 확인**한다. 내 소유가 아닌 파일(01 §4 소유권 표, 각 repo `.github/CODEOWNERS`, 파일 첫 줄 `@owner`)은 수정하지 않고 "변경 요청 Issue 초안"을 보여준다.
- 브랜치: `feature/{이니셜}-{기능}`. dev/main 에 직접 커밋/푸시 금지. 머지는 PR로 feature → dev → main.
- DB는 로컬 PostgreSQL(`helpnest-back/docker-compose.yml`)만 사용. AWS 자원은 신수진이 배포 단계에서 연결한다.
- 다른 도메인과의 연동은 `docs/02_architecture.md` §5 이벤트/포트 계약만 사용한다.
- UI는 `docs/08_design-system.md` 규칙(D1~D7): 의미 토큰 클래스만, `components/ui`·`components/common` 재사용.
  `components/ui`·`globals.css`·`helpnest-docs/claude/`는 수정하지 않는다(백성준 소유). 새 shadcn 컴포넌트가 필요하면 CR 초안.
- 의존성(`package.json`, `pom.xml`) 추가는 CR Issue로 백성준에게 요청한다.
- 포맷터는 내가 수정한 파일에만 적용한다.
- 프론트는 Next.js 16 — 코드 작성 전 `helpnest-front/AGENTS.md` 안내대로 `node_modules/next/dist/docs/`를 확인한다.
