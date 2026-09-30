---
name: git-commit
description: HelpNest 커밋 규칙(docs/01 §5.1)에 맞춰 변경사항을 검사하고 커밋한다. 브랜치·파일 소유권·비밀값을 확인한 뒤 `<type>(<scope>): <요약> [이니셜]` 형식으로 커밋. "커밋해줘", "commit", "/git-commit" 요청 시 사용. push는 사용자가 요청할 때만 R1 절차로 진행.
---

# HelpNest git-commit

> @owner BSJ — 수정이 필요하면 백성준에게 CR Issue

## 0. 대상 repo 찾기

Claude는 메인 디렉토리 `helpnest/`(git 저장소 아님)에서 실행된다. 커밋은 **repo마다 따로** 한다.

```bash
for r in helpnest-docs helpnest-front helpnest-back; do
  echo "== $r ($(git -C $r branch --show-current))"; git -C $r status --short
done
```

- 변경이 있는 repo마다 아래 1~4를 **각각** 수행한다. 이후 모든 git 명령은 `git -C <repo>` 로 실행한다 (`cd` 로 이동하지 않음).
- 한 커밋에 여러 repo를 섞을 수 없다. front+back 을 함께 바꾼 작업은 repo별 커밋 2개가 된다.

## 1. 사전 검사 (하나라도 걸리면 커밋하지 말고 사용자에게 알린다)

```bash
git -C <repo> branch --show-current
git -C <repo> diff --staged --stat   # 스테이징이 비어 있으면 git -C <repo> diff --stat
```

| 검사 | 기준 | 걸리면 |
|---|---|---|
| 브랜치 | `main` / `dev` / `master` 에서 커밋 금지 (R3) | `feature/{이니셜}-{기능}` 브랜치 생성 제안 (최신 dev에서) |
| 이니셜 | 브랜치명 `feature/BSJ-...` 에서 추출 (`BSJ`\|`PMJ`\|`SSJ`) | 추출 불가면 사용자에게 질문 |
| 소유권 (R2) | 변경 파일이 내 소유인지: 파일 첫 줄 `@owner`, `.github/CODEOWNERS`, docs/01 §4.2 공용 파일 목록(백성준) | 남의 파일이 섞였으면 **커밋에서 제외**하고 CR Issue 초안 출력 |
| 비밀값 | `.env`, `*secret*`, `application-local-secret.yml`, API 키·JWT 키·AWS 키 문자열 | 스테이징 해제 후 경고. 절대 커밋 금지 (Public repo) |
| 로컬 전용 | `.shrimp/`, `.mcp.json`, `shrimp-rules.md`, IDE 파일(`.idea/`, `.settings/`, `bin/`, `out/`) | 스테이징 제외, `.gitignore` 누락이면 백성준에게 CR |

## 2. 커밋 단위 나누기

- 한 커밋 = 한 가지 의미. 성격이 다른 변경(기능 + 설정 + 문서)은 파일별로 `git -C <repo> add <path>` 해서 여러 커밋으로 나눈다 (path는 repo 기준 상대경로).
- `git add .` / `git add -A` 는 쓰지 않는다 (남의 파일·비밀값 혼입 방지).

## 3. 메시지 형식 (docs/01 §5.1)

```
<type>(<scope>): <요약> [<이니셜>]

<본문: 왜 바꿨는지, 필요할 때만 1~3줄>
```

- **type**: `feat` | `fix` | `refactor` | `test` | `docs` | `style` | `chore`
- **scope**: `auth`, `member`, `inquiry`, `attachment`, `faq`, `template`, `survey`, `customer`, `ticket`, `assign`, `sla`, `noti`, `chat`, `ai`, `dashboard`, `report`, `infra`, `layout`, `ui`, `config`, `ci`, `db` …
- **요약**: 한국어, 50자 이내, 마침표 없음, "~추가 / ~수정 / ~제거" 형태
- 예) `feat(ticket): 상태 전이 서비스 검증 추가 [PMJ]`, `chore(config): Prettier·EditorConfig 공용 설정 추가 [BSJ]`

커밋은 HEREDOC 으로:

```bash
git -C <repo> commit -F - <<'EOF'
feat(auth): JWT 로그인·재발급 API 추가 [BSJ]
EOF
```

## 4. 커밋 후

- `git -C <repo> log --oneline -3` 으로 결과를 보여준다.
- **push는 사용자가 요청할 때만.** 요청 시 repo마다 R1 절차(docs/01 §3.2)를 반드시 수행:

```bash
b=$(git -C <repo> branch --show-current)
git -C <repo> fetch origin
git -C <repo> log --oneline HEAD..origin/$b    # 원격 내 브랜치에 새 커밋?
git -C <repo> log --oneline HEAD..origin/dev   # dev에 새 커밋?
# 출력이 있으면 → git -C <repo> pull origin $b / git -C <repo> pull origin dev (merge 방식)
# → 충돌 해결 → 빌드 확인 (front: npm --prefix helpnest-front run build / back: helpnest-back/mvnw -f helpnest-back verify)
git -C <repo> push origin $b
```

- `--force`, `--no-verify` 금지. pre-push 훅이 막으면 우회하지 말고 원인을 해결한다.
