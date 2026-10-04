# 03 — Backend: first test

> **Where this fits.** Every user story needs a backend that can answer
> requests. Before it can store applications (US-1, US-5), it has to
> exist and respond at all. This lesson builds the smallest possible
> backend — one endpoint that says "I'm alive" — and proves it works
> with your first test.

**Goal:** a Python project managed by uv, a FastAPI app with one
endpoint (`GET /health`), your first red → green TDD cycle, coverage in
the editor, and the backend tests running in your pre-commit hook.

---

## Concepts first

| Term | Plain English |
| --- | --- |
| **FastAPI** | A Python **web framework**: you write functions, and FastAPI calls them when matching HTTP requests arrive, then turns their return values into HTTP responses |
| **Endpoint** (or **route**) | One URL + method the server answers, like `GET /health` |
| **pytest** | Python's most popular test runner. It finds functions named `test_…` and runs them |
| **Package** | A folder of Python files you can import from, marked by an `__init__.py` file |

---

## Step 1 — Create the Python project with uv

From the **repo root**:

```bash
uv init backend --app --no-package --no-readme --vcs none --python 3.12
```

| Part | Meaning |
| --- | --- |
| `uv init backend` | Create a new Python project in a folder named `backend` |
| `--app` | This is an *application* (something you run), not a library |
| `--no-package` | Don't set it up to be built and published as an installable package |
| `--no-readme` | Skip creating a README (the repo already has one) |
| `--vcs none` | Don't create a new Git repo inside — we're already in one |
| `--python 3.12` | Use Python 3.12 for this project |

✅ A `backend/` folder appears containing:

- `pyproject.toml` — the project's settings and dependency list
- `.python-version` — tells uv which Python to use
- `main.py` — a sample file

Delete `backend/main.py`. Your app will live in its own folder instead.

Open `backend/pyproject.toml` and read it. It's short (yours may also
have an `authors` line, filled in from your Git settings):

```toml
[project]
name = "backend"
version = "0.1.0"
description = "Add your description here"
requires-python = ">=3.12"
dependencies = []
```

`dependencies = []` is empty — you haven't installed anything yet.

> ⚠️ **Does your `pyproject.toml` have a `[build-system]` section, or a
> `[project.scripts]` section?** Then uv created a *packaged* project —
> one meant to be built and published, which expects its code in
> `src/backend/`. Our app doesn't work that way. **Delete both
> sections**, delete any `src/` folder uv created, and continue. (The
> `--no-package` flag prevents this; some uv versions default
> differently.)

---

## Step 2 — Install packages

Move into the backend folder. **From now on in this lesson, stay in
`backend/`.**

```bash
cd backend
```

Install the app's dependencies:

```bash
uv add "fastapi[standard]"
```

✅ uv creates a `.venv/` folder, installs FastAPI into it, adds
`fastapi[standard]` to `dependencies` in `pyproject.toml`, and creates
`uv.lock`.

- **`[standard]`** is an "extra": it also installs the `fastapi` command
  line tool, **uvicorn** (the server that actually runs your app), and
  **httpx** (which you'll use for testing in lesson 06).
- **`uv.lock`** records exact versions of everything installed. Commit
  it, so everyone gets identical packages.

Now the development-only tools:

```bash
uv add --dev pytest pytest-cov ruff
```

| Package | What it does |
| --- | --- |
| `pytest` | Finds and runs your tests |
| `pytest-cov` | Adds coverage measurement to pytest |
| `ruff` | Lints (finds mistakes) and formats Python code |

✅ `pyproject.toml` now has a `[dependency-groups]` section with a `dev`
list. These are installed for development but aren't part of the app.

> **`uv run`** — you'll type this a lot. `uv run <command>` runs a
> command inside this project's `.venv`, so you never have to
> "activate" the virtual environment manually.

---

## Step 3 — Point VS Code at the virtual environment

1. Command Palette: `Ctrl+Shift+P` / `Cmd+Shift+P`
2. **Python: Select Interpreter**
3. Choose the interpreter whose path contains `backend/.venv`.
   If no option contains that path, that's okay — the environment may
   not show up in the list automatically. Choose **Enter interpreter
   path…** and browse to the project's environment (**not** the global
   Python installation):
   - Windows: `backend\.venv\Scripts\python.exe`
   - macOS / Linux: `backend/.venv/bin/python`

✅ Red squiggles under `fastapi` imports (which you'll write next) won't
appear, because VS Code can now see the installed packages.

---

## Step 4 — Configure pytest and Ruff

📄 **File:** `backend/pyproject.toml` — **edit**: add these sections at
the **bottom** of the file:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]
addopts = [
    "--cov=app",
    "--cov-report=term-missing",
    "--cov-report=xml",
]

[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I"]
```

| Setting | What it does |
| --- | --- |
| `testpaths` | Look for tests in the `tests/` folder |
| `pythonpath = ["."]` | Lets tests write `from app.main import app` |
| `--cov=app` | Measure coverage of the `app/` package |
| `--cov-report=term-missing` | Print a coverage table listing untested line numbers |
| `--cov-report=xml` | Also write `coverage.xml`, which Coverage Gutters reads |
| Ruff `line-length` | Lines up to 88 characters (Ruff's default, matching the Black formatter) |
| Ruff `select` | `E` = style errors (including indentation), `F` = likely bugs, `I` = import order |

**`addopts`** means "add these options every time pytest runs," so plain
`uv run pytest` always includes coverage.

---

## Step 5 — Create the folders

Create these inside `backend/`:

```text
backend/
├── app/
│   └── __init__.py   ← new, empty file
└── tests/            ← new, empty folder
```

📄 **File:** `backend/app/__init__.py` — **new**, leave it **empty**.

**What's `__init__.py`?** An empty file that tells Python "this folder is
a **package**" — a group of modules you can import from. It's what makes
`from app.main import app` work.

---

## Step 6 — 🔴 Red: write the first test

**Connect the dots.** We want an endpoint that answers "is the server
alive?" — a **health check**. Real deployments use these so monitoring
tools know the app is up.

- The **request** will be: `GET /health`
- The **response** should be: status `200`, body `{"status": "ok"}`
- To test it without starting a real server, FastAPI provides
  **`TestClient`**, which sends requests straight to your app in memory.

*Before reading on: using Arrange / Act / Assert, what would each part
of this test be?*

📄 **File:** `backend/tests/test_health.py` — **new**:

```python
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_health_returns_ok():
    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

Line by line:

- **Line 1** imports `TestClient`.
- **Line 3** imports `app` from `app/main.py` — a file that doesn't
  exist yet.
- **Line 5** is the *arrange* step: a client connected to your app.
- **Line 8**: pytest runs every function whose name starts with
  `test_`.
- **Line 9** is the *act*: send `GET /health`.
- **Lines 11–12** are the *assert*s. `assert` raises an error if the
  condition is false, which makes the test fail.

Run it (from `backend/`):

```bash
uv run pytest
```

🔴 **Expected:** an error mentioning
`ModuleNotFoundError: No module named 'app.main'`. That's red — the test
can't even find the code. Good.

📄 **File:** `backend/app/main.py` — **new**, with *only* this:

```python
from fastapi import FastAPI

app = FastAPI(title="Tiro Job Tracker")
```

- **Line 1** imports the `FastAPI` class.
- **Line 3** creates your application object. Every route you write
  will be attached to it. `title` appears at the top of the API docs.

Run again:

```bash
uv run pytest
```

🔴 **Expected:** `1 failed`, with `assert 404 == 200`. The app exists,
but `/health` doesn't — so FastAPI answers **404 Not Found**. Still red,
but now for the *right reason*. This is exactly what the test should
catch.

---

## Step 7 — 🟢 Green: make it pass

📄 **File:** `backend/app/main.py` — **edit**: add these lines **below
line 3** (leave two blank lines after line 3, as Python style expects
before a function):

```python
@app.get("/health")
def health():
    return {"status": "ok"}
```

The whole file is now:

```python
from fastapi import FastAPI

app = FastAPI(title="Tiro Job Tracker")


@app.get("/health")
def health():
    return {"status": "ok"}
```

- **Line 6, `@app.get("/health")`**, is a **decorator**: it registers the
  function below it to handle `GET` requests to `/health`.
- **Line 7** defines the function. Its name doesn't matter to the URL;
  the decorator decides that.
- **Line 8**: FastAPI turns the returned dictionary into JSON
  automatically.

Run:

```bash
uv run pytest
```

🟢 **Expected** (exact formatting may differ slightly):

```text
tests/test_health.py .                                   [100%]

---------- coverage: ... ----------
Name              Stmts   Miss  Cover   Missing
-----------------------------------------------
app/__init__.py       0      0   100%
app/main.py           5      0   100%
-----------------------------------------------
TOTAL                 5      0   100%
Coverage XML written to file coverage.xml

1 passed in 0.xx s
```

The `.` after the filename means one passing test. `Missing` is empty
because every line ran.

**🔵 Refactor?** Nothing to clean up yet. Move on.

---

## Step 8 — See it for real

Start the development server (still in `backend/`):

```bash
uv run fastapi dev app/main.py
```

✅ The terminal shows the server running at `http://127.0.0.1:8000`.

Open these in your browser:

- <http://127.0.0.1:8000/health> — shows `{"status":"ok"}`
- <http://127.0.0.1:8000/docs> — **interactive API docs** FastAPI
  generates for you. Click `/health` → **Try it out** → **Execute**.

`dev` mode **reloads** when you save a file, so you can leave it
running. Stop it with `Ctrl+C`.

---

## Step 9 — Coverage Gutters

1. Make sure `backend/coverage.xml` exists (it's created every time you
   run `uv run pytest`).
2. Open `backend/app/main.py`.
3. Command Palette → **Coverage Gutters: Display Coverage**.

✅ Green bars appear beside the lines your tests ran.

**Better:** click **Watch** in the bottom status bar. Coverage Gutters
will refresh automatically whenever `coverage.xml` changes — so every
test run updates the colors.

| Color | Meaning |
| --- | --- |
| Green | This line ran during tests |
| Red | This line never ran — no test checks it |
| Yellow | Partly ran (e.g. only one side of an `if`) |

---

## Step 10 — Lint and format

Still in `backend/`:

```bash
uv run ruff check .
uv run ruff format .
```

✅ `ruff check` prints `All checks passed!`. `ruff format` reports how
many files it reformatted (possibly `0`).

---

## Step 11 — Add backend tests to the pre-commit hook

📄 **File:** `.husky/pre-commit` (at the **repo root**) — **edit**:
replace the whole file with:

```sh
echo "Running backend tests..."
(cd backend && uv run pytest -q)
```

- The **parentheses** run the command in a *subshell*: it `cd`s into
  `backend`, runs the tests, and the `cd` doesn't leak into later
  lines.
- **`-q`** means *quiet*: shorter output.
- If any test fails, pytest exits with an error, and Husky cancels the
  commit.

---

## Step 12 — Commit

Go back to the repo root first:

```bash
cd ..
git add .
git commit -m "feat(backend): add FastAPI app with health check"
git push
```

✅ You see `Running backend tests...`, then the test output, then the
commit summary.

**Try breaking it on purpose:** in `backend/app/main.py`, change `"ok"`
to `"okay"` on line 8. Run `git add .` and `git commit -m "test hook"`.
The hook should fail and the commit should be cancelled. Change it back.

---

## Explain it back

**1. The first time you ran pytest, it failed with
`ModuleNotFoundError`. The second time it failed with `404 == 200`.
Why is the second failure more useful?**

<details>
<summary>Answer</summary>

The first failure only proved the file didn't exist. The second proved
the test really checks the endpoint's behavior: the app ran, the route
was missing, and the test caught it. That's the test failing for the
right reason.
</details>

**2. What does `uv run` do that plain `pytest` wouldn't?**

<details>
<summary>Answer</summary>

It runs the command using this project's `.venv`, where pytest and
FastAPI are installed. Plain `pytest` might not exist on your system, or
might use a different Python with different packages.
</details>

**3. What does `@app.get("/health")` do?**

<details>
<summary>Answer</summary>

It's a decorator that registers the function below it as the handler
for `GET` requests to `/health`.
</details>

**4. What file does Coverage Gutters read for the backend, and what
creates it?**

<details>
<summary>Answer</summary>

`backend/coverage.xml`, created by pytest-cov every time you run
`uv run pytest` (because of `--cov-report=xml` in `addopts`).
</details>

---

## Checkpoint

- ✅ `uv run pytest` (in `backend/`) prints `1 passed` and 100%
  coverage.
- ✅ <http://127.0.0.1:8000/docs> shows your `/health` endpoint.
- ✅ Coverage Gutters shows green lines in `app/main.py`.
- ✅ Committing runs your backend tests.

Docs for going deeper:

- FastAPI first steps: <https://fastapi.tiangolo.com/tutorial/first-steps/>
- FastAPI testing: <https://fastapi.tiangolo.com/tutorial/testing/>
- uv projects: <https://docs.astral.sh/uv/guides/projects/>
- pytest: <https://docs.pytest.org/en/stable/getting-started.html>

Next: [04 — Docker and PostgreSQL](04-docker-and-postgres.md)
