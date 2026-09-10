# 검색·분류·검증·반복 개선을 LangGraph로 구성하기

이번에는 Retriever를 사용하는 RAG Agent, 요청을 분류하는 Router, 입력과 출력을 검사하는 Guardrails를 살펴봤다.

마지막에는 제품 리뷰 요약을 Reflection과 Evaluator-Optimizer 두 가지 방식으로 구현했다. 특히 작업 흐름을 먼저 적고, 각 노드에서 다음 노드로 넘겨야 하는 값을 기준으로 State를 설계하는 연습을 했다.

> 아래 코드는 노트북의 핵심 부분을 정리한 것이다. 모델과 벡터 저장소는 해당 노트북에서 준비한 것을 사용하며, 리뷰 요약 실습의 입력은 노트북에 있는 `REVIEW`를 사용한다.

## 1. RAG Agent란?

RAG Agent는 Retriever를 Tool로 등록해서, LLM이 검색이 필요한지 판단하고 문서를 검색하도록 만든 Agent다.

이번에 비교한 기본 RAG 파이프라인과의 차이는 검색을 실행하는 시점이다.

| 구분 | 기본 RAG 파이프라인 | RAG Agent |
|---|---|---|
| 검색 실행 | 정해진 단계에서 검색 | LLM이 Tool 호출 여부 판단 |
| Retriever 사용 | 코드에서 직접 호출 | Tool로 등록해서 호출 |
| 후속 질문 | 대화 맥락을 별도로 전달 | Checkpointer를 연결해 대화 유지 가능 |

```
기본 RAG
질문 → 검색 → 검색 결과를 바탕으로 답변

RAG Agent
질문 → LLM 판단
       ├─ 검색 필요 → Retriever Tool → LLM → 답변
       └─ 검색 불필요 → 바로 답변
```

예를 들어 회사 휴가 규정을 물으면 문서 검색이 필요하지만, 방금 받은 답변을 영어로 번역해 달라는 요청은 이전 대화만으로 처리할 수 있다.

## 2. Retriever를 Tool로 만들기

먼저 벡터 저장소를 Retriever로 변환한다.

```python
retriever = vectorstore.as_retriever(
    search_kwargs={"k": 3}
)
```

`k=3`은 검색 결과로 가져올 문서 수를 지정한 것이다. 이후 `create_retriever_tool()`로 Agent가 사용할 Tool을 만든다.

```python
from langchain_core.prompts import PromptTemplate
from langchain_core.tools.retriever import create_retriever_tool

document_prompt = PromptTemplate.from_template(
    "[발행 시기: {year}년 {month}월]\n"
    "[출처: {source}]\n"
    "{page_content}"
)

retriever_tool = create_retriever_tool(
    retriever,
    name="spri_rag_agent_practice",
    description=(
        "2025년 9월, 10월, 11월 SPRi AI Brief에서 "
        "인공지능 산업 동향과 기업 소식을 검색한다. "
        "해당 기간의 AI 기술, 기업, 정책에 관한 질문에 사용한다."
    ),
    document_prompt=document_prompt,
    document_separator="\n\n---\n\n",
)
```

| 설정 | 역할 |
|---|---|
| `name` | Agent가 호출할 Tool 이름 |
| `description` | 어떤 정보가 있으며 언제 사용해야 하는지 설명 |
| `document_prompt` | 검색된 문서 하나를 전달할 텍스트 형식 |
| `document_separator` | 여러 검색 문서 사이의 구분자 |

`page_content`는 문서 본문이고, `year`, `month`, `source`는 문서 metadata에서 가져온다. 따라서 해당 키가 검색되는 모든 문서에 있어야 한다.

metadata가 벡터 저장소에 있다고 해서 전부 자동으로 LLM에게 전달되는 것은 아니다. 답변에 출처나 발행 시기를 사용하려면 전달할 문서 형식에 포함해야 한다.

## 3. 대화형 RAG Agent 구성하기

Retriever Tool과 Checkpointer를 Agent에 연결한다.

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import MemorySaver

rag_agent = create_agent(
    model=llm,
    tools=[retriever_tool],
    checkpointer=MemorySaver(),
    system_prompt="""
AI 산업 동향에 관한 질문에는 spri_rag_agent_practice 도구를 사용한다.
답변은 검색된 자료를 근거로 작성하고 발행 시기와 출처를 포함한다.
검색 결과에 충분한 근거가 없으면 해당 정보를 찾을 수 없다고 답한다.
문서에 없는 내용은 추측하지 않는다.
""",
)
```

호출할 때는 같은 대화를 구분할 `thread_id`를 전달한다.

```python
config = {
    "configurable": {
        "thread_id": "spri-study"
    }
}

result = rag_agent.invoke(
    {
        "messages": [
            ("human", "2025년 9월 AI 산업 동향을 알려줘.")
        ]
    },
    config=config,
)

print(result["messages"][-1].text)
```

같은 `thread_id`로 후속 질문을 보내면 앞선 대화를 이어갈 수 있다.

```python
result = rag_agent.invoke(
    {
        "messages": [
            ("human", "방금 설명에서 가장 중요한 내용만 요약해줘.")
        ]
    },
    config=config,
)

print(result["messages"][-1].text)
```

System Prompt는 문서에 근거한 답변을 유도하는 역할이다. 실제로 근거를 지켰는지는 검색 결과와 답변을 비교해서 확인해야 한다.

## 4. Router: 요청에 맞는 처리 경로 선택하기

Router는 요청을 분류해서 적절한 노드로 보내는 역할을 한다.

```
고객 문의 → 분류
           ├─ 환불 → 환불 처리
           ├─ 배송 → 배송 처리
           ├─ 일반 → 일반 상담
           └─ 불확실 → 추가 질문
```

라우팅 방법은 판단할 조건에 따라 선택한다.

| 방식 | 판단 기준 | 특징 |
|---|---|---|
| Deterministic | 키워드·상태·권한 등 코드 규칙 | 빠르고 결과를 예측하기 쉬움 |
| LLM | 요청의 문맥과 의도 | 표현이 다양해도 분류 가능 |
| Semantic | 대표 문장과의 의미 유사도 | 예시 문장과 임계값 설정이 중요 |
| Hybrid | 여러 방식 조합 | 명확한 요청은 빠르게, 애매한 요청은 추가 판단 |

LLM으로 분류할 때는 목적지를 정해진 값으로 제한한다.

```python
from typing import Literal
from pydantic import BaseModel, Field


class Classification(BaseModel):
    category: Literal[
        "refund", "shipping", "general", "uncertain"
    ]
    reason: str = Field(description="분류 근거 한 문장")


classifier = llm.with_structured_output(Classification)
```

`category`는 실제 분기에 사용하고, `reason`은 왜 해당 경로를 선택했는지 확인하는 데 사용한다. Hybrid 방식에서는 규칙으로 판단하기 어려운 요청만 LLM에게 넘긴다.

```python
def classify_hybrid(state: CustomerState):
    category = deterministic_route(state["query"])

    # 규칙으로 분류가 가능하면 바로 사용한다.
    if category != "uncertain":
        return {
            "category": category,
            "route_source": "rule",
            "reason": "단일 규칙 일치",
        }

    # 애매한 요청은 LLM으로 분류한다.
    try:
        result = llm_route(state["query"])

        return {
            "category": result.category,
            "route_source": "llm",
            "reason": result.reason,
        }

    except Exception:
        return {
            "category": "uncertain",
            "route_source": "fallback",
            "reason": "분류 호출 실패",
        }
```

분류 결과를 State에 저장한 뒤 조건부 Edge로 연결한다.

```python
def route_by_category(state: CustomerState):
    return state["category"]


builder.add_conditional_edges(
    "classify",
    route_by_category,
    {
        "refund": "refund",
        "shipping": "shipping",
        "general": "general",
        "uncertain": "uncertain",
    },
)
```

분류 노드는 State에 저장할 값을 반환하고, 라우팅 함수는 다음 경로를 고를 값을 반환한다.

### Flat과 Hierarchical의 차이

Flat은 한 번에 목적지를 고르고, Hierarchical은 큰 분류를 선택한 뒤 세부 분류로 내려간다.

```
Flat
문의 → Router → 환불 / 배송 / 계정 / 상품

Hierarchical
문의 → 분야 분류 → 주문 → 환불 / 배송
                 → 계정 → 등급 / 회원 정보
```

목적지가 적으면 Flat이 단순하다. 목적지가 많고 분야가 명확하게 나뉘면 Hierarchical로 구성할 수 있다.

## 5. Guardrails: 입력과 출력 검사하기

Guardrails는 Agent가 처리하는 입력, 실행하는 작업, 사용자에게 내보내는 결과를 검사하는 장치다. 이번 노트북에서는 Input Guardrail과 Output Guardrail을 구현했다.

### Input Guardrail

업무 처리 전에 입력을 검사한다.

```
입력 → check_input
       ├─ 안전 → handle_query → 종료
       └─ 위험 → reject → 종료
```

빈 입력과 길이 제한은 LLM을 호출하기 전에 코드로 검사한다.

```python
query = state["query"]

if not query.strip():
    return {
        "is_safe": False,
        "safety_reason": "empty_input",
    }

if len(query) > MAX_INPUT_LENGTH:
    return {
        "is_safe": False,
        "safety_reason": "input_too_long",
    }
```

그다음 맥락을 봐야 하는 입력은 구조화된 결과로 분류한다.

```python
class GuardrailCheck(BaseModel):
    category: Literal[
        "safe", "unsafe", "prompt_injection"
    ]
    reason: str


guardrail_checker = llm.with_structured_output(
    GuardrailCheck
)
```

```python
result = guardrail_checker.invoke([
    SystemMessage(GUARDRAIL_SYSTEM_PROMPT),
    HumanMessage(query),
])

return {
    "is_safe": result.category == "safe",
    "safety_reason": result.reason,
}
```

검사 모델에 오류가 발생하면 요청을 통과시키지 않도록 처리했다. 이를 fail-closed 방식이라고 한다.

### Output Guardrail

생성된 답변을 사용자에게 전달하기 전에 검사한다.

```
generate → validate_output
           ├─ 민감정보 없음 → pass_through
           └─ 민감정보 있음 → sanitize
```

노트북에서는 정규식으로 민감정보 형식을 찾고, 해당 부분을 마스킹했다.

```python
def validate_output(state: OutputState):
    text = state["raw_response"]

    for pattern in PII_PATTERNS:
        if re.search(pattern, text):
            return {"has_sensitive_info": True}

    return {"has_sensitive_info": False}
```

정규식은 정해진 형식을 검사하는 방법이다. 모든 표현이나 우회된 형식을 감지하는 것은 아니므로, 검사 범위에 맞는 테스트가 필요하다.

## 6. 내부 State와 외부 입출력 구분하기

그래프 내부에서는 원본 응답이나 검사 결과가 필요하지만, 사용자에게는 최종 응답만 반환하고 싶을 수 있다. 이때 `input_schema`와 `output_schema`를 사용한다.

```python
class GuardrailInput(TypedDict):
    query: str


class GuardrailOutput(TypedDict):
    final_response: str


builder = StateGraph(
    OutputState,
    input_schema=GuardrailInput,
    output_schema=GuardrailOutput,
)
```

```
외부 입력
query

내부 처리
query, raw_response, has_sensitive_info, final_response

외부 출력
final_response
```

내부 작업에 필요한 State와 외부에 보여줄 결과의 형태를 따로 정하는 것이다.

## 7. State는 어떻게 정할까?

코드부터 작성하기 전에 실행 흐름을 한글로 적었다.

```
리뷰 입력
→ 요약 생성
→ 요약 검토
→ 피드백 작성
→ 피드백을 반영해 다시 요약
→ 종료
```

그다음 각 노드에서 필요한 값과 새로 만드는 값을 확인했다.

| 노드 | 읽는 값 | 저장하는 값 |
|---|---|---|
| `generate` | 원본 리뷰, 이전 요약, 피드백 | 최신 요약 |
| `reflect` | 원본 리뷰, 최신 요약 | 피드백 |
| `should_continue` | 반복 횟수 | 다음 경로를 반환 |

그래서 다음 State가 필요했다.

```python
class ReflectionState(TypedDict):
    review: str
    summary: str
    feedback: str
    iteration: int
```

State를 정할 때는 “이 값이 다음 노드나 최종 출력에서도 필요한가?”를 생각하면 된다. 노드 안에서만 사용하는 프롬프트 같은 값은 일반 변수로 두고, 다음 작업에 필요한 결과는 State에 저장한다.

## 8. messages 누적 방식과 필드 분리 방식

기존 Reflection 예제는 메시지를 하나의 리스트에 모았다.

```python
class ReflectionState(TypedDict):
    messages: Annotated[list, add_messages]
    iteration: int
```

```
원본 요청 → 첫 요약 → 첫 피드백 → 수정 요약 → 다음 피드백
```

이번에는 원본 리뷰, 요약, 피드백을 각각 분리했다.

```python
class ReflectionState(TypedDict):
    review: str
    summary: str
    feedback: str
    iteration: int
```

| 방식 | 보관하는 내용 | 다음 생성에 전달하는 내용 |
|---|---|---|
| messages 누적 | 이전 메시지 기록 | 선택한 대화 기록 |
| 필드 분리 | 원본과 최신 요약·피드백 | 원본, 최신 요약, 최신 피드백 |

어떤 주제인지보다 이전 기록을 얼마나 사용할지에 따라 선택한다. 이메일도 필드로 나눌 수 있고, 제품 리뷰 요약도 메시지를 누적할 수 있다.

또한 State에 저장됐다고 LLM이 자동으로 보는 것은 아니다. `invoke()`에 필요한 값을 직접 넣어줘야 한다.

## 9. Reflection: 요약 생성과 검토 반복하기

Reflection은 생성된 결과에 피드백을 주고 다시 생성하는 구조다.

```
START → generate → 반복 횟수 확인
          ↑          ├─ 최대 횟수 도달 → END
          └─ reflect ←┘ 계속 진행
```

아래 정리에서는 과제 기준인 한 문단, 3~5문장으로 통일했다.

### 요약 생성 노드

```python
def generate(state: ReflectionState):
    system = SystemMessage(content="""
너는 제품 리뷰 요약 전문가다.
장점, 단점, 추천 대상을 균형 있게 포함하라.
원문에 없는 내용을 추가하지 마라.
이전 요약과 피드백이 있으면 반영하여 수정하라.
한 문단, 3~5문장으로 작성하고 요약문만 출력하라.
""")

    response = generator_llm.invoke([
        system,
        HumanMessage(content=(
            f"원본 리뷰:\n{state['review']}\n\n"
            f"이전 요약:\n{state.get('summary', '')}\n\n"
            f"피드백:\n{state.get('feedback', '')}"
        )),
    ])

    return {
        "summary": response.text,
        "iteration": state.get("iteration", 0) + 1,
    }
```

처음에는 요약과 피드백이 없으므로 `get()`으로 빈 문자열을 사용한다.

```python
state.get("feedback", "")
```

이 코드는 값이 없을 때 읽을 기본값을 지정한다. State 자체에 빈 문자열을 저장하는 것은 아니다.

### 검토 노드

```python
def reflect(state: ReflectionState):
    system = SystemMessage(content="""
너는 제품 리뷰 요약 편집자다.
원본 리뷰와 현재 요약을 비교해 개선할 점을 작성하라.

검토 기준:
- 장점과 단점이 포함되었는가?
- 추천 대상이 명확한가?
- 한 문단, 3~5문장인가?
- 원문에 없는 내용이 추가되지 않았는가?

개선이 필요한 항목의 문제점과 수정 방향만 작성하라.
요약문을 직접 다시 작성하지 마라.
""")

    response = reviewer_llm.invoke([
        system,
        HumanMessage(content=(
            f"원본 리뷰:\n{state['review']}\n\n"
            f"현재 요약:\n{state['summary']}"
        )),
    ])

    return {"feedback": response.text}
```

원문과 일치하는지 확인하려면 요약뿐 아니라 원본 리뷰도 전달해야 한다.

### 반복 종료와 그래프 연결

```python
MAX_ITERATIONS = 3


def should_continue(state: ReflectionState):
    if state["iteration"] >= MAX_ITERATIONS:
        return "end"

    return "continue"


builder = StateGraph(ReflectionState)

builder.add_node("generate", generate)
builder.add_node("reflect", reflect)

builder.add_edge(START, "generate")

builder.add_conditional_edges(
    "generate",
    should_continue,
    {
        "continue": "reflect",
        "end": END,
    },
)

builder.add_edge("reflect", "generate")

review_reflection_graph = builder.compile()
```

최대 생성 횟수가 3이면 다음 순서로 실행된다.

```
첫 요약 → 검토 → 두 번째 요약 → 검토 → 세 번째 요약 → 종료
```

마지막 생성 직후 종료하므로 생성은 3번, 검토는 2번 수행한다.

```python
result = review_reflection_graph.invoke({
    "review": REVIEW,
    "iteration": 0,
})

print(result["summary"])
```

최종 요약은 이미 `summary`에 있으므로 별도의 `final_summary` 필드는 필요하지 않다.

## 10. Evaluator-Optimizer와 루브릭

Evaluator-Optimizer는 항목별 평가 결과를 바탕으로 수정 여부를 결정한다.

| 구분 | 이번 Reflection | Evaluator-Optimizer |
|---|---|---|
| 평가 방식 | 자연어 피드백 | 항목별 점수와 근거 |
| 종료 조건 | 최대 생성 횟수 | 모든 항목 통과 또는 최대 평가 횟수 |
| 수정 방향 | 검토 의견 반영 | 미달 항목의 개선 지시 반영 |

루브릭은 평가 항목과 점수 기준을 정리한 채점표다.

```python
class CriterionResult(BaseModel):
    score: int = Field(
        description="1~10점",
        ge=1,
        le=10,
    )
    reason: str = Field(description="점수 근거 한 문장")


class EvaluationResult(BaseModel):
    completeness: CriterionResult = Field(
        description="장점과 단점 각각 1개 이상, 가격 정보 포함 여부"
    )
    accuracy: CriterionResult = Field(
        description="원문과 일치하며 새로운 정보를 추가하지 않았는지"
    )
    conciseness: CriterionResult = Field(
        description="불필요한 반복 없이 3~5문장인지"
    )
    recommendation: CriterionResult = Field(
        description="적합한 사용자와 부적합한 사용자를 모두 언급했는지"
    )
    summary: str = Field(description="전체 평가 요약")
```

`CriterionResult`는 평가 항목 하나의 결과다.

```json
{
    "score": 8,
    "reason": "장점과 단점, 가격 정보가 포함되어 있다."
}
```

`EvaluationResult`는 이런 결과를 항목별로 모은 전체 평가표다. 여기서 `EvaluationResult.summary`는 평가 총평이고, State의 `summary`는 제품 리뷰 요약이다.

## 11. Evaluator-Optimizer State 구성

```python
from operator import add


class EvalOptState(TypedDict):
    review: str
    summary: str
    evaluation: dict
    feedback: str
    iteration: int
    history: Annotated[list, add]
```

| 필드 | 역할 |
|---|---|
| `review` | 원본 리뷰 |
| `summary` | 최신 요약 |
| `evaluation` | 최신 항목별 점수와 근거 |
| `feedback` | 다음 생성에 사용할 개선 지시 |
| `iteration` | 평가 횟수 |
| `history` | 이전 요약·평가·개선 지시 기록 |

모델의 평가 결과를 정해진 형식으로 받도록 설정한다.

```python
MAX_ITERATIONS = 4
PASS_THRESHOLD = 8

EVALUATION_CRITERIA = (
    "completeness",
    "accuracy",
    "conciseness",
    "recommendation",
)

evaluator = evaluator_llm.with_structured_output(
    EvaluationResult
)
```

`EVALUATION_CRITERIA`는 점수를 확인할 항목 이름을 모아 둔 튜플이다. 평가 총평인 `summary`는 점수가 없으므로 포함하지 않는다.

## 12. 요약 생성 → 평가 → 개선 지시 작성

### 요약 생성

최초 실행에서는 원본 리뷰로 요약하고, 다음 실행부터는 기존 요약과 피드백도 사용한다.

```python
def generate_summary(state: EvalOptState):
    review = state["review"]
    summary = state.get("summary", "")
    feedback = state.get("feedback", "")

    prompt = (
        "제품 리뷰를 한 문단, 3~5문장으로 요약해.\n"
        "장점, 단점, 가격과 추천·비추천 대상을 포함해.\n"
        "원문에 없는 내용은 추가하지 마.\n"
        "이전 요약과 개선 지시가 있으면 반영해 수정해.\n"
        "요약문만 출력해.\n\n"
        f"원본 리뷰:\n{review}\n\n"
        f"이전 요약:\n{summary}\n\n"
        f"개선 지시:\n{feedback}"
    )

    response = generator_llm.invoke(prompt)

    return {"summary": response.text}
```

최초 생성과 재생성의 작성 기준이 같으므로 하나의 프롬프트로 합쳤다. 실습에서 작성한 `if feedback` 분기 방식으로 나누어도 된다.

### 요약 평가

```python
def evaluate_summary(state: EvalOptState):
    system = SystemMessage(content="""
너는 제품 리뷰 요약 평가자다.
각 항목을 1~10점으로 평가하고 근거를 한 문장으로 작성하라.

- completeness: 장점과 단점 각각 1개 이상, 가격을 포함하면 8점 이상
- accuracy: 제품 정보가 원문과 일치하고 추가한 사실이 없으면 8점 이상
- conciseness: 반복 없이 한 문단, 3~5문장이면 8점 이상
- recommendation: 적합·부적합 사용자를 원문 근거로 모두 언급하면 8점 이상

기준에 미달한 항목에는 7점 이하를 부여하라.
""")

    result = evaluator.invoke([
        system,
        HumanMessage(content=(
            f"원본 리뷰:\n{state['review']}\n\n"
            f"현재 요약:\n{state['summary']}"
        )),
    ])

    evaluation = result.model_dump()

    evaluation["overall_pass"] = all(
        evaluation[name]["score"] >= PASS_THRESHOLD
        for name in EVALUATION_CRITERIA
    )

    return {
        "evaluation": evaluation,
        "iteration": state.get("iteration", 0) + 1,
    }
```

`model_dump()`는 Pydantic 결과를 딕셔너리로 변환한다. `all()`은 모든 평가 항목이 8점 이상일 때만 `True`를 반환한다. 평균이 8점이어도 한 항목이 7점이면 미통과다.

### 개선 지시 생성

```python
def optimize_summary(state: EvalOptState):
    evaluation = state["evaluation"]

    failed_items = [
        f"- {name}: {evaluation[name]['reason']}"
        for name in EVALUATION_CRITERIA
        if evaluation[name]["score"] < PASS_THRESHOLD
    ]

    response = generator_llm.invoke(
        "현재 요약에서 평가 기준에 미달한 부분의 개선 지시를 작성해.\n"
        "요약을 직접 다시 쓰지 말고 유지할 내용과 바꿀 내용을 알려줘.\n"
        "원문에 없는 제품 정보는 추가하지 마.\n\n"
        f"원본 리뷰:\n{state['review']}\n\n"
        f"현재 요약:\n{state['summary']}\n\n"
        "미달 항목:\n" + "\n".join(failed_items)
    )

    return {
        "feedback": response.text,
        "history": [{
            "summary": state["summary"],
            "evaluation": evaluation,
            "feedback": response.text,
        }],
    }
```

동일한 피드백을 두 곳에 저장하는 이유는 사용 목적이 다르기 때문이다.

```
feedback
→ 다음 generate_summary가 사용할 최신 개선 지시

history
→ 실행 후 이전 요약과 평가, 개선 지시를 확인할 기록
```

`history`는 `add` Reducer를 사용하므로 새 기록만 리스트로 반환한다. 기존 기록까지 다시 반환하면 중복해서 쌓일 수 있다.

## 13. 통과 여부에 따라 반복하기

```python
def should_continue_eval(state: EvalOptState):
    if state["evaluation"]["overall_pass"]:
        return "end"

    if state["iteration"] >= MAX_ITERATIONS:
        return "end"

    return "fail"


builder = StateGraph(EvalOptState)

builder.add_node("generate", generate_summary)
builder.add_node("evaluate", evaluate_summary)
builder.add_node("optimize", optimize_summary)

builder.add_edge(START, "generate")
builder.add_edge("generate", "evaluate")

builder.add_conditional_edges(
    "evaluate",
    should_continue_eval,
    {
        "end": END,
        "fail": "optimize",
    },
)

builder.add_edge("optimize", "generate")

review_eval_opt_graph = builder.compile()
```

```
요약 생성 → 평가
            ├─ 모든 항목 통과 → 종료
            ├─ 최대 평가 횟수 도달 → 종료
            └─ 미통과, 횟수 남음 → 개선 지시 → 다시 요약
```

최대 횟수에 도달해서 종료한 것이 평가 통과를 의미하지는 않는다. 실제 통과 여부는 `evaluation["overall_pass"]`로 확인한다.

## 14. 실행 결과와 history 확인

최종 결과를 확인하려면 다음처럼 실행한다.

```python
result = review_eval_opt_graph.invoke({
    "review": REVIEW,
    "iteration": 0,
})

print("평가 횟수:", result["iteration"])
print("통과 여부:", result["evaluation"]["overall_pass"])
print("최종 요약:", result["summary"])
```

개선 과정은 저장된 `history`에서 확인한다.

```python
for i, item in enumerate(result.get("history", []), start=1):
    print(f"\n[{i}번째 개선 기록]")
    print("요약:", item["summary"])
    print("평가:", item["evaluation"])
    print("개선 지시:", item["feedback"])
```

현재 구현은 `optimize_summary()`에서 기록을 추가한다. 따라서 개선 노드까지 이동한 회차만 `history`에 남는다. 마지막 통과 회차나 최대 횟수로 바로 종료된 회차는 최종 `summary`, `evaluation`에서 확인한다.

실행 중 노드별 변경분을 보고 싶으면 `stream()`을 사용한다.

```python
for event in review_eval_opt_graph.stream(
    {"review": REVIEW, "iteration": 0},
    stream_mode="updates",
):
    print(event)
```

`invoke()`와 `stream()`은 각각 그래프를 실행한다. 이미 실행한 결과를 확인하려면 저장한 `result`를 읽으면 된다.

### 직접 확인한 실행 결과

```
1회차 요약
→ recommendation 7점
→ 비추천 대상이 명확하지 않아 FAIL

개선 지시
→ 통화를 자주 하는 사용자에게 비추천한다는 내용을 추가

2회차 요약
→ 비추천 대상 명시
→ 모든 평가 항목 8점
→ PASS 후 종료
```

미달 항목을 평가하고, 개선 지시를 다음 생성에 전달하고, 통과 후 종료하는 흐름을 확인했다.

다만 이는 해당 실행에서 평가 모델의 기준을 통과했다는 뜻이다. LLM이 주는 점수는 실행마다 달라질 수 있으므로 원문과 최종 요약도 함께 확인해야 한다.

## 15. 실습 중 헷갈린 코드

### 메시지 객체와 문자열의 차이

```python
return {
    "messages": [AIMessage(content=response.text)]
}
```

메시지 목록에는 메시지 객체를 넣는다.

```python
return {
    "summary": response.text
}
```

`summary: str`에는 문자열을 넣는다. `AIMessage`는 딕셔너리 키가 아니라 메시지 객체의 종류다.

### 문자열을 for문으로 출력한 오류

```python
for msg in result["summary"]:
    print(msg.type)
```

`summary`는 문자열이라 반복하면 글자 하나씩 나온다. 글자에는 `.type`이 없어서 오류가 발생했다.

```python
print(result["summary"])
```

문자열 요약은 바로 출력하면 된다. `len(result["summary"])`도 메시지 개수가 아니라 글자 수다.

### LLM에 전달할 메시지 묶기

```python
response = generator_llm.invoke([
    system,
    HumanMessage(content=review),
])
```

메시지 객체를 전달할 때는 리스트로 묶는다. 일반 문자열 프롬프트는 그대로 전달할 수 있다.

```python
response = generator_llm.invoke(prompt)
```

State를 필드로 나누었더라도 LLM에 전달할 때는 필요한 값을 메시지나 문자열로 구성해야 한다.

## 16. 이번 학습에서 기억할 점

- 먼저 작업 흐름을 적고, 각 노드와 최종 출력에서 필요한 값을 기준으로 State를 정한다.
- 노드는 필요한 State를 읽고, 변경할 값만 반환한다.
- State에 저장한 값도 LLM이 사용하려면 프롬프트에 직접 전달해야 한다.
- 최신 값은 필드에 저장하고, 이전 기록이 필요하면 Reducer로 누적한다.
- RAG Agent에서는 검색 Tool의 설명과 문서 metadata 전달 방식이 중요하다.
- Router에는 불확실한 요청을 처리할 fallback 경로가 필요하다.
- Guardrails는 입력 검사와 출력 검사처럼 여러 단계에 적용할 수 있다.
- Reflection은 피드백을 반영하고, Evaluator-Optimizer는 루브릭으로 미달 항목을 찾아 개선한다.
- State 설계와 함께 프롬프트, 평가 기준, 종료 조건도 작업에 맞게 정해야 한다.

## 17. 헷갈린 점

- 처음에는 기존 예제의 `messages`를 그대로 써야 하는지, 리뷰·요약·피드백을 나눠야 하는지 헷갈렸다. 이번에는 각 노드에서 필요한 값이 분명해서 필드를 나누었고, 최신 값과 누적 기록의 차이를 이해했다.
- 피드백과 `history`가 중복으로 보이기도 했다. 하지만 피드백은 다음 생성에 바로 사용할 값이고, `history`는 나중에 이전 과정을 확인하려고 남기는 기록이라 역할이 달랐다.
- State에 원본 리뷰가 있으면 평가 LLM도 알고 있을 것처럼 생각했는데, 실제로는 평가 프롬프트에 원본과 요약을 모두 넣어야 비교할 수 있다는 점을 알게 되었다.
- 이번 실습에서 가장 크게 느낀 점은 코드를 그대로 가져오는 것보다 먼저 흐름을 그리고 State를 정하는 것이 중요하다는 것이다. 그다음 각 함수가 무엇을 읽고 무엇을 반환하는지 맞춰보니 반복 구조를 이해하기 쉬워졌다.
