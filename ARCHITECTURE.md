# Nexus — Architecture

**OSDHack 2026 — Team QuantAI**

This document covers the system diagram, model/data pipeline, local vs. cloud split, and the key design decisions behind Nexus.

---

## 1. Two Deployment Topologies

Nexus ships as two variants of the same core logic (`intents.py`, `resilient.py`, `config.py`, `logger.py` are shared):

### 1.1 Production — Vercel Serverless (frontend + API unified)

```
┌──────────────────────────────────────────────────────────────────┐
│                      VERCEL (single deployment)                   │
│                                                                    │
│  Browser ──► Static Frontend (frontend/, served from CDN)         │
│                 │                                                  │
│                 │  POST /api/chat                                  │
│                 ▼                                                  │
│           api/index.py  (FastAPI, cold-started per request)        │
│                 │                                                  │
│                 ▼                                                  │
│           IntentRouter (intents.py)                                 │
│                 │                                                  │
│     ┌───────────┼────────────┬─────────────┐                       │
│     ▼           ▼            ▼             ▼                       │
│   LLM        Weather      Wolfram        Search                    │
│ (OpenRouter) (OpenWeather) (Wolfram API) (DuckDuckGo)               │
│     ▲                                                                │
│     └── all wrapped in resilient.py: CircuitBreaker + retry+backoff │
│         + Bulkhead concurrency limiting                              │
└──────────────────────────────────────────────────────────────────┘
```

Live at: **https://jarvis-ai-intelligence-system-llm.vercel.app**

- `vercel.json` rewrites every path to `api/index.py`, so the Python FastAPI app also serves the frontend's `index.html`, `app.js`, `style.css`, etc. via `frontend_assets.py` (assets embedded as Python strings so a single serverless function can serve everything).
- Each request is stateless — there is no persistent process, no in-memory session, and no database. `IntentRouter()` is instantiated once per cold start and reused across warm invocations.
- Logs write to `/tmp/logs/nexus.jsonl`, which is ephemeral per function instance.

### 1.2 Local Development — WebSocket + Audio

```
┌──────────────────┐     WebSocket ws://:8765     ┌───────────────────┐
│   frontend/       │ ◀───────────────────────────▶ │   main.py          │
│  (HTML/CSS/JS)     │                               │   NexusCore        │
└──────────────────┘                               └─────────┬──────────┘
                                                              │
                          ┌───────────────────────────────────┼───────────────────────────┐
                          ▼                                   ▼                           ▼
                    IntentRouter                        AudioEngine                  HTTP :9765
                  (same tool plugins                (mic capture via                (health/live,
                   as production)                 SpeechRecognition +               ready, deep;
                                                    pyttsx3 TTS output)              Prometheus /metrics)
```

- `main.py` runs one long-lived asyncio event loop: a `websockets` server for chat, an `aiohttp` server for health/metrics, and a background `AudioEngine.listen_loop()` task for the microphone.
- Every chat/voice turn goes through the identical `IntentRouter.route()` used in production — the only difference between local and Vercel is the transport (WebSocket vs. HTTP POST) and the availability of a real microphone/speaker.

---

## 2. Model / Tool Pipeline

```
User query (text or transcribed speech)
        │
        ▼
 IntentRouter.route(text)
        │
        ├── classifies intent (weather / math / search / scheduler / general chat)
        │
        ▼
 Selected tool plugin
        │
        ├── LLM path      → OpenRouter API → hosted model (e.g. anthropic/claude-sonnet-4)
        ├── Weather path   → OpenWeatherMap API
        ├── Wolfram path    → Wolfram Alpha API
        ├── Search path      → DuckDuckGo
        └── Scheduler path    → local timer logic (no external call)
        │
        ▼
 Result dict: {response, tool, log, latency_ms, type}
        │
        ├── returned as JSON (Vercel) or broadcast over WebSocket (local)
        └── (local mode only) spoken aloud via pyttsx3
```

Every external call above (LLM, weather, Wolfram, search) passes through the resilience decorators in `resilient.py` before it can fail the whole request:

- **CircuitBreaker** — opens after `Nexus_FAILURE_THRESHOLD` consecutive failures, short-circuits further calls for `recovery_timeout` seconds, then probes with a half-open state before fully closing again.
- **retry_with_backoff** — exponential backoff with jitter, up to `Nexus_MAX_RETRIES` attempts.
- **Bulkhead** — semaphore-based concurrency cap so one slow dependency can't exhaust all available request-handling capacity.

This is the main engineering contribution beyond "call an LLM API": a small but real resilience layer around every network dependency, plus structured JSON logging (`logger.py`) that captures which pattern fired (`circuit_breaker_opened`, `retry_attempt`, `bulkhead_at_capacity`, etc.) for observability.

---

## 3. Local vs. Cloud Components (Design Decision Summary)

| Layer | Where it runs | Rationale |
|---|---|---|
| Conversational reasoning | Cloud (OpenRouter → hosted LLM) | No local model is bundled — keeps the serverless function lightweight (no multi-GB weights to cold-start) and lets the model be swapped via `OPENROUTER_MODEL` without a redeploy |
| Weather / math / search | Cloud (respective APIs) | These are inherently live-data lookups; there's no meaningful "on-device" equivalent |
| Intent classification / routing | On-device (in-process Python) | Deterministic, fast, and doesn't need a model call — keeps routing latency and cost low |
| Resilience (circuit breaker, retry, bulkhead) | On-device (pure control flow) | Protects the cloud dependencies above without adding any external dependency itself |
| Text-to-speech (local mode) | On-device (`pyttsx3`, uses OS voice engine) | The one component that works fully offline — chosen specifically so local voice output doesn't depend on network access |
| Speech-to-text | Browser Web Speech API (production) or `SpeechRecognition` default backend (local) | Convenience choice for the hackathon build; **not configured to use an offline recognizer**, so this still depends on network access in both modes today |
| Logging / metrics | On-device (`/tmp` logs, Prometheus in local mode) | Ephemeral, per-instance — no data warehousing |

**Honest summary:** Nexus's intelligence layer is cloud-hosted; the codebase's own contribution is the routing/resilience/observability scaffolding around that cloud call, plus a fully offline TTS path in local mode. There is no on-device inference of an LLM, STT, or any ML model in this build.

---

## 4. Data Flow — Single Request, End to End (Vercel)

1. Browser sends `POST /api/chat` with `{"text": "..."}`.
2. FastAPI (`api/index.py`) validates the payload via a Pydantic `QueryRequest` model.
3. `IntentRouter.route(text)` is awaited; internally this may call one external API, wrapped by the resilience layer.
4. A result dict (`response`, `tool`, `log`, `latency_ms`, `type`) is returned as JSON.
5. The frontend renders the response text in the HUD and (if voice mode is enabled) speaks it via the browser's `SpeechSynthesis` API.
6. Nothing from this exchange is persisted beyond the ephemeral JSON log line for that function invocation.

---

## 5. Key Design Decisions

- **Single codebase, two transports.** `intents.py`/`resilient.py`/`config.py` are transport-agnostic, so the exact same tool-routing logic runs whether the caller is a WebSocket client (`main.py`) or an HTTP request (`api/index.py`). This avoids maintaining two parallel implementations of the "brain."
- **No bundled model weights.** Given the serverless cold-start constraint on Vercel, using a hosted LLM via OpenRouter avoids multi-second-plus cold starts that would come with loading a local model into a short-lived function.
- **Resilience as a cross-cutting concern, not per-tool code.** `CircuitBreaker`, `retry_with_backoff`, and `Bulkhead` are implemented once as decorators/context managers and applied uniformly, so adding a new tool doesn't mean re-implementing failure handling.
- **Frontend assets embedded in Python (`frontend_assets.py`).** Vercel's rewrite rule points every route at `api/index.py`; embedding the built HTML/CSS/JS as strings lets one serverless function serve the whole app without a separate static-hosting configuration.
- **Demo mode as a safety net.** `Nexus_DEMO_MODE` lets the app return canned/offline responses when API keys aren't configured — useful for judges running the project without needing to provision every third-party key.

---

## 6. Known Limitations

- No offline/on-device LLM or STT — every "smart" response depends on internet connectivity and valid API keys.
- CORS is fully open (`allow_origins=["*"]`) in `api/index.py` — fine for a hackathon demo, not appropriate for production.
- No persistent conversation memory across turns beyond what's implemented inside `IntentRouter` — check `intents.py` if multi-turn context is claimed elsewhere in your demo script, and only state what's actually implemented.
- No automated evaluation harness is included in this repo — see the Evaluation section of `README.md` for what to add before judging if you want measured accuracy numbers rather than qualitative demo behavior.
