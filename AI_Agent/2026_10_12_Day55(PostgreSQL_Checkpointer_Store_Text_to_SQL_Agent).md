# 26.10.12 55일차(PostgreSQL Checkpointer · Store · Text-to-SQL Agent)

## [TIL] PostgreSQL에 대화를 저장하고, DB를 조회하는 Agent 만들기

오늘은 LangGraph의 대화 기록과 사용자 프로필을 **PostgreSQL**에 저장해 커널을 재시작해도 이어지게 만들었다. 이어서 조회 전용 계정으로 DB에 접속하는 Agent를 **고정 SQL 도구** 방식과 **Text-to-SQL** 방식으로 각각 만들고, 마지막으로 자연어 문의를 분류해 구매 내역과 연결해 저장하는 문의 접수 서비스를 실습했다.

이론과 실습은 다음 흐름으로 이어졌다.

```plain text
PostgresSaver로 대화(State) 저장
→ PostgresStore로 사용자 프로필 저장
→ 조회 전용 DB 계정(GRANT) 준비
→ 고정 SQL 도구 Agent
→ Text-to-SQL 그래프 (스키마 → SQL 생성 → 검사·실행 → 답변)
→ Tool Calling 방식 Text-to-SQL
→ dvdrental 조회 Agent 실습
→ 자연어 문의 접수 서비스 실습
```

---

## 1. PostgreSQL로 대화 저장하기 (Checkpointer)

### 1-1. 패키지와 접속 정보

PostgreSQL에 LangGraph 체크포인트를 저장하려면 `langgraph-checkpoint-postgres`와 `psycopg`가 필요하다.

```bash
pip install langgraph-checkpoint-postgres "psycopg[binary]"
```

실습용 DB를 먼저 만들고, `.env`에는 접속 정보를 **개별 변수**로 저장했다.

```sql
CREATE DATABASE langgraph_prac;
```

```plain text
POSTGRES_USER=postgres
POSTGRES_PASSWORD="YOUR_PASSWORD"
POSTGRES_DB=langgraph_prac
```

비밀번호에 특수문자가 있어도 깨지지 않도록 `quote()`로 인코딩한 뒤 URI를 조합한다.

```python
user = quote(POSTGRES_USER, safe="")
password = quote(POSTGRES_PASSWORD, safe="")
db = quote(DB_NAME, safe="")

DB_URI = f"postgresql://{user}:{password}@localhost:5432/{db}"
```

### 1-2. PostgresSaver 준비

`conn`은 Python에서 PostgreSQL에 접속해 데이터를 읽고 쓰는 **연결 객체**다. 이 연결로 `PostgresSaver`를 만들고, `setup()`으로 체크포인트용 테이블을 준비한다.

```python
conn = psycopg.connect(
    DB_URI,
    autocommit=True,       # 변경 사항 즉시 반영
    prepare_threshold=0,   # 첫 실행부터 준비된 쿼리 사용
    row_factory=dict_row,  # 조회 결과를 딕셔너리로 반환
)
checkpointer = PostgresSaver(conn)
checkpointer.setup()
```

`setup()`이 만드는 테이블은 다음과 같다. 직접 다루지 않고 `PostgresSaver`가 자동으로 관리한다.

| 테이블 | 역할 |
|---|---|
| `checkpoints` | 체크포인트의 상태 정보와 메타데이터 저장 |
| `checkpoint_blobs` | 메시지 목록 등 State 값의 직렬화된 데이터 저장 |
| `checkpoint_writes` | 노드 실행 중 발생한 중간 결과 저장, 복구할 때 활용 |
| `checkpoint_migrations` | Checkpointer 테이블의 스키마 변경 이력 관리 |

### 1-3. 챗봇 연결과 대화 이어 가기

`create_agent`에 `checkpointer`만 넘기면 대화가 DB에 저장된다. 같은 `thread_id`로 호출하면 이전 메시지를 다시 보내지 않아도 저장된 대화를 참고한다.

```python
graph = create_agent(model=llm, tools=[], system_prompt="...", checkpointer=checkpointer)
config = {"configurable": {"thread_id": "postgres-chat-demo2"}}

graph.invoke({"messages": [("human", "내 이름은 민수야. PostgreSQL을 공부하고 있어.")]}, config=config)
graph.invoke({"messages": [("human", "내 이름과 공부 중인 내용을 알려줘.")]}, config=config)
```

두 번째 질문에 "이름은 **민수**이고, 공부 중인 내용은 **PostgreSQL**"이라고 답했다. `graph.get_state(config)`로 DB에 저장된 메시지 목록도 확인할 수 있다.

### 1-4. 커널 재시작 후 복원

커널을 재시작한 뒤 환경 설정 → Checkpointer 준비 → 챗봇 생성 → `config` 정의만 다시 실행하고, 대화 저장 셀은 실행하지 않았다. 같은 `thread_id`로 `get_state()`를 호출하자 이전 대화가 그대로 복원됐고, "공부 중인 주제로 복습 문제를 내줘"라고 하자 PostgreSQL 문제를 냈다. **메모리가 아니라 DB에 저장되어 있기 때문**이다.

---

## 2. PostgreSQL로 사용자 프로필 저장하기 (Store)

### 2-1. PostgresStore 준비

Checkpointer가 **스레드 단위 대화**를 저장한다면, Store는 **스레드와 무관한 사용자 정보**를 저장한다. `PostgresStore`도 같은 패키지에 있고, 같은 DB를 쓰지만 테이블은 따로 만든다.

```python
store_conn = psycopg.connect(DB_URI, autocommit=True, prepare_threshold=0, row_factory=dict_row)
store = PostgresStore(store_conn)
store.setup()
```

| 테이블 | 역할 |
|---|---|
| `store` | namespace와 key별 JSON 데이터 저장 (`prefix` = namespace 문자열, `value` = JSONB) |
| `store_migrations` | Store 테이블의 스키마 변경 이력 관리 |

### 2-2. 프로필 저장과 조회

데이터는 **namespace + key**로 구분한다. 같은 namespace와 key로 다시 `put`하면 갱신된다.

```python
user_namespace = ("users", "user-001")
store.put(namespace=user_namespace, key="profile", value={"name": "철수", "interest": "Python"})

profile = store.get(user_namespace, "profile")
# namespace: ('users', 'user-001') / key: profile / value: {'name': '철수', 'interest': 'Python'}
```

### 2-3. Checkpointer와 Store를 함께 쓰는 그래프

노드 함수에 `config`와 `store`를 인자로 받으면 LangGraph가 주입해 준다. `config`의 `user_id`로 프로필을 읽어 시스템 프롬프트에 넣었다.

```python
def memory_chatbot(state: MessagesState, config: RunnableConfig, store: BaseStore):
    user_id = config["configurable"]["user_id"]
    profile = store.get(("users", user_id), "profile")
    system = "한국어로 간결하게 답하세요. ..."
    if profile:
        system += f"\n이름: {profile.value['name']}\n관심 분야: {profile.value['interest']}"
    response = llm.invoke([SystemMessage(content=system), *state["messages"]])
    return {"messages": [response]}

profile_graph = profile_builder.compile(checkpointer=checkpointer, store=store)
```

실행 설정에는 `thread_id`(대화)와 `user_id`(사용자)를 함께 넣는다.

```python
profile_config = {"configurable": {"thread_id": "postgres-profile-chat", "user_id": "user-001"}}
```

### 2-4. 기존 스레드 vs 새 스레드

"나는 등산을 좋아해"라고 말한 뒤 커널을 재시작하고, 같은 `user_id`로 **기존 스레드**와 **새 스레드**에 같은 질문을 했다.

- 기존 대화 → 이름 **철수**, 관심 분야 **Python**, 취미 **등산**
- 새 대화 → 이름 **철수**, 관심 분야 **Python**, 취미는 **모름**

이름·관심 분야는 Store에 있으므로 어느 스레드에서나 보이고, 취미는 기존 스레드의 State에만 있으므로 새 스레드에서는 모른다. 모든 실습이 끝나면 `conn.close()`, `store_conn.close()`로 연결을 닫는다. 연결을 닫아도 DB의 데이터는 남는다.

---

## 3. 조회 전용 DB 계정 만들기 (DCL)

### 3-1. GRANT와 REVOKE

DCL(Data Control Language, 데이터 제어어)은 **누가 어떤 작업을 할 수 있는지** 권한을 관리하는 SQL이다.

| 명령 | 의미 | 예시 |
|---|---|---|
| `GRANT` | 권한 부여 | `GRANT SELECT ON country TO agent_reader;` |
| `REVOKE` | 부여한 권한 회수 | `REVOKE SELECT ON country FROM agent_reader;` |

Agent 전용 계정 `agent_reader`를 만들고 `world` DB의 테이블에 **조회 권한만** 부여했다.

```sql
CREATE ROLE agent_reader LOGIN PASSWORD 'YOUR_AGENT_PASSWORD';
GRANT CONNECT ON DATABASE world TO agent_reader;
GRANT USAGE ON SCHEMA public TO agent_reader;
GRANT SELECT ON country, city, countrylanguage TO agent_reader;
```

| 권한 | 의미 |
|---|---|
| `CONNECT` | 데이터베이스에 접속할 수 있는 권한 |
| `USAGE` | 스키마 안의 테이블·뷰 등에 접근할 수 있는 권한 |
| `SELECT` | 테이블 데이터를 조회할 수 있는 권한 |

### 3-2. 권한 확인

`WHERE FALSE`로 실제 영향 행을 없애고, `force_rollback=True` 트랜잭션으로 감싸 INSERT·UPDATE·DELETE가 막히는지만 확인했다.

```python
with psycopg.connect(DB_URI) as conn:
    with conn.transaction(force_rollback=True):
        conn.execute("DELETE FROM city WHERE FALSE")
# → psycopg.errors.InsufficientPrivilege
```

세 쿼리 모두 `권한이 없어 차단됨`이 출력됐다.

### 3-3. 조회 함수와 이중 안전장치

`execute()` 결과는 `fetchone()`(첫 행 하나, 없으면 `None`)과 `fetchall()`(모든 행 리스트, 없으면 빈 리스트)로 꺼낸다. `dict_row`를 쓰면 행이 컬럼명을 key로 하는 딕셔너리가 된다.

계정 권한에 더해 `conn.read_only = True`로 **읽기 전용 트랜잭션**을 걸어 한 번 더 막았다.

```python
def fetch_rows(query: str, params: tuple = ()) -> list[dict]:
    with psycopg.connect(DB_URI, row_factory=dict_row) as conn:
        conn.read_only = True
        return conn.execute(query, params).fetchall()
```

---

## 4. 고정 SQL 도구 Agent

### 4-1. 파라미터 바인딩

SQL의 `%s`는 값이 들어갈 자리이고, 두 번째 인자 튜플의 값이 순서대로 바인딩된다.

- `%s` 개수와 값 개수는 같아야 한다.
- 숫자도 `%s`를 쓰고, `%s` 주위에 따옴표를 붙이지 않는다.
- 값이 하나인 튜플은 `(값,)`처럼 쉼표를 붙인다.
- `ILIKE`는 대소문자를 구분하지 않는 `LIKE`다.

### 4-2. 국가 검색·도시 조회 도구

SQL은 개발자가 미리 작성하고, LLM은 **어떤 도구를 어떤 인자로 부를지**만 고른다.

```python
@tool
def search_countries(name_keyword: str) -> str:
    """영문 국가명의 일부로 검색한다. 국가 코드, 국가명, 대륙, 인구를 최대 10건 반환한다."""
    rows = fetch_rows(
        """
        SELECT code, name, continent, population
        FROM country
        WHERE name ILIKE %s
        ORDER BY name, code
        LIMIT 10
        """,
        (f"%{name_keyword}%",),
    )
    return json.dumps({"countries": rows}, ensure_ascii=False, default=str)


@tool
def get_country_cities(country_code: str) -> str:
    """국가 코드로 인구가 많은 도시를 최대 10건 조회한다. 국가 코드 예시는 KOR이다."""
    rows = fetch_rows(
        """
        SELECT ci.id, ci.name AS city_name, co.name AS country_name, ci.population
        FROM city AS ci
        JOIN country AS co ON co.code = ci.countrycode
        WHERE co.code = %s
        ORDER BY ci.population DESC, ci.id
        LIMIT 10
        """,
        (country_code.upper(),),
    )
    return json.dumps({"cities": rows}, ensure_ascii=False, default=str)
```

### 4-3. Tool Calling 결과

"대한민국의 인구가 많은 도시를 알려줘"라고 묻자 Agent가 도구를 **두 번 연속** 호출했다.

```plain text
search_countries(name_keyword="Korea") → PRK, KOR 확인
→ get_country_cities(country_code="KOR")
→ Seoul 9,981,619 / Pusan 3,804,522 / Inchon 2,559,424 ...
```

없는 코드 `ZZZ`로 조회하면 `{"cities": []}`가 반환된다.

---

## 5. Text-to-SQL 그래프

### 5-1. Text-to-SQL이란

Text-to-SQL은 자연어 질문을 **LLM이 SQL로 변환**하는 방식이다. 고정 도구에는 "국가별 도시 수"나 "대륙별 인구" 같은 집계 SQL이 없지만, Text-to-SQL은 스키마를 보고 JOIN·집계 SQL을 직접 만든다. 대신 잘못된 SQL이나 변경 명령을 만들 수 있으므로 **실행 전 검사**와 **재시도**가 필요하다.

![Text-to-SQL 그래프](./images55/01_text_to_sql_graph.png)

`load_schema → generate_sql → execute_sql` 순서로 진행하고, 오류가 나면 `generate_sql`로 돌아가 최대 3번까지 재시도한 뒤 `answer`로 간다.

### 5-2. 스키마 조회

`SQLDatabase.get_table_info()`는 컬럼·타입·PK·FK를 `CREATE TABLE` 형태 텍스트로 돌려준다. 실행하는 SQL이 아니라 **LLM에 주는 구조 설명**이다. `world.sql`에는 FK가 없어서 관계를 직접 덧붙였다.

```python
world_db = SQLDatabase.from_uri(
    DB_URI.replace("postgresql://", "postgresql+psycopg://", 1),
    include_tables=["country", "city", "countrylanguage"],
    sample_rows_in_table_info=0,
)

def read_schema() -> str:
    return world_db.get_table_info() + """
city.countrycode → country.code
countrylanguage.countrycode → country.code
country.capital → city.id
"""
```

### 5-3. SQL 생성 노드

`with_structured_output(SQLQuery)`로 SQL 문자열만 받는다. 재시도일 때는 **이전 SQL과 오류 메시지**도 함께 넘겨 고치게 한다.

```python
def generate_sql(state: SQLState):
    query = sql_model.invoke([
        SystemMessage(content=(
            "PostgreSQL 조회 SQL을 작성하세요. 제공된 테이블만 사용하세요. "
            "SELECT 문 한 개만 반환하고 데이터 변경이나 관리 명령은 만들지 마세요. "
            "전체 집계는 전체 대상 행으로 계산하고, 최종 출력 행은 LIMIT 20으로 제한하세요. "
            "조회 컬럼에는 중복되지 않는 별칭을 붙이세요. "
            "순위나 목록에는 질문에 맞는 ORDER BY를 사용하세요.\n"
            + state["schema"]
        )),
        ("human", json.dumps({
            "question": state["question"],
            "previous_sql": state.get("sql", ""),
            "error": state.get("error", ""),
        }, ensure_ascii=False)),
    ])
    return {"sql": query.sql, "attempts": state["attempts"] + 1, "error": ""}
```

### 5-4. SQLGlot으로 변경 SQL 차단

**SQLGlot**은 SQL 문자열을 구문 트리로 분석하는 라이브러리다. `SELECT`로 시작하는지만 보는 게 아니라 `walk()`로 내부 노드까지 순회해 숨은 변경 명령을 찾는다.

```python
def validate_sql(query: str) -> None:
    try:
        statements = [item for item in parse(query, read="postgres") if item is not None]
    except ParseError as error:
        raise ValueError(f"SQL 구문을 확인하세요: {error}") from error

    if len(statements) != 1 or not isinstance(statements[0], exp.Select):
        raise ValueError("SELECT 문 한 개만 허용합니다.")

    forbidden = (exp.Insert, exp.Update, exp.Delete, exp.Merge,
                 exp.Create, exp.Drop, exp.Alter, exp.Into)
    if any(isinstance(node, forbidden) for node in statements[0].walk()):
        raise ValueError("데이터 또는 테이블을 변경하는 SQL은 허용하지 않습니다.")
```

| SQL | 결과 |
|---|---|
| `SELECT code, name FROM country LIMIT 5` | 허용 |
| `INSERT` / `UPDATE` / `DELETE` / `DROP` | 차단 (SELECT 문 한 개만 허용) |
| `SELECT * INTO city_copy FROM city` | 차단 (변경 SQL) |
| `WITH removed AS (DELETE ... RETURNING *) SELECT ...` | 차단 (변경 SQL) |
| `SELECT 1; DELETE FROM city` | 차단 (SELECT 문 한 개만 허용) |

### 5-5. 실행 노드와 재시도 분기

검사와 실행 중 오류는 `error`에 담아 다음 SQL 생성에 쓴다.

```python
def execute_sql(state: SQLState):
    try:
        validate_sql(state["sql"])
        rows = fetch_rows(state["sql"])
        return {"rows": rows, "error": ""}
    except (psycopg.Error, ValueError) as error:
        return {"rows": [], "error": str(error)}


def route_result(state: SQLState):
    if state["error"] and state["attempts"] < 3:
        return "generate_sql"
    return "answer"
```

`answer` 노드는 오류가 남아 있으면 결과를 추측하지 않고 실패 메시지를 반환하고, 성공하면 조회 결과에 근거해서만 답하게 했다.

---

## 6. Text-to-SQL 조회 결과 확인

### 6-1. 등록된 도시가 많은 국가

```sql
SELECT c.code AS country_code, c.name AS country_name, COUNT(ci.id) AS city_count
FROM country c
JOIN city ci ON c.code = ci.countrycode
GROUP BY c.code, c.name
ORDER BY city_count DESC, c.code ASC
LIMIT 5
```

- 한 번에 생성 성공 (SQL 생성 횟수 1)
- CHN 363 → IND 341 → USA 274 → BRA 250 → JPN 248

### 6-2. 대륙별 인구

```sql
SELECT continent AS continent_name, SUM(population) AS total_population
FROM country
GROUP BY continent
ORDER BY total_population DESC, continent_name ASC
LIMIT 20
```

- Asia 3,705,025,700 → Africa 784,475,000 → Europe 730,074,600 → North America 482,993,000 → South America 345,780,000 → Oceania 30,401,150 → Antarctica 0

### 6-3. 데이터 추가 요청

"city 테이블에 Test City를 추가해줘"라고 요청하자, 그래프 방식에서는 LLM이 INSERT 대신 `WHERE c.name = 'Test City'` **SELECT를 생성**했고 "데이터가 존재하지 않는다"고 답했다. 최종 답변만 보면 놓치므로 **생성된 SQL을 꼭 같이 확인**해야 한다.

---

## 7. Tool Calling 방식 Text-to-SQL

### 7-1. 스키마·실행을 도구로 제공

그래프에서는 코드가 실행 순서와 재시도를 정했다면, 여기서는 **LLM이 도구 결과를 보고 다음 호출을 결정**한다.

```python
@tool
def get_schema() -> str:
    """SQL 작성에 필요한 테이블 구조와 연결 관계를 조회한다."""
    return read_schema()


@tool
def query_sql(query: str) -> str:
    """PostgreSQL SELECT 문을 검사하고 실행해 조회 결과 또는 오류를 반환한다."""
    try:
        validate_sql(query)
        rows = fetch_rows(query)
        return json.dumps({"rows": rows, "error": ""}, ensure_ascii=False, default=str)
    except (psycopg.Error, ValueError) as error:
        return json.dumps({"rows": [], "error": str(error)}, ensure_ascii=False)


sql_tool_agent = create_agent(
    model=llm,
    tools=[get_schema, query_sql],
    system_prompt="""
get_schema로 테이블 구조를 확인하고, 질문에 맞는 SELECT 문을 query_sql로 실행하세요.
오류가 발생하면 SQL을 수정해 다시 실행하세요.
조회 결과에 근거해 한국어로 답하세요.
""",
)
```

### 7-2. 결과

- 대륙별 인구 → 그래프 방식과 같은 7개 대륙 합계를 표로 답했다.
- 도시 추가 요청 → 이번에는 LLM이 실제로 `INSERT INTO city ...`를 `query_sql`에 넘겼고, `validate_sql()`이 `"SELECT 문 한 개만 허용합니다."`로 차단했다. Agent는 "조회 전용이라 추가할 수 없다"고 안내했다.

### 7-3. 두 방식 비교

| 구분 | 고정 SQL 도구 | Text-to-SQL |
|---|---|---|
| SQL 작성 | 개발자가 미리 작성 | LLM이 질문마다 생성 |
| LLM이 결정하는 값 | 도구와 인자 | 컬럼, 조건, JOIN, 집계 |
| 새 조회 요구 | 도구 추가 또는 수정 | 스키마를 바탕으로 SQL 생성 |

| 구분 | 벡터 유사도 검색 기반 RAG | Text-to-SQL 기반 질의응답 |
|---|---|---|
| 검색 기준 | 질문과 의미가 유사한 내용을 찾음 | 질문을 SQL 조건으로 바꿔 조건을 만족하는 데이터를 찾음 |

---

## 8. 실습: dvdrental 조회 Agent

### 8-1. 요구사항

- SELECT만 허용하고 그 외 SQL은 제한한다.
- 영화 관련 정보는 조회하되 `customer` 등 고객 테이블에는 접근할 수 없게 한다.

`agent_reader` 계정으로 권한을 확인하자 `film`에 대한 INSERT·UPDATE·DELETE와 `customer` SELECT가 모두 `권한이 없어 차단됨`으로 나왔다. **고객 테이블 차단을 DB 권한으로** 처리한 것이다.

### 8-2. 고정 SQL 도구 버전

질문 예시 세 개를 각각 도구로 만들었다.

```python
@tool
def search_movie_clock_120_over() -> str:
    """액션 영화 중 상영 시간이 120분 이상인 영화 10개를 제목, 상영 시간으로 보여줌"""
    rows = fetch_rows(
        """
        SELECT f.title, f.length
        FROM film AS f
        JOIN film_category AS fc ON fc.film_id = f.film_id
        JOIN category AS c ON c.category_id = fc.category_id
        WHERE c.name = 'Action'
          AND f.length >= 120
        ORDER BY f.length DESC, f.title
        LIMIT 10
        """
    )
    return json.dumps({"movies": rows}, ensure_ascii=False, default=str)
```

같은 방식으로 `count_movies_by_category`(카테고리별 영화 수), `max_film_actor`(출연 영화가 많은 배우 5명)를 만들었다.

- 액션 120분 이상 → Darn Forrester 185분, Worst Banger 185분, Casualties Encino 179분 … 10건
- "모든 고객의 이름과 이메일" → 개인정보 조회는 처리할 수 없다고 거절

### 8-3. Text-to-SQL 버전

`include_tables`에 `actor`, `category`, `film`, `film_actor`, `film_category`, `language`만 넣어 **고객 테이블은 스키마에서부터 제외**했다.

- 배우 출연 수 → Gina Degeneres 42 → Walter Torn 41 → Mary Keitel 40 → Matthew Carrey 39 → Sandra Kilmer 37
- 고객 조회 → `SELECT '접근 제한' AS result;`
- 대여료 0으로 변경 → `SELECT '접근 제한: 데이터 변경 및 관리 명령은 차단되었습니다.' AS message`

### 8-4. 심화: 내 대여 내역 조회

현재 사용자를 `customer_id = 1`로 가정하고, `rental → inventory → film`을 JOIN하는 도구를 추가했다. 조건을 **도구 SQL 안에 고정**해 다른 고객 id가 들어갈 틈을 없앴다.

```python
@tool
def search_unreturned_movies() -> str:
    """현재 사용자가 빌린 영화 중 아직 반납하지 않은 영화의 제목과 대여일을 반환"""
    rows = fetch_rows(
        """
        SELECT f.title, r.rental_date
        FROM rental AS r
        JOIN inventory AS i ON i.inventory_id = r.inventory_id
        JOIN film AS f ON f.film_id = i.film_id
        WHERE r.customer_id = %s
          AND r.return_date IS NULL
        ORDER BY r.rental_date DESC
        """,
        (1,),
    )
    return json.dumps({"categories": rows}, ensure_ascii=False, default=str)
```

- 최근 대여 5건 → Bikini Borrowers (2005-08-22 20:03:46) 등 5건
- 미반납 → 없음
- `customer_id`가 2인 고객 대여 내역 → 1번 고객만 조회 가능하다고 거절

---

## 9. 실습: 자연어 문의 접수 서비스

### 9-1. 요구사항

`users`, `purchases`, `inquiries` 세 테이블을 쓰고, 상품 정보는 `purchases`에 함께 저장한다.

- 사용자 ID는 주어진다고 가정한다 (로그인 없음, `user_id = 1`).
- 자연어 문의를 **배송 / 환불 / 교환 / 일반**으로 분류한다.
- 구매 문의면 관련 구매 내역을 조회하고, 여러 건이거나 없으면 추가 정보를 요청한다.
- 구매 문의는 `purchase_id`와 연결해, 일반 문의는 연결 없이 `inquiries`에 저장한다.

### 9-2. 그래프 구조

![문의 접수 그래프](./images55/02_inquiry_graph.png)

`classify_inquiry`에서 일반 문의는 곧바로 `save_general_inquiry`로 가고, 구매 문의는 Text-to-SQL 흐름을 탄다. `execute_sql` 이후 조회 결과 행 수에 따라 분기한다.

```python
def route_result(state: SQLState):
    if state["error"] and state["attempts"] < 3:
        return "generate_sql"
    if state["error"]:
        return "answer"
    if len(state["rows"]) == 1:
        return "save_inquiry"
    return "ask_more"
```

| 조회 결과 | 다음 노드 | 동작 |
|---|---|---|
| 오류 (3회 미만) | `generate_sql` | SQL 재생성 |
| 오류 (3회) | `answer` | 실패 메시지 |
| 1건 | `save_inquiry` | 해당 구매와 연결해 문의 저장 |
| 0건 또는 여러 건 | `ask_more` | 상품명·구매 시기 추가 요청 |

### 9-3. 분류와 저장

분류는 `with_structured_output(InquiryClass)`로 유형 문자열만 받았다. SQL 생성 프롬프트에는 `purchases.user_id = 1` 조건을 반드시 넣고, 여러 건일 때 구분할 수 있도록 `purchase_id`, `product_name`, `order_status`, `purchased_at`을 조회하게 했다.

저장은 LLM이 SQL을 만들지 않고 **코드가 파라미터 바인딩으로 INSERT**한다.

```python
def save_inquiry(purchase_id: int | None, inquiry_type: str, content: str):
    with psycopg.connect(DB_URI) as conn:
        conn.execute(
            """
            INSERT INTO inquiries (user_id, purchase_id, inquiry_type, content)
            VALUES (%s, %s, %s, %s)
            """,
            (1, purchase_id, inquiry_type, content),
        )
```

### 9-4. 실행 결과

- 분류는 모두 의도대로 됐다: 키보드 배송 3건 → **배송**, 마우스·노트북 → **환불**, 회원 정보 수정 → **일반**
- 일반 문의 → SQL 없이 `inquiries`에 저장 완료
- 구매 문의 5건 → SQL은 `WHERE user_id = 1 AND product_name LIKE '%키보드%'`처럼 정상 생성됐지만, 3번 모두 `'utf-8' codec can't decode byte ...` 오류로 실패했다. 이 부분은 다음에 이어서 해결해야 한다.

---

## 10. 헷갈린 점

### Checkpointer와 Store는 뭐가 다른가

처음에는 둘 다 "DB에 저장하는 메모리"라서 같은 것이라고 생각했다. 하지만 Checkpointer는 `thread_id` 단위로 **대화 State**를 저장하고, Store는 `namespace + key` 단위로 **스레드와 무관한 데이터**를 저장한다. 그래서 새 스레드에서도 이름(Store)은 알지만 취미(이전 스레드의 대화)는 몰랐다.

```plain text
Checkpointer: thread_id → 대화 기록 (스레드마다 따로)
Store:        ("users", user_id) + "profile" → 프로필 (모든 스레드에서 공유)
```

### "접근 제한"을 SELECT로 답하는 건 진짜 차단이 아니다

dvdrental Text-to-SQL 실습에서 프롬프트에 "SELECT문이 아니면 접근 제한을 반환해"라고 넣었더니, LLM이 `SELECT '접근 제한' AS result;`를 만들었다. 결과는 막힌 것처럼 보이지만, 실제로는 **LLM이 스스로 SQL을 안 만든 것**일 뿐이고 `validate_sql()`이나 DB 권한이 동작한 것이 아니다. 진짜 차단은 계정 권한(`GRANT`), 읽기 전용 트랜잭션, `validate_sql()`, 스키마에서 테이블 제외 같은 **코드와 DB 레벨**에서 일어나야 한다. 프롬프트는 보조 수단이다.

### 그래프 방식과 Tool Calling 방식에서 INSERT 요청 결과가 달랐다

같은 "도시 추가" 요청인데 그래프 방식은 SELECT를 생성했고, Tool Calling 방식은 INSERT를 실제로 도구에 넘겼다가 `validate_sql()`에서 막혔다. 즉 **LLM이 어떤 SQL을 만들지는 매번 다를 수 있으므로** 최종 답변이 아니라 생성된 SQL과 오류를 같이 봐야 검사 로직이 실제로 동작했는지 알 수 있다.

### 문의 서비스의 utf-8 오류는 검사 단계 문제가 아니었다

처음에는 한글 문자열 리터럴 때문에 SQLGlot 파싱이 깨진다고 보고, `validate_sql()`에서 `'...'` 값을 `'value'`로 치환한 뒤 파싱하게 바꿨다. 그래도 같은 오류가 3번 반복됐다. `UnicodeDecodeError`는 `ValueError`의 하위 클래스라서 `except (psycopg.Error, ValueError)`에 잡혀 `error`로 들어간 것이고, 리터럴을 치환한 뒤에도 났으니 **검사보다는 DB 실행·인코딩 쪽**을 확인해야 한다. 일반 문의 저장은 성공했기 때문에 한글이 들어간 SELECT 실행 경로를 중심으로 다시 볼 예정이다.

---

## 추가!

오늘 배운 것 정리

- `PostgresSaver` + `setup()`으로 대화 State를 PostgreSQL에 저장하면 커널을 재시작해도 같은 `thread_id`로 이어진다.
- `PostgresStore`는 `namespace + key`로 사용자 프로필처럼 스레드와 무관한 데이터를 저장한다.
- 노드 함수 인자에 `config`, `store`를 선언하면 LangGraph가 주입해 준다.
- Agent용 DB 계정은 `GRANT SELECT`만 주고, `conn.read_only = True`로 한 번 더 막는다.
- SQL 값은 문자열 포맷이 아니라 `%s` 파라미터 바인딩으로 넘긴다.
- 고정 SQL 도구는 안전하지만 새 질문마다 도구가 필요하고, Text-to-SQL은 유연하지만 검사·재시도가 필요하다.
- `SQLDatabase.get_table_info()`로 스키마를 텍스트로 만들어 LLM에 주고, FK가 없으면 관계를 직접 적어 준다.
- SQLGlot `parse()` + `walk()`로 `WITH` 안의 DELETE, `SELECT INTO`, 다중 문장까지 차단할 수 있다.
- 접근 제한은 프롬프트가 아니라 DB 권한, 스키마 제외, 도구 SQL 고정 조건으로 걸어야 한다.
- 결과를 볼 때는 최종 답변뿐 아니라 생성된 SQL, 오류, 시도 횟수를 같이 확인한다.

DB를 조회하는 Agent를 만들 때는 다음 순서로 생각하면 된다.

```plain text
조회 전용 계정 만들기 (GRANT SELECT, 필요한 테이블만)
→ 읽기 전용 연결 함수 (read_only, dict_row, %s 바인딩)
→ 자주 묻는 질문은 고정 SQL 도구로
→ 집계·자유 질문은 Text-to-SQL (스키마 + 관계 설명)
→ 실행 전 SQLGlot 검사, 오류 시 재시도 (최대 N회)
→ 생성된 SQL · 결과 · 답변 함께 검증
```

#부트캠프 #멀티캠퍼스부트캠프 #AI캠퍼스 #AI에이전트엔지니어
