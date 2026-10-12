# DECISIONS.md (about 150-200 words in total - plain language, your own words)

1. **Indexes.** Unique `emp_code` (dup-create race + lookups), unique `(emp_code,date)` (punch-in race), `(date desc,emp_code asc)` for list sort/paginate, `(date)` for month ranges, `(emp_code,punch_in)` for punch-out search, `(department)` + `(joined_on)` for headcount. Rejected a `status`-only index: only five values, so it filters almost nothing and the planner would still scan.

2. **Punch-in race.** Both requests compute the same IST date and try `insert_one` at once. The unique `(emp_code,date)` index lets only one win; the loser gets `DuplicateKeyError`, which I map to 409. I never do check-then-insert, so there is no gap to slip through.

3. **Ties.** I rank with `$rank` (1,2,2,4), then keep rows with `rank <= limit`. A tie at the cutoff all comes back, so the response can be longer than `limit`. Ordering is total desc, `emp_code` asc.

4. **Headcount.** I start from `employees` with `joined_on <= month-end` and left-join logs, so people with zero logs still count and future joiners do not. Averages divide summed `work_hours` by record count, not by averaging per-person averages.

5. **One thing you would change** if this had to serve 100x the data. I would shard `attendance_logs` by `emp_code`, pre-aggregate monthly rollups, and move trend reads to a read replica.
