# [TIL] 계획을 세우고 실행하며, 조사 결과에 따라 다음 작업 결정하기

이번에는 복잡한 요청을 여러 단계로 나누어 처리하는 Plan-and-Execute를 학습했다. 먼저 여행 계획 예제로 Planner, Executor, Replanner의 역할을 살펴본 뒤, 2026년 생성형 AI 트렌드를 조사하는 그래프를 직접 작성했다.

이후 Replanner를 제거하고 처음 만든 계획을 순서대로 모두 실행하는 구조도 구현했다. 계획을 수정하는 판단과, 남은 작업이 있는지 확인하는 조건문이 서로 다른 역할이라는 점을 이해하는 데 집중했다.

```plain text
Replanner가 있는 구조
계획 생성 → 한 단계 조사 → 결과 검토
                           ├─ 부족하면 계획 수정 → 다음 조사
                           └─ 충분하면 최종 응답 → 종료

Replanner가 없는 구조
계획 생성 → 한 단계 조사 → 완료한 계획 제거
                           ├─ 남은 계획 있음 → 다음 조사
                           └─ 남은 계획 없음 → 최종 보고서 → 종료
```

아래 내용은 `20-plan-and-execute.ipynb`, `20-study.ipynb`와 실습 중 질문한 내용을 바탕으로 정리했다. 코드에서는 스터디의 `plans`, `results` 이름을 사용하고, 프롬프트의 오타와 표현은 읽기 쉽게 다듬었다. Replanner 프롬프트는 완료한 작업 제외와 최종 응답 기준이 드러나도록 보완한 예시다. `llm`, 공통 import, `MODEL_NAME`은 노트북에서 준비한 값을 사용한다.

## 1. Plan-and-Execute란?

Plan-and-Execute는 사용자 요청을 해결할 계획을 먼저 만들고, 계획에 있는 작업을 실행하는 방식이다. 이번 기본 예제에서는 작업을 한 단계 실행할 때마다 Replanner가 조사 결과를 확인하고 남은 계획을 수정한다.

예를 들어 여행 계획을 요청하면 다음과 같은 조사 작업을 만들 수 있다.

```plain text
1. 왕복 대중교통 시간과 요금 조사
2. 관광지 후보와 운영시간 조사
3. 식당 후보와 가격 조사
4. 관광지 사이의 이동 방법 조사
5. 전체 일정과 예산에 필요한 부족한 정보 조사
```

이 목록은 이해를 위한 예시다. 실제 계획은 사용자 요청과 모델의 응답에 따라 달라진다.

중요한 점은 **계획을 세우는 것과 실제 정보를 조사하는 것을 분리한다**는 것이다. Planner가 계획에 적었다고 해서 조사가 완료된 것은 아니다.

## 2. 계획 5개와 그래프 노드 5개는 다르다

프롬프트에서 “조사 단계 5개를 작성해”라고 지시한 것은 조사할 일 목록을 5개 만들라는 의미다. 그래프에 노드를 5개 등록하라는 뜻이 아니다.

```plain text
노드: planning, execute, replanning

계획: [조사 A, 조사 B, 조사 C, 조사 D, 조사 E]

execute를 다시 호출할 때마다 현재 계획의 첫 작업 하나를 수행
```

같은 `execute` 노드를 반복해서 사용하므로 계획이 늘어났다고 그 수만큼 새로운 함수를 만들 필요는 없다. 또한 “트렌드 5개를 선정한다”는 최종 결과물의 개수와 “조사 계획 5개”도 서로 다른 조건이다. 한 조사에서 여러 트렌드를 비교할 수도 있다.

## 3. 각 함수의 역할부터 정하기

함수 내용을 작성하기 전에 각 함수가 어떤 일을 맡을지 먼저 구분했다.

| 함수 | 역할 | 반환하는 주요 값 |
|---|---|---|
| `planning` | 사용자 요청으로 최초 계획 생성 | `plans` |
| `execute` | 현재 계획의 첫 단계 조사 | `results` |
| `replanning` | 조사 결과를 보고 남은 계획 수정 또는 최종 응답 작성 | `plans` 또는 `response` |
| `should_continue` | 현재 State를 보고 다음 경로 선택 | 경로 이름 문자열 |
| `final` | Replanner가 없는 버전에서 최종 보고서 작성 | `response` |

`replanning`이 조사 내용의 충분함을 판단하고, `should_continue`는 그 판단 결과가 저장된 State를 보고 경로를 선택한다. 라우터 자체가 검색하거나 보고서를 작성하는 것은 아니다.

## 4. 사용자 요청과 State 정의하기

```python
research_request = (
    "2026년 생성형 AI 트렌드를 조사하고, "
    "주요 트렌드 5개를 선정해 근거와 함께 요약 보고서를 작성해줘. "
    "각 트렌드의 특징과 예상 영향을 정리하고 출처를 포함해줘."
)

llm = ChatGoogleGenerativeAI(model=MODEL_NAME)

class State(TypedDict):
    input: str
    plans: list[str]
    results: Annotated[list, operator.add]
    response: str
```

State는 그래프의 여러 노드가 이어서 사용할 데이터를 정의한다.

| 필드 | 저장하는 내용 |
|---|---|
| `input` | 사용자의 원래 요청 |
| `plans` | 앞으로 실행할 계획 목록 |
| `results` | 실행한 계획과 조사 결과의 목록 |
| `response` | 사용자에게 보여줄 최종 응답 |

`TypedDict`로 필드를 선언했다고 모든 값이 자동으로 만들어지는 것은 아니다. 최초 입력과 각 노드가 반환하는 딕셔너리를 통해 값이 채워진다.

## 5. plans는 교체하고 results는 누적하기

계획과 조사 결과는 업데이트 방식이 다르다.

```python
plans: list[str]
results: Annotated[list, operator.add]
```

`plans`에는 Reducer가 없으므로 새 목록을 반환하면 기존 계획을 교체한다. `results`는 `operator.add`가 기존 목록과 새 목록을 연결한다.

```plain text
기존 plans: [A, B, C]
새 plans:   [B, C]
저장 결과:  [B, C]

기존 results: [[A, A의 조사 결과]]
새 results:   [[B, B의 조사 결과]]
저장 결과:    [[A, A의 조사 결과], [B, B의 조사 결과]]
```

따라서 Executor는 이번에 만든 결과만 반환하면 된다. 기존 결과 전체를 다시 반환하면 같은 내용이 중복 누적될 수 있다.

## 6. State와 구조화 출력 모델의 차이

```python
class Plan(BaseModel):
    plans: list[str]

class Response(BaseModel):
    response: str

class ReplanDecision(BaseModel):
    decision: Union[Plan, Response]
```

State는 그래프에서 관리할 데이터이고, `BaseModel`로 만든 클래스는 LLM 응답의 형식이다.

```plain text
Plan: 계획 목록을 이런 형식으로 응답해줘.
Response: 최종 글을 이런 형식으로 응답해줘.
ReplanDecision: decision에 Plan 또는 Response를 넣어줘.
State: 각 노드의 실행 결과를 저장하고 다음 노드로 전달한다.
```

`Plan.plans`와 `State.plans`가 같은 이름이어도 동일한 객체의 필드가 아니다. LLM이 반환한 객체에서 계획을 꺼내 State에 저장하는 과정이 필요하다.

```python
return {"plans": response.plans}
```

왼쪽 `"plans"`는 State의 키이고, 오른쪽 `response.plans`는 모델 응답 객체에서 꺼낸 목록이다. State와 출력 모델은 자주 쓰는 틀이 있지만, 작업에 따라 필요한 필드가 달라질 수 있다.

## 7. Union과 구조화 출력을 사용하는 이유

`Union[Plan, Response]`는 값으로 `Plan` 또는 `Response` 타입을 허용한다는 의미다. 여기서는 남은 조사 계획과 최종 보고서 중 하나를 반환하게 한다.

```python
chain = prompt | llm.with_structured_output(ReplanDecision)
```

프롬프트에 “Plan이나 Response로 답해”라고만 적으면 설명 문장으로 답하거나 형식을 다르게 작성할 수 있다. 구조화 출력은 선언한 스키마에 맞춰 응답을 받도록 하고, 코드에서 객체의 필드를 읽거나 타입을 확인할 수 있게 한다.

단, 응답 형식과 판단의 정확성은 별개다. `Response` 형식으로 반환되었다고 조사 근거가 충분하거나 모든 사실이 정확하다는 것까지 보장되지는 않는다. 형식에 맞지 않는 응답은 파싱이나 검증 오류로 이어질 수도 있다.

## 8. Planner: 사용자 요청을 조사 계획으로 바꾸기

```python
def planning(state: State):
    prompt = ChatPromptTemplate.from_messages([
        (
            "system",
            """당신은 조사 계획을 세우는 Planner입니다.
요청 해결에 필요한 조사 단계 5개를 작성하세요.
각 단계는 구체적인 조사 목표를 가지도록 하세요.
최종 보고서 작성은 제외하고 조사 단계만 반환하세요.""",
        ),
        ("human", "{input}"),
    ])

    chain = prompt | llm.with_structured_output(Plan)
    response = chain.invoke({"input": state["input"]})

    return {"plans": response.plans}
```

스터디에서는 간단하게 “조사 단계 5가지를 작성해”라고 작성했다. 위 코드는 강사님 풀이의 취지를 반영해 최종 보고서 작성과 조사 단계를 구분한 예시다. 강사님 풀이의 “최대 5개”와 스터디의 “5개”는 표현상 차이가 있다. 현재 `list[str]` 선언만으로는 개수 제한을 검증하지 않는다.

함수 안에서는 전역 변수 `research_request`를 직접 사용하기보다 `state["input"]`을 읽는다. 그래야 그래프에 다른 요청을 넣어도 같은 함수를 재사용할 수 있다.

## 9. 프롬프트, 체인, invoke를 구분하기

```python
chain = prompt | llm.with_structured_output(Plan)
response = chain.invoke({"input": state["input"]})
```

| 구성 | 의미 |
|---|---|
| `prompt` | 모델에게 전달할 지시와 입력 자리 준비 |
| `with_structured_output(Plan)` | 모델 응답을 Plan 형식으로 받도록 설정 |
| `|` | 앞 단계의 출력을 다음 단계의 입력으로 연결 |
| `invoke()` | 현재 입력을 넣어 실제 호출 실행 |

```plain text
입력 딕셔너리
→ 프롬프트의 {input} 채우기
→ 모델 호출
→ Plan 객체로 응답 받기
```

체인을 만들었다고 모델이 바로 실행되는 것은 아니다. `invoke()`를 호출해야 현재 입력으로 작업을 수행한다. 또한 함수 안의 체인과 노드 사이의 그래프 연결은 구분해야 한다. 체인은 프롬프트와 모델의 처리 순서를 묶고, 그래프는 여러 노드의 이동과 반복을 연결한다.

## 10. Executor: 현재 계획 하나를 조사하기

```python
def execute(state: State):
    current_plan = state["plans"][0]

    prompt = ChatPromptTemplate.from_messages([
        (
            "system",
            """당신은 조사원입니다.
현재 단계에 대해서만 Google Search로 조사하고,
핵심 결과와 출처 URL을 정리하세요.""",
        ),
        (
            "human",
            """조사 요청:
{input}

현재 단계:
{current_plan}

이전 조사 결과:
{results}""",
        ),
    ])

    search_llm = llm.bind_tools([{"google_search": {}}])
    chain = prompt | search_llm

    response = chain.invoke({
        "input": state["input"],
        "current_plan": current_plan,
        "results": "\n".join(
            f"{plan} - {text}"
            for plan, text in state.get("results", [])
        ),
    })

    return {"results": [[current_plan, response.text]]}
```

`state["plans"][0]`은 현재 목록의 첫 번째 작업을 꺼낸다. 첫 번째 작업을 자동으로 삭제하는 코드는 아니다.

이번 예제는 Gemini의 내장 Google Search 도구를 연결한 호출이다. 변수 이름에 `ReAct_agent`라고 적는 것만으로 별도의 ReAct 반복 그래프가 생성되는 것은 아니다.

### 현재 단계와 이전 결과를 함께 주는 이유

`current_plan`은 지금 수행할 작업이고, `results`는 이미 확인한 내용이다. 이전 결과는 참고 문맥이지만 다음 조사가 앞선 결과에 의존한다면 중요한 입력이 된다.

```plain text
원래 요청: 트렌드 5개와 근거를 포함한 보고서 작성
현재 단계: 후보 트렌드의 기업 도입 사례 조사
이전 결과: 앞 단계에서 수집한 트렌드 후보와 출처
```

State에 기록이 있어도 LLM이 자동으로 읽는 것은 아니다. 이번 코드처럼 프롬프트에 넣어 전달해야 참고할 수 있다.

## 11. join으로 이전 조사 결과를 문자열로 합치기

```python
"results": "\n".join(
    f"{plan} - {text}"
    for plan, text in state.get("results", [])
)
```

이 코드는 다음 순서로 동작한다.

1. `state.get("results", [])`로 지금까지의 결과를 가져온다. 키가 없으면 빈 목록을 사용한다.
2. 각 항목에서 계획과 조사 결과를 `plan`, `text`로 나누어 받는다.
3. `f"{plan} - {text}"`로 한 줄의 문자열을 만든다.
4. `"\n".join(...)`으로 문자열 사이에 줄바꿈을 넣어 합친다.

```python
results = [
    ["트렌드 후보 조사", "후보 A, B, C를 확인함"],
    ["도입 사례 조사", "관련 기업 발표를 확인함"],
]
```

위 목록을 합치면 다음 형태가 된다.

```plain text
트렌드 후보 조사 - 후보 A, B, C를 확인함
도입 사례 조사 - 관련 기업 발표를 확인함
```

“하나씩 꺼낸다”는 말은 읽어서 변수에 대입한다는 뜻이다. 원래 `results` 목록에서 항목이 제거되지는 않는다.

현재 계획을 전달할 때 쓰는 `"\n".join(state["plans"])`도 같은 원리다. `plans`는 문자열 목록이므로 별도의 계획·결과 분리 없이 바로 합칠 수 있다.

## 12. Replanner: 계획을 수정하거나 최종 응답 만들기

Replanner는 원래 요청, 현재 계획, 완료한 조사 결과를 함께 보고 다음 행동을 결정한다. 아래는 스터디의 간단한 프롬프트에 판단 기준을 명시한 예시다.

```python
def replanning(state: State):
    prompt = ChatPromptTemplate.from_messages([
        (
            "system",
            """당신은 조사 진행 상황을 판단하는 Replanner입니다.
원래 요청과 조사 결과를 비교하세요.
정보가 부족하면 완료한 작업을 제외하고 필요한 조사만 Plan으로 반환하세요.
필요하면 남은 계획을 수정하거나 추가하세요.
근거가 충분하면 Response로 최종 보고서를 반환하세요.
최종 보고서에는 주요 트렌드 5개, 특징, 예상 영향, 근거, 출처 URL을 포함하세요.
조사로 확인하지 못한 내용은 확인되지 않았다고 밝히세요.""",
        ),
        (
            "human",
            """조사 요청:
{input}

현재 계획:
{current_plan}

조사 결과:
{results}""",
        ),
    ])

    chain = prompt | llm.with_structured_output(ReplanDecision)

    response = chain.invoke({
        "input": state["input"],
        "current_plan": "\n".join(state["plans"]),
        "results": "\n".join(
            f"{plan} - {text}"
            for plan, text in state.get("results", [])
        ),
    })

    if isinstance(response.decision, Response):
        return {"response": response.decision.response}

    return {"plans": response.decision.plans}
```

### LLM은 어디에서 판단 기준을 받는가?

판단 기준은 `replanning`의 system 프롬프트에 있다.

```plain text
정보가 부족하면 필요한 조사만 Plan으로 반환
근거가 충분하면 최종 보고서를 Response로 반환
```

`invoke()`는 이 지시와 현재 조사 데이터를 함께 모델에 전달한다. 구조화 출력은 응답 형식을 정하고, 조건문은 모델이 반환한 객체를 구분한다.

### 계획을 수정한다는 의미

```plain text
처음 계획: [후보 조사, 사례 조사, 영향 조사]

후보 조사 후 일부 후보의 근거가 부족함
→ [부족한 후보의 근거 추가 조사, 사례 조사, 영향 조사]

이미 충분한 자료가 확보됨
→ 남은 작업이 있어도 최종 보고서 작성 가능
```

Replanner는 다음 계획 목록을 반환하여 `plans`를 교체한다. 이번 Replanner 있는 버전의 Executor는 계획을 직접 줄이지 않으므로, 완료한 작업을 다음 Plan에서 제외하도록 지시하는 것이 중요하다. 스터디의 “단계가 남아 있으면 plan”이라는 문장만으로는 이 기준이 충분히 명확하지 않다.

## 13. response.decision.response를 읽는 방법

```python
if isinstance(response.decision, Response):
    return {"response": response.decision.response}
```

이름이 반복되지만 각각 가리키는 대상은 다르다.

| 표현 | 의미 |
|---|---|
| `response` | `invoke()`가 반환한 ReplanDecision 객체 |
| `response.decision` | 그 안에서 선택된 Plan 또는 Response 객체 |
| `response.decision.response` | 선택된 Response 객체 안의 최종 글 |
| `"response"` | 최종 글을 저장할 State의 키 |

```python
# 응답 객체의 형태를 이해하기 위한 예시
ReplanDecision(
    decision=Response(response="최종 트렌드 보고서 내용")
)
```

`isinstance()`의 두 번째 인자는 타입이어야 한다. 따라서 소문자 변수 `response`가 아니라 클래스 이름인 대문자 `Response`를 사용한다.

## 14. Replanner가 있는 그래프 연결하기

```python
def should_continue(state: State):
    return "end" if state.get("response") else "continue"

builder = StateGraph(State)
builder.add_node("planning", planning)
builder.add_node("execute", execute)
builder.add_node("replanning", replanning)

builder.add_edge(START, "planning")
builder.add_edge("planning", "execute")
builder.add_edge("execute", "replanning")
builder.add_conditional_edges(
    "replanning",
    should_continue,
    {"continue": "execute", "end": END},
)

graph = builder.compile()
display(Image(graph.get_graph().draw_mermaid_png()))
```

`should_continue`가 반환하는 문자열은 경로의 이름이다. `"end"`라는 단어 자체가 종료 기능을 가진 것은 아니다. 위 딕셔너리에서 `"end"`를 `END`에 연결했기 때문에 종료된다.

Replanner는 전체 요청을 충족할 수 있는지 검토하는 역할을 한다. 이번 코드에는 각 조사 단계의 정확도를 별도 기준으로 검사하는 전용 Validator 노드는 없다. Replanner의 검토를 엄밀한 사실 검증과 동일하게 보아서는 안 된다.

## 15. 추가 실습: Replanner 없이 처음 계획대로 실행하기

추가 실습은 Replanner를 제거하고 Planner가 처음 만든 계획을 순서대로 모두 실행하도록 바꾸는 것이다.

```plain text
planning → execute → 남은 계획 확인
              ↑            │
              └─ 남아 있음 ┘
                           └─ 비어 있음 → final → END
```

Replanner가 없어졌다고 검색이 한 번에 합쳐지거나 병렬 실행되는 것은 아니다. 이번 구현에서는 Executor가 계속 한 단계씩 수행한다. 이전 조사 결과도 그대로 참고한다.

바뀌는 부분은 다음과 같다.

| 항목 | 변경 내용 |
|---|---|
| `ReplanDecision` | 두 응답 중 선택하지 않으므로 제거 |
| `replanning` | 프롬프트, 체인, 함수, 노드 제거 |
| `execute` | 조사 후 완료한 계획을 목록에서 제외 |
| `should_continue` | 최종 응답 유무 대신 남은 계획 유무 확인 |
| `final` | 조사 결과로 최종 보고서 작성 |
| 그래프 | Executor에서 반복 또는 final로 분기 |

## 16. plans[1:]로 완료한 작업 제외하기

현재 작업을 선택하는 코드는 그대로 둔다.

```python
current_plan = state["plans"][0]
```

조사가 끝난 뒤 반환값에 남은 계획을 추가한다.

```python
return {
    "results": [[current_plan, response.text]],
    "plans": state["plans"][1:],
}
```

`[1:]`는 인덱스 1부터 끝까지 가져오는 슬라이싱이다.

```plain text
실행 전: [A, B, C]
A 조사 후 반환: [B, C]
B 조사 후 반환: [C]
C 조사 후 반환: []
```

슬라이싱 자체가 원본 목록을 직접 삭제하는 것은 아니다. 첫 항목을 제외한 새 목록을 반환하고, 그래프가 그 목록으로 State의 `plans`를 교체한다.

`[0]`은 지금 실행할 작업 하나를 고르고, `[1:]`는 실행한 뒤 남길 작업 목록을 만든다. 사용하는 위치와 목적이 다르다.

## 17. 남은 계획을 기준으로 경로 선택하기

```python
def should_continue(state: State):
    return "continue" if state.get("plans") else "final"
```

```plain text
plans = [B, C] → continue → execute
plans = [C]    → continue → execute
plans = []     → final    → 최종 보고서 작성
```

이 함수는 LLM을 호출하지 않는다. 리스트가 비었는지만 Python 조건문으로 확인한다. 조사 결과가 부족하더라도 계획이 비었다면 final로 이동한다는 점이 Replanner 버전과 다르다.

기존의 `return "end" if state.get("response") else "continue"`를 그대로 사용하면, 최종 응답이 아직 없는 상태에서 계속 Executor로 돌아갈 수 있다. 그러다 계획이 비면 `[0]`에서 오류가 날 수 있으므로 종료 기준도 함께 바꿔야 한다.

## 18. final에서 최종 보고서 만들기

기존에는 Replanner가 최종 응답까지 만들었다. 제거 버전에서는 `final`이 그 역할을 맡는다.

```python
def final(state: State):
    prompt = ChatPromptTemplate.from_messages([
        (
            "system",
            """당신은 조사 결과를 정리하는 보고서 작성자입니다.
원래 요청과 지금까지의 조사 결과를 바탕으로 보고서를 작성하세요.
주요 트렌드 5개, 특징, 예상 영향, 근거, 출처 URL을 포함하세요.
확인하지 못한 내용은 추측하지 말고 한계를 밝히세요.""",
        ),
        (
            "human",
            """조사 요청:
{input}

전체 결과:
{results}""",
        ),
    ])

    chain = prompt | llm
    response = chain.invoke({
        "input": state["input"],
        "results": "\n".join(
            f"- {step}: {step_result}"
            for step, step_result in state["results"]
        ),
    })

    return {"response": response.text}
```

스터디에서는 “지금까지 결과를 바탕으로 보고서를 만들어”라는 간단한 지시를 사용했다. 위 예시는 원래 요청의 결과물 조건을 다시 명시했다.

### 일반 응답과 구조화 출력의 차이

```python
# 이번 스터디의 final: 일반 모델 응답에서 텍스트를 꺼낸다.
chain = prompt | llm
# 호출 후 사용: response.text

# 다른 구현: Response 객체의 response 필드를 꺼낸다.
chain = prompt | llm.with_structured_output(Response)
# 호출 후 사용: response.response
```

출력 변수 이름 때문에 접근 방식이 정해지는 것이 아니라 반환된 객체의 타입에 따라 달라진다. 이번 스터디 제거 버전에는 `Response` 클래스가 남아 있지만 일반 LLM 호출만 사용하므로 해당 클래스는 최종 응답 생성에 쓰이지 않는다.

## 19. Replanner 없는 그래프 연결하기

Executor의 반환값과 라우터를 제거 버전으로 수정한 뒤 그래프를 다시 만든다.

```python
builder = StateGraph(State)
builder.add_node("planning", planning)
builder.add_node("execute", execute)
builder.add_node("final", final)

builder.add_edge(START, "planning")
builder.add_edge("planning", "execute")
builder.add_conditional_edges(
    "execute",
    should_continue,
    {"continue": "execute", "final": "final"},
)
builder.add_edge("final", END)

graph = builder.compile()
display(Image(graph.get_graph().draw_mermaid_png()))
```

조건부 연결의 출발점은 이제 `replanning`이 아니라 `execute`다. 계획이 없으면 final로 이동하고, 보고서를 만든 뒤 END로 종료한다.

```plain text
계획이 3개인 경우
START → planning → execute(A) → execute(B) → execute(C) → final → END
```

이 예제는 Planner가 적어도 한 개의 작업을 만든다는 전제다. 빈 초기 계획까지 처리하려면 planning 이후에도 목록이 비었는지 확인하는 경로가 필요하다.

## 20. 강사님 풀이와 직접 만든 코드 비교

두 제거 버전의 핵심 흐름은 같다. 다만 변수 이름과 최종 응답 형식이 일부 다르다.

| 구분 | 강사님 제거 버전 | 스터디 제거 버전 |
|---|---|---|
| 계획 필드 | `plan` | `plans` |
| 결과 필드 | `past_steps` | `results` |
| 조사 함수 | `execute_no_replan` | `execute` |
| 완료한 계획 제외 | `state["plan"][1:]` | `state["plans"][1:]` |
| 최종 작성 함수 | `finalize` | `final` |
| 최종 응답 추출 | `response.text` | `response.text` |

메인 노트북에 직접 만든 또 다른 제거 버전은 finalizer에 `with_structured_output(Response)`를 사용하여 `result.response`로 최종 글을 꺼낸다. 이 차이가 전체 실행 흐름을 바꾸지는 않는다.

강사님 제거 예제는 여행 조사 문구와 일정·비용·이동 방법을 포함한 최종 응답 지시가 남아 있다. AI 트렌드 조사에 적용할 때는 조사 대상과 최종 보고서 요구사항을 해당 주제에 맞게 바꿔야 한다.

## 21. invoke와 stream으로 실행 결과 확인하기

### 최종 결과만 출력하기

```python
response = graph.invoke(
    {"input": research_request},
    config={"recursion_limit": 20},
)

print(response["response"])
```

`graph.invoke()`는 최종 State를 반환한다. 따라서 `print(response)`는 계획, 조사 기록 등을 포함한 전체 딕셔너리를 출력하고, `print(response["response"])`는 최종 보고서만 출력한다.

### 노드별 진행 상황 확인하기

```python
for event in graph.stream(
    {"input": research_request},
    config={"recursion_limit": 20},
    stream_mode="updates",
):
    for node_name, value in event.items():
        if node_name == "planning":
            print("\n[초기 계획]")
            print(value["plans"])
        elif node_name == "execute":
            step, result = value["results"][-1]
            print(f"\n[실행] {step}")
            print(result)
        elif node_name == "final":
            print("\n[최종 응답]")
            print(value["response"])
```

이 출력 코드는 스터디의 Replanner 없는 그래프 이름에 맞춘 예시다. Replanner 있는 그래프에서는 `replanning`의 반환값에 `plans`가 있는지, `response`가 있는지 구분해 출력한다.

| stream 모드 | 전달되는 내용 |
|---|---|
| `updates` | 각 노드가 반환한 State 변경 내용 |
| `values` | 해당 시점의 전체 State |

두 모드를 함께 요청하면 `(mode, event)`로 구분해서 받는다. 단계별 조사 결과가 길고 전체 State까지 출력하면 화면에 많은 내용이 나올 수 있다. 출력이 많다는 사실만으로 무한 반복이라고 판단할 수는 없다.

## 22. recursion_limit과 오래 걸리는 이유

`recursion_limit=20`은 20번을 채워서 실행하라는 뜻이 아니다. 종료 조건에 도달하지 못한 채 그래프가 계속 진행할 때 실행을 제한하는 상한이다. 엄밀하게는 그래프의 superstep 기준이며, 이번처럼 노드가 순차로 실행되는 구조에서는 노드 진행 횟수와 가깝게 이해할 수 있다.

한도를 초과하면 정상 보고서를 자동으로 만들어 주는 것이 아니라 `GraphRecursionError`가 발생한다.

Replanner 없는 버전에서 계획이 5개라면 기본적인 모델 호출 흐름은 다음과 같다.

```plain text
계획 생성 1회
→ 검색 도구가 연결된 조사 모델 호출 5회
→ 최종 보고서 생성 1회
```

각 조사 호출에서 실제 검색이 몇 번 일어나는지는 모델과 도구 동작에 따라 달라진다. 조사들이 순차 실행되고 이전 결과도 계속 전달되므로 전체 시간이 길어질 수 있다. 실습 화면에서는 최종 보고서가 나오기까지 약 2분 40초가 걸렸다. 이것은 해당 실행에서 관찰한 시간이며 고정된 소요 시간은 아니다.

흐름만 테스트할 때는 조사 단계를 2개 정도로 줄이고, `updates`로 현재 어떤 작업이 실행 중인지 확인하면 편하다. Replanner를 제거하면 재판단 호출은 줄어들지만, 처음 계획을 모두 수행하므로 항상 더 빨라진다고 단정할 수는 없다.

## 23. Executor에서 전체 실행 후 바로 종료해도 될까?

가능하다. Executor 내부에서 계획 전체를 `for`문으로 실행하고 최종 보고서까지 작성한 뒤 END로 연결할 수 있다.

```plain text
이번 실습
한 단계 실행 → 그래프가 다음 실행 결정 → 마지막에 final

다른 구현
execute_all 내부에서 모든 조사와 최종 작성 → END
```

다만 조사 결과만 저장한 채 END로 연결하면 최종 보고서가 자동으로 만들어지지는 않는다. 최종 글이 필요하다면 어느 함수에서는 반드시 보고서 작성과 `response` 반환을 수행해야 한다.

이번처럼 조사와 최종 작성을 분리하면 노드별 진행 상황을 확인하고 각 단계의 역할을 이해하기 쉽다. 실무에서도 고정 순서 작업인지, 조사 중 계획 수정이 필요한지, 중간 기록과 재시도가 필요한지에 따라 구조를 선택한다.

## 24. 실습 중 발생한 오류와 확인 방법

### State가 정의되지 않은 오류

```plain text
NameError: name 'State' is not defined
```

함수를 정의하기 전에 State 클래스가 있는 셀을 실행해야 한다. 커널을 재시작했다면 import와 모델 준비 셀도 다시 실행한다.

### 프롬프트 생성 메서드와 역할 이름

이번 템플릿 생성에는 `ChatPromptTemplate.from_messages(...)`를 사용한다. `format_messages`는 생성된 템플릿의 값을 채울 때 사용하는 메서드이므로 두 용도를 구분한다. 메시지 역할은 `"human"`이며 `"humman"`은 오타다.

### 프롬프트 변수와 invoke 키 불일치

```plain text
프롬프트: {current_plan}, {results}
invoke:   "current_plan", "results"
```

프롬프트에 `{current_step}`을 남겨 두고 `"current_plan"`만 전달하면 필요한 값을 채울 수 없다. State 필드명과 프롬프트 변수명은 반드시 같을 필요는 없지만, 템플릿의 변수명과 invoke 딕셔너리의 키는 맞아야 한다.

### 딕셔너리 문법과 철자

```python
# 입력은 딕셔너리로 감싼다.
response = graph.invoke({"input": research_request})

# 반환도 key: value 형태로 작성한다.
return {
    "results": [[current_plan, response.text]],
    "plans": state["plans"][1:],
}
```

`results`를 `resuls`로 쓰거나, 키와 값을 불필요한 괄호로 묶지 않도록 확인한다. 여러 값이 있다는 이유로 딕셔너리 전체 안에 튜플 괄호를 추가하는 것은 아니다.

### 제거한 노드 이름이 그래프에 남아 있는 경우

Replanner를 제거했다면 `add_edge`와 `add_conditional_edges`에서도 해당 이름을 정리해야 한다. 제거 버전의 조건부 연결은 `execute`에서 시작하고, 마지막 작성 노드는 END와 연결한다.

### 함수 수정 후 이전 그래프를 실행하는 경우

```plain text
State와 함수 정의 셀
→ builder 생성·노드 연결·compile 셀
→ invoke 또는 stream 실행 셀
```

노트북에는 같은 이름의 함수와 `graph`를 여러 번 정의한 셀이 있으므로 현재 어떤 버전을 실행했는지 확인해야 한다. 함수를 수정한 뒤에는 그래프도 다시 구성하고 컴파일한다.

## 25. 이번 학습에서 기억할 점

- Plan-and-Execute는 계획 생성과 실제 실행을 구분하는 패턴이다.
- 조사 단계의 개수, 노드의 개수, 최종 결과물의 개수는 서로 다르다.
- State는 공유할 데이터이고, 구조화 출력 모델은 LLM이 반환할 형식이다.
- `plans`는 교체하고 `results`는 Reducer로 누적한다.
- `invoke()`가 실제 입력으로 모델이나 그래프 실행을 시작한다.
- 이전 결과는 State에 저장하는 것과 별개로 프롬프트에 전달해야 한다.
- `join()`은 결과를 읽어서 문자열로 합치며 원본 목록을 삭제하지 않는다.
- Replanner의 판단 기준은 프롬프트에 있고, 조건문은 반환된 객체를 구분한다.
- Replanner 없는 버전은 `[1:]`로 계획을 줄이고 목록이 비었는지 확인한다.
- Replanner를 제거하면 최종 보고서 작성 역할을 다른 함수에 배치해야 한다.
- `response.text`와 `response.response`는 응답 객체의 타입에 따라 구분한다.
- 경로 이름은 조건부 연결의 매핑에 따라 실제 노드나 END로 이어진다.
- `recursion_limit`은 반복 목표 횟수가 아니라 실행 상한이다.
- 함수 정의, 그래프 컴파일, 실행 순서를 맞춰 현재 코드를 확인한다.

## 26. 헷갈린 점

처음에는 계획이 5개라는 말을 그래프에 단계가 5개 있다는 뜻으로 이해했다. 실제로는 Planner가 만든 조사 목록이 5개이고, 같은 Executor를 다시 호출하면서 목록의 작업을 하나씩 수행하는 구조였다.

Replanner와 라우터의 역할도 헷갈렸다. Replanner는 LLM에게 요청과 조사 결과를 보여 주고 계획을 수정할지 최종 응답을 만들지 판단하게 한다. 라우터는 그 결과가 State에 저장된 뒤 다음 경로를 고르는 조건문이었다.

특히 LLM이 어떻게 Plan과 Response 중 하나를 고르는지 이해하기 어려웠다. 선택 기준은 Replanner의 프롬프트에 적혀 있고, `invoke()`가 현재 데이터를 넣어 그 판단을 실행한다. `with_structured_output`은 판단 기준 자체가 아니라 응답을 코드로 다룰 수 있는 형식으로 받기 위한 설정이었다.

Replanner 없는 버전은 이전 내용을 참고하지 않고 한 번에 실행하는 것으로 생각했다. 하지만 이전 결과는 계속 전달하고, 계획만 다시 판단하지 않는 방식이었다. Executor가 작업 하나를 끝낸 뒤 `[1:]`로 완료한 계획을 제외하고 남은 목록을 차례대로 실행했다.

`response.decision.response`도 같은 단어가 반복되어 복잡해 보였다. 호출 결과 안의 decision을 꺼내고, 선택된 Response 객체에서 최종 글을 꺼낸다고 나누어 보니 이해하기 쉬웠다.

이번 실습에서는 전체 코드를 외우는 것보다 **각 함수가 무엇을 받고, 어떤 값을 만들고, 어디에 저장하고, 다음에 어디로 이동하는지**를 먼저 정하는 것이 중요하다는 점을 배웠다. 기본 구조를 이해한 뒤 예제 코드를 참고하고, 작은 단위로 실행하며 확인하는 방식으로 연습했다.
