# 26.09.21 45일차(Python [Fault Tolerance, Agent Tool Control, LangGraph Logging])

## [TIL] 장애 복구, Agent Tool 통제, LangGraph 실행 기록

오늘은 LangGraph에서 실패한 작업을 다시 시도하고 이어서 실행하는 **Fault Tolerance**, Agent가 Tool을 호출하기 전후를 검사하는 **Tool Control Middleware**, 실행 과정과 오류를 기록하는 **Logging**을 학습했다.

세 주제는 서로 달라 보이지만 실제 서비스에서 다음 순서로 연결된다.

```plain text
요청 실행
→ 일시적 오류는 재시도
→ 계속 실패하면 실패 상태로 전환
→ Tool 호출 전 권한·인자 검사
→ Tool 호출 후 결과 검사
→ 전체 실행 과정과 오류를 로그로 기록
```

```plain text
Fault Tolerance
실패 → RetryPolicy로 재시도 → Checkpoint에서 재개 → 최종 실패 처리

Tool Control
LLM의 Tool 호출 제안 → 실행 전 Middleware → 실제 Tool → 실행 후 Middleware

Logging
Node 시작 → 변수와 결과 기록 → 오류 발생 시 traceback 기록
```

아래 내용은 `27-fault-tolerance.ipynb`, `28-agent-tool-control.ipynb`, `29-langgraph-logging.ipynb`를 바탕으로 정리했다.

---

## 1. Fault Tolerance란?

Fault Tolerance는 실행 중 일부 작업이 실패해도 전체 서비스를 바로 포기하지 않고 재시도하거나 저장된 지점부터 이어서 처리하는 방식이다.

- 일시적인 네트워크 오류는 자동 재시도한다.
- 완료된 노드는 다시 실행하지 않고 실패한 노드부터 재개한다.
- 최대 재시도 횟수를 소진하면 오류 처리 함수로 이동한다.
- 외부 저장 작업은 중복 실행되지 않도록 멱등성을 적용한다.

```plain text
START
→ prepare
→ call_service
   ├─ 성공 → END
   └─ ConnectionError → 같은 노드 재시도
```

---

## 2. RetryPolicy로 자동 재시도하기

```python
retry_policy = RetryPolicy(
    initial_interval=1.0,
    backoff_factor=1.0,
    max_interval=1.0,
    max_attempts=3,
    jitter=True,
    retry_on=ConnectionError,
)
```

| 설정 | 의미 |
|---|---|
| `initial_interval=1.0` | 첫 재시도 전 1초 대기 |
| `backoff_factor=1.0` | 대기 시간을 늘리지 않고 같은 간격 유지 |
| `max_interval=1.0` | 최대 대기 시간은 1초 |
| `max_attempts=3` | 최초 실행을 포함해 최대 3번 시도 |
| `jitter=True` | 여러 요청이 동시에 재시도하지 않도록 대기 시간에 작은 변동 추가 |
| `retry_on=ConnectionError` | `ConnectionError`일 때만 재시도 |

정책은 재시도할 노드에 연결한다.

```python
builder.add_node(
    "call_service",
    call_unstable_service,
    retry_policy=retry_policy,
)
```

`prepare`는 한 번 실행되고 `call_service`만 세 번 실행된다.

```plain text
prepare 실행 1회
call_service 시도 1 → 실패
call_service 시도 2 → 실패
call_service 시도 3 → 성공
```

---

## 3. 재시도되는 오류와 재시도되지 않는 결과

```python
retry_on=ConnectionError
```

이 설정이면 `ConnectionError`는 재시도하지만 `ValueError` 같은 다른 예외는 바로 밖으로 전달된다.

또한 다음 코드는 예외가 아니라 정상적인 반환이다.

```python
return {"status": "failed"}
```

따라서 `RetryPolicy`는 실행되지 않는다. 재시도를 원하면 `raise ConnectionError(...)`처럼 정책에 등록한 예외를 발생시켜야 한다.

---

## 4. RetryPolicy, with_retry, with_fallbacks 차이

| 방식 | 적용 범위 | 역할 |
|---|---|---|
| `RetryPolicy` | LangGraph Node | 실패한 노드 자체를 재실행 |
| `with_retry()` | Runnable 또는 Chain | 모델·체인 실행을 다시 시도 |
| `with_fallbacks()` | Runnable 또는 Chain | 실패하면 다른 모델이나 체인 실행 |

LangGraph의 특정 노드 실패를 다룰 때는 `RetryPolicy`, 일반 LangChain 체인 호출을 다시 시도할 때는 `with_retry()`를 사용한다.

---

## 5. Checkpointer를 이용한 노드 재개

재시도는 같은 실행 안에서 즉시 다시 시도하는 기능이고, 재개는 실패 상태를 저장해 두었다가 나중에 이어서 실행하는 기능이다.

```python
graph = builder.compile(
    checkpointer=InMemorySaver()
)

config = {
    "configurable": {
        "thread_id": "fault-sequential-1"
    }
}
```

첫 실행에서 `call_service`가 실패했다면 상태를 확인한다.

```python
snapshot = graph.get_state(config)

print(snapshot.values)
print(snapshot.next)
```

```plain text
values → prepare가 만든 prepared 값이 저장됨
next   → ('call_service',)
```

같은 `thread_id`와 `None`을 넣어 재개한다.

```python
result = graph.invoke(None, config)
```

새 입력 대신 `None`을 넣는 이유는 Checkpointer에 기존 State가 저장되어 있기 때문이다.

---

## 6. Super-step과 병렬 노드 재개

Super-step은 함수 이름이 아니라 **같은 실행 차례에 처리되는 노드 묶음**을 설명하는 말이다.

```python
builder.add_edge(START, "prepare")
builder.add_edge("prepare", "stable_branch")
builder.add_edge("prepare", "unstable_branch")
builder.add_edge("stable_branch", "summarize")
builder.add_edge("unstable_branch", "summarize")
builder.add_edge("summarize", END)
```

```plain text
첫 번째 super-step: prepare
두 번째 super-step: stable_branch + unstable_branch
세 번째 super-step: summarize
```

호출 횟수 확인용 딕셔너리는 다음과 같다.

```python
call_count = {
    "prepare": 0,
    "stable": 0,
    "unstable": 0,
}
```

각 노드가 실행될 때 자기 값을 1씩 증가시킨다. 첫 실행에서 `stable_branch`는 성공하고 `unstable_branch`만 실패하면 Checkpointer는 성공한 결과를 저장한다. 이후 같은 thread를 재개하면 완료된 `stable_branch`는 다시 실행하지 않고 미완료 상태인 `unstable_branch`만 실행한다.

```plain text
첫 실행 후: prepare=1, stable=1, unstable=1
재개 후:   prepare=1, stable=1, unstable=2
```

`unstable=2`가 된 이유는 실패한 노드가 한 번 더 실행됐기 때문이다. `stable=1`이 유지된 이유는 성공 결과가 이미 Checkpoint에 저장됐기 때문이다.

---

## 7. 노드 실행 시간 제한

```python
builder.add_node(
    "slow_service",
    slow_service,
    timeout=TimeoutPolicy(run_timeout=2.0),
    retry_policy=RetryPolicy(
        max_attempts=2,
        retry_on=NodeTimeoutError,
    ),
)
```

노드가 2초 안에 끝나지 않으면 `NodeTimeoutError`가 발생하고, 해당 오류를 대상으로 최대 두 번 시도한다. 비동기 노드는 `await graph.ainvoke(...)`로 실행한다.

---

## 8. 재시도 소진 후 Error Handler

```python
def service_error_handler(
    state: RecoveryState,
    error: NodeError,
) -> Command:
    return Command(
        update={
            "status": "failed",
            "error_message": f"{error.node}: {error.error}",
        },
        goto="finalize",
    )
```

```python
builder.add_node(
    "call_service",
    failing_service,
    retry_policy=RetryPolicy(
        max_attempts=2,
        retry_on=ConnectionError,
    ),
    error_handler=service_error_handler,
)
```

흐름은 다음과 같다.

```plain text
call_service 실행
→ ConnectionError
→ RetryPolicy로 재시도
→ 최대 횟수까지 계속 실패
→ service_error_handler 실행
→ status와 error_message 저장
→ finalize로 이동
→ 그래프 정상 종료
```

그래프가 예외로 그대로 끝나는 대신 실패 상태를 State에 남기고 마무리하도록 만든 것이다.

---

## 9. 공통 노드 정책 설정

여러 노드에 같은 재시도 정책과 오류 처리기를 반복해서 넣어야 한다면 Builder의 기본 정책으로 지정할 수 있다.

```python
builder.set_node_defaults(
    retry_policy=common_retry_policy,
    error_handler=common_error_handler,
)
```

이 설정은 해당 Builder에 추가되는 노드들의 기본값이다. 프로그램 전체에 적용되는 완전한 전역 설정은 아니다. 특정 노드에서 별도의 정책을 넣으면 그 노드의 설정이 우선한다.

---

## 10. 멱등성과 중복 저장 방지

멱등성은 같은 요청을 여러 번 실행해도 외부 결과가 한 번만 반영되도록 만드는 성질이다.

```python
processed_request_ids = set()
external_rows = []

def idempotent_save(state):
    request_id = state["request_id"]

    if request_id not in processed_request_ids:
        external_rows.append({
            "request_id": request_id,
            "content": state["content"],
        })
        processed_request_ids.add(request_id)

    if not save_gate["allow_completion"]:
        raise ConnectionError(
            "저장 후 응답을 받지 못했습니다."
        )

    return {"saved": True}
```

첫 실행에서 외부 저장은 성공했지만 응답을 받지 못할 수 있다. 다시 실행할 때 같은 `request_id`가 Set에 있으므로 `external_rows.append(...)`를 건너뛴다. 따라서 같은 데이터가 두 번 저장되지 않는다.

---

# Agent Tool Control

## 11. Agent Tool 통제가 필요한 이유

LLM은 사용자의 자연어를 읽고 Tool 이름과 인자를 제안한다. 하지만 LLM이 제안했다고 바로 실행하면 안 된다.

```plain text
사용자 자연어
→ LLM이 Tool 이름과 인자 생성
→ 입력 형식 검사
→ 권한·한도·승인 검사
→ 실제 Tool 실행
→ Tool 결과 형식 검사
→ LLM이 최종 답변 생성
```

Middleware는 Tool 호출 앞뒤에서 공통 검사를 자동으로 실행하도록 연결하는 장치다.

---

## 12. Tool 입력 Schema

```python
class LookupAccountArgs(BaseModel):
    account_id: str = Field(
        min_length=3,
        max_length=40,
    )


class TransferArgs(BaseModel):
    account_id: str = Field(
        min_length=3,
        max_length=40,
    )
    amount: int = Field(gt=0, le=1000000)
    request_id: str = Field(
        min_length=8,
        max_length=80,
    )
```

이 클래스들은 Structured Output이 아니라 **Tool이 받을 입력 형식**이다.

```python
@tool(
    "lookup_account",
    args_schema=LookupAccountArgs,
)
def lookup_account_tool(account_id: str):
    return account_responses[account_id].copy()
```

`args_schema`를 Tool에 연결하면 LLM에게 필요한 인자 구조를 알려주고, 실제 호출 시에도 형식을 검사할 수 있다.

---

## 13. Tool 이름과 Schema 매핑

```python
TOOL_SCHEMAS = {
    "lookup_account": LookupAccountArgs,
    "transfer": TransferArgs,
}
```

`proposal["name"]`에는 LLM이 호출하려는 Tool 이름이 들어 있다.

```python
schema = TOOL_SCHEMAS.get(
    proposal["name"]
)
```

`lookup_account`면 `LookupAccountArgs`, `transfer`면 `TransferArgs`를 가져온다. 등록되지 않은 이름이면 허용되지 않은 Tool로 처리한다.

---

## 14. model_validate로 인자 검사

```python
try:
    args = schema.model_validate(
        proposal["args"]
    )
except ValidationError:
    return failure(
        "INVALID_ARGUMENT",
        "Tool 인자가 올바르지 않습니다.",
    )
```

`model_validate()`는 딕셔너리가 Pydantic Schema 조건에 맞는지 검사한다.

```plain text
입력: {"account_id": "acct-001"}
→ LookupAccountArgs 조건 통과
→ args.account_id로 사용 가능

입력: {"account_id": "x"}
→ 최소 길이 3 미만
→ ValidationError
```

---

## 15. 권한, 송금 한도, 승인 검사

```python
def validate_proposal(proposal, context):
    if context["cancelled"]:
        return {"status": "cancelled", ...}

    schema = TOOL_SCHEMAS.get(proposal["name"])
    if schema is None:
        return failure("UNKNOWN_TOOL", ...)

    try:
        args = schema.model_validate(proposal["args"])
    except ValidationError:
        return failure("INVALID_ARGUMENT", ...)

    if args.account_id not in context["allowed_accounts"]:
        return failure("FORBIDDEN", ...)

    if proposal["name"] == "transfer":
        if args.amount > MAX_TRANSFER:
            return failure("LIMIT_EXCEEDED", ...)

        if args.request_id in context["rejected_request_ids"]:
            return {"status": "cancelled", ...}

        if args.request_id not in context["approved_request_ids"]:
            return {"status": "waiting_for_human", ...}

    return args
```

검사에 실패하거나 승인을 기다려야 하면 사유가 담긴 `dict`를 반환한다. 모든 검사를 통과하면 Pydantic 인자 객체를 반환한다.

---

## 16. 실행 Context

```python
execution_context = {
    "user_id": "user-1",
    "allowed_accounts": {"acct-001"},
    "approved_request_ids": set(),
    "rejected_request_ids": set(),
    "cancelled": False,
}
```

Agent 실행 시 전달한다.

```python
agent.invoke(
    {"messages": [...]},
    context=execution_context,
)
```

Middleware에서는 다음 코드로 읽는다.

```python
request.runtime.context
```

Context는 그래프의 State와 달리 현재 실행의 사용자 권한, 승인 목록, 설정 같은 실행 환경 정보를 전달한다.

---

## 17. 실행 전 Middleware

```python
@wrap_tool_call
def validate_tool_call(
    request: ToolCallRequest,
    handler,
):
    result = validate_proposal(
        request.tool_call,
        request.runtime.context,
    )

    if isinstance(result, dict):
        return ToolMessage(
            content=json.dumps(
                result,
                ensure_ascii=False,
            ),
            tool_call_id=request.tool_call["id"],
            name=request.tool_call["name"],
            status="error",
        )

    return handler(request)
```

`isinstance(result, dict)`는 단순히 값이 존재하는지 확인하는 코드가 아니다. 이 예제에서는 `dict`를 **실패·취소·승인 대기 결과**로 사용하기 때문에 실행을 막는 조건이다.

`handler(request)`는 다음 Middleware 또는 실제 Tool 실행으로 넘기는 함수다. 검사를 통과했을 때만 호출한다.

---

## 18. 실행 후 결과 검증 Middleware

```python
class LookupResponse(BaseModel):
    account_id: str
    balance: int
    currency: Literal["KRW"]
```

```python
@wrap_tool_call
def validate_tool_result(request, handler):
    message = handler(request)

    if (
        request.tool_call["name"] != "lookup_account"
        or message.status == "error"
    ):
        return message

    try:
        response = LookupResponse.model_validate_json(
            message.content
        )
    except ValidationError:
        return make_result_error(...)

    if response.account_id != request.tool_call["args"]["account_id"]:
        return make_result_error(...)

    return message
```

실행 전 검사와 달리 먼저 `handler(request)`를 호출한다. 즉 Tool을 실행해 결과 메시지를 받은 다음, 결과의 필드·형식과 요청 계좌 일치 여부를 검사한다.

```plain text
실행 전 Middleware
→ 실행 후 Middleware 진입
→ 실제 Tool 실행
→ 실행 후 Middleware가 결과 검사
→ 실행 전 Middleware로 결과 반환
```

---

## 19. create_agent와 Middleware 등록

```python
agent = create_agent(
    model=llm,
    tools=tools,
    system_prompt=system_prompt,
    middleware=[
        force_tool_call,
        validate_tool_call,
        validate_tool_result,
    ],
)
```

함수에 `@wrap_tool_call`을 붙였다고 자동으로 모든 Agent에 적용되는 것은 아니다. `create_agent(..., middleware=[...])` 목록에 넣어야 해당 Agent의 호출 흐름에 연결된다.

직접 `StateGraph`로 Agent를 만들었다면 같은 검사를 별도 Node로 구성할 수 있다. `create_agent`는 일반적인 Tool 호출 Agent를 빠르게 만들 때 편리하다.

---

## 20. Tool 호출 횟수 제한

```python
limited_agent = create_agent(
    model=llm,
    tools=tools,
    middleware=[
        ToolCallLimitMiddleware(
            run_limit=0,
            exit_behavior="continue",
        ),
        force_tool_call,
        validate_tool_call,
        validate_tool_result,
    ],
)
```

`run_limit=0`이면 한 번의 Agent 실행에서 Tool 호출을 허용하지 않는다. 특정 Tool만 제한할 수도 있다.

```python
ToolCallLimitMiddleware(
    tool_name="lookup_account",
    run_limit=2,
)
```

---

## 21. Tool이 많아질 때 검사 함수 분리

```python
TOOL_GUARDS = {
    "lookup_account": validate_lookup,
    "transfer": validate_transfer,
}
```

```python
@wrap_tool_call
def validate_by_tool(request, handler):
    tool_call = request.tool_call
    guard = TOOL_GUARDS.get(tool_call["name"])

    if guard is None:
        result = failure("UNKNOWN_TOOL", ...)
    else:
        result = guard(
            tool_call["args"],
            request.runtime.context,
        )

    if isinstance(result, dict):
        return ToolMessage(...)

    return handler(request)
```

Tool이 적을 때는 하나의 `validate_proposal()`에서 이름으로 나눌 수 있다. Tool이 많아지면 Tool별 검사 함수를 만들고 매핑표에서 선택하면 코드가 읽기 쉬워진다.

---

## 22. Context로 Tool 실행 허용하기 실습

```python
@tool
def get_notice() -> str:
    """안내 문구를 조회한다."""
    print("Tool 실행")
    return "오늘 수업은 오후 6시에 시작합니다."
```

```python
@wrap_tool_call
def validate_tool_call(request, handler):
    result = request.runtime.context["allow_tool"]

    if not result:
        return ToolMessage(
            content=json.dumps(
                {
                    "message":
                    "Tool 실행이 허용되지 않았습니다."
                },
                ensure_ascii=False,
            ),
            tool_call_id=request.tool_call["id"],
            name=request.tool_call["name"],
            status="error",
        )

    return handler(request)
```

```plain text
allow_tool=True
→ 검사를 통과
→ handler(request)
→ get_notice 실행

allow_tool=False
→ ToolMessage(error) 반환
→ get_notice는 실행되지 않음
```

계좌 예제에서는 복잡한 `validate_proposal()`을 호출했고, 마지막 실습에서는 Boolean 하나만 확인하도록 단순화한 것이다.

---

# LangGraph Logging

## 23. print 대신 logging을 사용하는 이유

`print()`는 값을 잠깐 확인하기에는 편하지만 실제 서비스 기록에는 부족하다. `logging`은 시간, 중요도, 로거 이름을 함께 남기고 화면과 파일에 서로 다른 기준으로 출력할 수 있다.

```plain text
print
→ 단순 출력

logging
→ 시간 + 레벨 + 모듈 이름 + 메시지
→ 화면과 파일 동시 기록
→ 파일 크기별 교체
→ traceback 기록
```

---

## 24. Logger, Handler, Formatter

| 구성 요소 | 역할 |
|---|---|
| Logger | 로그를 생성하고 전체 최소 레벨을 결정 |
| Handler | 로그를 화면이나 파일 등 원하는 위치로 전달 |
| Formatter | 로그 한 줄의 출력 형식을 결정 |

```python
logger = logging.getLogger("lesson.langgraph")
logger.setLevel(logging.INFO)
logger.propagate = False

console_handler = logging.StreamHandler(
    sys.stdout
)

formatter = logging.Formatter(
    "%(asctime)s | %(levelname)s | "
    "%(name)s | %(message)s",
    datefmt="%H:%M:%S",
)

console_handler.setFormatter(formatter)
logger.addHandler(console_handler)
```

`propagate=False`는 상위 로거로 다시 전달되어 로그가 중복 출력되는 것을 막는다.

Notebook 셀을 반복 실행하면 Handler가 중복 등록될 수 있어 기존 Handler를 제거한다.

```python
for handler in logger.handlers[:]:
    logger.removeHandler(handler)
    handler.close()
```

---

## 25. 로그 레벨

```plain text
DEBUG < INFO < WARNING < ERROR < CRITICAL
```

| 레벨 | 사용 예 |
|---|---|
| `DEBUG` | 변수, 중간 결과 등 상세 디버깅 정보 |
| `INFO` | 노드 시작·완료 등 정상 실행 정보 |
| `WARNING` | 실행은 가능하지만 확인이 필요한 상황 |
| `ERROR` | 특정 작업 실패 |
| `CRITICAL` | 서비스 전체에 큰 영향을 주는 심각한 오류 |

`logger.setLevel(logging.WARNING)`이면 WARNING 이상만 통과하므로 DEBUG와 INFO는 출력되지 않는다.

---

## 26. 변수와 함께 로그 남기기

```python
logger.info(
    f"node={node_name} "
    f"documents={document_count} "
    f"elapsed_ms={elapsed_ms:.2f}"
)
```

문장만 기록하는 것보다 `key=value` 형태로 남기면 어떤 노드에서 몇 개의 문서를 처리했고 얼마나 걸렸는지 검색하고 비교하기 쉽다.

---

## 27. 화면과 파일에 서로 다른 레벨 기록

```python
file_handler = logging.FileHandler(
    log_path,
    mode="a",
    encoding="utf-8",
)

file_handler.setFormatter(formatter)
logger.addHandler(file_handler)

logger.setLevel(logging.DEBUG)
console_handler.setLevel(logging.INFO)
file_handler.setLevel(logging.DEBUG)
```

로그는 Logger의 기준을 먼저 통과한 뒤 각 Handler의 기준도 통과해야 한다.

```plain text
DEBUG 로그
→ Logger DEBUG 통과
→ Console INFO에서 차단
→ File DEBUG 통과
→ 파일에만 저장
```

---

## 28. 로그 파일 로테이션

```python
file_handler = RotatingFileHandler(
    log_path,
    maxBytes=1024,
    backupCount=2,
    encoding="utf-8",
)
```

로그 파일이 계속 커지지 않도록 최대 크기에 도달하면 새 파일로 교체한다.

```plain text
29-langgraph.log    → 현재 로그
29-langgraph.log.1  → 가장 최근 백업
29-langgraph.log.2  → 그 이전 백업
```

---

## 29. logger.error와 logger.exception 차이

```python
try:
    value = 10 / 0
except ZeroDivisionError:
    logger.error("계산 실패")
```

`logger.error()`는 오류 메시지만 기록한다.

```python
try:
    value = 10 / 0
except ZeroDivisionError:
    logger.exception("계산 실패")
    raise
```

`logger.exception()`은 `except` 안에서 사용하면 오류 메시지와 traceback을 함께 기록한다. `logging`은 오류를 해결하거나 재시도하지 않으므로, 그래프에도 실패를 전달하려면 다시 `raise`해야 한다.

---

## 30. LangGraph Node에 로그 적용하기

```python
def prepare(state: LoggingState):
    logger.info("prepare 시작")
    question = state["question"].strip()
    logger.info("prepare 완료")
    return {"question": question}
```

```python
def answer(state: LoggingState):
    logger.info("answer 시작")

    try:
        if not state["question"]:
            raise ValueError("질문이 비어 있습니다.")

        response = (
            f"질문을 받았습니다: {state['question']}"
        )
    except ValueError:
        logger.exception("answer 실패")
        raise

    logger.info("answer 완료")
    return {"answer": response}
```

정상 실행 로그는 다음 순서로 나온다.

```plain text
prepare 시작
prepare 완료
answer 시작
answer 완료
```

질문이 비어 있으면 `answer 시작` 다음에 `answer 실패`와 traceback이 기록되고 `answer 완료`는 출력되지 않는다.

---

## 31. 세 파일의 전체 연결

```plain text
사용자 요청
→ Agent가 Tool 호출 제안
→ 실행 전 Middleware가 인자·권한·승인 검사
→ Tool 또는 LangGraph Node 실행
→ 일시적 오류면 RetryPolicy로 재시도
→ 중단되면 Checkpointer에서 재개
→ 재시도 소진 시 Error Handler가 실패 State 기록
→ 실행 후 Middleware가 Tool 결과 검사
→ Logger가 시작·완료·오류·traceback 기록
→ 최종 응답
```

---

## 32. 외워야 할 코드

전체 코드를 통째로 외우기보다 다음 형태를 기억하면 된다.

### 1. Node 재시도

```python
builder.add_node(
    "node_name",
    node_function,
    retry_policy=RetryPolicy(
        max_attempts=3,
        retry_on=ConnectionError,
    ),
)
```

### 2. Checkpoint 재개

```python
config = {
    "configurable": {
        "thread_id": "thread-1"
    }
}

graph.invoke(input_state, config)
graph.invoke(None, config)
```

### 3. 최종 실패 처리

```python
def error_handler(state, error):
    return Command(
        update={"status": "failed"},
        goto="finalize",
    )
```

### 4. 실행 전 Tool 검사

```python
@wrap_tool_call
def validate_tool_call(request, handler):
    result = validate(...)

    if failed(result):
        return ToolMessage(status="error", ...)

    return handler(request)
```

### 5. 실행 후 Tool 검사

```python
@wrap_tool_call
def validate_tool_result(request, handler):
    message = handler(request)
    validate_result(message)
    return message
```

### 6. Node 로그

```python
def node(state):
    logger.info("node 시작")
    try:
        result = work(state)
    except Exception:
        logger.exception("node 실패")
        raise
    logger.info("node 완료")
    return result
```

---

## 33. 이번 학습에서 기억할 점

- `RetryPolicy`는 실패한 LangGraph Node를 자동으로 다시 실행한다.
- `max_attempts`는 최초 실행을 포함한 최대 시도 횟수다.
- 반환값이 `{"status": "failed"}`인 것은 예외가 아니므로 자동 재시도되지 않는다.
- 재개하려면 Checkpointer와 같은 `thread_id`가 필요하다.
- `invoke(None, config)`는 저장된 State를 사용해 미완료 노드부터 이어서 실행한다.
- 병렬 실행에서는 완료된 노드 결과를 저장하고 실패한 노드만 재개한다.
- Super-step은 함수가 아니라 같은 차례에 실행되는 노드 묶음이다.
- Error Handler는 재시도를 모두 소진한 뒤 실패 상태를 기록하고 다음 노드로 보낼 수 있다.
- 공통 노드 정책은 해당 Builder의 기본값이며 프로그램 전체 전역 설정은 아니다.
- 멱등성은 같은 `request_id`의 외부 작업이 중복 반영되지 않게 한다.
- Tool Schema는 Tool이 받을 입력 형식이다.
- `model_validate()`는 딕셔너리가 Pydantic Schema에 맞는지 검사한다.
- `request.runtime.context`에는 현재 실행의 권한·승인·허용 여부 같은 환경 정보가 들어 있다.
- 실행 전 Middleware는 `handler()` 전에 검사하고, 실행 후 Middleware는 `handler()` 결과를 검사한다.
- `handler(request)`를 호출해야 다음 Middleware 또는 실제 Tool이 실행된다.
- Middleware는 `create_agent(..., middleware=[...])`에 등록해야 적용된다.
- Logger는 로그를 만들고, Handler는 출력 위치를 정하고, Formatter는 출력 모양을 정한다.
- `logger.exception()`은 오류 메시지와 traceback을 함께 기록한다.
- Logging은 오류를 처리하지 않으므로 필요한 경우 `raise`로 다시 전달한다.

---

## 34. 헷갈린 점

처음에는 병렬 그래프를 재개할 때 왜 `unstable_branch`만 다시 실행되는지 이해하기 어려웠다. Edge가 자동으로 실패 노드만 판단하는 것이 아니라, Checkpointer가 성공한 `stable_branch`의 결과와 실패한 Task 상태를 저장하기 때문에 LangGraph가 미완료 Task만 이어서 실행하는 것이다.

`if isinstance(result, dict)`도 딕셔너리 안에 값이 있으면 실행한다는 뜻으로 보였다. 하지만 계좌 예제에서는 검증 실패 결과를 일부러 딕셔너리로 반환하도록 약속했기 때문에, 딕셔너리면 Tool을 실행하지 않고 오류 `ToolMessage`를 반환하는 코드였다.

Tool 입력 Schema와 Structured Output도 비슷해 보였다. `LookupAccountArgs`와 `TransferArgs`는 Tool에 들어갈 입력 형식이고, `LookupResponse`는 Tool이 실행된 뒤 반환해야 할 결과 형식이다.

Middleware의 `handler`는 직접 만든 함수가 아니라 LangChain이 제공하는 다음 실행 함수다. 실행 전 검사를 통과하면 `handler(request)`를 호출해 실제 Tool로 보내고, 실행 후 검증에서는 먼저 `handler(request)`로 Tool을 실행한 뒤 결과를 검사한다.

Logging도 Error Handler처럼 오류를 해결하는 기능으로 보일 수 있지만 역할이 다르다. Logging은 실행 상황을 기록하고, RetryPolicy나 Error Handler가 실제 재시도와 실패 흐름을 처리한다.

---

## 추가!

오늘 배운 것 정리

Fault Tolerance

- 일시적인 오류는 `RetryPolicy`로 같은 노드를 다시 실행한다.
- 나중에 이어서 실행하려면 Checkpointer와 `thread_id`를 사용한다.
- 병렬 실행에서는 성공 결과는 유지하고 실패한 Task만 재개한다.
- 재시도까지 모두 실패하면 Error Handler로 실패 상태를 남기고 마무리한다.
- 외부 저장 작업에는 `request_id`를 이용한 멱등성이 필요하다.

Agent Tool Control

- LLM이 만든 Tool 이름과 인자를 그대로 신뢰하지 않는다.
- Pydantic Schema로 입력 형식을 확인한다.
- Context로 사용자 권한, 승인, 취소, Tool 허용 여부를 확인한다.
- 실행 전 Middleware에서 막을 수 있고 실행 후 Middleware에서 반환 결과도 검사할 수 있다.
- 일반적인 Tool Agent는 `create_agent`와 Middleware로 빠르게 만들고, 복잡한 흐름은 `StateGraph`의 Node로 직접 구성할 수 있다.

LangGraph Logging

- Node의 시작, 완료, 주요 변수, 오류를 로그로 남긴다.
- 화면에는 INFO 이상, 파일에는 DEBUG 이상처럼 출력 기준을 나눌 수 있다.
- `RotatingFileHandler`로 로그 파일 크기를 관리한다.
- 오류가 발생하면 `logger.exception()`으로 traceback을 남기고 `raise`로 그래프에 실패를 전달한다.

코드를 외우기보다 다음 흐름을 먼저 확인하면 된다.

```plain text
입력
→ 어떤 검사와 Node가 실행되는가
→ 실패하면 누가 재시도하는가
→ State 또는 Context에 무엇이 들어 있는가
→ 어떤 Middleware가 Tool을 막거나 통과시키는가
→ 최종 결과와 오류가 어디에 기록되는가
```
