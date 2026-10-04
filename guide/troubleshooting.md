# Troubleshooting

Find the symptom, read the cause, apply the fix. If your problem isn't
here, **read the whole error message from the top** — the first line
usually names the problem, and the last lines usually say where it
happened.

---

## First, always check these four

1. **Am I in the right folder?** Run `pwd` (print working directory).
   Backend commands run in `backend/`, frontend commands in `frontend/`,
   Git, Docker, and Husky commands at the repo root.
2. **Did I save the file?** An unsaved file has a dot on its tab in VS
   Code.
3. **Is the database running?** `docker compose ps` at the repo root
   should show `db` as **healthy**. If not: `docker compose up -d --wait`.
4. **Did I restart the thing?** New installs need a new terminal. Config
   changes may need the dev server restarted.

---

## General

| Symptom | Cause | Fix |
| --- | --- | --- |
| `command not found` right after installing a tool | The terminal started before the install | Close and reopen the terminal (or all of VS Code) |
| Random `EPERM`, `EBUSY`, or "file in use" errors during `npm install` or `uv sync` | The repo is inside OneDrive / iCloud / Dropbox, which locks files while syncing | Move the repo to an unsynced folder like `C:\Users\<you>\dev\` |
| `No pyproject.toml found` or `Missing script` | Wrong folder | `cd` into `backend/` or `frontend/` |

---

## uv and project setup

| Symptom | Cause | Fix |
| --- | --- | --- |
| `pyproject.toml` has `[build-system]` and/or `[project.scripts]`; `uv add` fails building a package named `backend` | uv created a *packaged* project | Delete both sections and any `src/` folder (lesson 03, Step 1). Use `--app --no-package` next time |
| VS Code: `Import "fastapi" could not be resolved`, and the interpreter list doesn't show `.venv` | VS Code is using a different Python | **Python: Select Interpreter → Enter interpreter path…** → `backend\.venv\Scripts\python.exe` (Windows) or `backend/.venv/bin/python` |
| Ruff reports `E111` / `E114` (indentation is not a multiple of four) | The file mixes 2- and 4-space indentation | Check `.vscode/settings.json` has the `[python]` indentation settings from lesson 02, then run `uv run ruff format .` in `backend/` |

---

## Docker and PostgreSQL

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Cannot connect to the Docker daemon` / `docker: command not found` | Docker Desktop isn't running (or isn't installed) | Start Docker Desktop and wait until it says it's running |
| `Bind for 0.0.0.0:5433 failed: port is already allocated` | Something else uses port 5433 — often another copy of this project | `docker compose down` in the other copy, or `docker ps` to find what's using it |
| `docker compose ps` never shows **healthy** | PostgreSQL failed to start | `docker compose logs db` and read the last lines |
| `\l` doesn't list `tiro_test` | Init scripts only run when the volume is **empty** — the volume existed before you added the script | `docker compose down -v` (**deletes all data**), then `docker compose up -d --wait` |
| `database "tiro_test" does not exist` when running tests | Same as above | Same as above |
| `password authentication failed for user "tiro"` | The URL doesn't match `compose.yaml`, or the volume was created with different credentials | Check the URL. Credentials only apply on first start; `down -v` recreates them |
| Your data vanished | You ran `docker compose down -v`, which deletes volumes | Only use `-v` when you mean to start fresh |
| The container keeps restarting after changing the image version | A different PostgreSQL major version can't read the old data | Change the version back. To switch versions, `down -v` and start fresh (development data only) |
| A `.sql` init script fails with odd characters | Windows (CRLF) line endings | Check `.gitattributes` (lesson 02); in VS Code, click `CRLF` in the status bar and choose `LF` |

---

## Async and the database

| Symptom | Cause | Fix |
| --- | --- | --- |
| `ConnectionRefusedError` / `connection refused` on port 5433 | The database isn't running | `docker compose up -d --wait` at the repo root |
| `RuntimeWarning: coroutine '…' was never awaited`, and nothing happened | A missing `await` | Find the call named in the warning and add `await` before it |
| `TypeError: object … can't be used in 'await' expression` | `await` on something that isn't async (like `db.add`) | Remove that `await`. Only database operations are awaited |
| `SyntaxError: 'await' outside async function` | `await` inside a plain `def` | Make the function `async def` |
| Error mentioning **greenlet** | SQLAlchemy was installed without its asyncio extra | `uv add "sqlalchemy[asyncio]"` in `backend/` |
| `Can't load plugin: sqlalchemy.dialects:postgresql.psycopg2` | The URL is missing `+asyncpg` | Use `postgresql+asyncpg://…` |
| `MissingGreenlet: greenlet_spawn has not been called` | Code read an attribute that SQLAlchemy needed to reload from the database without an `await` | Make sure the session factory has `expire_on_commit=False`, and `await db.refresh(obj)` after a commit |
| Tests: `… attached to a different loop` / `Event loop is closed` | Mixing `TestClient` (its own loop) with async database fixtures, or a pooled connection reused across tests | Use the `client` fixture from lesson 06 (an `AsyncClient`), and `poolclass=NullPool` in the test engine |
| Tests are skipped, or `async def functions are not natively supported` | pytest-asyncio isn't configured | Check `asyncio_mode = "auto"` in `pyproject.toml` (lesson 05, Step 2) |
| Tests: `relation "applications" does not exist` | The test isn't using the `engine`/`session`/`client` fixture, or models weren't imported | Take `client` or `session` as a parameter; check `conftest.py` has `from app import models` |
| Server: `relation "applications" does not exist` | Migrations haven't been applied to the **development** database | `uv run alembic upgrade head` in `backend/` |

---

## Backend

| Symptom | Cause | Fix |
| --- | --- | --- |
| `ModuleNotFoundError: No module named 'app'` | pytest can't see the `app` package | Check `pythonpath = ["."]` in `pyproject.toml`, that `app/__init__.py` exists, and that you ran pytest from `backend/` |
| `alembic revision --autogenerate` creates an empty migration | Alembic can't see your models, or the database already matches them | Check `target_metadata = Base.metadata` and the `from app import models` import in `migrations/env.py` |
| `Target database is not up to date` | You're generating a new migration before applying the last one | `uv run alembic upgrade head` first |
| `422` when you expected `201` | The request body doesn't match the schema | Print it: `print(response.json())` in the test. The `detail` field says exactly which field is wrong |
| `[Errno 48]` / `[WinError 10048]` address already in use | Another server is already on port 8000 | Stop the other one (`Ctrl+C` in its terminal) |
| `ruff format --check` fails in the hook | A file isn't formatted | `uv run ruff format .` in `backend/` |
| Coverage Gutters shows nothing | No report yet, or it isn't watching | Run `uv run pytest` (creates `coverage.xml`), open a file in `app/`, then **Coverage Gutters: Display Coverage** |
| The Testing sidebar doesn't list Python tests | Wrong interpreter, or settings missing | Select the `.venv` interpreter; check `.vscode/settings.json` from lesson 02 |

---

## Git and Husky

| Symptom | Cause | Fix |
| --- | --- | --- |
| The hook didn't run on commit | Hooks not installed on this machine | Run `npm install` at the repo root (it runs `prepare` → `husky`) |
| The hook fails at "Starting the database…" | Docker Desktop isn't running | Start it and commit again |
| `\r: command not found` or `$'\r'` in hook output | The hook file has Windows (CRLF) line endings | Open `.husky/pre-commit`, click `CRLF` in VS Code's status bar, choose `LF`, save |
| Committing from the terminal works, but VS Code's Source Control button says `uv: command not found` | VS Code's Git integration doesn't see programs added to your PATH after it started | Restart VS Code. If it persists, commit from the terminal |
| `error: src refspec main does not match any` | No commits yet, or your branch is named `master` | Make a commit first, then `git branch -M main` |
| You need to commit *right now* and the hook is failing for an unrelated reason | — | `git commit --no-verify` skips hooks. Use it rarely, and fix the cause right after |

---

## Frontend

| Symptom | Cause | Fix |
| --- | --- | --- |
| Every `npm` command fails with a JSON error | A missing or extra comma in `package.json` | Check the `"scripts"` block: commas between entries, none after the last |
| `Invalid Chai property: toBeInTheDocument` | jest-dom isn't loaded | Check `setupFiles: './src/setupTests.ts'` in `vite.config.ts` and the first line of `setupTests.ts` |
| `Found multiple elements with the text…` in a test that should render one thing | Leftovers from a previous test | Check the `afterEach(() => { cleanup() })` in `setupTests.ts` |
| `Unable to find an element with the text…` | The text doesn't match *exactly*, or it's split across elements | Add `screen.debug()` in the test to print the rendered HTML, and compare |
| Typing into the date field doesn't work in a test | Date inputs behave differently in the fake browser | Import `fireEvent` from `@testing-library/react` and use `fireEvent.change(input, { target: { value: '2026-10-01' } })` |
| `'X' is a type and must be imported using a type-only import` | `verbatimModuleSyntax` is on in the template's settings | Change `import { X }` to `import type { X }` |
| Red squiggle under `test:` in `vite.config.ts` | Missing reference line | First line must be `/// <reference types="vitest/config" />` |
| Page says "Could not load applications" | Backend not running, database not running, or CORS | Check all three; then the browser console (`F12`) for the exact error |
| CORS error after adding the middleware | The origin doesn't match *exactly* | Open the app at `http://localhost:5173`, not `127.0.0.1:5173` — they're different origins. Restart the backend if it didn't reload |
| `npm run lint` fails on files in `coverage/` | ESLint is checking the coverage report | Add `'coverage'` next to `'dist'` in `eslint.config.js` (lesson 09) |
| The pre-commit hook hangs after frontend tests | It's running watch mode | The hook must use `npm run test:run`, not `npm test` |
| Every file is reformatted with double quotes and semicolons | Prettier isn't reading your settings | Check `frontend/.prettierrc` exists (lesson 07) |

---

## Still stuck?

1. Re-read the lesson step from the top — the "Connect the dots" part
   often explains the missing piece.
2. Compare your file line by line with the complete listing in the
   lesson.
3. Search the exact error message in the official docs linked at the end
   of each lesson.
4. Take a break. Seriously — a lot of bugs are found in the first
   minute back.
