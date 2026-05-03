# 아키텍처: ZIASTACK DR 복구 자동화

> 구현 진행 현황은 [STATUS.md](STATUS.md) 참조.
> RAG 설계 상세는 [RAG.md](RAG.md) 참조.

## 컴포넌트 구조

```
┌─────────────────────────────────────────────────────────┐
│  운영자 (Browser)                                         │
└────────────────┬────────────────────────────────────────┘
                 │ WebSocket / REST
┌────────────────▼────────────────────────────────────────┐
│  Django · Horizon  [Plan / Confirm UI]                   │
│  - 복구 요청 접수 (서버 이름 입력)                          │
│  - AI 정책 표시 + 승인/거절 (거절 사유 필수)                │
│  - 실시간 상태 스트리밍 (WebSocket)                         │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│  AI Agent  [Claude API + Tool Calling]                   │
│  - Plan: 리소스 수집 → RAG 검색 → 복구 정책 생성            │
│  - Execute: MCP Tool 호출로 VM 생성 및 복구 실행            │
│  - Report: 결과 리포트 자동 생성                            │
│  - Failure: 로그 분석 → 원인 추정 → 재시도                  │
├─────────────────────────┬───────────────────────────────┤
│  Chroma (Vector DB)      │  도구 목록 (MCP Tools)         │
│  - 런북 / 매뉴얼 (TBD)   │  - vm_query(name)             │
│  - 복구 성공 이력        │  - vm_create(spec)            │
│  - 장애 대응 지식        │  - recovery_execute(policy)   │
└─────────────────────────┴──────────┬────────────────────┘
                                     │ Streamable HTTP
┌────────────────────────────────────▼────────────────────┐
│  MCP Server  [Tool Gateway]                              │
│  - AI Agent → OpenStack 사이의 유일한 진입점               │
│  - Tool 인터페이스 정의 및 라우팅                           │
└────────────────────────────────────┬────────────────────┘
                                     │ HTTP + Pre-shared Key
┌────────────────────────────────────▼────────────────────┐
│  FastAPI  [Internal Adapter]                             │
│  - Pre-shared Key 헤더 인증 (Guard Clause)                │
│  - MCP 요청 → OpenStack SDK 호출 변환                      │
│  - Redis에 작업 상태 저장 / Lock 관리                       │
│  - Kafka에 Execute 이벤트 발행                             │
│  - Slack MCP / Jira MCP 알림 트리거                        │
└──────────────┬──────────────────┬──────────────────┬────┘
               │ SDK              │                  │
┌──────────────▼───┐  ┌───────────▼──┐  ┌────────────▼──┐
│  OpenStack        │  │  Redis        │  │  Kafka        │
│  Nova (VM)        │  │  State · Lock │  │  Execute      │
│  Neutron (Net)    │  │  TTL          │  │  Events       │
│  Cinder (Volume)  │  └──────────────┘  └───────────────┘
│  Glance (Image)   │
└──────────────┬───┘
               │ Userdata
┌──────────────▼────────────────────────────────────────┐
│  ZConverter Cloud AI 에이전트 (VM 내부)                  │
│  - VM 내부 설치 후 복구 작업 자동 진행                      │
│  - 세부 동작 TBD                                         │
└───────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│  DB (PostgreSQL)          Slack MCP       Jira MCP     │
│  - 복구 이력 영구 저장      - 완료/실패 알림  - 티켓 자동 생성 │
└────────────────────────────────────────────────────────┘
```

## 디렉토리 구조 (실제)

```
agent_server/
├── agent_server/
│   ├── main.py                         # FastAPI 앱, lifespan (MCP 클라이언트 + Agent 초기화)
│   └── app/
│       ├── agent/
│       │   ├── agent.py                # ✅ LangGraph ReAct 에이전트 (현재 qwen2.5, 목표 Claude API)
│       │   └── tools.py                # ✅ 파괴적 도구 interrupt 래핑 (create_vm, execute_recovery)
│       ├── auth/
│       │   ├── jwt_verify.py           # ✅ RS256 JWT 검증 + Redis jti replay 방지
│       │   └── schema.py               # ✅ TokenPayload 모델
│       ├── common/
│       │   ├── config.py               # ✅ pydantic-settings 기반 환경 변수
│       │   └── redis.py                # ✅ Redis 비동기 클라이언트
│       ├── ws/
│       │   └── chat.py                 # ✅ WebSocket 엔드포인트 (이력 복원, interrupt 재개)
│       ├── knowledge/
│       │   └── runbooks/               # ❌ 런북 파일 없음 (추가 필요)
│       ├── rag/                        # ❌ 미구현 (RAG.md 참조)
│       │   ├── embeddings.py           # 다국어 임베딩 모델
│       │   ├── vectorstore.py          # ChromaDB 컬렉션 관리
│       │   ├── ingest.py               # 런북 + 이력 인제스트
│       │   ├── retriever.py            # MMR 검색 래퍼
│       │   └── policy_chain.py         # 복구 정책 생성 LCEL 체인
│       └── mcp_servers/
│           └── openstack-mcp-server/
│               ├── main.py             # ✅ stdio MCP 서버
│               ├── tools/              # ✅ Tool 스키마 정의
│               │   ├── compute.py      # get_server_info, create_vm
│               │   └── recovery.py     # execute_recovery, get_recovery_status
│               └── handlers/           # ⚠️ Mock 구현 (실제 SDK 연동 필요)
│                   ├── compute.py
│                   └── recovery.py
└── docs/
    ├── PRD.md
    ├── ARCHITECTURE.md     # 이 파일
    ├── REQUIREMENTS.md
    ├── ADR.md
    ├── USECASES.md
    ├── UI_GUIDE.md
    ├── STATUS.md           # ✅ 구현 현황 추적
    └── RAG.md              # ✅ RAG 설계서
```

## 데이터 흐름

### Plan Phase
```
운영자 입력(서버 이름)
  → Django → AI Agent
  → Nova API: 서버 이름으로 VM 조회 (타겟 존재 여부 판단)
  → Nova API: 소스 서버 리소스 수집 (CPU/Memory/Disk/Network)
  → Chroma RAG: 관련 런북 검색
  → Claude API: 복구 정책 생성
  → Django UI에 정책 표시
```

### Confirm Phase
```
운영자 승인 → Execute Phase로 진행
운영자 거절 (사유 필수) → AI Agent 정책 재생성 (최대 3회)
3회 초과 → blocked 상태 전환
```

### Execute Phase
```
승인된 정책
  → AI Agent → MCP Server → FastAPI (Pre-shared Key 인증)
  → Redis Lock 획득 (중복 실행 방지)
  → [타겟 없는 경우] Nova API: VM 생성
  → [타겟 없는 경우] Userdata: ZConverter AI 에이전트 설치
  → 복구 실행 (ZConverter AI 에이전트가 VM 내 처리)
  → Kafka: Execute 이벤트 발행
  → WebSocket: 실시간 상태 스트리밍
```

### Report Phase
```
복구 완료/실패
  → Claude API: 리포트 자동 생성
  → DB: 이력 저장
  → Chroma: 정책/장애 지식 누적
  → Slack MCP / Jira MCP: 알림 발송
```

## 상태 관리

- **작업 상태** (Redis): `pending → planning → confirming → executing → completed / failed / blocked / cancelled`
- **분산 Lock** (Redis): 동일 작업 ID 중복 실행 방지, TTL 설정
- **이벤트** (Kafka): Execute Phase 진행 이벤트 — UI WebSocket 스트리밍용
