# 02. 아키텍처

## 1. 기술 스택 (백성준)

| 영역 | 기술 | 비고 |
|---|---|---|
| Front | Next.js 16.3.x 최신 안정 (App Router, TypeScript) | Sprint 0에 백성준이 `create-next-app@latest`로 생성 후 버전 고정 |
| UI | Tailwind CSS + shadcn/ui + lucide 아이콘 | 공용 컴포넌트·디자인 토큰은 백성준 ([08](08_design-system.md)) |
| 폼·조회 | react-hook-form + zod / TanStack Query | PRD Q18·Q19 확정 |
| 상태/통신 | fetch 래퍼(`lib/api/client.ts`, 기본 경로 상대 `/api`) + JWT 자동 재발급 (Access 메모리 / Refresh httpOnly 쿠키) | 백성준 |
| API 프록시 | `next.config.ts`의 `rewrites`: `/api/:path*` → `${BACKEND_ORIGIN}/api/:path*` (§2.1) | 백성준 |
| 실시간 | `@stomp/stompjs` | 박민재 |
| 차트 | 라이브러리 없음 — CSS 막대(`components/dashboard/DistributionBars`) | 신수진 |
| Back | Spring Boot **4.1.1** / Java 21 / **Maven** | 백성준 초기 세팅 |
| 보안 | Spring Security + JWT(Access 30분 / Refresh 14일) | 백성준 |
| ORM | Spring Data JPA (+ 통계는 JPQL/Native Query) | 각 도메인 소유자 |
| 마이그레이션 | Flyway (PRD Q7 확정) | 각자 자기 테이블 |
| 실시간 | Spring WebSocket + STOMP (SimpleBroker) | 박민재 |
| AI | Spring AI **2.0.1** (`ChatClient`) — 제공자 설정으로 교체 | 신수진 |
| 스케줄 | `@Scheduled` (SLA 1분, 자동 종료 10분) | 박민재 |
| DB | PostgreSQL (로컬 docker / 배포 RDS) | |
| AWS | S3, SES, EC2, RDS | 신수진 (Sprint 4) |

> **확정 버전 (2026-09-29 기준)**
> | 항목 | 버전 | 비고 |
> |---|---|---|
> | Next.js | **최신 안정 16.3.x** (작성 시점 16.3.6) | S0에 `create-next-app@latest`로 설치 → 설치된 정확한 버전을 `package.json`에 고정(`^` 제거) |
> | Node.js | 20.9 이상 (팀·CI는 **22 LTS** 사용) | Next.js 16 최소 요구 20.9 |
> | Spring Boot | **4.1.1** | |
> | Spring AI | **2.0.1** (최신 GA) | Spring Boot 4.0/4.1 호환 라인. `spring-ai-bom` 2.0.1로 버전 관리 |
> | Java | 21 | |
> | PostgreSQL | 16 | 로컬 docker·CI·RDS 동일 메이저 |

---

## 2. 시스템 구성

```mermaid
flowchart LR
  subgraph Client
    CU[고객 / 비회원]
    AG[상담원 · 팀장 · 관리자]
  end
  subgraph Front[helpnest-front · Next.js]
    FP[페이지]
    WSClient[STOMP Client]
  end
  subgraph Back[helpnest-back · Spring Boot]
    API[REST Controller]
    SVC[Service · 상태 전이 검증]
    EVT[(Spring Event)]
    WS[STOMP Broker]
    SCH[Scheduler SLA/자동종료]
    AI[AI Service · Spring AI]
  end
  DB[(PostgreSQL)]
  LLM[[LLM API]]
  S3[(S3 / 로컬 디스크)]
  SES[[SES / 로그 메일]]

  CU --> FP
  AG --> FP
  FP -- REST+JWT --> API
  WSClient <-- STOMP --> WS
  API --> SVC --> DB
  SVC --> EVT --> AI --> LLM
  EVT --> WS
  SCH --> SVC
  SVC --> S3
  EVT --> SES
```

### 2.1 요청 경로 (배포 환경)

> 프론트(`*.amplifyapp.com`)와 백엔드(`*.cloudfront.net`)는 서로 다른 사이트다. Refresh 쿠키가 서드파티 쿠키로 차단되지 않도록 REST는 프론트 도메인을 거쳐 보낸다 (PRD Q21).

| 요청 | 경로 | 인증 | 이유 |
|---|---|---|---|
| 일반 REST API | 브라우저 → Amplify(Next.js) `/api/*` → **rewrites** → CloudFront → EC2 | Access 토큰(헤더) + Refresh 쿠키(`/api/auth/*`) | 쿠키가 프론트 도메인 1st-party → Safari 등에서도 유지 |
| WebSocket(STOMP) | 브라우저 → CloudFront `wss://…/ws` **직접** → EC2 | STOMP `CONNECT` 헤더의 Access 토큰 | 쿠키 불필요. 백엔드는 `/ws`에 `FRONT_ORIGIN`만 허용 |
| 첨부 업로드 | 브라우저 → CloudFront `/api/attachments` **직접** | Access 토큰(회원) / 없음(비회원, 요청 제한 적용) | Amplify SSR의 크기 제한 회피(응답 5.72MB 제한은 문서 확인, 요청 제한은 미명시) → 백엔드 CORS는 이 경로만 `FRONT_ORIGIN` 허용 |
| 첨부 다운로드 | 브라우저 → S3 presigned URL **직접** | URL 서명 | 대용량 응답이 프록시를 거치지 않게 |
| 로컬 개발 | `localhost:3000/api/*` → rewrites → `localhost:8080` | 동일 | 배포와 같은 구조로 개발 |

- 요청 제한(RateLimitFilter)은 **실제 클라이언트 IP** 기준으로 센다. CloudFront 는 받은 `X-Forwarded-For` 뒤에 덧붙이므로 왼쪽 값은 위조할 수 있다 → `server.forward-headers-strategy: none` 으로 원문을 받아 **오른쪽에서** 고른다(back #105 실측). CloudFront 직접 호출은 오른쪽 1번째, Amplify 경유(`proxy.ts` 가 붙인 `X-Helpnest-Proxy` 비밀 헤더가 `PROXY_SECRET` 과 같을 때만)는 오른쪽 3번째(`클라이언트, Amplify CloudFront, Amplify 컴퓨트`).
- CSV 다운로드는 수 MB 미만이라 프록시 경유로 충분 (5.72MB를 넘을 정도로 커지면 직접 호출로 변경).

---

## 3. Frontend 구조 (helpnest-front)

```
src/
├─ app/
│  ├─ layout.tsx                       (백성준) 루트 레이아웃, Provider
│  ├─ (auth)/login, signup, forgot-password, reset-password  (백성준)
│  ├─ (main)/layout.tsx                (백성준) ★ 공용 Header + Sidebar + Footer
│  ├─ (main)/page.tsx                  (백성준) 홈(고객: FAQ/문의 진입, 직원: 콘솔로 리다이렉트)
│  ├─ (main)/inquiry/new, complete     (백성준) 문의 접수·접수 완료
│  ├─ (main)/inquiry/lookup, lookup/reset (백성준) 비회원 조회·조회 비밀번호 재설정
│  ├─ (main)/my/profile                (백성준) 내 정보·비밀번호 변경
│  ├─ (main)/my/inquiries, [id]        (백성준) 내 문의 (상세 타임라인은 박민재 컴포넌트 사용)
│  ├─ (main)/faq                       (백성준)
│  ├─ (main)/survey/[token]            (백성준)
│  ├─ (main)/chat                      (박민재) 고객 채팅
│  ├─ (main)/console/tickets, [id]     (박민재) 상담 콘솔
│  ├─ (main)/console/chat              (박민재) 상담원 채팅
│  ├─ (main)/console/dashboard         (신수진)
│  ├─ (main)/console/reports           (신수진)
│  ├─ (main)/console/customers/[key]   (백성준) 고객 이력
│  ├─ (main)/console/surveys           (백성준) 설문 결과
│  ├─ (main)/admin/members, faq, templates (백성준)
│  ├─ (main)/dev/ui                    (백성준) 공용 컴포넌트 사용 예시 (배포 전 접근 차단)
│  └─ (main)/admin/sla                 (박민재)
├─ components/
│  ├─ layout/  Header, Sidebar, Footer, NotificationBellSlot   (백성준)
│  ├─ ui/      shadcn/ui 원본 컴포넌트 (button, input, dialog, table, badge, toast ...)  (백성준)
│  ├─ common/  PageHeader, DataTable, EmptyState, ErrorState, LoadingSkeleton, ConfirmDialog, StatusBadge, PriorityBadge, SentimentBadge, PlainText  (백성준)
│  ├─ inquiry/, faq/, survey/, template/(TemplatePicker), customer/(CustomerHistoryPanel)  (백성준)
│  ├─ ticket/  TicketTable, TicketTimeline, ReplyEditor (박민재)
│  ├─ notification/ NotificationBell    (박민재)
│  ├─ chat/    ChatWindow               (박민재)
│  ├─ ai/      AiAnalysisPanel, AiDraftButton (신수진)
│  └─ dashboard/ KpiCard, DistributionBars(CSS 막대), AgentTable, CsvButton (신수진)
├─ lib/
│  ├─ api/client.ts                    (백성준) 공통 fetch + 토큰 재발급
│  ├─ api/{auth,faq,template,survey,attachment,customer}.ts (백성준)
│  ├─ api/{ticket,notification,chat}.ts (박민재)
│  ├─ api/{ai,dashboard,report}.ts     (신수진)
│  ├─ auth/                            (백성준) useAuth, AuthGuard(역할 가드) — Access 토큰 메모리 보관·새로고침 시 refresh 는 api/client.ts
│  └─ ws/stompClient.ts                (박민재)
├─ config/menu.ts                      (백성준) 역할별 사이드바 메뉴 정의
├─ config/badge.ts                     (백성준) 상태·우선순위·감정 → 배지 variant 매핑
├─ lib/format.ts                       (백성준) 날짜·시간·숫자 포맷 함수
├─ types/                              도메인별 소유 (auth.ts 백성준 / ticket.ts 박민재 / ai.ts 신수진)
└─ proxy.ts                            (백성준) 라우트 1차 확인 (Refresh 쿠키 유무, Next 16 에서 middleware.ts → proxy.ts) — 최종 권한 판단은 lib/auth의 AuthGuard
```

### 3.1 공용 레이아웃 규칙
- **Header**(백성준): 로고, 사용자 메뉴, 알림 벨 **슬롯**. 알림 벨 컴포넌트 자체는 박민재가 `components/notification/NotificationBell.tsx`로 만들고, 백성준이 Header에 1회 배치한다.
- **Sidebar**(백성준): `config/menu.ts`의 역할별 메뉴를 렌더링. 메뉴 추가는 백성준에게 CR.
- **Footer**(백성준): 고객 화면 하단 정보.
- 각 담당자는 `(main)/` 아래 **자기 페이지 폴더만** 만든다 → 레이아웃은 자동 적용.
- 다른 담당자 페이지에 들어가는 컴포넌트는 "컴포넌트 제공자가 만들고, 페이지 소유자가 import" 방식.
  - 티켓 상세(박민재) ← `AiAnalysisPanel`, `AiDraftButton`(신수진), `TemplatePicker`, `CustomerHistoryPanel`(백성준)

### 3.2 역할별 사이드바 메뉴 (초안)
| 역할 | 메뉴 |
|---|---|
| GUEST/CUSTOMER | 홈, FAQ, 문의하기, 내 문의(회원)/문의 조회(비회원), 1:1 채팅(회원), 내 정보(회원) |
| AGENT | 티켓함, 채팅 상담, 내 처리현황, 설문 결과(본인) |
| LEAD | + 전체 티켓, 대시보드, 월간 리포트, 설문 결과(전체), FAQ 관리, 템플릿 관리 |
| ADMIN | + 계정 관리, SLA 정책 |

---

## 4. Backend 구조 (helpnest-back)

```
com.helpnest
├─ HelpNestApplication.java                 (백성준)
├─ global/                                  (백성준) ★ 공용
│  ├─ config/  SecurityConfig, CorsConfig, JpaAuditingConfig, AsyncConfig, SchedulingConfig
│  ├─ security/ JwtProperties, JwtProvider(Access·Guest 토큰 발급, `guestTicketId(jwt)` 로 Guest 판정), RateLimitFilter (JWT 검증은 OAuth2 Resource Server 가 처리 — 별도 JwtAuthFilter·UserDetails 없음, 컨트롤러는 `@AuthenticationPrincipal Jwt` → `JwtProvider.memberId(jwt)`)
│  ├─ common/  ApiResponse<T>, PageResponse<T>, BaseTimeEntity
│  ├─ error/   ErrorCode(interface), CommonErrorCode, BusinessException, GlobalExceptionHandler
│  └─ websocket/ WebSocketConfig, StompAuthInterceptor      (박민재) ← global 안이지만 박민재 소유
├─ domain/
│  ├─ auth/, member/, attachment/, faq/, template/, survey/, customer/   (백성준)
│  ├─ ticket/, assignment/, sla/, notification/, chat/                  (박민재)
│  └─ ai/, dashboard/, report/                                          (신수진)
└─ infra/                                   (신수진)
   ├─ storage/ FileStorage(interface), LocalFileStorage(!prod), S3FileStorage(prod)
   ├─ mail/    MailSender(interface) → DefaultMailSender(MailTemplates·MAIL_LOG) → MailTransport: LogMailTransport(!prod), SesMailTransport(prod)
   ├─ llm/     LlmClient(interface), MockLlmClient(mock), SpringAiLlmClient
   └─ aws/     AwsConfig(prod) — S3Client·S3Presigner·SesV2Client, 자격 증명은 SDK 기본 체인(EC2 IAM Role)
```
- 도메인 내부 구조: `controller / service / repository / entity / dto / event / port`
- **인증 예외 경로(permitAll)** 는 `SecurityConfig`(백성준)에서 [04 API 명세](04_api-spec.md)의 권한 열이 "공개"인 엔드포인트 + `/ws/**`(STOMP 인증은 `StompAuthInterceptor`가 담당) + `/actuator/health` 기준으로 관리한다. 새 공개 API가 생기면 API 명세 PR과 함께 백성준에게 CR.
- 프로필: `local`(docker DB, 로컬 저장, 로그 메일) / `prod`(RDS, S3, SES)

---

## 5. 도메인 간 연동 계약 (이벤트 · 포트)

> 남의 코드를 수정하지 않고 연동하기 위한 **유일한 통로**. 인터페이스는 **제공자(구현자) 패키지**에 두고, Sprint 0에 **스텁 구현**을 먼저 머지한다.
>
> **읽기 전용 예외:** 통계·목록 조회(대시보드·리포트(신수진), 고객 이력·설문 결과(백성준), SLA·배정(박민재)의 MEMBER 조회 등)는 **자기 패키지 안의 읽기 전용 Native Query/JPQL**로 다른 테이블을 JOIN해도 된다. 단 **쓰기(INSERT/UPDATE/DELETE)는 반드시 포트**, 남의 Entity·Repository 클래스는 import하지 않으며, 참조하는 컬럼 목록을 PR 본문에 적어 소유자가 스키마 변경 시 알 수 있게 한다.

### 5.1 이벤트 (발행자 소유 클래스)
| 이벤트 | 발행(소유) | 구독 | 처리 |
|---|---|---|---|
| `TicketCreatedEvent(ticketId)` | 박민재 | 신수진 `AiClassifyListener` | `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` 로 LLM 분류 |
| `TicketAssignedEvent(ticketId, agentId)` | 박민재 | 박민재 `NotificationListener` | 상담원 알림 |
| `TicketStatusChangedEvent(ticketId, from, to, actorId)` | 박민재 | 백성준 `SurveyListener`, 박민재 알림 | to=RESOLVED → 설문 생성(재해결이면 재발급) → 메일 발송 / RESOLVED→IN_PROGRESS(재문의) → 미제출 설문 만료 |
| `ReplyCreatedEvent(ticketId, replyId, writerType, isInternal)` | 박민재 | 박민재 알림 | 고객 답글 → 상담원 알림 / 상담원 공개 답변 → 고객 웹 알림(회원) + `MailSender.sendAgentReplyMail`(신수진) |
| `SurveySubmittedEvent(ticketId, rating)` | 백성준 | 박민재 `TicketCloseListener` | 티켓 CLOSED |

### 5.2 포트 (인터페이스 소유 = 구현자)
| 포트 | 소유/구현 | 호출자 | 시그니처 |
|---|---|---|---|
| `TicketClassificationPort` | 박민재 | 신수진 | `applyClassification(ticketId, category, priority, sentiment)` / `applyClassificationFailed(ticketId)` → 우선순위·SLA 재계산 후 자동 배정 |
| `TicketQueryPort` | 박민재 | 백성준, 신수진 | `getTicketSummary(ticketId)`, `findResolvedReplies(category, limit)` (AI 초안 컨텍스트) |
| `AttachmentPort` | 백성준 | 박민재 | `linkToTicket(List<attachmentId>, ticketId, replyId)` |
| `FaqQueryPort` | 백성준 | 신수진 | `findPublishedByCategory(category, keyword, limit)` |
| `MailSender` | 신수진 | 백성준, 박민재 | `sendResolvedMail(ResolvedMailCommand)` (고객명, 이메일, 티켓번호, 답변 요약, 설문 링크) / `sendAgentReplyMail(AgentReplyMailCommand)`(박민재) / `sendPasswordResetMail(PasswordResetMailCommand)`(백성준, 회원·비회원 공용) |
| `FileStorage` | 신수진 | 백성준 | `upload(MultipartFile, keyPrefix)` → key(`keyPrefix/UUID.ext`), `getDownloadUrl(key)`(prod presigned GET 10분 / local `null`), `load(key)`(local 스트림 전용), `delete(key)` |
| `AiResultPort` | 신수진 | 박민재 | `markOverridden(ticketId, memberId, category, priority)` (상담원 수동 분류 수정 기록) |
| `NotificationPort` | 박민재 | 백성준, 신수진 | `notify(receiverId, type, ticketId, message)` |
| `TicketGuestPort` | 박민재 | 백성준 | `verifyGuest(ticketNo, email)` → ticketId, `findGuestPasswordHash(ticketId)` → BCrypt 해시, `updateGuestPassword(ticketId, passwordHash)` (비회원 로그인·조회 비밀번호 재설정). 세 메서드 모두 없으면 예외가 아니라 null — 존재 여부가 응답으로 드러나면 계정 열거가 된다. BCrypt 비교·해싱은 호출자(백성준) 책임이라 원문 비밀번호는 경계를 넘지 않는다 |
| `MemberQueryPort` | 백성준 | 박민재, 신수진 | `findAssignableAgents()`, `getMember(id)`, `touchLastAssigned(agentId)` |

### 5.3 핵심 시퀀스 — 접수부터 종료까지
```mermaid
sequenceDiagram
  participant CU as 고객(Front 백성준)
  participant T as TicketService(박민재)
  participant AI as AiService(신수진)
  participant AS as AssignmentService(박민재)
  participant N as Notification/WS(박민재)
  participant S as SurveyService(백성준)
  participant M as MailSender(신수진)

  CU->>T: POST /api/tickets (+attachmentIds)
  T->>T: 저장(RECEIVED, 기본 NORMAL, SLA due 계산)
  T-->>CU: 201 티켓번호
  T--)AI: TicketCreatedEvent (after commit, async)
  AI->>AI: LLM 분류(유형/긴급도/감정)
  AI->>T: applyClassification(...)
  T->>T: 우선순위·SLA 재계산
  T->>AS: 자동 배정(최소 부하)
  AS->>N: TicketAssignedEvent → /user/queue/notifications
  Note over T: 상담원 답변(AI 초안 활용) → IN_PROGRESS → RESOLVED
  T--)S: TicketStatusChangedEvent(to=RESOLVED)
  S->>S: SURVEY 토큰 생성
  S->>M: sendResolvedMail(설문 링크)
  CU->>S: POST /api/surveys/{token}
  S--)T: SurveySubmittedEvent → CLOSED
```

---

## 6. WebSocket (박민재)
| 항목 | 값 |
|---|---|
| 엔드포인트 | `/ws` (SockJS 미사용, 순수 WebSocket) |
| 앱 prefix | `/app` |
| 브로커 | `/topic`, `/queue` (SimpleBroker) |
| 인증 | STOMP `CONNECT` 헤더 `Authorization: Bearer {accessToken}` → `StompAuthInterceptor` |
| 개인 알림 | `/user/queue/notifications` |
| 채팅 대기 상태 | `/user/queue/chat-status` (대기 순번·연결·시간 초과) |
| 대기열 처리 | 10초 주기 스케줄러 + 상담원 available ON/티켓 종료 시 즉시 재시도, FIFO |
| 콘솔 공지 | `/topic/console/tickets` (신규/상태 변경 → 목록 갱신) |
| 채팅 수신 | `/topic/chat/{roomId}` |
| 채팅 송신 | `/app/chat/{roomId}/send` |

---

## 7. 로컬 환경 (백성준)
`docker-compose.yml`
```yaml
services:
  postgres:
    image: postgres:16
    container_name: helpnest-db
    environment:
      POSTGRES_DB: helpnest
      POSTGRES_USER: helpnest
      POSTGRES_PASSWORD: helpnest
      TZ: Asia/Seoul
    ports: ["5432:5432"]
    volumes: [helpnest-data:/var/lib/postgresql/data]
volumes:
  helpnest-data:
```
환경 변수 (`.env.example`만 커밋)
| 키 | 사용 | 소유 |
|---|---|---|
| `JWT_SECRET` | 토큰 서명 | 백성준 |
| `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | DB | 백성준 |
| `LLM_PROVIDER`, `LLM_API_KEY`, `LLM_MODEL` | AI | 신수진 |
| `AWS_REGION`, `S3_BUCKET`, `SES_FROM_EMAIL` | AWS (prod) | 신수진 |
| `FRONT_ORIGIN` | CORS(`/api/attachments`·`/ws`), 메일 링크(설문·비밀번호 재설정·답변 알림). 기본값은 `local` 프로필에만 | 백성준 |
| `REFRESH_COOKIE_SECURE` | Refresh 쿠키 Secure 여부 (로컬 `false` / 배포 `true`). SameSite는 항상 `Lax`, Path `/`, Domain 미지정 — REST가 프록시로 같은 출처가 되므로 가능 | 백성준 |
| `BACKEND_ORIGIN` | Front **서버 전용**(rewrites 대상). 로컬 `http://localhost:8080` / 배포 `https://xxxx.cloudfront.net` | 백성준 |
| `NEXT_PUBLIC_UPLOAD_BASE_URL` | Front — 첨부 업로드 직접 호출 주소 (로컬 `http://localhost:8080` / 배포 CloudFront) | 백성준 |
| `NEXT_PUBLIC_WS_URL` | Front — `ws://localhost:8080/ws` / 배포 `wss://xxxx.cloudfront.net/ws` | 백성준 |
| `PROXY_SECRET` | Front·Back **같은 값**. Amplify 경유 요청 표시용 비밀 헤더(§2.1). 로컬은 비워 둔다 | 백성준 |

### 7.1 배포 환경변수 체크리스트

**백엔드(EC2)** — 키·예시는 `helpnest-back/deploy/helpnest.env.example`(신수진)을 그대로 채운다. `SPRING_PROFILES_ACTIVE=prod` 는 `deploy/helpnest-back.service` 가 넣는다.
백성준 소유 키는 기본값이 `local` 프로필에만 있어 **비면 기동이 실패**한다(`Could not resolve placeholder '…'`).

| 키 | 배포 값 | 비었을 때 |
|---|---|---|
| `JWT_SECRET` | 32바이트 이상 랜덤 | 기동 실패 |
| `DB_URL`·`DB_USERNAME`·`DB_PASSWORD` | RDS 엔드포인트·계정 | 기동 실패 |
| `FRONT_ORIGIN` | Amplify 도메인 `https://main.xxxx.amplifyapp.com` (끝 `/` 없이) | 기동 실패. 값이 틀리면 첨부 업로드·WebSocket 이 CORS 로 막히고 메일 링크가 엉뚱한 곳을 가리킨다 |
| `REFRESH_COOKIE_SECURE` | `true` (생략 시 기본 `true`) | — |
| `PROXY_SECRET` | 32바이트 이상 랜덤, **Amplify 와 같은 값** | 기동은 된다. Amplify 경유 요청이 전부 Amplify 서버 IP 하나로 묶여 로그인·티켓 접수 요청 제한을 전체 사용자가 함께 쓴다 |

**프론트(Amplify)** — 세 값 모두 **빌드 결과물에 고정**된다(rewrites 대상, `NEXT_PUBLIC_*` 인라인). Amplify 콘솔 환경변수에 넣고, **값을 바꾸면 재빌드**해야 반영된다.
빠지면 빌드는 성공하고 로그에 `[배포 확인] 빌드 환경변수 없음: …` 경고만 남는다(CI 빌드가 값 없이 돌아 실패시키지 않음).

| 키 | 배포 값 | 비었을 때 |
|---|---|---|
| `BACKEND_ORIGIN` | `https://xxxx.cloudfront.net` | REST 가 `localhost:8080` 으로 프록시 → 로그인부터 전부 실패 |
| `NEXT_PUBLIC_UPLOAD_BASE_URL` | `https://xxxx.cloudfront.net` | 첨부 업로드가 Amplify 를 거쳐 크기 제한에 걸릴 수 있다(§2.1) |
| `NEXT_PUBLIC_WS_URL` | `wss://xxxx.cloudfront.net/ws` | STOMP 연결 실패 → 알림·채팅·콘솔 목록 실시간 갱신 중단 |
| `PROXY_SECRET` | 백엔드와 같은 값 (`next.config.ts` `env` 로 `proxy.ts` 에만 박힌다, 클라이언트 번들 아님) | 백엔드와 같음 — 요청 제한이 Amplify IP 하나로 묶인다. **백엔드보다 먼저 넣고 재빌드**한다 |

---

## 8. AWS 배포 (신수진, Sprint 4)

운영 URL — 프론트 `https://main.dznupx7lhr770.amplifyapp.com` · API `https://d1rf8nrptjrzmx.cloudfront.net` (서울 `ap-northeast-2`, 10/8 운영 반영).
public repo 라서 EC2 IP·인스턴스 ID, RDS 엔드포인트, 보안그룹 ID, 계정 ID, 비밀값은 적지 않는다(담당자 로컬 기록).

| 자원 | 용도 | 구성 |
|---|---|---|
| RDS PostgreSQL 16 | 운영 DB | `db.t4g.micro`, 퍼블릭 액세스 없음, 보안그룹은 EC2 에서만 5432. 스키마는 백엔드 기동 시 Flyway `db/migration` 만 적용(시드 없음). 최초 ADMIN 1명은 EC2 에서 psql 로 1회 생성(back #102), 나머지 계정은 관리자 API |
| EC2 | Spring Boot 백엔드 (8080) | Amazon Linux 2023 · Corretto 21 · systemd `helpnest-back`(`deploy/`), `prod` 프로필. **CloudFront 뒤에 배치**(PRD Q20) — 보안그룹은 CloudFront 관리형 prefix list(`com.amazonaws.global.cloudfront.origin-facing`)만 8080 허용. 환경변수는 `/etc/helpnest/helpnest.env`(600 root, 키 목록은 `deploy/helpnest.env.example`) |
| **CloudFront** | 백엔드 HTTPS·WSS (`*.cloudfront.net` 인증서) | Origin: EC2 **퍼블릭 DNS**(IP 불가), HTTP 8080 / Viewer **HTTPS only**. `/api/*`: **CachingDisabled** + **AllViewerExceptHostHeader**, GET~DELETE 전체. `/ws*`: CachingDisabled + WebSocket 헤더 전달. 받은 `X-Forwarded-For` 는 지우지 않고 **뒤에 덧붙인다**(§2.1 클라이언트 IP 판별) |
| **Amplify Hosting** | Next.js 프론트 (PRD Q9) | 앱 `helpnest-front`(WEB_COMPUTE). GitHub `helpnest-front` `main` 만 연결 → main 머지 시 자동 빌드·배포(dev 자동 배포 없음). GitHub 연결은 repo 소유자(백성준) 계정의 **Amplify GitHub App(ap-northeast-2, `helpnest-front` 만 선택)** + PAT 로 `update-app` — 콘솔에서 앱을 새로 만들면 도메인(appId)이 바뀌어 CORS 를 다시 맞춰야 한다. 빌드: Node **22**(Live package updates), `amplify.yml` 없음(자동 감지). 빌드 환경변수 4개는 §7.1 |
| S3 | 첨부 (Private + Presigned URL) | 버킷 `helpnest-attachments-prod-2026`, 퍼블릭 차단, 버전관리 없음. **CORS: Amplify origin 의 `GET` 만** 허용(presigned URL 을 `fetch` 로 받음) |
| SES | 결과/설문·답변 알림·재설정 메일 | 도메인 `helpnest.kro.kr` 인증(Easy DKIM), **프로덕션 액세스 승인**(10/6). 발신 `no-reply@helpnest.kro.kr` |
| IAM | EC2 인스턴스 Role `helpnest-ec2-role` | `s3:PutObject/GetObject/DeleteObject`(운영 버킷 한정) + `ses:SendEmail`. 앱은 키 없이 SDK 기본 체인으로 Role 사용 |
| 연결 확인 | 배포 전후 | 로컬: `AWS_LIVE=true AWS_PROFILE=<프로필> SES_FROM_EMAIL=… SES_TEST_TO=… ./mvnw test -Dtest=AwsLiveTest`(10/6 2/2). 운영: EC2 Role 로 S3 업로드·presigned GET·삭제 + 앱 경로 SES 실수신(10/8) |
| 비밀값 | EC2 env 파일 / Amplify 환경변수 | 커밋 금지. `PROXY_SECRET` 처럼 두 곳이 같아야 하는 값은 사람이 옮기지 않고 생성 즉시 두 곳에 넣고 해시 앞자리로 대조 |

### 8.1 운영 절차

| 상황 | 절차 |
|---|---|
| 백엔드 재배포 | 로컬(Git Bash)에서 `deploy/deploy.sh ec2-user@<host> <key.pem>` — **커밋된 HEAD 만** 빌드(작업 폴더 빌드는 `application-local-secret.yml` 이 jar 에 들어감) → 이전 jar 를 `.bak` 으로 보관 → 재시작 → health 대기. 운영은 **main** 기준으로 맞춘다 |
| 동시 실행 금지 | `deploy.sh` 는 고정 경로 `/tmp/helpnest-back.jar` 를 쓴다. 두 번 겹치면 한쪽 업로드가 지워지고 `.bak` 이 현재 jar 로 덮인다(10/8 실제 발생). 자동 배포 스크립트를 걸었다면 **멈춘 뒤 프로세스가 실제로 끝났는지** 확인 |
| 롤백 | `sudo cp -p /opt/helpnest/helpnest-back.jar.bak /opt/helpnest/helpnest-back.jar && sudo systemctl restart helpnest-back`. 배포 후 `sha256sum /opt/helpnest/*.jar*` 로 `.bak` 이 **직전 운영 버전**(현재와 다른 해시)인지 확인. Flyway 는 되돌리지 않는다 — 스키마 롤백은 새 마이그레이션으로 |
| 로그 | `sudo journalctl -u helpnest-back -f` / `-n 200` |
| 환경변수만 변경 | 백엔드: env 파일 수정(백업 `.bak-<시각>`) → `sudo systemctl restart helpnest-back`. 프론트: Amplify 환경변수 수정 → **재빌드해야 반영**(빌드 결과물에 고정) |
| 코드 + 환경변수가 함께 바뀌는 릴리스 | **환경변수를 먼저** 두 곳에 넣고 → 머지·배포. 새 값을 읽지 않는 이전 버전에는 넣어 둬도 영향 없음 (예: back #107·front #59 의 `PROXY_SECRET`) |
| Amplify 환경변수를 CLI 로 변경 | `aws amplify update-app --environment-variables` 는 **전체를 덮어쓴다** — 기존 키(`_LIVE_UPDATES` 포함)를 모두 함께 넣는다. 값에 쉼표·따옴표가 있으면 `--cli-input-json file://…` |
| `FRONT_ORIGIN`(Amplify 도메인) 변경 | EC2 env `FRONT_ORIGIN` + S3 버킷 CORS 를 함께 바꾼다(첨부 업로드·`/ws`·메일 링크) |
| `PROXY_SECRET` 교체 | `openssl rand -hex 32` → EC2 env + Amplify 환경변수에 같은 값 → 백엔드 재시작 + Amplify 재빌드. 사이에는 Amplify 경유 요청이 서버 IP 하나로 묶인다 |

### 8.2 운영 확인 기록 (10/8 스모크 테스트)
- 06 통합 시나리오 16개 + AWS 연결을 운영 URL 에서 실행 → 15개 바로 통과, CSV 는 수정 후 통과. SES 8통 Gmail 실수신. 테스트 데이터는 `[SMOKE]` 제목·`t2684258+smoke-*` 계정으로 구분
- 운영에서만 드러난 문제와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| CSV 다운로드가 끝나지 않음 | Amplify 프록시가 `text/csv` 본문의 **BOM 3바이트를 지우고 Content-Length 는 유지** | 응답을 `application/octet-stream` 으로(back #104). BOM 은 본문에 그대로 |
| 위조 `X-Forwarded-For` 로 IP 요청 제한 우회 | `forward-headers-strategy: framework` 가 위조 가능한 왼쪽 값 사용 | 계정·티켓 기준 실패 제한(back #106) + XFF 오른쪽 신뢰 위치·`X-Helpnest-Proxy`(back #107·front #59). 실측 체인은 back #105 |
| 오래 열어 둔 탭이 재연결 시 `STOMP 인증 실패` 무한 반복 | 재연결 때 만료된 메모리 토큰 사용 | 재연결 전 토큰 재발급(front #58) |

- QA·스모크 데이터 정리: SQL 초안(박민재) + 안전 검사 보완(신수진)으로 운영 DB 예행 실행(끝에 ROLLBACK) 통과 → **Final 직전** 실행 후 S3 객체 삭제
