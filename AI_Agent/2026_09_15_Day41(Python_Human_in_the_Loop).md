# [TIL] AI의 실행을 멈추고 사람의 승인과 입력으로 다시 진행하기

이번에는 AI가 작업을 진행하다가 사람의 판단이 필요한 지점에서 멈추는 Human-in-the-Loop를 학습했다.

결제와 게시글 발행 전에 승인을 받고, Planner가 만든 계획을 사람이 검토한 뒤 수정하거나 실행하는 흐름을 구현했다. 추가로, 사용자에게 필요한 정보를 질문하고 답변을 정리하는 구조도 살펴봤다.

```plain text
AI 작업 진행
→ 사람의 판단이 필요한 지점에서 중단
→ 질문과 검토할 내용을 사용자에게 전달
→ 승인·수정 의견·답변 입력
→ 저장된 실행 재개
```

아래 내용은 `21-human-in-the-loop.ipynb`와 실습 중 질문한 내용을 바탕으로 정리했다. 코드 예시는 설명에 맞게 변수 이름과 입력 처리를 일부 다듬었다. 마지막 추가 질문 예제는 파일에 문제 설명만 있어 보충한 구현이다.

## 1. Human-in-the-Loop란?

Human-in-the-Loop, 줄여서 HITL은 AI의 처리 과정에 사람이 개입하는 방식이다.

예를 들면 다음과 같다.

- 결제를 실행하기 전에 금액과 수신자를 확인한다.
- 게시글을 공개하기 전에 제목과 내용을 검토한다.
- AI가 만든 계획을 승인하거나 수정 의견을 전달한다.
- 요청을 처리하기에 정보가 부족하면 추가 질문에 답한다.

사람이 개입한다고 항상 승인과 거절만 있는 것은 아니다. 수정할 내용이나 부족한 정보를 입력하는 것도 HITL이다.

## 2. interrupt()는 무엇인가?

`interrupt`의 발음은 **인터럽트**다. LangGraph에서 제공하는 함수이며, Python 자체의 내장 함수는 아니다.

```python
from langgraph.types import Command, interrupt
```

```python
decision = interrupt(
    {
        "question": "이 계획을 승인하겠습니까?",
        "plan": state["plan"],
    }
)
```

이 코드에 도달하면 그래프 실행이 중단되고, 딕셔너리에 담은 질문과 계획이 호출자에게 전달된다.

처음 중단될 때는 아직 사용자의 결정이 없으므로 아래쪽 코드는 실행되지 않는다. 나중에 `Command(resume=...)`로 값을 전달하면 그 값이 `interrupt()`의 반환값이 된다.

```plain text
# 외부에서 전달
Command(resume={"action": "approve"})

# 재개된 함수 안에서 받는 값
decision = {"action": "approve"}
```

`interrupt()`에 넣는 값과 `interrupt()`가 반환하는 값은 서로 다르다.

## 3. State, interrupt 값, resume 값 구분하기

| 구분 | 역할 | 예시 |
|---|---|---|
| State | 노드들이 공유하는 실행 데이터 | 계획, 완료 기록, 피드백 |
| interrupt에 넣는 값 | 사용자에게 보여줄 검토 내용 | 질문, 계획, 도구 인자 |
| resume에 넣는 값 | 사용자가 돌려주는 답변 | 승인, 거절, 수정 의견 |

```python
decision = interrupt(
    {
        "question": "게시글을 발행할까요?",
        "args": {"title": title, "content": content},
    }
)
```

여기서 사용자에게 전달되는 내용은 다음과 같다.

```plain text
{
    "question": "게시글을 발행할까요?",
    "args": {
        "title": "테스트",
        "content": "오늘 날씨가 좋다",
    },
}
```

사용자가 승인하면 별도의 값을 돌려준다.

```plain text
Command(resume={"action": "approve"})
```

그러면 함수 안의 `decision`은 다음과 같다.

```plain text
{"action": "approve"}
```

따라서 처음 `interrupt()`에 넣은 딕셔너리에 `"action"`이 없어도 된다. `"action"`은 나중에 재개할 때 전달하는 딕셔너리의 키다.

## 4. action, approve는 누가 정한 이름인가?

다음 이름들은 개발자가 정한 약속이다.

```plain text
{"action": "approve"}
```

- `"action"`: 어떤 결정을 했는지 저장할 키
- `"approve"`: 승인이라는 의미로 사용한 문자열
- `"reject"`: 거절이라는 의미로 사용한 문자열
- `"edit"`: 수정 요청이라는 의미로 사용한 문자열

LangGraph가 이 단어들을 보고 자동으로 승인하거나 거절하지 않는다. 코드가 값을 읽고 조건문이나 라우터로 처리한다.

```python
if decision["action"] == "approve":
    return "승인되었습니다."
```

이 코드는 `"action"`이 있는지 확인하는 코드가 아니다.

```python
decision["action"]  # action의 값을 꺼냄
```

키가 있는지 확인하려면 다음과 같이 작성한다.

```python
if "action" in decision:
    ...
```

키가 없는 경우 기본값을 사용하려면 `get()`을 쓴다.

```python
reason = decision.get("reason", "사유 없음")
```

## 5. Checkpointer와 thread_id가 필요한 이유

중단된 실행을 나중에 이어가려면 실행 상태를 저장하고 다시 찾을 수 있어야 한다.

```python
graph = builder.compile(checkpointer=MemorySaver())

config = {
    "configurable": {
        "thread_id": "payment-1",
    }
}
```

| 구성 | 역할 |
|---|---|
| `State` | 어떤 데이터를 공유할지 정의 |
| `MemorySaver()` | 그래프 상태와 중단 정보를 메모리에 저장 |
| `thread_id` | 저장된 실행 문맥을 구분 |

State와 Checkpointer는 역할이 다르다.

```plain text
State: 저장할 데이터의 구조
Checkpointer: 실행 상태를 기록하고 복원하는 장치
thread_id: 어떤 기록을 사용할지 구분하는 값
```

`MemorySaver()`는 메모리 기반이다. 노트북 커널이나 프로세스가 종료되어도 복원해야 한다면 DB 기반 Checkpointer 같은 영속 저장 방식이 필요하다.

또한 같은 `thread_id`로 재개해야 기존 중단 지점을 이어갈 수 있다. [공식 문서: Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)

## 6. 중단 상태는 어디에 담겨 오는가?

```python
result = graph.invoke(
    {"messages": [("user", "홍길동에게 50000원 결제해줘")]},
    config,
)
```

실행 중 `interrupt()`가 발생하면, 이번 노트북에서 사용하는 반환 방식에서는 `result`에 `"__interrupt__"` 키가 포함된다.

```python
request = result["__interrupt__"][0].value
```

하나씩 나누면 다음과 같다.

| 코드 | 의미 |
|---|---|
| `result` | 그래프 실행이 반환한 결과 |
| `["__interrupt__"]` | 중단 정보를 꺼냄 |
| `[0]` | 첫 번째 중단 정보 선택 |
| `.value` | `interrupt()`에 넣었던 실제 내용 |

```python
print(request["question"])
print(request["args"])
```

`"__interrupt__"`라는 문자열이 그래프를 멈추는 것은 아니다. 그래프를 멈추는 것은 함수 안의 `interrupt()` 호출이고, `"__interrupt__"`는 반환된 중단 정보를 읽을 때 쓰는 키다.

## 7. 결제 Tool 안에서 승인받기

다음은 노트북의 결제 예제를 승인 값까지 명확히 확인하도록 다듬은 코드다.

```python
@tool
def send_payment(amount: int, recipient: str) -> str:
    """수신자에게 지정한 금액의 결제를 요청한다."""

    decision = interrupt(
        {
            "question": "이 결제를 실행하시겠습니까?",
            "tool": "send_payment",
            "args": {
                "amount": amount,
                "recipient": recipient,
            },
        }
    )

    if decision["action"] == "reject":
        reason = decision.get("reason", "사유 없음")
        return f"결제가 취소되었습니다. 사유: {reason}"

    if decision["action"] != "approve":
        raise ValueError("승인 또는 거절 값을 보내야 합니다.")

    # 실제 결제 API를 호출한다면 이 위치에서 처리한다.
    return f"{recipient}에게 {amount:,}원 결제 완료."
```

`"tool"`과 `"args"`는 사용자가 무엇을 승인하는지 확인하도록 전달하는 정보다.

이 이름과 구조가 `interrupt()`의 필수 양식은 아니다. 다만 결제 금액과 수신자를 보여줘야 사용자가 내용을 확인하고 승인할 수 있다.

현재 예제는 문자열만 반환한다. `"결제 완료"`가 출력되어도 실제 결제 시스템에 돈을 보내는 코드는 없다.

## 8. 재개하면 함수의 어느 부분부터 실행되는가?

중단된 노드는 재개할 때 처음부터 다시 실행된다. 중단 줄 바로 다음부터 Python의 지역 변수까지 그대로 복원하는 방식은 아니다.

```python
def review_step(state):
    print("검토 시작")  # 재개할 때 다시 실행될 수 있음

    decision = interrupt({"question": "승인할까요?"})

    return {"action": decision["action"]}
```

재개된 실행에서는 해당 `interrupt()`가 전달받은 resume 값을 반환하고 다음 코드로 진행한다.

따라서 결제나 게시글 발행처럼 외부 상태를 바꾸는 작업은 승인 검사 뒤에 배치해야 한다. [공식 문서: Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)

## 9. ToolNode와 Agent 연결하기

```python
class MessageState(TypedDict):
    messages: Annotated[list, add_messages]


tools = [send_payment]

payment_llm = ChatGoogleGenerativeAI(
    model=MODEL_NAME
).bind_tools(tools)


def agent_step(state: MessageState):
    response = payment_llm.invoke(state["messages"])
    return {"messages": [response]}


builder = StateGraph(MessageState)

builder.add_node("agent", agent_step)
builder.add_node("tools", ToolNode(tools))

builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)
builder.add_edge("tools", "agent")

payment_graph = builder.compile(
    checkpointer=MemorySaver()
)
```

```plain text
사용자 메시지
→ agent가 다음 행동 결정
   ├─ 도구 호출 없음 → 종료
   └─ 도구 호출 있음 → ToolNode
                         → 도구 실행
                         → agent가 결과 확인
```

`bind_tools()`는 모델에게 사용할 도구 정보를 전달한다. 모델이 도구 호출을 만들면 `ToolNode`가 해당 Python 함수를 실행한다.

`messages`를 State로 관리하는 이유는 사용자 요청, 모델 응답, 도구 실행 결과를 다음 모델 호출에서 함께 사용하기 위해서다.

`add_messages`는 메시지 목록을 병합하며, 같은 메시지 ID에 대한 갱신도 처리한다.

## 10. Tool과 일반 노드의 차이

| 구분 | Tool | 일반 노드 |
|---|---|---|
| 등록 방식 | `@tool`, 도구 목록 | `add_node()` |
| 실행 결정 | 이 예제에서는 LLM의 도구 호출 | 그래프 연결과 라우터 |
| 입력 | 도구 인자 | State |
| 사용 예 | 결제, 게시글 발행 | 계획 생성, 검토, 결과 정리 |

사람이 답변한다는 사실만으로 Tool인지 노드인지 결정되지는 않는다.

```plain text
LLM이 게시글 발행 도구를 선택
→ Tool 안에서 승인 요청

계획을 만들면 반드시 검토
→ planner 다음에 approval 노드 연결
```

`interrupt()`는 두 경우 모두 사용할 수 있다.

## 11. 첫 실행과 재개는 역할이 다르다

```python
config = {
    "configurable": {"thread_id": "payment-example-1"}
}

# 1. 새로운 요청으로 실행
result = payment_graph.invoke(
    {"messages": [("user", "홍길동에게 50000원 결제해줘")]},
    config,
)

# 2. 중단된 경우 검토할 정보 출력
if "__interrupt__" in result:
    request = result["__interrupt__"][0].value
    print(request["question"])
    print(request["args"])

    # 3. 같은 실행에 승인 값을 전달
    result = payment_graph.invoke(
        Command(resume={"action": "approve"}),
        config,
    )

print(result["messages"][-1].text)
```

마지막 출력은 다음 의미다.

```plain text
result["messages"]  # 메시지 목록
[-1]                # 마지막 메시지
.text               # 해당 메시지의 텍스트
```

`Command`는 그래프에 재개, 상태 변경, 이동 등을 전달하는 객체다.

| 사용 | 의미 |
|---|---|
| `Command(resume=...)` | 중단된 `interrupt()`에 답변 전달 |
| `Command(update=...)` | State 갱신 내용 전달 |
| `Command(goto=...)` | 다음 실행 위치 지정 |

이번 계획 검토 예제에서는 조건부 연결로 경로를 선택하므로 `goto`가 필요 없다.

## 12. 실제 서비스에서 input()은 무엇으로 바뀌는가?

노트북에서는 `input()`으로 답변을 받지만, 실제 서비스에서는 화면의 버튼이나 입력창이 그 역할을 한다.

```plain text
1. 프론트에서 작업 요청
2. 서버가 graph.invoke() 실행
3. interrupt 발생
4. 서버가 질문과 thread_id를 응답
5. 프론트에서 승인·수정 화면 표시
6. 사용자가 답변
7. 서버가 같은 thread_id로 Command(resume=...) 실행
8. 결과 응답
```

프론트는 보통 다음과 같은 JSON을 보낸다.

```plain text
{
  "thread_id": "payment-example-1",
  "action": "approve"
}
```

서버가 받은 데이터를 사용해 Python의 `Command`를 만든다.

```python
result = payment_graph.invoke(
    Command(resume={"action": action}),
    {"configurable": {"thread_id": thread_id}},
)
```

사용자가 승인할 때까지 최초 HTTP 요청 하나를 계속 연결해 둘 필요는 없다.

## 13. 게시글 발행과 create_agent

```python
from langchain.agents import create_agent


@tool
def publish_post(title: str, content: str) -> str:
    """사용자가 게시글 발행을 요청할 때 사용한다.

    발행 전에 사용자에게 제목과 내용을 보여주고 승인을 받는다.
    """

    decision = interrupt(
        {
            "question": "이 게시글을 공개하겠습니까?",
            "tool": "publish_post",
            "args": {"title": title, "content": content},
        }
    )

    if decision["action"] == "approve":
        # 실제 게시글 저장·공개 처리는 이 위치에 추가한다.
        return f"게시글 발행 성공: {title}"

    if decision["action"] == "reject":
        return "게시글 발행이 취소되었습니다."

    raise ValueError("올바른 승인 값을 보내야 합니다.")


post_agent = create_agent(
    model=ChatGoogleGenerativeAI(model=MODEL_NAME),
    tools=[publish_post],
    checkpointer=MemorySaver(),
)
```

`create_agent()`를 사용하면 에이전트의 도구 실행 흐름을 직접 모두 연결하는 코드를 줄일 수 있다.

하지만 도구 설명이 필요 없어지는 것은 아니다. 모델이 언제 도구를 사용해야 하는지 이해할 수 있도록 독스트링과 사용자 요청을 구체적으로 작성해야 한다.

도구를 등록했다고 매번 반드시 호출되는 것도 아니다.

## 14. Plan-and-Execute에 검토 단계 추가하기

첨부 파일의 계획 검토에서는 다음 값을 사용한다.

```plain text
"approve"  # 계획 실행
"edit"     # 피드백을 반영해 계획 다시 생성
```

```plain text
START → planner → approval
                    ├─ approve → executor
                    │              ├─ 남은 계획 있음 → executor
                    │              └─ 남은 계획 없음 → END
                    └─ edit → planner → approval
```

검토 노드는 사람의 답변을 받고 State에 저장한다. 라우터는 저장된 값을 읽어서 다음 경로를 고른다.

## 15. 계획 검토에 사용할 State

```python
import operator

from pydantic import BaseModel, Field
from langgraph.graph import END


class SimplePlanState(TypedDict):
    task: str
    plan: list[str]
    completed: Annotated[list[str], operator.add]
    action: str
    feedback: str


class SimplePlan(BaseModel):
    steps: list[str] = Field(
        description="순서대로 수행할 단계"
    )
```

| 필드 | 의미 |
|---|---|
| `task` | 사용자의 원래 요청 |
| `plan` | 앞으로 실행할 계획 |
| `completed` | 완료한 단계의 기록 |
| `action` | 사용자의 승인·수정 결정 |
| `feedback` | 계획에 반영할 수정 의견 |

`SimplePlan`은 LLM 응답 형식이고, `SimplePlanState`는 그래프에서 공유할 데이터 구조다.

## 16. Planner에서 피드백 반영하기

```python
simple_planner = ChatGoogleGenerativeAI(
    model=MODEL_NAME
).with_structured_output(SimplePlan)


def simple_plan_step(state: SimplePlanState):
    task = state["task"]
    feedback = state.get("feedback", "")

    if feedback:
        task += (
            f"\n기존 계획: {state.get('plan', [])}"
            f"\n수정 의견: {feedback}"
            "\n기존 계획을 수정 의견에 맞게 다시 작성하세요."
        )

    result = simple_planner.invoke(
        f"{task}\n실행 계획을 1~3단계로 작성하세요."
    )

    return {"plan": result.steps}
```

```python
return {"plan": result.steps}
```

- 왼쪽 `"plan"`은 State에 저장할 키다.
- 오른쪽 `result.steps`는 LLM이 반환한 계획 목록이다.

피드백을 State에 저장하는 것만으로 LLM이 자동으로 참고하지는 않는다. 위 코드처럼 프롬프트에 넣어 전달해야 한다.

## 17. 검토 노드에서 resume 값 받기

```python
def human_approval(state: SimplePlanState):
    decision = interrupt(
        {
            "question": "이 계획으로 실행할까요?",
            "plan": state["plan"],
        }
    )

    return {
        "action": decision["action"],
        "feedback": decision.get("feedback", ""),
    }
```

외부에서 다음 값을 보내면,

```python
Command(
    resume={
        "action": "edit",
        "feedback": "온라인 회의 기준으로 바꿔줘.",
    }
)
```

함수 안에서는 다음과 같이 받는다.

```plain text
decision = {
    "action": "edit",
    "feedback": "온라인 회의 기준으로 바꿔줘.",
}
```

노드의 `return`을 통해 State가 갱신된다.

```plain text
{
    "action": "edit",
    "feedback": "온라인 회의 기준으로 바꿔줘.",
}
```

이 구조에서는 피드백을 `resume`에 넣고 노드가 State로 옮기므로, 외부에서 별도로 `Command(update=...)`를 사용할 필요가 없다.

## 18. 검토 라우터와 실행 라우터

```python
def route_human_approval(state: SimplePlanState):
    action = state["action"]

    if action not in {"approve", "edit"}:
        raise ValueError("approve 또는 edit가 필요합니다.")

    return action


def route_execution(state: SimplePlanState):
    return "continue" if state["plan"] else "end"
```

두 함수는 확인하는 값이 다르다.

| 라우터 | 확인하는 값 | 선택 |
|---|---|---|
| `route_human_approval` | 사람의 결정 | 실행 또는 계획 수정 |
| `route_execution` | 남은 계획 | 다음 단계 또는 종료 |

`"end"`라는 문자열 자체가 종료 명령인 것은 아니다. 그래프에서 `"end"`를 `END`에 연결했기 때문에 종료된다.

## 19. Executor는 실제로 무엇을 하는가?

```python
def simple_execute_step(state: SimplePlanState):
    step = state["plan"][0]

    return {
        "plan": state["plan"][1:],
        "completed": [f"완료: {step}"],
    }
```

이 실습의 Executor는 실제 회의 준비나 주문을 수행하지 않는다. 첫 계획을 꺼내 완료 문구를 기록하는 단순화된 예제다.

```plain text
실행 전 plan: [A, B, C]

첫 실행
plan: [B, C]
completed: ["완료: A"]

두 번째 실행
plan: [C]
completed: ["완료: A", "완료: B"]
```

- `[0]`: 현재 실행할 첫 단계
- `[1:]`: 첫 단계를 제외한 나머지 목록
- `operator.add`: 새 완료 기록을 기존 목록 뒤에 누적

이 함수는 계획이 적어도 한 개 있다는 전제다. 빈 계획도 허용하려면 Executor로 들어가기 전에 확인하는 처리가 필요하다.

## 20. 그래프 연결하기

```python
base_builder = StateGraph(SimplePlanState)

base_builder.add_node("planner", simple_plan_step)
base_builder.add_node("approval", human_approval)
base_builder.add_node("executor", simple_execute_step)

base_builder.add_edge(START, "planner")
base_builder.add_edge("planner", "approval")

base_builder.add_conditional_edges(
    "approval",
    route_human_approval,
    {
        "approve": "executor",
        "edit": "planner",
    },
)

base_builder.add_conditional_edges(
    "executor",
    route_execution,
    {
        "continue": "executor",
        "end": END,
    },
)

hitl_graph = base_builder.compile(
    checkpointer=MemorySaver()
)
```

`planner → executor` 직접 연결은 추가하지 않는다. 검토 뒤에 실행하려면 `planner → approval`을 거쳐야 한다.

함수를 수정했다면 이 그래프 구성과 컴파일 코드도 다시 실행한다.

## 21. 요청부터 수정·승인까지 한 번에 실행하기

다음은 노트북의 분리된 실행 셀을 반복 입력으로 묶은 보충 예제다.

```python
config = {
    "configurable": {
        "thread_id": "plan-input-1",
    }
}

task = input("어떤 일을 계획할까요?: ")

result = hitl_graph.invoke(
    {
        "task": task,
        "completed": [],
        "feedback": "",
    },
    config,
)

while "__interrupt__" in result:
    request = result["__interrupt__"][0].value

    print("\n" + request["question"])

    for i, step in enumerate(request["plan"], start=1):
        print(f"{i}. {step}")

    answer = input(
        "승인하면 y, 수정하려면 n: "
    ).strip().lower()

    if answer == "y":
        decision = {"action": "approve"}

    elif answer == "n":
        feedback = input("수정 의견: ")
        decision = {
            "action": "edit",
            "feedback": feedback,
        }

    else:
        print("y 또는 n을 입력하세요.")
        continue

    result = hitl_graph.invoke(
        Command(resume=decision),
        config,
    )

print("\n실행 결과")
print(*result["completed"], sep="\n")
```

`while`을 사용하는 이유는 수정이 한 번으로 끝난다는 보장이 없기 때문이다.

```plain text
최초 계획 → 수정 요청
→ 수정된 계획 → 다시 수정 요청
→ 다시 수정된 계획 → 승인
→ 실행 완료
```

`print(*result["completed"], sep="\n")`은 완료 목록을 펼쳐서 각 항목을 한 줄씩 출력한다.

## 22. 이전 실행 결과까지 쌓이는 이유

같은 `thread_id`를 다시 사용하면 저장된 State를 이어서 사용할 수 있다.

특히 `completed`에는 `operator.add`가 있으므로 새로운 기록이 이전 기록 뒤에 추가된다.

```plain text
이전 실행: ["완료: 회의 안건 작성"]
새 실행:   ["완료: 야식 메뉴 선정"]

누적 결과:
["완료: 회의 안건 작성", "완료: 야식 메뉴 선정"]
```

새로운 독립 실습은 새로운 ID로 시작한다.

```python
config = {
    "configurable": {"thread_id": "plan-input-2"}
}
```

이미 기록이 있는 실행에 `"completed": []`를 전달해도 `operator.add`에서는 기존 목록과 빈 목록을 더하므로 기록이 초기화되지 않는다.

또한 `compile(checkpointer=MemorySaver())`를 다시 실행하면 새로운 메모리 저장소를 사용한다는 점도 구분해야 한다.

## 23. 보충 실습: 부족한 정보 질문하기

파일의 마지막 문제는 다음 흐름을 요구한다.

```plain text
사용자 요청
→ LLM이 추가 질문과 선택지 생성
→ 사용자 답변 대기
→ 답변 수집
→ 답변 내용 정리
→ END
```

여행 요청을 받더라도 이 실습의 마지막 작업은 여행 일정 작성이 아니라 **사용자가 제공한 정보 정리**다.

질문 생성과 답변 대기는 별도 노드로 나눈다. 같은 노드에서 LLM 호출 뒤 `interrupt()`를 사용하면 재개 시 LLM 호출도 다시 실행되어, 사용자에게 보여줬던 질문과 달라질 수 있기 때문이다.

## 24. 질문 형식과 State 정의하기

```python
class Question(BaseModel):
    id: str
    question: str
    choices: list[str]


class ListQuestion(BaseModel):
    questions: list[Question]


class InfoState(TypedDict):
    request: str
    questions: list[dict]
    answer: dict[str, str]
    summary: str


info_llm = ChatGoogleGenerativeAI(model=MODEL_NAME)

question_llm = info_llm.with_structured_output(
    ListQuestion
)
```

```python
Question(
    id="q1",
    question="누구와 여행하나요?",
    choices=["혼자", "친구", "가족"],
)
```

- `Question`: 질문 한 개의 형식
- `ListQuestion`: 질문 여러 개를 담는 형식
- `InfoState`: 그래프가 공유할 요청, 질문, 답변, 정리 결과

구조화 출력은 LLM의 질문 생성 결과에 적용된다. 사용자가 나중에 보내는 답변까지 자동으로 검증하는 설정은 아니다.

## 25. 질문 생성 후 딕셔너리로 저장하기

```python
def generate_questions(state: InfoState):
    result = question_llm.invoke(
        f"""
사용자 요청:
{state["request"]}

요청을 처리하기 위해 필요한 추가 질문을 1~3개 작성하세요.
각 질문에는 객관식 선택지 3개를 제공하세요.
선택지 외에 사용자가 직접 답할 수도 있습니다.
질문 id는 q1, q2, q3처럼 서로 다르게 작성하세요.
"""
    )

    questions = [
        q.model_dump()
        for q in result.questions
    ]

    return {"questions": questions}
```

```python
questions = [q.model_dump() for q in result.questions]
```

이 코드는 다음 순서로 읽는다.

1. `result.questions`에서 질문 객체를 하나씩 가져온다.
2. `q.model_dump()`로 객체를 딕셔너리로 변환한다.
3. 변환된 딕셔너리를 목록에 담는다.

```plain text
[
    {
        "id": "q1",
        "question": "누구와 여행하나요?",
        "choices": ["혼자", "친구", "가족"],
    }
]
```

딕셔너리로 바꾸면 `q["question"]`처럼 읽거나 화면에 전달하기 편하다.

## 26. 답변을 resume으로 받아 저장하기

승인 예제처럼 바깥 키를 정해서 전달한다.

```python
def ask_answers(state: InfoState):
    response = interrupt(
        {
            "message": "추천을 위해 추가 정보가 필요합니다.",
            "questions": state["questions"],
        }
    )

    return {"answer": response["answer"]}
```

외부에서 보내는 모양은 다음과 같다.

```python
Command(
    resume={
        "answer": {
            "q1": "친구와 함께",
            "q2": "자연에서 쉬는 여행",
        }
    }
)
```

```plain text
resume 전체
→ response에 들어옴

response["answer"]
→ 질문별 답변 딕셔너리를 꺼냄

return {"answer": ...}
→ State의 answer에 저장
```

`"answer"`라는 이름도 개발자가 정한 약속이다. 보내는 코드와 받는 코드의 모양을 맞춰야 한다.

## 27. 답변 정리와 그래프 연결하기

```python
def summarize_answer(state: InfoState):
    result = info_llm.invoke(
        f"""
사용자의 원래 요청:
{state["request"]}

질문 목록:
{state["questions"]}

사용자 답변:
{state["answer"]}

질문 id에 맞춰 사용자가 제공한 정보를 정리하세요.
불명확하거나 질문과 관계없는 답변은 명확하지 않다고 표시하세요.
사용자가 말하지 않은 정보는 추측하지 마세요.
원래 요청을 수행하지 말고 답변만 정리하세요.
"""
    )

    return {"summary": result.text}


info_builder = StateGraph(InfoState)

info_builder.add_node(
    "generate_questions", generate_questions
)
info_builder.add_node("ask_answers", ask_answers)
info_builder.add_node(
    "summarize_answer", summarize_answer
)

info_builder.add_edge(START, "generate_questions")
info_builder.add_edge("generate_questions", "ask_answers")
info_builder.add_edge("ask_answers", "summarize_answer")
info_builder.add_edge("summarize_answer", END)

info_graph = info_builder.compile(
    checkpointer=MemorySaver()
)
```

답변 정리가 이 실습의 목표이므로 `summarize_answer` 뒤에 `END`를 연결한다.

## 28. 실제 질문 ID를 사용해 답변 보내기

```python
config = {
    "configurable": {"thread_id": "info-question-1"}
}

result = info_graph.invoke(
    {
        "request": input("어떤 도움이 필요한가요?: "),
    },
    config,
)

request = result["__interrupt__"][0].value
print(request["message"])

answers = {}

for q in request["questions"]:
    print("\n" + q["question"])

    for i, choice in enumerate(q["choices"], start=1):
        print(f"{i}. {choice}")

    # 선택지의 내용을 적거나 자유롭게 직접 답변한다.
    answers[q["id"]] = input("답변: ")

result = info_graph.invoke(
    Command(resume={"answer": answers}),
    config,
)

print(result["summary"])
```

다음 줄이 질문과 답변을 연결하는 부분이다.

```python
answers[q["id"]] = input("답변: ")
```

현재 질문의 ID가 `"q1"`이라면,

```plain text
answers["q1"] = "친구와 함께"
```

처럼 저장된다.

ID에 따라 자동 연결되는 특별한 기능이 있는 것이 아니라, 같은 ID를 딕셔너리의 키로 사용해서 연결 관계를 표현한 것이다.

## 29. 이상한 답변을 보내면 어떻게 되는가?

현재 구조에서는 사용자가 입력한 답변이 그대로 전달된다.

```plain text
{
    "answer": {
        "q1": "아무거나ㅋㅋ",
        "q2": "모르겠음",
    }
}
```

LLM이 그 내용을 정리할 수는 있지만, 올바른 답변인지 자동으로 보장되지는 않는다.

검증은 나누어 생각해야 한다.

| 검증 | 예시 |
|---|---|
| 형식 검증 | `answer`가 딕셔너리인가? |
| ID 검증 | 질문 ID와 답변 ID가 맞는가? |
| 값 검증 | 답변이 비어 있지 않은가? |
| 내용 검토 | 실제 질문에 대한 답인가? |

ID 확인 예시는 다음과 같다.

```python
question_ids = {
    q["id"] for q in request["questions"]
}

if set(answers) != question_ids:
    raise ValueError("질문 ID와 답변 ID가 맞지 않습니다.")
```

이 검사는 ID만 확인한다. `"누구와 여행하나요?"`에 `"파란색"`이라고 답한 것까지 판별하지는 않는다.

또한 직접 답변을 허용한 실습이므로 선택지와 문장이 다르다는 이유만으로 잘못된 답변은 아니다.

## 30. 실습 중 헷갈렸던 부분과 오류

### `KeyError: '__interrupt__'`

`result`에 중단 정보가 없는데 바로 꺼내면 발생한다.

```python
if "__interrupt__" in result:
    print(result["__interrupt__"][0].value)
else:
    print(result)
```

도구 호출이 없었다면 Tool 내부의 `interrupt()`에도 도달하지 않는다. 실제 응답과 도구 호출 여부를 확인해야 한다.

### Checkpointer는 있는데 config를 전달하지 않은 경우

첨부 파일에는 다음과 같은 이전 실행 셀이 남아 있다.

```python
result = hitl_graph.invoke(
    {"task": "팀 회의 준비하기"}
)
```

현재처럼 Checkpointer를 사용하는 구성에서는 실행을 구분하는 설정을 함께 전달해야 한다.

```python
result = hitl_graph.invoke(
    {"task": "팀 회의 준비하기"},
    config,
)
```

### `get()`을 대괄호로 호출한 경우

```python
# 잘못된 사용
decision.get["feedback"]

# 올바른 사용
decision.get("feedback", "")
```

### 노드와 라우터의 반환값을 혼동한 경우

```python
# 노드: State 갱신 내용 반환
return {"action": decision["action"]}

# 라우터: 경로 이름 반환
return state["action"]
```

라우터였던 함수를 노드로 바꾸면 State를 갱신하는 반환값과 별도의 경로 선택 함수가 필요할 수 있다.

### 함수만 수정하고 그래프를 다시 만들지 않은 경우

```plain text
import·State·함수 정의
→ 노드 등록과 연결
→ compile
→ invoke
```

노트북에서는 셀 실행 순서가 중요하다. 같은 `graph`, `llm`, `State` 이름을 여러 실습에서 사용하면 다른 실습의 객체가 남아 있을 수도 있다.

## 31. 이번 학습에서 기억할 점

- HITL은 승인뿐 아니라 수정 의견과 추가 정보 입력에도 사용한다.
- `interrupt()`는 LangGraph가 제공하는 중단 함수다.
- 중단할 때 보여주는 값과 재개할 때 받는 값은 다르다.
- `Command(resume=...)`의 값이 `interrupt()`의 반환값이 된다.
- `action`, `approve`, `edit` 같은 이름은 개발자가 정한 약속이다.
- `"__interrupt__"`는 반환된 중단 정보를 읽는 키다.
- Checkpointer는 상태를 저장하고 `thread_id`는 실행 문맥을 구분한다.
- 재개할 때 중단된 노드가 처음부터 다시 실행될 수 있다.
- 검토 노드는 답변을 저장하고 라우터는 다음 경로를 고른다.
- State에 피드백이 있어도 LLM 프롬프트에 전달해야 반영된다.
- 같은 실행의 완료 목록은 Reducer에 따라 누적될 수 있다.
- 구조화 출력과 사용자 답변 검증은 별개다.
- 함수 수정 후 그래프 구성과 컴파일도 다시 실행한다.

## 32. 헷갈린 점

처음에는 `interrupt()`에 `"action"`을 넣지 않았는데 `decision["action"]`을 읽는 것이 이상했다. 나중에 외부에서 `Command(resume={"action": "approve"})`를 보내면 그 딕셔너리가 `decision`에 들어온다는 것을 이해했다.

Checkpointer가 있으니 State를 따로 만들 필요가 없는지도 헷갈렸다. State는 노드들이 공유할 데이터 구조이고, Checkpointer는 그 실행 상태를 저장하는 역할이었다.

승인 코드를 Tool 안에 넣는 경우와 일반 노드로 만드는 경우도 혼동했다. LLM이 선택하는 게시글 발행 기능에는 Tool을 사용했고, 계획 생성 뒤 반드시 검토해야 하는 흐름에는 검토 노드를 연결했다.

피드백을 `resume`에 넣는 것과 State에 저장하는 것도 같은 과정이라고 생각했다. 실제로는 `resume` 값이 `interrupt()`의 반환값으로 들어오고, 노드가 그 값을 `return`해야 State에 반영되는 구조였다.

추가 질문에서는 질문 ID만 같으면 시스템이 알아서 답변을 연결해 준다고 생각했다. 실제로는 코드에서 질문 ID를 답변 딕셔너리의 키로 사용했고, 그 질문과 답변을 정리 함수에 함께 전달했다.

이번에는 코드를 외우기보다 **어떤 값을 보내고, 어디에서 받고, 어디에 저장하며, 어떤 조건으로 다음 단계에 가는지** 나누어 보는 것이 흐름을 이해하는 데 도움이 됐다.
