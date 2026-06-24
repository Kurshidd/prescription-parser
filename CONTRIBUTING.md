# Contributing

Thanks for considering a contribution! This project follows a few
conventions worth knowing before you open a PR.

## Project philosophy

This is a safety-critical-adjacent tool — incorrect drug, dosage, or
frequency output isn't a cosmetic bug. Please keep these principles in
mind for any contribution:

1. **Never silently guess.** If a stage isn't confident about a result
   (OCR text, drug name match, missing field), it should propagate that
   uncertainty via `needs_review` / `review_reason`, not hide it.
2. **Each pipeline stage stays independent.** Stages communicate only
   through the shared models in `schemas.py`. Don't reach into another
   stage's internals — if you need new data passed between stages, add
   a field to the relevant schema instead.
3. **External API calls must degrade gracefully.** Network calls (RxNorm,
   OpenFDA, Google Vision) should catch their own exceptions and fall
   back or return `None` — never let one flaky external call crash the
   whole pipeline. See `normalization/rxnorm_client.py` for the pattern.
4. **No invented medical facts.** Explanations and clinical info come
   from curated/verified sources (`drug_database.json`, OpenFDA), never
   freely generated. If you add an LLM-based feature, it may rephrase a
   verified fact but must not be the source of the fact itself.

## Setup for development

```bash
git clone https://github.com/<your-username>/prescription-parser.git
cd prescription-parser
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

## Running tests

```bash
python3 tests/test_pipeline.py
```

Please add a test for any new extraction rule, abbreviation, or
normalization behavior — especially edge cases that previously broke
something (see the test docstrings for examples of bugs we've already
caught this way).

## Adding new abbreviations or drugs

- Frequency/route/form abbreviations: `extraction/abbreviations.json`
- Local fallback drug entries: `normalization/drug_database.json`

Keep entries alphabetically sorted within their section where practical,
and prefer real-world abbreviations you've actually seen on a
prescription over speculative ones.

## Pull requests

- Keep PRs focused on one change/stage where possible.
- Describe what real-world case motivated the change (e.g. "OCR
  misreads 'BD' as '8D' on low-quality scans").
- Run the test suite before submitting.
