## LangGraph는 연결된 선으로 이루어져 있다고 생각하면 된다.

## 그래서 중간 단계를 기억해서 그 부분부터 재실행할 수 있다.

```markdown
# LangGraph 기초 — State, Node, Edge로 분기와 반복 만들기

> LangGraph의 핵심은 **작업 결과를 State에 저장하고, 그 상태에 따라 다음 작업을 실행하는 것**이다.

이번에는 기본 그래프부터 조건 분기, 반복, Reducer를 살펴보고, 마지막에는 **글 작성 → 검토 → 피드백 반영 → 재작성** 흐름으로 연결해 본다.

노트북에 있는 여러 풀이를 합쳐서 정리했다. 예제의 변수명과 종료 조건은 이해하기 쉽도록 일부 보완했다.
```

```markdown
## 1. 왜 LangGraph가 필요할까?

단순한 LLM 작업은 다음과 같은 흐름으로 구현할 수 있다.
```

```
입력 → 프롬프트 → LLM → 출력 파서 → 결과
```

```markdown
하지만 다음과 같은 요구사항이 추가되면 흐름이 복잡해진다.

- 검색 결과가 부족하면 질문을 바꾸어 다시 검색한다.
- 사용자의 입력을 분류하고 서로 다른 응답을 생성한다.
- 작성한 글을 검토하고 품질이 부족하면 다시 작성한다.
- 작업을 반복하되 최대 시도 횟수에 도달하면 중단한다.

일반 Python의 `if`, `for`, `while`로도 구현할 수 있다. 다만 조건과 반복이 늘어날수록 **작업을 수행하는 코드와 실행 순서를 관리하는 코드가 뒤섞이기 쉽다.**

LangGraph는 이러한 흐름을 **상태와 작업, 연결 관계**로 표현한다.
```

```
검색 → 결과 검토 → 충분함 → 종료
         └─────→ 부족함 → 질문 변경 → 검색
```

```markdown
노트북의 첫 검색 예제도 같은 문제를 보여준다. Chroma에서 문서를 검색하고 LLM으로 충분성을 판단하지만, 재검색 여부와 최대 횟수는 Python 반복문이 제어한다.

참고로 해당 예제는 기존 `./chroma_db`의 `spri_ai_brief` 컬렉션을 사용한다. 문서를 새로 적재하는 코드는 없으므로 기존 검색 데이터가 있어야 재현할 수 있다.

### Chain, Workflow, Agent의 차이

| 구분 | 실행 흐름을 결정하는 방식 | 예시 |
|---|---|---|
| 단순 Chain | 정해진 순서대로 처리 | 번역 → 요약 |
| Workflow | 개발자가 분기와 반복 규칙을 설계 | 검토 실패 시 재작성 |
| Agent | 주어진 목표와 도구 안에서 LLM이 다음 행동을 선택 | 검색할지 계산할지 결정 |

LangChain에도 분기와 병렬 구성 기능이 있으므로, "Chain은 직선 흐름만 가능하다"는 뜻은 아니다.

또한 **LLM이 들어 있다고 모두 Agent는 아니다.** 이번 실습은 개발자가 노드와 경로를 미리 설계하므로 Workflow에 가깝다.
```

```markdown
## 2. 핵심 개념: State, Node, Edge

| 개념 | 역할 | 예시 |
|---|---|---|
| **State** | 그래프에서 공유하는 데이터 | 질문, 답변, 피드백, 시도 횟수 |
| **Node** | 상태를 읽고 작업하는 함수 | 답변 생성, 감정 분석, 글 검토 |
| **Edge** | 다음에 실행할 노드를 연결 | 분석 후 응답 생성 |
| **Reducer** | 기존 값과 새 값을 합치는 규칙 | 메시지·로그 누적 |

전체 실행 방식은 다음과 같다.
```

```
현재 State
   ↓
Node가 필요한 데이터를 읽고 작업
   ↓
Node가 변경분을 반환
   ↓
변경분을 State에 반영
   ↓
Edge에 따라 다음 Node 실행
```

```markdown
예를 들어 현재 상태가 다음과 같다고 하자.
```

```json
{"question": "LangGraph가 뭐야?"}
```

```markdown
노드가 아래 값을 반환하면:
```

```json
{"answer": "상태와 그래프로 작업 흐름을 구성하는 프레임워크입니다."}
```

```markdown
최종 상태는 다음처럼 된다.
```

```json
{
    "question": "LangGraph가 뭐야?",
    "answer": "상태와 그래프로 작업 흐름을 구성하는 프레임워크입니다."
}
```

```markdown
**노드는 전체 State를 다시 반환하지 않아도 된다.** 업데이트할 필드만 반환하면 된다. 같은 필드에 Reducer가 없다면 새 값으로 덮어쓴다. [공식 Graph API 문서](https://docs.langchain.com/oss/python/langgraph/graph-api)
```

```markdown
## 3. 공통 실행 준비

아래 예제들은 공통 import와 모델 설정을 먼저 실행한 상태를 기준으로 한다.

`pip install langgraph langchain-core langchain-google-genai python-dotenv pydantic`

`.env`에는 API 키와 사용 가능한 모델 ID를 설정한다.
```

```
GOOGLE_API_KEY=본인의_API_키
GOOGLE_MODEL=사용_가능한_Gemini_모델_ID
```

```python
import os
import operator
from typing import Annotated, Literal, NotRequired, TypedDict

from dotenv import load_dotenv
from pydantic import BaseModel
from langchain_core.messages import AIMessage, HumanMessage
from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.graph import START, END, StateGraph
from langgraph.graph.message import add_messages

load_dotenv()

llm = ChatGoogleGenerativeAI(
    model=os.environ["GOOGLE_MODEL"]
)
```

```markdown
이 코드는 Python 3.11 이상을 기준으로 한다. 원본의 모델 ID를 고정해서 복사하기보다, 자신의 계정에서 사용할 수 있는 모델을 지정하도록 구성했다.

### TypedDict란?

딕셔너리에 어떤 키가 있고 각 값의 타입이 무엇인지 표현하는 도구다.
```

```python
class ChatState(TypedDict):
    question: str
    answer: NotRequired[str]
```

```markdown
- `question`: 입력으로 필요한 값
- `answer`: 실행 중 나중에 채워질 수 있는 값

`TypedDict`를 선언한다고 기본값이나 런타임 검증이 자동으로 생기는 것은 아니다. 실제 실행에서는 일반 딕셔너리처럼 사용한다.
```

```markdown
## 4. 가장 기본적인 그래프 만들기

먼저 질문에 답하는 노드 하나만 연결한다.
```

```
START → chatbot → END
```

```python
class ChatState(TypedDict):
    question: str
    answer: NotRequired[str]


def chatbot(state: ChatState):
    response = llm.invoke(state["question"])
    return {"answer": response.text}


builder = StateGraph(ChatState)

builder.add_node("chatbot", chatbot)

builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)

chat_graph = builder.compile()

result = chat_graph.invoke({
    "question": "LangGraph가 뭐야?"
})

print(result["answer"])
```

```markdown
### 코드 실행 순서

1. `StateGraph(ChatState)`로 상태 구조를 지정한다.
2. `add_node()`로 실행할 함수를 등록한다.
3. `add_edge()`로 실행 순서를 연결한다.
4. `compile()`로 실행 가능한 그래프를 만든다.
5. `invoke()`에 초기 상태를 넣어 실행한다.
```

```python
builder.add_node("chatbot", chatbot)
```

```markdown
여기서 첫 번째 `"chatbot"`은 **그래프에서 사용할 노드 이름**, 두 번째 `chatbot`은 **실제 Python 함수**다.

`compile()`은 그래프를 구성하는 단계이며, 이때 LLM이 호출되는 것은 아니다.

### Node의 기본 작성 패턴
```

```python
def node(state):
    # 1. 필요한 상태 읽기
    value = state["some_key"]

    # 2. 작업 수행
    result = ...

    # 3. 변경분 반환
    return {"result": result}
```

```markdown
## 5. 조건부 Edge: 언어에 따라 다른 응답 생성하기

입력 언어가 한국어면 한국어 답변을, 영어면 영어 답변을 생성한다.
```

```
START → detect_language ── ko → answer_ko → END
                        └─ en → answer_en → END
```

```python
class RouterState(TypedDict):
    question: str
    language: NotRequired[str]
    answer: NotRequired[str]


def detect_language(state: RouterState):
    response = llm.invoke(
        "다음 문장의 언어가 한국어면 ko, 영어면 en만 출력해: "
        + state["question"]
    )
    return {"language": response.text.strip().lower()}


def answer_ko(state: RouterState):
    response = llm.invoke(
        "한국어로 답변해줘: " + state["question"]
    )
    return {"answer": response.text}


def answer_en(state: RouterState):
    response = llm.invoke(
        "Answer in English: " + state["question"]
    )
    return {"answer": response.text}


def route_by_language(state: RouterState):
    language = state["language"]

    if language not in ("ko", "en"):
        raise ValueError(f"예상하지 못한 언어 판정: {language!r}")

    return language


builder = StateGraph(RouterState)

builder.add_node("detect_language", detect_language)
builder.add_node("answer_ko", answer_ko)
builder.add_node("answer_en", answer_en)

builder.add_edge(START, "detect_language")

builder.add_conditional_edges(
    "detect_language",
    route_by_language,
    {
        "ko": "answer_ko",
        "en": "answer_en",
    },
)

builder.add_edge("answer_ko", END)
builder.add_edge("answer_en", END)

router_graph = builder.compile()

result = router_graph.invoke({
    "question": "파이썬의 장점이 뭐야?"
})

print(result)
```

```markdown
### Node와 라우팅 함수의 차이
```

```python
def detect_language(state):
    return {"language": "ko"}
```

```markdown
노드는 **상태에 반영할 딕셔너리**를 반환한다.
```

```python
def route_by_language(state):
    return "ko"
```

```markdown
라우팅 함수는 이 패턴에서 **다음 경로를 선택할 값**을 반환한다.
```

```python
{"ko": "answer_ko"}
```

```markdown
이 매핑을 통해 `"ko"`가 반환되면 `"answer_ko"` 노드로 이동한다.

따라서 라우팅 함수의 반환값과 실제 노드 이름은 같을 필요가 없다.

> 원본은 한국어가 아니면 영어로 보내지만, 위 예제는 잘못된 분류 결과를 확인할 수 있도록 검사를 추가했다.
```

```markdown
## 6. invoke와 stream의 차이

`invoke()`는 실행이 끝난 뒤 최종 결과를 보고 싶을 때 사용한다.
```

```python
result = router_graph.invoke({
    "question": "파이썬의 장점이 뭐야?"
})
```

```markdown
`stream()`은 실행 중 어떤 노드가 어떤 값을 반환했는지 관찰할 때 사용한다.

### updates: 노드가 반환한 변경분
```

```python
for event in router_graph.stream(
    {"question": "파이썬의 장점이 뭐야?"},
    stream_mode="updates",
):
    print(event)
```

```markdown
출력 형태 예시:
```

```
{'detect_language': {'language': 'ko'}}
{'answer_ko': {'answer': '파이썬의 장점은 ...'}}
```

```markdown
### values: 변경분이 반영된 전체 상태
```

```python
for state in router_graph.stream(
    {"question": "파이썬의 장점이 뭐야?"},
    stream_mode="values",
):
    print(state)
```

```markdown
출력 형태 예시:
```

```
{'question': '파이썬의 장점이 뭐야?'}

{'question': '파이썬의 장점이 뭐야?', 'language': 'ko'}

{
    'question': '파이썬의 장점이 뭐야?',
    'language': 'ko',
    'answer': '파이썬의 장점은 ...'
}
```

```markdown
| 방식 | 확인하는 대상 |
|---|---|
| `invoke()` | 최종 State |
| `stream_mode="updates"` | 노드별 변경분 |
| `stream_mode="values"` | 단계별 전체 State |

토큰 단위 모델 출력을 전달하는 스트리밍과 상태 업데이트 스트리밍은 구분해야 한다. LangGraph에서는 토큰 출력을 위한 `messages` 모드도 제공한다. [공식 Streaming 문서](https://docs.langchain.com/oss/python/langgraph/streaming)

또한 **각 `invoke()`와 `stream()` 호출은 새로운 실행**이다. `stream()`이 이미 끝난 실행의 기록을 읽어 주는 것은 아니다.
```

```markdown
## 7. 반복: count가 5가 될 때까지 실행하기

조건부 Edge의 목적지를 이전 노드로 지정하면 반복을 만들 수 있다.
```

```
START → increment → count < 5 → increment
                  └ count ≥ 5 → END
```

```python
class CountState(TypedDict):
    count: int


def increment(state: CountState):
    return {"count": state["count"] + 1}


def route_by_count(state: CountState):
    if state["count"] >= 5:
        return "end"
    return "continue"


builder = StateGraph(CountState)

builder.add_node("increment", increment)
builder.add_edge(START, "increment")

builder.add_conditional_edges(
    "increment",
    route_by_count,
    {
        "continue": "increment",
        "end": END,
    },
)

count_graph = builder.compile()

for event in count_graph.stream(
    {"count": 0},
    stream_mode="updates",
):
    print(event)
```

```markdown
결과:
```

```
{'increment': {'count': 1}}
{'increment': {'count': 2}}
{'increment': {'count': 3}}
{'increment': {'count': 4}}
{'increment': {'count': 5}}
```

```markdown
중요한 순서는 다음과 같다.
```

```
increment 실행
→ count 갱신
→ 갱신된 count로 경로 판단
```

```markdown
따라서 `count=5`가 된 직후 종료한다.

이 그래프는 증가 작업을 먼저 실행한다. 처음부터 `count=5`를 넣으면 6으로 증가한 뒤 종료한다는 점도 흐름을 읽을 때 주의해야 한다.
```

```markdown
## 8. Reducer: 덮어쓰기와 누적

State의 필드는 기본적으로 새 값으로 덮어쓴다.

하지만 로그나 대화 기록은 이전 값을 유지하면서 새 값을 추가해야 한다. 이때 Reducer를 사용한다.
```

```python
class ReducerState(TypedDict):
    name: NotRequired[str]
    messages: Annotated[list, add_messages]
    logs: Annotated[list[str], operator.add]
```

```markdown
| 필드 | 갱신 방식 |
|---|---|
| `name` | 새 값으로 덮어쓰기 |
| `messages` | 메시지 전용 규칙으로 병합 |
| `logs` | 기존 리스트와 새 리스트 연결 |

### 실행 예제
```

```python
def step_a(state: ReducerState):
    return {
        "name": "A가 설정",
        "messages": [HumanMessage(content="안녕")],
        "logs": ["A 실행"],
    }


def step_b(state: ReducerState):
    return {
        "name": "B가 덮어씀",
        "messages": [AIMessage(content="반가워!")],
        "logs": ["B 실행"],
    }


builder = StateGraph(ReducerState)

builder.add_node("a", step_a)
builder.add_node("b", step_b)

builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("b", END)

reducer_graph = builder.compile()

result = reducer_graph.invoke({
    "messages": [],
    "logs": [],
})

print(result["name"])
print(result["logs"])
print([(m.type, m.content) for m in result["messages"]])
```

```markdown
결과:
```

```
B가 덮어씀
['A 실행', 'B 실행']
[('human', '안녕'), ('ai', '반가워!')]
```

```markdown
### Annotated는 무슨 뜻일까?
```

```python
Annotated[list[str], operator.add]
```

```markdown
- `list[str]`: 값의 타입
- `operator.add`: LangGraph가 상태를 갱신할 때 사용할 병합 함수

리스트의 경우 다음처럼 동작한다.
```

```python
["A 실행"] + ["B 실행"]
# ["A 실행", "B 실행"]
```

```markdown
### add_messages와 operator.add의 차이

`operator.add`는 단순히 리스트를 이어 붙인다.

`add_messages`는 메시지 형식을 처리하며, **같은 ID의 메시지가 들어오면 기존 메시지를 교체할 수도 있다.** 따라서 대화 기록에는 메시지 전용 Reducer를 사용하는 것이 적합하다. [공식 메시지 상태 설명](https://docs.langchain.com/oss/python/langgraph/graph-api#working-with-messages-in-graph-state)

누적 필드에는 새 항목만 반환해야 한다.
```

```python
return {"logs": ["이번 실행"]}
```

```markdown
기존 로그까지 함께 반환하면 Reducer가 다시 합치면서 항목이 중복될 수 있다.

또한 메시지를 State에 저장했다고 모델이 자동으로 대화를 기억하는 것은 아니다. 다음 모델 호출에도 해당 메시지를 입력해야 한다.
```

```markdown
## 9. 실습 1: 감정 분석 라우터

언어 분기와 같은 구조로 감정에 따라 답변을 바꿀 수 있다.
```

```
START → analyze → positive → respond_positive → END
                └ negative → respond_negative → END
```

```python
class SentimentState(TypedDict):
    text: str
    sentiment: NotRequired[str]
    response: NotRequired[str]


def analyze(state: SentimentState):
    result = llm.invoke(
        "다음 문장의 감정을 positive 또는 negative 하나로만 분류해: "
        + state["text"]
    )
    return {"sentiment": result.text.strip().lower()}


def respond_positive(state: SentimentState):
    result = llm.invoke(
        "기쁨에 공감하며 답해줘: " + state["text"]
    )
    return {"response": result.text}


def respond_negative(state: SentimentState):
    result = llm.invoke(
        "따뜻하게 위로해줘: " + state["text"]
    )
    return {"response": result.text}


def route_by_sentiment(state: SentimentState):
    label = state["sentiment"]

    if label not in ("positive", "negative"):
        raise ValueError(f"예상하지 못한 감정 판정: {label!r}")

    return label


builder = StateGraph(SentimentState)

builder.add_node("analyze", analyze)
builder.add_node("respond_positive", respond_positive)
builder.add_node("respond_negative", respond_negative)

builder.add_edge(START, "analyze")

builder.add_conditional_edges(
    "analyze",
    route_by_sentiment,
    {
        "positive": "respond_positive",
        "negative": "respond_negative",
    },
)

builder.add_edge("respond_positive", END)
builder.add_edge("respond_negative", END)

sentiment_graph = builder.compile()

result = sentiment_graph.invoke({
    "text": "오늘 승진했어! 너무 기뻐!"
})

print(result["response"])
```

```markdown
원본 강사 풀이에서는 분석 노드 안에 다음 Chain을 넣는다.
```

```python
chain = prompt | llm | StrOutputParser()
```

```markdown
즉, **노드 내부 작업에는 Chain을 사용하고, 노드 사이 실행 흐름에는 LangGraph를 사용할 수 있다.**
```

```markdown
## 10. 실습 2: 숫자 맞히기

LLM이 숫자를 추측하면 정답과 비교하고, 틀렸다면 다음 추측 범위를 좁힌다.
```

```
START → guess → check → 정답 또는 횟수 소진 → END
          ↑       │
          └───────┘ 오답이며 기회가 남음
```

```markdown
정답이 37인 경우:
```

```
50 추측 → 너무 큼 → high = 49
25 추측 → 너무 작음 → low = 26
다음 추측 범위: 26~49
```

```markdown
원본의 범위 조정과 최대 5회 제한을 합친 예제다.
```

```python
class GuessState(TypedDict):
    target: int
    low: int
    high: int
    attempt: int
    guess: NotRequired[int]
    status: NotRequired[Literal["retry", "success", "exhausted"]]


def guess_number(state: GuessState):
    response = llm.invoke(
        f"{state['low']}부터 {state['high']} 사이의 정수 하나만 출력해."
    )
    value = int(response.text.strip())

    if not state["low"] <= value <= state["high"]:
        raise ValueError("추측값이 현재 범위를 벗어났습니다.")

    return {
        "guess": value,
        "attempt": state["attempt"] + 1,
    }


def check_guess(state: GuessState):
    value = state["guess"]
    target = state["target"]

    # 마지막 시도에서 정답을 맞힌 경우도 성공으로 처리한다.
    if value == target:
        return {"status": "success"}

    if state["attempt"] >= 5:
        return {"status": "exhausted"}

    if value < target:
        return {"low": value + 1, "status": "retry"}

    return {"high": value - 1, "status": "retry"}


def route_guess(state: GuessState):
    return state["status"]


builder = StateGraph(GuessState)

builder.add_node("guess", guess_number)
builder.add_node("check", check_guess)

builder.add_edge(START, "guess")
builder.add_edge("guess", "check")

builder.add_conditional_edges(
    "check",
    route_guess,
    {
        "retry": "guess",
        "success": END,
        "exhausted": END,
    },
)

guess_graph = builder.compile()

for event in guess_graph.stream(
    {"target": 37, "low": 1, "high": 100, "attempt": 0},
    stream_mode="updates",
):
    print(event)
```

```markdown
이 실습에서 각 역할은 다음과 같다.

- `guess`: 현재 범위를 읽고 숫자를 추측한다.
- `check`: 정답 여부와 시도 횟수를 확인하고 범위를 수정한다.
- `route_guess`: 종료하거나 다시 추측하도록 경로를 선택한다.

5회 제한은 **성공을 보장하는 조건이 아니라 종료를 보장하는 조건**이다.

또한 모델이 정수 외의 문장을 출력하면 `int()` 변환이 실패한다. 프롬프트만으로 출력 형식이 완전히 보장되는 것은 아니다.
```

```markdown
## 11. 실습 3: 글 작성 → 검토 → 재작성

이번에는 State, 조건 분기, 반복을 함께 사용한다.
```

```
START → write → review
          ↑       ├─ pass → END
          │       ├─ fail + 최대 횟수 도달 → END
          └───────┘ fail + 기회 남음
```

```markdown
### 필요한 State 설계

| 필드 | 역할 |
|---|---|
| `topic` | 작성 주제 |
| `draft` | 현재 글 |
| `feedback` | 검토 의견 |
| `result` | `pass` 또는 `fail` |
| `attempt` | 작성 횟수 |
```

```python
class WritingState(TypedDict):
    topic: str
    draft: NotRequired[str]
    feedback: NotRequired[str]
    result: NotRequired[Literal["pass", "fail"]]
    attempt: NotRequired[int]
```

```markdown
원본의 `sentense` 필드는 의미가 명확하도록 `draft`로 바꿨다.

### 검토 결과는 구조화 출력으로 받기

검토에는 판정과 이유가 모두 필요하다.
```

```python
class ReviewResult(BaseModel):
    result: Literal["pass", "fail"]
    feedback: str
```

```python
reviewer = llm.with_structured_output(ReviewResult)
```

```markdown
이렇게 구성하면 검토 결과를 다음처럼 접근할 수 있다.
```

```python
response.result
response.feedback
```

```markdown
자유 형식 문자열을 잘라서 판정을 추출하는 것보다 응답 구조를 명확하게 다룰 수 있다. 다만 스키마 검증이 검토 내용의 정확성까지 보장하지는 않는다. [공식 Structured output 문서](https://docs.langchain.com/oss/python/langchain/structured-output)

### 완성 코드

작성과 검토 모델을 각각 전달받도록 클래스로 구성했다.
```

```python
class WritingWorkflow:
    def __init__(self, writer, reviewer, max_attempts=3):
        if max_attempts < 1:
            raise ValueError("max_attempts는 1 이상이어야 합니다.")

        self.writer = writer
        self.reviewer = reviewer
        self.max_attempts = max_attempts

    def write(self, state: WritingState):
        attempt = state.get("attempt", 0) + 1
        topic = state["topic"]

        # 검토 후 재작성 흐름을 관찰하기 위한 짧은 첫 초안
        if attempt == 1:
            return {
                "draft": f"{topic}은 아주 좋은 것이야!",
                "attempt": attempt,
            }

        response = self.writer.invoke(
            f"주제: {topic}\n"
            f"이전 글: {state.get('draft', '')}\n"
            f"피드백: {state.get('feedback', '')}\n"
            "피드백을 반영해서 구체적인 설명이 담긴 짧은 글을 작성해줘."
        )

        return {
            "draft": response.text,
            "attempt": attempt,
        }

    def review(self, state: WritingState):
        response = self.reviewer.invoke(
            f"주제: {state['topic']}\n"
            f"글: {state['draft']}\n"
            "주제를 구체적으로 설명하면 pass, 설명이 부족하면 fail로 판정해. "
            "feedback에는 이유와 개선할 점을 작성해줘."
        )

        return {
            "result": response.result,
            "feedback": response.feedback,
        }

    def route(self, state: WritingState):
        if state["result"] == "pass":
            return "success"

        if state["attempt"] >= self.max_attempts:
            return "exhausted"

        return "retry"

    def build(self):
        builder = StateGraph(WritingState)

        builder.add_node("write", self.write)
        builder.add_node("review", self.review)

        builder.add_edge(START, "write")
        builder.add_edge("write", "review")

        builder.add_conditional_edges(
            "review",
            self.route,
            {
                "success": END,
                "exhausted": END,
                "retry": "write",
            },
        )

        return builder.compile()
```

```markdown
실행:
```

```python
reviewer = llm.with_structured_output(ReviewResult)

writing_graph = WritingWorkflow(
    writer=llm,
    reviewer=reviewer,
    max_attempts=3,
).build()

for event in writing_graph.stream(
    {"topic": "AI Agent와 LangGraph"},
    stream_mode="updates",
):
    print(event)
```

```markdown
### 상태가 어떻게 전달될까?

아래는 첫 검토에서 실패하고 두 번째 검토에서 통과한 경우의 예시다.

| 단계 | 반환하는 변경분 | 다음 동작 |
|---|---|---|
| 첫 작성 | 짧은 초안, `attempt=1` | 검토 |
| 첫 검토 | `result="fail"`, 개선 피드백 | 재작성 |
| 두 번째 작성 | 개선된 초안, `attempt=2` | 검토 |
| 두 번째 검토 | `result="pass"`, 검토 의견 | 종료 |

두 번째 작성 시점에는 다음 값이 State에 남아 있다.
```

```json
{
    "topic": "AI Agent와 LangGraph",
    "draft": "AI Agent와 LangGraph은 아주 좋은 것이야!",
    "feedback": "두 개념의 의미와 관계를 구체적으로 설명해줘.",
    "result": "fail",
    "attempt": 1,
}
```

```markdown
`write()`는 이전 글과 피드백을 읽어 개선된 글을 만든다. 반환한 새 `draft`는 이전 글을 덮어쓴다.

### 종료와 성공은 다르다

원본에는 최대 횟수에 도달하면 종료 경로로 `"pass"`를 반환하는 풀이가 있다.

하지만 횟수 소진으로 끝난 글이 검토를 통과한 것은 아니다.

그래서 예제에서는 경로를 나눴다.
```

```python
"success"    # 검토 통과
"exhausted"  # 횟수 소진
"retry"      # 다시 작성
```

```markdown
두 종료 경로 모두 `END`로 연결되지만, 품질 판정인 `result`는 그대로 유지된다.
```

```markdown
## 12. Fake 모델로 API 없이 반복 흐름 확인하기

원본에서는 작성 모델을 `FakeListChatModel`로 바꾼다.

그런데 검토 함수는 전역 변수인 `llm_with_review_result`를 사용하므로 **작성 모델만 가짜로 바꿔도 검토 단계는 실제 API를 호출한다.**

그래프 전체를 API 없이 확인하려면 작성과 검토를 모두 바꿔야 한다.

위에서 만든 클래스는 두 모델을 각각 전달받으므로 쉽게 교체할 수 있다.
```

```python
from langchain_core.language_models.fake_chat_models import FakeListChatModel


class FakeReviewer:
    def __init__(self):
        self.calls = 0

    def invoke(self, prompt):
        self.calls += 1

        if self.calls == 1:
            return ReviewResult(
                result="fail",
                feedback="AI Agent와 LangGraph의 개념과 관계를 설명해줘.",
            )

        return ReviewResult(
            result="pass",
            feedback="두 개념과 관계가 설명되어 있음.",
        )


fake_writer = FakeListChatModel(
    responses=[
        "AI Agent는 목표 수행을 위해 도구를 선택하고 행동한다. "
        "LangGraph는 상태, 분기, 반복으로 이러한 실행 흐름을 구성한다."
    ]
)

fake_graph = WritingWorkflow(
    writer=fake_writer,
    reviewer=FakeReviewer(),
).build()

final_state = fake_graph.invoke({
    "topic": "AI Agent와 LangGraph"
})

assert final_state["attempt"] == 2
assert final_state["result"] == "pass"

print(final_state)
```

```markdown
이 예제는 필요한 import와 클래스 정의만 실행하면 되며, 실제 `llm` 생성은 필요 없다.

실행 순서는 고정된다.
```

```
write
→ review: fail
→ write
→ review: pass
→ END
```

```markdown
첫 작성은 고정 문자열을 반환하므로, 가짜 작성 모델은 두 번째 작성에서 처음 호출된다.

이 방법으로 확인하는 것은 **연결과 상태 전달, 반복 종료**다. 실제 LLM의 답변 품질을 검증하는 것은 아니다.
```

```markdown
## 13. Node는 언제 나누는 게 좋을까?

| 기준 | 예시 |
|---|---|
| 역할이 다른가? | 작성과 검토 |
| 중간 상태를 다음 작업이 사용하는가? | 언어 감지 결과로 답변 생성 |
| 분기하거나 재시도해야 하는가? | 검토 실패 후 작성으로 복귀 |
| 단계별 결과를 관찰해야 하는가? | 숫자 추측과 범위 변경 |

반대로 `.strip().lower()`처럼 작고 항상 함께 실행되는 처리를 모두 노드로 나눌 필요는 없다.

### 그래프 구조 확인하기
```

```python
print(writing_graph.get_graph().draw_mermaid())
```

```markdown
노트북에서 이미지로 확인하려면 다음 방법을 사용할 수 있다.
```

```python
from IPython.display import Image, display

display(
    Image(writing_graph.get_graph().draw_mermaid_png())
)
```

```markdown
이미지 렌더링 방식에 따라 네트워크나 추가 실행 환경이 필요할 수 있다.
```

```markdown
## 14. 이번 학습에서 기억할 점

- **State는 데이터**, **Node는 작업**, **Edge는 다음 실행 위치**를 담당한다.
- 노드는 전체 상태 대신 변경할 필드만 반환한다.
- 조건부 Edge는 갱신된 State를 보고 다음 경로를 선택한다.
- 이전 노드로 돌아가는 연결과 종료 조건을 조합하면 반복을 만들 수 있다.
- 최신 결과는 덮어쓰고, 메시지와 로그는 Reducer로 누적한다.
- `updates`는 변경분, `values`는 전체 상태를 확인할 때 사용한다.
- 반복 작업에는 성공 조건과 최대 시도 횟수가 함께 필요하다.
- 작성·검토 흐름에서는 이전 글과 피드백을 State로 전달해야 개선이 이어진다.
- 가짜 모델 테스트에서는 실제 모델을 호출하는 모든 의존성을 교체해야 한다.

이번 실습에서 가장 중요한 부분은 **각 함수가 무엇을 하는지뿐 아니라, 어떤 값을 State에 남기고 그 값이 다음 작업을 어떻게 바꾸는지 읽는 것**이었다.
```

## 헷갈린 점

- 처음에는 LangChain의 Chain과 LangGraph가 어떤 점에서 다른지, State에 저장된 값이 각 Node를 거치면서 어떻게 변경되고 다음 단계로 전달되는지 헷갈렸다.
- 특히 글 검토에 실패했을 때 피드백을 State에 저장하고, 조건부 Edge를 통해 다시 작성 Node로 돌아가 글을 개선하는 반복 흐름을 이해하는 데 어려움이 있었다.
- 그래서 State·Node·Edge의 역할부터 조건 분기, 반복, Reducer까지 여러 예제를 직접 연결하며 전체 실행 흐름을 정리했다. 또한 단순히 State를 사용하는 것만으로 중간 실행 지점이 영구 저장되는 것은 아니며, 중간 단계부터 실행을 이어가려면 체크포인터가 필요하다는 점도 구분해서 이해했다.
