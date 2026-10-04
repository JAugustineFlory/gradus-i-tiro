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

### User stories

A **user story** describes a feature from the point of view of the
person using it: *As a…, I want…, so that…*. Every lesson in this guide
exists to deliver one or more of these:

| ID | Story |
| --- | --- |
| **US-1** | As a job seeker, I want to **record** each application I send (company, role, date), so that I don't lose track of where I've applied. |
| **US-2** | As a job seeker, I want to **see all** my applications in one list, so that I know where I stand at a glance. |
| **US-3** | As a job seeker, I want to **update** an application's status as it moves along, so that the list reflects reality. |
| **US-4** | As a job seeker, I want to **delete** an application I logged by mistake, so that my list stays accurate. |
| **US-5** | As a job seeker, I want my applications to **still be there** tomorrow, so that I can rely on the tracker. |

All three GRADUS tiers build this **same app**. Each tier you start from
an empty folder, rebuild the core with less help, and add one new layer.

## Who this is for

Someone who has written a little code before but has **never used this
stack**. You do not need to know React, TypeScript, FastAPI, SQL,
Docker, or testing. Every tool is installed, explained, and used step by
step.

**No AI assistant required.** Everything you need is in this repo:
worked examples, expected output at every checkpoint, "explain it back"
questions with hidden answers, a troubleshooting page, a glossary, and
links to official documentation.

---

## The stack

### Backend

| Tool | What it does |
| --- | --- |
| **Python** | The backend's language |
| **uv** | Installs Python and packages, and manages the virtual environment |
| **FastAPI** | Web framework: receives HTTP requests and sends responses |
| **Pydantic** | Validates data coming in and shapes data going out |
| **SQLAlchemy** | ORM: lets Python classes stand in for database tables |
| **asyncpg** | The driver SQLAlchemy uses to talk to PostgreSQL, asynchronously |
| **Alembic** | Migrations: version control for your database's structure |
| **PostgreSQL** | The database — where the data actually lives |
| **Docker** | Runs PostgreSQL in an isolated container, identically on every machine |
| **pytest** (+ pytest-asyncio, pytest-cov) | Runs tests, including `async` ones, and measures coverage |
| **Ruff** | Lints and formats Python |

### Frontend

| Tool | What it does |
| --- | --- |
| **TypeScript** | JavaScript with type checking |
| **React** | Builds the UI from components |
| **Vite** | Dev server and build tool |
| **Vitest** + **React Testing Library** | Runs tests and renders components in them |
| **ESLint** + **Prettier** | Lints and formats TypeScript |

### Workflow

| Tool | What it does |
| --- | --- |
| **Git** + GitHub | Version control |
| **Husky** | Runs your checks automatically before each commit |
| **VS Code** | Editor, with recommended extensions |

> **FastAPI, Pydantic, SQLAlchemy, asyncpg, Alembic, and uv are not
> databases.** They are tools that *talk to* a database. The database is
> **PostgreSQL**, running inside **Docker**.

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

### Infrastructure

- **Docker** fundamentals: images, containers, volumes, port mappings
- **Docker Compose**: defining and starting a PostgreSQL service
- Creating a separate **test database**
- Exploring a database with **`psql`**

### Testing (TDD)

- The **red → green → refactor** loop
- Writing a test *before* the code it tests
- **pytest** tests, fixtures, and `conftest.py`
- **Async tests** with pytest-asyncio
- A dedicated PostgreSQL **test database**, reset for every test
- FastAPI testing with **httpx's `AsyncClient`** and dependency
  overrides
- **Vitest** tests for plain TypeScript functions
- **React Testing Library**: rendering components, querying by role and
  label, simulating clicks and typing with `user-event`
- Mock functions (`vi.fn()`) to check a callback was called
- Reading **coverage** reports and **Coverage Gutters** in the editor

### Backend

- **`async` / `await`** and why web servers use them
- A FastAPI app with a health-check endpoint
- **CRUD** endpoints: `POST`, `GET` (list and one), `PATCH`, `DELETE`
- HTTP status codes: `200`, `201`, `204`, `404`, `405`, `422`
- **Pydantic** schemas for input validation and output shape
- FastAPI **dependency injection** (`Depends`) for database sessions
- **CORS** — why the browser blocks requests, and how to allow them
- The auto-generated API docs at `/docs`

### Database

- What an ORM is and why it exists
- A **SQLAlchemy 2** model with typed columns
- **Async** engines and sessions with **asyncpg**
- **Alembic** with the async template: autogenerate, read, upgrade,
  downgrade

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

### The lessons

The guide lives in [`guide/`](guide/). Work through it in order.

| # | Lesson | Stories | You'll have at the end |
| --- | --- | --- | --- |
| 00 | [Orientation](guide/00-orientation.md) | all | A mental map of the app and of TDD |
| 01 | [Install your tools](guide/01-install-tools.md) | — | Git, Node, uv, Docker, VS Code, extensions |
| 02 | [Scaffold the repo](guide/02-scaffold-repo.md) | — | Ignore rules, Husky, VS Code settings |
| 03 | [Backend: first test](guide/03-backend-init.md) | — | FastAPI running, first passing test |
| 04 | [Docker and PostgreSQL](guide/04-docker-and-postgres.md) | US-5 | A real database server, and a test database |
| 05 | [Async database layer and migrations](guide/05-database-and-migrations.md) | US-5 | A model, an async session, a migration |
| 06 | [CRUD endpoints with TDD](guide/06-crud-endpoints.md) | US-1–4 | A fully tested API |
| 07 | [Frontend: first test](guide/07-frontend-init.md) | — | Vite + Vitest running, first passing test |
| 08 | [Components with TDD](guide/08-components.md) | US-1–4 | A working UI connected to the API |
| 09 | [Guardrails and debrief](guide/09-guardrails-and-debrief.md) | all | Full pre-commit hook, final checks, AAR |

Also keep open:

- [`guide/troubleshooting.md`](guide/troubleshooting.md) — common errors
  and their fixes
- [`guide/glossary.md`](guide/glossary.md) — every new term, in plain
  words

**Time:** plan on 10–16 hours total. Lessons 05, 06, and 08 are the
longest.

### Every lesson uses the same pattern

1. **Where this fits** — the user story it serves, and where the work
   sits in the big picture.
2. **Goal** — what you'll have when the lesson is done.
3. **Concepts first** — every new tool or idea, defined in plain words,
   with why it exists.
4. **Connect the dots** — what information a step needs and where it
   lives. Pause and answer the question before reading on.
5. **📄 File** — every code block says *exactly* which file it goes in,
   whether that file is new or being edited, and where in the file.
6. **🔴 Red** — write a test that fails. **🟢 Green** — write the
   smallest code that makes it pass. **🔵 Refactor** — clean up while
   the tests stay green.
7. **Explain it back** — answer in your own words, then check the hidden
   answer.
8. **Checkpoint** — the exact thing you should see before moving on.

The first time a concept appears, it gets a full explanation. The more
you've used it, the shorter the explanation gets — that's on purpose.

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
| Docker (`ms-azuretools.vscode-docker`) | Highlights `compose.yaml`; shows running containers |
| ESLint (`dbaeumer.vscode-eslint`) | TypeScript linting in the editor |
| Prettier (`esbenp.prettier-vscode`) | TypeScript formatting |
| Vitest (`vitest.explorer`) | Run frontend tests from the sidebar |
| Coverage Gutters (`ryanluker.vscode-coverage-gutters`) | Shows tested/untested lines in the editor margin |
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

- the lesson and step (e.g. "06, Cycle 3")
- what you expected to happen
- what happened instead, with the **full** error message
- your operating system (Windows / macOS / Linux)

---

## After Tiro

Move on to **GRADUS II — Miles**. You'll rebuild this app from an empty
folder using provided tests, add a `companies` table with a
relationship, restrict status to fixed values, and learn to mock API
calls in frontend tests.
