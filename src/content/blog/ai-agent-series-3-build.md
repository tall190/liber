---
title: '실제로 Agent 만들어보기: 단일 Tool에서 Multi-agent까지'
description: '동작하는 코드로 배우는 Agent 구현. 단일 Tool 에이전트부터 Multi-tool, Multi-agent 구조까지 단계별로 만들어보고, 흔한 실수와 해결 방법도 정리했어요.'
pubDate: '2026-05-23'
heroImage: '../../assets/ai-agent-series-3-build_hero_image.png'
tags: ['AI', 'Agent', 'Python', 'Anthropic', '코드']
series: 'AI Agent 완전정복'
---

개념과 프레임워크는 알겠는데, 막상 만들려고 하면 어디서 시작해야 할지 막막해요. 튜토리얼은 너무 단순하고, 실제 코드는 너무 복잡하죠.

이 글에서는 실제로 동작하는 Agent를 단계별로 만들어볼게요. 단일 Tool 에이전트부터 Multi-agent 구조까지, 코드 중심으로 설명해요.

## 환경 세팅

Python 3.11+ 환경과 Anthropic API 키가 필요해요.

```bash
pip install anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
```

이 글의 예시는 Anthropic Python SDK를 기준으로 해요. Tool Use 패턴 자체는 LangGraph나 Strands에서도 동일하게 적용돼요.

## Step 1: 단일 Tool 에이전트

가장 단순한 형태부터 시작해요.

```python
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "특정 도시의 현재 날씨를 조회한다",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "도시 이름 (예: 서울, Tokyo)"
                }
            },
            "required": ["city"]
        }
    }
]

def get_weather(city: str) -> dict:
    # 실제 구현에서는 외부 날씨 API 호출
    mock_data = {
        "서울": {"temp": 22, "condition": "맑음", "humidity": 60},
        "Tokyo": {"temp": 18, "condition": "흐림", "humidity": 75},
    }
    return mock_data.get(city, {"temp": 20, "condition": "알 수 없음", "humidity": 50})

def run_agent(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

        # 최종 답변이면 종료
        if response.stop_reason == "end_turn":
            return next(b.text for b in response.content if hasattr(b, "text"))

        # Tool 호출이면 실행하고 루프 계속
        if response.stop_reason == "tool_use":
            tool_block = next(b for b in response.content if b.type == "tool_use")
            result = get_weather(**tool_block.input)

            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": tool_block.id,
                    "content": json.dumps(result, ensure_ascii=False)
                }]
            })

print(run_agent("서울 날씨 어때?"))
# 출력: 현재 서울은 맑고 22°C예요. 습도는 60%예요.
```

핵심은 `while True` 루프예요. `stop_reason`이 `tool_use`이면 도구를 실행하고 결과를 메시지에 추가한 뒤 루프를 이어가요. `end_turn`이면 최종 답변을 반환해요.

## Step 2: Multi-tool 에이전트

Tool 하나로는 부족할 때, 여러 Tool을 조합할 수 있어요. 중요한 포인트가 하나 있어요. 하나의 LLM 응답에서 여러 Tool을 동시에 호출할 수 있어요. `for block in response.content`로 모든 Tool 호출을 한 번에 처리하는 게 그 이유예요.

```python
import anthropic
import json
import subprocess

client = anthropic.Anthropic()

tools = [
    {
        "name": "search_web",
        "description": "웹에서 정보를 검색한다",
        "input_schema": {
            "type": "object",
            "properties": {"query": {"type": "string"}},
            "required": ["query"]
        }
    },
    {
        "name": "run_python",
        "description": "Python 코드를 실행하고 결과를 반환한다",
        "input_schema": {
            "type": "object",
            "properties": {"code": {"type": "string"}},
            "required": ["code"]
        }
    },
    {
        "name": "write_file",
        "description": "파일에 내용을 저장한다",
        "input_schema": {
            "type": "object",
            "properties": {
                "filename": {"type": "string"},
                "content": {"type": "string"}
            },
            "required": ["filename", "content"]
        }
    }
]

def execute_tool(name: str, inputs: dict) -> str:
    if name == "search_web":
        # 실제 구현에서는 Tavily, SerpAPI 등 사용
        return f"'{inputs['query']}' 검색 결과: ..."

    elif name == "run_python":
        try:
            result = subprocess.run(
                ["python3", "-c", inputs["code"]],
                capture_output=True, text=True, timeout=10
            )
            return result.stdout if result.returncode == 0 else f"오류: {result.stderr}"
        except subprocess.TimeoutExpired:
            return "실행 시간 초과 (10초)"

    elif name == "write_file":
        with open(inputs["filename"], "w") as f:
            f.write(inputs["content"])
        return f"{inputs['filename']} 저장 완료"

    return "알 수 없는 Tool"

def run_agent(user_message: str, system: str = "") -> str:
    messages = [{"role": "user", "content": user_message}]
    kwargs = {
        "model": "claude-sonnet-4-6",
        "max_tokens": 4096,
        "tools": tools,
        "messages": messages
    }
    if system:
        kwargs["system"] = system

    while True:
        response = client.messages.create(**kwargs)

        if response.stop_reason == "end_turn":
            return next(b.text for b in response.content if hasattr(b, "text"))

        if response.stop_reason == "tool_use":
            # 여러 Tool 호출을 한 번에 처리
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
            kwargs["messages"] = messages

result = run_agent(
    "1부터 100까지 소수를 구하는 Python 코드 작성하고 실행한 뒤, 결과를 primes.txt에 저장해줘"
)
print(result)
```

## Step 3: Multi-agent 구조

복잡한 태스크는 여러 에이전트로 나누는 게 효과적이에요. Orchestrator가 Sub-agent를 관리하는 패턴이에요.

```python
import anthropic
import json
from concurrent.futures import ThreadPoolExecutor, as_completed

client = anthropic.Anthropic()

def run_subagent(system: str, task: str) -> str:
    """독립 컨텍스트 윈도우를 가진 Sub-agent"""
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # 서브에이전트는 저렴한 모델
        max_tokens=2048,
        system=system,
        messages=[{"role": "user", "content": task}]
    )
    return response.content[0].text

def orchestrator(user_goal: str) -> str:
    # 1단계: 태스크를 병렬 Sub-task로 분해
    plan = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="복잡한 태스크를 병렬 실행 가능한 독립 Sub-task로 분해해줘. JSON 배열로 반환 (각 항목: name/role/description).",
        messages=[{"role": "user", "content": user_goal}]
    ).content[0].text

    subtasks = json.loads(plan)

    # 2단계: 병렬 실행
    results = {}
    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = {
            executor.submit(
                run_subagent,
                f"당신은 {t['role']}이에요.",
                t["description"]
            ): t["name"]
            for t in subtasks
        }
        for future in as_completed(futures):
            results[futures[future]] = future.result()

    # 3단계: 결과 통합
    return client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system="여러 에이전트의 결과를 통합해 최종 보고서를 작성해요.",
        messages=[{
            "role": "user",
            "content": f"목표: {user_goal}\n\n결과:\n{json.dumps(results, ensure_ascii=False)}"
        }]
    ).content[0].text

print(orchestrator("AI 프레임워크 시장 동향 분석 후 2페이지 보고서 작성"))
```

Sub-agent에는 Haiku를, Orchestrator에는 Sonnet을 써요. Sonnet과 Haiku의 비용 차이는 약 15배인데, Sub-agent에 단순 작업을 위임하면 전체 비용을 크게 줄일 수 있어요.

## 흔한 실수 4가지

**실수 1: 무한 루프**

Tool이 항상 같은 결과를 반환하거나 에이전트가 루프를 탈출하지 못하는 경우가 있어요.

```python
# 나쁜 예
while True:
    response = client.messages.create(...)

# 좋은 예
for _ in range(15):  # 최대 반복 제한
    response = client.messages.create(...)
    if response.stop_reason == "end_turn":
        break
else:
    return "최대 반복 횟수 초과"
```

**실수 2: Tool 에러를 에이전트에게 숨김**

Tool 실행이 실패했을 때 에이전트에 알리지 않으면 잘못된 정보로 계속 진행해요.

```python
# 나쁜 예
result = json.dumps(call_api(inputs))

# 좋은 예: 에이전트가 오류를 인지하고 대안 탐색
try:
    result = json.dumps(call_api(inputs))
except Exception as e:
    result = f"오류 발생: {e}. 다른 방법을 시도해주세요."
```

**실수 3: 컨텍스트 폭발**

긴 대화에서 메시지가 쌓이면 컨텍스트 윈도우를 초과해요.

```python
def compress_if_needed(messages, threshold=80000):
    if sum(len(str(m)) for m in messages) < threshold:
        return messages
    recent = messages[-4:]
    summary = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=300,
        messages=[{"role": "user", "content": f"3문장으로 요약: {messages[:-4]}"}]
    ).content[0].text
    return [{"role": "user", "content": f"[이전 요약]: {summary}"}] + recent
```

**실수 4: System prompt 없이 시작**

Tool을 어떻게 써야 하는지, 어떤 형식으로 답해야 하는지 명확히 지정하지 않으면 에이전트 행동이 일관성 없어져요.

```python
system = """당신은 데이터 분석 에이전트예요.
- 항상 코드를 실행해서 결과를 확인하고 답해요
- 수치는 반드시 실제 계산 결과를 써요
- 가정으로 답하지 마요"""
```

## 정리

단순한 Tool 에이전트에서 시작해서 Multi-agent 구조까지 만들어봤어요. 핵심 패턴은 변하지 않아요: **Tool 호출 → 결과 수집 → 다음 행동 결정 → 반복**. 이 루프 위에 안정성 장치들을 붙여가면 프로덕션 에이전트가 돼요.

다음 편에서는 이 에이전트를 실제 프로덕션 환경에 올리기 위해 필요한 것들 — Observability, 비용 관리, 안정성, 보안 — 을 다뤄볼게요.
