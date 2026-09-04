## 많이 어렵다. 주말 간 반복 숙달한다!!!!!

그래도 흐름하고 대략적으로 어떤 방식으로 사용되는지는 이해했으니 하.. 그래도 힘들다..!!

```markdown
# RAG 평가

검색기가 정답 근거를 찾는지 먼저 측정하고, 검색된 근거로 만든 답변의 품질을 별도로 평가한다. 사내 규정 50문항으로 Similarity, MMR, BM25, Hybrid Search를 같은 조건에서 비교한다.
```

```markdown
### RAG 평가가 필요한 이유

RAG 파이프라인을 만들면 청크 크기, 검색 문서 수, 검색 전략과 프롬프트를 어떤 기준으로 선택할지 판단해야 한다. 대표 질문 몇 개를 눈으로 확인할 수 있지만 질문이 많아지면 느리고 주관적이다.

평가는 한 가지 도구에만 의존하지 않는다.

| 방법 | 특징 | 적합한 상황 |
|---|---|---|
| 수동 확인 | 대표 질문과 답변을 직접 읽음 | 초기 개발과 빠른 검증 |
| 검색 지표 | Hit@k, Precision@k, Recall@k, MRR | 검색 설정 비교 |
| RAGAS | LLM을 이용해 답변의 근거성과 관련성을 평가 | 파이프라인 비교 |
| LLM-as-Judge | 도메인에 맞는 기준을 직접 정의 | 규정 답변의 정확성·완전성 평가 |
| 사용자 피드백 | 실제 사용자의 반응을 수집 | 서비스 운영 |

```

```markdown
#### 패키지 설치
RAGAS 0.4.3은 최신 `langchain-community`와 호환 문제가 있으므로 다음 버전으로 설치한다. 설치 후 커널을 재시작한다.

`pip install "ragas==0.4.3" "langchain-community==0.4.1" "instructor[google-genai]==1.15.4"`

```

```python
# 평가에 사용할 라이브러리와 LangChain 구성 요소를 불러온다.
import json
import time
from pathlib import Path

import chromadb
from dotenv import load_dotenv
from kiwipiepy import Kiwi
from langchain_chroma import Chroma
from langchain_community.retrievers import BM25Retriever
from langchain_community.document_loaders import TextLoader
from langchain_classic.retrievers import EnsembleRetriever
from langchain_google_genai import ChatGoogleGenerativeAI, GoogleGenerativeAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter
from pydantic import BaseModel, Field

# .env 파일의 Gemini API 키를 환경 변수로 불러온다.
load_dotenv()

EMBEDDING_MODEL_NAME = "gemini-embedding-2"
MODEL_NAME = "gemini-3.6-flash"

# 검색용 임베딩 모델과 답변 생성용 LLM을 준비한다.
embeddings = GoogleGenerativeAIEmbeddings(model=EMBEDDING_MODEL_NAME)
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
```

```python
# 429 exhausted 해결
from google.genai.types import HttpOptions, HttpRetryOptions
from google import genai

retry_client = genai.Client(
    http_options=HttpOptions(
        retry_options=HttpRetryOptions(
            attempts=8,          # 최초 요청 포함 총 8회
            initial_delay=2.0,   # 첫 재시도 대기 시간
            max_delay=60.0,      # 최대 대기 시간
            exp_base=2,          # 지수 백오프 배수 (2초 → 4초 → 8초 → 16초 ...)
            jitter=1.0,          # 대기 시간에 랜덤 지연을 추가해 동시 재시도 요청 분산
        )
    ),
)
embeddings.client = retry_client

```

```markdown
### 평가 순서

| 평가 대상 | 확인할 내용 | 방법 |
|---|---|---|
| 검색 | 정답 규정을 찾았는가 | Hit@k, Precision@k, Recall@k, MRR |
| 답변 | 근거를 빠짐없이 사용했는가 | 직접 정의한 LLM-as-Judge |
| 근거성 | 검색 문서에 없는 내용을 만들지 않았는가 | 인용 검사, RAGAS |
| 운영 | 응답 시간이 적절한가 | latency |

검색 실패와 답변 생성 실패를 구분하기 위해 검색 평가부터 수행한다.
```

```markdown
### 평가 데이터셋 준비

평가 데이터셋은 질문, 기대 정답, 정답 문서를 사람이 미리 정리한 기준 데이터다. RAG 평가 레코드는 다음 네 필드로 구성할 수 있으며, 각 메트릭은 이 중 필요한 필드만 사용한다.

| 필드 | 설명 | 준비 방법 |
|---|---|---|
| `user_input` | 사용자 질문 | 직접 작성 |
| `reference` | 기대 정답 | 직접 작성 |
| `retrieved_contexts` | 검색된 문서 내용 | 검색기 실행 |
| `response` | LLM이 생성한 답변 | RAG 파이프라인 실행 |

`eval_qa_50.json`은 easy, medium, hard 질문을 섞어 구성한다. 쉬운 질문만 사용하면 점수가 지나치게 높아지고, 어려운 질문만 사용하면 기본 검색 문제와 복수 문서 검색 문제를 구분하기 어렵다.
```

```markdown
### 사내 규정 문서와 평가 데이터셋

검색기는 청크를 반환하고 정답은 규정 파일 단위로 제공한다. 따라서 각 청크의 `source`가 정답 파일에 포함되는지 평가한다.
```

```python
# 앞선 검색 실습과 같은 Chroma 경로와 컬렉션을 사용한다.
DATA_DIR = Path("data/company_rules")
PERSIST_DIR = "./chroma_db"
COLLECTION_NAME = "company_rules_search_advanced"

splitter = RecursiveCharacterTextSplitter(
    chunk_size=700, chunk_overlap=100,
    separators=["\n### ", "\n\n", "\n", " "],
)
source_documents = []
documents = []
for path in sorted(DATA_DIR.glob("*.md")):
    loaded_documents = TextLoader(path, encoding="utf-8").load()
    for doc in loaded_documents:
        doc.metadata.update({"source": path.name})
    source_documents.extend(loaded_documents)
    chunks = splitter.split_documents(loaded_documents)
    # 검색 결과를 구분할 수 있도록 청크마다 고유 ID를 부여한다.
    for index, chunk in enumerate(chunks):
        chunk.metadata["chunk_id"] = f"{path.stem}:{index}"
    documents.extend(chunks)

# True이면 앞선 실습에서 만든 벡터 저장소를 불러오고, False이면 새로 만든다.
LOAD_EXISTING_VECTORSTORE = True

client = chromadb.PersistentClient(path=PERSIST_DIR)
existing_collections = [
    collection.name for collection in client.list_collections()
]

if LOAD_EXISTING_VECTORSTORE:
    if COLLECTION_NAME not in existing_collections:
        raise ValueError(
            f"저장된 컬렉션 '{COLLECTION_NAME}'이 없습니다. "
            "LOAD_EXISTING_VECTORSTORE를 False로 변경하세요."
        )
    vectorstore = Chroma(
        collection_name=COLLECTION_NAME,
        embedding_function=embeddings,
        client=client,
    )
    print(f"기존 벡터 저장소를 불러왔습니다: {COLLECTION_NAME}")
else:
    # 재생성할 때 문서가 중복되지 않도록 기존 컬렉션을 삭제한다.
    if COLLECTION_NAME in existing_collections:
        client.delete_collection(COLLECTION_NAME)

    # 분할한 청크를 임베딩하여 Chroma에 저장한다.
    vectorstore = Chroma.from_documents(
        documents=documents,
        embedding=embeddings,
        collection_name=COLLECTION_NAME,
        client=client,
    )
    print(f"규정 청크 {len(documents)}개를 새로 저장했습니다.")
```

```python
# 질문별 기대 답변과 정답 출처가 담긴 평가 데이터를 불러온다.
eval_cases = json.loads((DATA_DIR / "eval_qa_50.json").read_text(encoding="utf-8"))
for case in eval_cases:
    case["relevant_sources"] = case["source"]

print(f"전체 문항: {len(eval_cases)}")
```

```markdown
### 검색 평가와 답변 평가

검색 평가는 `question`, `relevant_sources`, `retrieved`만 있으면 실행할 수 있다. 답변 평가는 검색 결과로 답변을 생성한 뒤 `retrieved_contexts`, `response`, `reference`를 추가로 사용한다.

```text
질문과 정답 문서
→ 검색기 실행
→ Hit@k·Recall@k·MRR
→ 답변 생성
→ 인용 검사·RAGAS·LLM-as-Judge
```
```

```markdown
### 검색 전략

모든 전략은 같은 사내 규정 문서와 동일한 후보 개수를 사용한다. Hybrid Search는 `EnsembleRetriever`로 벡터 검색과 BM25 순위를 RRF 방식으로 결합한다.
```

```python
# 앞선 검색 실습과 같은 Kiwi 형태소 분석기를 사용한다.
kiwi = Kiwi()

def kiwi_tokenize(text):
    tokens = kiwi.tokenize(text)
    return [
        token.form.lower()
        for token in tokens
        if token.tag.startswith(("N", "V", "M", "X"))
        or token.tag in {"SL", "SN"}
    ]

# 모든 검색 전략이 동일한 수의 후보 청크를 반환하도록 설정한다.
FETCH_K = 5

similarity_retriever = vectorstore.as_retriever(
    search_kwargs={"k": FETCH_K}
)
mmr_retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": FETCH_K, "fetch_k": 20, "lambda_mult": 0.5},
)
bm25_retriever = BM25Retriever.from_documents(
    documents, preprocess_func=kiwi_tokenize, k=FETCH_K
)
# 벡터 검색과 BM25 결과를 같은 가중치로 결합한다.
hybrid_retriever = EnsembleRetriever(
    retrievers=[similarity_retriever, bm25_retriever],
    weights=[0.5, 0.5],
    id_key="chunk_id",
)

def similarity_search(query):
    return similarity_retriever.invoke(query)

def mmr_search(query):
    return mmr_retriever.invoke(query)

def bm25_search(query):
    return bm25_retriever.invoke(query)

def hybrid_search(query):
    return hybrid_retriever.invoke(query)

# 반복 평가할 검색 전략을 이름과 함수의 딕셔너리로 관리한다.
SEARCH_STRATEGIES = {
    "Similarity": similarity_search,
    "MMR": mmr_search,
    "BM25": bm25_search,
    "Hybrid": hybrid_search,
}
```

```markdown
### 검색 평가 예시

하나의 검색 결과로 각 지표의 계산 방법과 결과를 차례대로 확인한다.
```

```python
# 8번 문항을 실제 Chroma 검색으로 확인한다.
case = eval_cases[7]
docs = similarity_search(case["question"])

# retrive한 검색 결과 상위 5개
ranked = [doc.metadata["source"] for doc in docs[:5]]

# 내가 만든 검색 정답.
relevant = case["relevant_sources"]

print("질문:", case["question"])
print("검색 결과:", ranked)
print("정답 문서:", relevant)
```

```markdown
### Hit@k

Hit@k는 상위 `k`개 검색 결과에서 정답 출처를 하나라도 찾았는지 측정한다. 청크의 `source`가 정답 출처에 포함되면 정답을 찾은 것으로 판단한다.

```text
Hit@k = 정답 출처가 하나라도 있으면 1, 없으면 0
```

값이 1이면 상위 `k`개 안에서 답변에 사용할 근거를 찾았고, 0이면 찾지 못했다는 의미다.
```

```python
# 상위 k개 안에 정답 문서가 하나라도 있으면 1을 반환한다.
def hit_at_k(ranked, relevant, k):
    return int(bool(set(ranked[:k]) & set(relevant)))

    # 만약 relevant가 확실히 1개라면, `relevant in ranked[:k]` 라는 뜻.

print("Hit@1:", hit_at_k(ranked, relevant, 1))
print("Hit@3:", hit_at_k(ranked, relevant, 3))
```

```markdown
### Precision@k

Precision@k는 상위 `k`개 검색 결과 중 질문과 관련 있는 청크의 비율을 측정한다. 청크의 `source`가 정답 출처에 포함되면 관련 있는 청크로 판단한다.

```text
Precision@k = 상위 k개 중 관련 있는 청크 수 / 상위 k개 청크 수
```

값이 높을수록 LLM에 전달되는 불필요한 문맥이 적다는 의미다.
```

```python
# 상위 k개 검색 결과 중 정답 문서가 차지하는 비율을 계산한다.
def precision_at_k(ranked, relevant, k):
    selected = ranked[:k]
    hits = sum(source in relevant for source in selected)
    return hits / len(selected) if selected else 0.0

print("Precision@3:", precision_at_k(ranked, relevant, 3))
```

```markdown
### Recall@k

Recall@k는 필요한 전체 정답 출처 중 상위 `k`개에서 찾은 출처의 비율을 측정한다. 같은 출처의 청크가 여러 번 검색돼도 하나의 출처를 찾은 것으로 판단한다.

```text
Recall@k = 상위 k개에서 찾은 정답 출처 수 / 전체 정답 출처 수
```

값이 높을수록 답변에 필요한 근거를 빠짐없이 가져왔다는 의미다.
```

```python
# 전체 정답 출처 중 상위 k개에서 찾은 출처의 비율을 계산한다.
def recall_at_k(ranked, relevant, k):
    hits = len(set(ranked[:k]) & set(relevant))
    return hits / len(set(relevant))

print("Recall@3:", recall_at_k(ranked, relevant, 3))
```

```markdown
### RR(Reciprocal Rank)과 MRR(Mean Reciprocal Rank)

RR은 한 질문에서 처음 등장한 정답 출처 순위의 역수다. 정답 출처가 여러 개여도 가장 먼저 등장한 출처의 순위만 사용한다. 아래 예시에서는 질문 하나를 평가하므로 RR을 계산한다.

```text
RR = 1 / 첫 번째 정답 출처의 순위
```

RR이 1에 가까울수록 첫 번째 정답 출처가 검색 결과의 앞쪽에 있다는 의미다.

MRR은 여러 질문에서 계산한 RR의 평균이다. 마지막 종합 평가에서는 평가 데이터의 50개 질문마다 RR을 계산한 뒤 평균을 낸다.

```text
MRR = 모든 질문의 RR 합 / 전체 질문 수
```

MRR이 1에 가까울수록 여러 질문에서 첫 번째 정답 출처가 전반적으로 앞에 나타난다는 의미다.
```

```python
# 처음 등장한 정답 문서 순위의 역수를 계산한다.
def reciprocal_rank(ranked, relevant):
    relevant = set(relevant)
    for rank, source in enumerate(ranked, start=1):
        if source in relevant:
            return 1 / rank
    return 0.0

print("RR:", reciprocal_rank(ranked, relevant))
```

```markdown
### 동일 평가셋으로 전략 비교

각 질문에서 실제 상위 5개 청크의 출처를 평가한다. latency는 질문당 평균 검색 시간이다.
```

```python
# 하나의 검색 전략을 전체 질문에 실행하고 질문별 지표를 기록한다.
def evaluate_strategy(label, search_fn, cases, max_k=5):
    rows, started = [], time.perf_counter()
    for case in cases:
        docs = search_fn(case["question"])
        ranked = [doc.metadata["source"] for doc in docs[:max_k]]
        relevant = case["relevant_sources"]
        rows.append({
            "id": case["id"], "question": case["question"],
            "difficulty": case["difficulty"],
            "relevant": relevant, "retrieved": ranked,
            "hit@1": hit_at_k(ranked, relevant, 1),
            "hit@3": hit_at_k(ranked, relevant, 3),
            "hit@5": hit_at_k(ranked, relevant, 5),
            "precision@5": precision_at_k(ranked, relevant, 5),
            "recall@3": recall_at_k(ranked, relevant, 3),
            "recall@5": recall_at_k(ranked, relevant, 5),
            "rr": reciprocal_rank(ranked, relevant),
        })
    latency = (time.perf_counter() - started) * 1000 / len(cases)
    return {"label": label, "rows": rows, "latency_ms": latency}

# 질문별 점수를 평균 내어 검색 전략의 전체 성능을 구한다.
def average(rows, key):
    return sum(row[key] for row in rows) / len(rows) if rows else 0.0

def summarize(result, rows=None):
    rows = result["rows"] if rows is None else rows
    keys = ["hit@1", "hit@3", "hit@5", "precision@5", "recall@3", "recall@5"]
    return {"strategy": result["label"], 
            "count": len(rows), 
            **{key: average(rows, key) for key in keys}, 
            "mrr" : average(rows, "rr"),
            "latency_ms": result["latency_ms"]}

# 여러 검색 전략의 요약 결과를 표 형태로 출력한다.
def show_table(items):
    keys = ["strategy", "count", "hit@1", "hit@3", "hit@5", "precision@5", "recall@3", "recall@5", "mrr", "latency_ms"]
    print(" | ".join(f"{key:>11}" for key in keys))
    print("-" * 152)
    for item in items:
        values = [item["strategy"], item["count"]]
        values += [f"{item[key]:.3f}" for key in keys[2:]]
        print(" | ".join(f"{str(value):>11}" for value in values))

# 동일한 평가 데이터로 네 가지 검색 전략을 비교한다.
evaluation_results = {label: evaluate_strategy(label, fn, eval_cases) for label, fn in SEARCH_STRATEGIES.items()}
show_table([summarize(result) for result in evaluation_results.values()])
```

```python
# 전체 평균에 가려진 차이를 확인하기 위해 난이도별로 다시 집계한다.
for difficulty in ["easy", "medium", "hard"]:
    print(f"\n=== {difficulty} ===")
    show_table([
        summarize(result, [row for row in result["rows"] if row["difficulty"] == difficulty])
        for result in evaluation_results.values()
    ])
```

```markdown
### 실패 사례

전체 평균뿐 아니라 `Recall@5 < 1`인 질문에서 어떤 출처를 놓쳤는지 확인한다.
```

```python
# 정답 출처를 모두 찾지 못한 질문을 검색 전략별로 확인한다.
for label, result in evaluation_results.items():
    failures = [row for row in result["rows"] if row["recall@5"] < 1]
    print(f"\n=== {label}: {len(failures)}개 ===")
    for row in failures[:10]:
        print(f"[{row['id']}] {row['question']}")
        print("  정답:", row["relevant"])
        print("  검색:", row["retrieved"])
```

```markdown
### 결과 해석

- Hit@k는 최소 하나의 근거를 찾는 능력을 보여준다.
- Recall@k는 여러 규정을 종합하는 질문의 검색 범위를 보여준다.
- MRR은 제한된 context에서 첫 근거가 얼마나 빨리 나오는지 보여준다.
- 검색 품질이 비슷하면 latency와 구현 복잡도가 낮은 전략을 선택할 수 있다.
- 가장 복잡한 전략이 항상 가장 좋은 것은 아니다.

`retrieval_strategy_scenarios.json`은 특징 관찰용 fixture이며 전체 성능 평가에는 사용하지 않는다.
```

```markdown
### 검색 문서 수 비교

`k`는 검색기가 반환하는 상위 문서 수다.

- `k`가 작을 때: LLM에 전달하는 문서가 적어 비용과 불필요한 정보는 줄지만, 답변에 필요한 근거를 놓칠 수 있다.
- `k`가 클 때: 필요한 근거를 포함할 가능성은 높아지지만, 관련 없는 문서와 context 길이가 늘어나 답변 품질, 비용, 응답 시간에 영향을 줄 수 있다.

적절한 `k`는 질문과 문서의 특성에 따라 달라지므로 검색 지표와 생성 답변을 함께 확인해 정한다.
```

```markdown
### 답변 생성과 규칙 기반 검사

앞에서는 검색기가 정답 문서를 찾았는지 평가했다. 이제 검색된 문서를 LLM에 전달해 실제 답변을 만들고, 답변에 표시한 출처를 간단한 규칙으로 검사한다. LLM은 답변과 사용한 규정 파일명을 구조화된 형태로 반환한다. 코드에서는 다음 두 조건을 참과 거짓으로 확인한다.

- `valid_citation`: LLM이 표시한 출처가 실제 검색된 문서에 포함되는가
- `relevant_citation`: LLM이 표시한 출처 중 정답 문서가 하나 이상 있는가

이 검사는 같은 답변과 출처에 대해 항상 같은 결과를 내지만, 답변의 문장별 내용이 근거와 일치하는지 또는 질문에 충분히 답했는지는 판단하지 못한다. 이러한 의미적 품질은 이어지는 RAGAS와 직접 정의한 Judge에서 평가한다.

다음 예제는 Similarity와 Hybrid에 같은 질문 3개를 입력하므로 총 6개의 답변이 생성된다.
```

```python
# 답변과 사용한 출처를 구조화된 형태로 받는다.
class GroundedAnswer(BaseModel):
    answer: str = Field(description="검색 근거로 작성한 답변")
    citations: list[str] = Field(description="답변에 사용한 규정 파일명")

answer_llm = llm.with_structured_output(GroundedAnswer)

# 하나의 검색 전략으로 질문 하나의 답변을 생성한다.
def generate_answer(strategy, search_fn, case, k=5):
    docs = search_fn(case["question"])[:k]
    context = "\n\n".join(
        f"[출처: {doc.metadata['source']}]\n{doc.page_content}"
        for doc in docs
    )
    result = answer_llm.invoke(
        "검색 근거만 사용해 답하고, 사용한 파일명을 citations에 작성하세요.\n\n"
        f"질문: {case['question']}\n\n검색 근거:\n{context}"
    )

    retrieved_sources = {doc.metadata["source"] for doc in docs}
    citations = set(result.citations)
    return {
        **case,
        "strategy": strategy,
        "response": result.answer,
        "citations": result.citations,
        "retrieved_sources": list(retrieved_sources),
        # Judge가 인용 파일과 본문을 연결할 수 있도록 출처명을 함께 보존한다.
        "retrieved_contexts": [
            f"[출처: {doc.metadata['source']}]\n{doc.page_content}"
            for doc in docs
        ],
        # 인용이 하나 이상 있고, 인용한 모든 출처가 실제 검색 결과에 포함되면 True다.
        # 집합에서 citations <= retrieved_sources는 citations가 부분집합인지 확인한다.
        "valid_citation": bool(citations) and citations <= retrieved_sources,
        # 인용 출처와 정답 출처의 교집합이 하나라도 있으면 True다.
        # 집합에서 &는 두 집합에 공통으로 들어 있는 값을 구한다.
        "relevant_citation": bool(citations & set(case["relevant_sources"])),
    }

# 두 검색 전략에 동일한 질문 3개를 사용한다.
GENERATION_CASE_LIMIT = 3
GENERATION_STRATEGIES = {
    "Similarity": similarity_search,
    "Hybrid": hybrid_search,
}
generation_results = [
    generate_answer(strategy, search_fn, case)
    for strategy, search_fn in GENERATION_STRATEGIES.items()
    for case in eval_cases[:GENERATION_CASE_LIMIT]
]

for item in generation_results:
    print(
        f"[{item['strategy']}] {item['id']} | "
        f"검색 문서만 인용: {item['valid_citation']} | "
        f"정답 문서 인용: {item['relevant_citation']}"
    )
```

```markdown
### RAGAS를 이용한 답변 평가

RAGAS는 RAG 파이프라인의 검색 결과와 생성 답변을 평가하는 라이브러리다. 정답 문서의 순위만 계산하는 검색 지표와 달리, LLM과 embedding을 이용해 답변의 의미를 평가할 수 있다. 이 과정에서는 다음 네 가지 정보를 한 문항으로 묶어 전달한다.

| 입력 | 의미 |
|---|---|
| `user_input` | 사용자의 질문 |
| `response` | RAG가 생성한 답변 |
| `retrieved_contexts` | 답변 생성에 사용한 검색 문서 |
| `reference` | 사람이 준비한 기준 답변 |

RAGAS에는 평가하려는 대상에 따라 여러 메트릭이 있다.

| 메트릭 | 사용하는 주요 필드 | 확인하는 내용 |
|---|---|---|
| Faithfulness | `user_input`, `response`, `retrieved_contexts` | 답변의 주장이 검색 문서로 뒷받침되는가 |
| Answer Relevancy | `user_input`, `response` | 답변이 사용자 질문의 핵심과 관련되는가 |
| Context Precision | `user_input`, `reference`, `retrieved_contexts` | 검색 결과의 앞쪽에 정답 작성에 유용한 문서가 배치되었는가 |
| Context Recall | `user_input`, `reference`, `retrieved_contexts` | 기준 정답에 필요한 내용을 검색 문서가 빠짐없이 포함하는가 |
| Factual Correctness | `response`, `reference` | 생성 답변의 사실이 기준 정답과 일치하는가 |
| Semantic Similarity | `response`, `reference` | 생성 답변과 기준 정답의 의미가 유사한가 |

모든 메트릭을 한 번에 사용할 필요는 없다. 검색 품질, 답변 근거성, 정답 일치처럼 평가 목적에 맞는 메트릭을 선택한다. 이 수업에서는 검색 지표와 역할이 겹치지 않도록 Faithfulness와 Answer Relevancy를 사용한다. RAGAS에서 사용하는 LLM은 사용자에게 보여 줄 답변을 만드는 것이 아니라 생성된 답변을 평가하는 역할을 한다.

RAGAS는 LLM을 여러 번 호출하므로 비용과 실행 시간이 발생한다. 따라서 `RUN_RAGAS = False`를 기본값으로 두고 필요한 경우에만 실행한다.
```

```python
# 검색 전략별 생성 결과를 RAGAS 입력으로 변환한다.
ragas_records = {
    strategy: [
        {
            "user_input": item["question"],
            "response": item["response"],
            "retrieved_contexts": item["retrieved_contexts"],
            "reference": item["expected_answer"],
        }
        for item in generation_results
        if item["strategy"] == strategy
    ]
    for strategy in GENERATION_STRATEGIES
}
input_response_pairs = [
    {
        "strategy": strategy,
        "user_input": record["user_input"],
        "response": record["response"],
    }
    for strategy, records in ragas_records.items()
    for record in records
]
print(json.dumps(input_response_pairs, ensure_ascii=False, indent=2))
```

```python
# 여기 코드는 전-혀 볼 필요 없습니다.

# True로 변경하면 RAGAS의 LLM 기반 평가를 실행한다.
RUN_RAGAS = True

if RUN_RAGAS:
    import instructor
    from google import genai
    from ragas.embeddings.base import embedding_factory
    from ragas.llms import InstructorLLM
    from ragas.metrics.collections import AnswerRelevancy, Faithfulness

    # collections 메트릭에서 사용하는 RAGAS 전용 모델을 준비한다.
    google_client = genai.Client()
    instructor_client = instructor.from_genai(
        google_client, use_async=True
    )
    evaluator_llm = InstructorLLM(
        client=instructor_client, model=MODEL_NAME, provider="google"
    )
    evaluator_embeddings = embedding_factory(
        "google",
        model=EMBEDDING_MODEL_NAME,
        client=google_client,
    )

    # 단건과 batch의 embedding 차원이 달라지지 않도록 하나씩 처리한다.
    async def embed_texts_individually(texts, **kwargs):
        return [
            await evaluator_embeddings.aembed_text(text, **kwargs)
            for text in texts
        ]

    evaluator_embeddings.aembed_texts = embed_texts_individually
    faithfulness_metric = Faithfulness(llm=evaluator_llm)
    relevancy_metric = AnswerRelevancy(
        llm=evaluator_llm, embeddings=evaluator_embeddings
    )

    ragas_results = {}
    for strategy, records in ragas_records.items():
        # 전략별 답변의 근거 충실성과 질문 관련성을 일괄 평가한다.
        faithfulness_inputs = [
            {
                "user_input": record["user_input"],
                "response": record["response"],
                "retrieved_contexts": record["retrieved_contexts"],
            }
            for record in records
        ]
        relevancy_inputs = [
            {
                "user_input": record["user_input"],
                "response": record["response"],
            }
            for record in records
        ]
        faithfulness_scores = await faithfulness_metric.abatch_score(
            faithfulness_inputs
        )
        relevancy_scores = await relevancy_metric.abatch_score(
            relevancy_inputs
        )
        ragas_results[strategy] = [
            {
                **record,
                "faithfulness": faithfulness.value,
                "answer_relevancy": relevancy.value,
            }
            for record, faithfulness, relevancy in zip(
                records, faithfulness_scores, relevancy_scores
            )
        ]

    # 각 검색 전략의 RAGAS 평균 점수를 비교한다.
    ragas_summary = [
        {
            "strategy": strategy,
            "faithfulness": sum(
                result["faithfulness"] for result in results
            ) / len(results),
            "answer_relevancy": sum(
                result["answer_relevancy"] for result in results
            ) / len(results),
        }
        for strategy, results in ragas_results.items()
    ]
    print(json.dumps(ragas_summary, ensure_ascii=False, indent=2))
```

```markdown
### RAGAS 결과 해석

RAGAS 점수는 0에서 1 사이이며 높을수록 해당 기준을 잘 만족한다. 그러나 모든 프로젝트에 공통으로 적용할 절대 합격선은 없다. 같은 평가 데이터셋으로 설정 A와 설정 B를 비교하고, 점수가 낮은 문항을 직접 확인하는 데 사용한다.

#### Faithfulness

Faithfulness는 생성 답변을 검증 가능한 여러 주장으로 나눈 뒤, 각 주장이 `retrieved_contexts`로 뒷받침되는지 확인한다. 문맥이 지지하는 주장 비율이 높으면 점수가 높고, 검색 문서에 없는 수치·조건·예외를 답변이 추가하면 점수가 낮아진다.

예를 들어 검색 문서에는 ‘재택근무는 주 2회 가능’이라고만 되어 있는데 답변이 ‘팀장 승인 없이 주 2회 가능’이라고 말하면, ‘주 2회 가능’은 근거가 있지만 ‘팀장 승인 없이’는 근거가 없으므로 Faithfulness가 낮아질 수 있다.

Faithfulness가 높다고 정답이라는 뜻은 아니다. 검색 문서 자체가 질문과 무관하거나 오래된 문서여도, 답변이 그 문서에 충실하기만 하면 높은 점수가 나올 수 있다. 따라서 낮은 문항에서는 답변에 근거 없는 주장이 있는지 확인하고, 높은 문항에서도 검색된 문서가 올바른지는 Hit@k와 함께 확인한다.

#### Answer Relevancy

Answer Relevancy는 생성 답변만 보고 그 답변에 대응하는 질문을 여러 개 다시 만든다. 현재 설정에서는 기본값에 따라 질문 3개를 생성하고, 각 생성 질문의 embedding을 원래 `user_input`의 embedding과 비교한 평균 유사도로 점수를 계산한다. 답변이 회피성이라고 판단되면 점수가 낮아질 수 있다.

질문의 핵심에 직접 답하면 점수가 높고, 장황한 배경 설명만 하거나 다른 주제를 설명하면 낮아진다. 다만 이 지표는 `retrieved_contexts`나 `reference`와 사실을 대조하지 않으므로, 질문에 그럴듯하게 답한 잘못된 내용도 높은 점수를 받을 수 있다.

#### 두 점수를 함께 보는 법

| Faithfulness | Answer Relevancy | 해석 |
|---|---|---|
| 높음 | 높음 | 검색 근거를 사용해 질문에 직접 답한 경우 |
| 높음 | 낮음 | 근거에는 충실하지만 질문의 핵심을 벗어난 경우 |
| 낮음 | 높음 | 질문에는 직접 답했지만 근거 밖 내용을 만든 경우 |
| 낮음 | 낮음 | 근거성도 부족하고 질문과의 관련성도 낮은 경우 |

점수가 낮으면 `response`만 보지 말고 `user_input`과 `retrieved_contexts`를 함께 확인한다. 원인은 생성 모델뿐 아니라 앞 단계의 검색 실패일 수도 있다. 또한 한국어 표현과 어미 차이, 평가 LLM이 만든 역질문의 품질에 따라 Answer Relevancy가 달라질 수 있으므로 절대 점수보다 같은 문항·모델·평가 설정에서의 상대 비교에 집중한다.
```

```markdown
### RAGAS와 직접 정의한 Judge

RAGAS는 범용적인 근거성과 답변 관련성을 빠르게 평가할 때 유용하다. 직접 정의한 Judge는 규정 답변에 필요한 정확성, 완전성, 출처 정확성처럼 평가 기준을 자유롭게 정의할 수 있다.

| 구분 | RAGAS | 직접 정의한 Judge |
|---|---|---|
| 장점 | 공통 메트릭을 바로 사용 | 도메인 기준을 직접 정의 |
| 주의점 | 평가 모델과 언어에 따라 점수가 달라짐 | 채점 프롬프트와 기준을 검증해야 함 |

Judge에는 “좋은 답변인가?”처럼 모호한 질문보다 수치와 조건의 일치, 누락 여부, 검색 근거 밖의 주장 여부처럼 구체적인 기준을 준다.
```

```markdown
### 직접 정의한 LLM-as-Judge

규정 답변의 정확성, 완전성, 근거성, 출처 정확성을 각각 1~10점으로 평가하고 네 항목의 평균을 구한다. 1점은 기준을 거의 충족하지 못한 경우, 5점은 일부만 충족한 경우, 10점은 빠짐없이 충족한 경우다. 평균만 보지 않고 항목별 점수와 감점 이유를 함께 확인한다.
```

```python
# 직접 정의한 평가 항목과 각 점수의 범위를 정의한다.
class JudgeScore(BaseModel):
    correctness: int = Field(
        ge=1, le=10,
        description=(
            "기준 정답과 비교한 사실 정확성. 수치, 조건, 대상, 예외가 "
            "틀리면 감점한다. 대부분 틀리면 1점, 핵심은 맞지만 일부 오류가 "
            "있으면 5점, 모든 핵심 사실이 정확하면 10점이다."
        ),
    )
    completeness: int = Field(
        ge=1, le=10,
        description=(
            "질문과 기준 정답이 요구하는 핵심 내용을 빠짐없이 포함한 정도. "
            "핵심 내용을 거의 답하지 못하면 1점, 일부만 답하면 5점, 필요한 "
            "조건과 예외까지 모두 답하면 10점이다."
        ),
    )
    groundedness: int = Field(
        ge=1, le=10,
        description=(
            "모델 답변의 주장이 검색 문서로 뒷받침되는 정도. 주요 주장이 "
            "근거에 없거나 모순되면 1점, 일부만 근거가 있으면 5점, 모든 "
            "검증 가능한 주장이 검색 문서에 근거하면 10점이다."
        ),
    )
    citation_accuracy: int = Field(
        ge=1, le=10,
        description=(
            "인용한 파일이 실제 답변 내용을 지지하고 올바른 검색 출처인지 "
            "평가한다. 인용이 없거나 잘못되면 1점, 일부 인용만 적절하면 "
            "5점, 필요한 인용이 모두 정확하면 10점이다."
        ),
    )
    reason_of_correctness: str = Field(
        description="항목별 판단 근거와 감점된 구체적인 이유",
    )

judge_llm = llm.with_structured_output(JudgeScore)

# 기준 답변과 검색 문서를 바탕으로 생성 답변을 채점한다.
def judge_answer(item):
    contexts = "\n\n".join(item["retrieved_contexts"])
    result = judge_llm.invoke(
        "각 항목을 설명된 기준에 따라 1~10점의 정수로 평가하세요. "
        "reason에는 항목별 판단과 감점 이유를 구체적으로 작성하세요.\n\n"
        f"질문: {item['question']}\n기준 정답: {item['expected_answer']}\n"
        f"모델 답변: {item['response']}\n인용: {item['citations']}\n검색 문서:\n{contexts}"
    )
    score = result.model_dump()
    metric_names = [
        "correctness", "completeness",
        "groundedness", "citation_accuracy",
    ]
    score["average"] = round(
        sum(score[name] for name in metric_names) / len(metric_names),
        2,
    )
    return score

# 생성한 답변을 모두 채점하고 항목별 점수, 평균과 이유를 출력한다.
judge_results = [judge_answer(item) for item in generation_results]
for item, score in zip(generation_results, judge_results):
    print(
        f"[{item['strategy']}] [{item['id']}] 평균 {score['average']}/10 | "
        f"정확성 {score['correctness']} | 완전성 {score['completeness']} | "
        f"근거성 {score['groundedness']} | 출처 정확성 {score['citation_accuracy']}\n"
        f"{score['reason']}"
    )
```

```markdown
### 평가 비용과 실행 시간

Hit@k, Precision@k, Recall@k, MRR과 같은 검색 지표는 정답 문서와 검색 순위만으로 계산하며 LLM을 호출하지 않는다. 따라서 반복 실행하기 쉽다. 답변 생성, RAGAS, LLM-as-Judge는 질문 수와 메트릭 수에 따라 API 호출이 증가한다.

- 개발 중에는 전체 검색 평가를 자주 실행한다.
- 생성 평가는 난이도별 대표 문항부터 실행한다.
- RAGAS와 Judge는 필요한 메트릭만 선택한다.
- 비교 결과에는 품질 점수와 latency를 함께 기록한다.
```

```markdown
### 평가 적용 시점

| 시점 | 평가 방법 |
|---|---|
| 초기 개발 | 대표 질문을 직접 확인 |
| 검색 설정 변경 | Hit@k·Recall@k·MRR 회귀 평가 |
| 프롬프트 변경 | RAGAS 또는 Judge로 답변 비교 |
| 문서 변경 | 관련 질문의 정답과 검색 결과 재검증 |
| 서비스 운영 | 사용자 피드백과 실패 질문 수집 |

평가 데이터셋은 한 번 만들고 끝나는 파일이 아니다. 실패한 질문을 추가하고 문서 내용이 바뀌면 기대 정답과 정답 출처도 함께 갱신한다.
```

## 주말 간 복습 할 것 및 과제임!

```markdown
# 공공문서 RAG 검색 평가 실습

검색 평가에 집중할 수 있도록 PDF 로드, 메타데이터 정규화, 문서 분할 코드를 제공한다.
```

```markdown
### 실습 목표

- 제공된 전처리 결과를 사용해 벡터 저장소를 구성한다.
- File Hit@k, Page Hit@k, MRR을 계산한다.
- Similarity Search와 MMR의 검색 성능을 비교한다.
- 실패 문항의 검색 결과를 확인하고 원인을 분석한다.
```

```markdown
### 제공 데이터

- 검색 대상 문서: `data/public/*.pdf`
- 평가 데이터: `data/public/public_paragraph_golden_set.json`

golden set의 `question`은 검색 질의, `target_answer`는 기준 답변, `target_file_name`과 `target_page_no`는 정답 위치를 나타낸다.

`PyPDFLoader`가 만든 `source`에는 `data/public/파일명.pdf`처럼 경로가 포함될 수 있지만, golden set의 `target_file_name`에는 파일명만 들어 있다. 두 값을 바로 비교할 수 있도록 `Path(source).name`을 사용해 경로를 제거한다. 이 코드는 아래에 제공되어 있으므로 별도로 구현하지 않는다.

또한 `PyPDFLoader`의 `page`는 첫 페이지를 0으로 나타내지만 golden set의 `target_page_no`는 첫 페이지를 1로 나타낸다. 비교에 사용할 `page_no`에는 `page + 1`을 저장한다.
```

```python
import json
import time
from pathlib import Path

from dotenv import load_dotenv
from langchain_chroma import Chroma
from langchain_community.document_loaders import PyPDFLoader
from langchain_google_genai import GoogleGenerativeAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter

load_dotenv()

DATA_DIR = Path("data/public")
GOLDEN_SET_PATH = DATA_DIR / "public_paragraph_golden_set.json"
EMBEDDING_MODEL_NAME = "gemini-embedding-2"
```

```markdown
### Golden set 확인

평가 문항 수와 필드 구성을 확인한다.
```

```python
eval_cases = json.loads(GOLDEN_SET_PATH.read_text(encoding="utf-8"))

print(f"평가 문항: {len(eval_cases)}개")
print(json.dumps(eval_cases[:5], ensure_ascii=False, indent=2))
```

```markdown
### PDF 로드

모든 PDF를 페이지 단위로 불러온다. `PyPDFLoader.load()`의 결과에서 문서 하나는 PDF의 한 페이지에 해당한다.

불러온 페이지의 메타데이터는 다음과 같이 정리한다.

```python
{
    "source": "2024년 행정안전부 업무계획.pdf",
    "page": 0,
    "page_no": 1,
}
```

`source`는 golden set과 비교하기 위한 파일명이고, `page_no`는 golden set과 비교하기 위한 페이지 번호이다. 원래 로더가 제공한 `page`도 그대로 유지한다.
```

```python
page_documents = []

for pdf_path in sorted(DATA_DIR.glob("*.pdf")):
    pages = PyPDFLoader(pdf_path).load()

    for page in pages:
        # source에는 전체 경로 대신 golden set과 같은 파일명만 저장한다.
        page.metadata["source"] = Path(page.metadata["source"]).name
        # PyPDFLoader의 page는 0부터 시작하므로 사람이 보는 페이지 번호로 바꾼다.
        page.metadata["page_no"] = page.metadata["page"] + 1

    page_documents.extend(pages)

print(f"로드한 PDF: {len(list(DATA_DIR.glob('*.pdf')))}개")
print(f"로드한 페이지: {len(page_documents)}개")
print(page_documents[0].metadata)
```

```markdown
### 문서 분할

페이지 정보를 유지한 채 문서를 청크로 분할한다. 이 실습에서는 제공된 전처리 결과를 사용한다.
```

```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=700,
    chunk_overlap=100,
)

documents = splitter.split_documents(page_documents)

print(f"생성한 청크: {len(documents)}개")
```

```markdown
### 벡터 저장소와 검색기 구성

동일한 문서로 Similarity Search와 MMR을 구성한다. 이 실습에서는 별도 저장 경로 없이 `Chroma.from_documents()`로 벡터 저장소를 만들고, 검색기 딕셔너리의 키는 `Similarity`와 `MMR`을 사용한다.
```

```python
embeddings = GoogleGenerativeAIEmbeddings(model=EMBEDDING_MODEL_NAME)

# TODO: vectorstore와 retrievers를 구성하세요.
vectorstore = None
retrievers = {}

vectorstore = Chroma.from_documents(
    documents=documents,
    embedding=embeddings,
)

retrievers['Similarity'] = vectorstore.as_retriever(search_kwargs={"k": 5})
retrievers['MMR'] = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k" : 5,
        "fetch_k" : 20,
        "lambda_mult" : 0.5
    }
)
```

```markdown
### 평가 지표 구현

File Hit은 정답 파일의 검색 여부를, Page Hit은 정답 파일과 페이지가 함께 일치하는지를 확인한다. MRR은 최초로 등장한 정답 페이지 순위의 역수이다.

검색 결과 문서 `doc`과 golden set 문항 `case`는 다음 값으로 비교한다.

```python
파일 일치: doc.metadata["source"] == case["target_file_name"]
페이지 일치: doc.metadata["page_no"] == case["target_page_no"]
정답 페이지: 파일 일치 and 페이지 일치
```

예를 들어 정답 페이지가 검색 결과의 세 번째에 처음 등장하면 RR은 `1 / 3`이다. 상위 k개에 정답이 없으면 Hit은 0이고, 전체 검색 결과에 정답 페이지가 없으면 RR은 0.0이다.
```

```python
def file_hit_at_k(docs, case, k):
    for doc in docs[:k]:
        if doc.metadata["source"] == case["target_file_name"]:
            return 1
    return 0

def page_hit_at_k(docs, case, k):
    # TODO
    for doc in docs[:k]:
        if doc.metadata["source"] == case["target_file_name"] and doc.metadata["page_no"] == case["target_page_no"]:
            return 1
    return 0

def reciprocal_rank(docs, case):
    # TODO
    rank = 1
    for doc in docs:
        if doc.metadata["source"] == case["target_file_name"] and doc.metadata["page_no"] == case["target_page_no"]:
            return 1 / rank
        rank += 1
    return 0.0
```

```markdown
### 전체 평가

모든 질문을 검색하고 전략별 지표와 실행 시간을 기록한다. latency는 `retriever.invoke()` 한 번에 걸린 시간을 밀리초 단위로 측정한다. 실패 원인을 확인할 수 있도록 검색 결과의 순위, 파일명, 페이지 번호도 함께 저장한다.
```

```python
def evaluate_retriever(name, retriever, cases):
    rows = []

    for case in cases:
        docs = retriever.invoke(case["question"])
        rows.append({
            "name": name,
            "file_hit_1": file_hit_at_k(docs, case, 1),
            "page_hit_1": page_hit_at_k(docs, case, 1),
            "page_hit_3": page_hit_at_k(docs, case, 3),
            "page_hit_5": page_hit_at_k(docs, case, 5),
            "reciprocal_rank": reciprocal_rank(docs, case),
        })

    return rows

# TODO: 모든 검색 전략을 평가하세요.
evaluation_results = []
for name, retriever in retrievers.items():
    rows = evaluate_retriever(name, retriever, eval_cases)
    evaluation_results.extend(rows)

print(json.dumps(evaluation_results[:5], ensure_ascii=False, indent=2))
```

```markdown
### 결과 비교

검색 전략별 평균 점수를 비교한다. 전략마다 문항 수 `count`와 `file_hit_1`, `page_hit_1`, `page_hit_3`, `page_hit_5`, `reciprocal_rank`, `latency_ms`의 평균을 계산한다. 여러 문항의 `reciprocal_rank` 평균이 MRR이다.
```

```python
# TODO: 검색 전략별 결과를 요약하세요.
summary = []

def average(rows, key):
    return sum(row[key] for row in rows) / len(rows) if rows else 0.0

for name in retrievers:
    rows = [
        row for row in evaluation_results
        if row["name"] == name
    ]

    summary.append({
        "name": name,
        "count": len(rows),
        "file_hit_1": average(rows, "file_hit_1"),
        "page_hit_1": average(rows, "page_hit_1"),
        "page_hit_3": average(rows, "page_hit_3"),
        "page_hit_5": average(rows, "page_hit_5"),
        "reciprocal_rank": average(rows, "reciprocal_rank"),
    })
    
print(json.dumps(summary, ensure_ascii=False, indent=2))
```

```markdown
### 실패 문항 분석

전체 평가 결과에서 `page_hit_5 == 0`인 행을 확인한다. 각 실패 행에 저장한 `retrieved`를 정답 파일·페이지와 비교하여 같은 파일의 다른 페이지가 검색되었는지, 다른 파일만 검색되었는지 분석한다.
```

```python
# TODO: 실패 문항을 추출하세요.
failures = []

for row in evaluation_results:
    if row['page_hit_5'] == 0:
        failures.append(row)

print(json.dumps(failures, ensure_ascii=False, indent=2))
```

```markdown
### 결과 정리

다음 내용을 작성한다.

- 가장 성능이 좋은 검색 전략과 판단 근거
- File Hit과 Page Hit 결과가 다른 이유
- 실패 문항에서 확인한 검색 결과의 특징
- 성능을 개선하기 위해 변경해 볼 설정
```

## 우선은 이런데 연습해야 된다!! 아자아자!!