# 구현 현황: ZIASTACK DR 복구 자동화

> 최종 업데이트: 2026-04-29

---

## 전체 진행률

| Phase | 상태 | 비고 |
|-------|------|------|
| 인프라 기반 (Auth, Redis, WebSocket) | ✅ 완료 | 프로덕션 수준 |
| MCP 서버 연동 (Slack, OpenStack) | ✅ 완료 | 핸들러는 Mock |
| 파괴적 도구 승인 루프 (Interrupt) | ✅ 완료 | LangGraph interrupt |
| RAG (Chroma + 런북) | ❌ 미구현 | 다음 구현 대상 |
| Plan/Confirm/Execute 상태 머신 | ⚠️ 부분 | 채팅 에이전트로 대체 중 |
| 실제 OpenStack SDK 연동 | ❌ 미구현 | 핸들러 전체 Mock |
| Kafka 이벤트 | ❌ 미구현 | |
| PostgreSQL 이력 저장 | ❌ 미구현 | |
| Report 자동 생성 | ❌ 미구현 | |
| Slack/Jira 알림 | ⚠️ 부분 | Slack MCP 연결만 됨 |

---

## 컴포넌트별 상세

### ✅ JWT 인증 (`app/auth/`)
- RS256 공개키 검증
- `iss`, `aud`, `exp` 클레임 확인
- `jti` Redis 기반 replay 방지
- `session_id` Redis 세션 존재 확인
- 완성도: 프로덕션 수준

### ✅ WebSocket 채팅 (`app/ws/chat.py`)
- JWT 토큰 쿼리 파라미터 인증
- 이전 대화 이력 복원 (Redis checkpointer)
- `confirm_response` 메시지 타입으로 Interrupt 재개
- 완성도: 기능 완성. Plan 특화 메시지 포맷 추가 필요

### ✅ LangGraph 에이전트 (`app/agent/agent.py`)
- ReAct 에이전트 (langgraph prebuilt)
- Redis 기반 Checkpointer (대화 영속성)
- **⚠️ 이슈**: LLM이 `qwen2.5:7b` (Ollama) 사용 중. ADR-001에서 Claude API 사용을 결정했으나 불일치.

### ✅ 파괴적 도구 승인 (`app/agent/tools.py`)
- `create_vm`, `execute_recovery` 실행 전 `interrupt()` 발동
- 사용자 `approved: true/false` 응답으로 재개/취소
- 완성도: 핵심 승인 루프 완성

### ✅ OpenStack MCP 서버 (`app/mcp_servers/openstack-mcp-server/`)
- 도구: `get_server_info`, `create_vm`, `execute_recovery`, `get_recovery_status`
- stdio 전송 방식 (개발 환경)
- **⚠️ 이슈**: 핸들러 전체 Mock. 실제 OpenStack SDK 미연동.
- **⚠️ 이슈**: `get_server_info`가 UUID만 받음. ADR-003은 서버 이름 기반 조회를 명시.

### ✅ Slack MCP 연동 (`app/agent/agent.py`)
- `@modelcontextprotocol/server-slack` 연결
- 허용 도구: `slack_post_message`, `slack_reply_to_thread`, `slack_add_reaction`, `slack_get_channel_history`
- 완성도: 연결 완료. 실제 알림 트리거 로직은 미구현.

### ❌ RAG (`app/knowledge/`, `app/rag/` 미존재)
- `app/knowledge/runbooks/__init__.py` 만 존재, 내용 없음
- ChromaDB 연동 없음
- 복구 정책 생성 시 런북/이력 검색 미구현
- **→ 다음 구현 대상**. 설계는 `RAG.md` 참조.

### ❌ Plan Phase 상태 머신
- 문서: `pending → planning → confirming → executing → completed/failed/blocked/cancelled`
- 현재: 단일 채팅 에이전트로 처리. 명시적 상태 전이 없음.
- **→ RAG 구현 후 LangGraph Graph로 재구성 예정.**

### ❌ FastAPI Internal Adapter
- 문서: MCP Server → FastAPI (Pre-shared Key) → OpenStack SDK
- 현재: MCP Server에 Mock 핸들러가 직접 포함됨. FastAPI 어댑터 레이어 없음.
- **→ OpenStack SDK 연동 시 분리 예정.**

### ❌ Kafka
- Execute Phase 이벤트 발행 미구현.

### ❌ PostgreSQL
- 복구 이력 영구 저장 미구현.

---

## 핵심 불일치 사항

| 항목 | 문서 | 현재 코드 | 조치 필요 |
|------|------|-----------|-----------|
| LLM | Claude API (ADR-001) | qwen2.5:7b via Ollama | 교체 필요 |
| 서버 조회 | 서버 이름 기반 (ADR-003) | UUID 기반 | 핸들러 수정 필요 |
| OpenStack 연동 | SDK 실제 호출 | 전체 Mock | SDK 연동 시 교체 |
| RAG | Chroma + 런북 | 미구현 | 신규 구현 필요 |
| 상태 머신 | 명시적 상태 전이 | 채팅 에이전트 | LangGraph 재구조화 |
