### **Structured Output과 Output Parser**

```markdown
# Structured Output과 Output Parser

Gemini 응답을 문자열, JSON(dict), Pydantic 객체로 받는 방법을 비교합니다.
```

```python
from typing import Literal

from dotenv import load_dotenv
from langchain_core.exceptions import OutputParserException
from langchain_core.language_models.fake_chat_models import FakeListChatModel
from langchain_core.output_parsers import (
    CommaSeparatedListOutputParser,
    JsonOutputParser,
    PydanticOutputParser,
    StrOutputParser,
)
from langchain_core.prompts import ChatPromptTemplate, PromptTemplate
from langchain_core.runnables import RunnableLambda
from langchain_google_genai import ChatGoogleGenerativeAI
from pydantic import BaseModel, Field

load_dotenv()
llm = ChatGoogleGenerativeAI(model="gemini-2.5-flash", temperature=0)
```

```markdown
## Output Parser란?

Output Parser를 체인의 마지막에 연결하면 LLM 응답을 필요한 Python 타입으로 바꿀 수 있습니다.

| 방법 | 결과 타입 | 사용 시점 |
|---|---|---|
| `StrOutputParser` | `str` | 일반 텍스트만 필요할 때 |
| `CommaSeparatedListOutputParser` | `list[str]` | 단순 문자열 목록이 필요할 때 |
| `JsonOutputParser` | `dict` 또는 `list` | 유연한 JSON 데이터가 필요할 때 |
| `PydanticOutputParser` | Pydantic 객체 | JSON 파싱과 스키마 검증이 필요할 때 |
| `with_structured_output()` | Pydantic 객체 등 | Gemini의 네이티브 구조화 출력을 사용할 때 |

Parser 방식은 `get_format_instructions()`로 출력 규칙을 만든 뒤 프롬프트에 넣습니다. 이는 모델에게 형식을 **강제**하는 기능이 아니라 자연어 지시로 형식 준수를 **유도**하는 방식입니다. 모델은 설명 문장을 덧붙이거나 필드를 누락하는 등 지시를 어길 수 있습니다.

- 프롬프트와 `get_format_instructions()`: 원하는 출력 형식을 안내합니다.
- Output Parser: 실제 응답을 변환하고 형식 또는 schema 위반을 검출합니다.
- `with_structured_output()`: 스키마를 모델 API에 직접 전달합니다.

따라서 parser가 형식 준수를 보장하는 것은 아닙니다. 모델이 잘못된 결과를 반환하면 파싱 또는 validation 오류가 발생하며, 애플리케이션에서 이 실패를 처리해야 합니다. 지원 모델에서는 `with_structured_output()`이 일반적으로 더 간결하고 안정적이지만, 업무적으로 올바른 값까지 보장하지는 않습니다.
```

```markdown
### 구조화 출력의 검증 단계

JSON으로 변환됐다고 해서 애플리케이션에서 사용할 수 있는 올바른 데이터라는 뜻은 아닙니다.

```text
JSON 문법 검증
→ schema 검증
→ domain 규칙 검증
→ policy 검증
```

| 검증 단계 | 확인 내용 | 실패 예 |
|---|---|---|
| JSON 문법 | 올바른 JSON인가 | 따옴표나 중괄호 누락 |
| Schema | 필드와 타입이 맞는가 | `intensity` 필드 누락 |
| Domain | 업무 범위에 맞는 값인가 | 감정 강도가 1~5 범위를 벗어남 |
| Policy | 서비스 정책상 허용되는가 | 근거 없이 위험한 판단을 확정함 |

Output Parser와 Pydantic은 주로 JSON 문법과 schema를 검증합니다. 업무 규칙이나 서비스 정책은 별도의 validator와 애플리케이션 로직으로 확인해야 합니다.
```

```markdown
## PromptTemplate.partial()

Parser 예제의 `.partial()`은 프롬프트 변수 일부를 미리 채워 새로운 PromptTemplate을 만드는 메서드입니다. Python의 `functools.partial()`처럼 매번 바뀌지 않는 값을 사전에 바인딩한다고 생각하면 됩니다.

```python
prompt = PromptTemplate.from_template(
    "입력: {text}\n{format_instructions}"
)

prompt = prompt.partial(
    format_instructions=parser.get_format_instructions()
)

# format_instructions는 이미 채워졌으므로 실행할 때 text만 전달합니다.
prompt.invoke({"text": "분석할 문장"})
```

여기서 `format_instructions`는 parser가 정한 고정 규칙이고 `text`는 호출마다 달라지는 사용자 입력입니다. `.partial()`을 사용하지 않는다면 `invoke()`를 호출할 때 두 값을 모두 전달해야 합니다.

`.partial()`은 문자열을 즉시 완성하거나 LLM을 호출하지 않습니다. 아직 채워지지 않은 변수만 입력으로 받는 새로운 프롬프트 템플릿을 반환합니다.
```

```markdown
## StrOutputParser

Gemini의 `AIMessage`에서 텍스트만 꺼내는 가장 기본적인 parser입니다.
```

```python
text_prompt = ChatPromptTemplate.from_template(
    "{topic}을 처음 배우는 사람에게 두 문장으로 설명해 주세요."
)
text_chain = text_prompt | llm | StrOutputParser()
text_result = text_chain.invoke({"topic": "LangChain"})

print(type(text_result).__name__)
print(text_result)
```

```markdown
## CommaSeparatedListOutputParser

쉼표로 구분된 짧은 목록을 `list[str]`로 바꿉니다. 태그나 키워드처럼 단순한 1차원 목록에 적합합니다. 항목 안에 쉼표가 들어가거나 구조가 복잡해지면 `JsonOutputParser`를 사용하세요.
```

```python
list_parser = CommaSeparatedListOutputParser()
list_prompt = PromptTemplate.from_template(
    "{topic}과 관련된 핵심 키워드 5개를 작성해 주세요.\n{format_instructions}"
).partial(format_instructions=list_parser.get_format_instructions())

list_chain = list_prompt | llm | list_parser
list_result = list_chain.invoke({"topic": "생성형 AI"})

print(type(list_result).__name__)
print(list_result)
```

```markdown
## JsonOutputParser

고정된 모델까지는 필요 없지만 JSON을 Python `dict`로 받고 싶을 때 사용합니다.
```

```python
json_parser = JsonOutputParser()
json_prompt = PromptTemplate.from_template(
    """다음 주제의 핵심 개념 3개를 JSON으로 정리해 주세요.
주제: {topic}
{format_instructions}"""
).partial(format_instructions=json_parser.get_format_instructions())

json_chain = json_prompt | llm | json_parser
json_result = json_chain.invoke({"topic": "RAG"})

print(type(json_result).__name__)
print(json_result)
```

```markdown
## PydanticOutputParser

LLM이 만든 JSON 텍스트를 파싱하고 Pydantic 스키마로 검증합니다. 필수 필드, 타입, 값의 범위 등을 적용할 수 있습니다.
```

```python
class SentimentResult(BaseModel):
    sentiment: Literal["긍정", "부정", "중립"] = Field(description="문장에서 나타나는 감정")
    intensity: int = Field(ge=1, le=5, description="감정 강도")
    reason: str = Field(min_length=1, description="판단 근거")

pydantic_parser = PydanticOutputParser(pydantic_object=SentimentResult)
pydantic_prompt = PromptTemplate.from_template(
    """다음 문장의 감정을 분석해 주세요.
문장: {text}
{format_instructions}"""
).partial(format_instructions=pydantic_parser.get_format_instructions())

pydantic_chain = pydantic_prompt | llm | pydantic_parser
pydantic_result = pydantic_chain.invoke({"text": "새 기능이 기대보다 훨씬 좋아서 정말 만족합니다."})

print(type(pydantic_result).__name__)
print(pydantic_result)
```

```markdown
### 설명이 아니라 타입으로 허용값 제한하기

필드 설명은 모델에게 원하는 값을 안내하지만 실제 허용값을 제한하지는 않습니다. `sentiment: str`을 사용하면 `"만족"`, `"혼합"`, `"알 수 없음"` 같은 값도 문자열이므로 검증을 통과합니다.

`Literal["긍정", "부정", "중립"]`처럼 허용값을 타입에 명시하면 Pydantic이 그 밖의 값을 검증 단계에서 차단합니다. 숫자는 `ge`, `le`, 문자열은 `min_length`처럼 데이터 특성에 맞는 제약을 함께 지정할 수 있습니다.
```

```markdown
## Gemini Structured Output

`with_structured_output()`은 Pydantic 스키마를 Gemini에 직접 전달합니다. 별도 format instructions나 parser 없이 검증된 객체를 바로 받습니다.
```

```python
structured_llm = llm.with_structured_output(SentimentResult)
structured_result = structured_llm.invoke(
    "배송은 빨랐지만 제품 포장이 찢어져 있어서 아쉬웠습니다."
)

print(type(structured_result).__name__)
print(structured_result)
```

```markdown
### PydanticOutputParser를 사용하는 경우

모델이 네이티브 structured output을 안정적으로 지원한다면 `with_structured_output()`을 우선 고려할 수 있습니다. 하지만 다음과 같은 상황에서는 `PydanticOutputParser`가 더 적합합니다.

- 사용하는 provider나 모델이 네이티브 structured output을 지원하지 않는 경우
- 여러 provider에서 같은 프롬프트와 파싱 방식을 유지해야 하는 경우
- 이미 저장된 JSON 문자열이나 외부에서 받은 텍스트를 Pydantic 객체로 검증해야 하는 경우
- 모델의 원본 텍스트를 전처리한 뒤 직접 파싱하거나 파싱 실패 과정을 세밀하게 제어해야 하는 경우
- `FakeListChatModel`처럼 출력이 문자열인 테스트 모델로 성공과 실패를 재현해야 하는 경우

| 구분 | `with_structured_output()` | `PydanticOutputParser` |
|---|---|---|
| 구조 전달 방식 | 스키마를 모델 API에 전달 | format instructions를 프롬프트에 전달 |
| 모델 요구사항 | 해당 모델의 구조화 출력 지원 필요 | 일반 텍스트 출력 모델에서도 사용 가능 |
| 검증 시점 | 모델 호출 과정과 결합 | 문자열 응답을 받은 뒤 파싱·검증 |
| 제어 범위 | 간결하지만 provider 동작에 영향을 받음 | 전처리, 예외 처리, 재시도 흐름을 직접 구성 가능 |

`PydanticOutputParser`는 프롬프트로 형식을 안내하므로 모델이 JSON 형식을 지키지 않을 수 있습니다. 따라서 네이티브 structured output의 단순한 하위 호환 기능이라기보다, 문자열 응답을 직접 다뤄야 할 때 선택하는 별도의 실행 방식으로 이해합니다.
```

```markdown
---

## Structured Output이 실패할 때

실패 원인은 크게 세 가지이며, 실패 위치에 따라 처리 방법이 달라집니다.

- **Provider/API 실패**: 모델이 네이티브 structured output을 지원하지 않거나 API 요청 자체가 실패한 경우입니다. 원본 응답이 없으므로 예외가 발생합니다.
- **파싱 실패**: 응답은 받았지만 JSON 문법이나 필요한 구조가 잘못된 경우입니다.
- **검증 실패**: JSON 구조는 맞지만 Pydantic 타입, 필수 필드 또는 범위 조건을 위반한 경우입니다.

먼저 원본 응답과 파싱 오류를 함께 보면서 원인을 구분해야 합니다. Gemini의 `with_structured_output(..., include_raw=True)`는 `raw`, `parsed`, `parsing_error`를 함께 반환하므로 디버깅에 유용합니다.

주의할 점은 `include_raw=True`일 때 파싱·검증 실패가 예외가 아니라 `parsing_error` 값으로 반환될 수 있다는 것입니다. `.with_retry()`와 `.with_fallbacks()`는 기본적으로 **예외가 발생해야** 동작하므로, 이 진단용 결과를 그대로 retry/fallback 체인에 연결하면 자동 복구가 실행되지 않을 수 있습니다. 진단 모드와 운영 복구 체인은 분리하는 편이 명확합니다.
```

```python
diagnostic_llm = llm.with_structured_output(SentimentResult, include_raw=True)
diagnostic_result = diagnostic_llm.invoke(
    "생각보다 나쁘지는 않았지만 다시 구매할지는 모르겠습니다."
)

print("parsed:", diagnostic_result["parsed"])
print("parsing_error:", diagnostic_result["parsing_error"])
print("raw:", diagnostic_result["raw"])
```

```markdown
### 재시도

이 예제는 네이티브 structured output이 아니라 **프롬프트 + 일반 `PydanticOutputParser` 방식의 재시도**를 보여줍니다. 실제 LLM이 우연히 잘못된 응답을 만들기를 기다리면 실행할 때마다 결과가 달라지므로 `FakeListChatModel`에 응답을 미리 지정합니다. 첫 번째 호출은 Pydantic 검증에 실패하고 두 번째 호출은 성공합니다.

실행 흐름은 다음과 같습니다.

```text
첫 번째 호출: intensity=10 → Pydantic 범위 검증 실패 → OutputParserException
두 번째 호출: intensity=4  → Pydantic 검증 성공 → SentimentResult 반환
```

`.with_retry()`는 같은 문자열에 parser만 다시 적용하지 않습니다. `RunnableLambda | prompt | fake_llm | parser` 전체를 다시 실행하여 모델로부터 새로운 응답을 받습니다. `retry_if_exception_type`으로 재시도할 오류를 제한하고, `stop_after_attempt`로 무한 반복을 막습니다. 실제 운영에서는 `fake_llm` 자리에 Gemini를 사용합니다.
```

```python
attempts = {"count": 0}

def count_attempts(value):
    attempts["count"] += 1
    print(f"시도 {attempts['count']}회")
    return value

fake_retry_llm = FakeListChatModel(
    responses=[
        # 첫 번째 응답: intensity가 허용 범위(1~5)를 벗어납니다.
        '{"sentiment": "긍정", "intensity": 10, "reason": "만족한다고 표현했습니다."}',
        # 두 번째 응답: 검증을 통과합니다.
        '{"sentiment": "긍정", "intensity": 4, "reason": "만족한다고 표현했습니다."}',
    ]
)

retry_demo_chain = (
    RunnableLambda(count_attempts)
    | pydantic_prompt
    | fake_retry_llm
    | pydantic_parser
).with_retry(
    retry_if_exception_type=(OutputParserException,),
    stop_after_attempt=3,
)

retry_result = retry_demo_chain.invoke({"text": "이 제품은 정말 좋습니다."})
print("총 시도 횟수:", attempts["count"])
print("최종 결과:", retry_result)
```

```markdown
### Fallback

Fallback 예제는 **네이티브 structured output 실패 → 일반 parser 방식 전환**을 단순화해서 보여줍니다. 실제 장애를 기다리지 않고 primary Runnable에서 의도적으로 `RuntimeError`를 발생시킵니다. Primary가 정상 값을 반환하면 fallback은 실행되지 않으며, 예외가 밖으로 전달될 때만 `.with_fallbacks()`가 같은 입력을 다음 체인에 전달합니다.

```text
primary_chain 실행 → RuntimeError
같은 {text} 입력을 fallback_chain에 전달
Fake 모델의 JSON 문자열 → PydanticOutputParser 검증 → 성공 객체 반환
```

여기서 primary는 실패 재현을 위한 가짜 Runnable이고 fallback은 일반 parser 체인입니다. 실제 운영에서는 primary에 `llm.with_structured_output(Schema)`를 연결하고, fallback에는 parser 기반 체인이나 다른 모델을 연결할 수 있습니다. 인증 오류나 정책 위반처럼 전환해도 해결되지 않는 오류까지 무조건 fallback하지 않도록 대상 예외를 구분해야 합니다.
```

```python
def raise_structured_output_error(_):
    print("primary 실행: 의도적인 실패")
    raise RuntimeError("네이티브 structured output을 사용할 수 없습니다.")

fake_fallback_llm = FakeListChatModel(
    responses=[
        '{"sentiment": "중립", "intensity": 2, "reason": "장점과 단점이 함께 언급되었습니다."}'
    ]
)

primary_chain = RunnableLambda(raise_structured_output_error)
fallback_chain = pydantic_prompt | fake_fallback_llm | pydantic_parser
resilient_chain = primary_chain.with_fallbacks([fallback_chain])

fallback_result = resilient_chain.invoke(
    {"text": "속도는 빨라졌지만 오류가 더 자주 발생합니다."}
)
print("fallback 결과:", fallback_result)
```

```markdown
### 모든 실패를 재시도하지는 않는다

구조화 출력이 실패했다고 해서 항상 같은 요청을 반복하면 안 됩니다. 실패 원인에 따라 처리 방법을 구분합니다.

| 실패 상황 | 권장 처리 |
|---|---|
| JSON 형식이 일시적으로 잘못됨 | 횟수를 제한하여 재시도 |
| 필수 입력이 부족함 | 사용자에게 추가 정보 요청 |
| 허용 범위를 벗어난 값 | 오류를 알리고 중단하거나 다시 입력받기 |
| 정책상 허용되지 않는 결과 | 재시도하지 않고 중단 |
| 일시적인 provider 장애 | 조건에 따라 fallback 모델 사용 |

현재 예제의 재시도는 재현 가능한 형식·검증 실패를 대상으로 합니다. 인증 오류, 잘못된 사용자 입력, 정책 위반처럼 반복해도 해결되지 않는 오류는 재시도 대상에서 제외해야 합니다. 재시도에는 반드시 최대 횟수를 설정하여 무한 반복을 막습니다.
```

```markdown
## 선택 기준

- 단순 텍스트: `StrOutputParser`
- 유연한 JSON: `JsonOutputParser`
- 프롬프트 기반 JSON 파싱과 엄격한 검증: `PydanticOutputParser`
- 모델이 지원하고 스키마가 명확한 경우: `with_structured_output()`

운영 환경에서는 파싱·검증 실패에 대비해 예외 처리와 재시도도 구성합니다. 예: `chain.with_retry(stop_after_attempt=3)`.
```

### tool_agent

# **Tool + 기본 Agent**

```python
from dotenv import load_dotenv
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.messages import HumanMessage, ToolMessage
from langchain_core.tools import tool
from langchain_google_genai import ChatGoogleGenerativeAI

load_dotenv()
MODEL_NAME="gemini-3.6-flash"
```

```markdown
### Tool 개념

- Tool은 LLM이 외부 세계와 상호작용할 수 있게 해주는 함수이다
- LLM 자체는 텍스트만 생성할 수 있지만, Tool을 통해 웹 검색, 계산, 외부 API 조회 등을 수행할 수 있다
- LLM이 "이 Tool을 호출해야겠다"고 판단하면, Tool 이름과 인자를 반환한다

```
사용자 질문 → LLM이 판단 → Tool 호출 필요?
  → Yes: Tool 이름 + 인자 반환 → Tool 실행 → 결과를 LLM에 전달 → 최종 응답
  → No: 바로 텍스트 응답
```

예를 들어 사용자가 "서울 날씨 알려줘"라고 하면, LLM은 스스로 날씨를 알 수 없다. 대신 "날씨 검색 Tool을 '서울'이라는 인자로 호출해야겠다"고 판단한다. 개발자가 실제로 Tool을 실행하고 결과를 LLM에 돌려주면, LLM이 그 결과를 자연어로 정리하여 답변한다.
```

```markdown
---

### @tool 데코레이터

- Python 함수를 LangChain Tool로 변환하는 가장 간단한 방법
- 함수의 docstring이 Tool의 설명(description)이 된다
- `@tool` 데코레이터를 붙이면 일반 함수가 LangChain Tool 객체로 변환된다. LLM은 이 Tool의 `name`, `description`, `args_schema`를 보고 어떤 Tool을 어떤 인자로 호출할지 판단한다. 따라서 **docstring(설명)과 타입 힌트가 매우 중요하다**.

`@tool(parse_docstring=True)`를 사용하고 docstring을 Google 스타일의 `Args:` 형식으로 작성하면, LangChain이 각 parameter의 설명을 자동으로 추출해 `args_schema`의 `description`에 넣어준다. 별도의 Pydantic 스키마 없이도 인자의 의미를 LLM에게 전달할 수 있다.

```python
@tool(parse_docstring=True)
def search_weather(city: str) -> str:
    """도시의 현재 날씨를 검색한다.

    Args:
        city: 날씨를 검색할 도시 이름
    """
```

`search_weather.args_schema.model_json_schema()`로 자동 생성된 parameter description을 확인할 수 있다.

#### Tool 설계 원칙

| 원칙 | 설명 |
|------|------|
| 명확한 이름 | `search_weather` > `func1` |
| 구체적인 설명 | "주어진 도시의 현재 날씨 정보를 검색한다" > "데이터를 가져온다" |
| 타입 힌트 필수 | `city: str` — LLM이 어떤 값을 넣어야 하는지 알 수 있다 |
| 예시 포함 | docstring에 입력 예시를 넣으면 정확도가 높아진다 |
| 에러 메시지 | Tool 실행 실패 시 LLM이 이해할 수 있는 메시지를 반환한다 |

**Tool을 나누는 기준**: API 엔드포인트가 아니라 **LLM이 docstring만 보고 언제 쓸지 판단할 수 있는 단위**로 나눈다. 용도가 다르면 분리하고(검색 vs 상세 조회), 파라미터 하나 차이면 합쳐도 된다. 너무 많으면 선택 정확도가 떨어지고, 너무 합치면 파라미터가 복잡해져서 LLM이 헷갈린다.
```

```python
@tool(parse_docstring=True)
def search_weather(city: str) -> str:
    """주어진 도시의 현재 날씨를 검색한다.

    Args:
        city: 날씨를 검색할 도시 이름. 예: '서울', '부산'
    """
    weather_data = {
        "서울": "맑음, 22도, 습도 45%",
        "부산": "흐림, 19도, 습도 72%",
        "제주": "비, 17도, 습도 88%",
    }
    return weather_data.get(city, f"{city}의 날씨 정보를 찾을 수 없습니다.")

# Tool 정보 확인
print(f"이름: {search_weather.name}")
print(f"설명: {search_weather.description}")
print(f"스키마: {search_weather.args_schema.model_json_schema()}")

# Tool 직접 호출
print(f"\n서울 날씨: {search_weather.invoke({'city': '서울'})}")
```

```markdown
---

### Tool 바인딩

- LLM에 사용할 수 있는 Tool 목록을 알려주는 것
- `bind_tools()`를 사용한다

`bind_tools()`로 Tool을 바인딩하면, LLM은 사용자의 질문을 보고 두 가지 중 하나를 선택한다.

1. **Tool 호출이 필요한 경우** — `tool_calls`에 호출할 Tool 정보를 담아서 반환 (content는 비어 있을 수 있음)
2. **Tool 호출이 불필요한 경우** — 일반 텍스트 응답을 content에 담아서 반환

중요한 점은, `bind_tools()`만으로는 **Tool이 실제로 실행되지 않는다**. LLM은 "이 Tool을 이 인자로 호출해줘"라고 요청할 뿐이고, 실제 실행은 개발자가 해야 한다.
```

```python
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
llm_with_tools = llm.bind_tools([search_weather])

# Tool 호출이 필요한 경우
response = llm_with_tools.invoke("서울 날씨 알려줘")
print("content:", response.content)        # 비어 있을 수 있음
print("tool_calls:", response.tool_calls)   # Tool 호출 정보

print()

# Tool 호출이 불필요한 경우
response2 = llm_with_tools.invoke("안녕하세요")
print("content:", response2.content)        # 일반 응답
print("tool_calls:", response2.tool_calls)  # 빈 리스트
```

```markdown
---

### Function Calling 비교: Google Gen AI SDK vs LangChain

| 비교 항목 | Google Gen AI SDK | LangChain @tool |
|-----------|-------------|------------------|
| Tool 정의 | JSON 스키마 직접 작성 | Python 함수 + docstring |
| 파라미터 | 수동으로 properties 정의 | 타입 힌트에서 자동 추출 |
| 결과 확인 | provider SDK 고유 응답 구조 | `AIMessage.tool_calls`로 표준화 |
| 모델 교체 | API별 코드 재작성 | `ChatGoogleGenerativeAI` → 다른 ChatModel 한 줄 변경 |
```

```markdown
---

### Tool 호출 루프

Agent는 모델이 Tool을 요청하는 동안 **모델 호출 → Tool 실행 → `ToolMessage` 추가 → 모델 재호출** 과정을 반복한다. 모델은 Tool을 직접 실행하지 않으므로 애플리케이션이 `tool_calls`를 읽고 해당 Tool을 실행해야 한다.

```
사용자 입력 → 모델 판단 → Tool 실행 → 결과 전달 → 모델 재호출 → 최종 응답
```
```

```python
tools = [search_weather]
tool_map = {tool.name: tool for tool in tools}
llm_with_tools = llm.bind_tools(tools)

MAX_ITERATIONS = 5

def run_agent(user_input: str) -> str:
    messages = [HumanMessage(content=user_input)]

    for _ in range(MAX_ITERATIONS):
        response = llm_with_tools.invoke(messages)
        messages.append(response)

        if not response.tool_calls:
            return response.content

        for tool_call in response.tool_calls:
            tool_result = tool_map[tool_call["name"]].invoke(tool_call["args"])
            messages.append(
                ToolMessage(
                    content=str(tool_result),
                    tool_call_id=tool_call["id"],
                )
            )

    return "최대 반복 횟수를 초과했습니다."
```

```python
# 다양한 질문으로 테스트
questions = [
    "서울 날씨 어때?",
    "서울이랑 부산 날씨 비교해줘",
    "안녕하세요!",
]

for q in questions:
    print(f"\n사용자: {q}")
    answer = run_agent(q)
    print(f"Agent: {answer}")
```

> ****참고****: 모델은 한 번에 여러 Tool을 요청하거나 Tool 결과를 보고 추가 Tool을 요청할 수 있다. 반복 횟수를 제한해 무한 호출을 방지해야 한다. 이 반복 구조는 이후 LangGraph에서 `ToolNode`와 `tools_condition`으로 구성한다.

```markdown
---

### 실습 문제

#### 영화 목록과 상세 조회 Agent

TMDB API와 LangChain Tool을 사용해 영화 목록을 조회하고, 후속 질문에서 특정 영화의 상세 정보를 조회하는 Agent를 구현하세요.

```.env
TMDB_API_KEY=...
```

다음 Tool을 구현합니다.

| Tool | 역할 |
|---|---|
| `get_movie_list` | 인기, 현재 상영, 개봉 예정 목록에서 영화 ID, 제목, 개봉일을 조회한다 |
| `get_movie_detail` | 영화 ID로 줄거리, 장르, 러닝타임 등 상세 정보를 조회한다 |

목록 Tool은 상세 정보를 포함하지 않습니다. 후속 질문에 답하려면 목록 결과의 영화 ID를 사용해 `get_movie_detail`을 호출해야 합니다.

다음 두 질문을 **같은 세션**에서 순서대로 실행합니다.

```text
현재 상영 중인 영화 5개를 알려줘.
그중 첫 번째 영화의 줄거리와 러닝타임을 알려줘.
```

다른 세션에서 바로 `"그중 첫 번째 영화의 줄거리를 알려줘."`라고 질문했을 때 이전 목록을 알 수 없다고 답하는지도 확인하세요.
```

```python
import os
import requests
from langchain_core.tools import tool
from dotenv import load_dotenv
from pprint import pprint

# 도구 생성!
load_dotenv()

TMDB_TOKEN = os.getenv("TMDB_TOKEN")

BASE_URL = "https://api.themoviedb.org/3/movie"

@tool(parse_docstring=True)
def get_movie_list(list_type : str):
    """영화 목록을 조회한다.
    
    Args:
        list_type: 조회할 영화 목록 종류
    """

    url = f"{BASE_URL}/{list_type}"
    headers = {
        "accept": "application/json",
        "Authorization": f"Bearer {TMDB_TOKEN}"
    }
    params = {
        "language" : "ko-KR",
        "page" : 1,
    }
    response = requests.get(url, headers=headers, params=params)

    response = response.json()
    results = response.get('results')
    return results

# pprint(get_movie_list("now_playing"))
@tool(parse_docstring=True)
def get_movie_detail(movie_id : int):
    """영화 ID로 상세정보를 조회한다.
        
    Args:
        movie_id: 상세정보를 조회 할 영화ID
    """

    url = f"{BASE_URL}/{movie_id}"
    headers = {
        "accept": "application/json",
        "Authorization": f"Bearer {TMDB_TOKEN}"
    }
    params = {
        "language" : "ko-KR",
    }
    response = requests.get(url, headers=headers, params=params)
    response = response.json()
    return response

# pprint(get_movie_detail(15))

```

```python
MODEL_NAME="gemini-3.6-flash"

llm = ChatGoogleGenerativeAI(model=MODEL_NAME)

tools = [get_movie_list, get_movie_detail]
tool_map = {tool.name: tool for tool in tools}
llm_with_tools = llm.bind_tools(tools)
MAX_ITERATIONS = 5

messages = []

def run_agent(user_input: str) -> str:
    messages.append(HumanMessage(content=user_input))

    for _ in range(MAX_ITERATIONS):
        response = llm_with_tools.invoke(messages)
        messages.append(response)

        if not response.tool_calls:
            return response.content[0]["text"]

        for tool_call in response.tool_calls:
            tool_result = tool_map[tool_call["name"]].invoke(tool_call["args"])
            messages.append(
                ToolMessage(
                    content=str(tool_result),
                    tool_call_id=tool_call["id"],
                )
            )

    return "최대 반복 횟수를 초과했습니다."

while True :
    user_input = input("입력 : ")

    if user_input == "종료":
        break

    result = run_agent(user_input)
    pprint(result)
    
```

```python
# 강사님 것
import requests
import os
from pprint import pprint

load_dotenv()

TMDB_TOKEN = os.getenv('TMDB_TOKEN')

@tool(parse_docstring=True)
def get_movies_tool(category: str, size: int = 10):
    """TMDB API를 활용해서 상영중인 영화의 목록을 가져오는 함수.

    Args:
        category: 어떤 영화의 목록을 가져올건지 정하는 파라미터. 예 : now_playing - 현재 상영중, popular - 인기있는, top_rated - 순위권인, upcoming - 개봉 예정인.
        size: 영화 목록의 사이즈. 예: 5, 3, 7
    """

    URL = f"https://api.themoviedb.org/3/movie/{category}"
    headers = {
        'authorization' : f'Bearer {TMDB_TOKEN}'
    }
    try:
        response = requests.get(URL, headers=headers)
        response.raise_for_status
        # print(response.json())
        data = response.json()
        result = [
            {
                'id' : movie.get('id'),
                'title' : movie.get('title'),
                'release_date' : movie.get('release_date')
            }
            for movie in data.get('results')
        ]
        return result[:size]
    
    except Exception as e:
        print(e)
        return {}

@tool(parse_docstring=True)
def get_movie_by_id_tool(id: int):
    """TMDB API를 활용해서 주어진 ID에 해당하는 영화의 상세 정보를 가져오는 함수.

    Args:
        id: TMDB API에서 활용되는 영화 id. 예 : 969681
    """

    URL = f"https://api.themoviedb.org/3/movie/{id}"
    headers = {
        'authorization' : f'Bearer {TMDB_TOKEN}'
    }
    try:
        response = requests.get(URL, headers=headers)
        response.raise_for_status
        # print(response.json())
        data = response.json()
        return data

    except Exception as e:
        print(e)
        return {}

print(get_movies_tool.name)
pprint(get_movies_tool.args_schema.model_json_schema())
print(get_movie_by_id_tool.name)
print(get_movie_by_id_tool.args_schema.model_json_schema())
```

```python
# 강사님 것
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
tools = [get_movies_tool, get_movie_by_id_tool]

tool_map = {
    'get_movies_tool' : get_movies_tool,
    'get_movie_by_id_tool' : get_movie_by_id_tool
}

tool_map = {
    func.name : func
    for func in tools
}

llm_with_movie_tool = llm.bind_tools(tools)

# # 실행 과정
# # 내가 지금 뭘 하고싶지?
# # 1. 상영중인 영화 5개 가져오기.
# # get_movies() 함수를 실행한다.
# data = get_movies(size=5)

# # 2. 첫번째 영화의 줄거리와 러닝타임 알려줘.
# # get_movie_by_id() 함수를 실행한다. <- id로 첫번째 get_movies()에서 가져온 movie의 id를 넣겠다.
# first_id = data[0].get('id')
# print(first_id)

# movie = get_movie_by_id(first_id)
# # pprint(movie)

# # 결과가 나온다.
# print(movie.get('overview'))

def run_agent(request: str, max_attempt: int = 5):
    messages = [HumanMessage(content=request)]

    for _ in range(max_attempt):

        response = llm_with_movie_tool.invoke(messages)

        messages.append(response)

        # 만약 일반 응답이면 그냥 실행해줘
        if not response.tool_calls:
            pprint(messages)
            return response.content

        # 만약 함수를 실행하라는 명령이면 함수를 실행해줘.
        # 그리고 해당 응답을 담아서 다시 요청해줘.

        for tool_call in response.tool_calls:
            func_name = tool_call.get('name')
            args = tool_call.get('args')
            tool_id = tool_call.get('id')
            
            func = tool_map.get(func_name)

            result = func.invoke(args)
            messages.append(
                ToolMessage(
                    content = str(result),
                    tool_call_id = tool_id
                )
            )

result = run_agent('상영중인 영화 5개 가져와줘.')
print(result)

result = run_agent('첫번째 영화의 줄거리와 러닝타임 알려줘.')
print(result)
```

```python
# 강사님 것 

from langchain_core.chat_history import InMemoryChatMessageHistory

# 대화 내역을 기억하는 세션을 만들어서 대화가 지속적으로 이어질 수 있도록 하고 싶다.

llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
tools = [get_movies_tool, get_movie_by_id_tool]

tool_map = {
    'get_movies_tool' : get_movies_tool,
    'get_movie_by_id_tool' : get_movie_by_id_tool
}

tool_map = {
    func.name : func
    for func in tools
}

llm_with_movie_tool = llm.bind_tools(tools)

momory = {
    'user-1' : InMemoryChatMessageHistory(),
    'user-2' : InMemoryChatMessageHistory()
}

def run_agent(user_id: str, request: str, max_attempt: int = 5):
    user_memory = momory.get(user_id)
    user_message = HumanMessage(content=request)
    
    user_memory.add_message(user_message)

    # messages = [
    #     *user_memory.messages, 
    #     ]

    for _ in range(max_attempt):

        response = llm_with_movie_tool.invoke(user_memory.messages)

        # messages.append(response)
        user_memory.add_message(response)

        # 만약 일반 응답이면 그냥 실행해줘
        if not response.tool_calls:
            # pprint(messages)
            pprint(user_memory.messages)
            return response.content

        # 만약 함수를 실행하라는 명령이면 함수를 실행해줘.
        # 그리고 해당 응답을 담아서 다시 요청해줘.

        for tool_call in response.tool_calls:
            func_name = tool_call.get('name')
            args = tool_call.get('args')
            tool_id = tool_call.get('id')
            
            func = tool_map.get(func_name)

            result = func.invoke(args)
            tool_message = ToolMessage(
                content = str(result),
                tool_call_id = tool_id
                )
            # messages.append(tool_message)
            user_memory.add_message(tool_message)

result = run_agent('user-1', '상영중인 영화 5개 가져와줘.')
print(result)
print()
result = run_agent('user-2', '첫번째 영화의 줄거리와 러닝타임 알려줘.')
print(result)
```

### RAG

# **RAG 기초 - 문서 처리와 임베딩**

```markdown
#### 필요한 패키지 설치
`pip install pypdf`
```

```python
import math

from dotenv import load_dotenv
from langchain_community.document_loaders import PyPDFLoader, TextLoader
from langchain_google_genai import GoogleGenerativeAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter

load_dotenv()
EMBEDDING_MODEL_NAME = "gemini-embedding-2"

```

```markdown
### RAG (Retrieval-Augmented Generation)

LLM은 학습 시점 이후의 정보나 비공개 문서를 알지 못하며, 알고 있는 것처럼 부정확한 내용을 생성할 수도 있다. 예를 들어 우리 회사의 사내 규정, 최신 제품 매뉴얼, 비공개 문서 등은 모델의 기존 학습 지식만으로 신뢰할 수 있게 답하기 어렵다.

**RAG**는 사용자의 질문과 관련된 문서를 먼저 **검색(Retrieval)** 한 뒤, 그 문서를 LLM의 프롬프트에 함께 넣어서 **응답을 생성(Generation)** 하는 방식이다. 모델을 다시 학습시키지 않고도 외부 문서를 답변의 근거로 활용할 수 있다. 다만 검색된 문서가 부정확하거나 질문과 맞지 않으면 답변 역시 부정확할 수 있다.

```
사용자 질문: "연차 신청은 어떻게 해?"

❌ LLM만 사용 → "일반적으로 HR 부서에 문의하세요..." (모호한 답변)
✅ RAG 사용  → 사내 규정 문서 검색 → "사내 포털 > 인사 > 연차 신청에서 가능합니다" (문서에 근거한 답변)
```

### RAG 파이프라인 전체 흐름

```
[ 인덱싱 단계 — 사용자 질문 전에 미리 수행 ]
문서 → 로드(DocumentLoader) → 분할(TextSplitter) → 임베딩 → 벡터 DB 저장

[ 질의 응답 단계 — 사용자 질문이 들어오면 수행 ]
질문 → 임베딩 → 벡터 DB 검색 → 관련 문서 추출 → LLM에 전달 → 응답 생성
```
```

```markdown
### Document Loader

RAG에서 사용할 문서를 로드하는 도구이다. LangChain은 다양한 형식의 문서 로더를 제공한다.

| 로더 | 형식 | 패키지 |
|------|------|--------|
| `TextLoader` | `.txt` | `langchain_community` |
| `PyPDFLoader` | `.pdf` | `langchain_community` + `pypdf` |
| `CSVLoader` | `.csv` | `langchain_community` |
| `WebBaseLoader` | 웹페이지 | `langchain_community` |

여기서 사용하는 로더의 `load()` 메서드는 `Document` 객체의 리스트를 반환한다.

```python
# Document 구조
Document(
    page_content="문서의 텍스트 내용...",
    metadata={"source": "파일경로", "page": 0, ...}
)
```

`page_content`는 검색과 임베딩에 사용할 본문이고, `metadata`는 출처와 페이지 번호 같은 **부가 정보**를 담는 딕셔너리이다. 로더마다 자동으로 채우는 항목은 다르며, 이 정보는 나중에 "이 내용이 어디서 왔는지"를 추적할 때 사용한다.

로더마다 문서를 나누는 기준이 다르다. `TextLoader`는 파일 전체를 하나의 Document로 반환하고, `PyPDFLoader`는 페이지 단위로 나누어 반환한다. `PyPDFLoader`의 `page` metadata는 0부터 시작한다.

Document Loader가 파일의 구조를 항상 완벽하게 복원하는 것은 아니다. 이미지로만 구성된 스캔 PDF는 OCR이 필요할 수 있고, 표나 다단 편집 문서는 텍스트의 읽기 순서가 달라질 수 있다.

아래에서는 텍스트 파일과 PDF 파일을 로드하는 방법을 살펴본다.

### 텍스트 파일 로드
```

```python
loader = TextLoader("data/sample_rules.txt", encoding="utf-8")
docs = loader.load()

print(f"문서 수: {len(docs)}")
print(f"metadata: {docs[0].metadata}")
print(f"내용 미리보기:\n{docs[0].page_content[:200]}")
```

```markdown
### PDF 파일 로드

`PyPDFLoader`는 PDF를 페이지 단위로 분리하여 각 페이지를 하나의 `Document`로 반환한다.
```

```python
loader = PyPDFLoader("data/SPRi AI Brief_9월호_산업동향_0909_F.pdf")
docs = loader.load()

print(f"문서 수 (= 페이지 수): {len(docs)}")
for doc in docs[:3]:  # 처음 3페이지만 미리보기
    print(f"\n--- 페이지 {doc.metadata['page']} ---")
    print(doc.page_content[:200])
```

```markdown
### Text Splitter (텍스트 분할)

문서를 통째로 임베딩하면 두 가지 문제가 생긴다.

1. **검색 정확도 저하**: 긴 문서 안에 여러 주제가 섞여 있으면, 질문과 관련 없는 내용까지 포함된다
2. **입력 크기 제한**: 임베딩 모델과 LLM은 한 번에 처리할 수 있는 토큰 수에 제한이 있다

따라서 문서를 적절한 크기의 **청크(chunk)** 로 나눈 뒤 각 청크를 임베딩한다.

### 핵심 파라미터

| 파라미터 | 설명 | 예시 |
|---------|------|------|
| `chunk_size` | 청크의 최대 크기 (문자 수) | 500 |
| `chunk_overlap` | 인접 청크 간 겹치는 부분 | 50 |

```
원본: [ABCDEFGHIJ]

chunk_size=5, overlap=2:
  청크1: [ABCDE]
  청크2: [DEFGH]
  청크3: [GHIJ]
       ↑ overlap으로 문맥 단절 방지
```

`chunk_overlap`이 있으면 청크 경계에서 문맥이 끊기는 것을 줄일 수 있다.

### RecursiveCharacterTextSplitter

일반 텍스트 분할에 권장되는 범용 분할기이다. `\n\n`(빈 줄, 문단 경계) → `\n`(줄바꿈) → ` `(공백) 순서로 분할을 시도한다. 문단, 줄, 단어 경계를 우선 사용하고, 그래도 크면 더 작은 문자 단위로 나눈다.
```

```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
)

chunks = splitter.split_documents(docs)

print(f"원본 문서 수: {len(docs)}")
print(f"분할 후 청크 수: {len(chunks)}")

for i, chunk in enumerate(chunks[8:12]):
    print(f"\n--- 청크 {i} (길이: {len(chunk.page_content)}, 페이지: {chunk.metadata.get('page')}) ---")
    print(chunk.page_content)
```

```markdown
### 다른 청킹 방법론

`RecursiveCharacterTextSplitter` 외에도 문서 특성에 맞는 청킹 방법이 있다.

| 방법 | 기준 | 적합한 상황 |
|--------|------|------------|
| `RecursiveCharacterTextSplitter` | 문자 수 (빈 줄 → 줄바꿈 → 공백 → 문자 단위) | 문단·줄·단어 경계를 우선하는 범용 문서 |
| `TokenTextSplitter` | 토큰 수 | 임베딩 모델의 토큰 제한을 정확히 맞춰야 할 때 |
| `MarkdownHeaderTextSplitter` | 마크다운 헤더 (`#`, `##` 등) | 마크다운 문서. 섹션 구조를 유지하며 분할 |
| `HTMLHeaderTextSplitter` | HTML 헤더 (`h1`, `h2` 등) | 웹페이지 크롤링 결과 |
| `SemanticChunker` | 인접 문장의 임베딩 유사도 | 주제 전환을 기준으로 나누고 싶을 때. 문장 단위 임베딩이 필요해 계산량과 API 사용량이 증가할 수 있음 |
| Parent-Child Chunking | 작은 검색용 청크와 큰 전달용 청크를 연결 | 세밀한 검색과 넓은 문맥이 모두 필요할 때. 청크 간 매핑 관리가 필요 |
```

```markdown
### 임베딩 (Embedding)

임베딩은 텍스트를 **숫자 벡터(리스트)** 로 변환하는 것이다. 의미가 비슷한 텍스트는 비슷한 벡터로 변환되므로, 벡터 간의 거리를 측정하면 텍스트의 의미적 유사도를 계산할 수 있다.

**개념 예시**

```
"고양이"  → [0.12, -0.34, 0.56, ...] ─┐
"강아지"  → [0.11, -0.31, 0.55, ...] ─┤ 가까움 (의미 유사)
"자동차"  → [-0.87, 0.42, -0.15, ...] ─ 멂 (의미 다름)
```

- 임베딩 벡터의 차원 수는 모델과 설정에 따라 다르다
- 벡터의 개별 숫자를 사람이 직접 해석하기는 어렵고, 임베딩은 의미의 정답이 아니라 모델이 학습한 방식으로 텍스트의 특징을 표현한 결과이다
- 중요한 것은 벡터 간의 **상대적 거리**이다
- 문서와 질문을 비교하려면 같은 임베딩 모델과 같은 차원 설정으로 변환해야 한다

`RecursiveCharacterTextSplitter`의 `chunk_size`는 기본적으로 문자 수를 기준으로 하지만, 임베딩 모델의 입력 제한은 토큰 수를 기준으로 하므로 청크가 모델의 최대 입력 크기를 넘지 않는지 확인해야 한다.
```

```markdown
### Gemini Embedding API

LangChain의 임베딩 모델은 두 가지 메서드를 제공한다. 두 메서드는 모두 텍스트를 벡터로 변환하지만, 검색 질문과 저장할 문서라는 입력의 역할에 맞게 구분해 사용한다.

| 메서드 | 용도 | 입력 | 출력 |
|--------|------|------|------|
| `embed_query()` | 검색 질문을 임베딩 | 단일 문자열 | 벡터 하나 |
| `embed_documents()` | 저장할 문서를 임베딩 | 문자열 리스트 | 벡터 리스트 |
```

```python
embeddings = GoogleGenerativeAIEmbeddings(model=EMBEDDING_MODEL_NAME)

# 단일 텍스트 임베딩
vector = embeddings.embed_query("고양이는 귀엽다")

print(f"벡터 차원: {len(vector)}")
print(f"처음 5개 값: {vector[:5]}")
```

```python
# 여러 텍스트를 한 번에 임베딩
texts = [
    "고양이는 귀엽다",
    "강아지는 충성스럽다",
    "Python은 프로그래밍 언어다",
]

vectors = embeddings.embed_documents(texts)

for text, vec in zip(texts, vectors):
    print(f"\"{text}\" → 차원: {len(vec)}, 처음 3개: {vec[:3]}")
```

```markdown
### 코사인 유사도 (Cosine Similarity)

두 벡터가 얼마나 비슷한 방향을 가리키는지를 측정한다. 값의 범위는 -1 ~ 1이며, 1에 가까울수록 유사하다.

```
cosine_similarity(A, B) = (A · B) / (|A| × |B|)
```

- `A · B`: 두 벡터의 내적 (각 원소를 곱한 후 합산)
- `|A|`: 벡터 A의 크기 (각 원소의 제곱합의 제곱근)

2차원 벡터로 예를 들면 직관적으로 이해할 수 있다.

!image.png

실제 임베딩은 수천 차원의 벡터이지만 계산 원리는 동일하다. 방향이 비슷하면 유사도가 높고, 다르면 낮다.

이 값은 벡터의 방향 관계를 나타내며, 자연어의 `같은 의미`, `무관한 의미`, `반대 의미`를 그대로 보장하지는 않는다. 또한 관련 여부를 나누는 보편적인 점수 기준은 없다. 점수 분포는 임베딩 모델과 데이터에 따라 다르므로 검색 결과를 비교하거나 평가 데이터로 기준을 정해야 한다.

벡터 간 거리를 측정하는 방법에는 코사인 유사도 외에도 유클리드 거리(L2), 내적(Inner Product) 등이 있다. 텍스트 임베딩 검색에서는 코사인 유사도가 흔히 사용되지만, 거리 함수는 임베딩 모델과 벡터 저장소의 설정에 맞춰야 한다.
```

```python
def cosine_similarity(vec1: list[float], vec2: list[float]) -> float:
    """차원이 같은 두 벡터의 코사인 유사도를 계산한다."""
    if len(vec1) != len(vec2):
        raise ValueError("두 벡터의 차원이 같아야 합니다.")

    dot_product = sum(a * b for a, b in zip(vec1, vec2))
    magnitude1 = math.sqrt(sum(a ** 2 for a in vec1))
    magnitude2 = math.sqrt(sum(b ** 2 for b in vec2))
    if magnitude1 == 0 or magnitude2 == 0:
        raise ValueError("0 벡터의 코사인 유사도는 계산할 수 없습니다.")

    return dot_product / (magnitude1 * magnitude2)
```

```python
sentences = [
    "고양이가 소파에서 낮잠을 잔다",
    "고양이가 침대에서 자고 있다",
    "강아지가 공원에서 뛰어놀고 있다",
    "주식 시장이 급락했다",
]

vectors = embeddings.embed_documents(sentences)

# 첫 번째 문장과 나머지 문장의 유사도 비교
base = sentences[0]
print(f"기준: \"{base}\"\n")

for i in range(1, len(sentences)):
    sim = cosine_similarity(vectors[0], vectors[i])
    print(f"  vs \"{sentences[i]}\"")
    print(f"  → 유사도: {sim:.4f}\n")
```

```markdown
### 임베딩 기반 유사도 검색

아래 예제는 사내 규정 문서를 임베딩한 뒤, 질문 벡터를 모든 문서 벡터와 직접 비교하여 코사인 유사도가 높은 문서 2개를 반환한다. 질문과 문서에 서로 다른 표현이 사용되어도 의미가 유사하면 찾을 수 있으며, 순위와 점수의 차이도 함께 확인할 수 있다.
```

```python
documents = [
    "연차 신청은 그룹웨어의 근태 관리 메뉴에서 제출합니다.",
    "재택근무를 하려면 근무일 3일 전까지 팀장의 승인을 받아야 합니다.",
    "법인카드 사용 후 영수증은 경비 처리 시스템에 등록해야 합니다.",
    "사내 계정의 비밀번호는 보안 포털에서 재설정할 수 있습니다.",
    "회의실은 사내 예약 시스템에서 최대 2시간까지 예약할 수 있습니다.",
    "주차 등록은 총무팀에 차량 번호를 제출하여 신청합니다.",
]

document_vectors = embeddings.embed_documents(documents)

queries = [
    "휴가를 쓰려면 어디에서 신청하나요?",
    "집에서 일하려면 어떤 절차가 필요한가요?",
    "회사 카드로 결제한 뒤 무엇을 해야 하나요?",
]

for query in queries:
    query_vector = embeddings.embed_query(query)

    scores = []
    for document, document_vector in zip(documents, document_vectors):
        similarity = cosine_similarity(query_vector, document_vector)
        scores.append((similarity, document))

    scores.sort(key=lambda item: item[0], reverse=True)
    print(f'질문: "{query}"')
    for rank, (score, document) in enumerate(scores[:2], start=1):
        print(f"  {rank}. [{score:.4f}] {document}")
    print()
```