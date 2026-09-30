# 07. Shrimp Task Manager 사용 가이드

> Shrimp Task Manager는 Claude Code용 MCP 서버로, 큰 요구사항을 **계획 → 분해 → 실행 → 검증** 단위의 태스크로 관리한다.
> 우리 팀은 **각자 자기 PC에서 자기 태스크만** 관리하고, 팀 전체 진행 현황은 [06_roadmap.md](06_roadmap.md) 체크박스로 공유한다.

## 1. 설치 (각자 1회)

설치·설정 방식은 버전에 따라 바뀔 수 있으니 [공식 저장소](https://github.com/cjo4m06/mcp-shrimp-task-manager) README를 기준으로 확인한다.

```bash
git clone https://github.com/cjo4m06/mcp-shrimp-task-manager.git ~/tools/mcp-shrimp-task-manager
cd ~/tools/mcp-shrimp-task-manager
npm install
npm run build
```

### 1.1 MCP 설정 (메인 디렉토리)
Claude Code를 메인 디렉토리 `helpnest/`에서 실행하므로 설정도 **메인 디렉토리에 1개**만 둔다. 메인 디렉토리는 git 저장소가 아니라 커밋될 일이 없다.

`.mcp.json` (또는 `claude mcp add`로 등록)
```json
{
  "mcpServers": {
    "shrimp-task-manager": {
      "command": "node",
      "args": ["/절대경로/tools/mcp-shrimp-task-manager/dist/index.js"],
      "env": {
        "DATA_DIR": "/절대경로/helpnest/.shrimp",
        "TEMPLATES_USE": "ko",
        "ENABLE_GUI": "false"
      }
    }
  }
}
```
- `DATA_DIR`은 **절대 경로**. front/back 태스크가 한 목록에 섞이므로 태스크 이름 앞에 `[front]`/`[back]`을 붙인다.
- `TEMPLATES_USE`: 한국어 템플릿이 없으면 `en` 사용.
- 실수로 repo 안에 만들어도 올라가지 않도록 front/back `.gitignore`에 포함되어 있다(S0 반영 완료):
  ```
  .shrimp/
  .mcp.json
  shrimp-rules.md   # "나는 X 담당"처럼 사람마다 내용이 달라서 공유하지 않음
  ```

## 2. 작업 순서

| 단계 | 명령(Claude에게 말하기) | 비고 |
|---|---|---|
| ① 규칙 초기화 | `init project rules` | 메인 디렉토리에서 1회. 생성된 `shrimp-rules.md` 맨 위에 §3 규칙 블록 붙여넣기 |
| ② 계획 | `plan task: <로드맵 항목>` | 로드맵의 **내 섹션 항목 1~2개** 단위로 |
| ③ 분석·분해 | (자동) analyze → reflect → split | 태스크마다 "완료 조건"에 pull 확인 포함되었는지 확인 |
| ④ 목록 확인 | `list tasks` | |
| ⑤ 실행 | `execute task <id>` | 한 번에 하나. 연속 모드는 공용 파일 오염 위험이 있어 **지양** |
| ⑥ 검증 | `verify task <id>` | 빌드/테스트 결과 첨부 |
| ⑦ 커밋·푸시 | 커밋 → **pull 확인** → push | [01 §3.2](01_collaboration-rules.md#32-push-직전-매번) |
| ⑧ 로드맵 체크 | docs repo에서 내 섹션 체크 | feature 브랜치 → dev PR |

## 3. `shrimp-rules.md` 최상단에 넣을 규칙 블록

```md
## 0. 최우선 규칙 (모든 태스크에 우선)
1. 나는 HelpNest [백성준|박민재|신수진] 담당이다. 문서는 helpnest-docs/ 에 있고, 아래 docs/ 경로는 그 기준이다. git 명령은 git -C <repo> 로 실행한다.
2. 모든 태스크의 마지막 단계는 다음 순서다:
   git add → git commit → git fetch origin → (behind 확인) → git pull origin <현재브랜치> → git pull origin dev
   → 충돌 해결 → 빌드/테스트 → git push origin <현재브랜치>
3. 내가 만들지 않은 파일은 수정하지 않는다. (소유권: docs/01_collaboration-rules.md §4, .github/CODEOWNERS)
   수정이 필요하면 코드를 고치지 말고 "CR Issue 초안(제목/이유/원하는 변경)"을 출력하고 해당 태스크를 보류한다.
4. 작업 브랜치는 feature/{이니셜}-{기능}. dev/main 체크아웃 상태에서 커밋하지 않는다.
5. DB는 로컬 PostgreSQL(docker compose)만 사용한다. AWS 자원 연결 코드는 신수진만 작성한다.
6. 다른 도메인 연동은 docs/02_architecture.md §5의 이벤트/포트만 사용한다.
7. 새 파일 상단에 소유자 주석(// @owner X)을 단다.
8. package.json / pom.xml 의존성 추가가 필요하면 CR Issue 초안으로 대체한다 (백성준 소유).
9. UI는 docs/08_design-system.md D1~D7 준수: 의미 토큰 클래스만, components/ui·common 재사용, 로딩/빈/에러 상태 포함.
   components/ui, globals.css, helpnest-docs/claude/ 는 수정 금지. 태스크 완료 조건에 "08 §14 셀프 체크 통과"를 넣는다.
10. 코드 스타일은 docs/10_coding-conventions.md 를 따르고, 포맷은 수정한 파일에만 적용한다.
```

## 4. 프롬프트 예시

**백성준 — S1 문의 접수**
```
plan task: HelpNest 백성준 담당. docs/06_roadmap.md 백성준-S1 "문의 접수 페이지(회원/비회원), 첨부 UI, 접수 완료 화면".
참고: PRD FR-INQ-01~03, docs/04_api-spec.md §3, §7(POST /api/tickets 는 박민재 소유라 호출만),
화면은 src/app/(main)/inquiry/new 아래에만 생성. 공용 레이아웃은 기존 것을 사용.
각 태스크 완료 조건에 "npm run build 성공 + push 전 pull 확인"을 넣어줘.
```

**박민재 — S1 상태 전이**
```
plan task: HelpNest 박민재 담당. TicketStateMachine 과 상태 변경 API (PATCH /api/console/tickets/{id}/status).
PRD 5장 전이표를 서비스 계층에서 검증하고, TICKET_HISTORY 기록과 TicketStatusChangedEvent 발행.
전이표 전체 경우의 단위 테스트 포함. domain/ticket 패키지 안에서만 작업.
```

**신수진 — S1 AI 분류**
```
plan task: HelpNest 신수진 담당. docs/05_ai-automation.md §3 AI-1 분류.
TicketCreatedEvent(박민재 소유, 수정 금지)를 AFTER_COMMIT + @Async 로 구독하고
TicketClassificationPort(박민재 소유)를 호출. LLM 실패 시 applyClassificationFailed 호출.
MockLlmClient로 테스트 가능하게. domain/ai, infra/llm 에서만 작업.
```

## 5. 자주 하는 실수
| 실수 | 예방 |
|---|---|
| Claude가 공용 `layout.tsx`, `SecurityConfig`를 "고쳐주는" 경우 | 규칙 블록 3번, PR에서 diff 확인 후 되돌리기 |
| 태스크 여러 개를 한 번에 실행해 커밋 단위가 커짐 | execute는 하나씩, 태스크마다 커밋 |
| pull 없이 push 하다 거절됨 → `--force` 시도 | force push 금지, pre-push 훅 활성화 |
| 포맷터가 남의 파일까지 수정 | 저장 시 포맷은 "변경한 파일만" 설정, 일괄 포맷 명령 금지 |
| 로컬 DB 스키마가 서로 달라짐 | dev pull 후 앱 재시작으로 Flyway 적용, 필요 시 `docker compose down -v` 후 재생성 |
