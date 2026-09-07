# Farmer Advisory Voice Agent — Work Detail

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

## PENDING PHASES

---

### Phase 8 — LangChain Agent (Tool Orchestration)
**What it will do:**
- Replace the manual pipeline with a proper LangChain agent
- Agent decides dynamically which tools to call based on the query
- Tools registered: crop knowledge, weather, scheme RAG
- Agent reasons over tool outputs and generates the final answer
- Handles multi-intent queries (e.g. "weather + crop advice" together)

---

### Phase 9 — End-to-End English Pipeline Test
**What it will do:**
- Full integration test with all tools wired together
- Test cases covering all intent types
- Verify pipeline handles edge cases (unknown crop, missing location, etc.)

---

### Phase 10 — IndicTrans2 Output + Indic TTS
**What it will do:**
- Translate the English answer back to the farmer's regional language using IndicTrans2
- Pass translated text to `ai4bharat/indic-parler-tts` to generate audio
- Farmer hears the answer in their own language
- Audio plays automatically in the Gradio UI

---

### Phase 11 — Full Gradio Pipeline Wiring
**What it will do:**
- Connect all phases into one seamless UI flow
- Remove all stub labels and phase markers from UI
- Final output: voice answer plays automatically in regional language

---

### Phase 12 — Testing & Evaluation
**What it will do:**
- Test all supported languages (Hindi, Marathi, Punjabi)
- Test all crop types and growth stages
- Test all intent types
- Evaluate translation quality
- Evaluate answer relevance and accuracy

---

### Phase 13 — Optimization
**What it will do:**
- Reduce memory footprint for HF Spaces T4 GPU (16GB VRAM)
- Model quantization where applicable
- Response latency optimization
- Caching for repeated queries

---

### Phase 14 — Hugging Face Spaces Deployment
**What it will do:**
- Push to Hugging Face Spaces repository
- Configure `app.py` entry point for HF Spaces
- Set up HF Spaces secrets (HF_TOKEN, etc.)
- Switch model IDs to production versions (whisper-large-v3, etc.)
- Test on T4 GPU

---

### Phase 15 — Documentation
**What it will do:**
- Write README with project description, setup instructions, usage guide
- Document architecture decisions
- Add example queries and expected outputs
- Add contribution guidelines

---

## Current Dev Mode Settings (`.env`)

| Setting | Current Value | What it means |
|---|---|---|
| `WHISPER_MODEL_ID` | `openai/whisper-tiny` | Lightweight STT for CPU |
| `USE_STUB_TRANSLATION` | `true` | Skip 4GB IndicTrans2 download |
| `USE_STUB_LLM` | `true` | Use crop-context-aware StubLLM |
| `USE_HF_INFERENCE_API` | `true` | Use HF API for LLM (when stub is off) |
| `USE_STUB_RAG` | `false` | Use real FAISS scheme index (set `true` to skip load) |
| `DEV_MODE` | `true` | Show pipeline trace panel in UI |

## GitHub Repository

https://github.com/sehscape/Farmer-Advisory-Voice-Agent
