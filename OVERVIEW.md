# OVERVIEW — design_study: screenshot-to-code

Scansione del progetto eseguita il 2026-08-31 (branch `arena/01a05822-design-study-screenshot-to-cod`, base `d026163`).

---

## 🧭 Panoramica

Clone/fork di **screenshot-to-code** (di Abi Raja) usato come *design study*: converte screenshot, mockup, testo o registrazioni schermo in codice front-end funzionante, tramite un **agente LLM con tool calling**. Non è più "una chiamata al modello e via": il cuore è un agent loop che scrive file, estrae asset, genera/modifica immagini e **verifica visivamente il risultato** in un browser headless.

**Dimensioni:** ~21.400 righe di Python (113 file, di cui 37 di test) + ~18.200 righe TS/TSX nel frontend (73 componenti `.tsx`).

---

## 🧱 Stack

### Backend — `backend/` (Python 3.10+, Poetry)

| Cosa | Dettaglio |
|---|---|
| Framework | **FastAPI** + **Uvicorn**, CORS aperto (`*`) |
| Comunicazione | **WebSocket** per la generazione (streaming), HTTP REST per tutto il resto |
| Client LLM | `openai` (AsyncOpenAI, con `base_url` custom), `anthropic` (AsyncAnthropic), `google-genai` (Gemini) |
| Immagini | `httpx` verso **Replicate**, `Pillow` + `pillow-heif` (crop/EXIF) |
| Browser | **Playwright** (Chromium headless) per l'auto-verifica visiva |
| Video | `moviepy` (estrazione frame delle registrazioni) |
| Altro | `pydantic`, `beautifulsoup4` (export zip), `python-dotenv` |
| Qualità | `pytest` (37 file di test), `pyright` (type checking strict), `pre-commit` |

> `langfuse` è nel `pyproject.toml` ma **non è usato da nessuna parte** nel codice: dipendenza orfana.

### Frontend — `frontend/` (Vite + React 18 + TS)

- **Vite 6** (dev server su `0.0.0.0`, proxy `/generate-code` ws, `/api`, `/local-assets` → backend)
- **React 18** + **TypeScript**, **Tailwind CSS** + **shadcn/ui** (Radix primitives), **Zustand** per lo stato
- **CodeMirror 6** per l'editor, `react-dropzone`, `html2canvas`, `react-syntax-highlighter`, `react-router-dom`
- Test: **Jest** (+ Puppeteer per un QA end-to-end), **ESLint**
- Package manager: **pnpm** (workspace root)

### Infra

- `docker-compose.yml` (backend:7001 + frontend:5173), Dockerfile per entrambi
- Nessun database "vero": persistenza su filesystem (`run_logs/`, `evals_data/`, `local_assets/`) + **SQLite** come indice dei run (`fs_logging/agent_runs.py`)
- Nessuna autenticazione: le API key arrivano dal dialog Impostazioni o da `backend/.env`

---

## 📁 Struttura (backend)

```
agent/              engine.py (loop agente), runner.py, state.py
  providers/        openai.py | anthropic.py | gemini.py | factory.py   ← astrazione multi-provider
  tools/            definitions.py (schemi tool), runtime.py (dispatch), extract_assets.py, screenshot_preview.py
prompts/            pipeline.py, plan.py, system_prompt.py, create/{image,text,video}.py, update/
routes/             generate_code.py (WS, ~900 righe), screenshot.py, export.py, evals.py, eval_sets.py,
                    design_systems.py, agent_runs.py, prompt_reports.py, capabilities.py
image_generation/   replicate.py, generation.py
asset_extraction.py     crop asset con Gemini (structured output + box_2d)
preview_screenshot/     Playwright, con registry/probe
costs/              pricing.py (prezzi per-token), token_usage.py
evals/              harness di valutazione + sessioni
fs_logging/         agent_runs.py (registrazione completa run su disco), prompt_reports.py
uploaded_assets/    store + tools (mount /local-assets)
```

---

## ⭐ Caratteristiche principali

1. **Generazione agentica** — l'LLM non restituisce codice in chat: usa tool (`create_file`, `edit_file`, …). Se finisce senza output → `EmptyOutputError`.
2. **Multi-provider, multi-variante** — 4 varianti in parallelo su modelli diversi (2 in modalità *update*, 2 in *video*). La scelta dei modelli dipende da quali chiavi API hai (`routes/model_choice_sets.py`): con tutte e tre le chiavi → mix migliore; con una sola → solo quel provider.
3. **Estrazione asset reale** — Gemini (`gemini-3.6-flash`) individua logo/immagini nello screenshot con `box_2d` normalizzati, poi Pillow ritaglia e salva su `/local-assets`. Gli asset sono **content-addressed** (`asset_<sha256[:24]>.png`).
4. **Self-verification visiva** — tool `screenshot_preview`: Chromium headless renderizza l'HTML generato (desktop + mobile) e l'agente confronta e corregge con `edit_file`.
5. **Manipolazione immagini** — `generate_images`, `edit_images` (batch, con reference images e aspect ratio), `remove_backgrounds`.
6. **Input modes** — `image` (screenshot), `text`, `video` (richiede Gemini), più **update** di una generazione precedente (da storico conversazione o da snapshot del file).
7. **Storico a "commit"** — il frontend modella ogni generazione come un commit con N varianti e un `head` (simil-git), in `store/project-store.ts`.
8. **Select & edit** — puoi selezionare un elemento nel preview e chiedere una modifica mirata (viene passato l'`outerHTML` come localizzatore).
9. **Budget e sicurezza** — cap di spesa `$3.00` per variante (`GENERATION_MAX_COST_USD`), abortito fra un turno e l'altro; max 30 step di tool; errori OpenAI mappati su messaggi umani (chiave errata, rate limit, modello non trovato).
10. **Osservabilità** — con `PROMPT_REPORTS_ENABLED=1` ogni run viene salvata su disco (eventi JSONL, payload completi, token, costi, `final.html`, asset) e sfogliabile a `/evals/prompt-reports`.

---

## 🔄 Flusso principale

### 1. Frontend → WebSocket

`App.tsx` → `generateCode.ts` apre `ws://<backend>/generate-code` e invia un payload con: stack, input mode, generationType (`create`|`update`), prompt (testo + data URL di immagini/video), history, `fileState`, API key, design system, flag immagini/asset.

### 2. Pipeline a middleware (`routes/generate_code.py`)

```
WebSocketSetup → ParameterExtraction → StatusBroadcast → PromptCreation → CodeGeneration → PostProcessing
```

- **ParameterExtraction**: valida stack/input mode, risolve le chiavi (UI → env), normalizza `fileState` e `optionCodes`.
- **PromptCreation** (`prompts/pipeline.py`): `plan.py` decide la strategia → `create` (image/text/video) oppure `update` (da history o da snapshot file).
- **CodeGeneration**: seleziona i modelli, poi **lancia le varianti in parallelo** con `asyncio.gather` e invia `variantCount` / `variantModels`.

### 3. Agent loop (`agent/engine.py`) — per ogni variante

```
crea ProviderSession (openai | anthropic | gemini)
└─ for step in 1..30:
     stream_turn()  ──► eventi: thinking_delta / assistant_delta / tool_call_delta
     ├─ nessun tool call → finalize (HTML dal file_state o estratto dal testo) → fine
     ├─ controllo budget ($3)
     └─ per ogni tool call: toolStart → esegui → toolResult → append_tool_results → nuovo turno
```

Il codice viene **streammato in preview** al frontend durante la generazione (`setCode` incrementale), anche mentre il tool call è ancora in corso.

### 4. Tool disponibili all'agente

| Tool | Cosa fa | Dipendenza |
|---|---|---|
| `create_file` | crea l'HTML single-file | — |
| `edit_file` | replace esatti (anche multipli) | — |
| `extract_assets` | crop asset dallo screenshot | **Gemini** |
| `generate_images` | genera immagini da prompt (batch da 20) | **Replicate** |
| `edit_images` | editing/upscale batch con reference images | **Replicate** (`prunaai/p-image-edit`) |
| `remove_backgrounds` | rimozione sfondo batch | **Replicate** |
| `screenshot_preview` | rendering headless desktop+mobile | **Playwright/Chromium** |
| `save_assets` | salva immagini caricate dall'utente | — |
| `retrieve_option` | recupera l'HTML di un'altra variante | — |

I tool sono **abilitati condizionalmente** (chiave presente / Chromium disponibile / input con immagine statica), così non si espone qualcosa che fallirebbe.

### 5. Output

Messaggi WS: `chunk`, `status`, `setCode`, `thinking`, `assistant`, `toolStart`, `toolResult`, `variantComplete`, `variantError`, `variantCount`, `variantModels`, `error`. Il frontend aggiorna lo store Zustand, mostra le varianti in preview (iframe + CodeMirror) e permette export/download/screenshot.

---

## 🌐 API esterne

### Servizi chiamati dal backend

| Servizio | Endpoint | Uso | Chiave |
|---|---|---|---|
| **OpenAI** | SDK `AsyncOpenAI` | code-gen (GPT-5.5, 5.6-sol, 5.4-mini…) | `OPENAI_API_KEY` (+ `OPENAI_BASE_URL` per proxy) |
| **Anthropic** | SDK `AsyncAnthropic` | code-gen (Claude Opus 5 / 4.8 / Fable 5 / Sonnet 4.6) | `ANTHROPIC_API_KEY` |
| **Google Gemini** | SDK `google-genai` | code-gen + **asset extraction** (`gemini-3.6-flash`, obbligatorio per il video) | `GEMINI_API_KEY` |
| **Replicate** | `https://api.replicate.com/v1` (POST + polling `predictions/{id}`) | `prunaai/z-image-turbo`, `black-forest-labs/flux-2-klein-4b`, `prunaai/p-image-edit`, remove-background | `REPLICATE_API_KEY` (**solo da `.env`**) |
| **ScreenshotOne** | `https://api.screenshotone.com/take` | importa uno screenshot da URL (import "da sito live") | `screenshotOneApiKey` passata dal frontend |
| **Playwright Chromium** | locale | `screenshot_preview` – nessuna rete | installazione `playwright install chromium` |

### CDN referenziati **nel codice generato** (non chiamate backend)

`cdn.tailwindcss.com`, `cdn.jsdelivr.net` (Bootstrap, React, Ionic, ionicons), `unpkg.com` (Babel standalone 7.25.6 — pinnato apposta), `cdnjs.cloudflare.com` (Font Awesome), `registry.npmmirror.com` (Vue), `placehold.co`, Google Fonts.

### Lato frontend

`backend.buildpicoapps.com/form` (form di contatto, solo hosted), `buy.stripe.com` (checkout hosted), `codepen.io/pen/define` (export a CodePen), `plausible.io` (analytics, solo se `VITE_IS_DEPLOYED=true`).

---

## 🔬 Sottosistemi secondari

- **Evals** (`evals/` + `routes/evals.py`, `eval_sets.py`): dataset di 16 screenshot in `evals_data/inputs`, esecuzione in parallelo, rating 1–4, UI a `/evals`, sessioni e confronto input OpenAI.
- **Export** (`routes/export.py`): scarica un **ZIP** del progetto riscrivendo tutti gli asset remoti in locali (limiti: max 50 asset, 20 MB, 5 redirect, controlli anti-SSRF su IP privati).
- **Design systems** (`routes/design_systems.py`): prompt di stile salvabili e riusabili.
- **Agent runs** (`routes/agent_runs.py` + `fs_logging/`): storico run con output e asset, pruning.
- **Capability probe** (`/api/capabilities`): dice al frontend se `screenshot_preview` è disponibile.

---

## 📌 Punti degni di nota

- L'output è **sempre un singolo file HTML** con librerie via CDN: niente build step per l'utente finale.
- Il sistema prompt (`prompts/system_prompt.py`) è molto prescrittivo: niente HTML in chat, `create_file` una sola volta, `edit_file` per gli update, chiamata obbligatoria a `screenshot_preview` dopo le modifiche.
- `NUM_VARIANTS = 4` (create) / `2` (update, video) — definito in `config.py`.
- Il repo è anche un "laboratorio": `plan.md`, `design-docs/`, `AGENTS.md`, `QA.md`, `Evaluation.md`, `TESTING.md` documentano rifattorizzazioni in corso (variant system non-blocking, prompt history, agentic runner).
- Il branch `hosted` (citato in `AGENTS.md`) punta a un backend SaaS separato; qui giri in self-host.
- Non c'è autenticazione né multi-utenza: chiunque raggiunga il backend usa le tue chiavi.

---

## ▶️ Avvio rapido

```bash
# Backend
cd backend
echo "GEMINI_API_KEY=..." > .env      # almeno una chiave fra OPENAI / ANTHROPIC / GEMINI
poetry install
poetry run playwright install chromium
poetry run uvicorn main:app --reload --port 7001

# Frontend
cd frontend
pnpm install
pnpm dev        # http://localhost:5173
```

Oppure, dalla root: `docker-compose up -d --build` (app su http://localhost:5173).
