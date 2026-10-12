# TODO — HROne Attendance & Analytics API

Source of truth: `PROBLEM_STATEMENT.md` > `openapi.yaml` (contract + R1–R10) > `DATA_MODEL.md` (storage). Guide time: 4–5h.

## 0. Setup
- [ ] `python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
- [ ] Start Mongo 6.0+ (local `docker run -p 27017:27017 mongo:7` or Atlas M0), set `MONGO_URI` / `MONGO_DB` (env vars must win over `.env`)
- [ ] `python sample_seed.py` loads 9 sample docs; verify `uvicorn app.main:app --port 8000` + `GET /health` 200 in <20s
- [ ] Build adversarial test data: overnight shift, grace edges (09:40:00/01, 10:15:59), overtime <30/≥30, half-day <4.50, weekend logs, tie at `limit`, zero-log employee, mid-month joiner, missing weekday vs ABSENT, legacy docs missing `history`/`half_day`

## A. Review starter `app/main.py` (45 min, 10%)
- [ ] Audit helpers: `compute_late_minutes` (IST, grace strictly >10:00, floor), `compute_work_hours` (half-up via Decimal, not `round`), `compute_overtime` (IST, floor, ≥30 gate, overnight shift_end next day), add truncation-to-seconds + attendance-date R1 helper
- [ ] Fix `GET /health`: ping Mongo, 503 on fail (not unconditional 200)
- [ ] Fix `POST /employees`: DB unique index on `emp_code` → 409 on race, validate `EMP\d{4,6}`, email, `HH:MM`, `shift_start != shift_end`, `joined_on`; server-set UTC `created_at` → epoch ms out; never expose `_id`
- [ ] Fix `GET /employees`: sort `emp_code` asc, `skip=(page-1)*size`, `total` over filtered set, validate page/page_size (R10)
- [ ] Fix `POST /attendance/punch-in`: 404 unknown emp, IST date R1 + overnight rule, accept only PRESENT/WFH/ON_DUTY, strict epoch-ms validation, race-safe insert via unique (`emp_code`,date) → 409, init `punch_out/work_hours=null, overtime=0, half_day=false, history=[]`, epoch-ms out, no `id` field
- [ ] Fix `GET /attendance`: DB-side filter/sort (`date` desc, `emp_code` asc)/paginate, `total` reflects filters, `date_from>date_to` → 422, strip `_id`, epoch-ms conversion
- [ ] Fill `REVIEW.md`: table [# | where | what wrong | how noticed | fix] + non-defects section

## B. Build missing endpoints (2.5–3h, 60% total)
- [ ] `POST /attendance/punch-out`: close most recent `punch_in <= punched_at` (cross-midnight); 404/409 (already closed, single 200 on race via atomic update)/422 (`punched_at<=punch_in` or >24h after); compute R3–R5
- [ ] `PATCH /attendance/{emp_code}/{date}`: only `status/punch_in/punch_out` mutable + required `reason` (5–200) + `regularized_by` (1–50); rules: ABSENT/LEAVE clears punches (punch+absent → 422), presence needs `punch_in`, `punch_in` stays on R1 date, `punch_out` after `punch_in` ≤24h, no-op → 422, 404 missing; recompute derived + append 1 `history` entry with only-changed `from/to`; atomic push (409 on concurrent conflict acceptable)
- [ ] `GET /analytics/employees/{emp_code}/monthly?month=YYYY-MM`: aggregation; `working_days` R7 (Mon–Fri from `joined_on`), `present_days` Mon–Fri half=0.5, `leave_days`, `late_count` (`late_minutes>0`), sums, `attendance_pct` R8 or null; read stored values; 404/422
- [ ] `GET /analytics/departments/summary?month=&department=`: cross-collection aggregation, sort dept asc; `headcount` R9 incl. zero-log excl. future-joiner, omit headcount-0; `present_days`, `avg_work_hours` (mean over present+non-null records, R8), `late_count/total_late_minutes`, `leave_count`, `on_duty_count`
- [ ] `GET /analytics/leaderboard/late?month=&limit=&department=`: rank by total `late_minutes>0` desc; competition rank 1,2,2,4 via window fn (not array index); `limit` after ranking; order total desc, `emp_code` asc; join name/dept, ignore orphan logs
- [ ] `GET /analytics/departments/{department}/trend?from=&to=`: exactly 1 row/day incl. weekends/empty via densify (no Python loop); `is_working_day`, `headcount` R9 per-day, `present_count` (half 0.5), `late_count`, `attendance_rate` 4-dec (working day + headcount>0 else null; empty working day = 0.0), `moving_avg_7d` in-range window; 404 unknown dept; 422 `to<from` or >92d

## C. Prove & explain (30 min, 15%)
- [ ] Create all indexes at startup idempotently: unique `emp_code`, unique (`emp_code`,date), + list/pipeline support indexes; decide + document 1 rejected index
- [ ] `GET /admin/explain/{endpoint}` (`attendance_list|employee_monthly|department_summary|late_leaderboard|department_trend`): same builder as real endpoint, `explain(executionStats)`, return `{endpoint,collection,explain}`; missing params → 422; verify IXSCAN present, COLLSCAN absent on ~100k docs
- [ ] Fill `DECISIONS.md` (150–200 words total): 1 indexes+why/rejected, 2 punch-in race walkthrough, 3 tie at cutoff, 4 zero-log headcount, 5 100x change

## Final hardening + submit
- [ ] Never 500; exact field names/sorts/codes; naive datetimes → aware UTC; `history`/`half_day` missing → `[]`/`false`; no-doc weekday ≠ ABSENT; no Python fetch-loops in analytics; concurrency re-test (parallel punch-in/out/PATCH)
- [ ] Update `README.md` (run + gaps); repo contains only `app/main.py`, `requirements.txt`, `REVIEW.md`, `DECISIONS.md`, `README.md` (+ PRD/ROADMAP/todo); no `.env`/secrets/Dockerfile/venv/dumps
- [ ] Clean-clone check: `pip install -r requirements.txt && uvicorn app.main:app --port 8000`, all indexes auto-created, public Git link submitted
