# 반복 개선에서 병렬 처리와 동적 작업 분배까지

이번에는 제품 리뷰를 요약하는 Reflection과 Evaluator-Optimizer의 강사님 풀이를 살펴봤다. 이후 여러 작업을 동시에 실행하는 Parallelization과, 요청에 따라 작업을 나누고 Worker에게 배정하는 Orchestrator-worker를 학습했다.

병렬 번역과 맞춤형 학습 자료 생성을 통해 결과 누적과 작업 분배를 연습했고, 추가로 지원자 정보를 바탕으로 면접 질문을 만드는 구조를 설계했다.

```
Reflection
생성 → 검토 → 피드백 반영 → 재생성

Parallelization
미리 정해진 여러 작업을 병렬 실행 → 결과 모으기

Orchestrator-worker
요청 분석 → 작업 계획 → 작업별 병렬 실행 → 결과 모으기
```

> 아래 코드는 노트북과 실습 대화의 핵심을 정리한 것이다. `llm`, 공통 import, `REVIEW` 등은 노트북에서 준비한 값을 사용한다. 정답본의 미완성 부분은 따로 설명하고, 예제는 이해하기 쉽게 변수명과 프롬프트를 정리했다.

## 1. Reflection 실습: 메시지를 누적해서 요약 개선하기

첫 번째 실습은 제품 리뷰를 요약한 뒤 피드백을 받아 다시 작성하는 구조다.

강사님은 원본 리뷰, 요약, 피드백을 각각의 필드로 나누지 않고 `messages`에 모았다.

```python
class ReflectionState(TypedDict):
    messages: Annotated[list, add_messages]
    iteration: int
```

| 필드 | 역할 |
|---|---|
| `messages` | 원본 리뷰, 요약, 피드백을 누적 |
| `iteration` | 요약을 생성한 횟수 |

실행 과정에서 메시지는 다음처럼 쌓인다.

```
원본 리뷰
→ 첫 요약
→ 검토 피드백
→ 수정 요약
```

### 요약 생성

```python
def generate(state: ReflectionState):
    system_prompt = SystemMessage(content="""
입력받은 리뷰를 한 문단, 3~5문장으로 요약해줘.
장점, 단점, 추천 대상을 포함하고 원문에 없는 내용은 추가하지 마.
이전 피드백이 있으면 반영하고 요약문만 응답해줘.
""")

    response = llm.invoke([
        system_prompt,
        *state["messages"],
    ])

    return {
        "messages": [response],
        "iteration": state.get("iteration", 0) + 1,
    }
```

`state["messages"]`는 메시지 목록을 펼쳐서 전달한다.

```python
[system_prompt, *state["messages"]]
```

두 번째 생성에서는 원본 리뷰뿐 아니라 이전 요약과 피드백도 함께 전달된다.

```
SystemMessage: 요약 작성 규칙
HumanMessage: 원본 리뷰
AIMessage: 이전 요약
HumanMessage: 수정 피드백
```

State에 기록을 저장하는 것과 LLM에게 전달하는 것은 별개다. 이 코드에서는 누적된 메시지를 `invoke()`에 직접 넣기 때문에 이전 내용을 참고할 수 있다.

### 요약 검토

```python
def reflect(state: ReflectionState):
    original_request = state["messages"][0]
    latest_summary = state["messages"][-1]

    system_prompt = SystemMessage(content="""
원본 리뷰와 요약문을 비교해 검토해줘.
내용이 편향되었는지, 중요한 장점과 단점이 누락되었는지 확인해줘.
다음 수정에 사용할 구체적인 피드백만 작성해줘.
""")

    human_prompt = HumanMessage(content=f"""
원본 리뷰:
{original_request.text}

현재 요약:
{latest_summary.text}
""")

    response = llm.invoke([
        system_prompt,
        human_prompt,
    ])

    return {
        "messages": [
            HumanMessage(content=response.text)
        ]
    }
```

첫 메시지에서 원본을, 마지막 메시지에서 최신 요약을 꺼낸다.

```python
state["messages"][0]   # 원본 리뷰
state["messages"][-1]  # 최신 요약
```

검토 결과를 새 `HumanMessage`로 감싸면 다음 생성 호출에서 수정 요청의 역할로 전달할 수 있다.

`human_prompt`는 긴 메시지를 읽기 쉽게 나눠 놓은 일반 변수다. `invoke()` 안에 직접 작성해도 같은 방식으로 전달된다.

### 종료 조건과 실행

강사님 풀이에서는 생성 횟수가 2가 되면 종료한다.

```python
def should_continue(state: ReflectionState):
    if state["iteration"] >= 2:
        return "end"

    return "reflect"


builder = StateGraph(ReflectionState)

builder.add_node("generate", generate)
builder.add_node("reflect", reflect)

builder.add_edge(START, "generate")
builder.add_edge("reflect", "generate")

builder.add_conditional_edges(
    "generate",
    should_continue,
    {
        "reflect": "reflect",
        "end": END,
    },
)

reflection_graph = builder.compile()
```

```
첫 요약 → 검토 → 수정 요약 → 종료
```

```python
result = reflection_graph.invoke({
    "messages": [HumanMessage(content=REVIEW)],
    "iteration": 0,
})

print(result["messages"][-1].text)
```

정답본 실행 셀에는 `message`라는 단수형 키가 있지만, 정의한 State는 `messages`다. 입력 키와 메시지 목록의 형태를 맞춰야 한다.

## 2. Evaluator-Optimizer 실습: 평가 결과를 다음 생성에 반영하기

두 번째 실습은 요약문을 항목별로 채점하고, 평가 결과를 참고해 다시 요약하는 구조다.

```
요약 생성 → 평가
             ├─ 모든 항목 통과 → 종료
             └─ 미달 → 평가 결과를 반영해 재생성
```

강사님 풀이에서는 별도의 `optimize` 노드를 만들지 않았다. 평가 결과를 다음 `generate()`가 직접 읽고 수정한다.

### 루브릭 정의

루브릭은 무엇을 어떤 기준으로 평가할지 정한 채점표다.

```python
class CriterionResult(BaseModel):
    score: int = Field(
        description="1~10점",
        ge=1,
        le=10,
    )
    reason: str = Field(description="점수를 매긴 근거")


class EvaluationResult(BaseModel):
    completeness: CriterionResult = Field(
        description="핵심 장점, 단점, 가격 정보가 포함되었는가?"
    )
    accuracy: CriterionResult = Field(
        description="원문과 일치하고 없는 정보를 추가하지 않았는가?"
    )
    conciseness: CriterionResult = Field(
        description="불필요한 반복 없이 3~5문장인가?"
    )
    recommendation: CriterionResult = Field(
        description="적합한 사용자와 부적합한 사용자를 모두 언급했는가?"
    )


evaluator = llm.with_structured_output(EvaluationResult)
```

평가 결과는 다음처럼 항목별 점수와 근거를 가진다.

```json
{
    "recommendation": {
        "score": 7,
        "reason": "통화를 자주 하는 사용자에게 부적합하다는 내용이 빠졌다."
    }
}
```

단순히 "좋다" 또는 "나쁘다"보다 어떤 부분을 수정해야 하는지 확인하기 쉽다.

### State와 생성 노드

```python
class EvalState(TypedDict):
    request: str
    result: str
    evaluation: dict
```

| 필드 | 역할 |
|---|---|
| `request` | 원본 리뷰 |
| `result` | 현재 요약 |
| `evaluation` | 최신 평가 결과 |

정답본에는 `history`도 선언되어 있지만 기록을 추가하는 코드는 아직 없다. State에 필드를 선언하는 것만으로 기록이 자동 저장되지는 않는다.

```python
def generate_summary(state: EvalState):
    evaluation = state.get("evaluation", {})

    response = llm.invoke(f"""
원본 리뷰를 한 문단, 3~5문장으로 요약해줘.
장점, 단점, 가격, 추천 대상과 비추천 대상을 포함해줘.
원문에 없는 내용은 추가하지 마.
이전 평가가 있다면 부족한 부분을 반영해줘.
요약문만 출력해줘.

원본 리뷰:
{state["request"]}

이전 평가:
{evaluation}
""")

    return {"result": response.text}
```

처음에는 평가가 없어서 원본만 보고 요약한다. 평가 이후 다시 실행되면 최신 평가 결과가 프롬프트에 포함된다.

### 평가 노드

```python
def evaluate_summary(state: EvalState):
    response = evaluator.invoke(f"""
원본과 요약을 비교하여 정해진 네 항목을 평가해줘.
각 항목을 1~10점으로 채점하고 구체적인 근거를 작성해줘.
기준을 충족하면 8점 이상, 충족하지 못하면 7점 이하를 부여해줘.

원본:
{state["request"]}

요약:
{state["result"]}
""")

    return {
        "evaluation": response.model_dump()
    }
```

`model_dump()`는 구조화된 Pydantic 객체를 딕셔너리로 바꾼다.

### 통과 판단

```python
CRITERIA = (
    "completeness",
    "accuracy",
    "conciseness",
    "recommendation",
)


def should_continue_eval(state: EvalState):
    passed = all(
        state["evaluation"][name]["score"] >= 8
        for name in CRITERIA
    )

    return "end" if passed else "generate"
```

`all()`을 사용하므로 네 항목이 전부 8점 이상이어야 통과한다. 평균이 8점이어도 한 항목이 7점이면 미통과다.

정답본의 종료 함수에는 설명용 의사코드가 남아 있다. 위 코드는 그 판단을 구체화한 예시이며, 실제 반복 실행에서는 최대 횟수도 함께 정해야 한다.

### 강사님 풀이와 별도 optimize 노드의 차이

```
강사님 풀이
평가 결과 → generate가 직접 반영

별도 optimize 노드 사용
평가 결과 → 수정 지시 생성 → generate가 수정 지시 반영
```

두 방식 모두 평가를 바탕으로 결과를 개선한다. 수정 지시를 따로 만드는 단계가 필요한지에 따라 구조를 선택할 수 있다.

## 3. Reflection과 Evaluator-Optimizer 비교

| 구분 | 이번 Reflection | 강사님 Evaluator-Optimizer |
|---|---|---|
| 검토 결과 | 자연어 피드백 | 항목별 점수와 근거 |
| 기록 방식 | 메시지 목록 누적 | 최신 요약과 평가를 필드에 저장 |
| 수정 방법 | 누적된 피드백 참고 | 평가 결과 직접 참고 |
| 종료 판단 | 생성 횟수 | 평가 점수 |
| 주요 설계 요소 | 피드백 내용 | 루브릭과 통과 기준 |

제품 리뷰 예시로 보면 다음과 같다.

```
첫 요약에서 비추천 대상이 빠짐

Reflection
→ "통화를 자주 하는 사용자에게 부적합하다는 내용을 추가해줘."

Evaluator-Optimizer
→ recommendation: 7점
→ 근거: 비추천 대상이 누락됨

다음 생성
→ 비추천 대상을 포함한 요약 작성
```

위 내용은 동작을 설명하기 위한 예시다. 실제 점수와 문장은 LLM 실행마다 달라질 수 있다.

## 4. Parallelization: 독립적인 작업을 동시에 실행하기

여러 작업이 서로의 결과를 기다릴 필요가 없으면 병렬로 실행할 수 있다.

하나의 문서를 법률, 재무, 기술 관점에서 분석할 때 각 분석은 같은 원본만 있으면 시작할 수 있다.

```
         ┌→ 법률 분석 ─┐
START ───┼→ 재무 분석 ─┼→ 종합 → END
         └→ 기술 분석 ─┘
```

순차 실행과 병렬 실행의 차이는 다음과 같다.

```
순차
법률 3초 → 재무 3초 → 기술 3초
총 약 9초

병렬
세 분석을 동시에 시작
총 약 3초 + 추가 처리 시간
```

시간은 이해를 위한 예시다. 실제로는 요청 지연, 동시 실행 제한, 모델 응답 시간 등의 영향을 받는다.

## 5. Fan-out과 Fan-in

Fan-out은 한 곳에서 여러 작업으로 퍼지는 연결이고, Fan-in은 여러 작업의 결과를 모으는 연결이다.

```python
# Fan-out: 세 분석을 병렬로 실행
builder.add_edge(START, "legal_analysis")
builder.add_edge(START, "financial_analysis")
builder.add_edge(START, "technical_analysis")
```

```python
# Fan-in: 세 분석이 모두 끝나면 종합
builder.add_edge(
    [
        "legal_analysis",
        "financial_analysis",
        "technical_analysis",
    ],
    "summarize",
)

builder.add_edge("summarize", END)
```

앞쪽 노드를 리스트로 전달하면 지정한 노드들이 모두 끝난 뒤 다음 노드로 진행한다.

조건에 따라 경로를 선택하는 것과도 구분해야 한다.

```
조건에 따른 경로 선택
→ 조건에 맞는 작업을 실행

현재 병렬 연결
→ 연결한 세 작업을 모두 실행
```

종합 노드가 꼭 필요한 것은 아니다. 결과 목록만 필요하면 각 작업을 `END`에 연결하고, 실행 후 State에서 결과를 꺼낼 수 있다.

```python
builder.add_edge("legal_analysis", END)
builder.add_edge("financial_analysis", END)
builder.add_edge("technical_analysis", END)
```

## 6. operator.add로 병렬 결과 누적하기

여러 노드가 같은 결과 목록에 데이터를 추가하려면 합치는 규칙이 필요하다.

```python
class AnalysisState(TypedDict):
    document: str
    analyses: Annotated[list, operator.add]
    summary: str
```

각 노드는 자신의 결과만 리스트로 반환한다.

```python
return {"analyses": ["법률 분석 결과"]}
return {"analyses": ["재무 분석 결과"]}
```

`operator.add`는 리스트를 `+`로 연결한다.

```python
["법률 분석 결과"] + ["재무 분석 결과"]
```

이런 합치기 규칙을 Reducer라고 한다. 기존 목록 전체를 다시 반환하면 중복으로 누적될 수 있으므로 새 결과만 반환한다.

### add_messages와의 차이

| 구분 | `operator.add` | `add_messages` |
|---|---|---|
| 대표 용도 | 일반 결과 리스트 | 대화 메시지 목록 |
| 기본 동작 | 기존 리스트와 새 리스트 연결 | 새 메시지 추가 |
| 동일 ID 처리 | ID를 확인하지 않음 | 기존 메시지 교체 |

```python
results: Annotated[list, operator.add]
messages: Annotated[list, add_messages]
```

`add_messages`도 메시지를 누적한다. 같은 ID를 다시 전달할 때는 기존 메시지를 수정할 수 있다는 차이가 있다.

## 7. 실행 순서와 결과 정렬 순서는 다르다

결과에 붙이는 번호는 출력 순서를 정하기 위한 값이다.

```python
return {"analyses": [(0, "법률 분석 결과")]}
return {"analyses": [(1, "재무 분석 결과")]}
return {"analyses": [(2, "기술 분석 결과")]}
```

병렬 실행 후 번호를 기준으로 정렬한다.

```python
ordered = sorted(
    state["analyses"],
    key=lambda item: item[0],
)
```

이렇게 하면 법률, 재무, 기술 순서로 정리할 수 있다.

```
실행 순서 제어
→ Edge를 순차로 연결

출력 순서 제어
→ 결과에 번호를 넣고 정렬
```

노드가 완료되는 순서와 최종 State 목록의 순서를 같은 것으로 가정하지 않는 것이 좋다. 원하는 출력 순서는 명시적으로 정렬한다.

## 8. 실습: 영어·일본어·스페인어 병렬 번역

하나의 문장을 세 언어로 번역하고, 영어 → 일본어 → 스페인어 순서로 출력하는 실습이다.

### State와 번역 노드

```python
class TranslationState(TypedDict):
    sentence: str
    translations: Annotated[list, operator.add]
    ordered_result: str
```

```python
def translate_en(state: TranslationState):
    response = llm.invoke(
        "다음 문장을 영어로 번역해줘. 번역문만 출력해.\n\n"
        f"{state['sentence']}"
    )

    return {
        "translations": [(0, "영어", response.text)]
    }


def translate_jp(state: TranslationState):
    response = llm.invoke(
        "다음 문장을 일본어로 번역해줘. 번역문만 출력해.\n\n"
        f"{state['sentence']}"
    )

    return {
        "translations": [(1, "일본어", response.text)]
    }


def translate_sp(state: TranslationState):
    response = llm.invoke(
        "다음 문장을 스페인어로 번역해줘. 번역문만 출력해.\n\n"
        f"{state['sentence']}"
    )

    return {
        "translations": [(2, "스페인어", response.text)]
    }
```

각 결과는 값 세 개가 들어 있는 튜플이다.

```
(순서, 언어, 번역문)
```

제공된 노트북의 번역 풀이에는 `(언어, 번역문)` 형태가 사용되어 있다. 위 코드는 학습 대화에서 작성한 방식으로, 순서를 명확하게 지정하기 위해 번호를 추가했다.

### 정렬하고 하나의 문자열로 합치기

```python
def order_translations(state: TranslationState):
    ordered = sorted(
        state["translations"],
        key=lambda item: item[0],
    )

    ordered_result = "\n\n".join(
        f"[{language}] {content}"
        for _, language, content in ordered
    )

    return {
        "ordered_result": ordered_result
    }
```

`for _, language, content`는 튜플의 값 세 개를 꺼낸다. 순서 번호는 출력에 사용하지 않아서 `_`로 받는다.

`"\n\n".join(...)`은 번역 결과 사이에 줄바꿈 두 개를 넣는다.

```
[영어] Hello.

[일본어] こんにちは。

[스페인어] Hola.
```

처음과 마지막에 줄바꿈을 추가하는 것이 아니라 각 문자열 사이에 구분자를 넣는다.

### 병렬 그래프와 실행

```python
builder = StateGraph(TranslationState)

builder.add_node("english", translate_en)
builder.add_node("japanese", translate_jp)
builder.add_node("spanish", translate_sp)
builder.add_node("order", order_translations)

builder.add_edge(START, "english")
builder.add_edge(START, "japanese")
builder.add_edge(START, "spanish")

builder.add_edge(
    ["english", "japanese", "spanish"],
    "order",
)
builder.add_edge("order", END)

translation_graph = builder.compile()
```

```python
result = translation_graph.invoke({
    "sentence": "안녕하세요. 만나서 반갑습니다.",
    "translations": [],
})

print(result["ordered_result"])
```

`invoke()`가 반환한 `result`에는 최종 State가 들어 있다. 여기서 정렬된 출력 문자열만 꺼내 표시한다.

## 9. 순차 실행과 소요 시간 비교

동일한 노드 함수를 사용하고 Edge만 일렬로 연결하면 순차 그래프가 된다.

```python
sequential_builder = StateGraph(TranslationState)

sequential_builder.add_node("english", translate_en)
sequential_builder.add_node("japanese", translate_jp)
sequential_builder.add_node("spanish", translate_sp)
sequential_builder.add_node("order", order_translations)

sequential_builder.add_edge(START, "english")
sequential_builder.add_edge("english", "japanese")
sequential_builder.add_edge("japanese", "spanish")
sequential_builder.add_edge("spanish", "order")
sequential_builder.add_edge("order", END)

sequential_graph = sequential_builder.compile()
```

같은 입력으로 시간을 측정한다.

```python
import time

sentence = "인공지능은 우리의 일상과 업무 방식을 변화시키고 있습니다."

for name, target_graph in [
    ("병렬", translation_graph),
    ("순차", sequential_graph),
]:
    start = time.perf_counter()

    result = target_graph.invoke({
        "sentence": sentence,
        "translations": [],
    })

    elapsed = time.perf_counter() - start
    print(f"{name}: {elapsed:.2f}초")
```

학습 중 비교에서는 병렬 실행이 더 빠르게 나왔다. 세 번역이 서로 독립적이므로 앞 작업이 끝날 때까지 기다리는 시간을 줄일 수 있었다.

다만 병렬 실행이 언제나 더 빠르다고 단정할 수는 없다. 한 번의 측정값은 당시 API와 네트워크 상태의 영향도 받는다.

## 10. Voting: 여러 후보 중 하나 선택하기

Voting 예제는 여러 방식으로 답변을 생성한 다음 Judge가 가장 좋은 후보 하나를 고르는 구조다.

```
         ┌→ 핵심 중심 답변 ────┐
START ───┼→ 예시 중심 답변 ────┼→ vote → 선택한 답변
         └→ 비유 중심 답변 ────┘
```

후보는 ID와 답변을 함께 저장한다.

```json
{
    "candidates": [{
        "id": 0,
        "answer": "후보 답변"
    }]
}
```

Judge의 결과는 구조화해서 받는다.

```python
class VoteResult(BaseModel):
    best_id: int = Field(description="선택한 후보의 ID")
    reason: str = Field(description="선택 이유")
```

선택된 ID로 실제 답변을 찾는다.

```python
best = next(
    candidate
    for candidate in candidates
    if candidate["id"] == result.best_id
)

return {"best_answer": best["answer"]}
```

여기서 `vote`는 후보를 비교해 선택하는 노드다. 여러 답변을 하나의 새 글로 합치는 `summarize`와 역할이 다르다.

## 11. Orchestrator-worker: 실행 중에 작업을 나누기

앞선 병렬 번역 실습에서는 영어, 일본어, 스페인어라는 작업이 미리 정해져 있었다.

이번에는 사용자 요청을 보고 LLM이 필요한 작업을 계획한다.

```
사용자 요청
→ Orchestrator가 작업 목록 생성
→ 목록의 각 작업을 Worker에게 전달
→ Worker들이 병렬 처리
→ 결과 종합
```

| 구분 | 고정 병렬화 | Orchestrator-worker |
|---|---|---|
| 작업 결정 | 개발자가 미리 정의 | 요청을 보고 실행 중 계획 |
| 병렬 실행 | 등록한 여러 노드 | 같은 Worker를 여러 입력으로 실행 |
| 입력 전달 | 그래프 State | `Send`로 작업별 입력 전달 |
| 예시 | 세 언어 번역 | 요청에 맞는 학습 목차 생성 |

Worker 함수는 하나만 정의해도 된다. 같은 함수가 서로 다른 입력을 받아 여러 작업을 처리한다.

## 12. 구조화 출력과 State는 어떤 기준으로 만들까?

먼저 작업 흐름을 적는다.

```
학습 요청 → 목차 계획 → 목차별 자료 작성 → 전체 자료 완성
```

그다음 두 가지를 구분한다.

```
LLM이 어떤 형태로 응답해야 하는가?
→ 구조화 출력 모델

다음 노드가 사용할 값을 어디에 보관할 것인가?
→ 그래프 State
```

### 목차 하나와 전체 목차

```python
class Section(BaseModel):
    title: str = Field(description="수업 자료 제목")
    objective: str = Field(description="학습 목표")


class CoursePlan(BaseModel):
    sections: list[Section] = Field(
        description="학습 순서에 맞는 목차 목록"
    )
```

`CoursePlan`은 학습 계획, `Section`은 목차 항목 하나를 뜻한다.

```
CoursePlan
└─ sections
   ├─ 제목 + 학습 목표
   ├─ 제목 + 학습 목표
   └─ 제목 + 학습 목표
```

이 구조 덕분에 긴 문자열을 직접 분리하지 않고 다음처럼 값을 사용할 수 있다.

```python
section.title
section.objective
```

### 전체 State와 Worker State

```python
class CourseState(TypedDict):
    request: str
    sections: list[Section]
    results: Annotated[list, operator.add]
    final_result: str


class SectionState(TypedDict):
    request: str
    title: str
    objective: str
    order: int
```

| 값 | 쓰임 |
|---|---|
| `request` | 전체 사용자 요청 |
| `sections` | 계획한 목차 목록 |
| `results` | Worker가 작성한 자료 목록 |
| `final_result` | 합쳐진 최종 학습 자료 |
| `order` | 결과를 원래 목차 순서로 정렬할 번호 |

Worker는 자신의 목차를 작성할 정보만 받는다. 다른 Worker의 입력이나 전체 목차 목록을 직접 찾을 필요가 없다.

## 13. Orchestrator가 목차 계획 만들기

```python
planner = llm.with_structured_output(CoursePlan)


def course_plan(state: CourseState):
    response = planner.invoke(f"""
다음 요청에 맞는 학습 목차를 3~5개 만들어줘.
학습 대상과 시간을 고려하고, 학습 순서대로 작성해줘.
각 목차에는 제목과 학습 목표를 포함해줘.

학습 요청:
{state["request"]}
""")

    return {
        "sections": response.sections
    }
```

이 단계에서는 실제 학습 내용을 작성하지 않는다. 무엇을 작성할지 계획한다.

```
제목: 변수와 자료형
학습 목표: 변수, 숫자, 문자열의 기본 사용법을 이해한다.
```

### response.sections를 꺼내는 이유

```python
return {"sections": response.sections}
```

왼쪽은 State에서 업데이트할 키이고, 오른쪽은 응답 객체에서 꺼낸 목록이다.

```
response
└─ sections 목록
       ↓ 꺼내기
CourseState의 sections에 저장
```

`sections` 안에 또 `sections`를 넣는 것이 아니다. 응답 객체 안의 리스트를 꺼내 State로 옮기는 것이다.

목록의 개수가 고정되어 있어도 이 방식으로 저장할 수 있다. 리스트는 여러 작업을 담고 반복 처리하기 위해 사용한다.

## 14. Send로 Worker에게 작업 배정하기

```python
def assign_sections(state: CourseState):
    return [
        Send(
            "write_section",
            {
                "request": state["request"],
                "title": section.title,
                "objective": section.objective,
                "order": index,
            },
        )
        for index, section in enumerate(state["sections"])
    ]
```

`enumerate()`는 목록에서 항목을 꺼내면서 번호도 붙인다.

```
0 → 파이썬 소개
1 → 변수와 자료형
2 → 조건문
```

`Send`의 첫 번째 값은 실행할 노드 이름이고, 두 번째 값은 그 작업에 전달할 입력이다.

```python
Send(
    "write_section",
    {
        "request": "초보자를 위한 파이썬 학습 자료",
        "title": "변수와 자료형",
        "objective": "변수와 기본 자료형을 이해한다.",
        "order": 1,
    },
)
```

입력 딕셔너리는 코드에서 만들고, `Send`는 그 입력으로 지정한 노드를 실행하도록 전달한다.

```
Worker용 입력 준비
→ Send로 전달
→ Worker 실행
→ 결과 반환
```

Worker 실행이 끝난 뒤에 입력 State를 만드는 것이 아니다.

`assign_sections()`는 학습 내용을 작성하는 Worker가 아니라 작업을 배정하는 라우팅 함수다.

## 15. Worker가 자료 작성하고 결과 합치기

### Worker

```python
def write_section(state: SectionState):
    response = llm.invoke(f"""
다음 요청과 담당 목차를 바탕으로 학습 자료를 작성해줘.
초보자가 이해하기 쉬운 설명과 간단한 예제를 포함해줘.
마크다운 형식으로 작성해줘.

전체 요청:
{state["request"]}

목차 제목:
{state["title"]}

학습 목표:
{state["objective"]}
""")

    return {
        "results": [{
            "title": state["title"],
            "objective": state["objective"],
            "content": response.text,
            "order": state["order"],
        }]
    }
```

이 단계에서 실제 설명과 예제가 만들어진다.

```
Worker 입력
→ 제목 + 학습 목표

Worker 생성
→ 실제 설명 + 예제

Worker 반환
→ 제목 + 목표 + 내용 + 순서
```

Worker별 `results`는 전체 State의 `operator.add`를 통해 하나의 목록으로 합쳐진다.

### 최종 자료 합치기

```python
def merge_results(state: CourseState):
    ordered = sorted(
        state["results"],
        key=lambda item: item["order"],
    )

    final_result = "\n\n".join(
        f"## {item['title']}\n"
        f"학습 목표: {item['objective']}\n\n"
        f"{item['content']}"
        for item in ordered
    )

    return {"final_result": final_result}
```

강사님 학습 자료 풀이에서는 마지막에 LLM을 다시 호출하지 않고 문자열을 합친다.

| 값 | 저장 형태 |
|---|---|
| `results` | 목차별 자료가 담긴 리스트 |
| `final_result` | 모든 자료를 연결한 문자열 |

목차별 결과만 필요하면 `results`만 사용해도 된다. 최종 문자열을 다른 곳에서도 사용하려면 `final_result`에 저장하면 편하다.

## 16. 동적 병렬 그래프 연결과 실행

```python
builder = StateGraph(CourseState)

builder.add_node("course_plan", course_plan)
builder.add_node("write_section", write_section)
builder.add_node("merge_results", merge_results)

builder.add_edge(START, "course_plan")

builder.add_conditional_edges(
    "course_plan",
    assign_sections,
    ["write_section"],
)

builder.add_edge("write_section", "merge_results")
builder.add_edge("merge_results", END)

course_graph = builder.compile()
```

이 부분은 다음과 같이 읽는다.

```python
builder.add_conditional_edges(
    "course_plan",       # 계획을 만든 다음
    assign_sections,     # 작업별 Send를 만들고
    ["write_section"],   # 이 노드로 보낼 수 있음
)
```

실제 Worker 실행 개수와 입력은 `assign_sections()`가 반환한 `Send` 목록이 결정한다.

```
목차 3개 → Send 3개 → Worker 실행 3개
목차 5개 → Send 5개 → Worker 실행 5개
```

이 예제처럼 같은 단계에서 배정된 Worker들이 한 단계로 끝나는 구조에서는 모든 Worker 결과가 합쳐진 뒤 `merge_results`가 실행된다.

### 그래프 그림 확인

```python
display(
    Image(
        course_graph.get_graph().draw_mermaid_png()
    )
)
```

그림에는 Worker 노드가 하나로 보일 수 있다. 이는 등록한 함수가 하나이기 때문이며, 실제로는 `Send` 수만큼 작업이 실행된다.

### 실행 과정 출력

```python
course_request = (
    "파이썬을 처음 배우는 비전공자를 위한 "
    "2시간 분량의 학습 자료를 만들어줘."
)

for event in course_graph.stream(
    {"request": course_request},
    stream_mode="updates",
):
    for node_name, update in event.items():
        print(f"\n[{node_name}]")

        if node_name == "course_plan":
            for section in update["sections"]:
                print(section.title)

        elif node_name == "write_section":
            item = update["results"][0]
            print(f"{item['title']} 작성 완료")

        elif node_name == "merge_results":
            print(update["final_result"])
```

`updates`는 각 노드가 반환한 변경분을 보여준다.

```
course_plan → 목차 목록
write_section → 해당 Worker의 자료
merge_results → 최종 학습 자료
```

최종 State만 받고 싶으면 `invoke()`를 사용한다.

```python
result = course_graph.invoke({
    "request": course_request
})

print(result["final_result"])
```

`invoke()`와 `stream()`은 각각 새로 그래프를 실행한다. 이미 받은 결과를 확인하려면 저장한 `result`를 읽으면 된다.

## 17. 동시에 실행하는 작업 수 제한하기

작업이 많으면 LLM 요청도 한꺼번에 늘어난다.

```python
result = course_graph.invoke(
    {"request": course_request},
    config={"max_concurrency": 3},
)
```

`max_concurrency=3`은 전체 작업을 3개로 줄이는 설정이 아니다. 한 번에 실행하는 작업 수를 최대 3개로 제한한다.

```
작업 5개
→ 먼저 최대 3개 실행
→ 빈자리가 생기면 남은 작업 실행
→ 최종적으로 5개 모두 처리
```

이 설정은 동시 실행 수를 제한하지만 분당 요청 수나 토큰 수를 직접 관리하는 것은 아니다.

## 18. 추가 실습: 지원자 정보로 면접 질문 만들기

추가 실습은 지원자의 직무와 경력 등을 보고 LLM이 평가 역량을 정하고, 역량별 Worker가 질문을 만드는 과제다.

```
지원자 정보
→ 평가 역량 계획
→ 역량별 Send
→ 질문과 모범 답안 생성
→ 면접 문제 목록 완성
```

직무, 경력, 자격증은 지원자 입력 정보다. 이를 바탕으로 평가할 역량을 LLM이 판단한다.

```
입력 정보
신입 AI 백엔드 개발자
Python, FastAPI 프로젝트 경험

평가 역량 예시
파이썬 기초 / API 설계 / 데이터 처리 / 문제 해결
```

제공된 `19-orchestrator-worker.ipynb`에는 추가 실습 문제만 있으며, 아래는 학습 대화에서 작성한 면접 실습 구조를 정리한 것이다.

### 구조화 출력과 State

```python
class InterviewSection(BaseModel):
    classification: str = Field(
        description="면접에서 평가할 역량"
    )
    question_detail: str = Field(
        description="해당 역량에서 구체적으로 평가할 내용"
    )


class InterviewPlan(BaseModel):
    sections: list[InterviewSection]


class InterviewState(TypedDict):
    request: str
    sections: list[InterviewSection]
    results: Annotated[list, operator.add]
    final_result: str


class InterviewWorkerState(TypedDict):
    request: str
    classification: str
    question_detail: str
```

Orchestrator의 출력에는 아직 질문과 답안이 없다.

```
Orchestrator 출력
분류: 파이썬 기초
평가할 내용: 자료형의 특징과 사용 상황에 대한 이해
```

Worker가 이 정보를 받아 실제 질문과 모범 답안을 작성한다.

```
Worker 출력
질문: 리스트와 튜플의 차이와 적절한 사용 상황을 설명해주세요.
모범 답안: ...
```

### 평가 역량 계획

```python
interview_planner = llm.with_structured_output(
    InterviewPlan
)


def interview_plan(state: InterviewState):
    response = interview_planner.invoke(f"""
지원자 정보를 바탕으로 평가 역량을 정확히 5개 만들어줘.
각 역량에는 분류와 평가할 내용을 포함해줘.

지원자 정보:
{state["request"]}
""")

    return {"sections": response.sections}
```

질문을 총 5개로 만들려면 계획 단계의 역량 수와 Worker당 질문 수를 함께 제한해야 한다.

### 역량별 배정

```python
def assign_interview_sections(state: InterviewState):
    return [
        Send(
            "question_section",
            {
                "request": state["request"],
                "classification": section.classification,
                "question_detail": section.question_detail,
            },
        )
        for section in state["sections"]
    ]
```

출력 순서를 따로 지정하지 않기로 했으므로 번호는 전달하지 않았다.

### 질문과 모범 답안 작성

```python
def question_section(state: InterviewWorkerState):
    response = llm.invoke(f"""
다음 정보를 바탕으로 면접 질문 1개와 모범 답안 1개만 작성해줘.

지원자 정보:
{state["request"]}

평가 역량:
{state["classification"]}

평가할 내용:
{state["question_detail"]}

다음 마크다운 형식을 사용해줘.

### 질문
질문 내용

### 모범 답안
모범 답안 내용
""")

    return {
        "results": [{
            "classification": state["classification"],
            "question_detail": state["question_detail"],
            "content": response.text,
        }]
    }
```

이번에는 질문과 답안을 `content` 하나에 담았다.

```json
{
    "classification": "파이썬 기초",
    "question_detail": "자료형 이해도 평가",
    "content": "### 질문\n...\n\n### 모범 답안\n..."
}
```

프롬프트의 형식 지정은 보기 좋은 출력을 유도한다. 질문과 답안을 각각 코드에서 처리해야 한다면 Worker에도 별도의 구조화 출력 모델을 적용할 수 있다.

### 결과 합치기와 그래프

```python
def merge_question(state: InterviewState):
    final_result = "\n\n".join(
        f"## {item['classification']}\n"
        f"평가할 내용: {item['question_detail']}\n\n"
        f"{item['content']}"
        for item in state["results"]
    )

    return {"final_result": final_result}
```

```python
builder = StateGraph(InterviewState)

builder.add_node("interview_plan", interview_plan)
builder.add_node("question_section", question_section)
builder.add_node("merge_question", merge_question)

builder.add_edge(START, "interview_plan")

builder.add_conditional_edges(
    "interview_plan",
    assign_interview_sections,
    ["question_section"],
)

builder.add_edge("question_section", "merge_question")
builder.add_edge("merge_question", END)

interview_graph = builder.compile()
```

```python
request = """
지원 직무: 신입 AI 백엔드 개발자
경력: 신입
프로젝트 경험: Python, FastAPI, LangGraph를 활용한 AI 서비스 개발
자격증: 정보처리기사
"""

result = interview_graph.invoke({
    "request": request
})

print(result["final_result"])
```

의도한 생성 개수는 다음과 같다.

```
평가 역량 5개 × 역량당 질문 1개 = 총 5문제
```

프롬프트만으로 개수가 항상 보장되는 것은 아니므로, 정확한 개수가 중요하면 실제 생성 개수도 확인해야 한다.

## 19. 실습 중 발생한 오류와 확인 방법

### Send에 함수 자체를 넣은 오류

```python
Send(question_section, {...})
```

`Send`의 목적지는 등록한 노드 이름 문자열이어야 한다.

```python
Send("question_section", {...})
```

실습에서는 다음 경고가 나타났다.

```
Ignoring unknown node name <function question_section ...>
```

Worker 작업이 무시되면서 결과 병합까지 진행되지 않았고, `final_result`도 만들어지지 않았다.

### request와 result를 혼동한 오류

```python
print(request["final_result"])
```

`request`는 입력 문자열이다. 문자열에서 `"final_result"`라는 키를 찾으려고 하면 오류가 발생한다.

```python
result = interview_graph.invoke({
    "request": request
})

print(result["final_result"])
```

입력은 `request`, 실행 결과는 `result`에 들어간다.

### State 키의 철자가 다른 오류

```
section ↔ sections
orderde_result ↔ ordered_result
final_report ↔ final_result
```

State 정의, 노드 반환값, 조회 코드가 같은 이름을 사용해야 한다.

```python
return {"final_result": final_result}
print(result["final_result"])
```

### 함수 수정 후 이전 그래프를 실행한 경우

함수 코드를 수정한 뒤에는 그래프 생성 셀도 다시 실행한다.

```
State와 함수 정의 셀
→ builder 생성·노드 연결·compile 셀
→ 실행 셀
```

노트북에서는 코드가 보이는 순서와 실제 실행 순서가 다를 수 있다. 오류가 계속되면 관련 셀을 위에서부터 일관된 순서로 다시 실행한다.

## 20. 이번 학습에서 기억할 점

- 작업 흐름을 먼저 적고 각 노드가 받는 값과 만드는 값을 구분한다.
- 구조화 출력은 LLM 응답 형식이고, State는 그래프에서 이어 사용할 데이터다.
- Reflection은 피드백을 반영하고, Evaluator-Optimizer는 평가 결과를 바탕으로 개선한다.
- 독립적인 작업은 병렬로 실행할 수 있다.
- Fan-out은 여러 작업으로 나누고, Fan-in은 결과를 모은다.
- `operator.add`는 일반 리스트를 연결하고, `add_messages`는 메시지 ID를 고려해 추가하거나 수정한다.
- 실행 순서와 출력 정렬 순서는 별개다.
- Orchestrator는 작업을 계획하고, 라우팅 함수는 `Send`로 배정하며, Worker는 실제 내용을 만든다.
- `Send`에는 노드 이름 문자열과 해당 작업의 입력을 전달한다.
- Worker 입력에는 작업 지시가 들어가고, Worker 출력에는 생성 결과가 들어간다.
- `results`는 개별 결과 목록이고, `final_result`는 합친 최종 문자열이다.
- 화면에 보여주는 용도라면 `content` 하나에 저장하고 프롬프트로 형식을 지정할 수 있다.
- State에 저장된 정보도 LLM이 사용하려면 프롬프트에 직접 넣어야 한다.

## 21. 헷갈린 점

- 처음에는 병렬 실행과 결과 정렬을 같은 의미로 생각했다. 결과에 붙인 번호는 노드의 실행 순서를 정하는 값이 아니라 출력 순서를 정리하기 위한 값이었다. 실제 실행 순서가 필요하면 Edge를 일렬로 연결해야 한다.
- Orchestrator-worker에서는 함수보다 구조화 출력과 State를 만드는 부분이 더 어려웠다. LLM이 어떤 형식으로 계획을 반환할지와, 각 단계에서 무엇을 저장하고 전달할지를 나눠 생각하니 구조가 보이기 시작했다.
- 면접 문제 실습에서는 계획 단계에 문제와 정답까지 넣으려고 했다. 하지만 Orchestrator는 평가할 역량과 내용을 정하고, Worker가 실제 질문과 모범 답안을 만드는 방식으로 역할을 나눠야 했다. Worker 입력과 출력이 서로 다르다는 점을 이해하는 데 도움이 됐다.
- `return {"sections": response.sections}`도 처음에는 같은 이름 안에 같은 이름을 넣는 것처럼 보였다. 실제로는 응답 객체에서 목록을 꺼내 State의 해당 필드에 저장하는 코드였다.
- 이번 실습을 통해 전체 코드를 외우기보다 **각 함수가 무엇을 받고, 무엇을 만들고, 다음 단계에 무엇을 넘기는지**를 먼저 정하는 것이 중요하다는 점을 배웠다.
