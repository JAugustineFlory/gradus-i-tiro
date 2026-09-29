# 08 — Guardrails and debrief

**Goal:** a pre-commit hook that checks linting, types, and tests on
both sides; proof that your repo works on a fresh clone; an optional
solo drill; and an After Action Review of the whole project.

---

## Step 1 — A type-check script

**Connect the dots.** Vitest runs your TypeScript, but it doesn't
*type-check* it — it strips the types and runs what's left. A type
error (passing a number where a string belongs) can hide behind green
tests. TypeScript's compiler, `tsc`, catches those.

In `frontend/package.json`, add to `"scripts"`:

```json
"typecheck": "tsc -b"
```

Run it from `frontend/`:

```bash
npm run typecheck
```

✅ No output means no type errors.

---

## Step 2 — The full pre-commit hook

Replace `.husky/pre-commit` at the repo root:

```sh
echo "Backend: lint"
(cd backend && uv run ruff check . && uv run ruff format --check .)

echo "Frontend: lint and types"
(cd frontend && npm run lint && npm run typecheck)

echo "Backend: tests"
(cd backend && uv run pytest -q)

echo "Frontend: tests"
(cd frontend && npm run test:run)
```

- **Fast checks first.** Linting takes a second; tests take longer. If
  there's a lint error, you find out immediately.
- **`&&`** means "run the next command only if the previous one
  succeeded."
- **`ruff format --check`** doesn't change files — it fails if any file
  *would* be reformatted. (Format-on-save should keep you clean.)

### Keep ESLint out of the coverage report

`npm run coverage` writes an HTML report into `frontend/coverage/`,
including some JavaScript files. ESLint would lint those and fail on
code you didn't write.

Open `frontend/eslint.config.js` and find `'dist'` (use `Ctrl+F`). It
sits in a list of ignored folders. Add `'coverage'` beside it:

```js
globalIgnores(['dist', 'coverage']),
```

(Older versions of the template write it as
`{ ignores: ['dist', 'coverage'] }` — same idea.)

### Run the hook without committing

Add a script to the **root** `package.json`, replacing the placeholder
`"test"` script that `npm init` created:

```json
"scripts": {
  "prepare": "husky",
  "check": "sh .husky/pre-commit"
}
```

Now, from the repo root:

```bash
npm run check
```

✅ All four stages print and finish without errors. Fix anything that
fails before continuing.

---

## Step 3 — Prove it works on a fresh clone

Your repo is only useful to someone else if it works on *their* machine.
Clone it somewhere new and set it up from zero — exactly what a
classmate would do.

From a folder **outside** your repo (e.g. your `dev` folder):

```bash
git clone <your-repo-url> tiro-fresh
cd tiro-fresh
npm install
```

✅ `npm install` at the root installs Husky and its hooks (via
`prepare`).

```bash
cd backend
uv sync
uv run alembic upgrade head
uv run pytest -q
```

- **`uv sync`** installs everything listed in `uv.lock` into a new
  `.venv`. It's how you set up an existing uv project.

✅ `15 passed`.

```bash
cd ../frontend
npm install
npm run test:run
```

✅ `8 passed`.

If anything fails here but works in your original folder, a file is
missing from Git. Check `git status` in the original, and check
`.gitignore` isn't hiding something it shouldn't.

Delete `tiro-fresh` when you're done.

---

## Step 4 — Document how to run it

Add a section at the **bottom** of the repo's `README.md` so anyone can
run your finished app:

````markdown
## Running the finished app

Requirements: Node.js LTS, uv.

```bash
npm install            # repo root: installs git hooks

cd backend
uv sync
uv run alembic upgrade head
uv run fastapi dev app/main.py    # http://127.0.0.1:8000/docs

cd ../frontend                    # in a second terminal
npm install
npm run dev                       # http://localhost:5173
```

Run every check: `npm run check` from the repo root.
````

Commit and push:

```bash
git add .
git commit -m "chore: full pre-commit checks and run instructions"
git push
```

---

## Step 5 — Read your coverage honestly

```bash
cd backend && uv run pytest -q && cd ..
cd frontend && npm run coverage && cd ..
```

| Area | Coverage | Why |
| --- | --- | --- |
| Backend | 100% | Every line is exercised by a test |
| `formatStatus`, `ApplicationList`, `ApplicationForm` | 100% | Tested with RTL |
| `App.tsx`, `api.ts` | 0% | They talk to the network. Testing them needs **mocking**, taught in GRADUS II |

A high number isn't the goal. **Knowing exactly what isn't tested, and
why,** is.

---

## Step 6 (optional) — Solo drill

This is **retrieval practice**: doing it without the guide is what makes
it stick. Close the lessons. Use only the docs and your own code as
reference.

**Add a `location` field** (e.g. `"Remote"`, `"Denver, CO"`) to
applications, end to end, test-first.

Checklist — every box, in roughly this order:

- [ ] A backend test that creates an application with a `location` and
      checks it comes back — 🔴 red
- [ ] The column on the model
- [ ] A new Alembic migration (autogenerate it, **read it**, upgrade)
- [ ] The field in the Pydantic schemas (should it be optional?)
- [ ] 🟢 green
- [ ] The field in the TypeScript types
- [ ] A form test that types a location and expects it in `onSubmit` —
      🔴 red
- [ ] The form field — 🟢 green
- [ ] A list test that shows the location — 🔴 red, then 🟢 green
- [ ] `npm run check` passes
- [ ] Try it in the browser

Stuck? Hints, in increasing order of help:

<details>
<summary>Hint 1</summary>

Follow the path of the data from lesson 00: model → migration → schema →
API test → TypeScript type → component test → component.
</details>

<details>
<summary>Hint 2</summary>

If `location` is optional, existing rows in your database have no
value for it. The column needs `nullable=True` (in the model:
`Mapped[str | None]`), and the schemas need `str | None = None`.
</details>

<details>
<summary>Hint 3</summary>

Autogenerate should detect an added column and write
`op.add_column('applications', sa.Column('location', ...))` in
`upgrade()` and `op.drop_column(...)` in `downgrade()`.
</details>

---

## Step 7 — After Action Review

An **After Action Review** (AAR) is a short, honest debrief used by the
U.S. military and many engineering teams after any operation — good or
bad. It's not about grading yourself; it's about deciding what to do
differently next time.

Answer these four questions. Writing them down (a notebook, a notes app,
anywhere) makes them much more useful than thinking them.

1. **What was supposed to happen?**
   (What did you expect to learn or build? How long did you expect it to
   take?)
2. **What actually happened?**
   (What did you build? How long did it take? Where did you get stuck?)
3. **Why was there a difference?**
   (Which concepts were harder than expected? Which errors cost the
   most time?)
4. **What will you sustain, and what will you improve?**
   (What worked that you'll keep doing? What will you do differently in
   GRADUS II?)

---

## Self-check before GRADUS II

Can you do each of these **without looking**? Be honest — anything you
can't is exactly what GRADUS II will drill.

- [ ] Explain red → green → refactor, and why you watch a test fail
- [ ] Create a uv project and add runtime and dev dependencies
- [ ] Write a FastAPI route with a path parameter and a `404`
- [ ] Write a SQLAlchemy model and a test fixture with an in-memory
      database
- [ ] Autogenerate, read, and apply an Alembic migration
- [ ] Explain what a dependency override is for
- [ ] Scaffold a Vite React + TypeScript app and configure Vitest
- [ ] Write an RTL test that finds elements by role and label
- [ ] Use `vi.fn()` to check a callback was called with the right values
- [ ] Explain CORS in two sentences
- [ ] Set up a Husky pre-commit hook

---

## What's next

**GRADUS II — Miles.** You'll rebuild this app from an empty folder. The
tests will be provided; the code is yours. New layers:

- A `companies` table and a **one-to-many relationship**
- Statuses restricted to fixed values, with **input validation**
- One source of truth for configuration
- **Mocking `fetch`** so `App` and the API module get tested
- Error handling for every user action

*Tiro* no longer. On to *miles*.
