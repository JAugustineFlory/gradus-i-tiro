# 00 — Orientation

**Goal:** before touching any code, understand what you're building,
for whom, what the pieces are, how they talk to each other, and what
"test-driven development" means.

No commands in this lesson. Read it, answer the questions at the end,
and move on.

---

## Who it's for: the user stories

Software exists to help someone do something. A **user story** captures
that in one sentence: *As a [person], I want [something], so that
[reason]*. Here are the five this tracker delivers (also in the README):

| ID | Story |
| --- | --- |
| **US-1** | As a job seeker, I want to **record** each application I send, so that I don't lose track of where I've applied. |
| **US-2** | As a job seeker, I want to **see all** my applications in one list, so that I know where I stand at a glance. |
| **US-3** | As a job seeker, I want to **update** an application's status, so that the list reflects reality. |
| **US-4** | As a job seeker, I want to **delete** an application I logged by mistake, so that my list stays accurate. |
| **US-5** | As a job seeker, I want my applications to **still be there tomorrow**, so that I can rely on the tracker. |

Every lesson starts with a **Where this fits** box naming the story it
serves. When a step feels abstract, look back at the story: that's the
person the step is for.

---

## The big picture

A web app is **three programs** working together:

```mermaid
flowchart LR
    U[You, in a browser] --> F
    subgraph Frontend [frontend/ — port 5173]
      F[React components<br/>written in TypeScript]
    end
    F -- "HTTP request<br/>(fetch)" --> B
    B -- "JSON response" --> F
    subgraph Backend [backend/ — port 8000]
      B[FastAPI routes] --> O[SQLAlchemy models<br/>+ asyncpg driver]
    end
    subgraph Docker [Docker container — port 5433]
      D[(PostgreSQL)]
    end
    O --> D
```

1. The **frontend** — what you see and click, running in your browser.
2. The **backend** — a Python server that receives requests, applies
   the rules, and talks to the database.
3. The **database** — PostgreSQL, a dedicated program whose only job is
   storing data safely. It runs inside **Docker**.

Follow one action all the way through — **adding an application**
(US-1):

1. You fill in a form in the browser and click **Add application**.
2. A **React component** collects what you typed.
3. It calls `fetch`, which sends an **HTTP POST request** containing
   JSON to `http://localhost:8000/applications`.
4. **FastAPI** receives the request and uses a **Pydantic schema** to
   check the JSON has the right fields and types.
5. FastAPI hands the data to **SQLAlchemy**, which turns a Python
   object into a SQL `INSERT` statement.
6. **asyncpg** carries that statement to **PostgreSQL**, without making
   the server sit idle while it waits.
7. PostgreSQL stores the row.
8. FastAPI sends back the saved row as JSON with status `201 Created`.
9. React adds it to the list on screen.

Every lesson builds one link in that chain. When you finish a lesson,
find its link in this list.

### Where the other tools fit

These don't appear in the diagram because they don't run *in* the app —
they help *you* build it.

| Tool | Job |
| --- | --- |
| **uv** | Installs Python and the backend's packages into an isolated folder (`.venv`) |
| **npm** | Installs the frontend's packages into `node_modules` |
| **Docker** | Runs PostgreSQL in a container, the same way on every machine |
| **Vite** | Runs the frontend dev server and reloads the page when you save |
| **Alembic** | Applies changes to the database's structure, in order, like commits |
| **pytest / Vitest** | Run your tests |
| **Husky** | Runs your tests before Git lets you commit |

---

## Test-driven development (TDD)

TDD flips the usual order. Instead of *write code, then check it*, you:

1. **🔴 Red** — Write a small test describing one thing the code should
   do. Run it. It **fails**, because the code doesn't exist yet.
2. **🟢 Green** — Write the *smallest* amount of code that makes the
   test pass. Not elegant, not complete — just passing.
3. **🔵 Refactor** — Clean up the code (names, duplication, structure).
   Run the tests again. They must stay green.

Then repeat with the next small thing.

### Why bother?

- **You always know if your code works.** The tests say so, not a
  person, not an AI.
- **Seeing red first proves the test can fail.** A test that has never
  failed might not be testing anything.
- **Small steps are easier to debug.** If a test breaks, the problem
  is in the last few lines you wrote.
- **You can change code without fear.** If you refactor and break
  something, a test turns red immediately.

Kent Beck, who popularized TDD, summarized the order of priorities as
"Make it work, make it right, make it fast." Red-green is *make it
work*. Refactor is *make it right*.

### What a test looks like

Every test has three parts, often called **Arrange, Act, Assert**:

```text
Arrange:  set up what the test needs
          (e.g. an empty test database)
Act:      do the one thing being tested
          (e.g. send a POST request)
Assert:   check the result
          (e.g. status code is 201)
```

You'll see this shape in every test in this repo.

### Tests are user stories, made checkable

A story says *what the user wants*. A test says *exactly how we'll know
it works*. US-1 becomes "a `POST /applications` with valid data returns
`201` and the saved row." When that test is green, US-1's backend half
is done — provably.

---

## Test coverage

**Coverage** measures which lines of your code ran during your tests.
If a line never ran, no test checked it.

- 100% coverage does *not* mean your code is correct.
- 0% coverage on a file *does* mean nothing checks it.

The **Coverage Gutters** extension paints each line in your editor
green (ran during tests) or red (didn't). You'll set it up in lessons
03 and 07.

---

## How the folders will look when you're done

```text
your-repo/
├── .husky/             ← Git hooks (pre-commit)
├── .vscode/            ← Editor settings and extensions
├── docker/             ← Database setup scripts
├── backend/
│   ├── app/            ← FastAPI app, models, schemas
│   ├── migrations/     ← Alembic migration files
│   ├── tests/          ← pytest tests
│   └── pyproject.toml  ← Python dependencies and settings
├── frontend/
│   ├── src/            ← React components and their tests
│   └── package.json    ← Frontend dependencies and scripts
├── guide/              ← These lessons
├── compose.yaml        ← Docker: how to run PostgreSQL
└── package.json        ← Repo-level scripts (Husky)
```

---

## Explain it back

Answer each in your own words *before* opening the answer.

**1. When you click "Add application", which program saves the data —
the frontend, the backend, or the database?**

<details>
<summary>Answer</summary>

The **database** (PostgreSQL) stores it; the **backend** decides to
store it. The frontend only collects your input and sends it over HTTP.
The frontend never touches the database directly.
</details>

**2. Which user story does PostgreSQL exist to serve, and why couldn't
the backend just keep the data in a Python list?**

<details>
<summary>Answer</summary>

US-5 — data must still be there tomorrow. A Python list lives in the
server's memory and vanishes whenever the server restarts. A database
writes to disk and survives restarts.
</details>

**3. Why do you run a test and watch it fail before writing the code?**

<details>
<summary>Answer</summary>

To prove the test can actually detect a problem. If a test passes
before the code exists, it's broken — it would pass no matter what.
</details>

**4. Your coverage report says 100%. Does that mean there are no bugs?**

<details>
<summary>Answer</summary>

No. Coverage only proves lines *ran* during tests. A line can run and
still do the wrong thing if no test checks its result carefully.
</details>

**5. Is SQLAlchemy a database? Is Docker?**

<details>
<summary>Answer</summary>

Neither. The database is PostgreSQL. SQLAlchemy is an ORM — a translator
between Python objects and SQL. Docker is a tool that *runs* PostgreSQL
in an isolated container.
</details>

---

## Checkpoint

You can sketch the request flow from browser to database from memory,
name the story each part serves, and explain red → green → refactor to
someone else.

Next: [01 — Install your tools](01-install-tools.md)
