# Implementation Plan: Registration (Step 2)

## Context

Spendly's `GET /register` route already renders a working form (`templates/register.html`)
but has no handler for the submission — `POST /register` doesn't exist yet, so
submitting the form currently 405s. Step 1 built out the database layer
(`get_db`, `init_db`, `seed_db`, and the `users`/`expenses` tables with a
`UNIQUE` email constraint on `users`). This step wires the form to that
database layer: accept the POST, validate input server-side, hash the
password, insert the user, and send them to `/login` to sign in. It also
establishes the validate → DB helper → redirect/re-render-with-error pattern
that later auth steps (login, logout, profile) will follow.

Full requirements are in `.claude/specs/02-registration.md` — this plan
translates that spec into concrete edits.

## Files to modify

### 1. `database/db.py` — add `create_user(name, email, password)`

Add a new function alongside `get_db`/`init_db`/`seed_db` (same file, no new
module — per `CLAUDE.md`, DB logic never lives in routes):

- Open a connection with `get_db()`.
- Hash the password with `generate_password_hash` (already imported at the
  top of this file for `seed_db`).
- `INSERT INTO users (name, email, password_hash) VALUES (?, ?, ?)` using the
  existing parameterized-query style seen in `seed_db()`.
- `commit()` and `close()` the connection, mirroring `seed_db()`'s pattern.
- Do **not** catch `sqlite3.IntegrityError` here — let it propagate so the
  route can distinguish "duplicate email" from other failures and choose the
  user-facing message.

### 2. `app.py` — replace the stub `register()` view

Current stub (lines ~21-23):
```python
@app.route("/register")
def register():
    return render_template("register.html")
```

Replace with a view that:
- Adds `methods=["GET", "POST"]` to the route decorator.
- On `GET`: renders `register.html` with no error (existing behavior,
  unchanged).
- On `POST`:
  1. Read `name`, `email`, `password` from `request.form`, `.strip()` name
     and email.
  2. Validate server-side (HTML5 `required`/`type=email` isn't trusted —
     forms can be submitted without JS/browser validation):
     - `name` non-empty
     - `email` non-empty
     - `password` length >= 8 (matches the field's placeholder "Min. 8
       characters")
     - On any failure: `render_template("register.html", error=<message>)`,
       do not touch the database.
  3. If validation passes, call `create_user(name, email, password)` inside a
     `try`/`except sqlite3.IntegrityError`:
     - On `IntegrityError` (duplicate email): re-render `register.html` with
       `error="An account with that email already exists."`
     - On success: `redirect(url_for("login"))`.
- Needs new imports at the top of `app.py`: `request`, `redirect`,
  `url_for` from `flask`; `sqlite3` for catching `IntegrityError`; and
  `create_user` from `database.db` (alongside the existing `get_db, init_db,
  seed_db` import).

Keep the view itself doing only orchestration (read form → validate → call
DB helper → respond) per the "one responsibility" rule in `CLAUDE.md` — all
persistence stays in `create_user`.

### 3. `templates/register.html` — fix hardcoded form action

Change:
```html
<form method="POST" action="/register">
```
to:
```html
<form method="POST" action="{{ url_for('register') }}">
```
No other markup changes — the `{% if error %}` block already renders
whatever `error` the route passes in.

## Out of scope (explicitly, per spec)

- No session/login-state handling — `POST /register` does not log the user
  in, it only creates the account and redirects to `/login`.
- No changes to `GET /login`, `/logout`, `/profile` (Steps 3/4).
- No schema changes — `users` table already has everything needed.
- `templates/login.html`'s own hardcoded `action="/login"` is a pre-existing
  issue but out of scope for this step (not mentioned in the spec's Files to
  Change) — leave it alone.

## Verification

1. Start the app (`python app.py`, port 5001) and confirm `GET /register`
   still renders the form with no error banner (no regression).
2. Submit the form with a new name/email/valid password → expect a redirect
   to `/login`; confirm via `sqlite3 spendly.db "SELECT * FROM users"` (or
   equivalent) that a new row exists with a bcrypt/werkzeug-style hash in
   `password_hash`, not the plaintext password.
3. Submit again with the same email → expect the register page to re-render
   with an "already exists" error, HTTP 200, and no new/duplicate row in
   `users`.
4. Submit with a password under 8 characters → expect re-render with a
   validation error and no row inserted.
5. Submit with empty name or email → expect re-render with a validation
   error and no row inserted.
6. View page source on the rendered register page and confirm the form's
   `action` attribute is `/register` (resolved via `url_for`), not a literal
   hardcoded string in the template source.
