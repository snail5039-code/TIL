# ReAct Agent, Memory & State

## 1. ReAct Agent란?

ReAct는 **Reasoning + Acting**의 줄임말로, LLM이 현재 상황을 판단하고 필요한 Tool을 호출한 뒤 실행 결과를 다시 확인하는 과정을 반복하는 패턴이다.

```
판단(Reason) → Tool 호출(Act) → 실행 결과 확인(Observe)
→ 다시 판단 → 최종 답변
```

단순한 Workflow는 개발자가 실행 순서를 미리 정하지만, ReAct Agent는 주어진 Tool 중 무엇을 사용할지, 몇 번 호출할지, 언제 답변을 끝낼지를 LLM이 판단한다.

## 2. Tool 정의

LLM은 Python 함수의 내부 코드를 직접 보는 것이 아니라 Tool의 이름, 설명, 입력값 구조를 보고 사용할 Tool을 선택한다.

```python
@tool(parse_docstring=True)
def search_products_tool(keyword: str) -> str:
    """상품명을 검색하여 상품 ID를 반환한다.

    Args:
        keyword: 상품명 검색어
    """
    return search_products(keyword)
```

기존 기능 함수와 Agent용 Tool을 분리하면 역할이 명확해진다.

```
search_products()      → 실제 상품 검색
search_products_tool() → Agent가 호출하는 Tool
```

Tool의 이름과 docstring이 모호하면 LLM이 잘못된 Tool이나 인자를 선택할 수 있으므로 구체적으로 작성해야 한다.

## 3. ReAct 그래프 직접 구성하기

ReAct Agent를 LangGraph로 직접 구성할 때는 LLM 노드와 Tool 실행 노드가 필요하다.

```python
def chatbot(state: MessagesState):
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}
```

`MessagesState`에는 사용자 메시지, AI의 Tool 호출, Tool 실행 결과와 최종 답변이 순서대로 누적된다.

```python
builder = StateGraph(MessagesState)

builder.add_node("chatbot", chatbot)
builder.add_node("tools", ToolNode(tools))

builder.add_edge(START, "chatbot")
builder.add_conditional_edges("chatbot", tools_condition)
builder.add_edge("tools", "chatbot")

graph = builder.compile()
```

실행 흐름은 다음과 같다.

```
START
→ chatbot이 다음 행동 판단
→ Tool이 필요하면 tools로 이동
→ Tool 결과를 State에 저장
→ chatbot으로 돌아가 다시 판단
→ Tool이 필요하지 않으면 END
```

`tools_condition`은 AI 메시지에 `tool_calls`가 있으면 `"tools"`로 보내고, 없으면 `END`로 보낸다. 따라서 `END` Edge를 따로 작성하지 않아도 된다.

상품 검색 실습에서는 다음 순서로 실행됐다.

```
상품 질문
→ search_products_tool로 상품 ID 검색
→ get_product_tool로 가격과 재고 조회
→ 조회 결과를 바탕으로 최종 답변
```

## 4. create_agent() 사용하기

직접 만든 ReAct 구조는 `create_agent()`를 사용해 간단하게 구성할 수도 있다.

```python
agent = create_agent(
    model=llm,
    tools=tools,
)
```

`create_agent()`는 전달받은 모델과 Tool을 이용해 LLM 노드, Tool 실행 노드, 조건 분기와 반복 구조를 내부에서 자동으로 만든다.

```
직접 구성: StateGraph → Node 등록 → Edge 연결 → compile
자동 구성: create_agent(model=llm, tools=tools)
```

두 방식 모두 ReAct 패턴이다. 직접 구성하면 흐름을 세밀하게 제어할 수 있고, `create_agent()`를 사용하면 기본적인 Tool Agent를 빠르게 만들 수 있다.

## 5. Local File Agent 실습

로컬 파일 실습에서는 다음 Tool을 만들었다.

```
search_files_tool → 파일 검색
read_files_tool   → 파일 읽기
write_files_tool  → 파일 쓰기
```

Agent는 사용자의 요청을 확인하고 필요한 Tool을 선택한다.

```
파일을 찾아 읽고 요약해 저장해줘
→ 파일 검색
→ 검색된 파일 읽기
→ 핵심 내용 작성
→ 새로운 txt 파일로 저장
```

모든 파일 작업은 현재 작업 폴더의 `.txt` 파일로 제한했다. Tool을 사용할 때는 LLM의 판단뿐 아니라 접근 경로, 확장자, 파일 크기와 같은 안전 검사도 필요하다.

## 6. State와 Checkpointer

Checkpointer가 없는 그래프에서는 각각의 `invoke()`가 독립적으로 실행된다.

```
invoke 1 → State 생성 및 사용 → 종료
invoke 2 → 새로운 State로 시작
```

Checkpointer를 연결하면 State의 스냅샷을 저장하고, 같은 `thread_id`에서 이전 대화를 이어갈 수 있다.

```python
memory = MemorySaver()

graph = builder.compile(
    checkpointer=memory,
)
```

호출할 때 대화창을 구분할 `thread_id`를 전달한다.

```python
config = {
    "configurable": {
        "thread_id": "conversation-1"
    }
}
```

- 같은 `thread_id`: 이전 대화를 기억
- 다른 `thread_id`: 별도의 대화로 시작

스냅샷은 그래프가 실행되는 특정 순간의 State를 저장한 복사본이다. `get_state()`는 최신 State를, `get_state_history()`는 과거 스냅샷을 최신순으로 반환한다.

## 7. Store를 이용한 Long-term Memory

Checkpointer는 하나의 대화창 안에서 State를 이어준다. Store는 서로 다른 `thread_id`에서도 사용자 정보를 공유하기 위해 사용한다.

```python
store = InMemoryStore()

store.put(
    namespace=("users", "jun"),
    key="profile",
    value={
        "name": "jun",
        "interest": "ai agent",
    },
)
```

저장 구조는 다음과 같다.

```
users
└─ jun
   └─ profile
      ├─ name: jun
      └─ interest: ai agent
```

호출할 때 사용자 식별자를 전달한다.

```python
config = {
    "configurable": {
        "thread_id": "conversation-1",
        "my_data": "jun",
    }
}
```

노드에서는 `"jun"`을 꺼내 Store 주소를 완성한다.

```python
my_data = config["configurable"]["my_data"]
profile = store.get(("users", my_data), "profile")
```

```
config에서 jun 확인
→ ("users", "jun") 주소 생성
→ 해당 위치의 profile 조회
→ 프로필을 SystemMessage로 LLM에게 전달
```

여기서 `my_data`는 LangGraph가 정한 이름이 아니라 개발자가 자유롭게 정한 키다. 호출하는 코드와 꺼내는 코드에서 같은 이름을 사용하면 된다.

### Checkpointer와 Store의 차이

| 구분 | Checkpointer | Store |
|---|---|---|
| 식별 기준 | `thread_id` | namespace와 key |
| 주요 데이터 | 현재 대화 메시지와 State | 사용자 프로필과 장기 설정 |
| 사용 범위 | 같은 대화창 | 여러 대화창 |
| 예시 | 방금 나눈 이야기 | 이름, 관심 분야, 선호 |

## 8. 대화 요약 관리

State에 메시지를 계속 누적하면 LLM에 전달되는 토큰과 처리 비용이 증가한다. 이를 줄이기 위해 오래된 메시지를 요약하고 최근 메시지만 원문으로 유지할 수 있다.

```python
class SummaryState(MessagesState):
    summary: str
```

`MessagesState`에 오래된 대화 요약을 저장할 `summary` 필드를 추가했다. 전체 메시지가 설정한 토큰 수를 넘으면 요약 노드로 이동한다.

```python
if token_count > MAX_TOKENS:
    return "summarize"
return END
```

최근 메시지는 `trim_messages()`로 선택한다.

```python
kept_messages = trim_messages(
    messages,
    max_tokens=100,
    strategy="last",
    token_counter=llm,
    start_on="human",
)
```

```
최근 100토큰의 메시지 → 원문 유지
나머지 오래된 메시지 → 요약 대상
```

요약 지시문과 오래된 메시지를 하나의 프롬프트로 합친 뒤 LLM에게 전달한다.

```python
summary_response = llm.invoke(prompt)
```

기존 요약이 있다면 이전 요약과 새로 잘라낸 대화를 함께 요약해 `summary`를 갱신한다. 요약에 반영된 원본 메시지는 `RemoveMessage`로 현재 State에서 삭제한다.

```python
return {
    "summary": summary_response.text,
    "messages": [
        RemoveMessage(id=message.id)
        for message in messages_to_summarize
    ],
}
```

다음 LLM 호출에는 **시스템 지시 + 이전 대화 요약 + 최근 메시지 원문**이 전달된다. 이를 통해 오래된 대화의 핵심 맥락은 유지하면서 토큰 사용량을 줄일 수 있다.

## 9. 이번 학습에서 기억할 점

- LangGraph는 큰 실행 틀이고, ReAct는 그 안에서 구현할 수 있는 Agent 패턴 중 하나다.
- 노드와 Edge를 사용한다고 모든 그래프가 ReAct인 것은 아니다.
- 판단, Tool 호출, 결과 관찰을 반복해야 ReAct라고 할 수 있다.
- `create_agent()`는 기본적인 ReAct 그래프를 내부에서 자동으로 구성한다.
- State는 현재 실행 데이터를 관리하고, Reducer는 기존 값과 새 값을 합치는 규칙이다.
- Checkpointer는 같은 `thread_id`의 대화를 이어준다.
- Store는 여러 대화창에서 공유할 사용자 정보를 저장한다.
- Checkpointer와 Store는 상하 관계가 아니라 서로 다른 기억을 담당한다.
- 대화가 길어지면 오래된 메시지를 요약하고 최근 메시지만 원문으로 유지할 수 있다.
- Store에 저장된 정보는 자동으로 수정되지 않으며, 사용자 정보를 갱신하려면 별도의 저장 기능이 필요하다.

## 10. 헷갈린 점

- 처음에는 직접 구성한 LangGraph와 `create_agent()`가 서로 다른 기술이라고 생각했지만, `create_agent()`도 내부적으로 LangGraph 기반의 ReAct 구조를 자동 생성한다는 점을 이해했다.
- State와 Reducer를 기억 기능으로 생각하기 쉬웠지만, State는 현재 실행 데이터이고 Reducer는 데이터를 합치는 규칙이며 다음 실행까지 State를 이어가려면 Checkpointer가 필요하다는 점을 구분했다.
- Checkpointer는 하나의 대화창을 이어주고 Store는 여러 대화창에서 사용자 프로필을 공유한다는 차이를 이해했다.
- 대화 요약은 전체 메시지를 단순히 삭제하는 것이 아니라, 최근 메시지는 원문으로 남기고 오래된 메시지만 요약해 다음 LLM 호출에 함께 전달하는 방식이라는 것을 알게 되었다.

### Store, Checkpointer, Summary를 같이 사용할 경우

세 기능을 합치면 챗봇 노드가 다음 정보를 함께 사용한다.

```
Checkpointer → 현재 대화창의 최근 메시지
Store        → 사용자 프로필
Summary      → 오래된 대화의 요약
```

```python
from langchain_core.messages import HumanMessage, RemoveMessage, SystemMessage
from langchain_core.runnables import RunnableConfig
from langchain_core.messages.utils import trim_messages
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import END, START, MessagesState, StateGraph
from langgraph.store.base import BaseStore
from langgraph.store.memory import InMemoryStore


# 최근 메시지와 오래된 대화 요약을 저장한다.
class ChatState(MessagesState):
    summary: str


# 사용자 프로필 저장소
store = InMemoryStore()
store.put(
    namespace=("users", "jun"),
    key="profile",
    value={
        "name": "jun",
        "interest": "AI Agent",
    },
)


def chatbot(
    state: ChatState,
    config: RunnableConfig,
    store: BaseStore,
):
    # Store에서 현재 사용자 프로필을 가져온다.
    user_id = config["configurable"]["user_id"]
    profile = store.get(("users", user_id), "profile")

    messages = []

    if profile:
        messages.append(
            SystemMessage(
                content=(
                    f"사용자 이름: {profile.value['name']}\n"
                    f"관심 분야: {profile.value['interest']}"
                )
            )
        )

    # 오래된 대화 요약이 있으면 함께 전달한다.
    if state.get("summary"):
        messages.append(
            HumanMessage(
                content=f"이전 대화 요약:\n{state['summary']}"
            )
        )

    # Checkpointer가 이어준 최근 대화를 추가한다.
    messages.extend(state["messages"])

    response = llm.invoke(messages)
    return {"messages": [response]}


def route_after_chatbot(state: ChatState):
    token_count = llm.get_num_tokens_from_messages(
        state["messages"]
    )

    if token_count > 200:
        return "summarize"

    return END


def summarize_conversation(state: ChatState):
    messages = state["messages"]
    existing_summary = state.get("summary", "")

    # 최신 메시지 일부는 원문으로 유지한다.
    kept_messages = trim_messages(
        messages,
        max_tokens=100,
        strategy="last",
        token_counter=llm,
        start_on="human",
    )

    kept_ids = {message.id for message in kept_messages}

    # 최신 메시지에 포함되지 않은 오래된 메시지만 고른다.
    old_messages = [
        message
        for message in messages
        if message.id not in kept_ids
    ]

    prompt = (
        "다음 대화에서 사용자 정보와 중요한 맥락을 "
        "2문장 이내로 요약해줘.\n\n"
    )

    if existing_summary:
        prompt += f"기존 요약:\n{existing_summary}\n\n"

    prompt += "새로 요약할 대화:\n"

    for message in old_messages:
        prompt += f"{message.type}: {message.content}\n"

    summary_response = llm.invoke(prompt)

    return {
        "summary": summary_response.text,
        "messages": [
            RemoveMessage(id=message.id)
            for message in old_messages
        ],
    }
```

그래프는 다음과 같이 구성한다.

```python
builder = StateGraph(ChatState)

builder.add_node("chatbot", chatbot)
builder.add_node("summarize", summarize_conversation)

builder.add_edge(START, "chatbot")
builder.add_conditional_edges(
    "chatbot",
    route_after_chatbot,
    {
        "summarize": "summarize",
        END: END,
    },
)
builder.add_edge("summarize", END)

graph = builder.compile(
    checkpointer=MemorySaver(),
    store=store,
)
```

실행할 때는 대화창 ID와 사용자 ID를 함께 전달한다.

```python
config = {
    "configurable": {
        "thread_id": "conversation-1",
        "user_id": "jun",
    }
}

result = graph.invoke(
    {"messages": [("human", "내 관심 분야가 뭐였지?")]},
    config=config,
)

print(result["messages"][-1].text)
```

실행 흐름은 다음과 같다.

```
user_id로 Store의 jun 프로필 조회
→ 같은 thread_id의 최근 대화 복원
→ 기존 summary가 있으면 함께 전달
→ LLM 답변
→ 200토큰 초과 여부 확인
→ 초과하면 오래된 메시지만 요약
→ Checkpointer가 변경된 State 저장
```

즉, Store, Checkpointer, Summary는 각각 별도의 기능이지만 하나의 챗봇 노드와 그래프 안에서 함께 사용할 수 있다.
