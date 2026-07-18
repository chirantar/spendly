# Spec: Login and Logout

## 1. Overview

Implement session-based authentication for Spendly. `GET /login` already
renders `login.html` with a form (email, password) that POSTs to `/login`.
This step adds the `POST /login` handler: verify credentials against the
`users` table, and on success store the user's identity in Flask's session.
It also implements `GET /logout` to clear that session. This establishes
the "logged-in user" concept that later steps (`/profile`, `/expenses/*`)
will depend on to scope data per-user.

## 2. Depends on

- Step 1 — Database setup (`users` table with `email`, `password_hash`).
- Step 2 — Registration (`create_user`, password hashing via
  `generate_password_hash`).

## 3. Routes

- `POST /login` — new. Public (no login required).
  - Reads `email`, `password` from the submitted form.
  - Looks up the user by email; verifies the password with
    `check_password_hash`.
  - On failure (no such user, or wrong password): re-renders `login.html`
    with a generic `error` message (don't reveal whether the email exists).
  - On success: stores `user_id` in `session`, redirects to `GET /profile`.
- `GET /login` — existing, unchanged.
- `GET /logout` — implement (was a stub). Login required. Clears the
  session and redirects to `GET /landing` (`/`).

Note: `GET /profile` stays a stub in this step per the "don't implement a
stub route unless the active task targets it" rule — `POST /login`
redirects there, but the route itself is Step 4's responsibility. Route
protection (a `login_required` guard reusable by `/profile` and
`/expenses/*`) is also out of scope here; this step only needs `/logout`
to check `session.get("user_id")` before clearing it.

## 4. Database Changes

No schema changes. `users` table already has everything needed
(`email`, `password_hash`).

New function in `database/db.py`:

- `get_user_by_email(email)` — parameterized `SELECT` returning the user
  row (or `None`) for the given email. Used by the login route to fetch
  `password_hash` for verification.

## 5. Templates

- `templates/login.html` — fix the hardcoded form `action="/login"` to
  `action="{{ url_for('login') }}"`, matching the fix already made to
  `register.html` in Step 2. No other markup changes — the error banner
  already matches what the route needs.

## 6. Files to Change

- `app.py` — replace the stub `login()` view with one handling both `GET`
  (render form) and `POST` (verify credentials, set session, redirect or
  re-render with error); replace the stub `logout()` view with one that
  clears the session and redirects to `landing`.
- `database/db.py` — add `get_user_by_email(email)`.
- `templates/login.html` — fix hardcoded form action to use `url_for()`.

## 7. Files to Create

None.

## 8. Dependencies

No new dependencies. Uses `werkzeug.security.check_password_hash`
(companion to `generate_password_hash`, same package) and Flask's built-in
`session`.

## 9. Rules for Implementation

- No SQLAlchemy or other ORMs
- Parameterized queries only — never string-format SQL
- Passwords verified with werkzeug's `check_password_hash` — never compare
  plaintext
- Styles use CSS variables — never hardcode hex values
- All templates extend `base.html`
- `@app.route("/login")` must accept `methods=["GET", "POST"]`
- Flask's `app.secret_key` must be set (required for `session` to work) —
  add it in `app.py` if not already present
- Login error message must be generic (e.g. "Invalid email or password")
  and identical for "no such user" and "wrong password" — don't leak which
  case occurred
- `/logout` must not error if visited with no active session — just
  redirect to landing either way

## 10. Definition of Done

- [ ] Visiting `/login` and submitting the seeded demo account
      (`demo@spendly.com` / `demo123`) redirects to `/profile` and sets a
      session cookie
- [ ] Submitting a non-existent email re-renders the login page with a
      generic error and does not crash the app
- [ ] Submitting a correct email with the wrong password re-renders the
      login page with the same generic error
- [ ] After logging in, visiting `/logout` clears the session and redirects
      to `/`
- [ ] Visiting `/logout` while not logged in does not crash the app and
      still redirects to `/`
- [ ] The login form's `action` uses `url_for('login')` instead of a
      hardcoded path
- [ ] `GET /login` still renders the form correctly (no regression)
