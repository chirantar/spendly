# Spec: Registration

## 1. Overview

Implement account creation for Spendly. The `GET /register` route already renders
`register.html` with a form (name, email, password) that POSTs to `/register`.
This step adds the `POST /register` handler: validate the submitted data, hash
the password, insert a new row into `users`, and redirect the user into the
login flow. This is the first step that writes to the database from a route,
and it establishes the pattern (validate → DB call in `database/db.py` →
redirect or re-render with error) that later auth/expense steps will reuse.

## 2. Depends on

Step 1 — Database setup (`database/db.py` implements `get_db()`, `init_db()`,
`seed_db()`, and the `users` table with a `UNIQUE` email constraint).

## 3. Routes

- `POST /register` — new. Public (no login required).
  - Reads `name`, `email`, `password` from the submitted form.
  - On validation failure or duplicate email: re-renders `register.html` with
    an `error` message (template already supports this via `{% if error %}`).
  - On success: creates the user, then redirects to `GET /login`.
- `GET /register` — existing, unchanged.

Note: session/login (setting a logged-in user, `GET /logout`, `GET /profile`)
is out of scope — those are Steps 3 and 4. Registration only creates the
account; the user still has to log in afterward.

## 4. Database Changes

No schema changes. `users` table (from Step 1) already has the columns this
feature needs: `name`, `email` (UNIQUE), `password_hash`.

New function in `database/db.py`:

- `create_user(name, email, password)` — hashes the password with
  `generate_password_hash` and inserts a row into `users` via a parameterized
  query. Lets the `sqlite3.IntegrityError` from the `UNIQUE` email constraint
  propagate; the route catches it to show a "email already registered" error.

## 5. Templates

- `templates/register.html` — modify only the `<form>` tag's `action`
  attribute: it's currently hardcoded as `action="/register"`, which violates
  the project's "always use `url_for()`" rule. Change to
  `action="{{ url_for('register') }}"`. No other markup changes — the error
  banner and field structure already match what the route needs.

## 6. Files to Change

- `app.py` — replace the stub `register()` view with one that handles both
  `GET` (render form) and `POST` (validate, call `create_user`, redirect or
  re-render with error).
- `database/db.py` — add `create_user(name, email, password)`.
- `templates/register.html` — fix hardcoded form action to use `url_for()`.

## 7. Files to Create

None.

## 8. Dependencies

No new dependencies. Uses `werkzeug.security.generate_password_hash`, already
imported in `database/db.py`.

## 9. Rules for Implementation

- No SQLAlchemy or other ORMs
- Parameterized queries only — never string-format SQL
- Passwords hashed with werkzeug's `generate_password_hash`
- Styles use CSS variables — never hardcode hex values
- All templates extend `base.html`
- `@app.route("/register")` must accept `methods=["GET", "POST"]`
- Basic server-side validation required even though the form has HTML5
  `required` attributes: name non-empty, email non-empty, password at least
  8 characters (matches the placeholder text "Min. 8 characters")
- Duplicate email must show a user-facing error, not a 500 — catch the
  `IntegrityError` in the route

## 10. Definition of Done

- [ ] Visiting `/register` and submitting valid name/email/password creates a
      new row in the `users` table with a hashed (not plaintext) password
- [ ] After successful registration, the browser is redirected to `/login`
- [ ] Submitting an email that already exists (e.g. `demo@spendly.com`)
      re-renders the register page with an error message and does not crash
      the app or create a duplicate row
- [ ] Submitting a password under 8 characters re-renders the register page
      with an error message and does not create a user
- [ ] Submitting with an empty name or email re-renders the register page
      with an error message and does not create a user
- [ ] The register form's `action` uses `url_for('register')` instead of a
      hardcoded path
- [ ] `GET /register` still renders the form correctly (no regression)
