# Troubleshooting

Find the symptom, read the cause, apply the fix. If your problem isn't
here, **read the whole error message from the top** — the first line
usually names the problem, and the last lines usually say where it
happened.

---

## First, always check these three

1. **Am I in the right folder?** Run `pwd` (print working directory).
   Backend commands run in `backend/`, frontend commands in `frontend/`,
   Git and Husky commands at the repo root.
2. **Did I save the file?** An unsaved file has a dot on its tab in VS
   Code.
3. **Did I restart the thing?** New installs need a new terminal. Config
   changes may need the dev server restarted.

---

## General

| Symptom | Cause | Fix |
| --- | --- | --- |
| `command not found` right after installing a tool | The terminal started before the install | Close and reopen the terminal (or all of VS Code) |
| Random `EPERM`, `EBUSY`, or "file in use" errors during `npm install` or `uv sync` | The repo is inside OneDrive / iCloud / Dropbox, which locks files while syncing | Move the repo to an unsynced folder like `C:\Users\<you>\dev\` |
| `No pyproject.toml found` or `Missing script` | Wrong folder | `cd` into `backend/` or `frontend/` |

---

## Git and Husky

| Symptom | Cause | Fix |
| --- | --- | --- |
| The hook didn't run on commit | Hooks not installed on this machine | Run `npm install` at the repo root (it runs `prepare` → `husky`) |
| `\r: command not found` or `$'\r'` in hook output | The hook file has Windows (CRLF) line endings | Open `.husky/pre-commit`, click `CRLF` in VS Code's bottom-right status bar, choose `LF`, save. `.gitattributes` prevents this for new files |
| Committing from the terminal works, but VS Code's Source Control button says `uv: command not found` | VS Code's Git integration doesn't see programs added to your PATH after it started | Restart VS Code. If it persists, commit from the terminal |
| `error: src refspec main does not match any` | No commits yet, or your branch is named `master` | Make a commit first, then `git branch -M main` |
| You need to commit *right now* and the hook is failing for an unrelated reason | — | `git commit --no-verify` skips hooks. Use it rarely, and fix the cause right after |

---

## Backend

| Symptom | Cause | Fix |
| --- | --- | --- |
| `ModuleNotFoundError: No module named 'app'` | pytest can't see the `app` package | Check `pythonpath = ["."]` in `pyproject.toml`, that `app/__init__.py` exists, and that you ran pytest from `backend/` |
| VS Code: `Import "fastapi" could not be resolved` | VS Code is using a different Python | Command Palette → **Python: Select Interpreter** → `backend/.venv` |
| Server error: `no such table: applications` | Migrations haven't run, **or** you started the server from the wrong folder | From `backend/`: `uv run alembic upgrade head`, then start the server *from `backend/`*. The URL `sqlite:///./tiro.db` means "relative to the current folder," so running from elsewhere creates a new empty `tiro.db` there |
| Tests: `no such table: applications` | The test isn't using the `engine`/`client` fixture, or models weren't imported | Make sure the test takes `client` or `session` as a parameter, and `conftest.py` has `from app import models` |
| `alembic revision --autogenerate` creates an empty migration | Alembic can't see your models, or the database already matches them | Check `target_metadata = Base.metadata` and the `from app import models` import in `migrations/env.py` |
| `Target database is not up to date` | You're generating a new migration before applying the last one | `uv run alembic upgrade head` first |
| `422` when you expected `201` | The request body doesn't match the schema | Print it: `print(response.json())` in the test. The `detail` field says exactly which field is wrong |
| `[Errno 48] / [WinError 10048] address already in use` | Another server is already on port 8000 | Stop the other one (`Ctrl+C` in its terminal) |
| `ruff format --check` fails in the hook | A file isn't formatted | `uv run ruff format .` in `backend/` |
| Coverage Gutters shows nothing | No report yet, or it isn't watching | Run `uv run pytest` (creates `coverage.xml`), open a file in `app/`, then Command Palette → **Coverage Gutters: Display Coverage** |
| The Testing sidebar doesn't list Python tests | Wrong interpreter, or settings missing | Select the `.venv` interpreter; check `.vscode/settings.json` from lesson 02 |

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
| Page says "Could not load applications" | Backend not running, or CORS | Is the backend running on port 8000? Check the browser console (`F12`) for the exact error |
| CORS error after adding the middleware | The origin doesn't match *exactly* | Open the app at `http://localhost:5173`, not `127.0.0.1:5173` — they're different origins. Restart the backend after editing `main.py` if it didn't reload |
| `npm run lint` fails on files in `coverage/` | ESLint is checking the coverage report | Add `'coverage'` next to `'dist'` in `eslint.config.js` (lesson 08) |
| The pre-commit hook hangs after frontend tests | It's running watch mode | The hook must use `npm run test:run`, not `npm test` |
| Every file is reformatted with double quotes and semicolons | Prettier isn't reading your settings | Check `frontend/.prettierrc` exists (lesson 06) |

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
