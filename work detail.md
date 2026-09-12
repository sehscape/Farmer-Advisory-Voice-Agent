# Farmer Advisory Voice Agent — Work Detail

---

## 📖 PROJECT EXPLAINED SIMPLY (read this first)

### What is this project, in one line?
A phone/computer app where a **farmer picks their language** (English, Hindi,
Punjabi or Marathi), **asks a question** by voice or typing, and the app **speaks
back real farming advice** — about their crop, the weather, and government schemes.

**Live link:** https://farmer-advisory-voice-agent.onrender.com (open it a couple of
minutes before a demo — the free server takes ~1 minute to wake up).

### Think of it like a small shop with 7 workers 🧑‍🌾
Each "worker" is one piece of code. A question passes down the line:

| # | Worker (the code file) | Its job, in plain words |
|---|---|---|
| 1 | 👂 **Ears** — `models/stt.py` (Whisper) | Listens to the voice and writes down the words |
| 2 | 🌐 **Translator (in)** — `models/translation.py` | Turns Hindi/Marathi/Punjabi words into English (the app thinks in English) |
| 3 | 🧭 **Router** — `agents/intent.py` + `models/llm.py` | Reads the question and decides *what* is being asked (crop? weather? scheme?). Knows farm words in English, Hindi, Marathi and Punjabi |
| 4 | 🗂️ **Scheme finder** — `tools/scheme_tool.py` + `rag/` | Searches real government-scheme documents for the answer |
| 5 | 🌦️ **Weather checker** — `tools/weather_tool.py` | Gets the live weather for the farmer's town |
| 6 | 🌱 **Crop expert** — `tools/crop_tool.py` | Looks up stage-by-stage advice for the crop |
| 7 | 🧠 **Brain** — `models/llm.py` (the AI) | Reads everything the workers found and writes one clear answer |
| → | 🌐 **Translator (out)** + 👄 **Mouth** — `translation.py` + `models/tts.py` | Turns the English answer back into the farmer's language and **speaks it aloud** |

The **manager** that decides which workers to call is a **LangChain agent**
(`agents/langchain_agent.py`) — it thinks step by step: "I need the weather… now
the scheme documents… now I can answer." If it ever gets stuck, a simpler
backup manager (`agents/orchestrator.py`) takes over. The **shop counter** the
farmer uses (the screen) is `ui/gradio_app.py`, and every word on it comes from
`ui/i18n.py` in all four languages.

### The journey of one question (start to finish)
```
Farmer picks a language  →  speaks, taps stop  →  Ears write it down (in that language)
   →  Understanding: what is asked? is anything missing?
        (missing crop age / village / crop → ask back out loud, then continue)
   →  LangChain agent picks the right workers
   →  they fetch crop + weather + scheme facts
   →  Brain writes the advice IN THE FARMER'S LANGUAGE  →  speak it out loud 🔊
```

### The golden rule of this project 🏅
**The AI is never allowed to make things up.** Scheme details come from real
documents, weather comes from a real weather service, crop advice comes from a
fixed knowledge file. The AI only *phrases* what the workers found. If there's no
reliable info, the app says "I don't have enough information" instead of guessing.

### What runs on your laptop (no GPU) vs. what needs a GPU
| Part | On your laptop? | Note |
|---|---|---|
| Screen in English / Hindi / Punjabi / Marathi | ✅ Yes | Pick at the top right; works on Render too |
| Voice questions in all 4 languages | ✅ Yes | Hindi is heard well; Marathi/Punjabi transcripts are rough, but the router still finds the crop, place and topic |
| Ears, Router, LangChain agent, Crop, Weather, Scheme search | ✅ Yes | All work fully |
| Voice output | ✅ Yes (gTTS) | Real voices, needs internet |
| The AI brain | ✅ Small version | A small model runs on CPU (~1 min/answer). The big 8B model needs a GPU |
| Answer in the regional language | ✅ With a free Groq key | Groq's cloud does the heavy listening + language work (laptop **and** Render). Without the key: English answers (the local translator is ~4 GB / GPU) |

### How to run it on your computer
```bash
cd multilingual-agri-agent
venv\Scripts\activate            # turn on the virtual environment
python scripts/build_scheme_index.py   # one time — prepares the scheme search
python app.py                    # opens the app at http://localhost:7860
```
Three switches live in the `.env` file:
- `GROQ_API_KEY=gsk_…` → **voice + answers in Hindi / Punjabi / Marathi** (free key, DEPLOY.md section C). Empty → off.
- `USE_STUB_LLM=true` → **fast** answers (simple, instant). `false` → **real AI** brain (slower on CPU).
- `USE_STUB_TRANSLATION=true` → without Groq, the voice speaks **English**. `false` → local translator (heavy model / GPU).

### Where things live (folder map)
```
app/agents/   → the router, the manager, and answer-writing
app/tools/    → crop, weather, and scheme "workers"
app/models/   → ears (STT), brain (LLM), voice (TTS), translator
app/rag/      → the government-scheme document search
app/ui/       → the screen the farmer sees
data/         → crop facts + scheme documents
scripts/      → test files that prove each part works
```

### Is it finished?
Yes — **all 15 phases are complete**, plus the LangChain agent, the 4-language
screen, voice + answers in all 4 languages on the live Render site (with a free
Groq key), and spoken follow-up questions when something is missing. Everything
is tested. The detailed, technical phase-by-phase log is below.

---

## Project Overview

A multilingual voice-first agricultural assistant for farmers in India.
Farmers speak in their regional language (Hindi, Marathi, or Punjabi) and receive
actionable farming advice in the same language via voice output.

**Target users:** Farmers in Hindi, Marathi, and Punjabi speaking regions of India
**Deployment target:** Hugging Face Spaces (Gradio)
**Total phases:** 15

---

## Architecture (English-Pivot Pipeline)

```
Farmer speaks regional language
        ↓
  Whisper STT → regional text (original_text)
        ↓
  IndicTrans2 → English text (english_text)
        ↓
  Intent Extraction (LLM) → crop, stage, intent flags
        ↓
  Tool Calls → crop knowledge / weather / government schemes
        ↓
  LLM Answer Generation → English answer (english_answer)
        ↓
  IndicTrans2 → regional text (regional_answer)
        ↓
  Indic TTS → regional audio output
```

All internal AI processing happens in English.
Regional language appears only at the input (STT) and output (TTS) boundaries.

---

## COMPLETED PHASES

---

### Phase 1 — Project Setup
**Status:** Complete

**What was built:**
- Full project folder structure (`app/`, `data/`, `tests/`, `scripts/`, `notebooks/`)
- Central configuration (`app/config.py`) — all settings controlled via `.env` file
- `AgentState` dataclass (`app/agents/state.py`) — single object that carries all data through the pipeline (transcription, English translation, intent, tool outputs, final answer)
- Prompt templates (`app/agents/prompts.py`) — intent extraction prompt and final answer prompt
- Gradio UI skeleton (`app/ui/gradio_app.py`) — basic layout with microphone input
- Entry points — `app.py` (Hugging Face Spaces) and `app/main.py` (local dev)
- `.env` / `.env.example` — environment variable configuration
- `.gitignore` — excludes secrets, virtual environment, model weights
- `requirements.txt` — all dependencies listed

**Key design decision:** English-pivot architecture — the LLM never sees regional language text. IndicTrans2 handles all translation at the boundaries.

---

### Phase 2 — Speech-to-Text (Whisper STT)
**Status:** Complete

**What was built:**
- `app/models/stt.py` — Whisper-based speech-to-text
- `WhisperSTT` class — loads OpenAI Whisper model, transcribes audio
- **Auto language detection** — detects Hindi, Marathi, Punjabi, or English automatically from the speech itself (no manual selection needed)
- `StubSTT` — returns hardcoded Hindi/Marathi/Punjabi samples for offline testing
- `app/utils/audio.py` — converts browser audio formats (webm/ogg) to 16kHz mono WAV for Whisper
- Test scripts: `tests/test_stt.py` and `scripts/test_stt_phase2.py`

**Models used:**
- Local dev (CPU): `openai/whisper-tiny`
- HF Spaces (GPU): `openai/whisper-large-v3`

**How it works:**
1. User records voice via browser microphone
2. Audio converted to WAV format
3. Whisper transcribes the speech AND detects which language was spoken
4. Returns: `{ "text": "मेरी गेहूं...", "language": "hi" }`

---

### Phase 3 — Translation (IndicTrans2)
**Status:** Complete

**What was built:**
- `app/models/translation.py` — IndicTrans2 translation (regional ↔ English)
- `IndicTrans2Translator` — loads AI4Bharat's IndicTrans2 models (1B parameters each direction)
- `PassthroughTranslator` — stub that echoes text with `[STUB-EN]` tag for fast local dev
- `app/models/indic_processor.py` — pure-Python text normalizer

**Problem solved:** The official `indictranstoolkit` library requires Microsoft Visual C++ Build Tools to compile on Windows. A pure-Python replacement was written using `indic-nlp-library` and `sacremoses` that provides the same interface without needing MSVC.

**Two translation directions:**
- Input boundary: regional language → English (`ai4bharat/indictrans2-indic-en-1B`)
- Output boundary: English → regional language (`ai4bharat/indictrans2-en-indic-1B`)

**Languages supported:**
| Language | Flores-200 Code |
|---|---|
| Hindi | `hin_Deva` |
| Marathi | `mar_Deva` |
| Punjabi | `pan_Guru` |
| English | `eng_Latn` |

**Config flag:** `USE_STUB_TRANSLATION=true` in `.env` skips the 4GB model download for fast local dev. Set to `false` to use real IndicTrans2.

**Gradio UI update:** Added "English translation" output box showing the translated text.

---

### Phase 4 — Intent Extraction + LLM Answer Generation
**Status:** Complete

**What was built:**
- `app/agents/intent.py` — extracts structured intent from the English query using LLM
- `app/agents/answering.py` — generates a structured English answer using LLM
- Updated `app/models/llm.py` — added context-aware StubLLM

**Intent extraction:**
The LLM reads the English-translated query and returns a JSON object:
```json
{
  "intent": "crop_advice",
  "crop": "wheat",
  "crop_stage_days": 40,
  "location": null,
  "needs_weather": true,
  "needs_scheme": false,
  "needs_crop_info": true
}
```

These fields populate `AgentState` and control which tools are called next.

**Intent types supported:**
`crop_advice`, `weather`, `government_scheme`, `irrigation`, `fertilizer`, `pest_or_disease`, `general_farming`, `multiple`, `unknown`

**Answer generation:**
LLM receives the farmer's English query + tool outputs (crop knowledge, weather data, scheme documents) and produces a structured answer:
```
Situation:
[1-2 sentences about the farmer's situation]

What you should do:
1. ...
2. ...
3. ...

Important:
[Key cautions]
```

**LLM options:**
- `HuggingFaceInferenceAPILLM` — calls HF Inference API (needs HF token)
- `HuggingFaceLocalLLM` — loads model locally (needs GPU, not yet implemented)
- `StubLLM` — builds crop-specific answers from injected context (works offline)

**Config flag:** `USE_STUB_LLM=true` in `.env` uses StubLLM without needing an HF token.

---

### Phase 5 — Crop Knowledge Tool
**Status:** Complete

**What was built:**
- `app/tools/crop_tool.py` — crop knowledge lookup tool
- `data/crops/` — JSON knowledge base for 6 crops

**Crops in knowledge base:**
| Crop | Local names recognized |
|---|---|
| Wheat | गेहूं, ਕਣਕ, गव्हू |
| Rice | धान, चावल, ਝੋਨਾ, भात |
| Onion | प्याज, कांदा, ਪਿਆਜ਼ |
| Tomato | टमाटर, ਟਮਾਟਰ, टोमॅटो |
| Cotton | कपास, ਕਪਾਹ, कापूस |
| Maize | मक्का, मकई, ਮੱਕੀ |

**How it works:**
1. Reads `state.crop` and `state.crop_stage_days` from AgentState (set by Phase 4)
2. Resolves crop name — handles English, Hindi, Marathi, Punjabi names
3. Finds the exact growth stage matching the crop's current age in days
4. Returns stage-specific advisory:
   - Current growth stage name and description
   - Irrigation schedule for this stage
   - Fertilizer recommendations for this stage
   - Pests and diseases to watch for at this stage
   - Practical tips

**Example output for Wheat at 40 days (Tillering stage):**
```
Current Stage: Tillering (days 26–50)
Irrigation  : Second irrigation at 40–45 days after sowing
Fertilizer  : Apply 25 kg Urea per acre with second irrigation
Watch for   : Aphids, Powdery mildew, Weed competition
Tips        : Remove weeds before 35 days
```

**Pipeline position:** Crop tool runs BEFORE the LLM generates the answer, so the LLM has real knowledge to work with instead of hallucinating advice.

---

## COMPLETED PHASES (continued)

---

### Phase 6 — Weather Tool
**Status:** Complete

**What was built:**
- `app/tools/weather_tool.py` — full weather data tool
- `scripts/test_weather_phase6.py` — smoke test (5 test cases, all pass)

**How it works:**
1. Farmer's location name (e.g. "Pune", "Ludhiana") is geocoded to lat/lon using the **Open-Meteo Geocoding API** (free, no API key)
2. **Open-Meteo Forecast API** is called to fetch:
   - Current conditions: temperature, feels-like temp, humidity, precipitation, wind speed, weather condition
   - 3-day daily forecast: min/max temp, rainfall amount, rain probability, wind speed
3. WMO weather codes are translated to human-readable descriptions (e.g. "Thunderstorm", "Light drizzle")
4. **Farming-specific advisories** are generated by rule:
   - Extreme heat (>38°C) → avoid pesticide spraying, irrigate
   - Near-frost (<5°C) → protect sensitive crops
   - High humidity (>85%) → fungal disease risk warning
   - Heavy rain (>20mm) → delay fertilizer application
   - Thunderstorm → keep people and equipment away from fields
5. Returns both raw data dict (for LLM reasoning) and a formatted English summary string

**Failure handling:** Unknown location, API timeout, or missing location all return clean error messages — no crashes.

**No new dependencies required** — `requests` was already in `requirements.txt`.

---

### Phase 7 — Government Scheme RAG (PDF → FAISS)
**Status:** Complete

**What was built:**
- `data/schemes/raw/` — 4 government scheme text files with real scheme information
- `app/rag/scheme_rag.py` — complete RAG pipeline
- `app/tools/scheme_tool.py` — agent-facing tool wrapper with lazy loading
- `scripts/build_scheme_index.py` — one-time index builder script
- `scripts/test_scheme_phase7.py` — smoke test (5 queries, all pass)
- `data/vectorstore/schemes.faiss` + `schemes_meta.json` — built FAISS index (15 chunks)

**Schemes in knowledge base:**
| Scheme | What it covers |
|---|---|
| PM-KISAN | Rs 6,000/year income support — eligibility, how to apply, documents needed |
| PMFBY | Crop insurance — crops covered, premium rates, claim process |
| Kisan Credit Card (KCC) | Agricultural loans — interest rates (4–7%), eligibility, insurance |
| Soil Health Card | Soil nutrient testing — 12 parameters tested, how to get the card |

**RAG Pipeline:**
1. **Load** — reads `.txt` and `.pdf` files from `data/schemes/raw/`
2. **Chunk** — splits text into overlapping chunks (1000 chars, 150 overlap) on natural boundaries (newlines, periods)
3. **Embed** — `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` (local dev) / `BAAI/bge-m3` (HF Spaces)
4. **Index** — FAISS `IndexFlatIP` with L2-normalized vectors (cosine similarity)
5. **Save/Load** — index persists to `data/vectorstore/schemes.faiss` + metadata JSON
6. **Query** — embed question → FAISS search → return top-K chunks with source and relevance score

**Config flag:** `USE_STUB_RAG=false` in `.env`. Set to `true` to skip FAISS load during fast local dev.

**Dependency fixes required:**
- `faiss-cpu` upgraded from `==1.8.0` to `>=1.9.0` (NumPy 2.x compatibility)
- `sentence-transformers` upgraded from `==3.0.1` to `>=3.3.0` (compatible with Gradio's huggingface-hub)

**To rebuild index after adding new scheme files:**
```
python scripts/build_scheme_index.py
```

---

### Phase 8 — Agent Orchestrator (Tool Orchestration)
**Status:** Complete

**What was built:**
- `app/agents/orchestrator.py` — sequential tool orchestrator
- `app/agents/answering.py` — updated weather/scheme formatters (removed Phase placeholder text)
- `app/ui/gradio_app.py` — wired orchestrator in place of manual Steps 4–7
- `scripts/test_agent_phase8.py` — smoke test (5 test cases, all pass)

**How it works:**
The orchestrator takes `AgentState` after intent extraction and decides which tools to call based on the intent flags set by the LLM:

```
AgentState (with intent flags)
    ├─ needs_crop_info=True  → Crop knowledge tool
    ├─ needs_weather=True    → Weather tool (Open-Meteo)
    ├─ needs_scheme=True     → Scheme RAG (FAISS)
    └─ always               → LLM answer generation
```

**Design decision:** Sequential orchestration (not ReAct loop) because:
- Intent extraction already determines which tools are needed
- Sequential is deterministic, fast, and avoids hallucination from looping
- Multi-intent queries (e.g. "crop + weather + scheme") are all handled in one pass

**Test cases verified:**
1. Crop advice (wheat at 40 days) — crop tool fires
2. Weather query with location (Pune) — crop + weather tools fire
3. Government scheme query — scheme RAG fires
4. Multi-intent (cotton + Nagpur + schemes) — all 3 tools fire
5. Weather without location — graceful skip with informative message

---

### UI Redesign
**Status:** Complete

**What was changed:**
- `app/ui/gradio_app.py` — full redesign for farmer-friendliness
- `app/main.py` — CSS moved to `launch()` (Gradio 6 compatibility); `share=True` enabled

**Key improvements over old UI:**

| Before | After |
|---|---|
| 7 stacked output boxes | 3 clean tabs (Answer / Transcript / Details) |
| No visual hierarchy | Answer box is the hero — large, green-tinted |
| No tool feedback | Colour-coded pills: 🌱 Crop · 🌤️ Weather · 📋 Scheme |
| No examples | Collapsible accordion with 4 examples each in Hindi, Marathi, Punjabi |
| Plain Gradio default | Agriculture-green theme with custom CSS |
| Language buried in output | Language badge prominently in input column |
| Dev trace always visible | Trace hidden in Details tab (dev mode only) |

**Tab structure:**
- **💬 Answer** — main answer box + tool badges + voice output placeholder
- **📝 Transcript** — what the farmer said + English translation
- **🔍 Details** — intent extraction result + pipeline trace (dev mode)

**Example queries added (in UI accordion):**
- 4 Hindi examples (wheat fertilizer, rain/irrigation, PM-KISAN, tomato yellowing)
- 4 Marathi examples (cotton fertilizer, Nashik weather, scheme documents, onion yellowing)
- 4 Punjabi examples (wheat fertilizer, Ludhiana rain, KCC application, rice pests)

**Public sharing:** `share=True` — Gradio prints a public URL on startup (valid for 72 hours).

---

### Phase 1–8 Hardening (spec-alignment pass)
**Status:** Complete

**Why:** Lock Phases 1–8 to a "runs perfectly and matches the master spec" state
before starting Phase 9. Fixes real bugs and closes gaps against the spec's
anti-hallucination / grounding requirements (spec §12, §19–22, §35, §33).

**What was changed:**

- **UTF-8 console fix (Windows bug).** Test scripts crashed with
  `UnicodeEncodeError` when printing the `→` character on Windows (cp1252 console).
  Added `enable_utf8_console()` in `app/utils/logging.py` and call it from
  `app/main.py`, `scripts/test_weather_phase6.py`, `scripts/test_scheme_phase7.py`,
  `scripts/test_agent_phase8.py`, and `scripts/build_scheme_index.py`.

- **StubLLM answer generation rewritten** (`app/models/llm.py`):
  - Fixed the "Your your crop crop" text bug.
  - The stub now composes the answer from **all** tool outputs — crop **and**
    weather **and** scheme — instead of only the crop block. Weather advisories
    become action steps; scheme facts get their own section plus a `Source:`
    citation line (spec §35). Handles scheme-only / weather-only queries cleanly
    (no more empty answer sections).
  - Never presents missing data as fact: placeholder blocks are detected and
    skipped.

- **StubLLM intent extraction improved** (`app/models/llm.py`):
  - Extracts **location** from a known-city list (previously always `null`).
  - Only sets `crop` when a real crop is mentioned, and sets `needs_*` flags
    accurately, so a weather-only question no longer forces the crop/RAG tools
    (spec §22). Chooses `multiple` when more than one category applies.

- **RAG "insufficient information" guardrail** (`app/config.py`,
  `app/tools/scheme_tool.py`): added `RAG_MIN_SCORE` (default `0.30`). Retrieved
  chunks below the threshold are dropped; if none qualify the tool returns a clear
  "documents do not contain sufficient information" message instead of presenting
  weak matches as fact (spec §12).
  - **Measured finding:** a relevance threshold alone cannot reject a
    *topically-related but unanswerable* query (e.g. "what new scheme comes next
    month?" scores ~0.50, as high as real scheme queries). That refusal is a
    faithfulness judgement made by the **real** LLM via `FINAL_ANSWER_PROMPT`; the
    stub is only guaranteed to stay retrieval-grounded.

- **New Phase 4→8 integration test** (`scripts/test_pipeline_phase4to8.py`):
  drives the full dev pipeline (English query → intent extraction → orchestrator →
  answer) and asserts correct tool **selection** (spec §22), location/crop/stage
  extraction, and that scheme answers are always retrieval-grounded. 5/5 pass.

**Test results after hardening (all green):**
| Suite | Result |
|---|---|
| `scripts/test_pipeline_phase4to8.py` (new) | 5/5 PASS |
| `scripts/test_agent_phase8.py` | 5/5 PASS |
| `scripts/test_weather_phase6.py` | ALL PASS |
| `scripts/test_scheme_phase7.py` | ALL PASS |
| `pytest` | 13 passed |

---

### Phase 9 — End-to-End English Pipeline Test
**Status:** Complete

**What was built:**
- `scripts/test_e2e_phase9.py` — the definitive dev-mode integration test. Drives
  real English queries through the full chain
  (`english_text → intent extraction → orchestrator → answer`) and checks:
  - **Part A — intent coverage (9/9):** every intent type is produced and routed
    correctly — `crop_advice`, `weather`, `government_scheme`, `fertilizer`,
    `irrigation`, `pest_or_disease`, `multiple`, `general_farming`, `unknown` —
    and the agent never calls unnecessary tools (spec §22).
  - **Part B — edge cases:** missing location (graceful weather skip),
    unrecognised crop (no crash, honest answer), and empty input
    (`intent=unknown`, no tools). Exceptions are recorded as failures so the
    "never crashes" guarantee is itself under test (spec §36).
  - **Part C — tool safety:** out-of-KB crop returns "not in the knowledge base";
    off-topic scheme query (score ~0.17) trips the `RAG_MIN_SCORE` guardrail and
    returns "insufficient information" instead of fabricating (spec §12/§33 TEST 5).
  - **Result: 14/14 checks pass.**

**Supporting changes to `app/models/llm.py` (StubLLM intent classifier):**
- Emits the fine-grained intent labels the spec lists (`fertilizer`, `irrigation`,
  `pest_or_disease`, `unknown`) via keyword groups, not just the coarse set.
- Robust query parsing: captures only the `Query:` line, so an empty query no
  longer accidentally matches the prompt template's own words.
- Tool-routing flags derived cleanly from keyword groups (weather / scheme /
  crop-topic), with `multiple` chosen when more than one category applies.

**Note:** STT + output translation are still stubs, so this phase validates the
*English* pipeline end-to-end (the internal reasoning core). The regional-language
voice boundaries are Phase 10.

**Full regression after Phase 9 — all green:**
| Suite | Result |
|---|---|
| `test_e2e_phase9.py` (new) | 14/14 PASS |
| `test_pipeline_phase4to8.py` | 5/5 PASS |
| `test_agent_phase8.py` | 5/5 PASS |
| `test_weather_phase6.py` | ALL PASS |
| `test_scheme_phase7.py` | ALL PASS |
| `pytest` | 13 passed |

---

### Phase 10 — IndicTrans2 Output + Indic TTS
**Status:** Complete

**What was built:**
- **Output translation** (`app/models/translation.py`): `translate_from_english`
  (English → Hindi/Marathi/Punjabi via IndicTrans2 `en-indic-1B`) was already
  implemented — now wired into the pipeline. Stub path speaks the English answer.
- **TTS engines** (`app/models/tts.py`), selected by `TTS_ENGINE` in `.env`:
  - `GttsTTS` — lightweight, CPU-friendly, needs internet. **Dev default.**
    Produces real Hindi/Marathi/Punjabi/English speech (`.mp3`).
  - `IndicParlerTTS` — production `ai4bharat/indic-parler-tts` (large, GPU).
    Fully implemented; lazy-loads and guards the missing dependency with a clear
    install hint (`pip install git+https://github.com/ai4bharat/indic-parler-tts`).
  - `StubTTS` — returns no audio (text answer still shown).
  - `generate(text, language)` now returns a **file path** (or `None`) so Gradio
    can play it directly.
- **Output stage wired** (`app/ui/gradio_app.py`, `_run_output_stage`): after the
  answer is generated it translates English → regional (when real translation is
  on) and synthesizes voice; the audio **auto-plays** in the UI's Answer tab.
  Every failure path is graceful — TTS/translation errors never break the text
  answer.
- **Config:** `TTS_ENGINE` (default `gtts`). **Requirements:** added `gTTS`.

**New test:** `scripts/test_tts_phase10.py` — gTTS audio for en/hi/mr/pa, engine
factory, StubTTS, the production-guard, and the full translate+TTS output stage.
**Result: 11/11 checks pass.**

**Dev vs production note:**
- On this CPU machine (dev), `TTS_ENGINE=gtts` gives real regional speech with no
  large download. To hear the answer in the *regional language* (not English),
  set `USE_STUB_TRANSLATION=false` (one-time IndicTrans2 `en-indic-1B` ~4 GB).
- On HF Spaces (GPU), set `TTS_ENGINE=parler` for production-quality Indic voices.

**Full regression after Phase 10 — all green:**
| Suite | Result |
|---|---|
| `test_tts_phase10.py` (new) | 11/11 PASS |
| `test_e2e_phase9.py` | 14/14 PASS |
| `test_pipeline_phase4to8.py` | 5/5 PASS |
| `test_agent_phase8.py` | 5/5 PASS |
| `test_weather_phase6.py` | ALL PASS |
| `test_scheme_phase7.py` | ALL PASS |
| `pytest` | 13 passed |

---

### Phase 11 — Full Gradio Pipeline Wiring
**Status:** Complete

- The UI runs the full chain end-to-end (STT → translate → intent → orchestrate →
  answer → translate-out → TTS) and the **voice answer auto-plays** in the Answer
  tab. Stub/pending labels removed (audio label is now live).
- Added **total response-time** tracking (spec §38): `_run_pipeline` records the
  end-to-end time into `state.latency["total"]` and appends `[Total] response
  time: X.Xs` to the pipeline trace (Details tab, dev mode).

---

### Phase 12 — Testing & Evaluation
**Status:** Complete

- `scripts/evaluate.py` — a reproducible metrics harness (spec §39) that scores:
  intent-classification accuracy, agent tool-selection accuracy (+ unnecessary-call
  count), RAG top-1 source accuracy & mean relevance, off-topic refusal rate, and
  answer quality (grounded / actionable / cited). Doubles as a CI gate (exits
  non-zero below sane floors).
- Baseline (dev/stub): intent **100%**, tool-selection **100%**, RAG top-1 **100%**
  (mean 0.60), off-topic refusal **100%**, answers actionable & cited **100%** →
  **HEALTHY**.
- Fixed an issue surfaced by the eval: the bare word "crop" was pulling the
  crop-knowledge tool into scheme queries like "crop insurance"; the classifier now
  triggers on a named crop or a specific topic word instead.

---

### Phase 13 — Optimization
**Status:** Complete

- **GPU auto-detect** (`app/utils/device.py`): `get_device()` / `get_dtype()` pick
  CUDA + float16 when a GPU is present, else CPU + float32. Wired into every model
  singleton (STT/translation/LLM/TTS), so the same code runs fast on the Spaces GPU
  with **zero change** on CPU.
- **Caching**: geocoding results are `lru_cache`-d (a place's coordinates are
  stable) so repeat weather lookups skip the network. Models remain singletons
  (loaded once — spec §30).
- Quantization / dtype documented for the production LLM (4-bit) in the README.

---

### Phase 14 — Hugging Face Spaces Deployment (prep)
**Status:** Complete

- **`README.md`** now begins with the Spaces YAML header (`sdk: gradio`,
  `sdk_version`, `app_file: app.py`, emoji, license).
- **`packages.txt`** added with `ffmpeg` (browser mic audio decoding on Linux).
- **`.env.example`** rewritten with clear DEV vs PRODUCTION profiles and all flags
  (`USE_STUB_*`, `TTS_ENGINE`, `RAG_MIN_SCORE`).
- **`requirements.txt`** documents the production Parler install.
- README has a step-by-step Spaces deployment section (GPU Space, secrets, model
  switch). *Actual push to a live Space is the only remaining manual step.*

---

### Phase 15 — Documentation
**Status:** Complete

- README updated: status table (all phases ✅), voice/TTS engine details, GPU
  auto-detect note, full Testing + Evaluation sections, Deployment guide,
  Limitations, and Future Improvements.
- Architecture decision (sequential orchestrator vs LangChain ReAct) documented.

---

### Local (No-GPU) Completion — real open-source LLM on CPU
**Status:** Complete

To satisfy the brief's "open-source LLM" on a laptop **without a GPU**:
- Implemented `HuggingFaceLocalLLM` (`app/models/llm.py`) — runs a small ungated
  instruct model on CPU via transformers (chat template, greedy decoding, token
  cap for responsiveness). Default `LOCAL_LLM_MODEL_ID=Qwen/Qwen2.5-0.5B-Instruct`
  (~1 GB, Apache-2.0); bump to `Qwen2.5-1.5B-Instruct` for better quality.
- **Intent routing** now uses a dedicated fast rule-based router
  (`_get_router_llm`), separate from the answer LLM — so tool selection stays
  instant and 100% reliable even when the answer model is a tiny CPU model.
- Verified end-to-end on CPU (`scripts/test_local_llm.py`, 2/2 pass): ~50–70 s per
  answer, grounded in the retrieved tool context.
- **STT** set to `whisper-small` (matches the brief; better Indic accuracy than
  tiny). Turn on the real brain with `USE_STUB_LLM=false` + `USE_HF_INFERENCE_API=false`.

**Known CPU tradeoffs (report to mentor):** small model → occasional factual
slips; ~1 min/answer on CPU. A GPU (or the 8B model via HF Inference API) removes
both. TTS speaks English until IndicTrans2 translation is enabled (GPU-friendly).

---

### Deployment — Colab (full) + Render Lite (free 24/7)
**Status:** Complete (ready to deploy; the final click-through is manual)

**Why two targets:** mid-2026 Hugging Face Spaces started requiring a **paid plan**
for Gradio apps, and Render's free tier is only **512 MB RAM** — too small for
Whisper + the embedding model. So:

- **Google Colab** (`notebooks/run_full_app_colab.ipynb`) — runs the **full** app
  (mic, real LLM, semantic search, voice) with a `gradio.live` link. Live only
  while the tab is open. For demos / viva.

- **Render "Lite"** (`render.yaml`, free tier, 24/7) — a new **`LITE_MODE`** that
  strips every heavy dependency so the app fits in ~350 MB:
  - `LITE_MODE=true` (in `app/config.py`) force-sets: keyword scheme search,
    rule-based answers, stub translation, gTTS, no dev banner.
  - **`app/rag/keyword_retriever.py`** — a pure-Python BM25 ranker over the same
    scheme `.txt` files (no torch / transformers / faiss). Two tweaks for this
    small structured corpus: a **name/header boost** (so "apply for PM-KISAN"
    ranks the PM-KISAN file above files that merely say "kisan" often) and a
    **≥2-matching-terms guard** (a single incidental rare-word match → "insufficient
    information", never fabrication).
  - **`requirements-lite.txt`** — 5 packages only (gradio, requests, numpy,
    python-dotenv, gTTS). No torch.
  - `app/utils/device.py` no longer imports torch unless it's installed;
    `_get_llm`/`_get_tts` skip device probing for stub/gtts. Verified: with
    torch/transformers/faiss *blocked*, the full typed pipeline still runs.
  - `app/main.py` reads Render's `PORT` and skips the share tunnel on any host
    (`SPACE_ID` / `RENDER` / `PORT` set).
  - UI hides the microphone in lite mode with a "voice available in full version"
    note; typed input drives everything.

- **`DEPLOY.md`** — click-by-click for both paths.

**New test:** `scripts/test_lite_mode.py` — forces `LITE_MODE`, checks the keyword
retriever picks the right scheme (4/4), refuses off-topic (3/3), runs the full
typed pipeline, and asserts **no heavy ML library is imported**. All pass.

**Regression after lite work:** pytest 13/13, Phase 9 14/14, Phase 10 11/11,
evaluate HEALTHY — all still green in normal (FAISS) mode.

**Deployment fixes along the way:** Render first defaulted to Python 3.13 (no
wheel for the pinned numpy → source build → `No module named 'pkg_resources'`) —
fixed with `.python-version` = 3.11.9 and loose lite pins. Then a stale
`gradio==4.44.0` pin paired with a too-new `huggingface_hub` (`HfFolder` import
error) — fixed by pinning `gradio==6.26.0`, the version the code runs on. The
unused `openai-whisper` (STT uses transformers) and `langchain` pins were dropped
from `requirements.txt` at that point.

---

### UI Redesign — dark-first editorial theme
**Status:** Complete (commit `717d1f7`)

Replaced the tabbed green theme with an editorial layout: near-black canvas (with
a full light palette following the viewer's system setting), hairline rules
instead of boxes, letterspaced section labels, a wheat-gold accent, a large hero
headline, a two-panel console (ask on the left, answer on the right, stacking on
narrow screens), tool-status chips, click-to-fill example questions, and a
collapsible "Transcript & pipeline" diagnostics panel. Theme and CSS live in
`app/ui/gradio_app.py` (`build_theme`, `_CSS`).

---

### LangChain Agent — default agent engine
**Status:** Complete · closes the brief's "LangChain agents with tool-calling"

- **`app/agents/langchain_agent.py`** — the three farm tools are LangChain `Tool`s
  (`crop_knowledge`, `weather_forecast`, `government_schemes`) and a ReAct
  `AgentExecutor` (`create_react_agent`) runs the loop: at each step the model
  picks the next tool or finishes; LangChain executes it and feeds back the
  observation. The final answer is then written from what the agent gathered, so
  the output format is identical to the sequential engine.
- **`app/models/langchain_llm.py`** — adapter exposing any project LLM
  (StubLLM / local Qwen / HF API) to LangChain.
- **Who decides on CPU:** `StubLLM._react_step` — a deterministic policy that
  speaks LangChain's exact ReAct text protocol, choosing tools from the same
  rule-based router. The AgentExecutor, tool calls and parsing are real
  LangChain. With a real LLM configured (Colab notebook: Qwen 1.5B; GPU: Llama /
  Gemma) the model makes the decisions — no code change.
- **Resilience:** if the loop errors, stalls, or can't parse the model's output,
  the deterministic orchestrator takes over — never a broken reply.
- **Config:** `AGENT_BACKEND=langchain` (default) | `sequential`. If LangChain
  isn't installed the app falls back automatically.
- **Render:** LangChain added to `requirements-lite.txt`. Measured with the heavy
  ML libraries absent (as on Render): 143 MB → 166 MB of the 512 MB limit.
  (LangChain's optional `transformers` import is skipped when it isn't installed.)

**New test:** `scripts/test_langchain_agent.py` — exact tool set for all 9 intents,
answers for each, fallback when the model ignores the ReAct format or loops
forever, and that the UI uses LangChain by default. **34/34 pass.**

---

### 4-Language Interface + Voice Questions in Every Language
**Status:** Complete

**The screen:** a picker in the top bar — `🇬🇧 English` · `🇮🇳 Hindi` ·
`ਪੰਜਾਬੀ Punjabi` · `मराठी Marathi` — switches the whole interface: brand, headline,
instructions, section labels, placeholders, the Ask button, the language
readout, tool-status chips, the audio label, diagnostics, footer, and every
error/help message. 47 strings × 4 languages live in **`app/ui/i18n.py`**. A
switch re-renders 28 components; the page is tagged `lang="hi|pa|mr"` so the CSS
relaxes the Latin-style letterspacing that would break Devanagari/Gurmukhi, and
Noto Sans Devanagari/Gurmukhi were added to the theme fonts.

**Voice questions:**
- Whisper is now **forced to the chosen language** (`transcribe(language=…)`)
  instead of guessing — auto-detect used to return "unknown" for Hindi.
- Without the 4 GB translator, regional speech reaches the English agent via
  **Whisper's own speech→English translation** (`transcribe(translate=True)`)
  **plus the original transcript** (`AgentState.routing_text()`).
- **Finding (tested):** whisper-small's translation works for Hindi but fails for
  Marathi (it just sounds the words out) and Punjabi ("My wheat is 40 days old"
  came out as "My wife is 40 days old"). The native transcripts still contain the
  key words, so the router gained a **Hindi/Marathi/Punjabi farm vocabulary**
  (`StubLLM._REG_*`): weather/scheme/fertiliser/pest/irrigation words, crop names
  in all three scripts (incl. Punjabi written in Devanagari, which Whisper often
  produces), 22 city names, and spoken crop ages (चालीस / चाळीस / ਚਾਲੀ दिन → 40 days).
  A place named with nothing else matching is treated as a weather question.
- Result: spoken questions in **all four languages** reach the right tool.

**Scope (decided):** typed questions are English — typed Hindi/Punjabi/Marathi
gets a polite message, in that language, asking for English (unless IndicTrans2
is configured, in which case it's translated). Answers and the spoken reply are
English without IndicTrans2; a note under "Response" says so in the chosen
language.

**New test:** `scripts/test_i18n.py` — string completeness (same placeholders in
every language), exact picker labels, the re-render wiring, localised messages,
all 12 spoken example phrases + 4 real garbled Whisper transcripts routed from
native words, and a **voice section** that synthesises a question in each
language (gTTS), runs Whisper, and checks the agent calls the right tool.
**33/33 pass.** `scripts/test_lite_mode.py` now *blocks* the heavy libraries
(as on Render) and also checks the Lite screen (mic hidden, picker present).

**Browser-verified:** all four languages switch every listed element; asking a
question in the Marathi screen returned live Pune weather with the readout and
chips in Marathi; switching to Punjabi afterwards re-rendered them in Punjabi;
no console errors.

---

### Voice + Answers in Every Language on the Free Host (Groq) — for farmers who can't read

**The problem (in simple words):** on Render, the mic was hidden and only English
worked. Why? Render's free computer has only 512 MB of memory. The "ears"
(Whisper) need about 1 GB and the translator about 4 GB — they simply don't fit.

**The fix (in simple words):** we "rent" the ears and the translator for free
from **Groq**, a cloud service. Our small app sends the farmer's voice to Groq,
Groq writes down the words (Whisper-large-v3 — the biggest, most accurate
Whisper), and an open-weight AI model (OpenAI's **gpt-oss-120b**, Apache-2.0)
works out what the farmer means and later writes the answer **in the farmer's
own language**, using only the facts our tools found. Nothing new to install;
the app still uses under 200 MB. The only thing needed is a free key
(`GROQ_API_KEY`, see DEPLOY.md section C). *Note: Groq's free plan no longer
includes Llama 3.x (Enterprise only, Sept 2026), so gpt-oss is the default; the
app tries `gpt-oss-120b → gpt-oss-20b → llama-3.3-70b` and skips any it can't use.*

**The farmer's journey now:**

| Step | What the farmer does | What the app does |
|---|---|---|
| 1 | Opens the link | Opens in their language (remembered on the phone) |
| 2 | Taps 🎙️, speaks, taps stop | Sends the question by itself — no other button |
| 3 | — | Groq Whisper writes the words; "You asked: …" is shown |
| 4 | — | The LLM understands it: crop, age, village, weather/scheme/crop care |
| 5a | If something is missing | Asks back **out loud, in their language**: "How many days old is your wheat?" / "Which village are you in?" |
| 5b | Replies with just "40 days" / "Nashik" | Remembers the first question and completes it |
| 6 | — | LangChain agent runs the tools (crop / weather / scheme) |
| 7 | Listens | The LLM writes the answer in their language from the facts; gTTS speaks it |

**Every "incorrect or incomplete" case is told to the farmer, spoken, in their language:**

| Farmer says | App says (in their language) |
|---|---|
| Crop question, no crop named | "Which crop is this about? I know wheat, rice, cotton, onion, tomato, maize." |
| Fertilizer/water question, no age | "How many days old is your wheat crop? e.g. 40 days" |
| Impossible age (wheat, 300 days) | "Wheat is usually ready in about 120 days, but you said 300. Please check." |
| A crop we don't cover (sugarcane) | "I don't have advice for sugarcane yet. I can help with…" |
| Weather, no place | "Which village or town are you in? Or tap 📍 Use my location." |
| A place the map can't find | "I could not find 'X'. Please say a nearby big town or your district." |
| A scheme we don't have | "I can tell you about PM-KISAN, crop insurance, KCC, Soil Health Card." |
| Not about farming / unclear / hello | What the assistant can help with, plus an example question |
| Silence or noise | "I could not hear you. Speak close to the phone, then tap stop." |
| Groq busy / down | "Please wait a minute and ask again" / "try again in a few minutes" |

If part of a question can be answered (e.g. weather yes, but crop age missing),
it answers that part **and** adds the question for the missing detail at the end.

**Other help for farmers who can't read:** 📍 button fills the location from GPS
(the weather tool accepts coordinates); the village and language are remembered;
examples are in their language (tap = ask); a "What I can help with" panel;
every message is spoken; bigger mic and answer text; Indian towns are preferred
when two places share a name ("Shirur, Pune" also works).

**New files:**
- `app/models/groq_client.py` — talks to Groq with plain `requests`: model
  fallback, rate-limit handling, the key is never logged.
- `app/models/stt.py` → `GroqWhisperSTT` — forced language, a farm-word hint per
  language, and silence / noise / "echo" detection.
- `app/agents/understanding.py` — what was asked + what's missing (LLM JSON,
  with the keyword rules as a backup), and follow-up merging.
- `app/agents/clarify.py` — decides what to answer and what to ask back.
- `app/agents/reply_writer.py` — the answer in the farmer's language, facts only;
  retries if the model writes the wrong script (e.g. Punjabi in Devanagari).
- `scripts/test_voice_languages.py` — **66/66**: fakes the Render host + Groq and
  walks every journey above in all 4 languages, plus failure paths.
- `scripts/test_groq_live.py` — the same with the real key (run after adding it).

**Also fixed:** the Hindi/Punjabi/Marathi letter-spacing rule never applied
(Gradio's own CSS prefix out-ranked it) — Indic text on buttons and headings is
no longer stretched apart.

**Browser-verified (local, Render settings):** mic shown; Hindi page; tapping a
Hindi example answered from the crop tool (tillering stage); 📍 filled and
remembered the location and the weather tool used it; reload restored Hindi and
the place; phone-size layout checked; with a broken key the app fell back to the
rules and told the farmer in Hindi — and the key never appeared in the logs.

---

### One-Screen Rebuild for Farmers Who Can't Read + Faster Replies

**What was wrong:** two columns of panels, technical readouts, chips and
instructions competing for attention, and a spoken question taking ~50 seconds.

**The screen now** — one narrow column, one job at a time, almost nothing to read:

| Before | Now |
|---|---|
| Two columns, 8 visible sections | One column: language → microphone → answer → samples |
| Small language links in a corner | Four big language buttons, always on screen (sticky), in their own scripts |
| Mic in a panel among many | One large bordered microphone, the only thing to tap |
| Tool chips: 3 always-on badges ("idle / used") | One quiet line naming the sources, only after an answer |
| "Question language" readout on screen | Moved into the folded "Transcript & pipeline" |
| Typing box, location box, notes all visible | Folded away: "Ask by typing", "📍 My village", "What I can help with" |
| Small player controls | Big **🔊 Listen again** button |
| Sample questions: English only, filled the box | In the farmer's own language; tapping one **asks it straight away** |
| — | The page scrolls to the answer; bigger type everywhere |

**Faster replies** (same question, measured in the browser):

| Step | Before | Now | How |
|---|---|---|---|
| Typed question → answer on screen | 11.9 s | **0.3–3.8 s** | The words are sent to the page as soon as they exist; the voice follows |
| Making the voice (gTTS, 420 chars) | 11.5 s | **1.2 s** | Google's endpoint takes ~100 characters per request; the parts are now fetched at the same time and joined |
| Voice question (Hindi, whisper-small) | 38–49 s | **~10 s** | Whisper was looping ("अगर अगर अगर…") and generating to its 448-token limit: capped the length, penalised repeats, and dropped the second English pass unless the keyword rules understood nothing |
| First question after a restart | +20–30 s | 0 | The speech model and scheme index load in a background thread at startup |
| A garbled transcript | Answered the wrong question | "I could not hear you — please say it again" | Repetition detector (`_is_degenerate`) |

Also: spoken text now stops before the standing "Important / Source" boilerplate,
and **typed Hindi / Punjabi / Marathi is accepted in every build** (the keyword
rules understand it; the answer is English without a Groq key, and the page says
so) — which is what makes the sample questions work in all four languages.

---

### Deployment status
- **Render (Lite) — live:** https://farmer-advisory-voice-agent.onrender.com
  (verified HTTP 200). Redeploy after a push with Render's **Manual Deploy →
  Deploy latest commit** — the service isn't connected via Render's GitHub app,
  so pushes don't deploy on their own.
- **Colab notebook — ready:** `notebooks/run_full_app_colab.ipynb` runs the full
  app (4-language mic, LangChain agent on Qwen 1.5B, semantic search, voice).
  Dry-run verified locally; a real Colab run needs your Google account.
- **Hugging Face Spaces — not deployed:** Gradio Spaces need a paid plan since
  mid-2026; the Spaces config is kept in the repo.

---

## Current Dev Mode Settings (`.env`)

| Setting | Current Value | What it means |
|---|---|---|
| `WHISPER_MODEL_ID` | `openai/whisper-small` | STT (matches brief; tiny for more speed) |
| `LOCAL_LLM_MODEL_ID` | `Qwen/Qwen2.5-0.5B-Instruct` | Small real LLM for CPU (no GPU) |
| `USE_STUB_TRANSLATION` | `true` | Skip 4GB IndicTrans2; voice speaks English |
| `USE_STUB_LLM` | `true` | `false` → use the real local LLM (CPU) |
| `USE_HF_INFERENCE_API` | `true` | With both stubs off + this false → local LLM |
| `USE_STUB_RAG` | `false` | Use real FAISS scheme index |
| `TTS_ENGINE` | `gtts` | Real Hi/Mr/Pa audio on CPU; `parler` for prod GPU |
| `RAG_MIN_SCORE` | `0.30` | Below this → "insufficient information" (no fabrication) |
| `RAG_BACKEND` | `faiss` | `bm25` = keyword search, no torch (LITE_MODE forces it) |
| `LITE_MODE` | `false` | `true` = tiny footprint for the free Render deploy |
| `AGENT_BACKEND` | `langchain` (default) | `sequential` = deterministic orchestrator |
| `DEV_MODE` | `true` | Show pipeline trace panel in UI |
| `GROQ_API_KEY` | *(your free key)* | Voice via Groq Whisper + answers in all 4 languages; empty = off |
| `GROQ_LLM_MODELS` | `openai/gpt-oss-120b,…` | LLMs tried in order (skipped if unavailable/over limit) |
| `STT_BACKEND` | `auto` | `auto` = Groq if a key is set, else local Whisper |

## Project Status — ALL 15 PHASES COMPLETE ✅ + LIVE

All 15 phases are built, tested and evaluated, plus the LangChain agent (default
engine), the 4-language interface, and a live free deployment on Render. With a
free Groq key the live site takes **voice questions and answers in Hindi,
Punjabi, Marathi and English**, and asks the farmer back whenever a question is
incomplete or unclear. Without the key it falls back to typed English.

## GitHub Repository

https://github.com/sehscape/Farmer-Advisory-Voice-Agent
