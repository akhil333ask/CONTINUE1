# SESSION LOG: read this first when resuming

Each step gets an entry here as soon as it is done, so a new session can carry on from the last entry.
Other docs: `INTEGRATION_PLAN.md` (the plan), `PROJECT_STATE.md` (what is verified), `NEXT_STEPS.md` (what to do next).

## Session 5 (2026-10-12): resumed from github.com/akhil333ask/CONTINUE. All 8 plan steps done.

1. DONE: imported the CONTINUE repo into the v0 project. Code was further along than PROJECT_STATE.md said: integration
   plan steps 1–6 (crawler client, claims, verification, all importers, identifier search) were already coded.
2. DONE: baseline `pytest -q` gives 16 passed.
3. DONE: `samples-load ../samples.zip` failed with `ModuleNotFoundError: openpyxl`. Added `openpyxl`, `xlrd` and `flask` to
   `backend/requirements.txt`. After that, all 8 sample files loaded in about 12 s: MCA 200, GLEIF L1 20, GLEIF L2 20 rels (+19 stubs),
   ICIJ 500 nodes (100 rels skipped, no endpoints in the sample), OpenSanctions 100, NSE 4,344 + 11,777 rows.
   A second run created 0 new rows, so imports are idempotent. `investigate <CIN>` resolves to the NSE debarred record.
4. DONE: added `backend/tests/test_integration.py` (13 tests) plus `tests/fixtures_crawler_wabtec.json`, the documented
   crawler response. Covered: the real samples load and are idempotent; GLEIF L2 subject = StartNode (child) and
   object = EndNode (parent); CIN/PAN search; crawler cache, no retry on 500, bounded 503 retries, errors not cached,
   RPM + daily budget, 5 concurrent identical calls = 1 upstream POST, not-configured. Claims: the Wabtec US-parent
   crawl gets the Indian subsidiary's CIN, which is stored as `related` and not merged. Placeholders are dropped and
   no crawler graph edges are created. Matching CIN + name gives `verified`.
   Fix: the overview claim had no source URL. It now uses the reproducible Google search URL (claims.py).
   `pytest -q` gives **29 passed**.
5. DONE: UI. Added `components/osint/intel-view.tsx`, a new default "Intelligence" tab: query by CIN/LEI/PAN/DIN/name;
   an opt-in crawl checkbox that warns about credits; detected identifiers; Subject, Verified facts, Connections
   (parents/subsidiaries kept separate), Unverified leads (status, confidence, method, source URL, retrieval time),
   Caveats. Added a `CrawlerPanel` (health + usage) in that tab and in "Sources & AI budget".
   Browser-verified against the sample DB: CIN L29246GJ1988PLC010888 → NSE debarment fact with source URL.
6. DONE (zero-credit live test): ran the real `services/crawler/app.py` with no SERPER_API_KEY.
   `crawler-health` → reachable, serper_configured=false. `investigate "Wabtec Corporation" --crawl` → crawler HTTP 500
   "Missing SERPER_API_KEY", **not retried** (1 attempt in the usage ledger). Real HTTP path proven; no credits spent.
   NOT done: a credit-spending live crawl (no key provided). See NEXT_STEPS.md for the exact single command.
7. DONE: updated PROJECT_STATE.md, NEXT_STEPS.md, CHANGELOG.md.

## How to run (any machine)
```
cd backend && python -m venv .venv && .venv/bin/pip install -r requirements.txt   # Windows: .venv\Scripts\python.exe -m pip ...
.venv/bin/python -m pytest -q                         # expect 29 passed
.venv/bin/python -m app.cli samples-load ../samples.zip
.venv/bin/uvicorn app.api:app --port 8000
cd .. && pnpm install && pnpm dev                     # http://localhost:3000
```
Crawler (optional): `cd services/crawler && pip install -r requirements.txt && SERPER_API_KEY=... python app.py`
(port 10000), then set `CRAWLER_URL=http://127.0.0.1:10000` in `backend/.env`.
