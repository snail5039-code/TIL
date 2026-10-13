# 26.10.13 56일차(HTML · CSS · 크롤링 · Parent-Child RAG)

## [TIL] 웹 문서를 수집하고 Parent-Child RAG로 검색·답변하기

오늘은 **HTML 문서 구조**, **CSS 선택자**, **DOM과 렌더링**을 익힌 뒤 `requests`·`BeautifulSoup`·`Playwright`로 웹 데이터를 수집했다. 이어서 위키백과 문서를 PostgreSQL과 Chroma에 나누어 저장하고, 작은 Child로 검색한 뒤 연결된 Parent section으로 답변하는 **Parent-Child RAG**를 실습했다.

이론과 실습은 다음 흐름으로 이어졌다.

```plain text
HTML 구조와 CSS 선택자 이해
→ DOM·SPA·SSR·CSR 구분
→ requests와 BeautifulSoup으로 정적 HTML 수집
→ Playwright로 동적 DOM 수집
→ 원문과 section을 PostgreSQL에 저장
→ section을 Child로 청킹하고 임베딩
→ Child 검색 결과로 Parent 조회
→ Parent 본문을 LLM에 전달해 답변 생성
```

---

## 1. HTML, CSS, DOM 기초

### 1-1. HTML 문서와 요소 관계

HTML(HyperText Markup Language)은 웹 페이지의 내용과 구조를 표현한다. 요소 안에 다른 요소를 넣으면 부모·자식·형제·자손 관계가 만들어지고, 이 구조는 이후 CSS 선택과 크롤링의 기준이 된다.

```html
<ul id="post-list">
  <li id="post-1" class="post">
    <a class="title" href="#post-1">HTML 기초</a>
    <div class="details">
      <p class="body">HTML은 웹 페이지의 구조를 나타낸다.</p>
    </div>
  </li>
</ul>
```

`id`는 문서 안에서 요소 하나를 식별하고, `class`는 여러 요소를 같은 그룹으로 묶는다. `href="#post-1"`처럼 작성하면 같은 문서에서 해당 `id`를 가진 위치로 이동한다.

### 1-2. CSS 선택자

CSS(Cascading Style Sheets)는 선택자로 요소를 찾고 속성과 값으로 스타일을 적용한다. 크롤링에서도 같은 선택자 문법으로 원하는 요소를 찾는다.

| 선택자 | 의미 | 예시 |
|---|---|---|
| 태그 선택자 | 같은 태그 전체 | `a` |
| 클래스 선택자 | 해당 클래스를 가진 요소 | `.title` |
| 아이디 선택자 | 해당 ID를 가진 요소 | `#post-list` |
| 자손 선택자 | 안쪽 모든 단계에서 탐색 | `.post .body` |
| 자식 선택자 | 바로 아래 단계만 탐색 | `.post > .title` |
| 속성 선택자 | 속성 조건으로 탐색 | `button[type="button"]` |

### 1-3. DOM과 JavaScript

브라우저는 HTML을 읽어 트리 구조의 객체인 DOM(Document Object Model)을 만든다. JavaScript는 DOM의 내용·속성·구조를 읽거나 바꿀 수 있다.

```javascript
document.querySelector("#post-1 .title").textContent = "JavaScript로 바꾼 제목";
```

게시글 예제에서는 `fetch()`로 JSONPlaceholder API를 호출하고, 응답 배열의 앞 3개를 DOM 요소로 만들어 목록을 교체했다.

```plain text
버튼 클릭
→ API에 GET 요청
→ JSON 배열 응답
→ JavaScript가 DOM 요소 생성
→ 게시글 목록 교체
```

### 1-4. SPA·MPA와 SSR·CSR

두 구분은 서로 다른 기준을 사용한다.

| 구분 | 판단 기준 |
|---|---|
| MPA / SPA | 화면 이동 시 새 HTML 문서를 받는지, 현재 문서를 유지하는지 |
| SSR / CSR | 데이터를 반영한 HTML을 서버에서 만드는지, 브라우저에서 DOM에 반영하는지 |

JavaScript로 일부 내용이 바뀐다고 모두 SPA인 것은 아니다. 또한 한 서비스에서 초기 화면은 SSR로 제공하고 이후 화면 변경은 CSR로 처리할 수 있다.

---

## 2. 웹 크롤링과 스크래핑

### 2-1. 크롤링과 스크래핑

크롤링(Crawling)은 URL을 방문하고 링크를 따라 수집 대상을 탐색하는 과정이다. 스크래핑(Scraping)은 문서에서 제목·링크 같은 필요한 값을 추출해 구조화하는 과정이다.

| 데이터가 있는 위치 | 적합한 방법 |
|---|---|
| 공식 API 응답 | API 호출 후 JSON 처리 |
| 서버가 보낸 HTML | `requests` + `BeautifulSoup` |
| JavaScript 실행 후 DOM | `Playwright` |

수집 전에는 이용 조건과 `robots.txt`를 확인하고, 요청 간격과 재시도 횟수를 제한해야 한다. `429 Too Many Requests`가 반환되면 `Retry-After`를 확인하고 대기한다.

### 2-2. requests와 BeautifulSoup

`requests`는 HTTP 요청과 응답을 처리하고, `BeautifulSoup`은 받은 HTML을 파싱한다. BeautifulSoup 자체는 사이트에 접속하거나 JavaScript를 실행하지 않는다.

```python
response = requests.get(URL, timeout=15)
response.raise_for_status()

soup = BeautifulSoup(response.text, "html.parser")
articles = soup.select("a.gPFEn")
```

태그에서 `get_text(strip=True)`로 텍스트를 읽고 `get("href")`로 링크를 꺼냈다. 상대 주소는 `urljoin()`으로 절대 주소로 바꾸고, 결과를 `title`과 `url` 딕셔너리 목록으로 정리해 CSV와 JSON으로 저장했다.

### 2-3. Playwright

초기 HTML에 데이터가 없고 JavaScript 실행 뒤 나타난다면 Playwright로 실제 브라우저를 제어한다. `Locator`를 만들기만 해서는 기다리지 않으므로, 필요한 요소가 보일 때까지 명시적으로 대기한 뒤 읽어야 한다.

```python
with sync_playwright() as pw:
    browser = pw.chromium.launch(headless=False)
    page = browser.new_page()
    page.goto(URL, wait_until="domcontentloaded")

    title_locator = page.locator(TITLE_SELECTOR)
    title_locator.first.wait_for(state="visible", timeout=15000)
```

`try`와 `finally`를 사용해 수집 중 오류가 발생해도 `browser.close()`가 실행되게 했다.

---

## 3. 크롤링 원문을 PostgreSQL에 저장하기

### 3-1. HTML 정리와 section 추출

위키백과 REST API에서 받은 HTML은 `extract_page()`로 제목과 본문을 추출했다. `script`, `style`, 탐색 상자, 각주처럼 검색에 불필요한 요소는 `decompose()`로 제거했다.

`split_sections()`는 최상위 `section[data-mw-section-id]`만 선택하고, 그 안의 제목·문단·목록·표·코드를 텍스트로 합쳤다. 각주, 참고 문헌, 외부 링크, 같이 보기 section은 제외했다.

### 3-2. 원문과 Parent 테이블

| 테이블 | 저장 내용 |
|---|---|
| `crawled_documents` | URL, 문서 제목, section 구조가 남은 본문 HTML, 수집 시각 |
| `rag_parent_chunks` | section별 제목과 본문, 원문 ID, section 순서 |

`crawled_documents.id`는 `GENERATED ALWAYS AS IDENTITY`로 자동 생성된다. 저장 함수의 `RETURNING id`는 방금 저장한 행의 번호를 받아 Python의 `document_id` 변수로 돌려준다.

```sql
INSERT INTO crawled_documents (source, title, content)
VALUES (%s, %s, %s)
ON CONFLICT (source) DO UPDATE SET
    title = EXCLUDED.title,
    content = EXCLUDED.content,
    crawled_at = CURRENT_TIMESTAMP
RETURNING id
```

`ON CONFLICT (source)`는 같은 URL이 있으면 새 행을 추가하지 않고 기존 행을 갱신한다. `EXCLUDED`는 이번에 새로 저장하려던 값을 뜻한다.

### 3-3. 메모리의 리스트와 DB 저장

`parent_rows`는 실행 중인 Python 메모리에 section 정보를 임시로 담는다. 노트북을 종료하면 리스트는 사라지므로 Parent를 PostgreSQL에 저장해 이후 검색에서도 재사용했다.

```plain text
원문 HTML
→ split_sections()로 section 분리
→ parent_rows에 Parent 구성
→ rag_parent_chunks에 영구 저장
```

---

## 4. Parent-Child Chunking

### 4-1. Parent와 Child

Parent는 위키 문서의 **최상위 section 전체**, Child는 Parent 본문을 더 작게 자른 조각이다. 검색은 작은 Child로 정확하게 하고, 답변에는 연결된 Parent section 전체를 사용한다.

```plain text
Parent p0
├── Child p0:c0
├── Child p0:c1
└── Child p0:c2
```

`child_size=350`은 Child 하나의 최대 글자 수이고, `child_overlap=50`은 인접 Child 사이에 겹치는 글자 수다. `RecursiveCharacterTextSplitter`를 만들 때는 규칙만 설정하며, 실제 분할은 `split_text()`에서 실행된다.

### 4-2. Parent와 Child ID

Parent ID는 컬렉션명, 원문 ID, Parent 순번을 합친 문자열이다. Child ID는 해당 Parent ID 뒤에 Child 순번을 붙여 만든다.

```python
parent_id = f"{collection_name}:{doc['id']}:p{parent_index}"
child_id = f"{parent['parent_id']}:c{child_index}"
```

안쪽 반복문이 도는 동안 현재 Parent는 고정되고 Child 번호만 `c0`, `c1`, `c2`로 바뀐다. `:`는 각 정보를 알아보기 쉽게 나눈 구분자일 뿐 특별한 기능은 없다.

### 4-3. Child Document와 메타데이터

LangChain의 `Document`는 본문과 부가 정보를 함께 담는 문서 형식이다. Child의 본문은 `page_content`에, Parent를 찾아가기 위한 정보는 `metadata`에 넣었다.

```python
Document(
    page_content=f"제목: {parent['title']}\n\n{child_text}",
    metadata={
        "parent_id": parent["parent_id"],
        "document_id": parent["document_id"],
        "source": source_by_id[parent["document_id"]],
        "title": parent["title"],
    },
)
```

메타데이터의 `parent_id`는 검색된 Child가 어느 Parent section에 속하는지 찾아가는 연결 키다.

---

## 5. 임베딩과 Chroma 저장

### 5-1. 임베딩과 코사인 거리

임베딩(Embedding)은 글의 의미를 숫자 배열인 벡터로 바꾸는 작업이다. 의미가 비슷한 문장은 벡터 공간에서도 가까이 놓인다.

Chroma 설정에서 `space: "cosine"`을 사용했으므로 검색 결과의 `distance`는 코사인 거리다. 값이 작을수록 질문과 Child의 의미가 가깝다.

### 5-2. 중복 제외와 배치 저장

Chroma에 이미 있는 ID는 제외하고 새 Child만 저장했다. `BATCH_SIZE=32`는 글자 수가 아니라 한 번에 임베딩하고 저장할 Child의 개수다.

```python
stored_ids = set(vectorstore.get(include=[])["ids"])
pending = [
    (child_id, doc)
    for child_id, doc in zip(child_ids, child_documents)
    if child_id not in stored_ids
]

for start in range(0, len(pending), 32):
    batch = pending[start:start + 32]
    vectorstore.add_documents(
        documents=[doc for _, doc in batch],
        ids=[child_id for child_id, _ in batch],
    )
```

`zip(child_ids, child_documents)`는 같은 위치의 ID와 Document를 묶는다. `add_documents()`가 실행될 때 설정된 임베딩 모델이 Child 본문을 벡터로 바꾸고 Chroma에 저장한다.

---

## 6. Child 검색과 Parent 조회

### 6-1. Child 유사도 검색

```python
child_hits = vectorstore.similarity_search_with_score(question, k=8)
```

`child_hits`에는 최대 8개의 `(Child Document, 거리 점수)` 쌍이 들어간다. Child 본문은 관련 위치를 정확히 찾는 데 사용한다.

### 6-2. Parent 가져오기

`fetch_parents()`는 검색된 Child의 메타데이터에서 `parent_id`를 꺼내 중복을 제거하고, 최대 3개의 Parent를 PostgreSQL에서 조회한다. 조회 결과의 본문은 `page_content`, 제목·URL·ID는 `metadata`에 넣어 다시 `Document`로 만든다.

```plain text
질문
→ Child 벡터 검색
→ Child의 parent_id 확인
→ PostgreSQL에서 Parent section 조회
→ Parent 본문을 LLM에 전달
```

Child는 LLM이 직접 답변할 자료라기보다 **관련 Parent section을 고르는 검색용 조각**이다. 따라서 이 방식은 `Child 기반 검색 + Parent 기반 답변`으로 정리할 수 있다.

### 6-3. 답변 체인과 RAG 함수

`format_context()`는 Parent의 제목·출처·본문을 하나의 문자열로 합친다. `answer_from_documents()`는 질문과 이 문자열을 `prompt → llm → StrOutputParser` 체인에 전달한다.

`ask()`는 전체 흐름을 한 함수로 묶는다.

```python
def ask(question: str, child_k: int = 8, max_parents: int = 3) -> dict:
    hits = vectorstore.similarity_search_with_score(question, k=child_k)
    parents = fetch_parents(hits, max_parents=max_parents)
    return {
        "question": question,
        "children": hits,
        "parents": parents,
        "answer": answer_from_documents(question, parents),
    }
```

반환 딕셔너리 전체를 LLM에 보내는 것은 아니다. LLM에는 `question`과 Parent를 포맷한 `context`만 전달하고, `children`과 `parents`는 검색 과정과 결과를 확인하기 위해 함께 반환한다.

---

## 7. 설정 비교 실습

마지막 실습은 `child_size`, `child_k`, `max_parents`를 바꾸고 같은 질문의 검색 결과와 답변을 비교하는 테스트다. 청킹 설정이나 임베딩 입력 형식을 바꾸면 기존 데이터와 섞이지 않도록 새로운 `collection_name`을 사용한다.

| 설정 | 의미 |
|---|---|
| `child_size` | 검색용 Child 한 개의 크기 |
| `child_overlap` | 인접 Child 사이의 겹침 |
| `child_k` | 검색할 Child 후보 수 |
| `max_parents` | LLM에 전달할 Parent section 최대 수 |

Parent section이 지나치게 크면 불필요한 내용과 토큰이 늘고 핵심이 묻힐 수 있다. 긴 section은 적당한 Parent 크기로 다시 나누고, 실제 질문으로 검색 정확도와 답변 품질을 비교해야 한다.

---

## 8. 헷갈린 점

처음에는 Child를 검색했다면 Child 본문을 그대로 LLM에 전달한다고 생각했다. 하지만 Parent-Child RAG에서는 Child가 관련 위치를 찾고, `parent_id`로 연결된 section 전체가 실제 답변 자료가 된다.

또한 `parent_rows`에 section을 이미 담았는데 왜 DB에 다시 저장하는지 헷갈렸다. 리스트는 현재 실행 중에만 존재하고, PostgreSQL에 저장해야 커널을 재시작한 뒤에도 Parent를 다시 조회할 수 있다.

```plain text
작은 Child → 정확한 검색
→ parent_id → 큰 Parent section 조회
→ 충분한 내용으로 답변
```

---

## 추가!

오늘 배운 것 정리

- HTML의 부모·자식 구조와 CSS 선택자는 화면 구성뿐 아니라 크롤링 요소 탐색에도 사용한다.
- DOM은 브라우저가 HTML을 객체 트리로 표현한 결과이며 JavaScript로 변경할 수 있다.
- `requests`는 JavaScript를 실행하지 않으므로 응답 HTML에 데이터가 있는지 먼저 확인한다.
- 동적으로 나타나는 요소는 Playwright에서 표시 조건을 기다린 뒤 읽는다.
- 크롤링 원문은 PostgreSQL에 저장해 다시 수집하지 않고 재사용할 수 있다.
- Parent는 section 전체이고 Child는 Parent를 작게 나눈 검색용 청크다.
- 임베딩은 문장의 의미를 벡터로 바꾸며 코사인 거리가 작을수록 더 유사하다.
- Child 메타데이터의 `parent_id`로 관련 Parent section을 조회한다.
- LLM에는 질문과 Parent 본문을 전달하고 Child는 Parent 선택에 간접적으로 사용한다.
- 청킹 크기와 검색 개수는 같은 질문으로 비교해 적절한 값을 정한다.

웹 문서를 RAG에 연결할 때는 다음 순서로 생각하면 된다.

```plain text
데이터가 API·HTML·동적 DOM 중 어디에 있는지 확인
→ 적절한 수집 도구 선택
→ 원문과 구조를 보존해 저장
→ Parent와 Child 생성
→ Child 임베딩 및 검색
→ Parent 조회
→ LLM 답변 생성
```

#부트캠프 #멀티캠퍼스부트캠프 #AI캠퍼스 #AI에이전트엔지니어
