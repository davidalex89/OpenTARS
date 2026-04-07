# OpenTARS
A home assistant project based on the robot from Interstellar

# AskTARS

A locally-run personal assistant with a voice. You ask something in natural language; a few seconds later a spoken reply comes back. No cloud services. No API keys. Everything runs on the machine.

---

## How it works — the short version

Three separate processes talk to each other over HTTP:

| Process | Role |
|---------|------|
| **Ollama** | Large language model — produces the text reply |
| **Chatterbox** (`tts_server.py`) | Neural TTS — turns text into WAV audio |
| **AskTARS** (`main.py` / uvicorn) | Orchestration — prompt assembly, LLM calls, TTS pipeline, audio streaming |

AskTARS never imports the others as libraries. It only talks to them over HTTP. Environment variables (`OLLAMA_BASE_URL`, `CHATTERBOX_BASE_URL`, `ASKTARS_PORT`) override the default ports.

```mermaid
flowchart LR
  subgraph clients["Clients"]
    CL["Caller / trigger"]
    OV["Audio overlay"]
  end
  subgraph local["Local machine"]
    AT["AskTARS"]
    OL["Ollama"]
    CB["Chatterbox"]
  end
  CL -->|POST /ask| AT
  OV -->|GET /state| AT
  AT -->|POST /api/generate| OL
  AT -->|POST /v1/audio/speech| CB
```

---

## How I think — prompt layering

Every response starts as text, shaped by a layered prompt system before the LLM ever sees the question. The full pipeline looks like this:

```mermaid
flowchart TD
  IN["Incoming request"]
  EP{"Entry point"}
  ASK["Normal ask\nfull classification + context fetch"]
  IDLE["Idle prompt\nlighter context, fewer actions"]
  EVT["Event trigger\nfocused, short prompt"]
  L1["Layer 1 — Static base\nidentity · rules · personality"]
  L2["Layer 2 — Dynamic context\ninjected per request"]
  LLM["Ollama LLM"]
  L3["Layer 3 — Post-processing\nnormalize · typos · EXECUTE logic"]
  CB["Chatterbox TTS"]

  IN --> EP
  EP -->|question or command| ASK
  EP -->|no recent engagement| IDLE
  EP -->|external event| EVT
  ASK --> L1
  IDLE --> L1
  EVT --> L1
  L1 --> L2
  L2 --> LLM
  LLM --> L3
  L3 --> CB
```

### Entry points

The same pipeline runs for all three, but the prompt shape is different each time:

- **Normal ask** — a question or command comes in. Full request classification and context-fetching runs before the prompt is built.
- **Idle / boredom** — no recent engagement; TARS speaks up unprompted. Lighter context, different allowed actions.
- **Event trigger** — an external event arrives (e.g. a visitor). Focused, short prompt with a specific goal.

### Layer 1 — Static base

A fixed system prompt defining identity, personality, rules, and what TARS will and won't do. Identical for every request. Not published here.

### Layer 2 — Dynamic context (injected per request)

Before the LLM sees the question, `main.py` classifies it and fetches only the context that's relevant. Each injection is conditional — most questions trigger only a small subset.

```mermaid
flowchart LR
  Q["Incoming question"]
  CL["Classify request type"]

  Q --> CL

  CL -->|music-related| NP["Now-playing info"]
  CL -->|weather| WX["Weather data"]
  CL -->|lore / canon| LR["User-submitted lore\nRAG retrieval"]
  CL -->|diagnostics| DX["System diagnostics"]
  CL -->|coin / dice / wheel| RNG["Pre-computed result"]
  CL -->|always| PR["Recent prior requests"]
  CL -->|always| DT["Date / time context"]
  CL -->|request type| RC["Type-specific reminders\nand constraints"]

  NP --> PROMPT["Build prompt\nbase + injected context"]
  WX --> PROMPT
  LR --> PROMPT
  DX --> PROMPT
  RNG --> PROMPT
  PR --> PROMPT
  DT --> PROMPT
  RC --> PROMPT
```

### Layer 3 — Post-processing

After Ollama responds, the text is normalized before it reaches Chatterbox.

```mermaid
flowchart LR
  RT["Raw LLM text"]
  NM["Normalize\nnumbers · currency · time"]
  TF["Typo fixes"]
  EX{"Contains EXECUTE?"}
  ACT["Queue action\nfire if agreed"]
  DROP["Drop action\nrefusal detected"]
  OUT["Clean text → Chatterbox"]

  RT --> NM --> TF --> EX
  EX -->|yes + not refused| ACT --> OUT
  EX -->|yes + refused| DROP --> OUT
  EX -->|no| OUT
```

If the request was refused anywhere in the response, no action fires even if `[EXECUTE]` appears in the text.

---

## End-to-end: from question to voice

```mermaid
sequenceDiagram
  participant C as Caller
  participant A as AskTARS
  participant O as Ollama
  participant T as Chatterbox
  participant P as Audio player

  C->>A: POST /ask
  A->>O: generate prompt
  O-->>A: reply text
  A->>A: sanitize, post-process
  loop Per sentence chunk
    A->>T: POST /v1/audio/speech
    T-->>A: WAV bytes
  end
  A-->>C: JSON with text and audio pointers

  loop Poll
    P->>A: GET /state
    A-->>P: latest reply + stream URL
  end
  P->>A: GET /audio/stream/id
  A-->>P: chunked PCM stream
```

The caller (whatever triggered the question) and the audio player are **separate clients**. The caller gets JSON back. The audio player independently polls `/state` to find out what to play and where.

---

## What Chatterbox actually implements

Chatterbox is a small **FastAPI** app (`tts_server.py`) with two audio endpoints:

- **`POST /v1/audio/speech`** — returns a full WAV in the response body. This is what sentence-split streaming uses: one call per sentence chunk.
- **`POST /v1/audio/speech/stream`** — Chatterbox generates a full WAV then streams it back in chunks. Used only when `ASKTARS_TTS_SENTENCE_SPLIT=0`.

The request body requires at least `input` (text to speak). Optional `exaggeration` and `cfg_weight` tune delivery style. Model inference runs in a **thread pool** so the async server doesn't block during GPU work.

Each `generate` produces a full utterance WAV in memory — there is no true phoneme-level stream from the model. "Streaming" is either AskTARS scheduling many short generations (sentence-split mode) or Chatterbox chunking one long generation over HTTP (native mode).

---

## Streaming modes

`main.py` routes TTS through `_stream_tts_background`, which branches on `ASKTARS_TTS_SENTENCE_SPLIT` (default: on).

```mermaid
flowchart TD
  BG["_stream_tts_background"]
  Q{"ASKTARS_TTS_SENTENCE_SPLIT = 1?"}
  SS["_stream_tts_sentence_split"]
  NV["_stream_tts_chatterbox_native"]
  BG --> Q
  Q -->|yes - default| SS
  Q -->|no| NV
```

### Sentence-split adaptive streaming (default)

`_stream_tts_sentence_split` — **not** one Chatterbox call per reply.

1. **Split** the response text with `_split_sentences`: breaks on `.` `!` `?` (ellipses protected), merges short fragments, ensures each chunk ends with punctuation before TTS.
2. **Generate** each chunk via a separate `POST /v1/audio/speech`. Bad audio triggers `_audio_is_bad` and a retry.
3. **Adaptive pre-buffer**: accumulated playback duration must reach ≥ 1.2× estimated remaining generation time before the audio player hears anything. When threshold is met, AskTARS sends one WAV header plus buffered PCM to the stream queue; further sentence PCM is pushed as it's ready.
4. **Disk**: the full reply is also written to a single WAV file for archival / fallback.

```mermaid
flowchart TB
  TXT["Sanitized response text"]
  SP["Split sentences"]
  G1["Chunk 1 → POST /v1/audio/speech"]
  G2["Chunk 2 → POST /v1/audio/speech"]
  GN["Chunk N → POST /v1/audio/speech"]
  BUF["Buffer PCM until adaptive threshold"]
  Q["Stream queue → GET /audio/stream/id"]

  TXT --> SP
  SP --> G1
  SP --> G2
  SP --> GN
  G1 --> BUF
  G2 --> BUF
  GN --> BUF
  BUF --> Q
```

### Native Chatterbox streaming (sentence-split off)

When `ASKTARS_TTS_SENTENCE_SPLIT=0`, AskTARS sends the full text in one HTTP streaming `POST` to Chatterbox's `/audio/speech/stream` endpoint. Chatterbox returns chunks; AskTARS forwards them to the same queue. This skips the adaptive sentence-split loop entirely — different tradeoffs.

### Non-streaming

When TTS streaming is disabled, AskTARS calls `get_tts_wav` and serves static WAV files from `audio_out/` instead of `/audio/stream/...`.

---

## Startup

On startup, the `lifespan` hook POSTs a short phrase to Chatterbox so the first real request doesn't pay full model-load latency. If Chatterbox isn't up yet, AskTARS retries with backoff — non-fatal, but both services should be running before use.

---

## Configuration

| Variable | Purpose |
|----------|---------|
| `OLLAMA_BASE_URL` | Where AskTARS finds Ollama |
| `CHATTERBOX_BASE_URL` | Where AskTARS finds Chatterbox |
| `ASKTARS_PORT` | Port AskTARS listens on |
| `CHATTERBOX_N_CFM_TIMESTEPS` | Set in Chatterbox's environment; fewer steps = faster, more risk of audio artifacts |
| `ASKTARS_CHATTERBOX_STREAMING_QUALITY` | `fast` / `balanced` / `high` — affects buffering behaviour |
| `ASKTARS_TTS_STREAMING` | Master switch for streaming TTS |
| `ASKTARS_TTS_SENTENCE_SPLIT` | `1` = sentence-split adaptive mode (default). `0` = native Chatterbox stream. |

Defaults live in `config.py`. Chatterbox reads its own timestep defaults from its environment in `tts_server.py`.

---

## Known issues

- If Chatterbox is down, AskTARS can still return text; audio endpoints fail or hang depending on path.
- **Hallucinated words** at the end of a sentence can come from the TTS model, not from the LLM output. Mitigated by sentence boundaries and short-fragment merging; not fully preventable.
- **Incorrect contextual information** in a reply is an LLM failure, not a Chatterbox one. Chatterbox only speaks what it's given.

---

*No poetry. No tiny hats.*

