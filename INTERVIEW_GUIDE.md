# SnackStack: Interview Guide

## 1-Minute Elevator Pitch

**SnackStack** is a voice-enabled AI assistant for food delivery built with **LangGraph** and **LLM-powered agents**. It accepts text or voice input, intelligently routes queries to specialist agents (menu discovery or order support), and responds via speech or text. The system demonstrates modern AI concepts: multi-agent orchestration, RAG for semantic search, human-in-the-loop workflows, and concurrent voice processing.

---

## What This Project Does

### Core Features

1. **Multi-Modal Input**
   - Text mode (terminal)
   - Voice mode with automatic speech detection (VAD)
   - 5-second fixed recording or dynamic voice-activity detection

2. **Intelligent Routing**
   - Orchestrator LLM classifies queries (greeting, menu search, order tracking, or combination)
   - Dispatches to 1 or 2 agents in parallel using LangGraph's `Send()`
   - Example: "Suggest a pizza AND track my order" → both agents run concurrently

3. **Two Specialist Agents**
   - **Menu Agent**: Searches the food catalog via RAG (ChromaDB vector DB)
   - **Order Agent**: Looks up order status; uses Human-in-the-Loop if missing order ID

4. **Multi-Turn Conversation**
   - Conversation history persists via `MemorySaver` with `thread_id`
   - Follow-up queries work naturally: "What's the price?" understands prior context

5. **Voice Output**
   - OpenAI TTS with multiple voice options (alloy, shimmer, etc.)
   - Buffered (complete WAV then play) or streaming (chunk-by-chunk via PCM)
   - Responses formatted for speech (no markdown, bullets, or lists)

---

## Architecture Overview

```
Input (Voice/Text)
    ↓
    ├─→ [Orchestrator] Classifies query type
    ↓
    └─→ Send() dispatches to agent(s)
        ├─→ [Menu Agent] → Conditional: search_menu_catalog tool?
        │   └─→ [ChromaDB RAG]
        │
        ├─→ [Order Agent] → Conditional: get_order_status tool?
        │   └─→ [HITL interrupt] if no order ID found
        │
        ↓
    [Synthesizer] Merges all responses
        ↓
    Output (TTS or Text)
```

### Key Architectural Patterns

| Concept | Why It Matters |
|---------|---------------|
| **StateGraph** | Manages conversation state, memory, and agent transitions |
| **Conditional Edges** | Agents decide: "Should I call my tool, or respond directly?" |
| **Parallel Dispatch** | `Send()` runs menu & order agents simultaneously |
| **RAG (Retrieval-Augmented Generation)** | Menu queries use semantic search instead of keyword matching |
| **Human-in-the-Loop** | `interrupt()` pauses the graph to ask user for missing info |
| **Multi-Turn Memory** | Context persists across turns for natural follow-ups |

---

## Project Structure

```
snackstack/
├── main.py              # CLI entry point, argument parsing, REPL
├── config.py            # OpenAI API, embeddings, LLM setup
├── state.py             # StackState Pydantic model (shared across all nodes)
├── graph.py             # StateGraph builder & compiler
├── logger.py            # Centralized logging
│
├── data/
│   ├── menu.py          # 8 hardcoded dishes (names, price, tags)
│   └── orders.py        # Mock order database + delivery status rules
│
├── tools/
│   ├── rag.py           # ChromaDB initialization & embedding
│   ├── menu_tools.py    # search_menu_catalog() with semantic search
│   └── order_tools.py   # get_order_status() lookup
│
├── agents/
│   ├── prompts.py       # System prompts for all LLM nodes
│   ├── orchestrator.py  # Routes queries → Menu/Order/Both
│   ├── menu_agent.py    # Discovers food items
│   ├── order_agent.py   # Tracks orders (uses interrupt for HITL)
│   └── synthesizer.py   # Merges agent outputs
│
└── voice/
    ├── recorder.py      # Whisper STT + energy-based VAD
    └── speaker.py       # OpenAI TTS + sounddevice playback
```

---

## How It Works: Step-by-Step

### Example: "Show me vegan dishes under 300"

1. **Input Capture**
   - Voice mode: Microphone → Whisper STT → "Show me vegan dishes under 300"
   - Text mode: User types the query

2. **Orchestrator (Router)**
   - LLM sees this is food-related → dispatches to **Menu Agent only**
   - Adds conversation history to context

3. **Menu Agent**
   - LLM receives: query, conversation history, system prompt
   - LLM decides: "I need to search the menu" → calls `search_menu_catalog`
   - `search_menu_catalog` → ChromaDB semantic search → finds vegan, price < 300
   - LLM receives results → generates natural language response

4. **Synthesizer**
   - Only one agent ran, so synthesizer wraps its response

5. **Output**
   - Voice mode: TTS converts response → plays audio
   - Text mode: Prints response to terminal

---

### Example: "Where is my order? Track ORD-203"

1. **Orchestrator**
   - Sees this is order-related → dispatches to **Order Agent only**

2. **Order Agent**
   - LLM checks: "Does the query contain an order ID?"
   - Found: `ORD-203` → calls `get_order_status` immediately
   - Results show order is being prepared → LLM generates response

3. **Output**
   - Voice/text mode: "Your Classic Cheeseburger is being prepared..."

---

### Example: "Suggest something good AND where's my ORD-201?"

1. **Orchestrator**
   - Detects both menu & order intent → dispatches to **both agents in parallel** via `Send()`

2. **Parallel Execution**
   - Menu Agent: searches catalog, generates suggestion
   - Order Agent: looks up ORD-201, generates status

3. **Synthesizer**
   - Combines: "I recommend our Butter Chicken. Your Butter Chicken order (ORD-201) is out for delivery."

4. **Output**
   - Voice/text: Combined response

---

## Technologies & Dependencies

| Component | Technology |
|-----------|-----------|
| **Agent Framework** | LangGraph (StateGraph, multi-agent routing) |
| **LLM** | OpenAI (configurable, defaults to GPT-4) |
| **Vector DB** | ChromaDB (semantic search for menu) |
| **Speech-to-Text** | Whisper (OpenAI) |
| **Text-to-Speech** | OpenAI TTS (streaming or buffered) |
| **Voice Processing** | sounddevice, soundfile (PCM playback) |
| **Validation** | Pydantic (structured routing schema) |
| **Runtime** | Python 3.11+, uv (dependency manager) |

---

## Features to Highlight

### ✅ What Works Well

1. **Multi-Agent Orchestration**
   - Demonstrates intelligent routing with LLM-based classification
   - Parallel dispatch via `Send()` shows scalability

2. **RAG Integration**
   - Real semantic search (not keyword matching) over menu items
   - Easy to extend to other catalogs

3. **Human-in-the-Loop**
   - Uses LangGraph's `interrupt()` to pause & ask user for missing info
   - Shows practical HITL pattern for partial data

4. **Conversation Memory**
   - `MemorySaver` + `thread_id` = multi-turn context
   - Follow-up queries work naturally

5. **Voice Pipeline**
   - VAD (voice activity detection) stops recording after silence
   - Streaming TTS chunks from OpenAI → immediate playback
   - Works in text, voice, and hybrid modes

### ⚠️ Known Limitations (Be Honest)

1. **Turn-Based, Not True Concurrent Voice**
   - No simultaneous listening during TTS playback
   - No barge-in/interruption during response (feature branch exists)
   - Whisper only transcribes after full recording

2. **Energy-Based VAD, Not Neural**
   - Uses RMS energy detection, not Silero VAD
   - May have false triggers in noisy environments
   - ~1 second of silence to end recording

3. **Mock Data**
   - 8 hardcoded dishes in `data/menu.py`
   - Fake order database in `data/orders.py`
   - Real system would connect to actual APIs

4. **Single LLM Provider**
   - Hardcoded to OpenAI (GPT-4)
   - Config is modular but would require refactor for multi-provider

---

## Running the Project

### Setup
```bash
uv sync                              # Install dependencies
cp .env.example .env                 # Add your OpenAI API key
```

### Modes
```bash
uv run snackstack                    # Text mode (default)
uv run snackstack --voice-out        # Text input → voice output
uv run snackstack --voice --vad      # Voice input → voice output (smart recording)
uv run snackstack --voice --vad --stream --continuous  # Hands-free conversation
```

### Key Commands (In-Session)
- `reset` — Start fresh conversation
- `quit` — Exit

---

## Interview Talking Points

### 1. Why LangGraph?
"I chose LangGraph for its composable graph model. Each agent is a node, and the orchestrator controls routing via conditional edges and `Send()`. It's cleaner than manual state management and scales to many agents."

### 2. Why Parallel Agents?
"When queries involve both menu and order info, running both agents in parallel is faster and avoids redundant LLM calls. LangGraph's `Send()` makes this trivial."

### 3. RAG vs. Simple Search?
"ChromaDB enables semantic search — 'vegan dishes under 300' finds relevant items even without exact keyword matches. It's more flexible and user-friendly than regex or SQL WHERE clauses."

### 4. Human-in-the-Loop?
"The Order Agent uses `interrupt()` to pause when the order ID is missing, ask the user, and resume. It's a clean pattern for multi-turn workflows where the LLM can't proceed independently."

### 5. Conversation Memory?
"`MemorySaver` with `thread_id` keeps conversation history, so follow-up queries work naturally. I could swap it for a persistent DB (PostgreSQL, Redis) at scale."

### 6. Voice Integration?
"Whisper handles transcription, energy-based VAD detects speech, and streaming TTS chunks provide low latency. The `--stream` flag shows real-time playback; buffered mode is simpler but slower."

### 7. What Would You Scale?
- Replace mock data with real APIs
- Add a persistent DB for conversation history
- Implement true concurrent listening (feature branch exists)
- Add error recovery and backoff strategies
- Instrument observability (OpenTelemetry)

---

## Common Interview Questions

**Q: How would you add a third agent (e.g., Loyalty or Payment)?**
A: Create a new agent in `snackstack/agents/`, define its system prompt & tools, add a route in the orchestrator's Pydantic schema, and update the graph to include it in `Send()`. The router would dispatch in parallel just like menu & order.

**Q: What happens if the LLM tool call fails?**
A: The tool raises an exception, which LangGraph catches. I could add retry logic or error handlers in the conditional edge. Currently it surfaces the error to the user.

**Q: Why not use `langchain.agents.AgentExecutor`?**
A: `AgentExecutor` is simpler for single agents but less flexible for multi-agent coordination and conditional routing. LangGraph gives me explicit control over the graph structure.

**Q: How do you handle latency?**
A: Streaming TTS reduces perceived latency by playing chunks as they arrive. For multi-agent queries, parallel dispatch via `Send()` is faster than sequential calls.

**Q: Could you replace OpenAI with another LLM?**
A: Yes. I'd update `config.py` to use a different `ChatModel` (e.g., LangChain's `ChatAnthropic` or `ChatGoogleGenerativeAI`). The rest of the code is provider-agnostic.

---

## Files to Reference During Interview

- **`graph.py`** — StateGraph structure, routing logic
- **`orchestrator.py`** — LLM-based query classification
- **`menu_agent.py`** — Conditional edge example (call tool or not?)
- **`order_agent.py`** — Human-in-the-Loop with `interrupt()`
- **`tools/rag.py`** — ChromaDB setup & semantic search
- **`voice/recorder.py`** — Whisper + VAD
- **`main.py`** — Entry point, mode flags

---

## Summary

**SnackStack** demonstrates a production-like multi-agent AI system with voice I/O. It combines:
- Intelligent orchestration (LangGraph)
- Semantic search (RAG)
- Multi-turn memory
- Human-in-the-Loop workflows
- Voice I/O (STT/VAD/TTS)
- Parallel agent dispatch

It's suitable for food delivery, customer support, or any domain needing multi-agent orchestration. The code is clean, modular, and ready to extend.
