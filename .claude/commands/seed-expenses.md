---
description: Seed random past expenses for an existing user
argument-hint: <user_id> <count> <months>
allowed-tools: Read, Bash(python3:*)
---

Arguments passed: $ARGUMENTS

Parse exactly three arguments from the above, in this order:
1. `user_id` — integer, the user to add expenses for
2. `count` — integer, total number of expenses to insert
3. `months` — integer, span of past months over which expenses
   should be randomly distributed

If any of the three arguments is missing or not a valid integer,
**do not run anything** — stop and print a usage error explaining
that all three (`user_id`, `count`, `months`) are required, e.g.
`/seed-expenses 2 30 6`.

Read database/db.py to understand the `expenses` table schema,
the `CATEGORIES` list, and the `get_db()` helper.

Then write and run a Python script using Bash that:

1. Looks up `user_id` in the `users` table. If no such user
   exists, print an error naming the missing user_id and stop
   without inserting anything.

2. Distributes `count` expenses as evenly as possible across all
   categories in `CATEGORIES` (any remainder from uneven division
   spread randomly across categories, not all dumped on one).

3. For each expense, picks a random date within the last `months`
   months from today (inclusive of today), formatted `YYYY-MM-DD`.

4. Picks a random amount (in rupees, 2 decimal places) from a
   range specific to that expense's category, so amounts stay
   realistic relative to what the category represents. Use these
   ranges (INR):
   - Food: 100 – 1200
   - Transport: 50 – 5000 (covers everything from an auto ride to
     a flight)
   - Bills: 300 – 5000
   - Health: 150 – 3000
   - Entertainment: 100 – 1500
   - Shopping: 200 – 6000
   - Other: 50 – 1000

5. Fills `description` with a short, plausible label for that
   category (e.g. Food → "Groceries" / "Dinner out" / "Lunch",
   Transport → "Cab ride" / "Flight ticket" / "Fuel", etc.) —
   vary it across rows rather than repeating the same string.

6. Inserts every expense using the same `get_db()` pattern found
   in db.py, with `user_id` set to the given user, and commits
   once at the end.

7. Prints a confirmation summary:
   - total expenses inserted
   - the user_id and months span used
   - a per-category count breakdown
   - the min/max amount actually inserted per category
