# RAG 설계: LangChain 기반 복구 정책 생성

> FR-05, FR-06 구현 설계서

---

## 목표

AI 에이전트가 OpenStack 서버 정보를 수집한 후, 사내 런북과 과거 복구 이력을 RAG로 검색해 구조화된 복구 정책을 자동 생성한다.

---

## 아키텍처

```
서버 이름 입력
  ↓
[1] get_server_info (MCP Tool)
    → CPU / Memory / Disk / Network / 상태 수집
  ↓
[2] RAG 검색 (ChromaDB)
    Query: "{서버명} {상태} {장애 유형}"
    Collections:
      - runbooks: 런북 청크 (markdown)
      - recovery_history: 과거 복구 성공/실패 사례
  ↓
[3] Policy Chain (Claude API)
    Input: server_info + rag_context
    Output: RecoveryPolicy (구조화된 JSON)
  ↓
[4] UI 표시 → 엔지니어 승인/거절
```

---

## 디렉토리 구조

```
agent_server/app/
├── knowledge/
│   ├── runbooks/               # 원본 런북 파일
│   │   ├── vm_recovery.md      # VM 장애 복구 절차
│   │   ├── network_issues.md   # 네트워크 장애 대응
│   │   └── storage_recovery.md # 스토리지 복구
│   └── __init__.py
└── rag/
    ├── __init__.py
    ├── embeddings.py           # 임베딩 모델 설정
    ├── vectorstore.py          # ChromaDB 초기화 및 컬렉션 관리
    ├── ingest.py               # 런북 인제스트 파이프라인
    ├── retriever.py            # MMR 검색 래퍼
    └── policy_chain.py         # 복구 정책 생성 LangChain 체인
```

---

## 구성 요소

### 1. 임베딩 모델 (`rag/embeddings.py`)

한국어 + 영어 혼합 콘텐츠를 지원하는 다국어 임베딩 사용.

```python
from langchain_huggingface import HuggingFaceEmbeddings

def get_embeddings():
    return HuggingFaceEmbeddings(
        model_name="sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2",
        model_kwargs={"device": "cpu"},
        encode_kwargs={"normalize_embeddings": True},
    )
```

**선택 이유**: 무료, 온프레미스 실행 가능, 한/영 이중 언어 지원. 추후 OpenAI `text-embedding-3-small`로 교체 가능.

---

### 2. ChromaDB 설정 (`rag/vectorstore.py`)

```python
import chromadb
from langchain_chroma import Chroma
from .embeddings import get_embeddings

CHROMA_PATH = "./data/chroma"

def get_vectorstore(collection_name: str) -> Chroma:
    client = chromadb.PersistentClient(path=CHROMA_PATH)
    return Chroma(
        client=client,
        collection_name=collection_name,
        embedding_function=get_embeddings(),
    )

# 두 개의 컬렉션 사용
# - "runbooks": 런북/매뉴얼 청크
# - "recovery_history": 복구 성공/실패 이력
```

---

### 3. 인제스트 파이프라인 (`rag/ingest.py`)

```python
from pathlib import Path
from langchain_community.document_loaders import DirectoryLoader, UnstructuredMarkdownLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from .vectorstore import get_vectorstore

RUNBOOKS_PATH = Path(__file__).parent.parent / "knowledge" / "runbooks"

def ingest_runbooks():
    loader = DirectoryLoader(
        str(RUNBOOKS_PATH),
        glob="**/*.md",
        loader_cls=UnstructuredMarkdownLoader,
    )
    docs = loader.load()

    splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,
        chunk_overlap=50,
        separators=["\n## ", "\n### ", "\n\n", "\n", " "],
    )
    chunks = splitter.split_documents(docs)

    # 메타데이터 보강
    for chunk in chunks:
        chunk.metadata["source_type"] = "runbook"

    vectorstore = get_vectorstore("runbooks")
    vectorstore.add_documents(chunks)
    return len(chunks)

def ingest_recovery_case(policy: dict, outcome: str, server_info: dict):
    """복구 완료/실패 후 Chroma에 이력 누적 (FR-23, FR-26)"""
    content = f"""
서버: {server_info.get('name')} ({server_info.get('flavor')})
상태: {server_info.get('status')}
복구 유형: {policy.get('recovery_type')}
실행 단계: {'; '.join(policy.get('execution_steps', []))}
결과: {outcome}
"""
    vectorstore = get_vectorstore("recovery_history")
    vectorstore.add_texts(
        [content],
        metadatas=[{"outcome": outcome, "server_name": server_info.get("name")}],
    )
```

---

### 4. 검색 래퍼 (`rag/retriever.py`)

```python
from .vectorstore import get_vectorstore

def get_policy_retriever():
    """
    두 컬렉션(런북 + 이력)을 합산 검색.
    MMR(Maximal Marginal Relevance)로 다양한 결과 확보.
    """
    runbook_store = get_vectorstore("runbooks")
    history_store = get_vectorstore("recovery_history")

    runbook_retriever = runbook_store.as_retriever(
        search_type="mmr",
        search_kwargs={"k": 3, "fetch_k": 15},
    )
    history_retriever = history_store.as_retriever(
        search_type="mmr",
        search_kwargs={"k": 2, "fetch_k": 10},
    )

    return runbook_retriever, history_retriever
```

---

### 5. 복구 정책 생성 체인 (`rag/policy_chain.py`)

핵심 컴포넌트. LCEL로 구성.

```python
import json
from pydantic import BaseModel
from typing import Optional
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.runnables import RunnableLambda
from .retriever import get_policy_retriever

class RecoveryPolicy(BaseModel):
    backup_point: str           # 백업 시점 (예: "2026-04-28 03:00 KST")
    recovery_type: str          # "full" | "incremental"
    execution_steps: list[str]  # 순서대로 나열된 실행 단계
    estimated_time: str         # 예상 소요 시간 (예: "약 25분")
    create_new_vm: bool         # 신규 VM 생성 필요 여부
    vm_spec: Optional[dict]     # create_new_vm=True 일 때만 포함
    rag_sources: list[str]      # 참조한 런북/이력 출처
    confidence: str             # "high" | "medium" | "low"
    warnings: list[str]         # 주의사항

POLICY_PROMPT = ChatPromptTemplate.from_messages([
    ("system", """당신은 OpenStack 인프라 재해복구 전문가입니다.
서버 정보와 참조 런북을 바탕으로 최적의 복구 정책을 생성합니다.

[참조 런북 및 복구 이력]
{context}

[출력 형식] 다음 JSON 스키마를 반드시 준수하세요:
{{
  "backup_point": "가장 안전한 최신 백업 시점 (ISO 8601)",
  "recovery_type": "full 또는 incremental",
  "execution_steps": ["1단계", "2단계", ...],
  "estimated_time": "예상 소요 시간 (한국어)",
  "create_new_vm": true 또는 false,
  "vm_spec": {{"flavor": "...", "image_id": "...", "network_id": "..."}} 또는 null,
  "rag_sources": ["참조 런북/이력 이름"],
  "confidence": "high(런북 다수 매칭) / medium(일부 매칭) / low(매칭 없음)",
  "warnings": ["주의사항 1", ...]
}}

- 런북 매칭이 없으면 confidence를 "low"로 설정하고 warnings에 명시하세요.
- 반드시 유효한 JSON만 출력하세요. 설명 텍스트 없이."""),
    ("human", "서버 정보:\n{server_info}"),
])

def build_policy_chain():
    runbook_retriever, history_retriever = get_policy_retriever()
    llm = ChatAnthropic(model="claude-opus-4-7", temperature=0)

    def retrieve_context(inputs: dict) -> str:
        query = inputs["query"]
        runbook_docs = runbook_retriever.invoke(query)
        history_docs = history_retriever.invoke(query)
        all_docs = runbook_docs + history_docs

        if not all_docs:
            return "[참조 런북 없음 — RAG 결과 없음]"

        return "\n\n---\n\n".join(
            f"[출처: {doc.metadata.get('source', '알 수 없음')}]\n{doc.page_content}"
            for doc in all_docs
        )

    chain = (
        RunnableLambda(lambda x: {
            "context": retrieve_context(x),
            "server_info": json.dumps(x["server_info"], ensure_ascii=False, indent=2),
        })
        | POLICY_PROMPT
        | llm
        | JsonOutputParser()
    )

    return chain

async def generate_policy(server_info: dict) -> RecoveryPolicy:
    """
    server_info: get_server_info MCP 도구 응답 딕셔너리
    Returns: RecoveryPolicy (validated Pydantic model)
    """
    chain = build_policy_chain()
    query = f"{server_info.get('name', '')} {server_info.get('status', '')} recovery"

    raw = await chain.ainvoke({
        "query": query,
        "server_info": server_info,
    })

    return RecoveryPolicy(**raw)
```

---

## 에이전트 통합 전략

### 현재 구조와의 통합 방법

현재 에이전트는 범용 ReAct 채팅 에이전트다. RAG 정책 생성을 통합하는 가장 낮은 마찰의 방법:

**`generate_recovery_policy` LangChain 도구 추가** — MCP가 아닌 Python 함수를 LangChain Tool로 래핑.

```python
# app/agent/agent.py 에 추가
from langchain_core.tools import tool
from app.rag.policy_chain import generate_policy
import json

@tool
async def generate_recovery_policy(server_info_json: str) -> str:
    """
    OpenStack 서버 정보를 받아 런북 RAG 검색 후 복구 정책을 생성합니다.
    get_server_info 도구 호출 결과(JSON 문자열)를 그대로 입력하세요.
    """
    server_info = json.loads(server_info_json)
    policy = await generate_policy(server_info)
    return json.dumps(policy.model_dump(), ensure_ascii=False, indent=2)
```

에이전트 프롬프트에 호출 순서 명시:
```
Plan Phase 순서:
1. get_server_info 로 서버 현황 수집
2. generate_recovery_policy 로 RAG 기반 복구 정책 생성
3. 정책을 사용자에게 표시하고 승인 대기
4. 승인 후 execute_recovery 실행
```

### 중장기: LangGraph 재구조화

RAG 구현 후 다음 단계로 LangGraph Graph 구조로 전환:

```
[plan_node]      → get_server_info + generate_recovery_policy → 정책 생성
[confirm_node]   → interrupt() → 엔지니어 승인/거절 루프 (최대 3회)
[execute_node]   → create_vm (선택) + execute_recovery → Kafka 이벤트
[report_node]    → 결과 리포트 생성 + Chroma 이력 저장 + Slack 알림
```

---

## 런북 작성 가이드

`app/knowledge/runbooks/` 에 추가할 마크다운 파일 구조:

```markdown
# VM 재부팅 복구 절차

## 적용 조건
- VM 상태: ERROR 또는 SHUTOFF
- 복구 유형: reboot

## 복구 단계
1. VM 상태 확인 (Nova API)
2. 강제 재부팅 실행 (nova reboot --hard)
3. 부팅 완료 확인 (ACTIVE 상태 대기)
4. 네트워크 연결 검증

## 예상 소요 시간
약 5~10분

## 주의사항
- 메모리 내 작업 데이터 손실 가능
- 복구 불가 시 rebuild 절차로 전환
```

---

## 의존성 추가 필요

```
langchain-anthropic
langchain-chroma
langchain-huggingface
chromadb
sentence-transformers
unstructured
```

---

## RAG 품질 개선 루프

복구 완료/실패 후 `ingest_recovery_case()` 를 Report Phase에서 자동 호출.

```
복구 성공 → ingest_recovery_case(policy, "success", server_info)  # FR-23
복구 실패 → ingest_recovery_case(policy, "failed", server_info)   # FR-26
```

시간이 지날수록 이력 데이터가 쌓여 유사한 장애 상황에서 RAG 정확도가 향상됨.
