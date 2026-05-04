# 프로젝트 현황 분석

## 전체 아키텍처

```
브라우저 (Horizon UI)
  │  ① JWT 발급 요청 (50초마다)
  │  ② WebSocket 연결 + 채팅
  ▼
[agent / Horizon Django]  ←── Keystone 세션 기반 인증 SSOT
  │  JWT 발급 (RS256, 60초)
  │  Redis에 세션 메타데이터 저장 (keystone_token 포함)
  ▼
[agent_server / FastAPI]  ←── AI 채팅 엔진
  │  JWT 검증 + replay 방지
  │  LangGraph StateGraph (graph_agent) 실행
  │  MCP Server 호출 (Slack, OpenStack)
  │  AsyncRedisSaver로 대화 체크포인트 저장
  ▼
Redis (공유)              ←── 세션·JTI·LangGraph 체크포인트
[redis/redis-stack-server] ←── RediSearch 모듈 필요 (AsyncRedisSaver 인덱스)
```

---

## 폴더별 현황

### agent/ (Horizon Django 플러그인)

Horizon에 붙는 Django 앱으로, 인증 SSOT 역할을 담당한다.

| 파일 | 역할 | 상태 |
|------|------|------|
| `ai/views.py` | JWT 발급 엔드포인트 (`POST /ai/session/issue/`) | ✅ 구현 완료 |
| `ai/urls.py` | URL 라우팅 | ✅ 구현 완료 |
| `jwt/jwt_utils.py` | RS256 서명으로 JWT 발급 | ✅ 구현 완료 |
| `common/redis_client.py` | Django용 Redis 동기 클라이언트 | ✅ 구현 완료 |

**구현된 내용:**
- 로그인된 Horizon 유저의 `user_id`, `username`, `project_id`, `roles`, `session_id`를 JWT payload에 포함
- JWT 유효기간 60초, RS256 서명 (private key 사용)
- Redis `chat:session:{session_key}`에 세션 메타데이터(JSON) 저장
  - 포함 필드: `user_id`, `username`, `project_id`, `roles`, `keystone_token`, `created_at`

**미구현 항목:**
- Celery 감사 로그 파이프라인 (AuditLog 모델, Task)

---

### agent_server/ (FastAPI AI Gateway)

LangGraph StateGraph 기반 AI 채팅 서버. WebSocket으로 클라이언트와 통신한다.

#### 파일 구조

```
agent_server/
└── agent_server/
    ├── main.py                        # FastAPI lifespan, MCP 클라이언트, 라우터 등록
    └── app/
        ├── graph_agent/
        │   ├── state.py               # ChatState 정의
        │   ├── schemas.py             # RouteDecision, RecoveryPolicy Pydantic 모델
        │   ├── agents.py              # LLM 초기화, mcp_tools, answer_generator
        │   ├── nodes.py               # 각 노드 함수
        │   └── graph.py               # StateGraph 조립 + 라우팅 함수
        ├── mcp_servers/
        │   └── openstack-mcp-server/
        │       ├── main.py            # MCP 서버 진입점 (stdio)
        │       ├── handlers/
        │       │   ├── compute.py     # handle_get_server_info, handle_create_vm (Mock)
        │       │   └── recovery.py    # handle_execute_recovery, handle_get_recovery_status (Mock)
        │       └── tools/
        │           ├── compute.py     # MCP 툴 정의 (get_server_info, create_vm)
        │           └── recovery.py    # MCP 툴 정의 (execute_recovery, get_recovery_status)
        ├── ws/
        │   └── chat.py                # WebSocket 엔드포인트, HITL 처리
        └── auth/
            ├── jwt_verify.py          # JWT 검증 + Redis 세션 확인
            └── schema.py              # TokenPayload (Keystone user_id, project_id 등 포함)
```

#### 파일별 상태

| 파일 | 역할 | 상태 |
|------|------|------|
| `main.py` | lifespan에서 MCP 초기화 + AsyncRedisSaver + Agent 빌드 | ✅ 구현 완료 |
| `app/auth/jwt_verify.py` | JWT 검증 (iss/aud/exp/jti/session 5단계) | ⚠️ Redis 키 프리픽스 수정 필요 |
| `app/auth/schema.py` | `TokenPayload` Pydantic 모델 (`sub` = Keystone user_id) | ✅ 구현 완료 |
| `app/common/redis.py` | FastAPI용 Redis 비동기 클라이언트 (`decode_responses=True`) | ✅ 구현 완료 |
| `app/common/config.py` | 환경변수 설정 | ✅ 구현 완료 |
| `app/graph_agent/state.py` | `ChatState(MessagesState)` 정의 | ✅ 구현 완료 |
| `app/graph_agent/schemas.py` | `RouteDecision`, `RecoveryPolicy` Pydantic 모델 | ✅ 구현 완료 |
| `app/graph_agent/agents.py` | LLM 초기화, mcp_tools 딕셔너리, answer_generator | ✅ 구현 완료 |
| `app/graph_agent/nodes.py` | 모든 그래프 노드 함수 | ✅ 구현 완료 |
| `app/graph_agent/graph.py` | StateGraph 조립 + 조건부 엣지 라우팅 | ✅ 구현 완료 |
| `app/ws/chat.py` | WebSocket 채팅: 인증·히스토리 복원·HITL·메시지 루프 | ✅ 구현 완료 |
| `app/mcp_servers/openstack-mcp-server/` | OpenStack MCP Server (stdio, Mock 핸들러) | ⚠️ Mock 상태 |

---

### graph_agent 플로우

```
START → intent_router
  ├── recover_server + server_id → get_server_info
  └── direct_response 또는 server_id 없음 → response

get_server_info
  ├── 성공 → generate_policy
  └── error → error_handler

generate_policy
  ├── 성공 → review_policy
  └── error → error_handler

review_policy (HITL)
  ├── 승인 → execute_recovery
  ├── 거절 + retry < 3 → generate_policy
  └── 거절 + retry ≥ 3 또는 error → error_handler

execute_recovery
  ├── 성공 → generate_report
  └── error → error_handler

generate_report
  ├── 성공 → END
  └── error → error_handler

response → END
error_handler → END
```

#### 노드별 상태

| 노드 | 상태 | 비고 |
|------|------|------|
| intent_router | ✅ 완료 | 마지막 HumanMessage만 사용 (전체 이력 전달 시 qwen이 function-calling 포맷 오응답) |
| get_server_info | ✅ 완료 | MCP 응답 list 파싱 처리 |
| generate_policy | ✅ 완료 | 거절 이유 반영, 재시도 카운트 |
| review_policy | ✅ 완료 | interrupt() HITL, 승인/거절/이유 처리 |
| execute_recovery | ✅ 완료 | create_vm MCP 호출, list 파싱 처리 |
| generate_report | ✅ 완료 | 문자열 조합, LLM 없음 |
| response | ✅ 완료 | response_agent (create_react_agent) 위임 |
| error_handler | ✅ 완료 | retry_count 기반 메시지 분기 |

#### MCP 툴 상태

| 툴 | 상태 | 비고 |
|----|------|------|
| get_server_info | ⚠️ Mock | 실제 OpenStack SDK 연동 필요 |
| create_vm | ⚠️ Mock | 실제 OpenStack SDK 연동 필요 (userdata ZConverter 설치 포함) |
| execute_recovery | ⚠️ Mock | ZConverter API 연동 필요 |
| get_recovery_status | ⚠️ Mock | ZConverter API 연동 필요 |

**Slack 허용 툴 (4개):**
```
slack_post_message       — 채널에 메시지 전송
slack_reply_to_thread    — 스레드에 답글
slack_add_reaction       — 메시지에 이모지 반응 추가
slack_get_channel_history — 채널 히스토리 조회
```

---

### horizon/ (Horizon 프론트엔드)

| 파일 | 역할 | 상태 |
|------|------|------|
| `static/dashboard/chat/chat.js` | JWT 갱신(50초) | ✅ 구현 완료 |
| `dashboards/project/overview/templates/overview/usage.html` | 채팅 UI + WebSocket + 히스토리 복원 + HITL 처리 | ✅ 구현 완료 |

**구현된 내용:**
- 페이지 로드 시 JWT 발급 (`POST /dashboard/ai/session/issue/`)
- 50초마다 JWT 자동 갱신, 탭 포커스 복귀 시 즉시 갱신
- `ws://192.168.88.4:8000/ws/chat?token={aiJwt}`로 WebSocket 연결
- 연결 상태 표시 (connected / disconnected / connecting)
- 메시지 송수신, 마크다운 렌더링, 로딩 애니메이션
- `type:"history"` 수신 시 이전 대화 복원 렌더링
- `type:"policy_review"` 수신 시 Claude-web 스타일 정책 카드 UI 표시
  - 정책 필드 key-value 표시, 거절 이유 입력 textarea (확장형)
  - 승인/거절 버튼 → `{type: "confirm_response", approved, reason}` 전송

---

## WebSocket 메시지 포맷

### 서버 → 클라이언트

```json
// 정책 검토 요청 (HITL)
{"type": "policy_review", "policy": {...}, "server_info": {...}, "message": "복구 정책을 검토하고 승인/거절해주세요."}

// 대화 이력 (새로고침 시)
{"type": "history", "messages": [{"role": "user/assistant", "content": "..."}]}

// 일반 텍스트 응답 (plain string)
"복구가 완료되었습니다..."
```

### 클라이언트 → 서버

```json
// 일반 메시지
{"content": "a1b2c3d4-0002 서버 복구 진행해줘"}

// 정책 승인
{"type": "confirm_response", "approved": true, "reason": ""}

// 정책 거절
{"type": "confirm_response", "approved": false, "reason": "flavor가 너무 작습니다"}
```

---

## 설계 변경 이력

### Flat Single Agent → StateGraph (graph_agent) 전환

**변경 전:**
- `create_react_agent` 단일 에이전트 패턴
- 모든 툴을 단일 ReAct 루프에서 처리
- HITL: 파괴적 툴 래퍼(`wrap_destructive_tools`)로 interrupt() 적용

**변경 후:**
- LangGraph StateGraph (Supervisor 패턴 X, 그래프가 라우팅 담당)
- intent_router → 복구 플로우 / 일반 응답 플로우 분기
- HITL: `node_review_policy`에서 `interrupt()` 사용, 정책 승인/거절 처리
- 정책 생성 실패 시 재시도 (최대 3회), 거절 이유를 다음 생성에 반영

**변경 이유:**
- 복구 플로우가 단순 툴 호출이 아닌 상태 기반 다단계 워크플로우
- 정책 검토(HITL), 재시도 로직, 에러 핸들러를 명시적 노드로 분리
- qwen2.5:7b 소형 모델에서 단일 ReAct로 복잡한 DR 플로우 처리 불안정

### intent_router 컨텍스트 제한

- 전체 대화 이력 대신 마지막 HumanMessage만 intent_llm에 전달
- 이유: ToolMessage/AIMessage 이력 포함 시 qwen이 function-calling JSON 포맷으로 오응답

### MCP 클라이언트 생명주기

- `langchain-mcp-adapters 0.1.0`부터 `async with MultiServerMCPClient(...)` 미지원
- `MultiServerMCPClient(...).get_tools()`로 단순 호출, 프로세스 종료는 Docker가 담당
- Slack + OpenStack MCP를 각각 별도 클라이언트로 관리

### MCP 툴 응답 파싱

- `langchain-mcp-adapters`가 응답을 `[{'type': 'text', 'text': '...json...'}]` 형태로 래핑
- `nodes.py`의 각 노드에서 list/str/dict를 모두 처리하도록 방어 코드 추가

### system_prompt 언어

- INTENT_SYSTEM, POLICY_SYSTEM: 영어 (JSON 구조화 출력 안정성)
- RESPONSE_SYSTEM: 영어 + "Always respond in Korean" 명시

---

## 알려진 버그 / 수정 필요

| 항목 | 파일 | 내용 |
|------|------|------|
| Redis 세션 키 프리픽스 누락 | `app/auth/jwt_verify.py` | `redis.exists(session_id)` → `redis.exists(f"chat:session:{session_id}")` 로 수정 필요 |

---

## Redis 키 사용 현황

| 키 패턴 | 타입 | 용도 | 구현 |
|---------|------|------|------|
| `chat:session:{session_id}` | String | 세션 메타데이터 (JSON, keystone_token 포함) | ✅ 구현 완료 |
| `chat:jti:{jti}` | String | Replay 방지 | ✅ 구현 완료 |
| `checkpoint:*` | Hash/Index | LangGraph 체크포인트 (AsyncRedisSaver) | ✅ 구현 완료 |
| `chat:audit:queue` | List | 감사 로그 큐 | ❌ 미구현 |

---

## 환경 변수

```
# OpenStack
OS_AUTH_URL
OS_USERNAME
OS_PASSWORD
OS_PROJECT_NAME

# Redis
REDIS_HOST
REDIS_PORT

# Slack MCP
SLACK_BOT_TOKEN
SLACK_TEAM_ID

# JWT
JWT_ALGORITHM
JWT_AUDIENCE
JWT_ISSUER
```

---

## 다음 구현 우선순위

### Phase 2 — OpenStack SDK 연동
1. `get_server_info` 실제 SDK 구현
   ```python
   conn.compute.find_server(server_id)
   conn.compute.get_server(server.id)
   conn.image.get_image(server.image.id)
   conn.network.ports(device_id=server.id)
   ```
2. `create_vm` 실제 SDK 구현 (userdata ZConverter 자동 설치 포함)
3. OpenStack 인증: 환경변수(`OS_AUTH_URL` 등) 또는 Redis 세션의 `keystone_token` 활용
4. 동기 SDK → `asyncio.to_thread()`로 비동기 처리

### Phase 3 — ZConverter 연동
5. `execute_recovery` ZConverter API 연동
6. `get_recovery_status` 상태 폴링

### Phase 4 — 플로우 고도화
7. `find_server` 복수 결과 시 `interrupt()`로 사용자 선택 지원
8. keystone_token 기반 인증 (Redis 세션 활용)
9. `jwt_verify.py` Redis 키 프리픽스 수정 (`chat:session:{session_id}`)
10. ZConverter 백업 이미지 자동 검색 (DB 또는 API 연동)

### Phase 5 — 고도화
11. RAG 기반 과거 복구 이력 참조
12. `direct_response` 플로우 고도화
13. 감사 로그 `chat:audit:queue` push + Celery drain
14. DB volume 마운트 추가 (`docker compose down` 후에도 체크포인트 유지)
