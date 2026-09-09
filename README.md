# SatQuery AI

**SIH26167 — Interactive Vision-Language Assistant for Multimodal Remote Sensing Image Analysis through Text Queries**
Organization: ISRO · Team submission for Smart India Hackathon 2026

Upload a satellite/aerial image, ask a question in plain English, get a grounded,
structured answer. Also supports comparing two images of the same area to describe
visible change.

This is a genuinely working system: images are actually sent to a vision-language
model (Google Gemini) and the answer you see is generated from that specific image,
not a canned response.

---

## 1. How it works (architecture)

```
React frontend (Vite)
        │  image + question, multipart/form-data
        ▼
FastAPI backend  ──►  validate file type/size
        │
        ▼
Image preprocessing (Pillow)  ──►  resize / crop / (tiling ready for large images)
        │
        ▼
Prompt builder  ──►  picks a remote-sensing-aware prompt template based on the question
        │
        ▼
Vision service (isolated) ──►  Gemini 2.0 Flash Vision API
        │
        ▼
Response parser  ──►  {answer, observations, features, location, confidence}
        │
        ▼
JSON  ──►  React renders a structured "answer readout" card
```

Every arrow above is a real file boundary in the code (see section 3), so any single
piece — the model, the prompts, the frontend — can be replaced without touching the rest.

## 2. Why Gemini Vision for the MVP (and not something else)

| Approach | Why / why not |
|---|---|
| **Gemini Vision API (chosen)** | Free tier, no GPU needed, reliable during a live demo, genuinely handles aerial/satellite imagery reasonably well out of the box. |
| Self-hosted open-source VLM (LLaVA, Qwen-VL) | Needs a strong GPU most student laptops don't have; too fragile to depend on for a live judged demo. Good future upgrade if your college has GPU access. |
| Traditional CV / classifiers (EuroSAT-trained CNN etc.) | Great for quantitative backing (see Phase 9 below) but can't answer open-ended questions on its own. |
| RS-specific research VLMs (GeoChat, RSGPT) | Immature tooling, high setup risk — not worth it for a 36-48hr build. |

The `services/vision_service.py` file is the **only** file that imports the Gemini SDK.
To swap models later, you only edit that one file.

## 3. Project structure

```
satquery-ai/
├── backend/
│   ├── main.py                 # FastAPI app entrypoint
│   ├── routes/analyze.py       # /api/analyze, /api/compare, /api/analyze-region
│   ├── services/
│   │   ├── vision_service.py   # ONLY file that talks to the AI model
│   │   └── image_processor.py # resize, tiling, region-cropping
│   ├── prompts/templates.py    # all prompt engineering lives here
│   ├── utils/validators.py     # input validation, friendly error messages
│   ├── config/settings.py      # reads .env, single source of config
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── App.jsx              # tab navigation (analyze / compare)
│   │   ├── api.js                # the only file that calls the backend
│   │   └── components/           # UploadSlot, AnswerReadout, AnalyzeView, CompareView, HistoryLog
│   └── package.json
├── .env.example
└── README.md   (this file)
```

## 4. Setup (Windows) — exact commands, beginner-friendly

### Step 1 — Install Python
Download Python 3.11+ from https://python.org/downloads and during install,
**check "Add Python to PATH"**. Confirm it worked:
```
python --version
```

### Step 2 — Install Node.js
Download the LTS version from https://nodejs.org. Confirm:
```
node --version
npm --version
```

### Step 3 — Get a free Gemini API key
Go to https://aistudio.google.com/app/apikey → "Create API key" → copy it.
(This is free for the usage levels a hackathon demo needs.)

### Step 4 — Backend setup
```
cd satquery-ai\backend
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
copy ..\.env.example .env
```
Now open `backend\.env` in Notepad and paste your real key into `GEMINI_API_KEY=`.

Run the backend:
```
uvicorn main:app --reload --port 8000
```
Leave this terminal open. Visit http://localhost:8000/docs — if you see the FastAPI
docs page, the backend is alive. Visit http://localhost:8000/health — it should say
`"ai_configured": true` once your key is in place.

### Step 5 — Frontend setup (in a NEW terminal)
```
cd satquery-ai\frontend
npm install
npm run dev
```
Open the URL it prints (usually http://localhost:5173).

### Step 6 — Use it
1. Upload a satellite image (JPG/PNG/TIFF).
2. Type or click a suggested question.
3. Click "Analyze" and watch the structured answer appear.
4. Try the "Compare two images" tab with two images of the same area.

## 5. Test procedure

- **Sanity check**: upload any clear daytime aerial photo of a city or coastline
  and ask "Describe this image" — you should get a plausible, specific description,
  not a generic paragraph.
- **Water test**: use an image with a visible river/lake/coastline, ask
  "Is there a water body?" — answer should mention it and roughly where.
- **Error handling test**: try uploading a `.pdf` — you should get a clean
  "unsupported file type" message, not a crash.
- **Comparison test**: upload the same image twice as Image A and B, ask
  "what changed?" — it should correctly say nothing significant changed
  (this proves it isn't hallucinating differences).

## 6. Recommended datasets (for development/testing images, and future training)

Start with **EuroSAT** first:
- **What**: 27,000 labeled Sentinel-2 patches across 10 land-cover classes
  (residential, industrial, river, forest, etc.)
- **Why first**: small (~2GB), clean, well-documented, perfect for quickly testing
  your pipeline end-to-end and for grabbing realistic sample images for your demo.
- **Format**: 64×64 RGB/multispectral GeoTIFF or JPEG patches.
- **Get it**: `https://github.com/phelber/EuroSAT` or via Hugging Face `datasets` (`eurosat`).
- **Preprocessing**: none needed for demo purposes beyond resizing up if you want
  a larger, more "satellite-like" looking demo image.

For later stages / stronger claims to judges:
- **UC Merced Land Use** — 21-class aerial imagery, good for land-cover Q&A variety.
- **LEVIR-CD** — building change-detection image pairs, ideal for stress-testing
  the comparison feature with real "before/after" pairs.
- **BigEarthNet** — large-scale multispectral (Sentinel-2), useful once you build
  real multispectral support (Phase 9).
- **SpaceNet** — building/road footprints, useful if you add object-detection overlays.

Don't download all of these — EuroSAT + a handful of LEVIR-CD pairs for the
comparison demo is enough for the hackathon.

## 7. Evaluation framework

Since this is a VLM-based system (not a classifier with fixed labels), measure:

- **Response latency** — already logged automatically in every API response (`latency_ms`).
- **VQA accuracy** — manually curate ~20 image+question+expected-answer pairs from
  EuroSAT/UC Merced (e.g. an image labeled "River" → question "is there a water body?"
  → expected "yes") and score how many the system gets right. This gives you a real,
  honest accuracy number for your presentation instead of an invented one.
- **Feature-detection precision/recall** — compare the `features` list the system
  returns against the dataset's ground-truth label for that image.
- **Change-detection sanity** — the "same image twice → no change" test above is a
  simple, honest regression check.

Do not report numbers you haven't actually measured — an honest "on our 20-sample
test set we got X% VQA accuracy" is far more credible to judges than an invented 95%.

## 8. Known MVP limitations (be upfront about these to judges)

- Uses ordinary RGB imagery — no multispectral (NIR/SWIR band) analysis yet.
- No true geo-referencing — the system never claims real-world coordinates by design.
- Change detection is AI-interpreted, not pixel-level quantitative change detection.
- Requires an internet connection (cloud API) — see Phase 11 below for an offline path.

## 9. Roadmap — what to build next, in order

**Phase 9 — Quantitative backing (highest judge-impact, moderate effort)**
Add an OpenCV-based NDVI-style vegetation heuristic and a simple water-mask
(HSV threshold on blue/dark regions) that runs alongside the VLM call and is
shown as a secondary "quantitative estimate" next to the AI's qualitative answer.
This is the single best upgrade for credibility: it shows real CV, not just an API call.

**Phase 10 — Multispectral support**
Swap Pillow for `rasterio` when the uploaded file is a multi-band GeoTIFF; extract
an RGB composite for the VLM call, and compute real NDVI from NIR+Red bands for
the Phase 9 overlay. `image_processor.py` is structured so this is a drop-in change.

**Phase 11 — Offline fallback**
Wrap `vision_service.py`'s call in a try/except that falls back to a small local
model (e.g. a quantized LLaVA via `ollama`) if the Gemini call fails — useful if
the venue's internet is unreliable during the demo. Only add this if you have
time to test it; an unreliable fallback is worse than none.

**Phase 12 — Large-image tiling**
`generate_tiles()` in `image_processor.py` is already implemented and unit-testable.
Wire it into `/api/analyze` when `needs_tiling()` is true: run the VLM on each tile,
then ask the VLM a final "synthesis" pass over the per-tile summaries.

## 10. Demo script (3–5 minutes)

1. Open the app, show the clean upload screen.
2. Upload a satellite image → ask "Describe this image" → show the structured answer.
3. Ask "Is there a water body?" then "Are there buildings?" — show it reasons per-question,
   not from a cached answer.
4. Switch to "Compare two images" → upload a before/after pair → ask "what changed?"
5. Point at the architecture diagram (section 1) and explain the model-swap design
   and the honesty guardrails (no invented coordinates, explicit confidence).
6. Close with real-world use cases (below) and the roadmap.

## 11. Real-world use cases to mention to judges

Urban planning, flood/disaster assessment, agricultural monitoring, forest monitoring,
infrastructure monitoring, and general land-use interpretation for regions where
manual analyst review doesn't scale — explicitly **not** claiming production-readiness,
positioning this as an assistant that accelerates a human analyst's first pass.

## 12. Likely judge questions

**"How do you prevent hallucination?"** → Prompt guardrails force separation of
observation vs inference, explicit confidence levels, and a hard rule against
inventing coordinates/measurements — see `prompts/templates.py`.

**"Why not train your own model?"** → Time/data/compute constraints of a hackathon;
a general VLM with domain-specific prompting is the pragmatic MVP choice, with a
clear roadmap (Phase 9-10) toward quantitative, remote-sensing-specific outputs.

**"Does this work on real satellite data (multispectral, large scenes)?"** → Not yet —
that's an explicitly stated MVP limitation with a concrete, already-scaffolded
upgrade path (tiling and rasterio support are implemented and ready to wire in).

**"What's your accuracy?"** → Point to the evaluation framework (section 7) and
whatever real numbers you've measured by demo day — don't invent a number.
