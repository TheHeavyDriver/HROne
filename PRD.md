# PRD — HROne Employee Attendance & Analytics API

Version: 2.0.0 (Contract v2.0) | Stack: Python 3.11+, FastAPI, PyMongo/Motor, MongoDB 6.0+ (grader 7.0) | Source of truth: `openapi.yaml` (contract + R1–R10) + `DATA_MODEL.md` (storage)

## 1. Problem Statement

HR teams punch employees in/out daily across day and overnight shifts, correct biometric errors with an audit trail, and ask analytics questions: who is late, which department is slipping, what is the daily trend?

Current state: a half-finished single-file starter service exists. Happy path works but it has 10+ defects (timezone handling, rounding, pagination, validation, concurrency, performance). Six endpoints are missing entirely (punch-out, regularization, 3 analytics pipelines + trend, explain). No indexes exist except `_id`. Seeded data uses strict BSON types that the starter mishandles. The hidden grader dataset (100k attendance docs, ties, weekend records, no-log employees, mid-month joiners, overnight shifts, missing weekdays) will fail any plausible-looking but imprecise implementation.

This PRD defines a correct, race-safe, indexed, aggregation-native implementation that passes automated contract tests and a live walkthrough.

## 2. Solution

Build a single-file FastAPI service exposing 12 operations:

- System: `GET /health` (ping-gated readiness).
- Employees: `POST /employees`, `GET /employees`.
- Attendance transactional: `POST /attendance/punch-in`, `POST /attendance/punch-out`, `GET /attendance`, `PATCH /attendance/{emp_code}/{date}`.
- Analytics (MongoDB aggregation only, no Python loops): `GET /analytics/employees/{emp_code}/monthly`, `GET /analytics/departments/summary`, `GET /analytics/leaderboard/late`, `GET /analytics/departments/{department}/trend`.
- Proof: `GET /admin/explain/{endpoint}` returning raw `executionStats` explain.

Key behaviors: IST-based date derivation (R1), grace/overtime/half-day math (R2–R5), presence/working-day/headcount semantics (R6/R7/R9), half-up rounding (R4/R8), competition ranking with post-rank limit (leaderboard), gap-filled daily series with 7-day moving average (trend), atomic concurrency (unique indexes + atomic updates), startup index creation, env-first config (`MONGO_URI`, `MONGO_DB`), strict status codes (404/409/422, never 500).

## 3. User Stories

### System / Operations
1. As a platform operator, I want `GET /health` to return 200 only when MongoDB pings, so orchestrators don't route to a dead app.
2. As a platform operator, I want the app to boot via `uvicorn app.main:app --port 8000` and be ready in <20s, so the grader can test deterministically.
3. As a platform operator, I want config from env vars overriding local `.env`, so grader-injected `MONGO_URI`/`MONGO_DB` always win.
4. As a platform operator, I want all indexes created idempotently at startup, so a pre-seeded 100k-doc DB is fast with no manual steps.

### Employees
5. As an HR admin, I want to create an employee with my own `emp_code` (format `EMP` + 4–6 digits), so codes match customer HR records.
6. As an HR admin, I want duplicate `emp_code` creation to return 409 (exactly one 201 under concurrent duplicates), so identity stays unique.
7. As an HR admin, I want validation (email pattern, `HH:MM` shifts, `shift_start != shift_end`, `YYYY-MM-DD joined_on`) to return 422, so bad data is rejected early.
8. As an HR admin, I want server-set `created_at` in epoch ms never accepted from clients, so creation time is trustworthy.
9. As an HR manager, I want to list employees sorted by `emp_code` asc with `department` exact filter and pagination (`page>=1`, `page_size 1–100 default 20`, `total` reflects filters), so I can browse large orgs.
10. As an API consumer, I want `_id` never exposed and `id` never invented in responses, so contract field names match exactly.

### Punch-in (R1, R2)
11. As an employee, I want to punch in with `emp_code` + optional `punched_at` (epoch ms) + `status` (PRESENT/WFH/ON_DUTY, default PRESENT), so on-site, home, and duty work are distinguished.
12. As an employee, I want omitted `punched_at` to mean now, so kiosks without clocks still work.
13. As a payroll analyst, I want attendance `date` derived as IST calendar date of truncated-to-second `punch_in`, with overnight rule (if IST time < `shift_end` when `shift_end <= shift_start`, date = previous day), so night shifts attribute correctly.
14. As a payroll analyst, I want `late_minutes` = floor minutes since `shift_start` only if punch-in > 10:00 after start (09:40:00→0, 09:40:01→10, 10:15:59→45), so grace is enforced precisely.
15. As an employee, I want second punch-in for same (`emp_code`, `date`) to return 409, with exactly one 201 under simultaneous duplicate requests, so double-taps don't duplicate days.
16. As an API consumer, I want unknown employee → 404, ABSENT/LEAVE on punch-in → 422, seconds-like/float/string timestamps → 422, so errors are distinguishable.
17. As a downstream consumer, I want new records to initialize `punch_out=null`, `work_hours=null`, `overtime_minutes=0`, `half_day=false`, `history=[]`, so open days are explicit.

### Punch-out (R3–R5)
18. As an employee, I want to punch out closing my most recent record with `punch_in <= punched_at` (working across midnight), so overnight shifts close correctly.
19. As a payroll analyst, I want `work_hours = (punch_out − punch_in)/3600` rounded 2-dec half-up (not banker's/float round), so pay math is exact.
20. As a payroll analyst, I want `overtime_minutes` = floor minutes from `shift_end` to `punch_out` only if ≥30 else 0, so short stays don't count.
21. As a payroll analyst, I want `half_day = rounded work_hours < 4.50`, so short days count 0.5 present.
22. As an employee, I want punch-out with `punched_at <= punch_in` or >24h after `punch_in` → 422, unknown/no-open record → 404, already-closed (including concurrent double punch-out → single 200) → 409.

### List attendance
23. As a manager, I want to filter attendance by `emp_code`/`date_from`/`date_to` (inclusive)/`status` sorted `date` desc then `emp_code` asc with pagination, so I can audit without loading 100k docs into memory.
24. As an API consumer, I want `date_from > date_to` → 422 and `total` to reflect filters, so misuse is signaled.

### Regularization / PATCH (audit trail)
25. As an HR admin, I want to correct one day's record by (`emp_code`, `date`) changing only `status`/`punch_in`/`punch_out` (omitted = unchanged), so errors are fixable.
26. As an auditor, I want derived fields recomputed from final times after every correction and `history` appended with exactly one `{at, by=regularized_by, reason, changes-only-changed-fields}`, never overwritten, so corrections are traceable.
27. As an auditor, I want `changes` epoch-ms values for punch times and native types otherwise, with punch-in/out never writing history, so trail semantics are clean.
28. As an HR admin, I want rules enforced: ABSENT/LEAVE clears punches (supplying punch with them → 422), presence requires `punch_in` (stored or supplied), `punch_in` stays on record's date (R1), `punch_out` after `punch_in` within 24h, no-op change → 422, missing employee/record → 404.
29. As an HR admin, I want `reason` (5–200 chars) + `regularized_by` (1–50) required, so every edit is justified and attributed.
30. As a concurrent editor, I want simultaneous PATCHes to never silently lose history entries (409 for loser acceptable), so audits don't drop.

### Employee monthly analytics (aggregation)
31. As a manager, I want per-employee monthly summary (`working_days` Mon–Fri from `joined_on` per R7, `present_days` Mon–Fri only with half=0.5, `leave_days` = LEAVE count, `late_count` = `late_minutes>0`, sums, `attendance_pct = present/working*100` 2-dec half-up or null if 0), reading stored derived values, so numbers match seeded truth.
32. As a manager, I want weekend records to contribute to late/overtime totals but not `present_days`, so weekends don't inflate attendance %.
33. As an API consumer, I want unknown employee → 404, bad `month` → 422.

### Department summary (cross-collection aggregation)
34. As a director, I want one row per department sorted asc with `headcount` per R9 (joined ≤ month-end, including zero-log people), `present_days` summed, `avg_work_hours` = mean over present+non-null `work_hours` records (average over records, R8), `late_count`/`total_late_minutes`, `leave_count`, `on_duty_count`, so slipping departments are visible.
35. As a director, I want zero-headcount departments omitted and optional exact `department` filter, so empty groups don't pollute.
36. As an engineer, I want this done as a cross-collection aggregation (department lives on employees), so it scales.

### Late leaderboard (window-function aggregation)
37. As a manager, I want late ranking by total `late_minutes` (>0 only) highest-first, ties share rank with skip (1,2,2,4), `limit` applied after ranking (response may exceed `limit` on ties), rows ordered total desc then `emp_code` asc, so ties are fair.
38. As a manager, I want optional department-scoped ranking (rank inside department only) with `name`/`department` joined from employees (orphan logs ignored), so I can focus interventions.
39. As an engineer, I want explicit instruction to not use array-index-as-rank, so cutoff ties are handled.

### Department trend (gap-filling aggregation)
40. As a director, I want exactly one row per calendar day `from→to` inclusive (weekends/no-record days included via densify-like stage, no Python loop) with `is_working_day`, `headcount` (joined ≤ that day), `present_count` (half=0.5, any day), `late_count`, `attendance_rate = present/headcount` 4-dec half-up only on working days with headcount>0 else null (no-record working day = 0.0), `moving_avg_7d` = mean of non-null rates in current + up to 6 prior in-range rows (shorter first window, null if none), so trends and 7-day smoothing are correct.
41. As an API consumer, I want unknown department → 404, `to < from` or range >92 days → 422.

### Explain / proof
42. As an evaluator, I want `GET /admin/explain/{endpoint}` for `attendance_list|employee_monthly|department_summary|late_leaderboard|department_trend` to run `explain(executionStats)` on the real query/pipeline (same filters/sort/pagination) and return raw output `{endpoint, collection, explain}`, so index use is provable.
43. As an evaluator, I want plans to show `IXSCAN` and no `COLLSCAN` on 100k docs, with missing required params → 422.

### Non-functional
44. As an evaluator, I want correct codes always (500 = failed test) and analytics via pipelines not fetch-and-loop, so scale holds.
45. As a candidate, I want `REVIEW.md` listing each starter defect (where/what/how-noticed/fix + non-defects) and `DECISIONS.md` (indexes, punch-in race walkthrough, tie cutoff, zero-log headcount, 100x change, 150–200 words), so review + decisions are graded.
46. As a candidate, I want `README.md` with run instructions and submission containing only app code + requirements + REVIEW/DECISIONS/README (no `.env`/secrets/Dockerfile/venv/dumps), so submission is clean.

## 4. Implementation Decisions

- **Time/rule engine (deep module, isolated + unit-tested):** single IST-aware helper set for truncate-to-second, attendance-date derivation incl. overnight shift, late (R2), overtime (R3), work-hours half-up (R4, Decimal-based), half-day (R5), working-day calendar (R7, Mon–Fri from `joined_on`), rate rounding (R8, 2-dec and 4-dec half-up). All transactional + analytics paths call this; analytics read stored values but corrections recompute via it. Naive PyMongo datetimes normalized to aware UTC at boundary; epoch-ms ↔ BSON conversion centralized with strict range check (reject <100000000000, floats, strings → 422).
- **Validation layer:** schema validation for `emp_code` pattern, email, `HH:MM`, `YYYY-MM-DD`/`YYYY-MM`, status enums (punch-in accepts only presence trio), derived-field stripping/ignore on input, pagination bounds (R10), date-range and month-range checks. Uniform `{"detail":…}` errors.
- **Persistence & indexing:** unique index on employee business code; unique compound on attendance (`emp_code`, `date`); supporting indexes for list filters/sort and each analytics pipeline's match/sort/group keys (attendance date range, employee+month, department join, late totals). One index considered and rejected will be documented (e.g., low-cardinality status-only index). All created idempotently at startup; explain endpoint reuses the exact query/pipeline builder.
- **Concurrency control:** no check-then-insert; rely on unique indexes + duplicate-key → 409 mapping. Punch-out uses atomic find-and-set on open record (single 200 winner). PATCH uses atomic append to history with recompute (optimistic retry / 409 on conflict) so concurrent edits don't drop entries.
- **Aggregation pipelines:** employee-monthly (match month + weekday logic + headcount-aware denominator computed in Python date math + grouped sums in DB); department summary (`$lookup` employees→logs by business code, headcount includes zero-log, average over records); leaderboard (`$group` totals → `$sort` → `$setWindowFields rank dense/competition` → post-rank `$match rank<=limit`); trend (`$densify`/calendar spine + `$lookup` + `$setWindowFields` 7-day moving avg, in-range window only). No Python accumulation loops.
- **API contracts:** field names, sort orders, and status codes frozen per contract section above; `_id` never serialized; missing history/`half_day` treated as `[]`/`false`; weekday-with-no-document distinct from ABSENT.
- **Startup/config:** env-wins-over-dotenv, ping-gated health, single-file deployable service.

## 5. Testing Decisions

- **What makes a good test:** asserts external HTTP behavior (status + body fields) on seeded + synthetic data, not internals; covers one rule or race in isolation with a minimal failing example (e.g., 09:40:01 → late 10, overnight punch attribution, tie at limit, zero-log headcount).
- **Modules to test:** rule engine (all R1–R9 edge values, half-up vs banker, truncation, overnight), transactional endpoints (validation matrix, 404/409/422, double punch-in/out races with parallel clients), PATCH history diffing (only-changed keys, epoch-ms nesting), each analytics pipeline against hand-computed fixtures including ties, weekends, mid-month joiner, missing weekday, orphan logs, plus explain assertions (IXSCAN present, COLLSCAN absent) and 100k-doc latency smoke.
- **Prior art:** sample seed + nine annotated sample docs as shape fixtures only; candidates must synthesize adversarial datasets (ties at cutoff, no-log employee, joiner mid-month, overnight cross-midnight, absent vs missing weekday) since sample pass proves nothing.

## 6. Out of Scope

- AuthN/Z (`regularized_by` is free text), UI, holiday calendars (Mon–Fri only), employee update/delete or `emp_code` mutation, creation of ABSENT/LEAVE via punch endpoints, external service calls, Docker packaging, secrets/`.env` in repo, holiday-aware working days, real-time streaming.

## 7. Further Notes

- Analytics must read stored derived values, not recompute — grader seeds authoritative numbers.
- Truncation before storage matters: `09:40:00.900` stores/returns `09:40:00`.
- Leaderboard limit-after-rank and trend in-range moving window are the two most common correctness traps; array-index rank and Python gap-filling are explicitly disallowed.
- Evaluation weights: Review 10%, CRUD/rules/concurrency 30%, Analytics 30%, Mongo usage/speed 15%, Decisions/quality 15%. Partial submissions welcome but must note gaps in README.

## 8. Roadmap (summary — see ROADMAP.md)

- M0 Setup & harness → M1 Review & fix starter (Part A) → M2 Punch-out + Regularize (Part B-core) → M3 Analytics pipelines (hard endpoints) → M4 Explain + indexes + 100k perf → M5 Hardening, docs, submission (Parts C + checklist). Each milestone has exit criteria; critical path is rule engine → atomic writes → pipelines → indexes.
