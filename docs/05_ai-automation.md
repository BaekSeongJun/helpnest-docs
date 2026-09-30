# 05. AI Automation 설계 (신수진)

## 1. 개요
| ID | 기능 | 트리거 | 결과 |
|---|---|---|---|
| AI-1 | 문의 자동 분류 (유형·긴급도·감정) | `TicketCreatedEvent` (비동기) / 재분류 버튼 | TICKET_AI_RESULT 저장 → `TicketClassificationPort` 로 티켓 반영 |
| AI-2 | 답변 초안 생성 | 상담원 "AI 초안" 버튼 | AI_DRAFT 저장 → 에디터에 삽입 → 상담원 수정 후 발송 |
| AI-3 | 결과 메일·설문 자동 발송 | `TicketStatusChangedEvent(RESOLVED)` → 백성준 설문 생성 → 신수진 `MailSender` | MAIL_LOG 저장 |

## 2. LLM 추상화
```
infra/llm/LlmClient.java            // interface: <T> T structured(String system, String user, Class<T> type)
infra/llm/SpringAiLlmClient.java    // Spring AI ChatClient 기반 구현
domain/ai/service/ClassifyService, DraftService
```
- 제공자는 `application-ai.yml` + 환경변수(`LLM_PROVIDER`, `LLM_API_KEY`, `LLM_MODEL`)로 교체 (PRD Q11).
- Spring AI **2.0.1** 사용 (Spring Boot 4.0/4.1 호환 GA 라인, `spring-ai-bom`으로 버전 관리) — [아키텍처 §1](02_architecture.md#1-기술-스택-백성준).
- 키가 없거나 `LLM_PROVIDER=mock`이면 **`MockLlmClient`**(키워드 규칙 기반)로 동작 → 백성준·박민재가 LLM 키 없이도 로컬 개발 가능.
- 타임아웃 10초, 재시도 1회, 입력 최대 4,000자(초과 시 앞부분 사용).
- **개인정보 마스킹:** 전송 전 전화번호(`010-****-1234`), 이메일(`h***@example.com`), 카드/계좌 형태 숫자열 마스킹.

## 3. AI-1 분류

### 3.1 흐름
1. `@TransactionalEventListener(phase = AFTER_COMMIT)` + `@Async` 로 `TicketCreatedEvent` 수신
2. `TicketQueryPort.getTicketSummary(ticketId)` 로 제목/본문/고객 선택 유형 조회
3. LLM 호출 → JSON 파싱·검증(enum 외 값이면 실패 처리)
4. TICKET_AI_RESULT 저장(`SUCCESS`)
5. 우선순위 결정: `urgency` 기준, `sentiment = NEGATIVE`면 1단계 상향(최대 URGENT)
6. `TicketClassificationPort.applyClassification(ticketId, category, priority, sentiment)` → 티켓 서비스(박민재)가 우선순위·SLA 재계산 후 자동 배정(`RECEIVED`일 때만, PRD 6.2)
7. 실패/타임아웃 → TICKET_AI_RESULT(`FAILED`) 저장 → `applyClassificationFailed(ticketId)` → 티켓 서비스(박민재)가 기본값(ETC·NORMAL)으로 자동 배정(`RECEIVED`일 때만)

### 3.2 프롬프트
**System**
```
너는 한국어 고객상담 티켓 분류기다. 고객 문의를 읽고 반드시 아래 JSON 형식으로만 답한다.
- category: DELIVERY(배송), REFUND(환불), EXCHANGE(교환), PAYMENT(결제), ACCOUNT(계정/로그인), SERVICE_ERROR(서비스 오류), ETC(기타) 중 하나
- urgency: URGENT(금전 손실·서비스 전체 장애·법적 언급), HIGH(업무/사용 불가), NORMAL(일반 문의), LOW(단순 정보 요청·제안) 중 하나
- sentiment: NEGATIVE(불만·분노·재문의 언급), NEUTRAL, POSITIVE 중 하나
- summary: 상담원이 한눈에 볼 수 있는 한 문장 요약(60자 이내)
- confidence: 0.0 ~ 1.0
고객이 선택한 유형은 참고만 하고 본문 내용을 우선한다.
```
**User**
```
[고객 선택 유형] {categoryHint}
[제목] {title}
[본문] {maskedContent}
```
**출력(JSON Schema로 검증)**
```json
{ "category": "DELIVERY", "urgency": "HIGH", "sentiment": "NEGATIVE",
  "summary": "10/1 주문 상품 배송 조회 불가, 지연에 불만", "confidence": 0.86 }
```

### 3.3 수동 수정
- 상담원이 티켓 상세 화면(박민재)에서 분류 수정 → 티켓 서비스(박민재)가 반영 후 `AiResultPort.markOverridden(...)` 호출 → `overridden_by` 기록.
- 대시보드에 "AI 분류 정확도(수정되지 않은 비율)" 지표 표시 (선택).

## 4. AI-2 답변 초안

### 4.1 참고 자료 검색 (RAG-lite, 추가 인프라 없음)
| 소스 | 방법 | 개수 |
|---|---|---|
| FAQ | `FaqQueryPort.findPublishedByCategory(category, keyword, 3)` — 같은 유형 + 제목/본문 키워드 `ILIKE` | 최대 3 |
| 과거 답변 | `TicketQueryPort.findResolvedReplies(category, 3)` — 같은 유형, RESOLVED/CLOSED, 만족도 4 이상 우선 | 최대 3 |
| 대화 맥락 | 현재 티켓의 고객 글 + 기존 답변 최근 5개 | - |

> 확장(선택): pgvector 임베딩 검색. MVP에서는 키워드/유형 기반으로 충분.

### 4.2 프롬프트
**System**
```
너는 {회사명} 고객상담원의 답변 초안을 작성하는 도우미다.
- 정중한 존댓말, 3~6문장, 첫 문장은 공감/사과, 마지막은 추가 문의 안내.
- [참고 자료]에 있는 정책·절차만 사용하고, 없는 내용(환불 금액, 날짜 약속 등)은 지어내지 말고 "[확인 필요: ...]"로 표시한다.
- 고객 감정이 NEGATIVE면 불편에 대한 사과를 먼저 한다.
```
**User**
```
[티켓] 유형={category}, 감정={sentiment}
[고객 문의] {maskedContent}
[이전 대화] {recentReplies}
[참고 자료]
- FAQ#{id}: Q {question} / A {answer}
- 과거답변#{id}: {content}
```
- 응답 저장: AI_DRAFT(`reference_refs`에 사용한 FAQ/답변 ID) → 화면에 참고 자료 링크 표시
- 상담원이 발송하면 티켓 서비스(박민재)가 `ticket_reply.ai_draft_id` 기록 → 초안 활용률 계산

### 4.3 프론트 컴포넌트 (신수진 소유, 박민재 페이지에서 사용)
| 컴포넌트 | props | 동작 |
|---|---|---|
| `AiAnalysisPanel` | `ticketId` | 유형/긴급도/감정 배지, 요약, 신뢰도, 재분류 버튼 |
| `AiDraftButton` | `ticketId`, `onInsert(text, draftId)` | 초안 생성 → 미리보기 모달 → "에디터에 삽입" 클릭 시 `onInsert` 호출 |

## 5. AI-3 결과 메일
| 항목 | 내용 |
|---|---|
| 트리거 | 백성준 `SurveyListener`가 설문 생성 후 `MailSender.sendResolvedMail(command)` 호출 |
| 수신자 | 회원 이메일 또는 `guest_email` |
| 제목 | `[HelpNest] 문의({ticketNo})가 해결되었습니다` |
| 본문 | 고객명, 문의 제목, 최종 답변 요약(LLM 1문장 요약, 실패 시 답변 앞 200자), 설문 버튼(`{FRONT_ORIGIN}/survey/{token}`), 설문 만료 시각 |
| 구현 | `local`: `LogMailSender`(콘솔 출력 + MAIL_LOG `LOGGED`) / `prod`: `SesMailSender` |
| 실패 | MAIL_LOG `FAILED` + 5분 간격 최대 3회 재시도 스케줄러 |

### 5.1 기타 메일 (같은 `MailSender`, 신수진 구현)
| 메일 | 호출자 | 내용 |
|---|---|---|
| `AGENT_REPLY` | 박민재 (`ReplyCreatedEvent`, 공개 답변) | `[HelpNest] 문의({ticketNo})에 답변이 등록되었습니다` / 답변 앞 200자 + 문의 보기 링크(회원: 내 문의, 비회원: 조회 페이지). 10분 내 연속 답변은 1통 |
| `PASSWORD_RESET` | 백성준 | 재설정 링크 `{FRONT_ORIGIN}/reset-password?token=` (30분) |
| `GUEST_PASSWORD_RESET` | 백성준 | 조회 비밀번호 재설정 링크 `{FRONT_ORIGIN}/inquiry/lookup/reset?token=` (30분) |

> 메일 HTML 템플릿(Thymeleaf 등)은 신수진 소유 `resources/templates/mail/`. 색·로고는 디자인 토큰의 primary 값과 맞춘다.

## 6. 품질 확인
- Sprint 1에 테스트용 문의 30건(유형별 4~5건, 불만 10건 포함)을 `src/test/resources/ai/samples.json`으로 만들고 분류 정확도를 측정해 README에 기록.
- 목표: 유형 정확도 80% 이상, 불만 감지 재현율 80% 이상.
- LLM 응답 시간·실패율은 TICKET_AI_RESULT의 `latency_ms`, `status`로 대시보드에서 확인(선택).
