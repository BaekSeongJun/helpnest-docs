# 04. API 명세

> 각 섹션 제목의 (백성준)/(박민재)/(신수진)가 **컨트롤러 소유자이자 이 섹션의 문서 수정 권한자**다. API를 변경하면 같은 날 이 문서 PR도 올린다.

## 1. 공통 규칙 (백성준)

- Base URL: `/api`
- 인증: `Authorization: Bearer {accessToken}` / 비회원 티켓 조회는 Guest 토큰(해당 ticketId 한정 scope)
- 토큰 보관: Access·Guest 토큰은 프론트 **메모리**, Refresh는 **httpOnly 쿠키**(JS 접근 불가). REST는 Next.js 프록시(`/api/*` rewrites)로 같은 출처에서 호출하므로 CORS 불필요. 예외로 **첨부 업로드(`POST /api/attachments`)와 `/ws`만** 백엔드에 직접 호출 → 이 두 경로만 CORS/Origin에 `FRONT_ORIGIN` 허용 ([02 §2.1](02_architecture.md#21-요청-경로-배포-환경))
- 날짜: ISO-8601 (`2026-10-01T10:30:00+09:00`)
- 페이지: `?page=0&size=20&sort=createdAt,desc`

### 1.1 응답 형식
```json
// 성공
{ "success": true, "data": { ... }, "error": null }
// 실패
{ "success": false, "data": null, "error": { "code": "TICKET_INVALID_TRANSITION", "message": "RECEIVED에서 RESOLVED로 변경할 수 없습니다." } }
// 페이지
{ "success": true, "data": { "content": [], "page": 0, "size": 20, "totalElements": 0, "totalPages": 0 } }
```

### 1.2 에러 코드 prefix
| prefix | 소유 | 예 |
|---|---|---|
| `COMMON_`(`COMMON_TOO_MANY_REQUESTS` 등), `AUTH_`, `MEMBER_`, `FILE_`, `FAQ_`, `TEMPLATE_`, `SURVEY_` | 백성준 | `AUTH_TOKEN_EXPIRED`, `FILE_SIZE_EXCEEDED`, `SURVEY_ALREADY_SUBMITTED` |
| `TICKET_`, `ASSIGN_`, `SLA_`, `CHAT_` | 박민재 | `TICKET_INVALID_TRANSITION`, `ASSIGN_NO_AVAILABLE_AGENT` |
| `AI_`, `REPORT_` | 신수진 | `AI_PROVIDER_TIMEOUT` |

---

## 2. 인증·회원 (백성준)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| POST | `/api/auth/signup` | 공개 | 고객 회원가입 |
| POST | `/api/auth/login` | 공개 | → 본문 `{accessToken, member}` + `Set-Cookie: refreshToken`(httpOnly, Secure, SameSite=Lax, Path=/) |
| POST | `/api/auth/refresh` | 공개(쿠키) | 쿠키의 Refresh로 Access 재발급 + Refresh 회전(새 쿠키) → `{accessToken, member}` |
| POST | `/api/auth/logout` | 로그인 | Refresh 폐기 + 쿠키 삭제 |
| POST | `/api/auth/guest` | 공개 | `{ticketNo, email, password}` → Guest 토큰 |
| POST | `/api/auth/password/reset-request` | 공개 | `{email}` → 항상 200 (계정 존재 여부 비노출), 재설정 메일 |
| POST | `/api/auth/password/reset` | 공개 | `{token, newPassword}` |
| POST | `/api/auth/guest/reset-request` | 공개 | `{ticketNo, email}` → 항상 200, 조회 비밀번호 재설정 메일 |
| POST | `/api/auth/guest/reset` | 공개 | `{token, newPassword}` → `TicketGuestPort.updateGuestPassword` |
| GET | `/api/members/me` | 로그인 | 내 정보 |
| PATCH | `/api/members/me` | 로그인 | `{name, phone}` 내 정보 수정 |
| PATCH | `/api/members/me/password` | 로그인 | `{currentPassword, newPassword}` → Refresh 토큰 전체 폐기 |
| PATCH | `/api/members/me/availability` | AGENT | `{available: true}` → 내 정보. 역할이 AGENT 가 아니면(LEAD·ADMIN 포함) `403 MEMBER_NOT_AGENT` |
| GET | `/api/admin/members` | ADMIN | 목록 `?role=&status=&page=&size=` (기본 가입일 최신순 20개) → 페이지 `{memberId, email, name, phone, role, status, available, createdAt}` |
| POST | `/api/admin/members` | ADMIN | `{email, password, name, phone?, role: AGENT\|LEAD}` → `201`. 검증은 회원가입과 동일 |
| PATCH | `/api/admin/members/{id}` | ADMIN | `{role?, status?}` (null 은 유지) |

> - 계정 관리 제약: 본인 역할 변경·비활성화 금지(`MEMBER_SELF_CHANGE_FORBIDDEN`), 고객 ↔ 직원(AGENT·LEAD·ADMIN) 역할 전환 금지(`MEMBER_ROLE_NOT_ALLOWED`). AGENT 가 아닌 역할이 되면 `available=false`.
> - 역할·상태가 바뀌면 그 회원의 Refresh 토큰을 전부 폐기한다 → 다음 재발급부터 새 권한/차단 적용. 이미 발급된 Access 토큰은 만료(30분)까지 유효.

## 3. 첨부 (백성준)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| POST | `/api/attachments` | 공개(비회원 포함) | multipart 필드 `files` 반복 (≤5개, 각 ≤10MB) → `201` `[{attachmentId, originalName, size}]` |
| GET | `/api/attachments/{id}/download` | 업로더, AGENT+, 티켓 고객 본인, Guest(같은 티켓) | prod `302` presigned URL / 로컬 스트림(`filename*=UTF-8''…`). 고객·Guest 는 내부 메모 첨부 불가 |

> - 허용 확장자: jpg·jpeg·png·gif·webp·pdf·txt·doc·docx·xls·xlsx·ppt·pptx·hwp·hwpx. Content-Type 은 클라이언트 값이 아니라 확장자로 서버가 정한다.
> - 오류: `ATTACHMENT_EMPTY`, `ATTACHMENT_TOO_MANY`, `ATTACHMENT_TOO_LARGE`, `ATTACHMENT_INVALID_TYPE`, `ATTACHMENT_LINK_DENIED`(티켓 접수 시 연결 불가 — 남의 첨부·이미 연결된 첨부)
> - `attachmentId` 는 추측 불가 난수(JS 안전 정수 범위)다. 비회원 첨부는 이 ID 를 아는 것이 소유 증명이므로 화면·URL 에 불필요하게 노출하지 않는다.
> - 업로드 후 24시간 안에 티켓·답글에 연결되지 않은 첨부는 자동 삭제된다.

## 4. FAQ · 템플릿 (백성준)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| GET | `/api/faqs` | 공개 | `?category=&keyword=` |
| GET | `/api/faqs/{id}` | 공개 | 조회수 증가 |
| GET | `/api/faqs/suggest` | 공개 | `?q=` 접수 폼 추천(상위 3) |
| POST/PUT/DELETE | `/api/admin/faqs[/{id}]` | LEAD, ADMIN | CRUD |
| GET | `/api/templates` | AGENT+ | `?category=&keyword=` |
| POST/PUT/DELETE | `/api/admin/templates[/{id}]` | LEAD, ADMIN | CRUD |

## 5. 만족도 설문 (백성준)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| GET | `/api/surveys/{token}` | 공개(토큰) | 티켓번호·제목·만료 여부 |
| POST | `/api/surveys/{token}` | 공개(토큰) | `{rating: 1~5, comment}` → SurveySubmittedEvent |
| GET | `/api/console/surveys` | AGENT(본인 담당분), LEAD+ | 설문 결과 목록 `?from=&to=&rating=&agentId=&category=&page=` |
| GET | `/api/console/surveys/summary` | AGENT(본인), LEAD+ | 응답률, 평균 별점, 별점 분포 (같은 필터) |

## 6. 고객 이력 묶음 (백성준)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| GET | `/api/console/customers/by-ticket/{ticketId}` | AGENT+ | 해당 티켓 고객의 과거 문의 목록 + 요약(총 건수, 평균 만족도) |
| GET | `/api/console/customers/{customerKey}/tickets` | AGENT+ | `customerKey` = `M-{memberId}` 또는 `G-{email}` |

---

## 7. 티켓 (박민재)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| POST | `/api/tickets` | 공개(비회원 포함) | 문의 접수 |
| GET | `/api/tickets/my` | CUSTOMER | 내 문의 목록 |
| GET | `/api/tickets/{id}` | 고객 본인/Guest 토큰/AGENT+ | 상세(고객에겐 내부 메모 제외) |
| POST | `/api/tickets/{id}/replies` | 고객 본인/Guest | 추가 답글 (RESOLVED면 재문의 → IN_PROGRESS) |
| GET | `/api/console/tickets` | AGENT(본인), LEAD+ | `?status=&priority=&category=&agentId=&sla=WARNING\|BREACHED&keyword=` |
| GET | `/api/console/tickets/{id}` | AGENT+ | 상세 + 이력 |
| PATCH | `/api/console/tickets/{id}/status` | 담당 AGENT, LEAD+ | `{toStatus, memo}` |
| PATCH | `/api/console/tickets/{id}/assign` | LEAD+ | `{agentId, memo}` 수동/재배정 |
| POST | `/api/console/tickets/{id}/assign/auto` | LEAD+ | 자동 배정 재시도 |
| PATCH | `/api/console/tickets/{id}/classification` | 담당 AGENT, LEAD+ | `{category, priority}` 수동 수정 (신수진의 AI 결과 overridden 기록은 신수진 포트 호출) |
| POST | `/api/console/tickets/{id}/replies` | 담당 AGENT, LEAD+ | `{content, isInternal, aiDraftId?, attachmentIds?}` |
| GET | `/api/console/tickets/{id}/histories` | AGENT+ | 상태 이력 |

> 요청 제한: 공용 `RateLimitFilter`(백성준, IP 기준) → 초과 시 `429 COMMON_TOO_MANY_REQUESTS` + `Retry-After`(초).
>
> | 요청 | 한도 |
> |---|---|
> | `POST /api/tickets`, `POST /api/attachments` | **비회원만**(Authorization 헤더 없음) 10분 5건 |
> | `POST /api/auth/login`, `POST /api/auth/guest`(조회 비밀번호 대입 방지) | 10분 10건 |
> | `POST /api/auth/password/reset-request`, `/api/auth/guest/reset-request` | 10분 5건 |
>
> 본문 값 기준 제한(비회원 **동일 이메일 1시간 5건**)은 서비스에서 `RateLimiter.tryAcquire("ticket-email:" + email, 5, Duration.ofHours(1))` 호출(박민재 `TicketService`). 테스트에서는 `app.rate-limit.enabled=false`(back `src/test/resources/config/application.yml`).
> 본문(`content`)은 일반 텍스트, 최대 5,000자. 서버는 원문 저장, 프론트는 이스케이프 출력.

**POST /api/tickets 요청 예**
```json
{
  "title": "주문한 상품이 아직 안 왔어요",
  "content": "10/1 주문했는데 배송 조회가 안 됩니다. 너무 늦네요.",
  "categoryHint": "DELIVERY",
  "attachmentIds": [12, 13],
  "guest": { "name": "홍길동", "email": "hong@example.com", "password": "1234" }
}
```
응답: `{ "ticketId": 101, "ticketNo": "HN-20261002-000123", "status": "RECEIVED" }`

## 8. SLA (박민재)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| GET | `/api/admin/sla-policies` | LEAD+ | 목록 |
| PUT | `/api/admin/sla-policies/{priority}` | ADMIN | `{responseMinutes, warningRatio}` |

## 9. 알림 (박민재)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| GET | `/api/notifications` | 로그인 | `?unreadOnly=true` |
| GET | `/api/notifications/unread-count` | 로그인 | |
| PATCH | `/api/notifications/{id}/read` | 본인 | |
| PATCH | `/api/notifications/read-all` | 본인 | |

## 10. 채팅 (박민재)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| POST | `/api/chat/rooms` | CUSTOMER | 채팅 요청 → `{roomId, status: WAITING\|OPEN, position}` (상담원 배정 시 OPEN + CHAT 티켓 생성) |
| (STOMP) | `/app/chat/{roomId}/send` | CUSTOMER | WAITING 상태에서도 전송 가능 — 대기 중 메시지는 저장되어 상담원 연결 시 표시, 전환 시 티켓 본문이 됨 |
| POST | `/api/chat/rooms/{roomId}/convert` | CUSTOMER | 대기 5분 초과 후 "문의로 남기기" → 대기 메시지로 티켓 생성, 방 CONVERTED → `{ticketNo}` |
| DELETE | `/api/chat/rooms/{roomId}` | CUSTOMER | 대기 중 나가기 → CANCELED |
| GET | `/api/chat/rooms` | CUSTOMER(본인), AGENT(담당) | 방 목록 |
| GET | `/api/chat/rooms/{roomId}/messages` | 참여자 | `?before={messageId}&size=50` |
| PATCH | `/api/chat/rooms/{roomId}/close` | 담당 AGENT | 종료 (선택: 티켓 RESOLVED) |

## 11. WebSocket / STOMP (박민재)
| 방향 | 목적지 | payload |
|---|---|---|
| 구독 | `/user/queue/notifications` | `{notificationId, type, ticketId, message, createdAt}` |
| 구독 | `/user/queue/chat-status` | `{roomId, status: WAITING\|OPEN\|TIMEOUT, position, agentName?}` (고객 대기열 상태) |
| 구독 | `/topic/console/tickets` | `{ticketId, event: CREATED\|UPDATED, status, priority, agentId}` |
| 구독 | `/topic/chat/{roomId}` | `{messageId, senderId, senderName, content, createdAt}` |
| 발행 | `/app/chat/{roomId}/send` | `{content}` |

---

## 12. AI (신수진)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| GET | `/api/console/tickets/{id}/ai` | AGENT+ | 분류 결과 `{category, urgency, sentiment, summary, confidence, status}` |
| POST | `/api/console/tickets/{id}/ai/classify` | AGENT+ | 재분류 (동기, 타임아웃 10초) |
| POST | `/api/console/tickets/{id}/ai/drafts` | 담당 AGENT, LEAD+ | 초안 생성 → `{draftId, content, references[]}` |
| GET | `/api/console/tickets/{id}/ai/drafts` | AGENT+ | 초안 목록 |

## 13. 대시보드 · 리포트 (신수진)
| Method | URL | 권한 | 설명 |
|---|---|---|---|
| GET | `/api/dashboard/summary` | LEAD+ | `?period=TODAY\|7D\|30D` KPI(전체/미배정/SLA위반율/평균응답/평균만족도) + 상태·유형 분포 |
| GET | `/api/dashboard/agents` | LEAD+ | 상담원별 처리현황 |
| GET | `/api/dashboard/agents/me` | AGENT | 본인 처리현황 |
| GET | `/api/reports/monthly` | LEAD+ | `?month=2026-10` 유형별 리포트 + 전월 대비 |
| GET | `/api/reports/monthly/export` | LEAD+ | `?month=2026-10` → `text/csv` (UTF-8 BOM), `Content-Disposition: attachment; filename=helpnest_report_2026-10.csv` |
| GET | `/api/dashboard/agents/export` | LEAD+ | `?period=` 상담원별 처리현황 CSV |
| GET | `/actuator/health` | 공개 | 배포 헬스체크 |
