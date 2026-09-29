# GRADUS I — Tiro

> **🚧 Beta.** This guide has not yet been fully tested end to end by
> students. Tool versions change, and some commands, outputs, or line
> numbers may not match what you see. If something doesn't work, check
> [`guide/troubleshooting.md`](guide/troubleshooting.md) first, then
> [report it](#found-a-problem) so the next person doesn't hit it.

**GRADUS**: **G**uided **R**epetitions in **A**pplied **D**evelopment
**U**sing the **S**tack

*Tiro* (Latin): a recruit, a beginner. This is the first of three tiers.

| Tier | Latin | Meaning | Guidance level |
| --- | --- | --- | --- |
| I | *tiro* | recruit | Full worked examples. Every line explained. |
| II | *miles* | soldier | Tests given to you. You write the code. |
| III | *veteranus* | veteran | Requirements only. You write tests and code. |

*Gradus* means "step" or "rank" — the root of *grade* and *graduate*.

---

## What you'll build

A **job application tracker**: a small full-stack web app where you log
the jobs you've applied to and move each one through a status
(applied → interviewing → offer / rejected).

- Add an application (company, role, status, date applied)
- See all your applications in a list
- Change an application's status
- Delete an application

All three GRADUS tiers build this **same app**. Each tier you start from
an empty folder, rebuild the core with less help, and add one new layer.
You spend your effort on *how to build it better*, not on learning a
new domain every time.

## Who this is for

Someone who has written a little code before but has **never used this
stack**. You do not need to know React, TypeScript, FastAPI, SQL, or
testing. Every tool is installed, explained, and used step by step.

**No AI assistant required.** Everything you need is in this repo:
worked examples, expected output at every checkpoint, "explain it back"
questions with hidden answers, a troubleshooting page, a glossary, and
links to official documentation. You may use an AI assistant if you
like, but you shouldn't need one.

---

## The stack

| Layer | Tool | What it does |
| --- | --- | --- |
| Frontend language | **TypeScript** | JavaScript with type checking |
| Frontend library | **React** | Builds the UI from components |
| Frontend tooling | **Vite** | Dev server and build tool |
| Frontend tests | **Vitest** + **React Testing Library** | Runs tests, renders components |
| Backend language | **Python** | — |
| Environment manager | **uv** | Installs Python, packages, and the virtual environment |
| Web framework | **FastAPI** | Receives HTTP requests, sends responses |
| ORM | **SQLAlchemy** | Lets Python classes stand in for database tables |
| Migrations | **Alembic** | Version control for your database's structure |
| Database | **SQLite** | The actual storage — a single file |
| Backend tests | **pytest** + **pytest-cov** | Runs tests, measures coverage |
| Git hooks | **Husky** | Runs your tests automatically before each commit |
| Linting | **Ruff** (Python), **ESLint** (TS) | Catches mistakes and style issues |

> **FastAPI, SQLAlchemy, Alembic, and uv are not databases.** They are
> tools that *talk to* a database. The database here is SQLite. In
> GRADUS III you swap it for PostgreSQL and see why that separation
> matters.

---

## Skills introduced in Tiro

Every skill below is **introduced** here with a full worked example.
GRADUS II and III repeat them with less help and add new ones.

### Tooling and workflow

- Installing Python with **uv** and managing a virtual environment
- Installing Node.js packages with **npm**
- Structuring one repo with `backend/` and `frontend/` folders
- Writing a `.gitignore` and `.gitattributes`
- Committing in small steps with clear commit messages
- **Husky** pre-commit hooks that run tests and linters
- **VS Code** workspace settings and recommended extensions

### Testing (TDD)

- The **red → green → refactor** loop
- Writing a test *before* the code it tests
- **pytest** tests, fixtures, and `conftest.py`
- Using a throwaway in-memory database for tests
- FastAPI's `TestClient` and dependency overrides
- **Vitest** tests for plain TypeScript functions
- **React Testing Library**: rendering components, querying by role
  and label, simulating clicks and typing with `user-event`
- Mock functions (`vi.fn()`) to check a callback was called
- Reading **coverage** reports and **Coverage Gutters** in the editor

### Backend

- A FastAPI app with a health-check endpoint
- **CRUD** endpoints: `POST`, `GET` (list and one), `PATCH`, `DELETE`
- HTTP status codes: `200`, `201`, `204`, `404`, `422`
- **Pydantic** schemas for input validation and output shape
- FastAPI **dependency injection** (`Depends`) for database sessions
- **CORS** — why the browser blocks requests, and how to allow them
- The auto-generated API docs at `/docs`

### Database

- What an ORM is and why it exists
- A **SQLAlchemy 2.0** model with typed columns
- Engines, sessions, and the session lifecycle
- **Alembic**: initializing, autogenerating a migration, upgrading
- Inspecting a SQLite database file

### Frontend

- Scaffolding a React + TypeScript app with **Vite**
- TypeScript **types**, `type` imports, and utility types (`Omit`)
- **Components** and **props**
- **State** with `useState`, side effects with `useEffect`
- **Controlled** form inputs and form submission
- Rendering lists with `key`
- Calling a backend with `fetch` and `async`/`await`
- Accessible labels (which also make components easier to test)

---

## How to use this repo

### Get your own copy first

**Don't work directly in this repo.** Your code would end up mixed into
the guide everyone else clones.

1. On this repo's GitHub page, click **Use this template → Create a new
   repository**. Give it a name (e.g. `tiro-yourname`).
2. Clone *your new repo* to your computer.
3. Start at lesson 00 below.

Your repo gets the guide and nothing else; your work stays yours.

### The lessons

The guide lives in [`guide/`](guide/). Work through it in order.

| # | Lesson | You'll have at the end |
| --- | --- | --- |
| 00 | [Orientation](guide/00-orientation.md) | A mental map of the app and TDD |
| 01 | [Install your tools](guide/01-install-tools.md) | Git, Node, uv, VS Code, extensions |
| 02 | [Scaffold the repo](guide/02-scaffold-repo.md) | Folders, ignores, Husky, VS Code settings |
| 03 | [Backend: first test](guide/03-backend-init.md) | FastAPI running, first passing test |
| 04 | [Database and migrations](guide/04-database-and-migrations.md) | A model, a migration, a `.db` file |
| 05 | [CRUD endpoints with TDD](guide/05-crud-endpoints.md) | A fully tested API |
| 06 | [Frontend: first test](guide/06-frontend-init.md) | Vite + Vitest running, first passing test |
| 07 | [Components with TDD](guide/07-components.md) | A working UI connected to the API |
| 08 | [Guardrails and debrief](guide/08-guardrails-and-debrief.md) | Full pre-commit hook, final checks, AAR |

Also keep open:

- [`guide/troubleshooting.md`](guide/troubleshooting.md) — common errors
  and their fixes
- [`guide/glossary.md`](guide/glossary.md) — every new term, in plain
  words

**Time:** plan on 8–14 hours total. Lessons 05 and 07 are the longest.

### Every lesson uses the same pattern

1. **Goal** — what you'll have when the step is done.
2. **Connect the dots** — what information the step needs and where it
   lives. Pause and answer the question before reading on.
3. **🔴 Red** — write a test that fails.
4. **🟢 Green** — write the smallest code that makes it pass.
5. **🔵 Refactor** — clean up while the tests stay green (when needed).
6. **Explain it back** — answer in your own words, then check the
   hidden answer.
7. **Checkpoint** — the exact thing you should see before moving on.

Type the code yourself instead of copying and pasting. It's slower, and
that's the point: your fingers learn the shapes.

---

## Recommended VS Code extensions

When you open this folder in VS Code, it will offer to install these
(they're listed in [`.vscode/extensions.json`](.vscode/extensions.json)).
Lesson 01 walks through each one.

| Extension | Why |
| --- | --- |
| Python (`ms-python.python`) | Run, debug, and test Python |
| Pylance (`ms-python.vscode-pylance`) | Python autocomplete and type checking |
| Ruff (`charliermarsh.ruff`) | Python linting and formatting |
| Even Better TOML (`tamasfe.even-better-toml`) | Highlights `pyproject.toml` |
| ESLint (`dbaeumer.vscode-eslint`) | TypeScript linting in the editor |
| Prettier (`esbenp.prettier-vscode`) | TypeScript formatting |
| Vitest (`vitest.explorer`) | Run frontend tests from the sidebar |
| Coverage Gutters (`ryanluker.vscode-coverage-gutters`) | Shows tested/untested lines in the editor margin |
| SQLite Viewer (`qwtel.sqlite-viewer`) | Open `.db` files as tables |
| Pretty TypeScript Errors (`yoavbls.pretty-ts-errors`) | Makes TS errors readable |

---

## Why it's built this way

- **Worked examples first, then fade them out.** New learners learn
  faster by studying complete solutions than by struggling from scratch
  (Sweller's *worked-example effect*). As skill grows, the help should
  shrink step by step — Renkl and Atkinson call this *fading*. If the
  help stays, it starts getting in the way (Kalyuga's *expertise
  reversal effect*). That's why Tiro shows everything and Veteranus
  shows almost nothing.
- **Rebuild instead of re-read.** Pulling knowledge out of your own
  head (*retrieval practice*) builds memory far better than reviewing
  it. Robert and Elizabeth Bjork call helpful struggle a *desirable
  difficulty*. Rebuilding the same app three times is retrieval
  practice; adding a new layer each time keeps it from becoming
  memorized answers.
- **Tests first.** Kent Beck's *Test-Driven Development: By Example*
  (2002) describes the red-green-refactor loop used throughout. Tests
  also make this repo AI-independent: they tell you, without anyone's
  help, whether your code works.

---

## Found a problem?

This is a beta, so problem reports are the most useful thing you can
give back. On this repo's GitHub page, open **Issues → New issue** and
include:

- the lesson and step (e.g. "05, Cycle 3")
- what you expected to happen
- what happened instead, with the **full** error message
- your operating system (Windows / macOS / Linux)

---

## After Tiro

Move on to **GRADUS II — Miles**. You'll rebuild this app from an empty
folder using provided tests, add a `companies` table with a
relationship, restrict status to fixed values, and learn to mock API
calls in frontend tests.
