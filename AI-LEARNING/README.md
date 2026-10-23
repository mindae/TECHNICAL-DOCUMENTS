# Enterprise AI Engineering Handbook

**Applied GenAI, Agentic AI, RAG, LangGraph, MCP & Java Integration**  
**Status:** Living learning notes · **Last updated:** 2026-10-23  
**Audience:** Senior Java/Spring Boot developers and solution architects moving into enterprise AI integration.  
**Repository destination:** `TECHNICAL-DOCUMENTS/AI-LEARNING/README.md`

> **Purpose:** Keep a self-contained technical reference accessible from a public Git repository when ChatGPT is unavailable in a corporate environment. This handbook records concepts **and what we actually ran locally**, including commands, architecture, results, errors, and next steps. It is a learning reference, **not** a production security or compliance blueprint.
>
> **Privacy:** The names, IDs and HR policies below are fictional/test examples. **Never commit** real employee data, credentials, JWTs, API keys, confidential policies, company source code, customer documents or vector database contents to a public repository.

## Contents

1. [Learning goals and approach](#1-learning-goals-and-approach)
2. [Current progress tracker](#2-current-progress-tracker)
3. [Local environment and commands](#3-local-environment-and-commands)
4. [LLM fundamentals and Ollama](#4-llm-fundamentals-and-ollama)
5. [Python foundations for a Java developer](#5-python-foundations-for-a-java-developer)
6. [LLM tool calling and safe orchestration](#6-llm-tool-calling-and-safe-orchestration)
7. [LangGraph workflows](#7-langgraph-workflows)
8. [RAG: retrieval-augmented generation](#8-rag-retrieval-augmented-generation)
9. [Vector storage with ChromaDB](#9-vector-storage-with-chromadb)
10. [FastAPI and Spring Boot integration](#10-fastapi-and-spring-boot-integration)
11. [MCP: Model Context Protocol](#11-mcp-model-context-protocol)
12. [Enterprise security and guardrails](#12-enterprise-security-and-guardrails)
13. [Troubleshooting journal](#13-troubleshooting-journal)
14. [Working project file inventory](#14-working-project-file-inventory)
15. [Next lessons and backlog](#15-next-lessons-and-backlog)
16. [How we'll maintain this handbook](#16-how-well-maintain-this-handbook)
17. [Quick reference and further reading](#17-quick-reference-and-further-reading)

---

## 1. Learning goals and approach

**Goal:** Work as an applied enterprise AI / agentic AI engineer or solution architect: integrate LLMs, enterprise APIs and knowledge, rather than train foundation models. Build from an existing background in Java, Spring Boot, REST, Linux, Docker, Kubernetes and AWS.

**Preferred learning method:** Short concept → CLI first → runnable Python/Java code → execute and share results → debug → record architecture and lessons. Keep a parallel **15-minute daily Python exercise**. Learn broadly in the first pass; revisit selected components for enterprise-grade depth later.

**Design rule:** LLMs may *propose* tool calls or produce language, but application code owns authentication, authorization, validation, dependent IDs, workflow rules and side-effect controls.

### Local reference architecture

```text
                  React / curl / external client
                             |
                             v
                  Spring Boot API :9085
                             | HTTP/JSON
                             v
                     FastAPI :8000
                             |
                         LangGraph
                          /     \
                  ChromaDB       Ollama
                 (vector DB)  (chat + embed)

Alternative tool integration:
    Ollama -> Python MCP client -> MCP server -> Spring Boot HR APIs
                        stdio OR Streamable HTTP
```

These are **learning examples**, not a requirement that every solution must include every layer. A Java-only approach using Spring AI may suit some teams.

---

## 2. Current progress tracker

| Area | Progress | Proven by |
|---|---|---|
| Python basics | ✅ First pass | Dictionaries, lists, functions, exceptions, sorting, typing, async |
| Local Ollama | ✅ | `qwen3:1.7b` via CLI, HTTP, Python |
| Tool calling | ✅ | Employee/department REST tools |
| LangGraph | ✅ | State, nodes, conditional edges, LLM router |
| RAG | ✅ | Embeddings, cosine similarity, Top-K, grounding |
| ChromaDB | ✅ | Persisted chunks across process runs |
| LangGraph + RAG | ✅ | Retrieval/answer/fallback nodes |
| FastAPI | ✅ | `/health` and `/api/v1/ai/ask` HTTP 200 |
| Java ↔ Python | ✅ | Spring Boot RestClient → FastAPI HTTP 200 |
| MCP in-memory | ✅ | Discovered and called two HR tools |
| MCP stdio | ✅ | Separate client/server processes |
| MCP Streamable HTTP | ✅ | Client connected to server on port 8001 |
| Ollama + MCP | ✅ | LLM selected `lookup_employee(101)` |
| Multi-step MCP agent | ✅ | Employee 101 → department 100 → final summary |
| Authorization basics | 🟡 Introduced | Example policy; hands-on execution not yet confirmed |
| OAuth2/JWT across AI services | ⬜ Planned | — |
| Prompt-injection defenses | ⬜ Planned | — |
| Evaluations, tracing, monitoring | ⬜ Planned | — |
| Docker/Kubernetes/AWS production deployment | ⬜ Planned | — |

**Most recent verified milestone (2026-10-23):** MCP Streamable HTTP connected; controlled three-round Ollama MCP agent selected employee 101, then department 100, then correctly summarized both records.

---

## 3. Local environment and commands

### Machine and versions observed

- macOS / Apple Silicon; shell `zsh`; terminal and IntelliJ IDEA.
- Project: `~/IdeaProjects/enterprise-ai`.
- Dedicated Python environment: `.venv` with **Python 3.12.15**. Outside `.venv`, `python` / `python3` may resolve differently (including Homebrew Python 3.12 or 3.14).
- Local Ollama: `http://localhost:11434`; models: `qwen3:1.7b` for chat, `nomic-embed-text` for embeddings.
- Recorded local package versions include: `ollama` 0.6.3; LangGraph 1.2.16; FastAPI 0.159.1; Uvicorn 0.54.0. Versions change; check your machine.
- `employee-service`: `http://localhost:8081/api/v1/employees`; `department-service`: `http://localhost:8082/api/v1/departments`.
- Spring Boot AI bridge on `:9085`; Python FastAPI on `:8000`; HTTP MCP server on `:8001`.
- Test JSON employee: `{"id":101,"name":"John","department":{"id":100,"name":"Finance"}}`.
- Test JSON department: `{"id":100,"name":"Finance"}`.
- **Important:** Current HR test APIs return a single example object from their list endpoint. `/api/v1/departments/100` was not implemented. Python tool wrappers check the returned object's ID to emulate lookup-by-ID.

### Everyday startup checklist

```bash
cd ~/IdeaProjects/enterprise-ai
source .venv/bin/activate      # shell now uses project Python
which python
python --version
python -m pip show ollama langgraph chromadb mcp fastapi uvicorn
ollama list
curl -s http://localhost:11434/api/tags
```

If `.venv` is not activated, bypass shell lookup:

```bash
.venv/bin/python your_script.py
```

Create a new environment only if needed:

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install ollama langgraph chromadb fastapi uvicorn mcp httpx pydantic
python -m pip freeze > requirements.txt
```

**CLI explanations:** `-m` runs a Python module; `pip show` checks package installation; `freeze` lists pinned installed versions; `source` activates the shell environment. Keep version-pinned `requirements.txt` once the working environment is stable.

### Ollama checks

```bash
ollama list
ollama pull qwen3:1.7b
ollama pull nomic-embed-text
curl -s http://localhost:11434/api/tags
```

For local experiments, `think=False` and optionally `options={"temperature": 0}` were used with `ollama.chat()` for quicker and more repeatable responses. Temperature 0 **does not guarantee deterministic or correct outputs**.

### Git and editor hygiene

Keep in `.gitignore`:

```gitignore
.venv/
__pycache__/
*.pyc
.env
.env.*
!.env.example
chroma_data/
.idea/
.DS_Store
```

IntelliJ's Python SDK/interpreter should point to `~/IdeaProjects/enterprise-ai/.venv/bin/python`; configuring the IDE does not always activate it in the integrated terminal. Check `which python` in each new terminal.

---

## 4. LLM fundamentals and Ollama

- **LLM (large language model):** Produces generated text and may request tools; it does not automatically read internal company systems or execute business functions.
- **Inference:** Running the model for responses, not training it.
- **Context window:** The text/tool history made available to the model; do not assume everything in the repository is in the context.
- **Prompt grounding:** Provide retrieved facts; still validate model claims.
- **Temperature:** Sampling parameter affecting response variation; lower often helps reproducibility but not correctness.
- **Structured output:** JSON Schema/Pydantic checks response *shape*, not whether the chosen route or extracted ID is semantically correct.

Minimal local Ollama Python:

```python
from ollama import chat

response = chat(
    model="qwen3:1.7b",
    think=False,
    messages=[{"role": "user", "content": "Explain RAG briefly."}],
)
print(response.message.content)
```

**Observed model limitation:** Qwen3 1.7B sometimes invented details or requested the wrong dependent department ID. Never assume a confident-sounding answer is factually grounded.

---

## 5. Python foundations for a Java developer

### Core syntax and mental models

| Python | Java analogy / observation |
|---|---|
| `dict` | Similar to `Map<String, Object>`; nesting allowed |
| `list[dict]` | List of objects/records |
| `tuple` | Immutable ordered grouping; can form hashable composite keys |
| `set` | Unique items and fast membership checks |
| `def f(x: int) -> str:` | Type **hint**, not automatic runtime enforcement |
| Pydantic `BaseModel` | DTO plus runtime parsing/validation; coercion by default |
| `TypedDict` | Typed shape for dictionaries; not runtime validation by itself |
| `employee.get("department")` | Like `Map.get`; missing key → `None` |
| `employee["department"]` | Missing key raises `KeyError` |
| `lambda e: e["exp"]` | Similar to a Java comparator/key extractor |
| `async def`, `await` | Python coroutine/event-loop programming; conceptually related to async Java, not identical |
| `if __name__ == "__main__":` | Run test code only when executed directly, not upon import |

### Nested JSON validation (Day 7)

```python
def get_department_id(employee: dict) -> int | None:
    department = employee.get("department")
    if not isinstance(department, dict):
        return None
    department_id = department.get("id")
    return department_id if type(department_id) is int else None
```

Why `type(x) is int`? In Python, `isinstance(True, int)` is `True`; for IDs we intentionally reject booleans. `"100"` is a string, so strict validation returns `None` rather than silently converting it to 100.

### Sorting and comprehensions (Days 9–13)

```python
employees = [{"name": "amit", "exp": 5}, {"name": "neha", "exp": 12}]
employees.sort(key=lambda emp: emp["exp"], reverse=True)
source_ids = [doc["id"] for doc in documents]
allowed_tool_names = {tool.name for tool in available.tools}
```

**Mistake fixed:** `employees["exp"]` raises `TypeError` because `employees` is a list; inside `lambda`, use the current dictionary `emp["exp"]`.

```python
for doc_id, score in zip(["HR-001", "HR-002"], [0.8, 0.5]):
    print(doc_id, score)

cache = {("lookup_employee", 101): {"id": 101}}
```

`zip()` pairs corresponding elements and stops at the shortest iterable; tuple keys are useful for read-only per-request caches.

### Pydantic structured output

```python
from typing import Literal
from pydantic import BaseModel

class RouteDecision(BaseModel):
    route: Literal["employee", "department", "employee_department", "general"]
    record_id: int | None

# Ollama: format=RouteDecision.model_json_schema()
# Then: RouteDecision.model_validate_json(response.message.content)
```

**Pitfall:** Even schema-valid JSON can contain an incorrect route or ID. The model once classified `Show employee 999` as `employee_department`.

### Python practice progress

| Day | Focus | Day | Focus |
|---|---|---|---|
| 1 | Dicts, strings, `if` | 11 | Tuples and sorting |
| 2 | Lists of dicts, loops | 12 | `zip()` |
| 3 | Functions and return values | 13 | Sets and membership |
| 4 | Imports and exceptions | 14 | `TypedDict` and state updates |
| 5 | JSON and `dict.get()` | 15 | `__name__` guard |
| 6 | Type hints, control flow, sets | 16 | HTTP decorators vs functions |
| 7 | Missing/optional fields | 17 | `async` / `await` |
| 8 | Pydantic runtime validation | 18 | Set comprehension |
| 9 | `sort`, `lambda` | 19 | `next()` / generators |
| 10 | List indexing and empty lists | 20 | Tuple keys for caching |
|  |  | 21 | Authorization function (introduced; verification pending) |

---

## 6. LLM tool calling and safe orchestration

### First REST-backed tools

`employee_tools.py` defines `get_employee(employee_id)` and `get_department(department_id)`. Each calls a Spring Boot API through `httpx`, validates the returned ID, and returns data or an `{"error": ...}` dictionary. HTTP failures and invalid JSON need handling.

```python
# Simplified concept, not complete production error handling
import httpx

def get_employee(employee_id: int) -> dict:
    response = httpx.get("http://localhost:8081/api/v1/employees", timeout=5)
    response.raise_for_status()
    employee = response.json()
    return employee if employee.get("id") == employee_id else {"error": "Employee not found"}
```

**Tool-calling sequence:** Model requests a named tool with arguments → application validates → application invokes tool → result is sent back as a tool message → model may answer or request another tool.

### Lessons from actual failures

- The model once requested `get_department(123)` without reading the employee's `department.id`.
- The model sometimes requested `get_employee(101)` twice in one response.
- In another loop, it requested `lookup_employee(101)` and `lookup_department(101)` in the **same round**. The application rejected department 101 because employee 101 belongs to department 100.
- After receiving that error, the model falsely claimed department 100 had failed, despite that call never occurring.

**Safety rules:** Allowlist tool names; validate type/shape and business invariants; derive dependent IDs from verified tool outputs; match tool responses to calls; restrict tools by workflow state; limit rounds; cache only appropriate read-only results; check authorization at the business service. A failed tool request must not be represented as a successfully executed backend call.

### Verified controlled workflow

```text
Round 1: lookup_employee(101) -> {id:101, department:{id:100,...}}
Round 2: lookup_department(100) -> {id:100, name:"Finance"}
Round 3: no tools exposed -> summary: Employee 101 is John; department Finance.
```

**Design principle:** LLM chooses *within* available state-appropriate tools; application controls the allowed transitions. Fully autonomous looping is not inherently better than a fixed workflow.

---

## 7. LangGraph workflows

LangGraph coordinates application state and nodes. It can be entirely deterministic, AI-driven, or hybrid. Core primitives: `StateGraph`, `START`, `END`, nodes, edges, conditional routing and typed state.

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class HRState(TypedDict, total=False):
    employee_id: int
    employee: dict
    department_id: int
    department: dict
    error: str

def employee_node(state: HRState) -> dict:
    employee = get_employee(state["employee_id"])
    if "error" in employee:
        return {"error": employee["error"]}
    return {"employee": employee, "department_id": employee["department"]["id"]}

def route_after_employee(state: HRState) -> str:
    return "end" if "error" in state else "department"

builder = StateGraph(HRState)
builder.add_node("employee", employee_node)
builder.add_node("department", department_node)  # defined separately
builder.add_edge(START, "employee")
builder.add_conditional_edges("employee", route_after_employee,
                              {"department": "department", "end": END})
builder.add_edge("department", END)
graph = builder.compile()
```

Nodes return **partial state updates**. `TypedDict` describes the expected state shape and doesn't automatically validate runtime dictionaries.

### Verified cases

- Employee 101 → employee node → department 100 → final state populated.
- Employee 999 → error → skip department.
- Employee service unavailable → controlled error and no dependent call.
- `llm_node` used `ollama.chat()` to formulate a final answer based on the retrieved employee/department records.
- LLM router used Pydantic `RouteDecision` for `employee`, `department`, `employee_department`, or `general`. Structured output worked on four initial samples, but routing sometimes misclassified an employee-only query. The route must still be validated.

---

## 8. RAG: retrieval-augmented generation

**Problem:** A chat model may not know private or updated company policy. **RAG** retrieves relevant passages from external information and supplies them to the model as context for an answer; it does not train/retrain the model.

### Two pipelines

```text
INDEXING: document -> chunks -> embedding model -> vector store
QUERY: question -> query embedding -> nearest chunks -> LLM -> grounded answer
```

### Embeddings and cosine

An embedding represents text as a numerical vector. We used Ollama's `nomic-embed-text` model for documents and questions; `qwen3:1.7b` was the generation model.

Cosine similarity:

```text
similarity(A,B) = dot(A,B) / (||A|| * ||B||)
cosine distance = 1 - cosine similarity
```

Higher similarity / lower cosine distance suggests closer vector alignment. **It is not a probability that the result is accurate or that the chunk answers the question.**

### Actual tests

| Exercise | Observation |
|---|---|
| Four-policy similarity search | Earned leave question ranked HR-001 first, similarity **0.8069** |
| Maternity leave absent from sources | HR-001 still ranked first, similarity **0.6021**; LLM correctly said it didn't know |
| Top-2 earned leave retrieval | HR-001 0.8069; FIN-001 0.5752; answered 30 days |
| Top-2 maternity leave retrieval | HR-001 0.6021; IT-001 0.5113; declined to invent |
| Whole handbook embedding | Similarity 0.7496 |
| Relevant HR-LEAVE chunk | Similarity 0.7574; chunk-level retrieval identified the narrower passage |

**Lesson:** Even unrelated documents have non-zero similarity. Top-K retrieves the nearest K matches, not necessarily K documents that answer the question. Use grounding and (later) evidence relevance/evaluation checks.

### Chunking and Top-K

- **Chunking:** Divide larger documents into smaller, meaningful passages. We started with one policy sentence/section per chunk.
- **Top-K:** Retrieve up to K most similar chunks, e.g. `results[:2]`.
- **Tradeoff:** Tiny chunks may lose context; huge chunks add noise and token costs. Later: token-based chunking, overlap, metadata, hybrid retrieval, rerankers, relevance thresholds.
- **Source IDs:** A retrieved document is not necessarily a document supporting the final claim; LLM-generated citations can be omitted or invented. Verify citation IDs and support in application logic.

### Minimal Ollama pattern

```python
from ollama import embed, chat

vector = embed(model="nomic-embed-text", input="earned leave policy").embeddings[0]
response = chat(
    model="qwen3:1.7b", think=False,
    messages=[
        {"role": "system", "content": "Answer only from supplied context. If absent, say you don't know."},
        {"role": "user", "content": "Context: Employees may carry forward 30 days of earned leave.\nQuestion: What is the limit?"},
    ],
)
print(response.message.content)
```

**Fictional sample policies:** earned-leave limit 30 days; work from home needs manager approval; expense submission within 15 days; password change every 90 days. These are demo records, **not actual employer policy**.

---

## 9. Vector storage with ChromaDB

Why a vector database? Recomputing every document embedding on each user query is inefficient. Persist document vectors once, then embed/search each new question.

### Ingestion (`rag_chroma_ingest.py`)

```python
import chromadb
from ollama import embed

client = chromadb.PersistentClient(path="./chroma_data")
collection = client.get_or_create_collection(
    name="hr_policies",
    configuration={"hnsw": {"space": "cosine"}},
)
text = "Employees may carry forward 30 days of earned leave."
vector = embed(model="nomic-embed-text", input=text).embeddings[0]
collection.upsert(ids=["HR-LEAVE"], documents=[text], embeddings=[vector],
                  metadatas=[{"category": "HR"}])
print(collection.count())
```

`upsert` inserts or updates the same document ID; repeat ingestion did **not** duplicate records. Three distinct sample chunks remained at count 3.

### Query (`rag_chroma_query.py`)

```python
client = chromadb.PersistentClient(path="./chroma_data")
collection = client.get_collection(name="hr_policies")
qvec = embed(model="nomic-embed-text", input="How many earned leave days?").embeddings[0]
results = collection.query(query_embeddings=[qvec], n_results=2,
                           include=["documents", "distances"])
for doc_id, text, dist in zip(results["ids"][0],
                              results["documents"][0], results["distances"][0]):
    print(doc_id, dist, text)
```

### Results verified

- HR-LEAVE cosine distance **0.1953** versus HR-TRAVEL **0.4216** for one earned-leave query.
- Second execution of the **query-only** script returned identical matches, confirming persistence.
- Combined ChromaDB + Ollama + LangGraph answered earned-leave and WFH questions with supporting IDs, and abstained on unsupported maternity-leave policy.

**Comparison:** ChromaDB is convenient for local learning. PostgreSQL + pgvector allows relational and vector querying together. Managed offerings (e.g., Amazon OpenSearch) can suit scaled or hybrid search workloads. Choice depends on operations, indexing, authorization filtering, workload and cost.

---

## 10. FastAPI and Spring Boot integration

### Python side — FastAPI (`ai_api.py`)

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field
from rag_langgraph_demo import graph

app = FastAPI(title="Enterprise AI Service")

class AskRequest(BaseModel):
    question: str = Field(min_length=1)

class AskResponse(BaseModel):
    answer: str
    retrieved_sources: list[str]

@app.get("/health")
def health():
    return {"status": "UP"}

@app.post("/api/v1/ai/ask", response_model=AskResponse)
def ask(request: AskRequest):
    result = graph.invoke({"question": request.question})
    return AskResponse(
        answer=result["answer"],
        retrieved_sources=[d["id"] for d in result.get("documents", [])],
    )
```

Guard test loops in imported modules:

```python
if __name__ == "__main__":
    # Only runs when executed directly, not upon import into FastAPI.
    ...
```

Run and test:

```bash
python -m uvicorn ai_api:app --host 127.0.0.1 --port 8000
curl -s http://127.0.0.1:8000/health
curl -s -X POST http://127.0.0.1:8000/api/v1/ai/ask \
  -H 'Content-Type: application/json' \
  -d '{"question":"How many earned leave days can I carry forward?"}'
```

Docs: `http://127.0.0.1:8000/docs`. Verified `200 OK`; JSON contained a grounded 30-day answer and retrieved source IDs `HR-LEAVE`, `HR-TRAVEL`.

### Java side — Spring Boot RestClient bridge

Sample request and response records:

```java
public record AiRequest(String question) {}
public record AiResponse(String answer, java.util.List<String> retrieved_sources) {}
```

```java
@Service
public class PythonAiClient {
    private final RestClient client;

    public PythonAiClient(RestClient.Builder builder) {
        this.client = builder
            .baseUrl("http://127.0.0.1:8000")
            .requestFactory(new SimpleClientHttpRequestFactory())
            .build();
    }

    public AiResponse ask(String question) {
        return client.post()
            .uri("/api/v1/ai/ask")
            .contentType(MediaType.APPLICATION_JSON)
            .body(new AiRequest(question))
            .retrieve()
            .body(AiResponse.class);
    }
}
```

Controller (`:9085`):

```java
@RestController
@RequestMapping("/api/v1/ai")
public class AiController {
    private final PythonAiClient python;
    public AiController(PythonAiClient python) { this.python = python; }

    @PostMapping("/ask")
    public AiResponse ask(@RequestBody AiRequest request) {
        return python.ask(request.question());
    }
}
```

Java snippets omit package/import declarations. `SimpleClientHttpRequestFactory` is a **verified local workaround**, not yet a researched production client recommendation. Configure timeout, URL, authentication and error mapping before deploying. When applications run in different containers, `127.0.0.1` refers to each container, not the host or other service.

Verified end-to-end test:

```bash
curl -i -X POST http://localhost:9085/api/v1/ai/ask \
  -H 'Content-Type: application/json' \
  -d '{"question":"Is manager approval required for work from home?"}'
# HTTP/1.1 200; answer cites [HR-WFH]
```

---

## 11. MCP: Model Context Protocol

### What MCP adds

MCP standardizes how AI hosts/clients discover and invoke tools, access resources and use prompt templates. MCP is **not** a replacement for REST, OAuth, LangGraph or the LLM. A server can wrap existing business REST APIs.

```text
AI host (Ollama/LangGraph application)
  └─ MCP client ── MCP transport ── MCP server
                                         ├─ lookup_employee -> Java :8081
                                         └─ lookup_department -> Java :8082
```

**Concepts:** Host owns the AI experience; client maintains a connection to one server; server exposes tools, resources and prompts. Tools have names, descriptions, argument schemas and result content.

### MCP Python SDK and server

The experiments used the locally installed MCP Python SDK exposing `MCPServer` and `Client`; check your installed SDK before following tutorials for older `FastMCP` APIs.

```python
# hr_mcp_server.py
from mcp.server import MCPServer
from employee_tools import get_employee, get_department

mcp = MCPServer("HR Tools")

@mcp.tool()
def lookup_employee(employee_id: int) -> dict:
    """Look up an employee by numeric ID."""
    return get_employee(employee_id)

@mcp.tool()
def lookup_department(department_id: int) -> dict:
    """Look up a department by numeric ID."""
    return get_department(department_id)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Connection modes we verified

| Mode | Client pattern | Meaning |
|---|---|---|
| In-memory | `async with Client(mcp)` | Same Python process; useful for tests |
| stdio | `Client(StdioServerParameters(command=sys.executable, args=[server_path]))` | Client starts MCP subprocess |
| Streamable HTTP | `Client("http://127.0.0.1:8001/mcp")` | Independent server, network transport |

HTTP server startup (in a **separate copy** of the server file):

```python
if __name__ == "__main__":
    mcp.run(transport="streamable-http", host="127.0.0.1", port=8001)
```

A browser `GET /mcp` returned HTTP **400** and `{}`; that's not evidence of a broken server. The real MCP client completed handshake, negotiated protocol **2026-07-28**, discovered both tools, and returned correct employee/department data.

### Read MCP results correctly

```python
import json
from mcp.types import TextContent

if result.is_error:
    raise RuntimeError("MCP tool failed")
if result.structured_content is not None:
    data = result.structured_content
else:
    text_blocks = [b.text for b in result.content if isinstance(b, TextContent)]
    data = json.loads(text_blocks[0])  # For our known one-JSON-object tools only
```

**Observed:** `structured_content` was `None`, while `content` held a `TextContent` block containing JSON text. This is valid; handle both paths. Not all MCP text is JSON.

### Ollama + MCP adapter

Discovery:

```python
available = await client.list_tools()
ollama_tools = [
    {"type": "function", "function": {
        "name": t.name,
        "description": t.description or "",
        "parameters": t.input_schema,
    }}
    for t in available.tools
]
```

Ollama picks the tool; **application** allowlists and validates name + args, executes `await client.call_tool(name, args)`, appends the tool response, and calls the model again. It does not magically execute tools itself.

### Sequential and model-driven MCP outcomes

1. One-round `ollama_mcp_demo.py`: Ollama selected `lookup_employee(101)`, then answered Finance.
2. Controlled `ollama_mcp_sequential_demo.py`: Application extracted department 100 from employee and called department via MCP, then asked Ollama to summarize.
3. First agent loop: Model called employee 101 **and** department 101 in round 1. The business invariant rejected department 101. Model then inaccurately described the error.
4. Improved agent loop exposed **only employee tool** until employee fetched, **only department tool** until department fetched, and **no tools** once both records arrived. Verified three rounds produced the correct final answer.

**Transport hygiene:** On stdio server processes, write protocol messages to stdout only; use stderr for logs. Production remote HTTP MCP requires transport security, authentication, authorization and careful network exposure.

---

## 12. Enterprise security and guardrails

**Introduced 2026-10-23; hands-on authorization lab not yet verified.** An MCP/LLM tool name being valid does **not** mean an authenticated user is permitted to use it.

### Four initial risk areas

1. **Unauthorized tools/data:** Enforce authenticated identity and per-record authorization in resource-owning services, not only AI prompts.
2. **Prompt injection:** Treat retrieved documents and tool outputs as untrusted data, not instructions overriding business policy.
3. **Excessive agency:** Restrict tools, arguments, execution steps, side effects and approval for impactful operations.
4. **Sensitive information disclosure:** Avoid logging confidential documents/questions and secrets. Minimize data in model prompts, tool results and traces.

### Intro authorization lab (`day21_tool_authorization.py`)

```python
def can_access_employee(user_id: int, roles: list[str], requested_employee_id: int) -> bool:
    if "HR_ADMIN" in roles:
        return True
    return user_id == requested_employee_id

for user_id, roles, requested_id in [
    (101, ["EMPLOYEE"], 101),
    (101, ["EMPLOYEE"], 102),
    (200, ["HR_ADMIN"], 101),
]:
    print(user_id, requested_id,
          can_access_employee(user_id, roles, requested_id))
```

Expected `True`, `False`, `True`. **Do not trust `user_id` or `roles` supplied by the model or frontend:** derive identity from verified authentication and enforce permissions downstream. The example illustrates the rule; it is not a complete authorization design.

### Upcoming production controls

OAuth2/OIDC and JWT propagation; per-tool authorization; prompt-injection test cases; retrieval filters; allowlist/JSON-schema validation; timeouts and retries; read/write separation; audit events; observability without leaking sensitive content; evaluation suites and regression testing.

---

## 13. Troubleshooting journal

| Problem | Root cause / evidence | Fix or lesson |
|---|---|---|
| `ModuleNotFoundError: langgraph / uvicorn / mcp` | Terminal using global/Homebrew Python instead of project `.venv` | `source .venv/bin/activate`, verify `which python`; or `.venv/bin/python script.py` |
| IntelliJ terminal package missing | SDK setting and shell activation are separate | Existing interpreter is `.venv/bin/python`; terminal may still need activation |
| Python 3.9 rejected `int | str` | Union `|` syntax needs modern Python | Use Python 3.12 project venv |
| `KeyError: 'department'` | Missing dictionary key, not key with `None` | Use `.get()` and validate nested structure |
| `TypeError: list indices must be integers ...` | `employees["exp"]` on list | `key=lambda emp: emp["exp"]` |
| `NameError: top_document` | Switched to Top-K `top_documents` but left old print statements | Iterate list and update variable names |
| Employee service unavailable | Spring Boot endpoint not running/reachable on first test | Check port and API; graph should return controlled error |
| Model invented department ID | Requested 123 or 101 rather than actual 100 | Validate IDs from employee API and restrict exposed tools by state |
| MCP tool result printed `None` | Only inspected `structured_content` | Parse JSON in `TextContent` from `content` |
| Browser `GET /mcp` returned 400 | MCP endpoint expects protocol handshake/headers | Use MCP client, not browser GET |
| FastAPI reported 422 `body` missing from Spring RestClient | Java interceptor showed correct 63-byte JSON before transmission; Python direct curl worked | Changing to `SimpleClientHttpRequestFactory` yielded HTTP 200; exact underlying transport discrepancy not yet diagnosed |
| Java exposed HTTP 500 for Python's 422 | Downstream exception propagated unhandled | Later: translate downstream failures into meaningful API responses |
| LLM omitted a requested citation | Prompt-following not guaranteed | Prompt correction helped; production requires app verification |
| Maternity-leave query retrieved irrelevant docs | Vector search always has nearest matches | LLM correctly abstained; later implement relevance/evidence evaluation |

Useful diagnostic commands:

```bash
which python
python -m pip show mcp uvicorn langgraph
curl -i http://127.0.0.1:8000/health
curl -i -X POST http://127.0.0.1:8000/api/v1/ai/ask \
  -H 'Content-Type: application/json' \
  -d '{"question":"Is manager approval required for work from home?"}'
```

**Debugging strategy:** Test each boundary independently: shell/interpreter → Python package → upstream REST → MCP result → model decision → vector retrieval → FastAPI → Spring HTTP client. A 500 from Spring may originate in downstream request/response handling.

---

## 14. Working project file inventory

Files created or discussed in the `enterprise-ai` learning project (the Markdown records them; **these source files are not included in this handbook download**):

| File | Purpose |
|---|---|
| `main.py` | First Ollama Python chat |
| `employee_tools.py`, `test_tool.py` | REST-backed HR tools and test calls |
| `tool_calling_demo.py`, `agent_loop_demo.py` | First LLM tool calls and loops |
| `sequential_tools_demo.py` | Application-controlled dependent tool call |
| `langgraph_demo.py` | LangGraph nodes, shared state, conditional edges |
| `langgraph_llm_demo.py` | Ollama as a graph node |
| `langgraph_router_demo.py` | Pydantic route decisions and graph routing |
| `rag_search_demo.py` | In-memory embeddings + cosine ranking |
| `rag_llm_demo.py` | Top-K retrieved context + LLM |
| `rag_chunking_demo.py` | Whole-document vs chunk similarity |
| `rag_chroma_ingest.py` | Persist test vectors in ChromaDB |
| `rag_chroma_query.py` | Query persistent vectors |
| `rag_chroma_llm_demo.py` | Persistent retrieval + Ollama answer |
| `rag_langgraph_demo.py` | Retrieval/answer/fallback LangGraph nodes |
| `ai_api.py` | FastAPI `/health` and `/api/v1/ai/ask` |
| `hr_mcp_server.py`, `hr_mcp_client.py` | MCP tools and in-memory connection |
| `hr_mcp_stdio_client.py` | MCP separate-process stdio transport |
| `ollama_mcp_demo.py` | LLM selects an MCP-discovered tool |
| `ollama_mcp_sequential_demo.py` | Safe employee → department execution |
| `ollama_mcp_agent_loop.py` | Multi-round tool use with stage-based exposure |
| `hr_mcp_http_server.py`, `hr_mcp_http_client.py` | Streamable HTTP MCP |
| `day7_dict_validation.py`, `day9_sort_lambda.py` | Python validation and sorting exercises |
| `day21_tool_authorization.py` | Planned/introduced authorization exercise |
| `requirements.txt` | Freeze project dependencies |

**Suggested future repository structure:** Keep this handbook as `AI-LEARNING/README.md`. If we later want to publish sanitized runnable code, add `AI-LEARNING/examples/` with independent examples plus tests and no private data. Avoid copying the entire local project indiscriminately into a public repo.

---

## 15. Next lessons and backlog

**Current position:** MCP in-memory, stdio, HTTP and controlled multi-step agent complete; first security lab introduced.

Recommended next first-pass sequence:

1. **Security lab:** Run authorization test; apply identity/permission checks around tool execution; distinguish caller identity vs tool input.
2. **OAuth2/JWT architecture:** Spring Boot resource server, FastAPI validation, MCP HTTP authorization, service-to-service identity; trusted claims and token audience.
3. **Prompt injection:** Safe handling of document/tool text; adversarial examples and boundaries.
4. **Evaluations:** Build a small reproducible dataset for route selection, ID extraction, retrieval relevance, answer grounding, citations and tool-call correctness.
5. **Observability:** Trace request IDs across Java → FastAPI → LangGraph → MCP → upstream HR services; metrics, latency, cost, logs without sensitive payloads.
6. **Production reliability:** Timeouts, retries, circuit breaking, cancellation, side-effect idempotency, rate limits, queueing, fallback policy.
7. **Container/cloud deployment:** Dockerize FastAPI/MCP, network/service discovery, AWS deployment choices, secrets/config, scalable Ollama or hosted models.
8. **Second pass (selected):** RAG ingestion pipelines, chunking strategies, hybrid retrieval/reranking, authorization-aware search, pgvector/OpenSearch, agent policy engines.

### Project milestones worth demonstrating in interviews

- Explained difference between *LLM-proposed* tool calls and *application-authorized* actions.
- Built deterministic vs agent-driven LangGraph routes with validation.
- Implemented semantic search and persistent RAG; demonstrated grounded answer vs abstention.
- Exposed AI workflow through FastAPI and called it from Spring Boot.
- Demonstrated MCP discovery/invocation over three connection methods.
- Diagnosed a cross-language HTTP 422/500 integration problem by isolating serialization vs transport.

---

## 16. How we'll maintain this handbook

Whenever we cover a new enterprise AI topic, simply say:

> **Update our AI learning Markdown notes.**

The maintenance workflow:

1. Preserve all earlier lessons, working commands and verified results.
2. Add the new lesson in the appropriate section; update table of contents and progress tracker.
3. Record **what actually ran** separately from example/expected output.
4. Note mistakes, error messages, diagnoses and working fixes—don't silently erase earlier failures.
5. Keep code examples small and runnable; explain unfamiliar Python syntax and CLI flags on first use.
6. Add a date to new updates and log any changed SDK/API conventions.
7. Keep repository content generic and public-safe; use placeholders for secrets and confidential data.
8. Save updated `AI-LEARNING/README.md`, then commit and push via Git/IntelliJ.

### Suggested update log

| Date | Update |
|---|---|
| 2026-10-14 to 2026-10-19 | Python basics; local Ollama and HR REST-backed tools; LangGraph fundamentals |
| 2026-10-20 to 2026-10-22 | LLM routing; RAG; ChromaDB; LangGraph RAG; FastAPI and Spring Boot integration |
| 2026-10-23 | MCP in-memory, stdio and Streamable HTTP; Ollama MCP tools and controlled agent loop; security basics introduced |

### Suggested Git operations (macOS/Windows Git Bash)

From the root of a local clone of `TECHNICAL-DOCUMENTS`:

```bash
mkdir -p AI-LEARNING
# Copy the provided README.md into AI-LEARNING/README.md

git status
git add AI-LEARNING/README.md
git diff --cached --stat
git diff --cached
git commit -m "docs(ai): add applied enterprise AI learning handbook"
git push origin main
```

`git diff --cached` reviews staged content before committing; `git push origin main` uploads the branch to GitHub. If your default branch is different, use the correct branch name. Check `git status` before staging. Never push secrets or internal data.

---

## 17. Quick reference and further reading

### Starting services

```bash
# Ollama: check local model API
curl -s http://localhost:11434/api/tags

# FastAPI: from activated .venv, project root
python -m uvicorn ai_api:app --host 127.0.0.1 --port 8000

# MCP HTTP: separate terminal, server process stays running
python hr_mcp_http_server.py

# MCP HTTP client: another terminal
python hr_mcp_http_client.py

# Test local RAG endpoint
curl -s -X POST http://127.0.0.1:8000/api/v1/ai/ask \
  -H 'Content-Type: application/json' \
  -d '{"question":"How many earned leave days can I carry forward?"}'
```

### Docs

- Ollama Python: <https://github.com/ollama/ollama-python>
- LangGraph: <https://docs.langchain.com/oss/python/langgraph/overview>
- ChromaDB: <https://docs.trychroma.com/>
- FastAPI: <https://fastapi.tiangolo.com/>
- MCP specification: <https://modelcontextprotocol.io/specification>
- MCP Python SDK: <https://github.com/modelcontextprotocol/python-sdk>
- Pydantic: <https://docs.pydantic.dev/>
- Spring Framework REST clients: <https://docs.spring.io/spring-framework/reference/integration/rest-clients.html>
- OWASP LLM Application Security: <https://genai.owasp.org/>

**End of first compiled edition.** Continue updating this document after each verified lesson. Maintain separate private development repositories for real company applications, credentials and policies.
