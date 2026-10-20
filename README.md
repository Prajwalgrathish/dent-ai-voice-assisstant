# DentVoice AI

A voice-first dental assistant: one Streamlit app with **four sidebar sections** on top of one FastAPI backend.

| # | Section | What it does |
|---|---|---|
| 1 | **AI Assistant** | Voice/typed assistant for clinicians: retrieves reports ("Open Priya's latest OPG report"), keeps patient/report context for follow-ups ("Summarize her report", "Download that as a PDF"), general chat with conversation memory, and **inline clinical dictation** (say or type "Lower right third molar is horizontally impacted" and get an editable draft). |
| 2 | **Patient Reports** | Search a patient, verify identity, view/download the latest report. |
| 3 | **Clinical Dictation** | Record or paste a dictation, get an editable structured draft (site, findings, uncertainty), review and sign off. |
| 4 | **Patient Voice Assistant** | Patients enter **only their Patient ID**, then speak or type questions about **their own** reports and hear/read plain-English answers (see below). |

Patient data is **synthetic**. All clinical output requires clinician review. Docs: [architecture](docs/ARCHITECTURE.md) ·
[privacy / authorization / latency / cost](docs/PRODUCTION_NOTES.md) · [demo API calls with real output](docs/DEMO.md) ·
[requirement traceability and test evidence](docs/REQUIREMENTS_TRACEABILITY.md).

### Hands-free voice (AI Assistant)
The AI Assistant greets once for your local time, speaks it, then starts listening by itself (after the first microphone
permission tap). Status labels follow what the browser really did. After 15 s of silence it stops listening; tap the
microphone, press *Resume listening* or type **"Listen to me"** to continue. Full design, states and limitations:
[docs/VOICE_WORKFLOW.md](docs/VOICE_WORKFLOW.md). Real-browser check of a running app:
`python scripts/browser_voice_check.py --url http://localhost:8502` (needs `playwright`, see requirements-dev.txt).

## Quick start

```powershell
# one-time (Python 3.11+): from the project folder
py -3.11 -m venv .venv ; .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt -r requirements-api.txt -r requirements-dev.txt -r frontend\requirements-frontend.txt
copy .env.example .env          # then edit .env (models, keys); see the variable list at the bottom of .env.example

# terminal 1: backend
uvicorn voiceassistant.api:create_app --factory --port 8000

# terminal 2: frontend  (works from any folder)
streamlit run frontend\app.py
```

Try it with **no models at all** (fake providers; answers use the deterministic glossary path):
```powershell
$env:STT_PROVIDER="fake"; $env:LLM_PROVIDER="fake"; $env:TTS_PROVIDER="fake"
uvicorn voiceassistant.api:create_app --factory --port 8000
```
For real speech and language you need the default stack described further down: faster-whisper (STT), Ollama or any
OpenAI-compatible LLM, Piper or pyttsx3 (TTS).

**Run the tests:** `python -m pytest -q` (no models, network or keys needed; the frontend integration tests start a real
backend on a free local port with fake providers). Latency: `python scripts\measure_patient_latency.py` (add `--real` to measure your real providers).

## Patient Voice Assistant

Demo Patient IDs **P001 to P006** (note P001 and P002 are both called Priya Sharma, P006 is Priya Naik).

- **Access:** `POST /v1/patient/session {"patient_id": "P001"}` returns an opaque bearer token. Every other patient call derives the
  patient only from that token. No request field can name another patient; a report ID that is not yours gets the same 404 as one that
  does not exist; unknown IDs, malformed IDs, expired tokens and repeated failures give clear, non-leaky errors.
- **Speak or type:** the microphone uses the existing `/v1/stt` (faster-whisper); answers are optionally spoken with the existing TTS.
- **Ask in your own words.** Examples: "Explain my report.", "Elaborate on my report.", "Describe my findings.", "What are molar teeth?",
  "What does this finding mean?", "Can you explain this in simple English?" Follow-ups reuse the selected report and the previous topic.
- **Grounding:** answers show *From your report* (verbatim stored text, with the report id/type/date and quotes as evidence) separately from
  *In general* (glossary or model background, labelled as not from your report). It refuses diagnosis, treatment/medication advice, severity
  predictions and questions about other people, escalates urgent symptoms, and says so plainly when a report has little detail.
- **Demo limitation: Patient ID-only access is for demonstration only.** Real patient deployments need stronger authentication
  (for example a one-time code to a registered phone, or an identity provider with MFA). See [PRODUCTION_NOTES](docs/PRODUCTION_NOTES.md).

## Model / provider choices

| Role | Default | Alternatives |
|---|---|---|
| Speech-to-text | faster-whisper `base.en` (local) | `fake` for tests |
| LLM | any OpenAI-compatible endpoint, default local Ollama | hosted API via `LLM_BASE_URL`/`LLM_API_KEY`; `fake` |
| Text-to-speech | Piper, falling back to pyttsx3 | `fake` |

The patient flow works **without** an LLM too (rules interpreter + glossary answers). With an LLM it additionally answers definitions of
terms outside the glossary and paraphrases explanations, always behind the guardrails. `PATIENT_USE_LLM=0` forces the deterministic path.

## Known limitations

- **Not verified in the authoring sandbox** (model hosts were unreachable): real Whisper transcription, a real Ollama/hosted LLM's wording,
  real Piper/pyttsx3 audio, live microphone capture with real speech. These need the manual checks in "Manual test procedure" below and
  `python scripts\measure_patient_latency.py --real`.
- Patient ID-only login is a demonstration credential. Sessions are in memory (one process). Rate limiting is per client address.
- Clinician endpoints are unauthenticated unless you set `DENTVOICE_CLINICIAN_API_KEY`; do not expose the backend beyond localhost without it.
- The synthetic reports contain one-line placeholder findings, so explanations are limited to what those lines say plus glossary background.
- The glossary (about 30 terms) is a curated starting point and has not been reviewed by a dentist. Guardrails are heuristic.
- Clinical Dictation's "readable draft" is a faithful echo of the dictation, not a rewritten report (existing behaviour, preserved).

---

# Original voice-pipeline notes (Phase 2)


Turn-based pipeline: **audio file / microphone → validation → speech-to-text → LLM → text-to-speech → playback**,
plus a **typed-input fallback** (clearly labelled "typed input - speech recognition not used").

Default stack: faster-whisper (STT, local) · any OpenAI-compatible LLM, default local Ollama · Piper (TTS, local) with pyttsx3 fallback.
Only Phase 2 exists so far: no conversation memory (Phase 3), no web UI or full reliability work (Phase 4).

## What is verified and what is not

Written and tested in a Linux sandbox with **Python 3.11.17 and no ability to install packages** (PyPI was blocked).

| Item | Status |
|---|---|
| All source, tests and scripts compile (`compileall`) on Python 3.11 | Verified |
| 82 unit tests pass on Python 3.11 and 3.12 (`unittest`); 9 API tests skipped (FastAPI not installable there) | Verified |
| Pipeline logic (validation, STT→LLM→TTS flow, chunking, TTS-failure-keeps-text, fallback TTS, error handling) using **fake** providers | Verified |
| OpenAI-compatible LLM adapter against a **local stub HTTP server** (retries, 429/5xx/401/404, timeout, malformed JSON, connection refused) | Verified against a stub only |
| Piper adapter against a **stub subprocess**; pyttsx3 and faster-whisper adapters against **fake modules** | Verified against stubs only |
| CLI end to end with fake providers; graceful errors when Ollama is down or packages are missing | Verified |
| **Package installation on Windows 11 + Python 3.11** (faster-whisper, piper-tts, pyttsx3) | **NOT verified: you must run it** |
| **Real speech recognition, real Ollama replies, real Piper/pyttsx3 audio, microphone capture, audio playback** | **NOT verified: manual tests below** |
| Whisper confidence threshold (`STT_MIN_CONFIDENCE`) is a sensible value | **NOT verified: heuristic, tune it** |
| Piper CLI flags and voice-download URLs (written from memory) | **NOT verified** |
| `voiceassistant/api.py` and `tests/test_api.py` | **Syntax-checked only; never executed** |

## Windows 11 setup (PowerShell, Python 3.11)

```powershell
# 1. Check Python 3.11 is installed
py -3.11 --version

# 2. From the project folder: create and activate a virtual environment
cd path\to\voice-assistant
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
#   If activation is blocked:  Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass

# 3. Install core + test dependencies
python -m pip install --upgrade pip
pip install -r requirements.txt
pip install -r requirements-dev.txt

# 4. Run the automated mock tests (no models, network or keys needed)
python -m pytest -q
```

Expected: tests pass; the 9 API tests show as skipped until you install `requirements-api.txt`.
(If you prefer no pytest: `python -m unittest discover -s tests -t .` also works.)

### Install Piper (separate, because its Windows compatibility is the least certain part)

```powershell
pip install -r requirements-piper.txt
python scripts\check_env.py
```
If `piper-tts` fails to install or `check_env` reports a Piper problem, skip it: the assistant falls back to pyttsx3 automatically.

Download a voice (URLs written from memory, so check them if they 404; voices are listed at https://huggingface.co/rhasspy/piper-voices):
```powershell
mkdir models
curl.exe -L -o models\en_US-lessac-medium.onnx      "https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/medium/en_US-lessac-medium.onnx"
curl.exe -L -o models\en_US-lessac-medium.onnx.json "https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/medium/en_US-lessac-medium.onnx.json"
```

### Install Ollama (default local LLM)

Install from https://ollama.com/download (or `winget install Ollama.Ollama`; verify the ID with `winget search ollama`), then:
```powershell
ollama pull llama3.2:3b
ollama list
```
On a CPU-only laptop expect a few seconds per reply. To use a hosted API instead, change only `LLM_BASE_URL`, `LLM_MODEL` and `LLM_API_KEY` in `.env`.

### Configure

```powershell
copy .env.example .env
notepad .env
```
Set `PIPER_MODEL_PATH` if you installed Piper. Everything else works with the defaults.

### Optional extras
```powershell
pip install sounddevice            # enables --record (microphone from the command line)
pip install -r requirements-api.txt   # enables the HTTP API
```

## Run

```powershell
python scripts\check_env.py                   # report what is installed / reachable (add --load-models to test Whisper)
python -m voiceassistant.cli --text "Explain machine learning in two sentences"   # typed input
python -m voiceassistant.cli --audio-file my_question.wav                          # audio file
python -m voiceassistant.cli --record 5                                            # microphone, 5 seconds
python -m voiceassistant.cli                                                       # interactive typed loop
uvicorn voiceassistant.api:create_app --factory --port 8000                        # optional HTTP API
```
Useful flags: `--no-play`, `--no-confirm` (skip the transcript review prompt), `--save-audio reply.wav`, `-v`.

Try the wiring with **no models at all** (fake providers):
```powershell
$env:STT_PROVIDER="fake"; $env:LLM_PROVIDER="fake"; $env:TTS_PROVIDER="fake"
python -m voiceassistant.cli --text "hello"
Remove-Item Env:STT_PROVIDER, Env:LLM_PROVIDER, Env:TTS_PROVIDER
```

## Manual test procedure (real providers; you must run these)

Run in order; stop at the first failure and send me the output.

1. **Environment:** `python scripts\check_env.py --load-models`. Expect OK for faster-whisper, Whisper model load, the LLM endpoint, and at least one TTS engine. Note whether Piper shows OK or FAIL.
2. **LLM only (typed):** `python -m voiceassistant.cli --text "Say hello in one sentence" --no-play`. Expect a sensible reply.
3. **TTS + playback (typed):** `python -m voiceassistant.cli --text "Say hello in one sentence"`. You should *hear* the reply. Check which engine spoke: the `(spoken reply: piper|pyttsx3 ...)` line.
4. **STT on a file:** record a short English clip (a `.wav` is safest; Windows Voice Recorder makes `.m4a`, which is allowed but untested) and run `python -m voiceassistant.cli --audio-file clip.wav`. Check the transcript, correct it at the prompt if needed, and listen to the reply.
5. **Microphone:** `pip install sounddevice`, then `python -m voiceassistant.cli --record 5` and speak. Same checks as step 4.
6. **Failure handling:** stop Ollama and repeat step 2. Expect a friendly message, not a traceback. Also try `--audio-file` on an empty file.
7. **Optional API:** `pip install -r requirements-api.txt`, run `python -m pytest -q tests\test_api.py`, then start uvicorn and open http://127.0.0.1:8000/docs.

## Privacy / data flow (default configuration)

- STT (faster-whisper), TTS (Piper/pyttsx3) and temp files run **locally**. The Whisper model is downloaded from the internet once.
- With local Ollama, conversation text stays on your machine. With a hosted LLM, the conversation text (not audio) is sent to that provider.
- Temporary reply audio is deleted after playback. Recordings made by `--record` are deleted after use. Your own `--audio-file` inputs are never modified or deleted.

## Configuration reference

See `.env.example` (every variable is documented there). Invalid values and a missing `LLM_API_KEY` for non-local URLs fail at startup with a clear message. API keys are never printed or logged.

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Could not reach the language model` | Ollama not running, or wrong `LLM_BASE_URL`. Run `ollama list`; default URL is `http://localhost:11434/v1`. |
| `Language model or endpoint not found` | Model not pulled: `ollama pull <LLM_MODEL>`. |
| `Speech recognition is not installed` | `pip install faster-whisper` (inside the activated venv). |
| `Could not load the speech model` | First run needs internet to download the Whisper model. |
| `No Piper voice model is configured` | Set `PIPER_MODEL_PATH` (and keep the `.onnx.json` beside the `.onnx`). Fallback speaks with pyttsx3 meanwhile. |
| `pip install piper-tts` fails | Known risk on Windows; use pyttsx3 for now and send me the error. |
| Recording is "silent or too quiet" | Wrong default input device, or mic muted/blocked in Windows privacy settings. |
| Transcript flagged low confidence | Heuristic only. Speak closer to the mic, or tune `STT_MIN_CONFIDENCE`. |

## Conversation Flow & Silence Timeout (DentVoice AI)

DentVoice AI supports hands-free, voice-first clinical interaction with natural follow-ups and automatic silence timeouts:

- **Configurable Silence Timeouts**:
  - `SILENCE_TIMEOUT_SECONDS = 15`: Inactivity silence timeout after the assistant finishes speaking a completed response.
  - `CLARIFICATION_TIMEOUT_SECONDS = 30`: Extended silence timeout when the assistant asks a clarification question (e.g. asking for patient ID or date of birth during ambiguous patient resolution).
- **Strict Timer Lifecycle**:
  - The silence timer begins strictly **after** the assistant's spoken TTS playback completely finishes.
  - The timer never runs while the assistant is transcribing, thinking, generating drafts, or speaking.
  - New user speech immediately resets the silence timer.
  - Microphone silence / empty audio is never submitted as an answer or empty turn.
  - When the timeout expires, the active voice session ends cleanly (`voice_state = "paused"`, `voice_mode_enabled = False`).
  - An inactivity banner is displayed: *"Voice session paused due to inactivity. You can resume whenever you're ready."*
  - The website remains open, chat history is preserved, and prepared reports/drafts are never discarded. The doctor can click **Resume Voice** at any time.

## Browser Audio & Microphone Considerations

- **Browser Permissions & Autoplay**: Browsers enforce strict user-gesture requirements before permitting microphone recording or automated audio playback. Voice mode starts when the doctor clicks **Enable Voice Mode**.
- **Streamlit Audio Input (`st.audio_input`)**: Streamlit's official audio capture widget operates on a per-utterance recording model (Record -> Speak -> Stop). DentVoice AI pairs this with Faster-Whisper local transcription and live inactivity countdowns, pausing cleanly when silence thresholds are reached without faking continuous background recording.
- English only (`LANGUAGE=en` is the primary supported value).

## Project layout

```
voiceassistant/
  config.py       settings from env/.env (stdlib)        errors.py     typed errors with safe user messages
  audio.py        file validation, WAV join               response.py   TTS text chunking
  pipeline.py     VoicePipeline: transcribe / respond     factory.py    config -> providers -> pipeline
  cli.py          typed / file / microphone front end     api.py        FastAPI layer (optional)
  providers/
    base.py       STT/LLM/TTS interfaces + FallbackTTS
    stt_faster_whisper.py   llm_openai_compat.py   tts_piper.py   tts_pyttsx3.py   fake.py
scripts/check_env.py   tests/   requirements*.txt   .env.example
```
