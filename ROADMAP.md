# Roadmap — HROne Attendance & Analytics API

Guide time: 4–5h (Review 45m, Build 2.5–3h, Prove & explain 30m). Stack: FastAPI + MongoDB 6.0+.

## Milestones & Exit Criteria

### M0 — Setup & Test Harness (15–20 min)
- Tasks: venv + `pip install -r requirements.txt`, local Mongo (Docker `mongo:7` or Atlas M0), `.env` (env-wins), `sample_seed.py` load, verify `uvicorn app.main:app --port 8000` + `GET /health`.
- Build adversarial fixtures: overnight shift, 09:40:00/01/10:15:59 lates, <30/≥30 overtime, <4.50 half-day, weekend logs, tie at limit, zero-log employee, mid-month joiner, missing weekday vs ABSENT, legacy docs missing `history`/`half_day`.
- Exit: health 200, sample shapes load, harness can run parallel-request races.

### M1 — Part A: Review & Fix Starter (45 min, 10% weight)
- Fix helpers: IST-aware late/overtime (replace naive `replace`/`fromisoformat`), half-up rounding (Decimal, not `round`), truncation to seconds, overnight date logic.
- Fix endpoints: health ping-gated (503 on fail), employee unique-index 409 (not check-then-insert), `created_at` UTC now + epoch-ms out, list pagination (`skip=(page-1)*size`, filtered `total`, sort `emp_code`), punch-in TZ/date/validation/race-safe insert + response shape (no `id`, epoch-ms), attendance list DB-side filter/sort/paginate (no in-memory load/sort, strip `_id`, epoch-ms conversion, `date_from>date_to` 422).
- Fill `REVIEW.md` (where/what/how-noticed/fix + non-defects).
- Exit: starter endpoints pass validation matrix + race smoke.

### M2 — Transactional Build: Punch-out + Regularize (45–60 min, part of 30%)
- Punch-out: find most recent `punch_in <= punched_at` (cross-midnight), validate `punched_at>punch_in` within 24h, atomic close (one 200), compute R3–R5, codes 404/409/422.
- PATCH `/{emp_code}/{date}`: apply only `status/punch_in/punch_out`, enforce ABSENT/LEAVE↔punch exclusivity, presence-requires-punch, R1 date containment, recompute + append single history entry with only-changed `from/to`, no-op→422, atomic history push (409 on conflict).
- Exit: overnight close, recompute, and concurrent PATCH no-loss verified.

### M3 — Analytics Pipelines (60–75 min, 30% weight) — the decider
- Employee-monthly: aggregation for sums/counts; `working_days`/`attendance_pct` per R7–R8; weekend handling.
- Department summary: `$lookup` by `emp_code`, headcount incl. zero-log excl. future-joiners, average over records.
- Leaderboard: `$group` → `$setWindowFields` competition rank (1,2,2,4) → filter `rank<=limit` → sort total desc, `emp_code` asc; department-scoped variant.
- Trend: calendar spine via densify, per-day headcount/present/late/rate, in-range 7-day moving avg; 404/422 guards (92-day cap).
- Exit: all four match hand-computed adversarial fixtures; no Python loops.

### M4 — Part C: Explain + Indexes + Perf (30 min, 15% weight)
- Create indexes idempotently at startup: unique employee code, unique (`emp_code`,date), list/pipeline support indexes; document one rejected index.
- Implement `GET /admin/explain/{endpoint}` reusing real builders, `executionStats`, `{endpoint,collection,explain}`; verify IXSCAN present + COLLSCAN absent on ~100k docs; latency check list + analytics.
- Fill `DECISIONS.md` (150–200 words: indexes, race walkthrough, tie cutoff, zero-log headcount, 100x change).
- Exit: explain passes on 100k dataset.

### M5 — Hardening & Submission (15–30 min, 15% weight)
- Global: never 500, exact field names, env-wins config, <20s readiness, concurrency re-run, no fetch-loop remains.
- Update `README.md` (run + gaps), verify repo has only app code + requirements + REVIEW/DECISIONS/README, no `.env`/secrets/Dockerfile/venv/dumps; clean-clone `pip install + uvicorn` test.
- Exit: submission checklist (Sec. 12 of problem statement) green.

## Dependencies / Critical Path
Rule engine → atomic writes (M2) → pipelines (M3) → indexes/explain (M4). M1 unblocks all. M5 needs M1–M4.

## Risks
1. TZ/rounding shortcuts → systematic R1–R4 failure — mitigate with isolated rule-engine unit tests.
2. Check-then-insert races → duplicate days — mitigate with unique indexes + parallel-race tests.
3. Array-index rank / Python gap-fill / avg-of-averages → analytics failure — mitigate with window/densify/lookup pipelines + tie/zero-log fixtures.
4. COLLSCAN on 100k → perf failure — mitigate with M4 explain gate.
5. 500s / wrong field names → auto-fail — mitigate with strict validation + contract test pass.

## What to Demo Live
Walk through rule engine, race-safe punch-in/out, one history diff, leaderboard rank-vs-limit query, trend moving window, and one explain plan.
