# HROne — Employee Attendance & Analytics API (implemented)

All code in `app/main.py`. Contract: `openapi.yaml` (R1–R10). Storage: `DATA_MODEL.md`.

## Run
```bash
pip install -r requirements.txt
# MongoDB 6.0+ (grader 7.0): docker run -p 27017:27017 mongo:7
export MONGO_URI=mongodb://localhost:27017 MONGO_DB=attendance_db
python sample_seed.py
uvicorn app.main:app --port 8000
curl http://localhost:8000/health  # {"status":"ok"} in <20s
```
Real env vars win over `.env`. Indexes are created idempotently at startup. No `.env`/secrets committed.

## What is implemented
- `GET /health` (ping-gated 503), `POST/GET /employees`, `POST /attendance/punch-in`, `GET /attendance` (fixed: IST R1/R2, half-up R4/R8, truncation, race-safe unique indexes, DB-side sort/paginate, exact shapes).
- `POST /attendance/punch-out` (most-recent `punch_in<=punched_at`, cross-midnight, R3–R5, atomic single-200).
- `PATCH /attendance/{emp_code}/{date}` (recompute + `$push` history with only-changed `from/to`, ABSENT/LEAVE/date-containment/no-op rules).
- Analytics as aggregations (stored values, no Python loops): employee monthly, department summary (`$lookup` by `emp_code`, zero-log headcount), late leaderboard (`$rank` 1,2,2,4, limit-after-rank), department trend (`$densify` + `$setWindowFields` 7d avg).
- `GET /admin/explain/{endpoint}` with `executionStats` (IXSCAN, no COLLSCAN).

## Notes
- IST (`UTC+5:30`) everywhere; instants truncated to seconds, epoch-ms at API, BSON dates in Mongo (`tz_aware=True`).
- Analytics read stored derived fields. Missing `history`/`half_day` treated as `[]`/`false`. No-doc weekday ≠ `ABSENT`.
- See `REVIEW.md` (12 starter defects), `DECISIONS.md`, `PRD.md`/`ROADMAP.md`/`todo.md`.
