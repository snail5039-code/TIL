# [TIL] 사람의 개입, 과거 State 재실행, 검색 결과 평가와 병렬 검색

오늘은 LangGraph에서 사람이 중간에 개입하는 **Human-in-the-Loop**, Checkpoint를 이용해 과거 시점부터 다시 실행하는 **Time Travel**, 문서를 검색하고 근거를 평가하는 **Agentic RAG**를 학습했다.

세 기능은 서로 달라 보이지만 모두 다음 구조를 사용한다.

```plain text
현재 State 확인
→ Node에서 작업 수행
→ 결과를 State에 저장
→ Edge 또는 Router가 다음 위치 결정
```

```plain text
Human-in-the-Loop
실행 → interrupt로 중단 → 사용자 입력 → Command(resume=...)로 재개

Time Travel
실행 → checkpoint 저장 → 과거 checkpoint 선택 → Replay 또는 Fork

Agentic RAG
질문 → 검색 → 근거 평가
              ├─ 충분함 → 답변
              └─ 부족함 → 질문 수정·재검색 또는 답변 보류
```

아래 내용은 `21-human-in-the-loop_answer.ipynb`, `22-time-travel.ipynb`, `23-agentic-rag.ipynb`와 회사 규정 문서 실습 코드를 바탕으로 정리했다.

## 1. Human-in-the-Loop란?

Human-in-the-Loop는 AI가 모든 작업을 자동으로 끝내지 않고, 중요한 시점에 실행을 멈춰 사람의 판단이나 추가 정보를 받는 방식이다.

다음과 같은 작업에 사용할 수 있다.

- 결제, 삭제, 게시글 발행 전 승인
- AI가 만든 계획이나 문서 검토
- 요청 처리에 부족한 정보 질문
- 사용자의 수정 의견을 받은 뒤 다시 실행

LangGraph에서는 `interrupt()`로 실행을 멈추고 `Command(resume=...)`로 다시 시작한다.

```plain text
Graph 실행
→ interrupt(payload)
→ 실행 중단 및 checkpoint 저장
→ 사용자가 결정
→ Command(resume=decision)
→ 중단된 위치부터 실행 재개
```

## 2. interrupt에 전달하는 값

```python
decision = interrupt({
    "question": "이 결제를 실행하시겠습니까?",
    "tool": "send_payment",
    "args": {
        "amount": amount,
        "recipient": recipient,
    },
})
```

`interrupt()` 안의 딕셔너리는 사용자 화면이나 Client에 전달할 정보다.

| 값 | 역할 |
|---|---|
| `question` | 사용자에게 보여줄 질문 |
| `tool` | 실행하려던 Tool 이름 |
| `args` | Tool에 전달된 값 |
| `decision` | 재개 후 사용자가 보낸 결정 |

첫 번째 `invoke()`의 결과에서는 다음처럼 확인한다.

```python
result = graph.invoke(input_state, config)

request = result["__interrupt__"][0].value

print(request["question"])
print(request["tool"])
print(request["args"])
```

## 3. Command(resume=...)로 재개하기

사용자의 결정을 같은 `thread_id`로 전달하면 저장된 실행을 이어서 처리한다.

```python
result = graph.invoke(
    Command(resume={"action": "approve"}),
    config,
)
```

거절 사유도 함께 전달할 수 있다.

```python
result = graph.invoke(
    Command(
        resume={
            "action": "reject",
            "reason": "금액을 다시 확인해야 합니다.",
        }
    ),
    config,
)
```

`resume` 값은 `interrupt()`의 반환값이 된다.

```python
decision = interrupt({...})

if decision["action"] == "reject":
    return "작업이 취소되었습니다."

return "작업이 완료되었습니다."
```

## 4. 왜 resume 값을 딕셔너리로 보내는가?

문자열 하나만 보내도 구현할 수 있다.

```python
Command(resume="승인")
```

하지만 딕셔너리로 보내면 각 값의 의미가 명확하고 나중에 필드를 추가하기 쉽다.

```python
Command(
    resume={
        "action": "reject",
        "reason": "내용 수정 필요",
        "reviewer": "user",
    }
)
```

따라서 보통 다음처럼 사용한다.

```python
answer = interrupt({...})

return {
    "answer": answer["answer"]
}
```

`answer["answer"]`를 사용하는 이유는 재개할 때 다음 형태로 보냈기 때문이다.

```python
Command(resume={"answer": user_answer})
```

반대로 값만 보냈다면 바로 사용할 수 있다.

```python
Command(resume=user_answer)

answer = interrupt({...})

return {"answer": answer}
```

## 5. 추가 질문을 받는 HITL 그래프

오늘 확인한 그래프는 세 개의 노드로 구성된다.

```plain text
START
→ make_questions
→ ask_questions
→ summarize
→ END
```

State는 그래프 전체에서 공유할 값을 정의한다.

```python
class QuestionState(TypedDict):
    request: str
    message: str
    questions: list
    answer: list
    result: str
```

| 필드 | 저장하는 값 |
|---|---|
| `request` | 사용자의 원래 요청 |
| `message` | 질문 전에 보여줄 안내 문구 |
| `questions` | LLM이 만든 질문과 선택지 |
| `answer` | 사용자가 입력한 답변 |
| `result` | 답변까지 반영한 최종 결과 |

## 6. State와 BaseModel의 차이

```python
class Question(BaseModel):
    question: str = Field(description="질문")
    options: list[str] = Field(description="질문에 대한 선택지")


class Questions(BaseModel):
    message: str = Field(description="질문 안내 메시지")
    questions: list[Question]
```

`QuestionState`와 `Questions`는 역할이 다르다.

| 구분 | 역할 |
|---|---|
| `QuestionState` | 그래프 노드들이 공유하고 저장할 데이터 |
| `Questions` | LLM이 반환해야 할 응답 형식 |
| `Field(description=...)` | 각 필드에 어떤 내용을 생성해야 하는지 LLM에 설명 |

`message`를 구조화 출력에 포함하면 안내 문구도 LLM이 요청에 맞춰 생성한다.

안내 문구가 항상 같아도 된다면 굳이 LLM이 만들 필요는 없다.

```python
return {
    "message": "몇 가지 정보를 먼저 확인할게요.",
    "questions": [...]
}
```

## 7. model_dump를 사용하는 이유

```python
return {
    "message": response.message,
    "questions": [
        question.model_dump()
        for question in response.questions
    ],
}
```

`response.questions` 안의 값은 일반 딕셔너리가 아니라 `Question`이라는 Pydantic 객체다.

```python
Question(
    question="어떤 여행을 좋아하나요?",
    options=["휴식", "관광", "맛집"]
)
```

`model_dump()`를 사용하면 일반 딕셔너리로 바뀐다.

```python
{
    "question": "어떤 여행을 좋아하나요?",
    "options": ["휴식", "관광", "맛집"],
}
```

딕셔너리로 바꾸면 다음 작업이 편해진다.

- `interrupt()` payload로 전달
- JSON 응답으로 전송
- Checkpointer에 저장
- `question["question"]`처럼 접근

## 8. 실행 계획 검토 그래프

Planner가 계획을 만들고, 사용자가 승인하거나 수정 의견을 줄 수 있다.

```plain text
Planner
→ Approval에서 interrupt
   ├─ approve → Executor
   └─ edit → 피드백을 포함해 Planner 재실행
```

```python
def human_approval(state: SimplePlanState):
    decision = interrupt({
        "plan": state["plan"],
        "question": "이 계획으로 실행할까요?",
    })

    return {
        "action": decision["action"],
        "feedback": decision.get("feedback", ""),
    }
```

라우터는 사용자의 결정을 보고 다음 노드를 고른다.

```python
def route_human_approval(state: SimplePlanState):
    if state["action"] == "edit":
        return "edit"

    return "approve"
```

```python
builder.add_conditional_edges(
    "approval",
    route_human_approval,
    {
        "edit": "planner",
        "approve": "executor",
    },
)
```

`interrupt()`가 사용자의 결정을 받고, 라우터가 그 결정에 따라 이동한다.

# Time Travel

## 9. Time Travel이란?

Checkpointer를 연결하면 그래프가 실행되는 동안 State가 checkpoint로 저장된다.

Time Travel은 저장된 checkpoint를 이용해 과거 State를 확인하거나, 그 시점부터 다시 실행하는 기능이다.

| 기능 | 의미 |
|---|---|
| 조회 | 과거 State와 다음 실행 노드 확인 |
| Replay | 과거 State를 수정하지 않고 다시 실행 |
| Fork | 과거 State를 수정해 새로운 실행 경로 생성 |

실습 그래프는 다음과 같다.

```plain text
START
→ choose_topic
→ write_draft
→ review_draft
→ END
```

## 10. thread_id와 checkpoint_id

```python
config = {
    "configurable": {
        "thread_id": "post-1"
    }
}
```

`thread_id`는 하나의 실행 이력 전체를 구분한다.

그 실행 안에서 만들어지는 각각의 저장 시점은 `checkpoint_id`로 구분한다.

```plain text
thread_id: post-1

checkpoint A → 주제 정리 전
checkpoint B → 초안 작성 전
checkpoint C → 검토 전
checkpoint D → 실행 완료
```

| 값 | 의미 |
|---|---|
| `thread_id` | 하나의 세션 또는 실행 이력 |
| `checkpoint_id` | 그 실행 이력 안의 특정 시점 |

## 11. StateSnapshot 확인하기

```python
current = graph.get_state(config)

print("values:", current.values)
print("next:", current.next)
print("config:", current.config)
```

| 속성 | 의미 |
|---|---|
| `values` | 해당 시점까지 저장된 전체 State |
| `next` | 다음으로 실행할 노드 |
| `config` | `thread_id`, `checkpoint_id` 등 위치 정보 |

그래프 실행이 모두 끝났다면 `next`는 빈 튜플이다.

```python
next == ()
```

## 12. get_state_history로 과거 기록 가져오기

```python
history = list(
    graph.get_state_history(config)
)
```

`history`에는 해당 `thread_id`로 실행된 checkpoint들이 들어 있다.

```python
for snapshot in history:
    print(snapshot.values)
    print(snapshot.next)
    print(snapshot.config)
```

`get_state_history()`가 있어야 여러 checkpoint를 반복하면서 원하는 시점을 찾을 수 있다.

또한 그래프를 먼저 실행해야 기록이 생긴다.

```python
graph.invoke({"request": "LangGraph"}, config)

history = list(
    graph.get_state_history(config)
)
```

## 13. 특정 노드 실행 직전의 checkpoint 찾기

```python
before_review = None

for snapshot in history:
    if snapshot.next == ("review_draft",):
        before_review = snapshot
        break
```

이 코드는 `review_draft`가 이미 실행된 시점을 찾는 코드가 아니다.

`next`가 `review_draft`라는 것은 **다음에 `review_draft`를 실행할 차례**라는 뜻이다.

```plain text
write_draft 실행 완료
→ draft가 State에 저장됨
→ next = ("review_draft",)
→ review_draft 실행 직전
```

`before_review`에는 ID 하나만 들어가는 것이 아니라 전체 `StateSnapshot`이 저장된다.

```python
before_review.values
before_review.next
before_review.config
```

## 14. Replay

Replay는 과거 State를 수정하지 않고 그 시점부터 다시 실행한다.

```python
replayed = graph.invoke(
    None,
    before_draft.config,
)
```

`None`을 넣는 이유는 새로운 입력을 넣지 않고 checkpoint에 저장된 State를 그대로 사용하기 때문이다.

```plain text
저장된 State 복원
→ snapshot.next에 있는 노드부터 실행
→ 이후 결과를 새 checkpoint로 저장
```

Replay도 기존 이력을 덮어쓰지 않는다. 같은 값으로 실행하더라도 새로운 checkpoint가 추가된다.

## 15. Fork와 update_state

Fork는 과거 State 일부를 수정한 뒤 새로운 실행 경로를 만든다.

```python
fork_config = graph.update_state(
    before_review.config,
    {
        "draft": "LangGraph Time Travel 심화"
    },
    as_node="write_draft",
)
```

각 인자의 의미는 다음과 같다.

| 인자 | 의미 |
|---|---|
| `before_review.config` | 수정할 기준 checkpoint |
| `{"draft": ...}` | 변경할 State 필드 |
| `as_node="write_draft"` | 이 변경을 `write_draft`가 반환한 결과처럼 처리 |

`as_node="write_draft"`는 `write_draft`부터 다시 실행한다는 뜻이 아니다.

`write_draft`가 방금 작업을 끝냈다고 처리하므로 다음 노드인 `review_draft`부터 실행한다.

```plain text
write_draft가 수정된 draft를 반환한 것으로 처리
→ next = review_draft
```

## 16. Fork한 State 실행하기

```python
fork_state = graph.get_state(fork_config)

print(fork_state.values)
print(fork_state.next)
```

이 코드는 수정된 값을 확인하기 위한 코드이므로 없어도 실행에는 문제가 없다.

실제 실행은 다음 코드에서 발생한다.

```python
forked_result = graph.invoke(
    None,
    fork_config,
)
```

State는 `update_state()`에서 이미 저장되었기 때문에 새로운 입력 없이 `None`을 전달한다.

```plain text
기존 draft
LangGraph 입문: 쉽게 설명하는 게시글입니다.

update_state 후 draft
LangGraph Time Travel 심화

review_draft 실행 후 review
검토 완료: LangGraph Time Travel 심화
```

기존 문장이 사라진 이유는 `draft` 필드를 새로운 문자열로 덮어썼기 때문이다.

## 17. Replay와 Fork 비교

| 구분 | Replay | Fork |
|---|---|---|
| State 수정 | 없음 | 있음 |
| 실행 시작점 | 선택한 checkpoint의 `next` | 수정 후 생성된 checkpoint의 `next` |
| 사용 예 | 같은 조건으로 다시 생성 | 중간 결과를 수정해 새 결과 생성 |
| 기존 이력 | 유지 | 유지 |

실제 서비스에서는 다음 기능으로 사용할 수 있다.

```plain text
다시 생성
→ Replay

이 단계부터 다시 실행
→ Replay

내용 수정 후 계속
→ Fork

이 버전에서 새 문서 만들기
→ Fork
```

State만 과거로 돌아갈 수 있다는 점도 주의해야 한다. 이미 전송한 이메일이나 완료한 결제 같은 외부 작업까지 취소되는 것은 아니다.

# Agentic RAG

## 18. Agentic RAG란?

일반적인 RAG는 정해진 순서대로 검색하고 답변한다.

```plain text
질문 → 검색 → 답변
```

Agentic RAG는 질문과 검색 결과를 평가하여 다음 행동을 결정한다.

```plain text
질문
→ 검색 필요 여부 판단
→ 문서 검색
→ 검색 결과 평가
   ├─ 근거 충분 → 답변 생성
   ├─ 근거 부족 → 질문 재작성 후 재검색
   └─ 반복해도 부족 → 답변 보류
```

오늘 확인한 기본 그래프의 주요 구성 요소는 다음과 같다.

| 함수 | 역할 |
|---|---|
| `generate_query_or_respond` | 검색 Tool 호출 또는 직접 답변 선택 |
| `retrieve_documents` | 관련 문서 검색 |
| `grade_documents` | 검색 결과에 근거가 있는지 평가 |
| `rewrite_question` | 검색에 맞게 질문 수정 |
| `generate_answer` | 검색 문서만 사용해 답변 |
| `abstain` | 충분한 근거가 없을 때 답변 보류 |

## 19. 문서 로드와 Chunk 생성

PDF 예제에서는 다음 Loader를 사용했다.

```python
loader = PyPDFLoader("data/document.pdf")
docs = loader.load()
```

회사 규정 실습에서는 폴더 안의 Markdown 파일을 읽었다.

```python
from langchain_community.document_loaders import (
    DirectoryLoader,
    TextLoader,
)

loader = DirectoryLoader(
    "data/company_rules",
    glob="*.md",
    loader_cls=TextLoader,
    loader_kwargs={"encoding": "utf-8"},
)

docs = loader.load()
```

불러온 문서는 작은 조각으로 나눈다.

```python
chunks = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
).split_documents(docs)
```

각 Chunk에 ID도 추가했다.

```python
for chunk_id, chunk in enumerate(chunks):
    chunk.metadata["chunk_id"] = str(chunk_id)
```

`chunk_id`는 같은 문서 조각이 여러 검색어에서 반복 검색되었는지 확인할 때 사용할 수 있다.

```plain text
검색어 A → chunk 1, chunk 3
검색어 B → chunk 1, chunk 5

chunk_id로 중복 제거
→ chunk 1, chunk 3, chunk 5
```

## 20. Hybrid Retriever

오늘 실습에서는 두 검색 결과를 결합했다.

```plain text
Vector Search
문장의 전체 의미가 비슷한 문서 검색

BM25
질문과 같은 단어가 포함된 문서 검색
```

한국어 BM25 검색을 위해 Kiwi로 형태소를 나눴다.

```python
def kiwi_tokenize(text: str) -> list[str]:
    return [
        token.form.lower()
        for token in kiwi.tokenize(text)
        if token.form
    ]
```

두 검색기를 결합한다.

```python
hybrid_retriever = EnsembleRetriever(
    retrievers=[
        vector_retriever,
        bm25_retriever,
    ],
    weights=[0.5, 0.5],
    id_key="chunk_id",
)
```

이 구조는 의미가 비슷한 문서와 정확한 키워드가 포함된 문서를 함께 찾기 위해 사용한다.

## 21. 기본 검색 Tool

```python
@tool
def retrieve_company_rules(query: str) -> str:
    """회사 규정 문서에서 질문에 필요한 근거를 검색한다."""

    found = hybrid_retriever.invoke(query)[:HYBRID_TOP_K]

    if not found:
        return "검색된 문서가 없습니다."

    return "\n\n".join(
        f"[{i + 1}] "
        f"(출처 {doc.metadata.get('source', '?')}) "
        f"{doc.page_content}"
        for i, doc in enumerate(found)
    )
```

`source`는 어떤 파일에서 가져온 내용인지 표시하고, `page_content`는 실제 문서 내용이다.

```plain text
(출처 data/company_rules/재택근무규정.md)
재택근무는 주 2회까지 가능하다.
```

## 22. original_question이 필요한 이유

처음 실행할 때는 `messages`와 `original_question`의 내용이 같다.

```python
question = "재택근무는 주 몇 회까지 가능하고 예외 조건은 뭐야?"

graph.invoke({
    "messages": [
        HumanMessage(content=question)
    ],
    "original_question": question,
    "retry_count": 0,
})
```

하지만 재검색하면 `messages`의 마지막 질문은 바뀔 수 있다.

```plain text
original_question
재택근무는 주 몇 회까지 가능하고 예외 조건은 뭐야?

재작성된 messages의 마지막 질문
재택근무 주당 허용 횟수와 특별 승인 대상
```

`original_question`은 마지막 답변과 근거 평가에서 사용자의 원래 의도를 잃지 않기 위해 보관한다.

`termination_reason`은 노드가 실행하면서 만드는 결과이므로 최초 입력에는 넣지 않아도 된다.

## 23. Multi-query와 Send Worker

Multi-query는 하나의 질문에서 여러 검색어를 만든다.

```python
class MultiQueryResult(BaseModel):
    queries: list[str] = Field(
        description="검색에 사용할 서로 다른 검색어 목록"
    )
```

```python
def make_queries(state: State):
    prompt = ChatPromptTemplate.from_messages([
        (
            "system",
            "사용자 질문을 회사 규정 문서 검색에 "
            "적합한 검색어 3개로 바꾸세요.",
        ),
        ("human", "질문: {question}"),
    ])

    result = (
        prompt
        | llm.with_structured_output(MultiQueryResult)
    ).invoke({
        "question": state["original_question"]
    })

    return {
        "queries": result.queries
    }
```

생성된 검색어는 State의 `queries`에 저장된다.

```plain text
[
  "재택근무 주당 허용 횟수",
  "재택근무 예외 적용 대상",
  "재택근무 신청 및 승인 조건"
]
```

각 검색어를 `Send`로 Worker에 전달한다.

```python
def route_queries_workers(state: State):
    return [
        Send(
            "search_worker",
            {"query": query},
        )
        for query in state["queries"]
    ]
```

```plain text
query 1 ─→ search_worker
query 2 ─→ search_worker
query 3 ─→ search_worker
```

라우터는 별도 노드가 아니라 조건부 Edge를 만드는 함수이므로 그래프 그림에서 사각형 노드로 표시되지 않는다.

## 24. Worker 전용 State와 검색 결과 누적

Worker는 전체 State가 필요하지 않고 검색어 하나만 받으면 된다.

```python
class SearchWorkerState(TypedDict):
    query: str
```

```python
def search_worker(state: SearchWorkerState):
    docs = hybrid_retriever.invoke(
        state["query"]
    )[:HYBRID_TOP_K]

    return {
        "result_workers": [
            f"(출처 {doc.metadata.get('source', '?')}) "
            f"{doc.page_content}"
            for doc in docs
        ]
    }
```

여러 Worker 결과를 하나의 목록으로 누적하려면 Reducer가 필요하다.

```python
class State(TypedDict, total=False):
    original_question: str
    queries: list[str]
    result_workers: Annotated[
        list[str],
        operator.add,
    ]
```

```plain text
Worker A 결과: [문서 1, 문서 2]
Worker B 결과: [문서 3, 문서 4]

operator.add 적용
→ [문서 1, 문서 2, 문서 3, 문서 4]
```

`operator.add`는 병렬 Worker들이 반환한 목록을 합치는 Reducer다.

## 25. Multi-query, 질문 분해, HyDE의 차이

세 방법은 모두 여러 검색 입력을 만들 수 있지만 목적이 다르다.

| 방법 | 생성하는 것 |
|---|---|
| Multi-query | 같은 질문을 다른 검색 표현으로 변환 |
| 질문 분해 | 복합 질문을 여러 하위 질문으로 분리 |
| HyDE | 답이 들어 있을 것 같은 가상 문서 작성 |

```plain text
원래 질문
재택근무는 주 몇 회 가능하고 예외 조건은 뭐야?
```

Multi-query:

```plain text
재택근무 주당 허용 횟수
재택근무 예외 적용 대상
재택근무 승인 규정
```

질문 분해:

```plain text
재택근무는 주 몇 회 가능한가?
재택근무 횟수 제한의 예외는 무엇인가?
예외 신청에는 누구의 승인이 필요한가?
```

HyDE:

```plain text
회사의 재택근무 규정에 따르면 기본 재택근무는
주당 일정 횟수까지 허용되며, 육아·건강·긴급 상황에는
별도 승인으로 예외가 적용될 수 있다.
```

현재 회사 규정 실습의 `make_queries` 프롬프트는 가상 문서 단락을 만들도록 작성되어 있으므로 Multi-query보다 **HyDE를 세 개 생성하는 형태**에 가깝다.

## 26. 검색 결과 평가와 답변 생성

검색 결과는 문자열로 합쳐 평가한다.

```python
context = "\n\n".join(
    state["result_workers"]
)
```

```python
class GradeDocumentsResult(BaseModel):
    relevant: bool = Field(
        description="검색 결과에 질문의 근거가 있는지"
    )
```

```python
if result.relevant:
    return "generate_answer"

if state["retry_count"] >= 3:
    return "abstain"

return "rewrite_question"
```

답변 생성에서도 같은 검색 결과를 사용한다.

```python
def generate_answer(state: State):
    context = "\n\n".join(
        state["result_workers"]
    )

    response = (prompt | llm).invoke({
        "context": context,
        "question": state["original_question"],
    })

    return {
        "messages": [response],
        "termination_reason": "completed",
    }
```

검색 문서 평가는 답변 생성 전 문서의 관련성을 확인한다.

생성된 답변의 문장마다 근거가 있는지 검사하려면 `generate_answer` 다음에 별도의 답변 검증 노드를 추가해야 한다.

## 27. 현재 회사 규정 실습에서 보완할 부분

현재 수정된 실습 그래프의 큰 흐름은 다음과 같다.

```plain text
START
→ make_queries
→ Send(search_worker × 3)
→ grade_documents
   ├─ generate_answer → END
   ├─ rewrite_question → make_queries
   └─ abstain → END
```

다만 재검색 경로에는 두 가지를 더 연결해야 한다.

첫째, 현재 `retry_count`를 증가시키는 노드가 없다.

```python
return {
    "queries": result.queries,
    "retry_count": state.get("retry_count", 0) + 1,
}
```

둘째, `rewrite_question`이 새 질문을 `messages`에 저장하지만 `make_queries`는 계속 `original_question`만 읽는다.

```python
# 현재 형태
question = state["original_question"]

# 재작성 결과를 사용하려면
question = (
    state["messages"][-1].content
    if state.get("messages")
    else state["original_question"]
)
```

이 연결이 없으면 재검색하더라도 같은 원래 질문으로 비슷한 검색어를 다시 만들 수 있다.

병렬 검색 결과에서 같은 Chunk가 반복될 수 있으므로 최종 Context를 만들기 전에 `chunk_id` 기준 중복 제거도 추가할 수 있다.

## 28. 원래 코드와 실습 코드 중 무엇을 사용할까?

두 코드의 목적이 다르므로 상황에 맞게 선택한다.

| 상황 | 적합한 구조 |
|---|---|
| 기본 Agentic RAG 흐름 학습 | ToolNode 기반 원래 코드 |
| LLM이 검색 여부를 결정 | `bind_tools()` + `ToolNode` |
| 항상 문서를 검색 | `START → make_queries` |
| 하나의 질문으로 한 번 검색 | 기본 Retriever Tool |
| 여러 표현으로 검색 범위 확대 | Multi-query |
| 복합 질문을 나눠 검색 | 질문 분해 |
| 여러 검색을 병렬 실행 | `Send` + Worker |
| 근거가 부족할 때 다시 검색 | 평가 + 재작성 Loop |
| 검색 비용과 구조를 단순하게 유지 | 기본 구조 |

처음에는 기본 RAG를 정상 동작하게 만든 다음 Multi-query, 병렬 Worker, HyDE, 재검색을 하나씩 추가하는 방식이 이해하고 디버깅하기 쉽다.

## 29. 외워야 할 코드

전체 코드를 통째로 외울 필요는 없다.

다음 다섯 가지 형태를 기억하면 된다.

### 1. 노드의 기본 형태

```python
def node(state: State):
    value = state["key"]

    return {
        "result": new_value
    }
```

### 2. interrupt와 resume

```python
decision = interrupt(payload)
```

```python
graph.invoke(
    Command(resume=decision),
    config,
)
```

### 3. checkpoint 찾기

```python
history = list(
    graph.get_state_history(config)
)

for snapshot in history:
    if snapshot.next == ("node_name",):
        selected = snapshot
        break
```

### 4. Replay와 Fork

```python
# Replay
graph.invoke(None, selected.config)
```

```python
# Fork
fork_config = graph.update_state(
    selected.config,
    {"field": "new value"},
    as_node="previous_node",
)

graph.invoke(None, fork_config)
```

### 5. Send Worker

```python
def route_workers(state: State):
    return [
        Send(
            "worker",
            {"query": query},
        )
        for query in state["queries"]
    ]
```

## 30. 이번 학습에서 기억할 점

- `interrupt()`는 실행을 멈추고 사용자에게 필요한 정보를 전달한다.
- `Command(resume=...)`는 같은 `thread_id`의 중단된 실행을 재개한다.
- Resume 값을 딕셔너리로 보내면 여러 정보를 명확하게 전달할 수 있다.
- State는 그래프가 공유할 데이터이고 BaseModel은 LLM 응답 형식이다.
- `model_dump()`는 Pydantic 객체를 일반 딕셔너리로 바꾼다.
- `snapshot.next`는 해당 시점에서 다음에 실행할 노드다.
- `before_review`에는 ID만이 아니라 전체 `StateSnapshot`이 저장된다.
- Replay는 State를 그대로 사용하고 Fork는 State를 수정한다.
- `as_node`는 해당 노드가 State를 작성한 것처럼 처리한다.
- 과거 checkpoint에서 실행할 때는 저장된 State가 있으므로 `invoke(None, config)`를 사용한다.
- Hybrid Search는 의미 검색과 키워드 검색을 결합한다.
- Multi-query는 같은 질문의 검색 표현을 늘린다.
- 질문 분해는 복합 질문을 여러 하위 질문으로 나눈다.
- HyDE는 검색용 가상 문서를 만든다.
- `Send`는 여러 입력을 Worker에 나누어 전달한다.
- `operator.add`는 Worker 결과 목록을 State에 누적한다.
- Router는 노드가 아니므로 그래프 그림에 별도 상자로 나타나지 않는다.
- 검색 문서 평가와 생성된 답변 검증은 서로 다른 단계다.
- 코드를 외우기보다 **입력 → Node 반환값 → State 변경 → 다음 경로**를 먼저 확인해야 한다.

## 31. 헷갈린 점

처음에는 `snapshot.next == ("review_draft",)`가 `review_draft`를 실행한 checkpoint를 찾는 코드로 보였다. 실제로는 `review_draft`를 **다음에 실행할 시점**, 즉 검토 직전의 checkpoint를 찾는 코드였다.

`as_node="write_draft"`도 `write_draft`부터 다시 시작한다는 의미로 이해하기 쉬웠다. 실제로는 State 변경을 `write_draft`의 결과로 처리하므로, 그래프는 그 다음 노드인 `review_draft`부터 실행한다.

`graph.invoke(None, fork_config)`에서 `None`을 전달하는 이유도 처음에는 입력이 없어 보였다. 하지만 `fork_config`가 수정된 State가 저장된 checkpoint를 가리키므로 새로운 요청을 다시 넣을 필요가 없었다.

Agentic RAG에서는 Tool, Node, Worker의 역할이 섞여 보였다. ToolNode 구조에서는 LLM이 Tool 호출을 선택하지만, `Send` 구조에서는 그래프가 검색어를 Worker에 직접 전달한다. 병렬 검색을 적용하면 검색 자체는 `search_worker`가 담당하고, Retriever Tool을 반드시 그래프 경로에 넣을 필요는 없다.

가장 중요한 점은 기능을 한 번에 모두 붙이는 것이 아니었다. 기본 검색과 답변이 동작하는지 확인한 뒤 Multi-query, Worker, 평가, 재검색을 하나씩 추가해야 어느 단계에서 문제가 생겼는지 확인할 수 있었다.
