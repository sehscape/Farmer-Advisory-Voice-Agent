# Farmer Advisory Voice Agent — Work Detail (Beginner's Guide)

*Last updated: 22 September 2026*

This file explains the whole project from zero. You don't need to know anything
about AI or programming to follow it. Read it top to bottom once; after that, use
the table of contents to jump to what you need.

**Live link:** https://farmer-advisory-voice-agent.onrender.com
**Code:** https://github.com/sehscape/Farmer-Advisory-Voice-Agent

---

## Table of contents

1. [The project in one minute](#1-the-project-in-one-minute)
2. [Words you need to know](#2-words-you-need-to-know)
3. [The journey of one question](#3-the-journey-of-one-question)
4. [Each part explained in detail](#4-each-part-explained-in-detail)
5. [The three ways the app runs](#5-the-three-ways-the-app-runs)
6. [The settings file (.env)](#6-the-settings-file-env)
7. [Every important file, and what it does](#7-every-important-file-and-what-it-does)
8. [How to run it on your laptop](#8-how-to-run-it-on-your-laptop)
9. [How we prove it works (tests)](#9-how-we-prove-it-works-tests)
10. [Putting it online (deployment)](#10-putting-it-online-deployment)
11. [How we stop the AI from making things up](#11-how-we-stop-the-ai-from-making-things-up)
12. [Problems we hit, and how we solved them](#12-problems-we-hit-and-how-we-solved-them)
13. [Current status — what works, what doesn't](#13-current-status--what-works-what-doesnt)
14. [Limitations and future work](#14-limitations-and-future-work)
15. [How the project was built (timeline)](#15-how-the-project-was-built-timeline)

---

## 1. The project in one minute

### What is it?
A website where an Indian farmer:
1. **picks a language** — English, Hindi, Punjabi or Marathi,
2. **asks a farming question out loud** (or types it),
3. **hears the answer spoken back** in the language they picked.

It answers three kinds of questions:

| Kind | Example | Where the answer comes from |
|---|---|---|
| 🌱 **Crop care** | "My wheat is 40 days old, which fertilizer?" | A file of expert advice for 6 crops, stage by stage |
| 🌦️ **Weather** | "Will it rain in Nashik tomorrow?" | A live weather service on the internet |
| 🏛️ **Government schemes** | "How do I apply for PM-KISAN?" | Official scheme documents, searched by the app |

### Who is it for?
Farmers who speak only their regional language, and who may not read well. That's
why it is **voice-first**: every message on the screen is also spoken aloud.

### Why is it useful?
Good farming advice exists, but it's usually in long English or Hindi documents.
This app turns "reading a PDF" into "asking a question and listening".

### The one rule that matters most
**The AI is never allowed to invent facts.** It only rephrases what the tools
found (crop file, weather service, scheme documents). If it can't find a reliable
answer, it says so. This matters because wrong farm advice — a wrong fertilizer
dose, for example — can cost a farmer their crop.

---

## 2. Words you need to know

You'll see these words everywhere in this project and in interviews.

| Word | Plain meaning | In this project |
|---|---|---|
| **AI model** | A program that learned a skill from lots of examples | We use models for listening, understanding, translating |
| **LLM** (Large Language Model) | An AI that reads and writes text, like ChatGPT | `gpt-oss-120b` understands the question and writes the answer |
| **STT** (Speech-to-Text) | Turns a voice recording into written words | **Whisper** does this |
| **TTS** (Text-to-Speech) | Turns written words into a voice recording | **gTTS** does this |
| **API** | A way for one program to ask another program for something over the internet | We call the Groq API, the weather API |
| **API key** | A secret password that lets you use an API | `GROQ_API_KEY`, kept in `.env`, never shared |
| **Groq** | A company that runs AI models on its fast computers, with a free plan | Runs Whisper and the LLM for us, so our small server doesn't have to |
| **Open-weight model** | An AI model whose files are public, anyone can run it | `gpt-oss-120b`, Whisper, NLLB are all open |
| **Prompt** | The instructions and text you send to an LLM | e.g. "Reply only in Marathi, use only these facts…" |
| **Token** | A small piece of a word; LLMs read and write in tokens | Groq's free limits are counted in tokens |
| **Agent** | An AI program that decides *which tools to use* to answer | Decides: crop tool? weather tool? scheme tool? |
| **Tool** | A normal function the agent can call | `crop_knowledge`, `weather_forecast`, `government_schemes` |
| **LangChain** | A popular Python library for building agents | Runs our agent loop |
| **ReAct** | An agent style: *Reason* ("I need weather") → *Act* (call tool) → look at result → repeat | Our LangChain agent works this way |
| **RAG** (Retrieval-Augmented Generation) | First *find* the right text in your documents, then let the AI write the answer *from that text* | Used for government schemes |
| **Chunk** | A small piece of a long document | Scheme files are cut into 1000-character chunks |
| **Embedding** | A list of numbers that captures the *meaning* of a text | Similar meanings → similar numbers |
| **Vector database / FAISS** | A tool that stores embeddings and quickly finds the closest ones | Finds the scheme chunk closest to the question |
| **BM25** | An older search method that matches *keywords* (no AI, very light) | Used on the free server instead of FAISS |
| **Translation model** | An AI that translates between languages | **NLLB** (by Meta) translates English answers into Hindi/Marathi/Punjabi on the laptop |
| **Glossary** | A fixed list of word translations written by hand | Stops NLLB mistranslating farm terms |
| **Stub** | A simple stand-in for a heavy part, used for speed or testing | `StubLLM` = rule-based brain that needs no AI |
| **Fallback** | Plan B when Plan A fails | If Groq fails → translate with NLLB → else English with a notice |
| **Gradio** | A Python library for making web pages for AI apps | Builds our whole screen |
| **i18n** (internationalisation) | Making software work in many languages | `app/ui/i18n.py` holds every message in 4 languages |
| **Render** | A website-hosting company with a free plan | Hosts our live site |
| **Deploy** | Put the app on a server so anyone can open it | We deploy to Render |
| **Environment variable / `.env`** | Settings kept outside the code (like the API key) | `.env` on the laptop; "Environment" page on Render |
| **LITE_MODE** | Our special small version of the app for the free server | Turns off everything heavy |
| **Git / GitHub** | Tools that save every version of the code | Our code is on GitHub |
| **Virtual environment (venv)** | A private folder of Python libraries for one project | `venv/` in the project |

---

## 3. The journey of one question

Let's follow one real question: the farmer has **Hindi** selected and says
*"मेरी गेहूं 40 दिन की है, क्या खाद डालूं?"* ("My wheat is 40 days old, what
fertilizer should I apply?").

```
  1. 🗣️  Farmer taps the mic, speaks, taps stop
  2. 👂  Whisper writes down the words (in Hindi)
  3. 🧐  Understanding: crop = wheat, age = 40 days, wants = crop advice (fertilizer)
  4. ❓  Clarify: is anything missing? → No
  5. 🧭  LangChain agent: call the crop_knowledge tool
  6. 🌱  Crop tool: wheat at 40 days = "Tillering" stage → irrigation, urea dose, pests
  7. 🧠  Writer: turn those facts into a short answer IN HINDI
  8. 🔊  gTTS: turn the Hindi answer into speech; it plays by itself
```

| Step | What happens | Which file |
|---|---|---|
| 1 | The browser records audio. When the farmer taps stop, the question is sent automatically — no extra button. | `app/ui/gradio_app.py` |
| 2 | The audio goes to Whisper. We *tell* Whisper the language is Hindi (guessing was unreliable). | `app/models/stt.py` |
| 3 | The words go to the LLM on Groq, which returns a small structured summary (JSON): crop, age, place, what's wanted. | `app/agents/understanding.py` |
| 4 | The app checks for missing details. If the age were missing it would ask, in Hindi, "How many days old is your wheat?" | `app/agents/clarify.py` |
| 5 | The agent decides which tools to run, following the plan from step 3–4. | `app/agents/langchain_agent.py` |
| 6 | The crop tool opens `data/crops/wheat.json`, finds the stage that covers day 40 and reads its advice. | `app/tools/crop_tool.py` |
| 7 | The LLM writes a short Hindi answer using **only** those facts. | `app/agents/reply_writer.py` |
| 8 | The text appears on screen first; the voice follows a moment later. | `app/models/tts.py` |

Everything about this one question is carried from step to step in a single
Python object called **`AgentState`** (`app/agents/state.py`) — think of it as a
folder that travels down the line, and each worker adds a page to it (the words,
the crop, the weather result, the answer…). It also carries
**`response_language`** — the language the farmer picked — so every step knows
which language the answer must be in.

---

## 4. Each part explained in detail

### 4.1 The screen — `app/ui/gradio_app.py` and `app/ui/i18n.py`

**What the farmer sees:**
- a **language picker** at the top (English · Hindi · ਪੰਜਾਬੀ · मराठी),
- **01 Speak** — the microphone,
- **02 Or type** — a text box,
- **03 Location** — type a village, or tap **📍 Use my location** (GPS),
- example questions — tap one and it's asked immediately,
- the **answer**, with **tool chips** showing which tools were used (Crop / Weather / Scheme),
- the **spoken reply** player (it plays by itself),
- a folded **"Transcript & pipeline"** panel with technical details.

**How the 4 languages work:** every piece of text on the screen is stored in
`i18n.py` — 87 messages, each written in all 4 languages. When the farmer changes
the language, the app swaps every message at once.

**What's remembered:** the chosen language and the village are saved in the
phone's browser (`localStorage`), so the next visit opens the same way.

**The mode line:** under the big headline there's a small line like
`lite · voice groq whisper-large-v3 · llm groq openai/gpt-oss-120b · agent langchain · rag bm25 · tts gtts · 4 languages`.
It tells you exactly which parts are switched on. It is the fastest way to
check if a deployment is configured correctly.

### 4.2 The ears (speech-to-text) — `app/models/stt.py`

The app has two kinds of "ears":

| Class | Where it runs | When it's used |
|---|---|---|
| `GroqWhisperSTT` | Groq's servers, **Whisper-large-v3** (the biggest, most accurate Whisper) | When `GROQ_API_KEY` is set |
| `WhisperSTT` | Your laptop, **Whisper-small** | When there's no key (laptop only) |

Important details:
- **The language is forced** to the one the farmer picked. Automatic detection
  often got Hindi wrong, so we tell Whisper instead of letting it guess.
- **Farm-word hints:** Groq Whisper gets a short hint of farm words in that
  language, which improves spelling of crop names.
- **Silence and noise detection:** if the recording is empty or noise, the
  farmer hears "I could not hear you, please speak close to the phone".
- **Loop protection:** local Whisper sometimes repeated a word forever
  ("अगर अगर अगर…"). We limit its length, penalise repeats, and detect such output
  (`_is_degenerate`) and ask the farmer to speak again.

### 4.3 Understanding the question — `app/agents/understanding.py`

The goal: turn the farmer's words (in any of the 4 languages) into one clear
summary called an **`Understanding`**:

```
english          : "My wheat is 40 days old, which fertilizer should I apply?"
topic            : farming          (or greeting / off_topic / unclear)
crop             : wheat
crop_age_days    : 40
location         : —
wants_crop_advice: yes   (advice_topic: fertilizer)
wants_weather    : no
wants_scheme     : no
```

There are **two engines**:
1. **The LLM (Groq)** — reads any of the 4 languages directly, understands spelling
   mistakes from speech recognition, and returns this summary as JSON. We set its
   *temperature* to 0, which makes its output consistent rather than creative.
2. **The rules** — used when there's no key, or when the LLM fails. It's a
   keyword matcher (`StubLLM._classify` in `app/models/llm.py`) that knows farm
   words in English, Hindi, Marathi and Punjabi, 22 city names, and spoken
   numbers like "चालीस दिन" (forty days).

Even when the LLM is used, the rules run too and fill in anything the LLM left
empty.

**Follow-up memory:** if the app just asked "How many days old is your wheat?"
and the farmer replies only "40 days", the understanding step joins that reply
to the earlier question (`merge_followup`).

### 4.4 Checking what's missing — `app/agents/clarify.py`

Before calling any tool, the app asks: *can I actually answer this?*

| The farmer said… | The app replies (in their language) |
|---|---|
| A crop question with no crop | "Which crop is this about? I know wheat, rice, cotton, onion, tomato, maize." |
| Fertilizer/water question, no age | "How many days old is your wheat crop? e.g. 40 days" |
| An impossible age (wheat, 300 days) | "Wheat is usually ready in about 120 days, but you said 300. Please check." |
| A crop we don't cover (sugarcane) | "I don't have advice for sugarcane yet. I can help with…" |
| Weather, but no place | "Which village or town are you in? Or tap 📍." |
| A place the map can't find | "I could not find 'X'. Please say a nearby big town." |
| A scheme we don't have | "I can tell you about PM-KISAN, crop insurance, KCC, Soil Health Card." |
| Hello / off-topic / unclear | What the assistant can help with, plus an example |

If *part* of the question can be answered (say, the weather), the app answers
that part and adds the missing-detail question at the end. The missing detail is
remembered, so the farmer can reply with just "40 days".

The output of this step is a **plan**: which tools to run, with which crop, age,
place and search words.

### 4.5 The agent — `app/agents/langchain_agent.py`

The **agent** is the manager that calls the tools. It uses **LangChain** and the
**ReAct** style: it writes a thought ("I need the crop advice"), picks a tool,
LangChain runs the tool, the result comes back, and it decides the next step —
until it has everything.

The three tools it can call:

| Tool name | What it does | Input |
|---|---|---|
| `crop_knowledge` | Stage-by-stage advice for a crop | crop name |
| `weather_forecast` | Live weather + 3-day forecast + farm advisories | place name or GPS coordinates |
| `government_schemes` | Searches the scheme documents (RAG) | the question |

**Who makes the decisions inside the agent?** In this project, a small
rule-based policy (`StubLLM._react_step`) that follows the plan from the clarify
step and speaks LangChain's exact ReAct text format. The LangChain machinery
(executor, tool calls, parsing) is real. If you connect a real LLM (as in the
Colab notebook), the LLM makes the decisions instead — no code change. We chose
this because it is fast, free and never picks the wrong tool.

**Safety net:** if the agent loop crashes, loops too long, or can't read its own
output, a simpler **sequential orchestrator** (`app/agents/orchestrator.py`)
runs the tools directly. The farmer never sees a broken reply.

### 4.6 The crop expert — `app/tools/crop_tool.py` + `data/crops/*.json`

Each crop has one JSON file. Inside, the crop's life is split into **stages** by
day range. Each stage has: name, description, irrigation, fertilizer, pests to
watch for, and tips.

| Crop | Stages | Total days |
|---|---|---|
| Wheat | 8 | 120 |
| Rice | 6 | 135 |
| Onion | 5 | 130 |
| Tomato | 6 | 120 |
| Cotton | 6 | 180 |
| Maize | 6 | 100 |

How it works: take the crop and its age → find the stage whose day range
contains that age → return that stage's advice. For wheat at 40 days that's
**Tillering (days 26–50)**: second irrigation at 40–45 days, 25 kg urea per acre,
watch for aphids and powdery mildew, remove weeds.

Crop names are recognised in all languages (गेहूं / ਕਣਕ / गहू → wheat; paddy → rice).

### 4.7 The weather checker — `app/tools/weather_tool.py`

1. **Find the place:** the village name is turned into map coordinates using the
   free **Open-Meteo Geocoding API**. Indian places are preferred when two share a
   name. If the farmer tapped 📍, we already have coordinates.
2. **Get the weather:** the **Open-Meteo Forecast API** gives current conditions
   (temperature, humidity, rain, wind) and a 3-day forecast (rain chance and
   amount, min/max temperature).
3. **Turn codes into words:** weather codes like `61` become "Slight rain".
4. **Add farming advice by rule:**

| Condition | Advice |
|---|---|
| Temperature above 38 °C | Avoid spraying pesticides; irrigate |
| Below 5 °C | Near-frost — protect sensitive crops |
| Humidity above 85% | Fungal disease risk — check crops |
| Rain above 20 mm | Delay fertilizer application |
| Thunderstorm | Keep people and equipment away from fields |

No API key is needed. Place lookups are cached, so asking again is faster.
If the service is down, the farmer hears "the weather service isn't responding,
please ask again later" — the rest of the answer still works.

### 4.8 The scheme finder (RAG) — `app/rag/` + `app/tools/scheme_tool.py`

**What problem RAG solves:** an LLM on its own might "remember" scheme rules
wrongly or make them up. With RAG, we first **find** the right paragraph in the
official documents, then the answer is written **from that paragraph only**.

**The documents:** 4 text files in `data/schemes/raw/` — PM-KISAN, PM Fasal Bima
Yojana (crop insurance), Kisan Credit Card, Soil Health Card.

**Preparing the search (done once, `scripts/build_scheme_index.py`):**
1. Read the 4 files.
2. Cut them into overlapping **chunks** of 1000 characters (150 characters
   overlap, so a sentence cut at the edge still appears whole in one chunk).
   This gives **15 chunks**.
3. Turn each chunk into an **embedding** with the
   `paraphrase-multilingual-MiniLM-L12-v2` model.
4. Store them in a **FAISS** index (`data/vectorstore/schemes.faiss`).

**Answering a question:**
1. Turn the question into an embedding too.
2. FAISS finds the **5 closest chunks** (by *cosine similarity* — how close their
   meanings are, from 0 to 1).
3. **Guardrail:** chunks scoring below **0.30** are thrown away. If none are left,
   the tool answers *"The documents do not contain sufficient information…"*
   instead of guessing.
4. The surviving chunks, with their source file names, go to the answer writer.

**On the free server** there isn't enough memory for the embedding model, so a
second search method is used: **BM25** keyword search (`keyword_retriever.py`),
written in plain Python. Two extra rules make it reliable: a boost when a word
matches the scheme's own name, and a rule that the best chunk must match **at
least 2 different words** of the question — one accidental match is not enough.
The setting `RAG_BACKEND` chooses the method (`faiss` or `bm25`).

### 4.9 Writing the answer in the chosen language

This is where the farmer's chosen language is enforced. The app tries three
ways, in order:

**Way 1 — the LLM writes it directly** (`app/agents/reply_writer.py`, needs Groq).
The LLM gets: the farmer's question, the facts from the tools, and strict rules:
- reply **only** in the chosen language and script, even if the question was in another language,
- use **only** the facts — no dose, date, price or rule that isn't there,
- answer the question in 1–2 sentences, then at most 4 short steps,
- simple spoken sentences, no markdown or emojis, under 110 words.

If the reply comes back in the wrong script (e.g. Punjabi written in Devanagari),
the app asks once more; if it's still wrong, it moves to way 2.

**Way 2 — translate the English answer** (`app/models/translation.py`, laptop).
First, a rule-based English answer is built from the facts (`StubLLM._compose_answer`):
*Situation → What you should do (numbered steps) → Important → Source*. Then the
**NLLB-200** translation model (by Meta, 600 million parameters, ~2.5 GB, runs on
the CPU) translates it. Three protections were added because the model made
dangerous mistakes:

| Mistake the model made | Protection |
|---|---|
| "Tillering" (a growth stage) → "harvest" in Hindi and Marathi | Growth stages and section headings come from a **hand-written glossary** (`app/models/agri_glossary.py`), never from the model |
| "3–5 cm" of water → "35 cm" (it dropped the dash) | Number ranges are rewritten as "3 to 5" before translating |
| "per hill" → "per mountain"; "live weather" → "living weather" | These phrases are reworded before translating |

The answer is translated one sentence at a time (short inputs translate better),
all in one batch, and results are cached.

**Way 3 — English with a spoken notice.** If both fail (e.g. on Render, which has
no translator), the farmer sees the English advice and hears, in their language,
"I couldn't prepare the answer in Hindi right now, please ask again in a minute."

### 4.10 The voice — `app/models/tts.py`

**gTTS** (Google Text-to-Speech) makes an MP3 in Hindi, Marathi, Punjabi or
English. It only accepts about 100 characters per request, so long answers are
split into pieces that are fetched **at the same time** and joined — this cut the
wait from 11.5 s to about 1.2 s. The standing "Important / Source" lines are not
read aloud, to keep the audio short.

A higher-quality Indian voice (`ai4bharat/indic-parler-tts`) is built in too, but
it needs a GPU, so it is off by default.

### 4.11 The Groq connection — `app/models/groq_client.py`

One small file talks to Groq using plain web requests:
- **Model fallback:** it tries `gpt-oss-120b` → `gpt-oss-20b` → `llama-3.3-70b`.
  A model that's over its free limit is skipped for that request; a model the
  account can't use is skipped for good.
- **Clear errors:** each failure is labelled (wrong key, rate limit, too large,
  network…) so the farmer hears the right message ("please wait a minute").
- **The key is never written to the logs.**

---

## 5. The three ways the app runs

| | Laptop + Groq key | Laptop, no key | Render free server + Groq key |
|---|---|---|---|
| Voice questions | ✅ Groq Whisper-large-v3 | ✅ Local Whisper-small (rough for Marathi/Punjabi) | ✅ Groq Whisper-large-v3 |
| Understanding | Groq LLM | Rules | Groq LLM |
| Answer language | ✅ LLM writes it | ✅ NLLB translates (6–8 s) | ✅ LLM writes it |
| If Groq fails | NLLB translates instead | — | English + spoken notice |
| Scheme search | FAISS (meaning) | FAISS (meaning) | BM25 (keywords) |
| Memory | ~3–4 GB | ~3–4 GB | under 200 MB |

**Why is Render different?** Render's free plan gives only **512 MB of memory**.
Whisper, the embedding model and the translator together need several GB. So on
Render we switch on **`LITE_MODE`**, which removes all of them and uses Groq
(for listening and language) and BM25 (for search) instead. Without a Groq key,
Render can only take typed English questions.

---

## 6. The settings file (.env)

All settings are read by `app/config.py` from the `.env` file (laptop) or from
Render's "Environment" page (live site). Your current laptop settings:

| Setting | Your value | What it means |
|---|---|---|
| `GROQ_API_KEY` | *(your key, valid)* | Turns on Groq: voice via Whisper-large-v3, LLM understanding and answers |
| `USE_STUB_TRANSLATION` | `false` | Use a real offline translator when Groq can't write the answer |
| `TRANSLATION_ENGINE` | `nllb` | Which translator: `nllb` (free, CPU) or `indictrans2` (gated, GPU) |
| `USE_STUB_LLM` | `true` | Build the English answer with rules (fast and grounded) instead of a slow local LLM |
| `USE_STUB_RAG` | `false` | Use the real FAISS scheme search |
| `WHISPER_MODEL_ID` | `openai/whisper-small` | Local ears, used only without a Groq key |
| `TTS_ENGINE` | `gtts` | Voice engine |
| `DEV_MODE` | `true` | Show the technical pipeline panel |
| `LITE_MODE` | *(not set → false)* | `true` only on Render |

Other settings (defaults are fine): `AGENT_BACKEND=langchain`,
`RAG_BACKEND=faiss`, `RAG_MIN_SCORE=0.30`, `TOP_K=5`, `STT_BACKEND=auto`,
`GROQ_LLM_MODELS=openai/gpt-oss-120b,openai/gpt-oss-20b,llama-3.3-70b-versatile`.

On Render, `render.yaml` sets `LITE_MODE=true` and `PYTHON_VERSION=3.11.9`;
`GROQ_API_KEY` must be added by hand on the service.

---

## 7. Every important file, and what it does

```
app/
├── main.py                  starts the app, loads models in the background
├── config.py                reads every setting from .env
├── agents/
│   ├── state.py             AgentState — the "folder" that travels with each question
│   ├── understanding.py     what was asked + what's missing (LLM or rules)
│   ├── clarify.py           decides what to answer and what to ask back
│   ├── langchain_agent.py   the LangChain ReAct agent + its 3 tools
│   ├── orchestrator.py      backup agent: runs the tools one after another
│   ├── answering.py         builds the English answer from the tool results
│   ├── reply_writer.py      the LLM writes the answer in the chosen language
│   ├── prompts.py           prompt templates
│   └── intent.py            older intent extractor (still used by tests)
├── models/
│   ├── stt.py               ears: Groq Whisper, local Whisper, silence checks
│   ├── tts.py               voice: gTTS (default), Parler (GPU), stub
│   ├── llm.py               StubLLM (rules, ReAct policy, English answer), local/HF LLMs
│   ├── groq_client.py       talks to Groq: model fallback, errors, key safety
│   ├── translation.py       NLLB and IndicTrans2 translators
│   ├── agri_glossary.py     hand-written farm terms in Hindi/Marathi/Punjabi
│   ├── embeddings.py        turns text into embeddings for FAISS
│   ├── langchain_llm.py     lets LangChain use our LLMs
│   └── indic_processor.py   text clean-up for IndicTrans2 (Windows-friendly)
├── rag/
│   ├── scheme_rag.py        FAISS search: chunk → embed → index → query
│   └── keyword_retriever.py BM25 keyword search for the free server
├── tools/
│   ├── crop_tool.py         crop advice by growth stage
│   ├── weather_tool.py      Open-Meteo weather + farm advisories
│   └── scheme_tool.py       scheme search + "insufficient information" guardrail
├── ui/
│   ├── gradio_app.py        the screen and the whole question pipeline
│   └── i18n.py              every message in 4 languages, crop names, examples
└── utils/                   logging, audio conversion, CPU/GPU detection

data/crops/                  6 crop JSON files
data/schemes/raw/            4 scheme documents
data/vectorstore/            the built FAISS index (15 chunks)
scripts/                     tests, evaluation, index builder
tests/test_stt.py            pytest unit tests
notebooks/                   Colab notebook for the full app
render.yaml                  Render settings
requirements.txt             libraries for the full app
requirements-lite.txt        libraries for Render (no PyTorch)
DEPLOY.md                    step-by-step deployment guide
```

The questions pipeline itself lives in `_pipeline_iter()` inside
`app/ui/gradio_app.py` — read it if you want to see all the steps in one place.

---

## 8. How to run it on your laptop

```bash
cd "D:\JIO 26-27\Live Project\multilingual-agri-agent"
venv\Scripts\activate                  # switch on the project's Python libraries
python scripts/build_scheme_index.py   # only once — prepares the scheme search
python app.py                          # start the app
```

Open http://localhost:7860 (or the port shown). Startup takes 15–35 seconds while
models load in the background. The microphone needs a real browser like Chrome.

To check it's working, look at the mode line: with your Groq key it should say
`voice groq whisper-large-v3 · llm groq openai/gpt-oss-120b · … · 4 languages`.

---

## 9. How we prove it works (tests)

A **test** is a small program that runs the app with known questions and checks
the answers automatically. Current results (22 Sept 2026, laptop with Groq key):

| Test script | What it checks, in plain words | Result |
|---|---|---|
| `test_weather_phase6.py` | Weather for real places; bad places; no place | ✅ all pass |
| `test_scheme_phase7.py` | Scheme search finds the right scheme and refuses off-topic questions | ✅ all pass |
| `test_e2e_phase9.py` | Every type of question goes to the right tools; empty/odd inputs don't crash | ✅ 14/14 |
| `test_tts_phase10.py` | Voice is produced in all 4 languages | ✅ 11/11 |
| `test_langchain_agent.py` | The agent picks exactly the right tools for 9 question types; the backup takes over when needed | ✅ 34/34 |
| `test_i18n.py` | The screen really switches all 4 languages; spoken questions reach the right tool | ✅ 33/33 |
| `test_lite_mode.py` | The Render version runs with the heavy libraries missing | ✅ all pass |
| `test_voice_languages.py` | Pretends to be Render + Groq and walks every farmer journey in 4 languages, including failures | ✅ 67/67 |
| `test_language_consistency.py` | The chosen language always wins; glossary works; "3–5 cm" never becomes "35 cm" | ✅ 16/16 |
| `test_groq_live.py` | The same journeys with your real Groq key | ✅ all pass |
| `pytest -q` | Unit tests for speech-to-text | ✅ 13 passed |

**Evaluation** (`scripts/evaluate.py`) measures quality on a small test set:
intent 100%, tool choice 100%, scheme search top-1 100% (average relevance
0.60), off-topic refusal 100%, answers grounded / actionable / cited 100%.
Because the test set is small, this shows the pipeline is wired correctly — it is
**not** a measure of real-world accuracy.

---

## 10. Putting it online (deployment)

**Render (the live site).** The code is on GitHub; Render copies it, installs
`requirements-lite.txt`, and runs `python app.py` with `LITE_MODE=true`.
- Pushing to GitHub does **not** update the site by itself. After a push, open
  the service on Render → **Manual Deploy → Deploy latest commit**.
- The free server **sleeps** after ~15 minutes idle; the next visit wakes it in
  about a minute.
- **Adding the Groq key:** open the **web service** (not the project) →
  **Environment** → add `GROQ_API_KEY` → **Save, rebuild, and deploy**.
  Lesson learned: a key saved in an **Environment Group** does nothing until that
  group is **linked** to the service.

**Google Colab** (`notebooks/run_full_app_colab.ipynb`) runs the full app for a
demo, with a temporary public link. It stops when you close the tab.

**Hugging Face Spaces** needs a paid plan for Gradio apps since mid-2026, so it
isn't used (the settings header is kept at the top of the README).

Full steps: `DEPLOY.md`.

---

## 11. How we stop the AI from making things up

| Protection | Where | What it does |
|---|---|---|
| Facts come from tools, not the AI | whole design | Crop file, weather API and scheme documents are the only sources |
| Strict answer prompt | `reply_writer.py` | "Use only the facts; never add a dose, amount, price, date or rule" |
| RAG relevance threshold | `scheme_tool.py` | Weak matches (below 0.30) are dropped → "insufficient information" |
| BM25 two-word rule | `scheme_tool.py` | One accidental keyword match is not trusted |
| Ask-backs | `clarify.py` | Missing crop, age or place → ask, never guess |
| Impossible-age check | `clarify.py` | "Wheat, 300 days" → "please check the age" |
| Farm glossary | `agri_glossary.py` | Growth stages are never machine-translated |
| Range rewriting | `translation.py` | "3–5 cm" can't become "35 cm" |
| Wrong-script check | `reply_writer.py` | A Punjabi answer in the wrong alphabet is never shown |
| KVK reminder | answer template | "Confirm dosage with your local Krishi Vigyan Kendra" |

**Honest note:** the LLM still sometimes adds a small step that isn't in the
facts (e.g. "dissolve urea in water"). A next step is an automatic check that
every number in the answer appears in the facts.

---

## 12. Problems we hit, and how we solved them

These make good interview stories.

| Problem | Why it happened | Fix |
|---|---|---|
| IndicTrans2 tools wouldn't install on Windows | Needed Microsoft C++ build tools | Wrote a pure-Python replacement (`indic_processor.py`) |
| Test scripts crashed printing Hindi | Windows console used an old encoding | `enable_utf8_console()` in `app/utils/logging.py` |
| Whisper got the language wrong | Auto-detection is unreliable on short Indic clips | Force the language the farmer picked |
| Local Whisper repeated "अगर अगर अगर…" for ~40 s | A known repetition loop on CPU | Length cap + repeat penalty + repeat detector → ~10 s |
| Render's free server couldn't fit the models | Only 512 MB of memory | `LITE_MODE` + BM25 + Groq for listening and language |
| Render build failed | Render picked Python 3.13; then a Gradio/Hub version clash | Pinned Python 3.11.9 and `gradio==6.26.0` |
| Voice took 11.5 s to generate | gTTS sends ~100 characters per request, one by one | Fetch the pieces in parallel → 1.2 s |
| Hindi letters on buttons looked stretched | Letter-spacing meant for English text | Relaxed the spacing for Indic pages |
| Answers came in English without a key | No translator was switched on | Added NLLB, since IndicTrans2 is gated and Google's free endpoint blocks automated use |
| NLLB said "harvest" for "tillering", "35 cm" for "3–5 cm" | A general model doesn't know farm terms or keep dashes | Hand-written glossary + range rewriting + tests |
| A test failed after adding the Groq key | The test assumed local Whisper was always used | Test now checks the engine that's actually configured |
| Live site ignored the key | Key was saved in an Environment Group that wasn't linked | Add the key on the service itself (or link the group) |

---

## 13. Current status — what works, what doesn't

*(checked 21–22 September 2026)*

### ✅ Working
- **Laptop, with your Groq key:** voice questions, understanding by the LLM,
  answers in the chosen language (Hindi answer written in 0.8 s; Marathi ~5 s end
  to end), ask-backs, all three tools, spoken replies. All tests pass.
- **Laptop, without a key:** everything works; answers are translated by NLLB.
- **Live site (Render):** the page, language switching, crop advice, weather
  (Nashik, clear sky, 28.6 °C on 21 Sept), scheme keyword search, spoken replies,
  ask-backs, and the recorder is shown.

### ❌ Not working
- **The live site answers in English.** Render has a Groq key, but every Groq call
  fails — almost certainly the **old, invalid key**. Your laptop key works.
  **Fix:** Render → the web service → Environment → paste the key from your
  local `.env` into `GROQ_API_KEY` → Save, rebuild, and deploy.
- For the same reason, **voice questions on the live site** will fail until the
  key is fixed.

### 🚧 Started, not finished
- **Android app** (`mobile/`, React Native + Expo): the project is created, but
  the app is still Expo's starter screen, and the backend it needs
  (`app/api/mobile.py`) doesn't exist. `mobile/README.md` describes the planned
  app, not a working one.

---

## 14. Limitations and future work

**Limitations**
- Groq's free plan allows roughly 50–60 full answers a day on the best model;
  everyone on the live link shares one key.
- Questions go to Groq's servers (a privacy trade-off; production should self-host).
- The LLM sometimes adds small steps not in the facts.
- The offline translator is literal with pest names and takes 6–8 s.
- Render has no offline translator or semantic search (not enough memory).
- Small knowledge base: 6 crops, 4 schemes.
- gTTS needs internet and has one voice per language.

**Future work**
1. Fix the live Groq key.
2. Automatically check every number in the answer against the facts.
3. Add pest and disease names to the glossary.
4. More crops and schemes; real government PDFs with page citations.
5. Finish the Android app.
6. Self-host Whisper + an LLM on a GPU; a more natural Indian voice.

---

## 15. How the project was built (timeline)

Dates are from the git history (local work started in late August 2026; the
first commit is 6 September).

| When | What was built |
|---|---|
| 6 Sep 2026 | **Phases 1–3:** project setup; Whisper speech-to-text; IndicTrans2 translation |
| 7 Sep 2026 | **Phases 4–8:** intent extraction and answers; crop tool (6 crops); weather tool (Open-Meteo); scheme RAG (FAISS, 15 chunks); sequential tool orchestrator; first UI redesign |
| 7–9 Sep 2026 | Hardening pass (UTF-8 fix, better rule-based answers, RAG "insufficient information" guardrail) |
| 9 Sep 2026 | **Phases 9–15:** end-to-end test (14/14); voice output (gTTS, Parler); full screen wiring with auto-play; evaluation harness; GPU auto-detect and caching; deployment prep; documentation; small real LLM on CPU (Qwen 0.5B) |
| 10 Sep 2026 | **LITE_MODE** + BM25 for Render; Colab notebook; Render build fixes; live on Render; editorial UI theme |
| 11 Sep 2026 | **LangChain ReAct agent** as the default engine (34/34); **4-language screen** + voice questions in every language (33/33) |
| 12 Sep 2026 | **Groq**: Whisper-large-v3 + open LLM; understanding, ask-backs, answers in the farmer's language on Render (67/67) |
| 12–14 Sep 2026 | Faster replies (text first, parallel voice, Whisper loop fix); two-column screen kept |
| 16 Sep 2026 | **Chosen-language answers without Groq**: NLLB translator, farm glossary, range fix, `response_language` (16/16) |
| 17–22 Sep 2026 | Groq key added locally; Render key troubleshooting; test updated for Groq; docs rewritten |
