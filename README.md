# AI Essay Writer — Multi-Agent Essay Generation with LangGraph

## 1. Overview

This project implements a **multi-node essay writing pipeline** powered by **Llama 3.3 70B** via Groq, built on the LangGraph state machine framework. The system autonomously plans, researches, drafts, critiques, and revises essays through a cyclic graph of specialized nodes — stopping only when the maximum revision count is reached.

---

## 2. Workflow

### 2.1 LLM Setup
- Used **Llama 3.3 70B Versatile** via **Groq API** for fast, free inference.
- Configured with `temperature=0` for deterministic, consistent writing quality.
- API key is loaded securely from a `.env` file using `python-dotenv`:

```env
# .env
api_key=your_groq_api_key_here
TAVILY_API_KEY=your_tavily_api_key_here
```

```python
llm = ChatGroq(
    model="llama-3.3-70b-versatile",
    temperature=0.0,
    api_key=os.getenv('api_key')
)
```

### 2.2 State Design
The graph state tracks all intermediate artifacts across nodes:

```python
class AgentState(TypedDict):
    task: str            # user's essay topic
    plan: str            # high-level outline from planner
    draft: str           # current essay draft
    critique: str        # feedback from reflection node
    content: List[str]   # research results accumulated across runs
    revision_number: int # current revision count
    max_revisions: int   # stopping condition
```

### 2.3 Node Pipeline

#### Planner Node
- Receives the user's topic.
- Generates a high-level essay outline with section notes and writing instructions.
- Uses a dedicated `PLAN_PROMPT` focused on essay structure.

#### Research Plan Node
- Reads the outline and generates up to 3 targeted search queries.
- Uses **structured output** (`Queries` Pydantic model with `json_mode`) to enforce valid JSON.
- Executes each query against the **Tavily Search API** (`max_results=2` per query).
- Appends all retrieved content to the shared `content` list in state.

#### Generation Node
- Joins all accumulated research content into a single context block.
- Writes a 5-paragraph essay grounded in the research and outline.
- Increments `revision_number` on every call.

#### Reflection Node
- Acts as a teacher grading the draft.
- Generates detailed critique with recommendations on length, depth, and style.
- Output is stored as `critique` in state for the next research cycle.

#### Research Critique Node
- Reads the critique and generates up to 3 new search queries targeting identified weaknesses.
- Fetches additional research and appends it to the existing `content` list.
- Feeds enriched context back into the generation node.

### 2.4 Graph Structure

```
planner → research_plan → generate ──────────────────── END
                               ↑                    ↓ (revision_number > max_revisions)
                        research_critique ← reflect
                               ↑__________________|
                               (loops until max reached)
```

The conditional edge after `generate` checks `revision_number <= max_revisions`. If true, the graph continues to `reflect`; otherwise it terminates at `END`.

### 2.5 Memory & Persistence
- Integrated **`SqliteSaver`** (LangGraph checkpointer) backed by a local `.db` file.
- All graph state is persisted across runs — the essay can be resumed or inspected at any checkpoint.
- Memory is scoped per user via `thread_id` in the config:

```python
conn = sqlite3.connect("AI_Assistant_Essay_Writer.db", check_same_thread=False)
memory = SqliteSaver(conn)
thread = {"configurable": {"thread_id": "1"}}
```

### 2.6 Structured Output
- Both research nodes use `.with_structured_output(Queries, method="json_mode")` to force the LLM to return valid, parseable JSON search queries.
- Eliminates brittle string parsing and ensures reliable tool invocation.

### 2.7 Think Tag Stripping
- All LLM outputs pass through `remove_think_tag()` to strip `<think>...</think>` reasoning blocks emitted by some Groq models:

```python
def remove_think_tag(self, content):
    return re.sub(r"<think>.*?</think>", "", content, flags=re.DOTALL).strip()
```

---

## 3. Tech Stack

- Python
- LangGraph (`StateGraph`, `SqliteSaver`, `END`)
- LangChain (`ChatGroq`, `SystemMessage`, `HumanMessage`)
- Tavily Python Client (live web research)
- Pydantic v2 (structured output schema)
- SQLite (persistent graph checkpointing)
- python-dotenv (secure API key management)

---

## 4. Architecture

```
User Topic
     │
     ▼
  Planner Node
  (outline)
     │
     ▼
  Research Plan Node
  (Tavily search × 3 queries)
     │
     ▼
  Generation Node ◄────────────────────────┐
  (5-paragraph essay)                      │
     │                                     │
     ├── revision > max ──► END            │
     │                                     │
     ▼                                     │
  Reflection Node                          │
  (critique & recommendations)             │
     │                                     │
     ▼                                     │
  Research Critique Node                   │
  (Tavily search × 3 queries) ─────────────┘

  SqliteSaver → AI_Assistant_Essay_Writer.db
  (full state persisted at every node)
```

---

## 5. Key Design Decisions

### 5.1 Cyclic Graph for Iterative Refinement
Unlike a simple chain, the graph loops between generation, reflection, and research — mirroring how a human writer revises. The `max_revisions` parameter gives full control over quality vs. cost tradeoff.

### 5.2 Accumulated Research Content
The `content` list in state grows across both research nodes. Each revision cycle adds new search results on top of previous ones, giving the writer progressively richer context with every loop.

### 5.3 Structured Output for Search Queries
Using Pydantic's `Queries` model with `json_mode` instead of plain text parsing makes query generation robust and deterministic — no regex, no brittle splits.

### 5.4 Separation of Prompts
All five prompts (`PLAN_PROMPT`, `RESEARCH_PLAN_PROMPT`, `WRITER_PROMPT`, `REFLECTION_PROMPT`, `RESEARCH_CRITIQUE_PROMPT`) are defined as constants outside the class and injected via a `prompts` dict — making them easy to swap, version, or A/B test without touching the graph logic.

---

## 6. Usage

```python
thread = {"configurable": {"thread_id": "1"}}

input_state = {
    "task": "what is the difference between langchain and langsmith",
    "max_revisions": 2,
    "revision_number": 1,
}

for s in essay_writer.graph.stream(input_state, thread):
    print(s)
```

---

## 7. Conclusion

This project demonstrates a production-aligned, iterative essay writing system that combines LLM planning, live web research, and self-critique in a fully stateful LangGraph pipeline. The cyclic architecture, structured outputs, and persistent SQLite memory make it a solid foundation for any long-form content generation use case.
