# Plan: Database Setup (`database/db.py`)

## Context

`database/db.py` is currently a stub (comment-only) per `CLAUDE.md`. This is Step 1 of the Spendly build-out, per `.claude/specs/01-database-setup.md`. It establishes the SQLite data layer (`users` and `expenses` tables) that every later feature (auth, profile, expense CRUD) depends on. `app.py` currently has no DB wiring — it just renders templates and stub strings for unimplemented routes. This step only touches the data layer and app startup; no routes are implemented or changed.

## Approach

### 1. `database/db.py` — implement three functions

```python
import sqlite3
from datetime import date, timedelta
from werkzeug.security import generate_password_hash

DB_PATH = "spendly.db"

def get_db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    return conn
```

- **`init_db()`**: opens a connection via `get_db()`, runs two `CREATE TABLE IF NOT EXISTS` statements matching the spec schema exactly (users: id/name/email/password_hash/created_at; expenses: id/user_id/amount/category/date/description/created_at with `FOREIGN KEY (user_id) REFERENCES users(id)`), commits, closes.
- **`seed_db()`**: opens a connection, `SELECT COUNT(*) FROM users` — if > 0, close and return early. Otherwise:
  - Insert demo user (`Demo User`, `demo@spendly.com`, `generate_password_hash("demo123")`) via parameterized `INSERT`, capture `cursor.lastrowid` as `user_id`.
  - Insert 8 sample expenses tied to `user_id`, covering all 7 categories from the spec (`Food, Transport, Bills, Health, Entertainment, Shopping, Other`) with one extra doubled up, dates spread across the current month (computed via `datetime.date.today()`, formatted `YYYY-MM-DD` — no hardcoded dates so seed data stays "current" whenever it runs).
  - Commit, close.
- All SQL uses `?` placeholders — no f-strings/string formatting in queries, per project rules.

### 2. `app.py` — wire up DB startup

- Add `from database.db import get_db, init_db, seed_db` (import `get_db` even though unused directly here, per spec's explicit instruction — future steps will use it).
- After `app = Flask(__name__)`, add:
  ```python
  with app.app_context():
      init_db()
      seed_db()
  ```
- No route logic changes — placeholder routes stay exactly as-is (per `CLAUDE.md`: don't implement stub routes outside their designated step).

### 3. Verification

- Delete any stale `spendly.db` if present, then run `python app.py` (or `./run.sh` per this project's existing convention) and confirm no errors on startup.
- Inspect the resulting DB: `sqlite3 spendly.db ".tables"` and `.schema users` / `.schema expenses` to confirm schema matches spec.
- Confirm seed data: `sqlite3 spendly.db "SELECT * FROM users;"` and `"SELECT category, date FROM expenses;"` — expect 1 user, 8 expenses spanning all 7 categories.
- Restart the app a second time and re-check row counts — must stay at 1 user / 8 expenses (no duplication).
- Manually test constraints via a Python shell using `get_db()`: inserting a duplicate email should raise `sqlite3.IntegrityError`; inserting an expense with a bogus `user_id` should also raise `sqlite3.IntegrityError` (confirms `PRAGMA foreign_keys = ON` is active).
- Per `CLAUDE.md` subagent policy, dispatch a subagent after implementation to run/verify these checks.

## Files touched

- `database/db.py` — implement `get_db()`, `init_db()`, `seed_db()`
- `app.py` — add imports + `app_context()` startup block

No new files, no new dependencies (uses stdlib `sqlite3` + already-installed `werkzeug.security`).
