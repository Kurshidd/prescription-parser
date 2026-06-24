# 💊 Prescription Parser

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)
[![Streamlit](https://img.shields.io/badge/UI-Streamlit-FF4B4B.svg)](https://streamlit.io/)

Upload a prescription (image, PDF, or pasted text) and get back
**structured, plain-language medicine info** — drug name, strength,
frequency, duration, side effects, and warnings — with an honest
confidence score and a `needs_review` flag wherever the system isn't sure.

Built around real drug data (RxNorm, OpenFDA) rather than guesswork, and
designed so uncertainty is surfaced, never hidden.

```
Tab Azithromycin 500mg OD x 3 days
        ↓
Medicine: Azithromycin
Strength: 500mg
Frequency: Once Daily
Duration: 3 Days
Explanation: This is an antibiotic used to treat bacterial infections.
```

## Why it's built this way

Prescription parsing is safety-critical: a wrong drug name or dose isn't
a cosmetic bug. The architecture is built around one core principle —
**never silently guess**. Every stage that makes a judgment call (OCR
confidence, fuzzy drug matching, missing fields) propagates that
uncertainty downstream instead of hiding it, and the final output always
flags when a human (pharmacist/doctor) should double-check a line.

## Features

- 📷 **Multi-input**: JPG/PNG/PDF upload, or paste raw prescription text
- 🔍 **Two OCR engines**: Tesseract (free, local, great for printed text)
  and Google Cloud Vision (better for handwriting)
- 💊 **Real drug verification** via [RxNorm](https://lhncbc.nlm.nih.gov/RxNav/) (NIH), with offline fallback
- ⚕️ **Real clinical info** (side effects, interactions, warnings) via [OpenFDA](https://open.fda.gov/)
- 🔶 **Confidence + review flags** on every extracted field — nothing is silently assumed correct
- 🖥️ **Streamlit UI** included, or use the pipeline directly as a Python library

## Project structure

```
prescription-parser/
├── app.py                          # Streamlit frontend
├── pipeline.py                     # orchestrates all 5 stages
├── schemas.py                      # shared data contracts (Pydantic models)
├── requirements.txt
│
├── input/
│   ├── loader.py                   # detects JPG/PNG vs scanned-PDF vs text-PDF
│   └── preprocess.py                # deskew, denoise, threshold
│
├── ocr/
│   ├── ocr_engine.py                # Tesseract OCR
│   ├── cloud_ocr.py                 # Google Cloud Vision OCR (handwriting)
│   ├── pdf_text.py                  # direct text extraction for text-based PDFs
│   └── pdf_to_images.py             # rasterizes scanned PDF pages for OCR
│
├── extraction/
│   ├── rules.py                     # regex-based entity extraction
│   └── abbreviations.json           # OD/BD/TDS/QID → full meaning
│
├── normalization/
│   ├── rxnorm_client.py             # real drug-name verification (RxNorm API)
│   ├── drug_matcher.py              # local fuzzy-match fallback
│   ├── drug_database.json           # offline fallback drug reference
│   └── normalizer.py                # orchestrates matching + review flags
│
├── simplify/
│   ├── openfda_client.py            # side effects/interactions/warnings (OpenFDA API)
│   └── drug_explainer.py            # plain-language explanations
│
└── tests/
    └── test_pipeline.py             # regression tests
```

Every stage reads/writes the shared schemas in `schemas.py`, so any stage
can be swapped later (e.g. rule-based extraction → an ML NER model)
without touching anything else.

## Pipeline

```
Input (JPG/PNG/PDF/text)
   → Stage 1: input/loader.py            (classify input type)
   → Stage 1b: input/preprocess.py       (image cleanup)
   → Stage 2: ocr/*.py                   (OCR or direct PDF text)
   → Stage 3: extraction/rules.py        (regex → structured fields)
   → Stage 4: normalization/*.py         (RxNorm verification, abbreviation expansion)
   → Stage 5: simplify/*.py              (plain-language + clinical info)
   → Output: structured JSON + confidence + needs_review flags
```

## Setup

```bash
git clone https://github.com/<your-username>/prescription-parser.git
cd prescription-parser
python3 -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Tesseract itself (the OCR engine) is a system binary, not a pip package:

```bash
# macOS
brew install tesseract
# Ubuntu/Debian
sudo apt-get install tesseract-ocr
# Windows
# https://github.com/UB-Mannheim/tesseract/wiki
```

RxNorm and OpenFDA need **no setup** — they're free, public, unauthenticated APIs.

### Optional: Google Cloud Vision (for handwritten prescriptions)

```bash
# 1. Create a Google Cloud project, enable the "Cloud Vision API"
#    https://console.cloud.google.com/apis/library/vision.googleapis.com
# 2. Create a service account, download its JSON key
# 3. Copy .env.example to .env and set the path, or export directly:
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/your-key.json"
```

Without this set, selecting "Google Vision" as the OCR engine raises a
clear `CloudOCRNotConfigured` error rather than failing silently —
Tesseract remains the default and needs no setup at all.

## Run the app

```bash
streamlit run app.py
```

Opens at `http://localhost:8501`. Upload a prescription or paste text,
pick an OCR engine, and parse.

## Use as a Python library

```python
from pipeline import run_pipeline, run_pipeline_on_text
from schemas import OCREngine

# Full pipeline (image or PDF file on disk)
result = run_pipeline("path/to/prescription.jpg", ocr_engine=OCREngine.GOOGLE_VISION)

# Text-only shortcut (skips OCR — useful for testing extraction/normalization)
result = run_pipeline_on_text("Tab Azithromycin 500mg OD x 3 days")

print(result.dict())
```

Output shape:
```json
{
  "medicines": [
    {
      "medicine": "Azithromycin",
      "strength": "500mg",
      "frequency": "Once Daily",
      "duration": "3 day",
      "explanation": "This is an antibiotic used to treat bacterial infections.",
      "needs_review": false,
      "review_reason": null,
      "clinical_info": {
        "side_effects": ["Nausea", "Diarrhea", "Abdominal pain"],
        "interactions": ["..."],
        "warnings": ["..."],
        "contraindications": ["..."],
        "source": "openfda"
      }
    }
  ],
  "overall_confidence": 0.86,
  "raw_ocr_text": "Tab Azithromycin 500mg OD x 3 days"
}
```

## Run tests

```bash
python3 tests/test_pipeline.py
```

## What's real vs. local fallback

| Component | Current state | Notes |
|---|---|---|
| Drug name matching | **RxNorm API** (real, ~100k+ names), local JSON as offline fallback | `normalization/rxnorm_client.py` |
| Clinical info | **OpenFDA API** (real, FDA label data) | `simplify/openfda_client.py` — returns `None` gracefully if unavailable |
| Entity extraction | Regex + abbreviation dictionary | Swap-in path: spaCy/scispaCy NER once you have labeled real-world data |
| OCR (printed text) | Tesseract (free, local) | `ocr/ocr_engine.py` |
| OCR (handwriting) | **Google Cloud Vision** | `ocr/cloud_ocr.py` — meaningfully better than Tesseract, not perfect |
| Explanations | Static curated mapping | Any future LLM use should *rephrase* a verified fact, never invent one |

## Known limitations

- **Handwriting OCR is genuinely hard.** No OCR engine reliably solves
  messy doctor scripts — this project surfaces low confidence rather
  than pretending to solve it.
- **Local drug database is small (10 drugs)**, used only as an offline
  fallback when RxNorm is unreachable.
- **Not a substitute for pharmacist/doctor review.** The `needs_review`
  flag is the most important field in the output.
- **Regulatory note:** if used with real patient data, check the
  relevant health-data regulations for your jurisdiction (e.g. HIPAA in
  the US, DPDP Act in India) before handling real prescriptions.

## Roadmap

- [ ] Expand abbreviation/drug coverage from real-world prescription samples
- [ ] Multi-medicine table/column layout parsing
- [ ] spaCy/scispaCy NER model trained on labeled real prescriptions
- [ ] REST API wrapper (FastAPI) alongside the Streamlit UI

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) — covers project conventions,
especially around how uncertainty must be handled.

## License

[MIT](LICENSE)

## Disclaimer

This tool is for educational/informational purposes. It is **not**
a medical device and does not replace professional medical or
pharmacist advice. Always verify prescription details with a licensed
healthcare provider.
