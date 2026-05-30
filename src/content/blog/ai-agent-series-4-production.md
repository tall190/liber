---
title: 'Agent를 Production으로 보내기: Observability, 비용, 안정성, 보안'
description: '로컬에서 잘 돌던 에이전트가 프로덕션에서 무너지는 이유. Observability 도구 비교, 비용 50~80% 절감 방법, Retry/Timeout/Fallback 패턴, Prompt Injection 방어까지 실무 체크리스트로 정리했어요.'
pubDate: '2026-05-24'
heroImage: '../../assets/ai-agent-series-4-production_hero_image.png'
tags: ['AI', 'Agent', 'Production', 'Observability', '비용최적화', '보안']
series: 'AI Agent 완전정복'
---

로컬에서 잘 돌던 에이전트가 프로덕션에 올라가면 달라지기 시작해요. 비용이 예상보다 빨리 나가고, 특정 입력에서 무한 루프가 생기고, 어떤 Tool이 실패했는지 추적이 안 되죠.

이 글에서는 Agent를 프로덕션에 안정적으로 운영하기 위해 필요한 것들을 다뤄요. Observability, 비용 관리, 안정성, 보안 순서로 살펴볼게요.

## Observability: 에이전트 내부를 보는 법

LLM 앱과 Agent의 Observability는 달라요. API 응답 시간이나 에러율만으로는 부족해요. "어떤 Tool을 몇 번 호출했는가", "어디서 루프가 돌았는가", "어떤 프롬프트에서 비용이 튀었는가"까지 봐야 해요.

### 주요 도구 비교

| 도구 | 강점 | 약점 | 적합한 상황 |
|------|------|------|------------|
| LangSmith | LangGraph 네이티브, 에이전트 트리 시각화 | LangChain 의존, 이식성 낮음 | LangGraph 전용 팀 |
| Arize Phoenix | OTEL 표준, RAG 평가 최고 수준, 완전 self-hostable | — | 엔터프라이즈, 혼합 ML/LLM |
| W&B Weave | ML 실험 + 프로덕션 연계, 프롬프트 버전 관리 | per-seat 가격 | 연구 집약적 팀 |
| Langfuse | 오픈소스, 경량, 자가 호스팅 | 일부 기능 제한 | 소규모 팀, 비용 민감 |

프레임워크를 자유롭게 바꿀 가능성이 있다면 OpenTelemetry를 기본으로 지원하는 Phoenix나 Langfuse를 선택하는 게 유리해요. LangSmith는 LangGraph 생태계를 벗어나면 전체 재계측이 필요해요.

### 직접 계측: OpenTelemetry

```python
from opentelemetry import trace

tracer = trace.get_tracer("agent")

def run_agent(user_message: str) -> str:
    with tracer.start_as_current_span("agent_run") as span:
        span.set_attribute("user.message", user_message)
        iteration = 0

        while True:
            with tracer.start_as_current_span(f"llm_call_{iteration}") as llm_span:
                response = client.messages.create(...)
                llm_span.set_attribute("tokens.input", response.usage.input_tokens)
                llm_span.set_attribute("tokens.output", response.usage.output_tokens)

            if response.stop_reason == "tool_use":
                tool_block = next(b for b in response.content if b.type == "tool_use")
                with tracer.start_as_current_span("tool_execution") as tool_span:
                    tool_span.set_attribute("tool.name", tool_block.name)
                    result = execute_tool(tool_block.name, tool_block.input)

            iteration += 1
```

이렇게 하면 LLM 호출 횟수, Tool 실행 소요 시간, 어느 지점에서 실패했는지 추적할 수 있어요.

## 비용 관리: 예산을 지키는 법

체계적으로 최적화하면 50~80% 절감이 가능해요. 우선순위 순서로 적용할 방법들이에요.

### 1. 모델 라우팅

태스크 복잡도에 따라 모델을 나눠요. claude-sonnet-4-6과 claude-haiku-4-5 간 비용 차이는 약 15배예요.

```python
def select_model(task: str) -> str:
    simple_patterns = ["요약", "번역", "형식 변환", "분류"]
    if any(p in task for p in simple_patterns):
        return "claude-haiku-4-5-20251001"
    return "claude-sonnet-4-6"
```

Multi-agent 구조에서는 Sub-agent에 Haiku를, Orchestrator에 Sonnet을 배치하는 게 기본 패턴이에요.

### 2. 프롬프트 캐싱

반복되는 시스템 프롬프트를 캐싱하면 반복 호출 비용을 크게 줄일 수 있어요. Anthropic Prompt Caching은 캐시된 토큰을 표준의 10% 가격으로 제공해요.

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": long_system_prompt,
            "cache_control": {"type": "ephemeral"}  # 캐시 활성화
        }
    ],
    messages=messages
)
```

같은 시스템 프롬프트로 반복 호출할 때, 첫 호출 이후 캐시가 적중되면 비용이 90% 줄어요.

### 3. 예산 한도

```python
MAX_ITERATIONS = 15
TOKEN_BUDGET = 50000
token_used = 0

for _ in range(MAX_ITERATIONS):
    response = client.messages.create(...)
    token_used += response.usage.input_tokens + response.usage.output_tokens

    if token_used > TOKEN_BUDGET:
        return "토큰 예산 초과. 작업 중단."

    if response.stop_reason == "end_turn":
        break
```

## 안정성: 실패에 강한 에이전트

### Retry with Exponential Backoff

```python
import time
import random

def call_with_retry(func, max_retries=3, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return func()
        except anthropic.RateLimitError:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            time.sleep(delay)
        except anthropic.APIError as e:
            if e.status_code >= 500 and attempt < max_retries - 1:
                time.sleep(base_delay * (2 ** attempt))
            else:
                raise
```

LangGraph v1.1부터는 이 retry middleware가 프레임워크 수준에서 내장됐어요.

### Tool Timeout

Tool이 응답하지 않으면 에이전트 전체가 멈춰요. 각 Tool 호출에 타임아웃을 걸어요.

```python
import signal
from contextlib import contextmanager

@contextmanager
def timeout(seconds):
    def handler(signum, frame):
        raise TimeoutError(f"{seconds}초 초과")
    signal.signal(signal.SIGALRM, handler)
    signal.alarm(seconds)
    try:
        yield
    finally:
        signal.alarm(0)

def execute_tool_safe(name, inputs):
    try:
        with timeout(30):
            return execute_tool(name, inputs)
    except TimeoutError:
        return {"error": "도구 실행 시간 초과 (30초)"}
```

### Fallback 전략

주요 경로가 실패했을 때 대안을 준비해요.

```python
def get_data(query: str) -> str:
    try:
        return fetch_from_api(query)         # 1차: 실시간 API
    except APIError:
        try:
            return fetch_from_cache(query)   # 2차: 캐시
        except CacheError:
            return f"데이터 조회 실패. 가능한 대안: {suggest_alternatives(query)}"
```

## 보안

### Prompt Injection 방어

악의적인 사용자가 에이전트를 통해 의도하지 않은 행동을 유발할 수 있어요.

```python
DANGEROUS_PATTERNS = [
    "ignore previous instructions",
    "system:",
    "<|im_start|>",
    "IGNORE ALL PREVIOUS",
]

def validate_input(user_input: str) -> str:
    for pattern in DANGEROUS_PATTERNS:
        if pattern.lower() in user_input.lower():
            raise ValueError("의심스러운 입력 감지")
    return user_input
```

더 강력한 방법은 별도의 LLM으로 입력을 검증하는 거예요 (LLM-as-judge).

### 권한 최소화

에이전트에 꼭 필요한 Tool만 제공해요. 파일 읽기만 필요한 에이전트에 쓰기 권한 Tool을 주지 마요.

```python
# 나쁜 예: 모든 Tool을 하나의 에이전트에
agent = Agent(tools=[read_file, write_file, delete_file, execute_code])

# 좋은 예: 역할에 맞는 Tool만
read_agent = Agent(tools=[read_file, search_web])   # 읽기 전용
write_agent = Agent(tools=[write_file])              # 별도 격리
```

### 중요 행동에 Human-in-the-loop

파일 삭제, 결제, 이메일 발송처럼 되돌리기 어려운 행동 전에 사람의 확인을 받아요. LangGraph의 `interrupt` 메커니즘이나 Strands의 Human Interruption API를 쓰면 이 패턴을 체계적으로 구현할 수 있어요.

## 프로덕션 배포 전 체크리스트

```
Observability
□ 모든 LLM 호출에 트레이싱 적용
□ Tool 호출 성공/실패 로깅
□ 토큰 사용량 모니터링
□ 비용 이상 알림 설정

비용
□ 모델 라우팅 (복잡도별 모델 분리)
□ 프롬프트 캐싱 활성화
□ 최대 반복 횟수 및 토큰 예산 설정
□ 컨텍스트 압축 로직 구현

안정성
□ API 호출 Retry + Exponential Backoff
□ 모든 Tool에 Timeout 설정
□ Fallback 경로 정의
□ 최대 실행 시간 제한

보안
□ 사용자 입력 검증 (Prompt Injection 방어)
□ Tool 권한 최소화
□ 중요 행동에 Human approval 추가
□ API 키 환경변수 관리 (코드 하드코딩 금지)

테스트
□ 정상 케이스 E2E 테스트
□ Tool 실패 시 동작 검증
□ 컨텍스트 한계 근처 동작 확인
□ 악의적 입력에 대한 동작 확인
```

## 정리

프로덕션 에이전트 운영은 기능 구현보다 운영 인프라가 더 어려운 경우가 많아요.

Observability 없이 배포하면 뭔가 이상해도 어디서 문제가 생겼는지 알 수 없어요. 비용 제어 없이 배포하면 예산이 갑자기 터져요. 보안 없이 배포하면 공격 벡터가 생겨요.

이 시리즈를 통해 Agent의 개념부터 프레임워크 선택, 실제 구현, 프로덕션 운영까지 한 사이클을 돌아봤어요. 시작은 작게 — 도구 하나, 루프 하나부터예요.
