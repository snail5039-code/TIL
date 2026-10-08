# 26.09.30 49일차(Python LangGraph Agent 평가 · Golden Set · Interrupt · State 변화)

## [TIL] LangGraph 금융 Agent 평가 설계: Golden Set · Interrupt · 상태 변화

오늘은 `30-langgraph-evaluation.ipynb`의 마지막 실습을 바탕으로 가상 금융 업무 Agent의 평가 기준을 설계했다.

수업 노트북의 Tool 호출 평가를 그대로 복사하지 않고, 현재 프로젝트에서 실제로 확인할 수 있는 다음 세 항목으로 바꿨다.

- **답변 평가**: 최종 답변이 기대 기준을 충족하는가?

- **흐름 평가**: 필요한 interrupt를 올바른 순서로 거쳤는가?

- **상태 평가**: 실행 후 금융 데이터가 기대한 값으로 변했는가?

```plain text
사용자 입력
→ Agent 실행
→ 최종 답변 평가
→ interrupt 흐름 평가
→ 실행 전후 JSON 상태 변화 평가
```

---

## 1. 오늘의 목표

오늘의 목표는 Golden Set과 평가 러너의 골격을 만드는 것이었다.

```plain text
Golden Set 작성
→ Agent 실행
→ 답변 평가
→ interrupt 평가
→ 데이터 평가
→ 전체 결과 출력
```

현재까지 만든 범위는 다음과 같다.

- Golden Set 8개

- `run_agent()` 실행 구조

- `evaluate_answer()` 답변 평가

- `evaluate_interrupts()` 흐름 평가

아직 `evaluate_data()`와 전체 사례를 반복 실행할 `main()`은 구현 전이다.

## 2. 왜 세 가지를 평가하는가

금융 Agent는 답변만 자연스럽다고 올바른 것이 아니다.

예를 들어 “10만 원 이체가 완료되었습니다”라고 답해도 실제 잔액이 바뀌지 않았다면 실패다. 반대로 돈은 이동했지만 비밀번호 확인이나 사용자 승인을 건너뛰었다면 이것도 실패다.

| 평가 항목 | 확인할 내용 |
| --- | --- |
| 답변 | 금액, 계좌, 처리 결과를 정확히 안내했는가? |
| 흐름 | `secret` → `approval` 등 필요한 interrupt를 순서대로 거쳤는가? |
| 데이터 | 잔액, 거래내역, 예약이체, 요청 기록이 기대대로 변했는가? |

한 항목의 통과가 다른 항목의 정확성을 보장하지 않으므로 세 평가를 분리했다.

## 3. Golden Set 구조

Golden Set은 입력과 기대 결과를 미리 정의한 평가용 정답 데이터다.

`case_id`는 함수 이름이 아니라 평가 사례를 구분하는 고유한 이름이다.

```python
{
    "case_id": "transfer_approve",
    "inputs": {
        "turns": [
            "생활비에서 저축으로 10만원 보내줘",
            "1234",
            "승인",
        ],
    },
    "reference": {
        "answer_criteria": "생활비에서 저축으로 100,000원 이체 완료를 안내한다.",
        "expected_interrupts": ["secret", "approval"],
        "expected_data": {
            "accounts": {
                "acc-001": {"balance": 1331800},
                "acc-002": {"balance": 2380000},
            },
            "transactions_added": 2,
            "last_request": {
                "task_type": "이체",
                "status": "완료",
            },
        },
    },
}
```

`reference`는 기대값이다.

- `answer_criteria`: 기대 답변의 의미

- `expected_interrupts`: 기대하는 interrupt 종류와 순서

- `expected_data`: 실행 후 기대하는 저장 상태

평가 항목이 세 개이므로 답변만 평가하는 Golden Set보다 구조가 길어졌다.

## 4. 평가 사례 8개

정상 흐름과 예외 흐름이 함께 드러나도록 다음 사례를 선정했다.

| `case_id` | 입력 `turns` | 기대 답변 | 기대 interrupt | 핵심 데이터 기준 |
| --- | --- | --- | --- | --- |
| `account_list` | 계좌 목록과 잔액 조회 | 세 계좌와 잔액 안내 | 없음 | 변화 없음 |
| `transfer_approve` | 10만 원 이체 → 1234 → 승인 | 100,000원 이체 완료 | `secret` → `approval` | 두 계좌 잔액 변경, 거래 2건 추가 |
| `transfer_reject` | 10만 원 이체 → 1234 → 거절 | 이체하지 않았음을 안내 | `secret` → `approval` | 잔액 유지, 거래 0건, 요청 상태 거절 |
| `transfer_missing_amount` | 금액 없이 이체 요청 | 금액 추가 질문 | `question` | 변화 없음 |
| `transfer_insufficient` | 1억 원 이체 요청 | 잔액 부족 안내 | 없음 | 변화 없음 |
| `conditional_transfer` | 40만 원을 남기고 이체 → 1234 → 승인 | 1,031,800원 이체 완료 | `secret` → `approval` | 생활비 400,000원, 거래 2건 추가 |
| `account_history` | 이번 달 생활비 출금 내역 조회 | 출금 내역 안내 | 없음 | 변화 없음 |
| `scheduled_transfer` | 내일 오전 9시 이체 → 1234 → 승인 | 100,000원 예약 이체 안내 | `secret` → `approval` | 예약 1건 추가, 현재 잔액 유지 |

`inputs["turns"]`에는 최초 질문뿐 아니라 interrupt 이후 입력할 비밀번호와 승인·거절도 순서대로 넣는다.

## 5. `expected_data` 설계 기준

`changed`, `accounts`, `transactions_added`, `last_schedule`은 LangGraph 예약어가 아니다. 평가 코드에서 사용할 규칙을 직접 정한 것이다.

중요한 것은 이름 자체보다 Golden Set과 `evaluate_data()`가 같은 이름과 의미를 사용하도록 통일하는 것이다.

```plain text
Golden Set의 expected_data
        ↓ 같은 키로 해석
evaluate_data()
        ↓ before / after 비교
PASS 또는 FAIL
```

아무 데이터도 바뀌면 안 되는 조회·실패 사례는 간단히 표현한다.

```python
"expected_data": {
    "changed": False,
}
```

반대로 이체 승인처럼 실제 데이터가 변해야 하는 사례는 계좌별 잔액, 추가 거래 수, 마지막 요청 상태를 구체적으로 기록한다.

## 6. `run_agent`의 역할

`run_agent(inputs)`는 Golden Set 한 사례를 실제 Agent에 넣고 세 평가에 필요한 결과를 수집하는 실행 함수다.

```plain text
초기 데이터 복원
→ 실행 전 JSON 저장
→ 첫 turn으로 Agent 실행
→ interrupt 종류 기록
→ 다음 turn으로 같은 thread 재개
→ 실행 후 JSON 저장
→ answer / interrupts / before / after 반환
```

각 사례는 같은 초기 잔액에서 시작해야 한다.

```python
shutil.copy(INITIAL_DATA, DATA)
data_store.setup(DATA, INITIAL_DATA)
before = load_data()
```

또한 사례마다 새로운 `thread_id`를 사용해 이전 사례의 대화와 interrupt 상태가 섞이지 않게 한다.

```python
config = {
    "configurable": {"thread_id": str(uuid4())},
    "recursion_limit": 40,
}
```

interrupt는 발생 순서가 중요하므로 `interrupt_seen`은 딕셔너리 `{}`가 아니라 리스트 `[]`여야 한다.

## 7. 답변 평가와 LLM Judge

답변은 의미가 같아도 문장이 달라질 수 있다. 그래서 문자열 완전 일치 대신 LLM Judge가 실제 답변과 기대 기준을 비교한다.

```python
answer_judge = create_llm_as_judge(
    prompt=CORRECTNESS_PROMPT,
    judge=model,
    feedback_key="answer_correct",
)
```

```python
def evaluate_answer(inputs, outputs, reference_outputs):
    return answer_judge(
        inputs=inputs["turns"][0],
        outputs=outputs["answer"],
        reference_outputs=reference_outputs["answer_criteria"],
    )
```

노트북은 `inputs["question"]`을 사용했지만, 이번 Golden Set은 여러 turn을 담으므로 첫 질문인 `inputs["turns"][0]`을 전달한다.

## 8. interrupt 흐름 평가

interrupt는 의미 판단이 필요하지 않으므로 기대 목록과 실제 목록을 정확하게 비교한다.

```python
def evaluate_interrupts(inputs, outputs, reference_outputs):
    expected = reference_outputs["expected_interrupts"]
    actual = outputs["interrupts"]
    passed = actual == expected

    return {
        "key": "interrupt_flow",
        "score": passed,
        "comment": f"기대 interrupt: {expected}, 실제 interrupt: {actual}",
    }
```

- `expected`: Golden Set에 적은 기대 흐름

- `actual`: 실행 중 `run_agent()`가 수집한 흐름

- `passed`: 종류와 순서가 모두 같으면 `True`

`["secret", "approval"]`을 기대할 때 두 종류가 모두 발생했더라도 순서가 반대라면 실패한다.

## 9. 상태 변화 평가 방향

`evaluate_data()`는 노트북에서 복사하는 함수가 아니다. 현재 프로젝트에 맞게 새로 만드는 상태 비교 함수다.

| `expected_data` 키 | 비교 기준 |
| --- | --- |
| `changed` | `before != after` 결과 |
| `accounts` | 계좌 ID별 실행 후 잔액 |
| `transactions_added` | 실행 전후 거래내역 개수 차이 |
| `schedules_added` | 실행 전후 예약이체 개수 차이 |
| `last_schedule` | 마지막 예약의 출금 계좌, 입금 계좌, 금액, 상태 |
| `last_request` | 마지막 요청의 업무 유형과 처리 상태 |

예약 이체는 등록 즉시 잔액이나 거래내역을 바꾸지 않는다. 대신 예약 데이터 한 건이 추가되어야 한다. 따라서 단순한 `changed: True/False`만으로는 부족하고 기능별 상태를 구체적으로 적어야 한다.

DB를 사용해도 원리는 같다. 테스트 DB를 초기화한 뒤 실행 전후 행을 조회해 비교하면 된다.

## 10. 회고와 다음 작업

오늘 가장 크게 배운 점은 Agent 평가는 최종 문장 하나를 채점하는 작업이 아니라는 것이다.

금융처럼 실제 상태를 바꾸는 Agent는 다음을 따로 확인해야 한다.

- 사용자에게 무엇이라고 답했는가?

- 필요한 확인 절차를 어떤 순서로 거쳤는가?

- 실제 데이터는 어떻게 바뀌었는가?

가장 헷갈렸던 부분은 노트북 코드를 어디까지 가져와야 하는지였다. 노트북은 Tool 호출 기록 중심이지만 현재 Agent는 `secret`, `approval`, `question` interrupt와 JSON 상태 변화가 핵심이다.

따라서 함수의 목적은 참고하되 입력 구조와 평가 필드는 프로젝트에 맞게 바꿔야 했다.

다음 작업은 아래 순서로 진행한다.

1. `interrupt_seen = {}`를 `[]`로 수정한다.

1. `evaluate_data()`에서 기대 데이터의 각 키를 비교한다.

1. 불일치한 필드와 실제 값을 `comment`에 남긴다.

1. `main()`에서 Golden Set 8개를 반복 실행한다.

1. 답변·interrupt·데이터 평가 결과를 사례별 표로 출력한다.

1. 첫 실행 결과를 보고 Golden Set 오류와 Agent 오류를 구분해 수정한다.

```plain text
Golden Set 작성
→ run_agent로 실제 실행
→ answer / interrupt / data 평가
→ 실패 원인 확인
→ Agent 또는 Golden Set 수정
→ 같은 기준으로 다시 실행
```

오늘은 “무엇을 정답으로 볼 것인가”를 정하고 평가의 골격을 만드는 단계까지 진행했다.
