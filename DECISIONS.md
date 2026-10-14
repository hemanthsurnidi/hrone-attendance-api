# DECISIONS.md

1. **Indexes.**
   Created `employees: {emp_code: 1}` (unique) for identity and `{department: 1}` for filtering. In `attendance_logs`, created `{emp_code: 1, date: 1}` (unique) to enforce one record per day and prevent race conditions; `{date: -1, emp_code: 1}` for index-backed sorted pagination; `{emp_code: 1, punch_in: -1}` for fast punch-out lookups; and `{date: 1, status: 1}` for analytics queries.
   *Rejected*: An index on `history.at`, as history is an append-only audit log and never queried directly.

2. **Punch-in race.**
   Both requests reach `POST /attendance/punch-in` simultaneously. Both calculate the same attendance date and call `insert_one`. MongoDB's storage engine enforces the unique compound index `(emp_code, date)`. The first write succeeds and returns 201. The second write triggers a database `DuplicateKeyError`, which our API catches and translates into an HTTP 409 Conflict.

3. **Ties.**
   The aggregation uses `$rank` in `$setWindowFields` for standard competition ranking (1, 2, 2, 4). Filtering `rank <= limit` occurs after ranking. When employees tie at the cutoff, all tied employees share that rank and are returned (yielding more than `limit` items). Ties are sorted by `emp_code` ascending.

4. **Headcount.**
   The pipeline originates on the `employees` collection (matching `joined_on <= end_of_month`) and performs a `$lookup` to `attendance_logs`. Headcount is computed by grouping employee documents before unwinding logs, guaranteeing employees with zero logs are fully counted.

5. **One thing you would change if this had to serve 100x the data.**
   Pre-aggregate monthly attendance metrics asynchronously on punch-out/regularization, and shard `attendance_logs` on a hashed key `(emp_code, date)` to distribute write and query loads across database partitions.
