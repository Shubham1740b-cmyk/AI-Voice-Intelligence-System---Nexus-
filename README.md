# Nexus — Voice-Enabled AI Intelligence System

**OSDHack 2026 Final Submission — Team QuantAI**

> Nexus (Just A Rather Very Intelligent System) is a voice-enabled AI assistant with a cyberpunk neural HUD, real-time speech interaction, and multi-tool intent routing. It ships in two forms: a **Vercel serverless deployment** (frontend + Python API in one project) and a **local full-stack mode** (WebSocket server + microphone capture) for development.

- **Live demo:** https://jarvis-ai-intelligence-system-llm.vercel.app
- **Repo:** https://github.com/Shubham1740b-cmyk/AI-Voice-Intelligence-System---Nexus-

---

## Table of Contents

1. [What It Does](#what-it-does)
2. [Local AI Verification (What Runs Where)](#local-ai-verification-what-runs-where)
3. [Architecture Summary](#architecture-summary)
4. [Requirements](#requirements)
5. [Setup — Reproduce in 5 Steps](#setup--reproduce-in-5-steps)
6. [Running the Project](#running-the-project)
7. [Sample Input / Output](#sample-input--output)
8. [Environment Variables](#environment-variables)
9. [Technical Report](#technical-report)
10. [Evaluation](#evaluation)
11. [Privacy and Safety](#privacy-and-safety)
12. [Attribution](#attribution)
13. [Project Structure](#project-structure)
14. [Troubleshooting](#troubleshooting)

---

## What It Does

Nexus takes a spoken or typed query, classifies its **intent**, routes it to the right tool, and replies through chat and (optionally) synthesized speech:

| Query | Tool routed to |
|---|---|
| "Hello Nexus" | LLM (conversational) |
| "What's the weather in London?" | OpenWeatherMap |
| "Integrate x squared from 0 to 5" | Wolfram Alpha |
| "Who is Elon Musk?" | DuckDuckGo search |
| "Set a timer for 10 minutes" | Scheduler |

Behind the intent router sits a resilience layer (circuit breaker + retry-with-backoff + bulkhead isolation), structured JSON logging, and health/metrics endpoints — the parts of the system that make it a "production-ready" assistant rather than a demo script.

---

## Local AI Verification (What Runs Where)

Being transparent about this, since it matters for judging:

| Component | Runs | Needs internet? | Notes |
|---|---|---|---|
| Conversational LLM ("AI brain") | **Cloud** | Yes | Routed through OpenRouter to a hosted model (e.g. `anthropic/claude-sonnet-4`, configurable via `OPENROUTER_MODEL`) |
| Weather tool | **Cloud** | Yes | OpenWeatherMap API |
| Math/science tool | **Cloud** | Yes | Wolfram Alpha API |
| Web search tool | **Cloud** | Yes | DuckDuckGo |
| Speech-to-text (browser/Vercel mode) | **Browser (Web Speech API)** | Yes | Handled client-side by the browser; most implementations (Chrome) relay audio to the browser vendor's speech service, so this is not fully offline |
| Speech-to-text (local mode, `SpeechRecognition`) | Depends on backend | Yes, by default | The `SpeechRecognition` library defaults to Google's Web Speech API unless explicitly configured with an offline recognizer (e.g. Vosk, CMU Sphinx) — not configured in this build |
| Text-to-speech (local mode, `pyttsx3`) | **On-device** | **No** | `pyttsx3` synthesizes speech using the local OS voice engine (SAPI5/NSSpeechSynthesizer/espeak) — this is the one component that runs fully offline |
| Circuit breaker / retry / bulkhead | On-device (logic only) | No | Pure Python control-flow around the network calls above |

**Bottom line:** Nexus is a **cloud-hybrid assistant**. The only fully on-device, no-internet-required component is the local-mode TTS voice output (`pyttsx3`). No user data is stored server-side beyond a single request/response cycle and ephemeral logs (see [Privacy and Safety](#privacy-and-safety)).

---

## Architecture Summary

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full system diagram, data flow, and design rationale. In short:

- **Production (Vercel):** static frontend + `api/index.py` (FastAPI) as a serverless function, invoked per request. No persistent process.
- **Local dev:** `main.py` runs an async WebSocket server + AudioEngine (mic capture, TTS) + an aiohttp server for health/metrics, all in one long-lived process.

---

## Requirements

| Component | Version |
|---|---|
| Python | 3.11 (see `runtime.txt`) |
| Node.js | 18+ (Vercel CLI / build orchestration only — no Node runtime code) |
| Git | any recent |
| Vercel account | free tier is enough for the demo deployment |

Optional, local audio mode only:
- macOS: `brew install portaudio`
- Ubuntu/Debian: `sudo apt install portaudio19-dev python3-pyaudio`
- Windows: PyAudio wheels via pip

---

## Setup — Reproduce in 5 Steps

```bash
# 1. Clone
git clone https://github.com/Shubham1740b-cmyk/AI-Voice-Intelligence-System---Nexus-.git
cd AI-Voice-Intelligence-System---Nexus-

# 2. Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt            # minimum, matches Vercel
# OR, for local mic/voice mode:
pip install -r requirements-local.txt

# 4. Configure secrets
cp .env.example .env
# edit .env — set OPENROUTER_API_KEY at minimum

# 5. Verify the backend imports cleanly
python -c "from intents import IntentRouter; print('Backend OK')"
```

That's the whole reproduction path — no database, no build step beyond `pip install`.

---

## Running the Project

### Option A — Vercel-like, single command (recommended for judges)

```bash
npm install -g vercel
vercel dev
```

Opens the frontend + serverless API together at `http://localhost:3000`. This mirrors exactly what runs at the live URL above.

### Option B — Full local mode (WebSocket + microphone + offline TTS)

```bash
# Terminal 1 — backend
python main.py

# Terminal 2 — frontend
python -m http.server 8080 --directory frontend
```

Open `http://localhost:8080` in Chrome/Edge/Safari; it connects to `ws://localhost:8765`.

### Option C — Makefile shortcuts

```bash
make dev     # equivalent to: python main.py
make test    # pytest tests.py -v
```

---

## Sample Input / Output

**Request** (`POST /api/chat`):
```json
{ "text": "What's the weather in Tokyo?" }
```

**Response:**
```json
{
  "response": "The current temperature in Tokyo is 27°C with clear skies.",
  "tool": "weather",
  "log": "OpenWeatherMap query succeeded",
  "type": "response",
  "latency_ms": 842
}
```

**Health check:**
```bash
curl https://jarvis-ai-intelligence-system-llm.vercel.app/api/health
# {"status": "ok", "message": "Nexus API is online.", "mode": "serverless"}
```

---

## Environment Variables

| Variable | Required | Purpose | Default |
|---|---|---|---|
| `OPENROUTER_API_KEY` | **Yes** | LLM access via OpenRouter | — |
| `OPENROUTER_MODEL` | No | Model string routed through OpenRouter | `anthropic/claude-sonnet-4` |
| `WOLFRAM_APP_ID` | No | Math/science queries | — |
| `OPENWEATHER_API_KEY` | No | Live weather | — |
| `Nexus_CITY` | No | Default weather city | `New York` |
| `Nexus_HOST` / `Nexus_PORT` | No | Local WebSocket bind address | `localhost` / `8765` |
| `Nexus_DEBUG` | No | Verbose logging | `false` |
| `Nexus_DEMO_MODE` | No | Offline fallback responses when no keys are set | `false` |
| `Nexus_HTTP_TIMEOUT` | No | Outbound HTTP timeout (s) | `15` |
| `Nexus_MAX_RETRIES` | No | Retry attempts on external calls | `3` |
| `Nexus_FAILURE_THRESHOLD` | No | Circuit breaker trip threshold | `5` |
| `Nexus_ENABLE_METRICS` | No | Prometheus metrics (local mode) | `true` |

Never commit `.env` — it's already covered by `.gitignore`.

---

## Technical Report

| Item | Detail |
|---|---|
| Model / runtime | Hosted LLM accessed via OpenRouter (default `anthropic/claude-sonnet-4`); no local model weights are shipped or loaded |
| Quantization / optimization | Not applicable — inference happens on the model provider's infrastructure, not on this device |
| Model size | N/A (remote, provider-hosted) |
| Inference latency | End-to-end request latency (network + LLM + tool call) reported per-response in `latency_ms`; typically in the hundreds of ms to low seconds depending on tool and network conditions |
| CPU/GPU/NPU usage | Local process usage is limited to I/O, JSON handling, and control flow (circuit breaker/retry/bulkhead) — negligible CPU, no GPU/NPU involvement |
| Peak memory usage | Local process footprint is small (FastAPI/aiohttp + async I/O); no model weights held in memory |
| Tested device specs | *(fill in: e.g. laptop model, OS version, RAM used during your test runs)* |
| Additional notes | Resilience layer (`resilient.py`) adds circuit breaker, exponential backoff with jitter, and bulkhead concurrency limits around every external call, so latency/failure characteristics are bounded even when a downstream API is slow or down |

---

## Evaluation

*(Fill in before submitting — suggested structure below.)*

- **Accuracy / quality:** intent-routing accuracy against a small labeled query set (e.g. X/Y correctly routed to weather/Wolfram/search/LLM)
- **Benchmark method:** manual test set of N representative queries per tool category, run against both `demo_mode=true` (offline fallback) and live keys
- **Baseline comparison:** compare intent-routing latency/success rate with and without the resilience layer enabled
- **Known failure cases:** ambiguous queries that could match more than one tool; tool timeouts when an API key is missing or the provider is down (falls back to `Nexus_DEMO_MODE` response if enabled)

---

## Privacy and Safety

- **Data sent off-device:** the text of each query is sent to whichever cloud service handles it (OpenRouter/LLM provider, OpenWeatherMap, Wolfram Alpha, or DuckDuckGo) — no query text is otherwise stored or shared.
- **Storage:** logs are written to `/tmp/logs/nexus.jsonl` in the serverless environment, which is ephemeral and cleared between cold starts; no persistent database or user-account storage exists.
- **Permissions:** local audio mode requests microphone access from the OS; no other device permissions are requested.
- **Limitations:** functionality depends entirely on network access and valid API keys — if a key is missing, tools degrade to whatever `Nexus_DEMO_MODE` fallback is configured for.
- **Risks:** API keys must not be committed to source control; CORS is currently open (`allow_origins=["*"]`) for demo convenience and should be scoped down for any non-hackathon deployment.

---

## Attribution

- **LLM access:** [OpenRouter](https://openrouter.ai/) (routes to hosted third-party models, e.g. Anthropic's Claude)
- **Math/science:** [Wolfram Alpha API](https://products.wolframalpha.com/api/)
- **Weather:** [OpenWeatherMap API](https://openweathermap.org/api)
- **Web search:** DuckDuckGo
- **Speech:** browser Web Speech API (production); `pyttsx3` and `SpeechRecognition` (local mode)
- **Libraries:** FastAPI, uvicorn, aiohttp, pydantic, python-dotenv, prometheus-client, websockets, pytest/pytest-asyncio (see `requirements.txt` / `requirements-local.txt` for the full pinned list)
- **Hosting:** Vercel (serverless functions + static hosting)

---

## Project Structure

```
AI-Voice-Intelligence-System---Nexus-/
├── api/
│   └── index.py            # Vercel serverless FastAPI entry point
├── frontend/                # Neural HUD UI (HTML/CSS/JS)
├── main.py                  # Local WebSocket + audio orchestrator
├── intents.py                # Intent classification and tool routing
├── resilient.py               # Circuit breaker / retry / bulkhead
├── config.py                  # Pydantic settings from environment
├── audio_engine.py             # Local microphone + TTS (not used on Vercel)
├── logger.py                  # Structured JSON logging
├── health.py / metrics.py       # Health probes and Prometheus metrics
├── requirements.txt             # Vercel / production dependencies
├── requirements-local.txt        # + audio/WebSocket deps for local dev
├── runtime.txt                  # Python version pin for Vercel (3.11)
├── vercel.json                  # Vercel routing/build configuration
├── start_server.py               # Local dev helper to launch a static file server
└── README.md / ARCHITECTURE.md
```

---

## Troubleshooting

| Issue | Fix |
|---|---|
| API returns 500 on Vercel | Check `OPENROUTER_API_KEY` is set in Vercel's Environment Variables |
| "Backend not reachable" | Serverless cold start — wait a few seconds and retry the first message |
| Voice not working | Use HTTPS (Vercel provides this automatically); grant microphone permission |
| WebSocket fails locally | Run `python main.py` first and confirm port 8765 is free |
| PyAudio install fails | Install the PortAudio system library first (see Requirements) |
| CORS errors locally | Use `vercel dev` or serve the frontend over HTTP, not `file://` |
