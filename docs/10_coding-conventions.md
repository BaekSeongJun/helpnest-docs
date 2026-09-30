# 10. 코딩 컨벤션 · CI

> 공용 설정 파일(`.prettierrc`, `eslint.config.*`, `.editorconfig`, `.gitattributes`, `.gitignore`, `tsconfig.json`)은 **백성준 소유**, CI 워크플로(`.github/workflows/`)는 **신수진 소유**(PRD Q12).
> **포맷터는 내가 수정한 파일에만** 적용한다. `prettier --write .` / IDE "프로젝트 전체 포맷" 금지 (남의 파일 diff 발생 → 규칙 2 위반).

---

## 1. 공통
| 항목 | 규칙 |
|---|---|
| 언어 | 코드·식별자 영어, 주석·커밋 메시지·UI 문구 한국어 |
| 소유자 주석 | 새 파일 첫 줄 `// @owner BSJ\|PMJ\|SSJ` |
| TODO | `// TODO(PMJ): 내용` — 이니셜 필수 |
| 비밀값 | 코드·커밋 금지, `.env.example`에 키 이름만 |
| API 변경 | 같은 날 `docs/04_api-spec.md` 자기 섹션 PR |
| 시간 | 서버·DB·화면 모두 `Asia/Seoul` 기준 표시, API는 ISO-8601(+09:00) |

---

## 2. Frontend (Next.js + TypeScript)

### 2.1 설정 (백성준, S0)
- `tsconfig.json`: `"strict": true`, 경로 별칭 `@/*` → `src/*`
- ESLint: Next.js 기본(core-web-vitals + typescript) — `no-explicit-any` 경고
- Prettier
```json
{ "semi": true, "singleQuote": true, "printWidth": 100, "trailingComma": "all",
  "plugins": ["prettier-plugin-tailwindcss"] }
```
- VS Code 권장: `editor.formatOnSave: true` (저장한 파일만 포맷됨)

### 2.2 네이밍·파일
| 대상 | 규칙 | 예 |
|---|---|---|
| 라우트 폴더 | kebab-case | `forgot-password/` |
| 컴포넌트 파일·이름 | PascalCase | `TicketTable.tsx` |
| 훅 | `useXxx` camelCase | `useTicketList.ts` |
| API 함수 | 동사 + 명사 | `getTickets`, `getTicket`, `createTicket`, `updateTicketStatus` |
| 타입 | PascalCase, 요청/응답 접미사 | `TicketResponse`, `TicketCreateRequest`, `TicketStatus` |
| 상수 | UPPER_SNAKE_CASE | `MAX_ATTACHMENT_COUNT` |
| enum 값 | 백엔드와 동일 문자열 유니온 | `type TicketStatus = 'RECEIVED' \| 'ASSIGNED' \| ...` |

### 2.3 구조 규칙
- 페이지(`page.tsx`)는 조립만, 로직은 도메인 컴포넌트·훅으로
- 브라우저 상호작용이 있는 컴포넌트만 `'use client'`
- 토큰을 `localStorage`·`sessionStorage`·일반 쿠키에 저장하지 않는다 (Access는 `lib/api/client.ts` 메모리, Refresh는 서버가 httpOnly 쿠키로 관리)
- REST 기본 주소는 상대 경로 `/api` (Next.js rewrites 프록시). 백엔드 주소를 코드에 하드코딩하지 않는다. 예외: 첨부 업로드는 `NEXT_PUBLIC_UPLOAD_BASE_URL`, WebSocket은 `NEXT_PUBLIC_WS_URL`
- API 호출은 반드시 `lib/api/{domain}.ts` 함수 경유 (컴포넌트에서 `fetch` 직접 호출 금지) → 공통 `client.ts`(백성준)가 토큰 첨부·재발급·에러 변환 처리
- 서버 데이터 조회 방식은 팀 전체 1가지로 통일 — **TanStack Query** (PRD Q19 확정)
- 폼: react-hook-form + zod, 스키마는 해당 폼 컴포넌트 옆 `schema.ts`
- 사용자 입력 본문 출력은 `PlainText` 컴포넌트만 사용 (`dangerouslySetInnerHTML` 금지)
- UI 규칙은 [08 디자인 시스템](08_design-system.md) D1~D7

---

## 3. Backend (Spring Boot 4 · Java 21 · Maven)

### 3.1 패키지 구조 (도메인별)
```
domain/ticket/
├─ controller/  TicketController, ConsoleTicketController
├─ service/     TicketService, TicketStateMachine
├─ repository/  TicketRepository
├─ entity/      Ticket, TicketStatus(enum)
├─ dto/         TicketCreateRequest, TicketResponse ...
├─ event/       TicketCreatedEvent ...
└─ port/        TicketClassificationPort, TicketQueryPort (+ 구현체 *Adapter)
```

### 3.2 네이밍
| 대상 | 규칙 | 예 |
|---|---|---|
| 클래스 | `XxxController` / `XxxService` / `XxxRepository` | `SurveyService` |
| DTO | Java `record`, `XxxCreateRequest`, `XxxUpdateRequest`, `XxxResponse` | `record TicketResponse(...)` |
| 메서드 | 조회 `get/find`, 생성 `create`, 변경 `change/update`, 삭제 `delete` | `changeStatus()` |
| 테이블·컬럼 | snake_case, 테이블명 단수 | `ticket_reply` |
| URL | `/api/{복수명사}`, kebab-case | `/api/sla-policies` |
| 에러 코드 | `{DOMAIN}_{설명}` UPPER_SNAKE | `TICKET_INVALID_TRANSITION` |

### 3.3 작성 규칙
- **Controller**: 검증(`@Valid`) + 서비스 호출 + `ApiResponse.ok(data)` 반환만. 비즈니스 로직 금지
- **Service**: 클래스에 `@Transactional(readOnly = true)`, 쓰기 메서드에 `@Transactional`. 상태 전이·권한 검증은 여기서
- **Entity**: `@Setter` 금지, `@NoArgsConstructor(access = PROTECTED)`, 의미 있는 변경 메서드(`ticket.assignTo(agent)`), enum은 `@Enumerated(STRING)`, 시간은 `OffsetDateTime`
- **예외**: `throw new BusinessException(TicketErrorCode.INVALID_TRANSITION)` — 도메인별 ErrorCode enum 사용
- **연동**: 다른 도메인의 Repository·Entity 클래스 import 금지 → 포트/이벤트만. 통계·목록용 **읽기 전용 Native Query/JPQL**은 자기 패키지에서 허용 (쓰기는 반드시 포트) ([02 §5](02_architecture.md))
- **로그**: `@Slf4j`, `System.out` 금지, 이메일·전화번호·토큰 로그 출력 금지
- Lombok: `@Getter`, `@RequiredArgsConstructor`, `@Builder`(생성자에) 허용
- 포맷: `.editorconfig`(4 spaces, LF) + 각 IDE 기본 포맷, **수정한 줄/파일만** (§3.5)

### 3.4 테스트
| 대상 | 방식 | 필수 |
|---|---|---|
| 상태 전이, 배정, SLA 계산, 요청 제한, AI 응답 파싱 | 단위 테스트 (JUnit 5 + AssertJ) | ✅ |
| 주요 API | `@SpringBootTest` + 테스트 DB | 권장 |
| 이름 | `@DisplayName("RECEIVED에서 RESOLVED로 변경하면 예외")` | |
| LLM | `LLM_PROVIDER=mock` 으로 실행 | ✅ |

---

### 3.5 IDE 환경 통일 (IntelliJ 1명 · Eclipse/STS 2명)
> 빌드는 IDE가 아니라 **Maven Wrapper(`./mvnw`)** 기준이다. IDE마다 다른 설정이 git에 섞이지 않게 아래를 지킨다.

**공용 설정 파일 (백성준, S0에 추가)**

`.gitignore` (추가분)
```
# IDE
.idea/
*.iml
.project
.classpath
.settings/
.factorypath
.vscode/
bin/
out/
# build
target/
```

`.gitattributes`
```
* text=auto eol=lf
*.cmd text eol=crlf
*.bat text eol=crlf
mvnw text eol=lf
*.jar binary
*.png binary
```

`pom.xml`
```xml
<properties>
  <java.version>21</java.version>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
</properties>
```

`.editorconfig`
```
root = true
[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 4
[*.{yml,yaml,json,md}]
indent_size = 2
```

**각자 1회 설정**
| 항목 | IntelliJ (백성준) | Eclipse / STS (박민재·신수진) |
|---|---|---|
| JDK | Project Structure → SDK **21** | Preferences → Java → Installed JREs에 **21** 추가 → 프로젝트 JRE 21 |
| 버전 | 최신 버전 | **최신 Spring Tools(STS)** — Spring Boot 4 인식 필요 |
| 인코딩 | Settings → Editor → File Encodings 모두 **UTF-8** | Preferences → General → Workspace → Text file encoding **UTF-8**, New text file line delimiter **Unix** |
| Lombok | Settings → Build → Compiler → Annotation Processors → **Enable** | `lombok.jar` 다운로드 → `java -jar lombok.jar` → Eclipse/STS 설치 경로 지정 → 재시작 |
| import | Code Style → Java → Imports: *Class count to use import with '\*'* **99**, *Names count to use static import with '\*'* **99** | 기본값 유지 (와일드카드 안 씀), 저장 시 Organize Imports는 **편집한 파일만** |
| 포맷 | 저장 시 자동 포맷을 쓰면 "변경된 줄만" 옵션 사용 | Save Actions → Format source code → **Format edited lines** 선택 |
| EditorConfig | 기본 지원 | Marketplace에서 **EditorConfig** 플러그인 설치 |
| Maven | Maven Wrapper 사용 설정 | Maven 프로젝트로 Import (`Existing Maven Projects`) |
| 실행 설정 | Run Configuration에 `SPRING_PROFILES_ACTIVE=local` + `.env.example` 항목 | Run As → Spring Boot App → Environment 탭에 동일 항목 |

**규칙**
- IDE Run 설정·워크스페이스 파일은 커밋하지 않는다.
- IDE에서 실행이 되더라도 **push 전에 `./mvnw verify` 1회** (Eclipse는 자체 컴파일러를 써서 결과가 다를 수 있음, CI와 같은 명령).
  - 터미널에서는 **`SPRING_PROFILES_ACTIVE=local ./mvnw verify`** 로 실행한다 (PowerShell: `$env:SPRING_PROFILES_ACTIVE='local'; ./mvnw verify`). 프로필이 없으면 `DB_URL` 등이 비어 테스트가 기동 실패한다 — CI도 같은 값을 넣는다(§4).
  - `LLM_PROVIDER`는 넣지 않아도 된다(기본 `mock` → Spring AI 자동 설정 비활성).
- Eclipse의 `bin/`, IntelliJ의 `out/`이 `git status`에 보이면 `.gitignore` 누락 → 백성준에게 CR.
- Windows 사용자는 `git config --global core.autocrlf input` 권장 (`.gitattributes`와 함께 줄바꿈 통일).

---

## 4. CI (GitHub Actions, 신수진)

### 4.1 원칙
- 실행 시점: **dev·main을 대상으로 한 PR에서만** (push마다 실행하지 않음 → 무료 한도 절약)
- 같은 PR에 새 커밋이 오면 이전 실행 취소 (`concurrency`)
- 무료 한도: 우리 repo는 **Public → 표준 러너 무료·무제한** (참고: Private이면 월 2,000분)
- 워크플로가 한 번 성공한 뒤 브랜치 보호 규칙에 **Required status checks**로 등록 (repo 설정 권한은 소유자 백성준에게만 있으므로 백성준이 등록) (실패 시 머지 불가)

### 4.2 `helpnest-front/.github/workflows/ci.yml`
```yaml
name: front-ci
on:
  pull_request:
    branches: [dev, main]
concurrency:
  group: front-ci-${{ github.head_ref }}
  cancel-in-progress: true
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit
      - run: npm run build
        env:
          BACKEND_ORIGIN: http://localhost:8080
          NEXT_PUBLIC_UPLOAD_BASE_URL: http://localhost:8080
          NEXT_PUBLIC_WS_URL: ws://localhost:8080/ws
```

### 4.3 `helpnest-back/.github/workflows/ci.yml`
```yaml
name: back-ci
on:
  pull_request:
    branches: [dev, main]
concurrency:
  group: back-ci-${{ github.head_ref }}
  cancel-in-progress: true
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: helpnest
          POSTGRES_USER: helpnest
          POSTGRES_PASSWORD: helpnest
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U helpnest"
          --health-interval 5s --health-timeout 5s --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
          cache: maven
      - run: chmod +x mvnw   # Windows에서 커밋하면 실행 권한이 빠질 수 있음
      - run: ./mvnw -B verify
        env:
          SPRING_PROFILES_ACTIVE: local
          DB_URL: jdbc:postgresql://localhost:5432/helpnest
          DB_USERNAME: helpnest
          DB_PASSWORD: helpnest
          JWT_SECRET: ci-only-secret-key-ci-only-secret-key-0000
          LLM_PROVIDER: mock
```
> 액션 버전(`@v4`)과 Node 버전은 S0에 최신 안정 버전으로 확인 후 고정.
> docs repo는 CI 없음.

### 4.4 CI 실패 시
1. Actions 로그에서 실패 단계 확인
2. **내 파일 때문이면** 내가 수정 후 push (pull 확인 절차 준수)
3. **남의 파일/dev 자체가 깨졌으면** 해당 소유자에게 알리고 CR Issue — 남의 코드를 고쳐서 통과시키지 않는다
