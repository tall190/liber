---
title: 'Agent 프레임워크 완전 비교: LangGraph, CrewAI, AutoGen, Claude SDK, Strands'
description: '2025~2026년 기준 주요 Agent 프레임워크 5개를 설계 철학부터 실제 사용 사례까지 비교해요. 어떤 상황에 어떤 프레임워크를 선택해야 하는지 기준을 정리했어요.'
pubDate: '2026-05-22'
heroImage: '../../assets/ai-agent-series-2-frameworks_hero_image.png'
tags: ['AI', 'Agent', 'LangGraph', 'CrewAI', 'Strands', '프레임워크']
series: 'AI Agent 완전정복'
---

Agent를 만들기로 결정하고 나면 곧 두 번째 질문이 생겨요. "어떤 프레임워크로 만들어야 할까?"

LangGraph, CrewAI, AutoGen, Claude Agent SDK, AWS Strands — 다들 "가장 쉽고 강력하다"고 주장해요. 직접 써보기 전에는 판단하기 어렵고, 잘못 선택하면 나중에 마이그레이션 비용이 만만치 않죠.

이 글에서는 2025~2026년 기준 주요 Agent 프레임워크 5개를 비교해볼게요. 각 프레임워크의 설계 철학, 핵심 기능, 장단점, 그리고 어떤 상황에 적합한지를 정리했어요.

## 5개 프레임워크 한눈에 보기

| 프레임워크 | 제작사 | 설계 철학 | GitHub Stars |
|-----------|--------|----------|-------------|
| LangGraph | LangChain | 그래프 기반 상태 머신 | 12k+ |
| CrewAI | CrewAI Inc. | 역할 기반 팀 협업 | 30k+ |
| AutoGen → MAF | Microsoft | 대화 기반 멀티에이전트 | 40k+ |
| Claude Agent SDK | Anthropic | 코드 기반 오케스트레이션 | — |
| Strands Agents | AWS | 모델 주도 최소 설정 | 6k+ |

이제 각각을 자세히 살펴볼게요.

---

## 1. LangGraph

### 설계 철학: 명시적인 그래프로 흐름을 제어한다

LangGraph는 Agent 워크플로우를 **노드(실행 단위)와 엣지(전이 조건)로 구성된 그래프**로 표현해요. "Build resilient agents"가 공식 슬로건인 만큼, 복잡한 분기와 루프를 정확하게 제어하는 게 핵심이에요.

2025년 10월 v1.0 GA 출시 이후 Uber, LinkedIn, Klarna 같은 대형 프로덕션 환경에서 사용 중이에요.

### 핵심 기능

**Durable Execution**: 체크포인팅으로 모든 상태 전이를 영속화해요. 실패하면 중단된 지점부터 재개할 수 있어요.

**Human-in-the-loop**: `interrupt` 메커니즘으로 그래프 실행을 일시정지하고, 사람의 입력을 받은 뒤 재개할 수 있어요. 스레드를 블로킹하지 않아요.

**Time-travel 디버깅**: 체크포인트 기반으로 과거 상태로 되돌려 디버깅할 수 있어요.

```python
from langgraph.graph import StateGraph
from langgraph.checkpoint.memory import MemorySaver

def call_llm(state): ...
def use_tool(state): ...
def should_use_tool(state): ...

graph = StateGraph(AgentState)
graph.add_node("llm", call_llm)
graph.add_node("tool", use_tool)
graph.add_conditional_edges("llm", should_use_tool)
app = graph.compile(checkpointer=MemorySaver())
```

### 장단점

**장점**
- 복잡한 조건 분기와 루프를 그래프로 명시적 표현
- Human-in-the-loop이 프레임워크 수준에서 기본 지원
- 실패 복구와 재시도가 내장 (v1.1부터 지수 백오프 retry middleware 포함)
- Uber, LinkedIn, Klarna 등 대규모 프로덕션 검증

**단점**
- 그래프 DSL 학습 곡선이 있어요
- 단순한 선형 워크플로우에는 오버엔지니어링이 될 수 있어요

**적합한 상황**: 복잡한 조건 분기, Human approval이 필요한 프로덕션 워크플로우, 장시간 실행 태스크, RAG + Agent 결합 아키텍처

---

## 2. CrewAI

### 설계 철학: AI 에이전트를 팀원처럼 구성한다

CrewAI는 각 에이전트에 **역할(Role), 목표(Goal), 배경(Backstory)**을 부여해요. "리서치 담당", "작성 담당", "검토 담당"처럼 팀 구성을 모델링하는 방식이에요.

LangChain에 의존하지 않는 순수 Python 프레임워크고, 2025년 10월 v1.1.0 기준으로 GitHub Stars 30k+를 기록했어요.

### 핵심 구조

- **Crew**: 에이전트 집합 + 실행 프로세스
- **Agent**: 역할, 목표, 도구를 가진 AI 행위자
- **Task**: 구체적인 작업 단위 (기대 출력 포함)
- **Process**: Sequential / Hierarchical 중 선택

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(
    role="Senior Researcher",
    goal="Find key facts about {topic}",
    backstory="You're an expert researcher with 10 years of experience.",
    tools=[search_tool]
)

writer = Agent(
    role="Content Writer",
    goal="Write a clear summary based on research",
    backstory="You simplify complex topics for engineers."
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential
)
result = crew.kickoff(inputs={"topic": "AI Agents"})
```

### 장단점

**장점**
- 역할 기반 설계로 팀 시뮬레이션에 직관적
- 빠른 프로토타이핑에 유리
- LangChain 비의존으로 의존성 경량

**단점**
- 복잡한 상태 관리는 LangGraph 대비 표현력이 낮아요
- Hierarchical 프로세스에서 매니저 LLM 비용이 추가 발생해요
- 에이전트 행동 예측성이 낮아 프로덕션 디버깅이 어려울 수 있어요

**적합한 상황**: 리서치 → 작성 → 검토 파이프라인, 콘텐츠 생성 자동화, 빠른 PoC

---

## 3. AutoGen → Microsoft Agent Framework (MAF)

### 중요한 맥락: AutoGen은 이제 유지보수 모드예요

AutoGen은 Microsoft가 만든 대화 기반 멀티에이전트 프레임워크예요. 그런데 2026년 4월, Microsoft는 AutoGen과 Semantic Kernel을 합쳐 **Microsoft Agent Framework(MAF) 1.0을 GA로 출시**했어요. 이후 AutoGen은 버그 수정과 보안 패치만 받고, 신규 기능 개발은 중단됐어요.

새 프로젝트라면 MAF를 선택하는 게 맞아요.

### MAF의 핵심

MAF는 AutoGen의 단순한 에이전트 추상화에 Semantic Kernel의 엔터프라이즈 기능을 결합했어요.

- 세션 기반 상태 관리와 타입 안전성 강화
- OpenTelemetry observability 내장
- Python + .NET 크로스 언어 지원
- MCP 프로토콜 기본 내장
- 5가지 오케스트레이션 패턴 제공

**적합한 상황**: 기존 Microsoft 생태계(Azure, .NET) 팀, 엔터프라이즈 환경, 코드 생성 에이전트

---

## 4. Claude Agent SDK (Anthropic)

### 설계 철학: 코드로 오케스트레이션을 표현한다

Claude Agent SDK는 Claude Code의 내부 인프라를 라이브러리로 공개한 형태예요. 복잡한 워크플로우 DSL 대신 **Python 코드로 도구 호출과 에이전트 간 흐름을 직접 표현**해요.

### 핵심 기능

- **컨텍스트 수집**: 파일시스템 탐색, 시맨틱 검색, 자동 컴팩션 (컨텍스트 오버플로우 방지)
- **MCP 통합**: Model Context Protocol로 외부 서비스 연결 표준화
- **Sub-agent**: 독립 컨텍스트 윈도우로 병렬 처리와 격리

```python
import anthropic

client = anthropic.Anthropic()

# Sub-agent에 독립 태스크 위임
subagent_response = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=4096,
    system="You are a specialized research agent.",
    messages=[{"role": "user", "content": task_description}]
)
```

### 장단점

**장점**
- Claude Code 실제 운영 인프라 기반 — 실전 검증된 아키텍처
- MCP로 외부 서비스 연결 표준화
- Anthropic 모델 최적화 (구조화 출력, 도구 사용)

**단점**
- Anthropic 생태계 종속 (타 LLM 연동은 추가 작업 필요)
- Observability, 상태 영속성은 직접 구축 필요

**적합한 상황**: Claude 중심 스택, MCP 기반 외부 서비스 통합, 코드 실행 에이전트

---

## 5. AWS Strands Agents

### 설계 철학: 모델이 알아서 하게, 코드는 최소로

2025년 5월 16일 GA 출시한 AWS의 오픈소스 Agent 프레임워크예요. 현재 누적 다운로드 1,400만+, v1.40.0(2026년 5월 기준)까지 출시됐어요. Apache 2.0 라이선스예요.

핵심 철학은 **"Model-driven approach"**예요. 복잡한 워크플로우 설계 대신, 현대 LLM의 네이티브 추론과 도구 사용 능력을 그대로 활용해요. 에이전트 정의에 필요한 건 딱 3가지예요: **모델 + 도구 + 프롬프트**.

```python
from strands import Agent, tool
from strands_tools import calculator, web_search

# 기본 에이전트 — 3줄
agent = Agent(tools=[calculator, web_search])
response = agent("AI 칩 시장 점유율 조사하고 M2 Ultra vs H100 성능 비교해줘")

# 커스텀 Tool 정의
@tool
def word_count(text: str) -> int:
    """텍스트의 단어 수를 세어요."""
    return len(text.split())
```

### Amazon Bedrock 연동

Strands의 기본 모델 프로바이더는 Amazon Bedrock이에요. Tool use와 스트리밍을 지원하는 모든 Bedrock 모델과 호환되고, Bedrock Guardrails로 안전 거버넌스도 적용할 수 있어요. Bedrock 외에도 Anthropic, OpenAI, Ollama 같은 외부 프로바이더도 지원해요.

Amazon 내부에서 Amazon Q Developer, AWS Glue, VPC Reachability Analyzer에 실제로 사용 중이에요.

### 다른 프레임워크와 차별점

| 항목 | Strands | LangGraph | CrewAI |
|------|---------|-----------|--------|
| Bedrock 통합 | 네이티브 기본값 | 추가 설정 | 추가 설정 |
| 코드량 | 최소 (3줄) | 중간~높음 | 중간 |
| MCP 지원 | 내장 | 별도 통합 | 별도 통합 |
| OpenTelemetry | 내장 | LangSmith 별도 | 별도 |
| Hook 시스템 | 내장 | 커스텀 노드 | 제한적 |

### 장단점

**장점**
- AWS/Bedrock 스택에 네이티브 통합
- 최소 코드로 시작 가능 (진입 장벽 가장 낮음)
- OpenTelemetry 기반 observability 내장
- Amazon 프로덕션에서 추출된 실전 검증 코드

**단점**
- 2025년 5월 출시로 커뮤니티가 LangGraph/CrewAI 대비 작아요
- 복잡한 상태 흐름 제어는 LangGraph 대비 표현력이 아직 부족해요

**적합한 상황**: AWS/Bedrock 중심 스택, 빠른 프로토타이핑, MCP 기반 통합, 최소 코드 선호

---

## 어떤 걸 선택해야 할까

5개를 비교해봤을 때, 선택 기준을 이렇게 정리할 수 있어요.

```
Q1: AWS/Bedrock 스택을 주로 쓰나요?
  → Yes → Strands Agents

Q2: 복잡한 조건 분기와 Human approval이 필요한가요?
  → Yes → LangGraph

Q3: Microsoft 생태계(.NET, Azure)에 있나요?
  → Yes → Microsoft Agent Framework

Q4: Claude/Anthropic 중심으로 빠르게 만들고 싶나요?
  → Yes → Claude Agent SDK

Q5: 역할 기반 팀 구조가 직관적으로 맞나요?
  → Yes → CrewAI
```

### 종합 비교

| 항목 | LangGraph | CrewAI | MAF | Claude SDK | Strands |
|------|-----------|--------|-----|-----------|---------|
| 진입 장벽 | 높음 | 낮음 | 중간 | 중간 | 낮음 |
| 상태 제어 | 최고 | 낮음 | 높음 | 중간 | 중간 |
| Bedrock 통합 | 추가 작업 | 추가 작업 | Azure 중심 | 추가 작업 | 네이티브 |
| Human-in-loop | 내장 | 제한 | 내장 | 직접 구현 | 내장 |
| 커뮤니티 | 큼 | 큼 | 중간 | 성장 중 | 성장 중 |
| Observability | LangSmith | 별도 | OTEL 내장 | 직접 구현 | OTEL 내장 |

프레임워크를 고르는 데 많은 시간을 쓰는 것보다, 팀의 기존 스택과 주요 사용 LLM을 기준으로 빠르게 결정하는 게 낫더라고요. 초기 선택이 완벽할 필요는 없어요. 작게 시작하고, 실제 문제가 생기면 그때 마이그레이션을 고려하면 돼요.

다음 편에서는 실제 코드로 Agent를 만들어볼게요. 단일 Tool 에이전트부터 시작해서 Multi-agent 구조까지 단계적으로 구현해봐요.
