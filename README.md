# Iterative Essay Writer Agent with Reflection & Research Loops

## 1. Overview
This project implements a production-style **iterative writing agent** built using **LangGraph**, combining planning, web research, essay generation, critique, and revision cycles into a fully stateful workflow.

The system uses a graph-based architecture where the agent:
- plans an essay
- researches the topic using Tavily search
- generates a draft
- critiques its own output
- performs additional targeted research
- iteratively improves the essay across multiple revision loops

The workflow is powered by a modern LangGraph state machine with persistent checkpoint memory using SQLite.

---

## 2. Workflow

### 2.1 LLM Setup
Configured an instruction-following LLM for structured planning, essay generation, critique generation, and structured query generation.

.env configuration:
```bash
api_key = your_groq_api_key
TAVILY_API_KEY = your_tavily_api_key
```

---

### 2.2 Graph Architecture
The system is built using StateGraph from LangGraph.

Nodes:
- planner → creates outline
- research_plan → generates queries + collects sources
- generate → writes essay draft
- reflect → critiques output
- research_critique → gathers revision research

---

### 2.3 State Management
Shared state schema:
```python
task: str
plan: str
draft: str
critique: str
content: List[str]
revision_number: int
max_revisions: int
```

---

### 2.4 Planning Phase
Generates structured essay outline before writing begins.

---

### 2.5 Research Pipeline
Uses Tavily API to:
- generate queries
- retrieve web results
- store contextual knowledge in `content`

---

### 2.6 Essay Generation
Combines:
- task
- plan
- research content

Produces draft + increments revision counter.

---

### 2.7 Reflection
Critiques:
- structure
- clarity
- depth
- completeness

---

### 2.8 Iteration Loop
```
generate → reflect → research_critique → generate
```

Stops when:
```
revision_number >= max_revisions
```

---

### 2.9 Memory
SQLite checkpointing via LangGraph:
- persistent state
- thread-based isolation
- execution history tracking

---

## 3. Tech Stack
- LangGraph
- LangChain Core
- Tavily API
- SQLite
- Pydantic
- Python-dotenv

---

## 4. Architecture Flow

User Task → Planner → Research → Generate → Reflect → Research Critique → Generate (loop)

---

## 5. Key Design Decisions

### 5.1 Iterative Refinement
Multi-pass generation improves quality vs single-shot LLM output.

### 5.2 Explicit Planning
Separates reasoning (plan) from generation (draft).

### 5.3 Live Research Integration
Uses external API to ground outputs in real data.

### 5.4 Stateful Execution
LangGraph enables resumable, traceable workflows.

### 5.5 Reflection Loop
Self-critique improves output quality iteratively.

---

## 6. Conclusion
A fully stateful AI writing system combining planning, research, generation, and reflection into a single LangGraph-based autonomous workflow.
