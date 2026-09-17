# [TIL] Subgraph, Supervisor, Handoff로 멀티 에이전트 구조 만들기

오늘은 LangGraph에서 여러 에이전트를 역할별로 분리하고 연결하는 방법을 학습했다.

어제까지는 Human-in-the-Loop, Time Travel, Agentic RAG를 통해 하나의 그래프 내부에서 실행을 제어하는 방법을 살펴봤다. 오늘은 여기서 더 나아가 큰 작업을 여러 그래프로 나누는 Subgraph, 필요한 에이전트를 선택하는 Supervisor, 작업을 다른 에이전트로 넘기는 Handoff를 학습했다.

현재 진행 중인 멀티 에이전트 과제에도 오늘 배운 내용을 적용하고 있다.

```plain text
Subgraph
큰 그래프 내부에 작은 그래프를 만들어 역할별 흐름을 분리한다.

Supervisor
사용자 요청을 분석해 실행할 에이전트와 실행 순서를 결정한다.

Handoff
현재 에이전트가 처리하기 어려운 작업을 다른 에이전트에게 전달한다.
```

## 1. Subgraph

Subgraph는 하나의 큰 그래프 안에서 특정 기능을 담당하는 작은 그래프다.

모든 기능을 하나의 그래프에 넣으면 State와 Node가 많아지고 연결 관계도 복잡해진다. 기능이나 역할별로 그래프를 분리하면 각 부분을 독립적으로 구현하고 테스트할 수 있다.

```plain text
메인 그래프
START
  ↓
설명 생성
  ↓
퀴즈 Subgraph
  ↓
END

퀴즈 Subgraph
START
  ↓
문제 생성
  ↓
END
```

### Subgraph를 사용하는 이유

- 복잡한 그래프를 역할별로 나눌 수 있다.
- 각각의 그래프를 독립적으로 테스트할 수 있다.
- 동일한 Subgraph를 다른 그래프에서도 재사용할 수 있다.
- 에이전트별 State와 실행 흐름을 분리할 수 있다.

## 2. 부모 그래프와 Subgraph가 같은 State를 사용하는 경우

부모 그래프와 Subgraph가 같은 State를 사용하면 컴파일된 Subgraph를 일반 Node처럼 등록할 수 있다.

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class LearningState(TypedDict):
    topic: str
    explanation: str
    questions: list[str]


def explain_node(state: LearningState):
    topic = state["topic"]

    return {
        "explanation": f"{topic}에 대한 설명"
    }


def make_questions_node(state: LearningState):
    explanation = state["explanation"]

    return {
        "questions": [
            f"{explanation}의 핵심 개념은 무엇인가요?",
            "이 개념을 사용하는 이유는 무엇인가요?"
        ]
    }
```

먼저 퀴즈를 생성하는 Subgraph를 만든다.

```python
quiz_builder = StateGraph(LearningState)

quiz_builder.add_node(
    "make_questions",
    make_questions_node
)

quiz_builder.add_edge(START, "make_questions")
quiz_builder.add_edge("make_questions", END)

quiz_graph = quiz_builder.compile()
```

컴파일된 `quiz_graph`는 메인 그래프의 Node로 등록할 수 있다.

```python
main_builder = StateGraph(LearningState)

main_builder.add_node("explain", explain_node)
main_builder.add_node("quiz", quiz_graph)

main_builder.add_edge(START, "explain")
main_builder.add_edge("explain", "quiz")
main_builder.add_edge("quiz", END)

learning_graph = main_builder.compile()
```

실행 예시는 다음과 같다.

```python
result = learning_graph.invoke({
    "topic": "LangGraph Subgraph",
    "explanation": "",
    "questions": []
})

print(result)
```

예상 결과는 다음과 같다.

```plain text
{
    "topic": "LangGraph Subgraph",
    "explanation": "LangGraph Subgraph에 대한 설명",
    "questions": [
        "LangGraph Subgraph에 대한 설명의 핵심 개념은 무엇인가요?",
        "이 개념을 사용하는 이유는 무엇인가요?"
    ]
}
```

## 3. 부모 그래프와 Subgraph의 State가 다른 경우

부모 그래프와 Subgraph가 서로 다른 State를 사용하는 경우에는 중간 Node에서 입력과 출력을 변환해야 한다.

```python
class LearningState(TypedDict):
    topic: str
    explanation: str
    questions: list[str]


class QuizState(TypedDict):
    source_text: str
    questions: list[str]
```

부모 State의 `explanation`을 Subgraph의 `source_text`로 전달한다.

```python
def call_quiz_subgraph(state: LearningState):
    result = quiz_graph.invoke({
        "source_text": state["explanation"],
        "questions": []
    })

    return {
        "questions": result["questions"]
    }
```

여기서 중요한 점은 Subgraph의 State가 부모 그래프의 State에 자동으로 합쳐지지 않는다는 것이다.

Subgraph를 실행한 Node가 결과를 직접 꺼내서 부모 State의 Key로 반환해야 한다.

```plain text
부모 State의 explanation
  ↓
Subgraph State의 source_text로 변환
  ↓
Subgraph 실행
  ↓
Subgraph의 questions 반환
  ↓
부모 State의 questions에 저장
```

## 4. Subgraph 실행 과정 확인하기

`stream()`의 `subgraphs` 옵션을 사용하면 Subgraph의 내부 실행 과정까지 확인할 수 있다.

```python
for chunk in learning_graph.stream(
    {
        "topic": "LangGraph",
        "explanation": "",
        "questions": []
    },
    subgraphs=False
):
    print(chunk)
```

`subgraphs=False`는 메인 그래프의 실행 결과를 중심으로 출력한다.

```python
for chunk in learning_graph.stream(
    {
        "topic": "LangGraph",
        "explanation": "",
        "questions": []
    },
    subgraphs=True
):
    print(chunk)
```

`subgraphs=True`를 사용하면 Subgraph 내부의 Node 실행 결과까지 확인할 수 있다.

## 5. Supervisor

Supervisor는 사용자 요청을 분석하고 어떤 에이전트를 실행할지 결정하는 역할을 한다.

이번 실습과 과제에서는 다음과 같은 전문 에이전트를 사용한다.

| 에이전트 | 역할 |
|---|---|
| Research | 필요한 정보와 근거 조사 |
| Planning | 조사 결과를 바탕으로 계획 작성 |
| Review | 작성된 결과의 문제점을 검토하고 수정 |

전체 흐름은 다음과 같다.

```plain text
사용자 요청
  ↓
Supervisor
  ↓
필요한 에이전트 선택
  ↓
Research → Planning → Review
  ↓
최종 결과
```

사용자가 조사만 요청했다면 Research Agent만 실행할 수 있다.

계획 작성만 요청했다면 Planning Agent를 실행하고, 조사부터 계획 작성과 검토까지 요청했다면 세 에이전트를 순서대로 실행한다.

## 6. 구조화된 출력으로 에이전트 선택하기

Supervisor의 응답을 일반 문자열로 받으면 실행할 에이전트 목록을 다시 파싱해야 한다.

Pydantic 모델과 `with_structured_output()`을 사용하면 필요한 값을 정해진 형태로 받을 수 있다.

```python
from typing import Literal
from pydantic import BaseModel, Field


class SupervisorDecision(BaseModel):
    agents: list[
        Literal["research", "planning", "review"]
    ] = Field(
        description="실행할 전문가 목록"
    )

    reason: str = Field(
        description="전문가를 선택한 이유"
    )
```

```python
llm_with_supervisor_output = (
    llm.with_structured_output(SupervisorDecision)
)
```

Supervisor Node는 사용자 요청을 분석한 뒤 실행할 에이전트 목록과 선택 이유를 State에 저장한다.

```python
from langchain_core.messages import HumanMessage, SystemMessage


def supervisor_node(state: SupervisorState):
    query = state["query"]

    result = llm_with_supervisor_output.invoke([
        SystemMessage(content=supervisor_prompt),
        HumanMessage(content=query),
    ])

    return {
        "selected_agents": result.agents,
        "supervisor_reason": result.reason
    }
```

예를 들어 다음과 같은 요청이 들어왔다고 가정한다.

```plain text
대학가에서 예산 5천만 원과 인력 2명으로 운영할 수 있는
음식점을 조사하고 사업 계획을 작성한 뒤 검토해 줘.
```

Supervisor의 결과는 다음과 같은 형태가 될 수 있다.

```plain text
{
    "agents": [
        "research",
        "planning",
        "review"
    ],
    "reason": (
        "시장 조사가 필요하고, 조사 결과를 바탕으로 "
        "계획을 작성한 뒤 최종 검토해야 합니다."
    )
}
```

## 7. 선택된 에이전트 순서대로 실행하기

Supervisor가 선택한 에이전트는 `run_agents_node`에서 순서대로 실행한다.

앞 단계에서 생성한 결과는 `current_context`에 추가되어 다음 에이전트에게 전달된다.

```python
def run_agents_node(state: SupervisorState):
    query = state["query"]
    selected_agents = state["selected_agents"]

    results = {}
    current_context = query

    for agent_name in selected_agents:
        print(f"실행 에이전트: {agent_name}")

        if agent_name == "research":
            research_result = research.invoke({
                "query": current_context
            })

            results["research_result"] = research_result

            current_context += (
                f"\n\n[조사 결과]\n{research_result}"
            )

        elif agent_name == "planning":
            planning_result = planning.invoke({
                "query": current_context
            })

            results["planning_result"] = planning_result

            current_context += (
                f"\n\n[기획 결과]\n{planning_result}"
            )

        elif agent_name == "review":
            review_result = review.invoke({
                "query": current_context
            })

            results["review_result"] = review_result

            current_context += (
                f"\n\n[검토 결과]\n{review_result}"
            )

    results["final_result"] = current_context

    return results
```

이 구조에서는 각 에이전트가 독립적으로 실행되면서도 이전 단계의 결과를 참고할 수 있다.

```plain text
Research
정보 조사
  ↓
Planning
사용자 요청 + 조사 결과를 바탕으로 계획 작성
  ↓
Review
사용자 요청 + 조사 결과 + 계획을 함께 검토
```

## 8. Supervisor Graph 구성

현재 과제에서는 `create_agent` 기반 Supervisor 대신 `StateGraph`를 이용해 Supervisor를 직접 구성하고 있다.

```plain text
START
  ↓
supervisor
  ↓
run_agents
  ↓
END
```

```python
from langgraph.graph import StateGraph, START, END


builder = StateGraph(SupervisorState)

builder.add_node(
    "supervisor",
    supervisor_node
)

builder.add_node(
    "run_agents",
    run_agents_node
)

builder.add_edge(START, "supervisor")
builder.add_edge("supervisor", "run_agents")
builder.add_edge("run_agents", END)

supervisor_graph = builder.compile()
```

Supervisor State는 다음과 같이 구성했다.

```python
class SupervisorState(TypedDict):
    query: str
    selected_agents: list[str]
    supervisor_reason: str
    research_result: str
    planning_result: str
    review_result: str
    final_result: str
```

그래프를 실행할 때는 `SupervisorState`에 정의한 `query`를 전달한다.

```python
result = supervisor_graph.invoke(
    {
        "query": user_input
    },
    config=config,
)

print(result["final_result"])
```

`create_agent.invoke()`는 일반적으로 `messages`를 전달하지만, 직접 만든 `StateGraph`는 State에 정의된 Key를 전달한다.

현재 과제의 State에서는 `query`를 사용하므로 다음과 같이 실행한다.

```python
{
    "query": user_input
}
```

## 9. Handoff

Handoff는 현재 에이전트가 작업을 다른 에이전트에게 넘기는 방식이다.

예를 들어 고객의 문의를 Sales Agent가 처리하다가 기술적인 문제가 확인되면 Support Agent에게 작업을 넘길 수 있다.

```plain text
사용자 요청
  ↓
Sales Agent
  ↓
기술 지원이 필요한지 판단
  ↓
Support Agent로 Handoff
  ↓
Support Agent가 이어서 처리
```

LangGraph에서는 `Command`를 사용해 State를 수정하면서 다음 실행 위치도 지정할 수 있다.

```python
from langgraph.types import Command


def review_node(state: ReportState):
    if state["approved"]:
        return Command(
            update={
                "status": "approved"
            },
            goto="publish"
        )

    return Command(
        update={
            "status": "revision_required"
        },
        goto="writer"
    )
```

`Command`에서 사용하는 주요 값은 다음과 같다.

| 속성 | 역할 |
|---|---|
| update | State에 저장하거나 수정할 값 |
| goto | 다음에 실행할 Node |

```plain text
현재 State 확인
  ↓
Node에서 작업 수행
  ↓
Command의 update로 State 변경
  ↓
Command의 goto로 다음 Node 결정
```

## 10. Edge와 Command의 차이

일반 Edge는 그래프를 만들 때 이동 경로가 정해진다.

```python
builder.add_edge("writer", "review")
```

`Command`는 Node를 실행한 결과에 따라 이동할 위치를 결정한다.

```python
return Command(
    update={
        "status": "revision_required"
    },
    goto="writer"
)
```

이를 이용하면 검토 결과에 따라 승인 또는 재작성을 선택하는 흐름을 만들 수 있다.

```plain text
writer
  ↓
review
  ├─ 승인 → publish
  └─ 수정 필요 → writer
```

## 11. 멀티 에이전트 과제 진행 상황

현재 과제에서는 조사, 기획, 검토 역할을 분리한 멀티 에이전트 보고서 시스템을 만들고 있다.

목표 흐름은 다음과 같다.

```plain text
main.py
  ↓
supervisor_graph
  ↓
supervisor_node
  ↓
run_agents_node
  ↓
research / planning / review Tool
  ↓
각 Tool 내부에서 자신의 Subgraph 실행
```

현재까지 진행한 내용은 다음과 같다.

- Supervisor State 정의
- 사용자 요청을 분석하는 Supervisor Node 구현
- 요청에 필요한 에이전트 목록 선택
- 선택된 에이전트를 순서대로 실행
- 이전 단계 결과를 다음 단계 Context에 추가
- `while`문을 사용한 반복 입력
- 이전 실행 결과를 후속 요청에 전달
- Research Agent에 Plan-and-Execute 패턴 적용

## 12. Research Agent에 Plan-and-Execute 적용

Research Agent는 사용자의 요청을 받자마자 검색하지 않는다.

먼저 조사 계획을 세우고, 계획에 따라 조사를 실행한 다음, 조사 내용을 최종 결과로 정리한다.

```plain text
START
  ↓
research_plan_node
  ↓
research_execute_node
  ↓
research_summarize_node
  ↓
END
```

Research State는 다음과 같이 구성했다.

```python
class ResearchState(TypedDict):
    query: str
    research_plan: str
    research_raw: str
    research_result: str
```

각 State 값의 역할은 다음과 같다.

| State Key | 역할 |
|---|---|
| query | 사용자가 입력한 요청 |
| research_plan | 조사할 항목과 조사 순서 |
| research_raw | 검색 및 조사 원본 |
| research_result | 정리된 최종 조사 결과 |

조사 계획을 생성하는 Node는 다음과 같다.

```python
def research_plan_node(state: ResearchState):
    query = state["query"]

    result = llm.invoke([
        SystemMessage(
            content=research_plan_prompt
        ),
        HumanMessage(
            content=query
        ),
    ])

    return {
        "research_plan": result.text
    }
```

계획을 바탕으로 실제 조사를 수행한다.

```python
def research_execute_node(state: ResearchState):
    query = state["query"]
    research_plan = state["research_plan"]

    result = search_llm.invoke([
        SystemMessage(
            content=research_execute_prompt
        ),
        HumanMessage(
            content=f"""
            사용자 요청:
            {query}

            조사 계획:
            {research_plan}
            """
        ),
    ])

    return {
        "research_raw": result.text
    }
```

마지막으로 조사 내용을 정리한다.

```python
def research_summarize_node(state: ResearchState):
    query = state["query"]
    research_plan = state["research_plan"]
    research_raw = state["research_raw"]

    result = llm.invoke([
        SystemMessage(
            content=research_summarize_prompt
        ),
        HumanMessage(
            content=f"""
            사용자 요청:
            {query}

            조사 계획:
            {research_plan}

            조사 결과:
            {research_raw}
            """
        ),
    ])

    return {
        "research_result": result.text
    }
```

Research Graph는 다음과 같이 연결한다.

```python
builder = StateGraph(ResearchState)

builder.add_node(
    "research_plan_node",
    research_plan_node
)

builder.add_node(
    "research_execute_node",
    research_execute_node
)

builder.add_node(
    "research_summarize_node",
    research_summarize_node
)

builder.add_edge(
    START,
    "research_plan_node"
)

builder.add_edge(
    "research_plan_node",
    "research_execute_node"
)

builder.add_edge(
    "research_execute_node",
    "research_summarize_node"
)

builder.add_edge(
    "research_summarize_node",
    END
)

research_graph = builder.compile()
```

외부에서는 Research Subgraph를 Tool로 호출한다.

```python
@tool
def research(query: str) -> str:
    result = research_graph.invoke({
        "query": query
    })

    return result["research_result"]
```

외부에서는 `research(query)`라는 단순한 형태로 사용하고, 내부에서는 계획, 조사, 요약 단계가 순서대로 실행된다.

## 13. 이전 대화 내용 이어가기

과제 조건에는 같은 실행 세션에서 이전 대화를 기억해야 한다는 내용이 있다.

`while`문만 사용하면 사용자 입력을 반복해서 받을 수는 있지만, 이전 실행 결과가 자동으로 다음 요청에 전달되지는 않는다.

현재는 `last_result` 변수에 이전 결과를 저장한 뒤 다음 요청과 함께 Supervisor에게 전달하고 있다.

```python
last_result = ""

while True:
    user_input = input(
        "요청을 입력하세요: "
    )

    if user_input in ["exit", "종료"]:
        print("시스템을 종료합니다.")
        break

    if last_result:
        query = f"""
        [이전 결과]
        {last_result}

        [새 요청]
        {user_input}
        """
    else:
        query = user_input

    result = supervisor_graph.invoke(
        {
            "query": query
        },
        config=config,
    )

    last_result = result["final_result"]

    print("\n=== 최종 응답 ===")
    print(result["final_result"])
```

다음과 같은 연속 요청을 테스트할 수 있다.

```plain text
첫 번째 요청
예산 5천만 원, 인력 2명으로 덮밥집 운영 계획을 세워줘.

두 번째 요청
방금 계획에서 영업시간만 오후 10시까지로 바꿔줘.
```

두 번째 요청에서도 다음 조건이 유지돼야 한다.

- 예산 5천만 원
- 운영 인력 2명
- 덮밥집
- 기존 계획의 나머지 내용

후속 수정 요청에 최신 정보가 필요하지 않다면 Research Agent를 다시 실행하지 않고 Planning 또는 Review Agent만 선택하도록 Supervisor Prompt에도 조건을 추가했다.

## 14. 과제에 적용할 에이전트 패턴

각 에이전트에는 역할에 맞는 Workflow 패턴을 적용할 예정이다.

| 에이전트 | 적용 패턴 | 실행 흐름 |
|---|---|---|
| Research | Plan-and-Execute | 계획 → 조사 → 요약 |
| Planning | Orchestrator-Worker | 작업 분리 → 개별 작성 → 통합 |
| Review | Reflection | 비판 → 수정 |

Planning Agent의 목표 구조는 다음과 같다.

```plain text
START
  ↓
planning_orchestrator_node
  ↓
planning_worker_node
  ↓
planning_synthesize_node
  ↓
END
```

Review Agent의 목표 구조는 다음과 같다.

```plain text
START
  ↓
review_critique_node
  ↓
review_revise_node
  ↓
END
```

## 15. 헷갈렸던 부분

### while문만 사용하면 이전 대화를 기억하는가?

아니다.

`while`문은 입력을 반복해서 받는 역할만 한다. 이전 결과를 변수에 저장하거나 Checkpointer를 사용해 다음 실행에 전달해야 한다.

### Subgraph의 결과가 메인 State에 자동으로 저장되는가?

아니다.

Subgraph를 호출한 Node에서 필요한 결과를 꺼내 메인 State의 Key로 반환해야 한다.

```python
result = research_graph.invoke({
    "query": query
})

return {
    "research_result": result["research_result"]
}
```

### Node에서는 전체 State를 반환해야 하는가?

아니다.

Node가 새로 생성하거나 수정한 값만 Dictionary 형태로 반환하면 된다.

```python
return {
    "selected_agents": result.agents,
    "supervisor_reason": result.reason
}
```

### 메인 그래프 이미지에서 Subgraph 내부까지 보이는가?

메인 그래프에서는 Subgraph가 하나의 Node처럼 표시된다.

Subgraph의 내부 구조를 확인하려면 Research, Planning, Review Graph를 각각 이미지로 출력해야 한다.

## 16. 다음 작업

- Research Graph 변경 사항 실행 확인
- `research_graph.png` 생성
- Research Graph에 3개 Node가 표시되는지 확인
- `main.py`에서 Supervisor Graph 실행 테스트
- 실제 실행된 에이전트 이름과 실행 순서 출력
- Planning Agent에 Orchestrator-Worker 적용
- Review Agent에 Reflection 적용
- 조사 전용 요청 테스트
- 기획 전용 요청 테스트
- 검토 전용 요청 테스트
- 조사, 기획, 검토가 모두 필요한 복합 요청 테스트
- 후속 요청에서 이전 조건이 유지되는지 확인

## 오늘의 정리

오늘은 Subgraph를 사용해 하나의 큰 그래프를 역할별 그래프로 분리하는 방법을 학습했다.

Supervisor를 사용해 사용자 요청을 분석하고 필요한 에이전트와 실행 순서를 결정했다. 선택된 에이전트는 순서대로 실행되며 앞 단계의 결과가 다음 단계의 입력으로 전달된다.

Handoff 실습에서는 `Command`의 `update`와 `goto`를 사용해 State를 수정하면서 다음 실행 위치를 결정하는 방법을 확인했다.

현재 과제에서는 Supervisor Graph와 Research Agent의 Plan-and-Execute 구조까지 진행했다. 다음 단계에서는 Planning Agent에 Orchestrator-Worker를 적용하고 Review Agent에는 Reflection 구조를 적용할 예정이다.

### 추가 기존 계획

과제를 보고 생각한 것.

조사 / 기획 / 검토 → 에이전트로 구현

최종 결과물

- 필요에 따라 에이전트를 선택할 수 있게
- 앞의 결과를 뒤에 에이전트가 알 수 있게
- 최종 보고서에는 검토에 따른 개선 사항을 반영

에이전트 구성

1. 슈퍼바이저
2. 핸드오프
3. 추가적으로 HITL을 통해 부족한 정보들 추가

내가 할 것 단계

1. Supervisor로 에이전트 기본 뼈대 만들기
2. 완료 후 각자 에이전트에 맞는 검증 과정 넣기
3. 검증 과정 이후 추가적인 정보가 필요할 시 HITL 추가
4. 시간 가용 시 worker, plan으로 세부적으로 조사할 수 있게 변경

```plain text
research: Plan-and-Execute
planning: Orchestrator-Worker
review: Reflection
```
