# Employee Attendance & Analytics API

FastAPI and MongoDB backend service for employee attendance tracking, manual regularization with audit trails, and aggregation-driven analytics.

## Tech Stack
- **Framework**: Python 3.11+, FastAPI, Uvicorn
- **Database**: MongoDB 6.0+ (Tested on MongoDB 8.3 / compatible with 7.0)
- **Driver**: PyMongo 4.7+
- **Validation**: Pydantic v2

## Setup & Running

1. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

2. **Configure environment variables** (or rely on defaults):
   ```bash
   export MONGO_URI="mongodb://localhost:27017"
   export MONGO_DB="attendance_db"
   ```

3. **Start the application**:
   ```bash
   uvicorn app.main:app --port 8000
   ```
   *Note: All required unique and compound database indexes are created idempotently on startup inside the application lifespan.*

4. **Verify health**:
   ```bash
   curl http://localhost:8000/health
   ```

## Implementation Highlights
- **Concurrency & Races**: Unique compound index on `(emp_code, date)` guarantees at most one punch-in per employee per date. Punch-out uses atomic `find_one_and_update` on `punch_out: None`.
- **Rules R1–R5**: Whole-second instant truncation, IST timezone conversion, overnight shift handling crossing midnight, strict 10-minute grace period floor math, and commercial round-half-up work hours.
- **Audit Trails**: Manual corrections (`PATCH /attendance/{emp_code}/{date}`) append immutable diffs to `history` protected by optimistic concurrency control.
- **Analytics Aggregations**: All analytics queries (`monthly`, `summary`, `leaderboard`, `trend`) execute natively inside MongoDB using `$lookup`, `$densify`, and `$setWindowFields` without in-memory Python looping.
- **Query Performance**: The `/admin/explain/{endpoint}` endpoint confirms all list and analytics endpoints use index scans (`IXSCAN`) without full collection scans (`COLLSCAN`).
