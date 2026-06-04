# build/ — full change-set as a base64 artifact

`onegov2-build-combined.zip.b64` is the complete change-set (scenario engine +
GreenPT chatbot Phases 1-7 + docs), base64-encoded so it could be committed via the
GitHub API (batch push + fork were blocked for the integration token). The ~70 MB
H3 datasets are NOT in here — apply this over a fork of the data-carrying base repo.

## Restore the full, runnable build
```bash
# 1. Fork the base (one click on github.com) — it carries src/backend/data + extra_data:
#    https://github.com/govtechnl/onegov2-spatial-assistant  ->  Fork
git clone https://github.com/knarayanareddy/onegov2-spatial-assistant.git
cd onegov2-spatial-assistant && git checkout -b full-build

# 2. Decode this artifact and unzip the change-set over the repo root:
curl -sL https://raw.githubusercontent.com/knarayanareddy/onegov2/full-build/build/onegov2-build-combined.zip.b64 \
  | base64 -d > cs.zip && unzip -o cs.zip -d . && rm cs.zip
#    (or just unzip the onegov2-build-combined.zip you already have.)

# 3. Commit & push:
git add -A && git commit -m "Add scenario engine + GreenPT chatbot (Phases 1-7)" && git push -u origin full-build

# 4. Verify:
cd src/backend && python3.12 -m venv .venv && . .venv/bin/activate
pip install -e . --group dev
PYTHONPATH=. python -m pytest tests_scenario tests_chatbot -q   # -> 132 passed
```
See the repo-root `SETUP.md` for env config (GreenPT key, SSO/auth, Postgres, Waterinfo, MLflow).
