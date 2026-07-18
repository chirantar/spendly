# Implementation Plan: Login and Logout (Step 3)

## 1. Summary

This implements session-based authentication for Spendly: verifying
credentials on `POST /login`, storing the logged-in user's id in Flask's
`session`, and clearing that session on `GET /logout`. It builds directly
on Step 2 (registration) — same `users` table, same
`generate_password_hash`/`check_password_hash` pairing, same
validate → DB-helper → redirect-or-re-render pattern used in
`register()`. This is the first use of `flask.session` in the codebase;
no `app.secret_key`, session usage, or auth guard exists yet (confirmed
by exploration — grep for `secret_key`, `session`, `login_required`,
`before_request` all come back empty in `app.py`/`database/db.py`).

## 2. Source Spec

`.claude/specs/03-login_and_logout.md`

## 3. Implementation Order

1. `database/db.py` — add `get_user_by_email(email)` (no dependents, build
   first so it can be exercised independently).
2. `app.py` — set `app.secret_key` (required before `session` can be used
   anywhere).
3. `app.py` — replace the stub `login()` view with the `GET`/`POST`
   handler.
4. `app.py` — replace the stub `logout()` view.
5. `templates/login.html` — fix the hardcoded form action.

## 4. File-by-File Plan

### `database/db.py` — add `get_user_by_email(email)`

Add alongside `get_db`/`create_user`, following the same
open-connection → parameterized query → return pattern already used
there:
- `SELECT` the full user row `WHERE email = ?` (parameterized, per
  `CLAUDE.md`'s no-f-string-SQL rule).
- `fetchone()` and return it directly (a `sqlite3.Row` or `None` if no
  match) — let the caller (the route) decide what "no match" means, don't
  raise here.
- Close the connection before returning, mirroring `create_user`'s
  cleanup.

### `app.py` — `app.secret_key`

Set immediately after `app = Flask(__name__)`. Needed for
`flask.session` to sign its cookie; without it every `session[...]`
write raises at request time.

### `app.py` — replace stub `login()`

Add `methods=["GET", "POST"]` to `@app.route("/login")`.
- `GET`: unchanged — `render_template("login.html")`, no error.
- `POST`:
  1. Read `email`, `password` from `request.form`.
  2. Call `get_user_by_email(email)`.
  3. If no row, or `check_password_hash(row["password_hash"], password)`
     is `False`: re-render `login.html` with one generic
     `error="Invalid email or password."` — same message for both
     failure modes, per the spec's "don't leak which case occurred"
     rule.
  4. On success: `session["user_id"] = row["id"]`, then
     `redirect(url_for("profile"))`.
- New imports needed in `app.py`: `session` from `flask`;
  `check_password_hash` from `werkzeug.security`; `get_user_by_email`
  added to the existing `from database.db import ...` line.
- Keep the view to orchestration only (read form → look up → verify →
  respond), matching the "one responsibility" rule and the shape of the
  existing `register()` view.

### `app.py` — replace stub `logout()`

- `session.pop("user_id", None)` — safe no-op if the key isn't present,
  so visiting `/logout` while already logged out doesn't error.
- `redirect(url_for("landing"))`.
- No login-required check needed here (spec explicitly scopes a reusable
  `login_required` guard out of this step).

### `templates/login.html` — fix form action

Change the hardcoded `action="/login"` to
`action="{{ url_for('login') }}"`, the same fix already applied to
`register.html` in Step 2. No other markup changes — the existing
`{% if error %}` block already renders whatever `error` the route
passes in.

## 5. Edge Cases & Error Handling

- Non-existent email → generic error, no stack trace (handled by the
  `get_user_by_email` → `None` → generic-error branch above).
- Correct email, wrong password → same generic error (handled by
  `check_password_hash` returning `False`).
- `/logout` with no active session → `session.pop(..., None)` avoids a
  `KeyError`; still redirects to `/`.
- Missing `app.secret_key` would surface as a `RuntimeError` on first
  `session` write — setting it in step 3.2 of the implementation order
  prevents ever hitting that state.

## 6. Open Questions / Ambiguities

None — the spec fully pins down behavior for this step. (`GET /profile`
itself stays a stub per the spec; `POST /login` redirects there but
implementing the route is Step 4's responsibility.)

## Verification

1. Start the app (`python app.py`, port 5001).
2. `GET /login` still renders the form with no error banner (no
   regression).
3. Submit the seeded demo account (`demo@spendly.com` / `demo123`) →
   expect redirect to `/profile` and a session cookie set in the browser.
4. Submit a non-existent email → expect the login page to re-render with
   "Invalid email or password.", HTTP 200, no crash.
5. Submit `demo@spendly.com` with a wrong password → expect the same
   generic error message as step 4.
6. After a successful login, visit `/logout` → expect redirect to `/`
   and the session cookie's `user_id` cleared (confirm by checking that a
   protected action relying on `session.get("user_id")` would now see
   `None`).
7. Visit `/logout` directly with no prior login → expect redirect to `/`
   with no error.
8. View page source of the rendered login page and confirm the form's
   `action` resolves to `/login` via `url_for`, not a literal hardcoded
   string in the template source.
