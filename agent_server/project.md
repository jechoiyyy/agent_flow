프로젝트 개요
ZIASTACK(OpenStack 기반) 환경에서 ZConverter 백업 이미지 기반 VM 자동 생성 및 복구를 수행하는 LLM 에이전트 시스템.
기술 스택

LangChain create_agent + LangGraph StateGraph
LLM: qwen2.5:7b (Ollama)
MCP 서버 (stdio transport)
OpenStack SDK
ZConverter 에이전트 (cloud-init/userdata 자동 설치)


1차 구현 목표 (Flowchart)
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

핵심 설계 원칙

그래프가 라우팅 담당 (Supervisor 패턴 X) → 컨텍스트 효율성
에이전트 사전 생성 (모듈 로드 시 1회) → 재사용
각 에이전트 독립 컨텍스트 (필요한 messages만 전달)
순서는 그래프 엣지로 강제 (system_prompt 의존 X)
HITL은 LangChain HumanInTheLoopMiddleware 활용
정책은 Pydantic RecoveryPolicy로 구조화 (qwen2.5:7b 안정성)
챗봇 연속성은 thread_id + AsyncRedisSaver


노드별 역할
intent_router

RouteDecision Pydantic으로 intent 분류 (recover_server / direct_response)
server_id 추출 (UUID 또는 이름 모두 허용)

get_server_info

conn.compute.find_server() 로 검색 (UUID/이름/부분이름)
복수 결과 시 interrupt()로 검색된 서버 중 사용자 선택
conn.compute.get_server() + 메타데이터 + flavor + image + volumes 수집

generate_policy

소스 서버 OS와 동일 major OS Glance 이미지 검색
ZConverter 최근 백업 이미지 매칭 (우선 사용자 openstack에 있는 이미지 매칭으로 테스트)
LLM이 RecoveryPolicy JSON 생성 (거절 이유 반영, 목업 Policy로 테스트 진행, 흐름 먼저 설계)

review_policy (HITL)

approve_policy_tool + HumanInTheLoopMiddleware
limit_policy_retries 미들웨어로 3회 한도
거절 시 reject_reason 저장 → 재생성 루프

execute_recovery

OpenStack SDK로 타겟 VM 생성
userdata로 ZConverter 에이전트 자동 설치
VM ACTIVE 대기 → ZConverter 복구 작업 시작
타겟 VM 생성 실패시 cleanup 로직 구현

generate_report

복구 결과 리포트 + 이력 저장

response (direct_response 플로우)

일반 질문/조회 처리
create_vm 단독 호출 시 HITL + limit_vm_retries

error_handler

state["error"] + retry_count 기반 메시지 분기
3회 거절 시 "ZConverter 문의 필요" 안내


프로젝트 구조
project/
├── mcp_server/
│   ├── server.py        # MCP 서버 (stdio)
│   ├── handlers.py      # handle_get_server_info, handle_create_vm 등
│   └── tools.py         # ALL_TOOLS 정의
│
└── agent/
    ├── main.py          # chat_loop + interrupt 처리
    ├── graph.py         # StateGraph 조립 + 라우팅 함수
    ├── state.py         # ChatState
    ├── schemas.py       # RouteDecision, RecoveryPolicy
    ├── nodes.py         # 각 노드 함수
    ├── agents.py        # 사전 생성 에이전트 + MCP 연결
    ├── tools_local.py   # approve_policy_tool, generate_report_tool, save_history_tool
    └── middleware.py    # limit_policy_retries, limit_vm_retries

mcp server는 이미 구현되어 있음
agent 폴더는 app/graph_agent에서 이어서 구현

State 정의
pythonclass ChatState(MessagesState):
    # 라우팅
    intent:    str | None
    server_id: str | None

    # 복구 플로우
    server_info:     dict | None     # name, flavor, image, address, volumes, metadata, status
    recovery_policy: dict | None     # name, flavor, image_id, network_id, backup_id, user_data
    vm_info:         dict | None
    report:          str | None

    # 거절 관리
    retry_count:   int
    reject_reason: str | None

    # 에러
    error: str | None

Pydantic 스키마
pythonclass RouteDecision(BaseModel):
    intent:    Literal["recover_server", "direct_response"]
    server_id: str | None  # UUID 또는 이름 모두 허용

class RecoveryPolicy(BaseModel):
    name:          str
    flavor:        Literal["m1.tiny", "m1.small", "m1.medium", "m1.large", "m1.xlarge"]
    image_id:      str          # 동일 major OS Glance 이미지
    network_id:    str
    recovery_type: Literal["snapshot_restore", "fresh_install", "config_replicate"]
    reason:        str
    # 추후 확장: backup_id, backup_timestamp, user_data

MCP 툴 (mcp_server)

get_server_info(server_id) - find_server + get_server
    - get_server를 통해서 어떤 데이터들이 들어오는지 직접 확인해보고
    - get_server로 받은 객체를 server 변수에 받고,
    - image = conn.image.get_image(server.image.id)
    - network = conn.network.ports(device_id=SERVER_ID) 등의 추가 데이터 저장 작업 필요
create_vm(name, flavor, image_id, network_id, user_data) - VM 생성
execute_recovery(server_id, recovery_type, reason) - 복구 실행
get_recovery_status(job_id) - 복구 상태 조회


에이전트 구성
에이전트    모델    툴  미들웨어    HITL
intent_agent    qwen2.5:7b  -   -   -
policy_agent    qwen2.5:7b  -   response_format=RecoveryPolicy  -
policy_review_agent qwen2.5:7b  approve_policy_tool limit_policy_retries    ✅
vm_agent(복구 플로우)   qwen2.5:7b  create_vm   -   -
response_agent(direct)  qwen2.5:7b  get_server_info,get_recovery_status,create_vm limit_vm_retries create_vm만

핵심 결정사항

OpenStack 이미지/스냅샷의 역할

VM 생성용 (깨끗한 OS 이미지)
실제 데이터 복구는 ZConverter 백업이 담당
image_id는 소스와 동일 major OS Glance 이미지


VM 생성 시 필수 파라미터

name, flavor_id, image_id, networks
추가: user_data (ZConverter 에이전트 자동 설치)


파티션 정보

OpenStack SDK는 볼륨 단위까지만 제어
파티션은 이미지/스냅샷에 포함 또는 cloud-init에서 처리
ZConverter 백업이 OS+Data+App 전영역 복구 (우선 userdata로 처리하는 것 까지만 진행)


재시도 한도 분리

limit_policy_retries (정책 검토)


사용자 편의성

find_server로 UUID/이름/부분이름 모두 허용
복수 결과 시 interrupt()로 선택




1차 작업 우선순위
필수 구현

get_server_info - find_server 기반, 복수 결과 HITL
generate_policy - Glance 이미지 검색 + LLM 정책 생성 (RAG 미구현, 목업 데이터로 진행)
review_policy - HITL + 재시도 한도(3회)
execute_recovery - VM 생성 + userdata로 ZConverter 에이전트 설치
generate_report - 결과 리포트 (목업 데이터로 테스트 진행)

2차 고도화 (1차 완료 후)

ZConverter 백업 이미지 자동 검색 (DB 또는 API 연동)
복구 진행 상태 폴링 (get_recovery_status)
RAG 기반 과거 복구 이력 참조
타겟 서버 존재 여부 확인 분기 (예/아니오)
direct_response 플로우 고도화


환경 변수
OS_AUTH_URL
OS_USERNAME
OS_PASSWORD
OS_PROJECT_NAME

----
mock에 있는 tool도 node에 적용하기