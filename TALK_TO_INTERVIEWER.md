# What to Say About SnackStack

## The Pitch (30 seconds)

"SnackStack is a voice-enabled AI assistant for food delivery. The core idea is that it takes either text or voice input, figures out what the user is asking — whether they want menu recommendations or to track an order — and routes that query intelligently to the right specialist agent. Everything runs on LangGraph, which gives us multi-agent orchestration and memory across conversation turns. The user gets responses back in voice or text."

---

## What It Does (1-2 minutes)

### The Flow
"So here's how it works. User says something like 'show me vegan dishes under 300 rupees' or 'where's my order ORD-203.' The system captures that, either through the microphone or terminal. Then an orchestrator LLM reads the query and decides: is this about the menu, or about tracking an order? Or both?

Once it knows, it dispatches to the right agent or agents. If the user says 'suggest something AND track my order,' both agents run in parallel. That's where LangGraph's Send() function is really useful — we can fan out work instead of doing it sequentially."

### The Agents
"The Menu Agent knows how to search our food catalog using semantic search — ChromaDB. So if someone says 'I want something spicy but no dairy,' it actually understands the intent, not just keyword matching.

The Order Agent handles tracking. It looks up the order ID in our database. But here's the thing — if someone says 'where's my order' without giving an ID, the agent pauses and asks them. That's human-in-the-loop. LangGraph has this interrupt() function that lets us pause execution, ask the user, and then resume. It's clean."

### Memory
"Every conversation has a thread ID. So if I ask 'what's the price of that?' on the next turn, the system remembers what 'that' refers to. The memory persists across turns, so follow-ups just work naturally."

---

## Why This Architecture? (Why LangGraph, why multi-agent?)

"I chose LangGraph because of how composable it is. Each agent is a node in the graph. The orchestrator is another node. They talk to each other through a shared state object. The beauty is conditional edges — an agent can decide 'I need to call my tool' or 'I can answer this directly.' That makes it really flexible.

Multi-agent is better than a single monolithic LLM because agents can specialize. The Menu Agent has prompts and tools optimized for food discovery. The Order Agent is built for customer support. If I need to add a third agent for loyalty points or payments, I just create a new file, add it to the graph, and update the router. The system scales without touching the core orchestration logic."

---

## The Voice Part (Why it matters)

"Voice is the interesting bit. We're using Whisper for speech-to-text, which is accurate. For recording, we have voice activity detection — it uses energy-based detection, so it stops recording after about a second of silence instead of waiting for a fixed timer. That makes it feel more natural.

On the output side, we're using OpenAI's TTS. But instead of generating the entire audio and then playing it, we can stream it — OpenAI sends us chunks of PCM audio, and we play them as they arrive using sounddevice. That's lower latency. So the user hears the response start almost immediately."

---

## What's Hard (Be Honest)

"One thing we're not doing yet — and this is on a feature branch — is true concurrent voice. Right now it's turn-based. The user speaks, we transcribe, the LLM thinks, we generate audio, then we speak. There's no simultaneous listening while we're playing the response. That would require more sophisticated concurrent recording and would be a bigger engineering lift.

Also, the VAD is energy-based, not neural. So in a really noisy cafe, it might get confused. For this project, it works fine. At scale, you'd probably want Silero or another neural VAD.

And obviously, the order and menu data are mocked. Real integration would connect to actual APIs and databases."

---

## What I'm Proud Of

**Parallel Dispatch** — When we get a query like 'give me a recommendation AND check my order status,' both agents run at the same time. That's faster and cleaner than sequential calls.

**RAG for Search** — We're not doing keyword matching. We embedded the menu items using OpenAI embeddings and stored them in ChromaDB. So semantic queries actually work. 'Something spicy but light' will find the right dishes.

**HITL Pattern** — The interrupt-and-resume pattern in the Order Agent is clean. Most people manually write if-else logic for this; LangGraph gives us a proper abstraction.

**Memory Across Turns** — Conversation history persists automatically. The orchestrator and agents see the full history. That's why follow-ups work.

---

## If They Ask: "How Would You Scale This?"

"Good question. A few things:

1. **Real Data** — Connect to actual menu APIs and order databases instead of mocks.

2. **Persistent Memory** — Right now we're using LangGraph's in-memory MemorySaver. At scale, I'd use PostgreSQL or Redis to store conversation threads.

3. **Concurrent Voice** — Implement simultaneous listening during TTS. There's actually a feature branch for barge-in, so users can interrupt the assistant mid-response.

4. **Observability** — Add structured logging and tracing. Maybe OpenTelemetry to see where latency is happening.

5. **Error Handling** — Right now if a tool fails, the user sees the error. I'd add retry logic and graceful degradation.

6. **Multi-Provider LLMs** — The code currently uses OpenAI. I'd refactor config.py to support Claude, Gemini, etc., so we're not locked in."

---

## If They Ask: "Why Not Use X?"

**"Why not just prompt a single LLM to handle everything?"**
"You could, but you lose specialization. Each agent has optimized prompts and tools. If I'm searching for food, I don't need the LLM to think about order tracking logic. Agents are cleaner."

**"Why LangGraph over something simpler?"**
"I looked at langchain.agents.AgentExecutor, which is simpler for single agents. But it doesn't give you multi-agent coordination and conditional routing as naturally. LangGraph's graph model is explicit — I can see exactly how information flows. Plus it's what we're using in production at my work, so I'm familiar with it."

**"Why ChromaDB for search?"**
"Keyword matching doesn't work well for food queries. 'Something light and spicy' won't match 'Tandoori Chicken' with regex. Semantic search understands intent. ChromaDB is lightweight and open-source. For a real system, I might use Pinecone or Weaviate, but the approach is the same."

---

## If They Ask: "What Would You Do Differently?"

"Honestly, if I were building this again:

1. I'd start with a simpler text-only version to nail the multi-agent routing, then add voice later. I did voice too early and it added complexity I didn't need upfront.

2. I'd add more error handling from day one. Right now errors surface directly to the user.

3. I'd make the LLM provider configurable from the start. I hardcoded OpenAI, which was fine for a prototype but limiting.

4. I'd instrument logging better. Debugging multi-agent flows is hard without good traces.

5. The mock data is fine for demos, but I'd wire up real APIs earlier to catch integration issues sooner."

---

## Demo Script (If They Ask to See It)

```
"Let me show you. I'll run it in voice mode with VAD.

[uv run snackstack --voice --vad --stream]

Watch — I just press Enter once, then I speak.

[speaks into mic] 'Show me vegan options under 300.'

[waits for transcription]

You can see Whisper transcribed it, the orchestrator routed it to the Menu Agent, the agent searched ChromaDB, and now TTS is playing the response. Notice the audio starts pretty much instantly — that's streaming.

Let me try something with order tracking.

[speaks] 'Where's my order?'

[waits]

No order ID in that query, so the agent paused and asked for one. This is the interrupt() function in action. I say the order ID...

[speaks] 'ORD-203'

[waits]

Now it has the ID, resumed, and gave the status. That's human-in-the-loop without manual if-else logic."
```

---

## Closing (What You Built)

"To step back — I built a system that shows how to orchestrate multiple AI agents for a real use case. It combines LangGraph for coordination, RAG for intelligent search, voice I/O for accessibility, and conversation memory for context. The code is clean and modular, so adding new agents or tools is straightforward. It's not production-ready — the data is mocked, error handling is minimal — but it's a solid foundation for a real food delivery or customer support assistant."

---

## One-Liners You Can Drop

- "LangGraph's Send() is how we run agents in parallel."
- "ChromaDB handles semantic search for the menu."
- "We use interrupt() to pause and ask for missing info."
- "Streaming TTS chunks reduce latency."
- "The state object keeps everything in sync across agents."
- "MemorySaver gives us conversation memory with a thread_id."

---

## Things NOT to Say

❌ "It's production-ready." (It's a prototype.)
❌ "The VAD is perfect." (It's energy-based, not neural.)
❌ "We support barge-in." (Feature branch, not main.)
❌ "It integrates with real restaurants." (Mock data only.)
❌ "It can handle hundreds of concurrent users." (No infrastructure for that yet.)
