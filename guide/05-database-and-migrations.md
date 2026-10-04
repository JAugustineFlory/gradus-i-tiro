# 05 — Async database layer and migrations

> **Where this fits.** Still **US-5** — *data that survives* — and the
> foundation for US-1 through US-4. Lesson 04 gave you a running
> PostgreSQL. This lesson connects the backend to it: a Python
> **model** that describes the `applications` table, an **async
> session** for talking to the database, and a **migration** that
> creates the real table. In the lesson 00 diagram, you're building the
> arrow from "SQLAlchemy models" to "PostgreSQL."

**Goal:** an `Application` model tested against the test database, and
an Alembic migration that creates the `applications` table in the
development database.

Make sure the database is running (repo root):
`docker compose up -d --wait`. Then work in `backend/` for this whole
lesson.

---

## Concepts first

### What's an ORM?

Databases speak **SQL**. To save an application in raw SQL you'd write:

```sql
INSERT INTO applications (company, role, status, applied_on)
VALUES ('Acme', 'Junior Developer', 'applied', '2026-10-01');
```

An **ORM** (Object-Relational Mapper) lets you write Python instead:

```python
db.add(Application(company="Acme", role="Junior Developer", ...))
```

and translates it into that SQL for you. **SQLAlchemy** is the ORM.

- A **model** is a Python class that represents one table.
- An **instance** of the class represents one row.
- An **attribute** on the class represents one column.

### Synchronous vs. asynchronous

This is the biggest new idea in the lesson, so take it slowly.

Picture a restaurant with **one waiter**.

- A **synchronous** waiter takes table 1's order, walks it to the
  kitchen, and **stands there** until the food is ready. Tables 2 and 3
  wait, even though the waiter is doing nothing but waiting.
- An **asynchronous** waiter takes table 1's order, hands it to the
  kitchen, and goes to take table 2's order **while the kitchen
  cooks**. When table 1's food is ready, they come back for it.

A web server is the waiter. Talking to a database is the kitchen: the
server sends a query, then *waits* — milliseconds, but an eternity for a
computer. An **async** server uses that waiting time to handle other
requests. That's why modern Python web servers, including FastAPI, are
built around it.

Python's syntax for this:

| Syntax | Meaning |
| --- | --- |
| `async def f():` | Defines a **coroutine function**: a function that's allowed to pause |
| `await something` | "Pause *this* function here until `something` finishes; let other work run meanwhile." Only allowed inside `async def` |
| **event loop** | The part of Python that keeps track of every paused function and resumes each one when what it was waiting for is ready. The waiter's to-do list |

> ⚠️ **The classic async bug:** forgetting `await`. Calling
> `db.commit()` *without* `await` doesn't commit — it creates a
> "coroutine" object and throws it away, and Python warns
> `coroutine ... was never awaited`. If something "does nothing,"
> look for a missing `await`.

**Rule of thumb for this project:** anything that talks to the database
is awaited.

### asyncpg

SQLAlchemy doesn't talk to PostgreSQL itself. It uses a **driver**: a
library that speaks PostgreSQL's network protocol. **asyncpg** is a fast
driver built for async code. You choose it in the **connection URL**:

```text
postgresql+asyncpg://tiro:tiro@localhost:5433/tiro
└───┬────┘ └──┬───┘   └┬─┘ └┬─┘ └───┬───┘ └┬─┘ └┬─┘
 database   driver   user  pass    host   port  database
 dialect                                         name
```

Every value in it comes from your `compose.yaml` (lesson 04).

### The SQLAlchemy async pieces

| Piece | Plain-English job |
| --- | --- |
| **Async engine** (`create_async_engine`) | Knows *where* the database is and keeps a small pool of open connections to it |
| **AsyncSession** | One conversation with the database: you add or change objects, then `await commit()` to save them all at once |
| **Session factory** (`async_sessionmaker`) | Makes a new `AsyncSession` each time you call it |
| **Base** | The parent class every model inherits from; it keeps a list (`Base.metadata`) of every table |
| **Model** | One class per table, like `Application` |

### What's a migration?

Your models describe what the tables *should* look like. The database
has what they *actually* look like. A **migration** is a small script
that changes the actual database to match — "create this table," "add
this column."

**Alembic** creates and runs migrations. Each migration has:

- **`upgrade()`** — apply the change
- **`downgrade()`** — undo it

Migrations run in order, like commits for your database structure. A new
teammate runs `alembic upgrade head` and gets exactly the tables you
have.

---

## Step 1 — Install the database packages

From `backend/`:

```bash
uv add "sqlalchemy[asyncio]" asyncpg alembic
```

| Package | What it adds |
| --- | --- |
| `sqlalchemy[asyncio]` | The ORM, plus its **`asyncio` extra** (a helper library called *greenlet* that SQLAlchemy's async support needs). Without the extra, async code fails with an error mentioning greenlet |
| `asyncpg` | The PostgreSQL driver |
| `alembic` | Migrations |

```bash
uv add --dev pytest-asyncio
```

**pytest-asyncio** lets pytest run `async def` tests and fixtures. Plain
pytest only knows how to call ordinary functions.

---

## Step 2 — Tell pytest about async

📄 **File:** `backend/pyproject.toml` — **edit**: in the
`[tool.pytest.ini_options]` section, add two lines below
`pythonpath = ["."]`. The section becomes:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
addopts = [
    "--cov=app",
    "--cov-report=term-missing",
    "--cov-report=xml",
]
```

- **`asyncio_mode = "auto"`** — any `async def` test or fixture is run
  automatically with an event loop. (Without it, you'd have to mark
  every async test by hand.)
- **`asyncio_default_fixture_loop_scope = "function"`** — every test gets
  a **fresh** event loop. Simplest and safest: nothing from one test can
  leak into the next.

---

## Step 3 — The database setup file

This file is **plumbing**, not behavior: it connects things. We write it
first so there's something to test models against.

📄 **File:** `backend/app/database.py` — **new**:

```python
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "postgresql+asyncpg://tiro:tiro@localhost:5433/tiro"

engine = create_async_engine(DATABASE_URL)

SessionLocal = async_sessionmaker(engine, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


async def get_db():
    async with SessionLocal() as session:
        yield session
```

Line by line:

- **Line 1** imports the two async building blocks.
- **Line 2** imports `DeclarativeBase`, the class `Base` is built from.
- **Line 4** is the **development** database's URL — the `tiro`
  database, on port 5433.
- **Line 6** creates the **engine**. Creating it doesn't connect yet;
  it connects the first time a query runs.
- **Line 8** creates the **session factory**. Calling `SessionLocal()`
  gives you a fresh session.
  - **`expire_on_commit=False`**: by default, after a commit SQLAlchemy
    "forgets" every object's values and reloads them from the database
    the next time you read them. In async code, that hidden reload
    can't happen automatically (it would need an `await` you didn't
    write), so you'd get an error. Turning it off keeps the values
    after a commit.
- **Lines 11–12** — `Base`. Every model will inherit from it.
- **Lines 15–17** — `get_db`, an **async generator**:
  - **`async with SessionLocal() as session:`** opens a session and
    guarantees it's **closed** when the block ends — even if an error
    happened.
  - **`yield session`** hands the session to whoever asked for it, and
    pauses. When they're done, the function resumes, the `async with`
    block ends, and the session closes.
  - FastAPI will use this in lesson 06 to give each request its own
    session.

---

## Step 4 — 🔴 Red: test the model

**Connect the dots.**

- We need a table named `applications` with columns: `id`, `company`,
  `role`, `status`, `applied_on`.
- `status` should default to `"applied"` when not given.
- Tests must **never** touch the development database. They use the
  **`tiro_test`** database you created in lesson 04 — same server, same
  port, different name.
- Each test should start with **empty tables**. The simplest way:
  create every table before each test, and drop them all after.
  `Base.metadata.create_all` creates every table `Base` knows about;
  `drop_all` removes them.

*Before reading on: what's the smallest test that proves "a new
application's status defaults to applied"?*

### Fixtures

A **fixture** is a function that prepares something a test needs. You
list it as a **parameter** of your test, and pytest runs the fixture
first and passes in what it `yield`s. Code after the `yield` runs after
the test — perfect for cleanup.

Fixtures that every test file needs live in a special file,
**`conftest.py`**. pytest loads it automatically; you never import it.

📄 **File:** `backend/tests/conftest.py` — **new**:

```python
import pytest
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine
from sqlalchemy.pool import NullPool

from app import models  # noqa: F401
from app.database import Base

TEST_DATABASE_URL = "postgresql+asyncpg://tiro:tiro@localhost:5433/tiro_test"


@pytest.fixture
async def engine():
    engine = create_async_engine(TEST_DATABASE_URL, poolclass=NullPool)
    async with engine.begin() as connection:
        await connection.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as connection:
        await connection.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest.fixture
async def session(engine):
    TestingSession = async_sessionmaker(engine, expire_on_commit=False)
    async with TestingSession() as db:
        yield db
```

Line by line:

- **Line 5** imports `models` only so that `Application` gets registered
  with `Base` — otherwise `create_all` wouldn't know the table exists.
  Nothing uses the name `models` directly, so **`# noqa: F401`** tells
  Ruff "this unused-looking import is on purpose."
- **Line 8** — the **test** database's URL. Compare it with line 4 of
  `database.py`: only the database name differs.
- **Lines 11–19** — the `engine` fixture:
  - **Line 13** creates an engine for the test database.
    **`NullPool`** means "don't keep connections open between uses."
    Each test runs in its own event loop, and a connection opened in
    one loop can't be reused in another.
  - **Lines 14–15** create the tables. `engine.begin()` opens a
    connection and a **transaction** that's committed when the block
    ends. Table creation is a regular (non-async) SQLAlchemy function,
    so **`run_sync`** runs it on the async connection.
  - **Line 16** hands the engine to the test.
  - **Lines 17–19** run *after* the test: drop every table, then
    **`dispose`** — close all connections cleanly.
- **Lines 22–26** — the `session` fixture asks for `engine` by naming
  it as a parameter (**fixtures can use other fixtures**), builds a
  session on it, and hands the session to the test.

📄 **File:** `backend/tests/test_models.py` — **new**:

```python
from datetime import date

from sqlalchemy import select

from app.models import Application


async def test_new_application_defaults_to_applied(session):
    application = Application(
        company="Acme",
        role="Junior Developer",
        applied_on=date(2026, 10, 1),
    )
    session.add(application)
    await session.commit()

    result = await session.scalars(select(Application))
    saved = result.one()

    assert saved.id is not None
    assert saved.company == "Acme"
    assert saved.status == "applied"
```

- **Line 8** — an **`async def`** test, because it awaits the database.
  `session` as a parameter tells pytest to run the fixture.
- **Lines 9–14** (*arrange* + *act*): build an object and **stage** it.
  `add` doesn't need `await` — it only notes the object in the session;
  nothing goes to the database yet.
- **Line 15** — `await session.commit()` sends the `INSERT` and saves
  it. *This* talks to the database, so it's awaited.
- **Line 17** — `select(Application)` builds a query for all rows;
  `await session.scalars(...)` runs it and returns model objects.
- **Line 18** — `.one()` insists on exactly one row (and fails the test
  otherwise).
- **Lines 20–22** — the assertions. We never set `id` or `status` — the
  database and the model's default should fill them in.

Run it (from `backend/`):

```bash
uv run pytest
```

🔴 **Expected:** `ModuleNotFoundError: No module named 'app.models'`.

---

## Step 5 — 🟢 Green: write the model

📄 **File:** `backend/app/models.py` — **new**:

```python
from datetime import date

from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from app.database import Base


class Application(Base):
    __tablename__ = "applications"

    id: Mapped[int] = mapped_column(primary_key=True)
    company: Mapped[str] = mapped_column(String(100))
    role: Mapped[str] = mapped_column(String(100))
    status: Mapped[str] = mapped_column(
        String(20),
        default="applied",
    )
    applied_on: Mapped[date]
```

- **Line 9** — inheriting from `Base` registers the table.
- **Line 10** — `__tablename__` is the table's name in the database.
- **`Mapped[int]`** is a **type hint**: "this column holds an `int`."
  SQLAlchemy reads it to pick the SQL column type.
- **Line 12** — `primary_key=True` makes `id` the unique identifier for
  each row. PostgreSQL assigns it automatically: 1, 2, 3… (the same
  idea as `SERIAL` in lesson 04's experiment).
- **`String(100)`** — text up to 100 characters. PostgreSQL **enforces**
  this: a longer value is an error, not silently stored.
- **Lines 15–18** — `default="applied"` fills in the status when a new
  row is inserted without one.
- **Line 19** — no `mapped_column` needed: `Mapped[date]` alone is
  enough for SQLAlchemy to make a `DATE` column.

```bash
uv run pytest
```

🟢 **Expected:** `2 passed` (the health test and the model test).

Look at the coverage table: `app/database.py` has **missing** lines — 16
and 17, the body of `get_db`. No test calls it yet. You'll fix that in
lesson 06. Open `app/database.py` with Coverage Gutters watching to see
them in red.

> **If instead you get `ConnectionRefusedError`** (or "connection
> refused" on port 5433): the database isn't running. From the repo
> root: `docker compose up -d --wait`.

---

## Step 6 — Set up Alembic (async template)

From `backend/`:

```bash
uv run alembic init -t async migrations
```

- **`-t async`** chooses Alembic's **async template**, whose generated
  code knows how to run migrations through an async engine and asyncpg.

✅ Creates:

- `alembic.ini` — Alembic's settings
- `migrations/env.py` — the script Alembic runs to connect to your
  database and find your models
- `migrations/versions/` — where migration files will go (empty)

### Tell Alembic where the database is

📄 **File:** `backend/alembic.ini` — **edit**: find the line starting
with `sqlalchemy.url` (use `Ctrl+F`). Replace that whole line with:

```ini
sqlalchemy.url = postgresql+asyncpg://tiro:tiro@localhost:5433/tiro
```

This is the **development** database — the same URL as line 4 of
`app/database.py`.

> Having the URL in two places is duplication you'll remove in GRADUS
> II, by reading it from one setting.

### Tell Alembic about your models

📄 **File:** `backend/migrations/env.py` — **edit**, two changes:

**1.** Near the top, find the line `from alembic import context`.
Directly **below** it, add:

```python
from app import models  # noqa: F401
from app.database import Base
```

**2.** Find this line (use `Ctrl+F`):

```python
target_metadata = None
```

and change it to:

```python
target_metadata = Base.metadata
```

- **`target_metadata`** is how Alembic learns what your tables *should*
  look like, so it can compare them to the actual database.
- **`from app import models`** isn't used directly, but importing it
  runs `models.py`, which registers `Application` with `Base`. Without
  it, `Base.metadata` would be empty.

> **How can `env.py` import `app`?** `alembic.ini` contains
> `prepend_sys_path = .`, which adds the `backend/` folder to Python's
> import path when Alembic runs.

### Keep Ruff out of generated files

Alembic writes its own files in its own style.

📄 **File:** `backend/pyproject.toml` — **edit**: in the `[tool.ruff]`
section, add one line so it reads:

```toml
[tool.ruff]
line-length = 88
extend-exclude = ["migrations"]
```

---

## Step 7 — Generate and apply the first migration

**Autogenerate** compares your models to the database and writes a
migration for the difference. From `backend/`:

```bash
uv run alembic revision --autogenerate -m "create applications table"
```

✅ Output ends with something like:

```text
INFO  [alembic.autogenerate.compare] Detected added table 'applications'
  Generating .../migrations/versions/xxxx_create_applications_table.py ...  done
```

**Always read a generated migration before running it.** Open the new
file in `backend/migrations/versions/`. You'll find:

- `upgrade()` calling `op.create_table('applications', ...)` with your
  five columns — notice `sa.String(length=100)`, `sa.Date()`, and the
  primary key
- `downgrade()` calling `op.drop_table('applications')`

Autogenerate is a helper, not an oracle. It can miss things (like
renamed columns, which it sees as "drop one, add another"). Reading the
file is your job.

Apply it:

```bash
uv run alembic upgrade head
```

- **`head`** means "the newest migration."

### Look inside

From the **repo root**:

```bash
docker compose exec db psql -U tiro -d tiro
```

Then:

```text
\dt
\d applications
```

✅ `\dt` lists two tables:

- **`applications`** — your table
- **`alembic_version`** — one row holding the ID of the last migration
  applied. That's how Alembic knows where you are.

✅ `\d applications` shows your five columns with their PostgreSQL
types: `integer`, `character varying(100)`, `character varying(20)`,
and `date`.

Now `\c tiro_test` and `\dt` — ✅ **no tables**. The test database only
has tables *while a test is running*: the `engine` fixture creates them
and drops them again. Quit with `\q`.

### Try undoing it

From `backend/`:

```bash
uv run alembic downgrade -1
```

In `psql`, `\dt` — `applications` is gone. Bring it back:

```bash
uv run alembic upgrade head
```

| Command (prefix each with `uv run`) | Does |
| --- | --- |
| `alembic current` | Show which migration the database is at |
| `alembic history` | List all migrations |
| `alembic upgrade head` | Apply everything not yet applied |
| `alembic downgrade -1` | Undo the last migration |

---

## Step 8 — Let the hook start the database

Your tests now need PostgreSQL running. If someone commits with Docker
stopped, the hook should start the database rather than fail
mysteriously.

📄 **File:** `.husky/pre-commit` (repo root) — **edit**: replace the
whole file with:

```sh
echo "Starting the database..."
docker compose up -d --wait

echo "Running backend tests..."
(cd backend && uv run pytest -q)
```

If the database is already running, `docker compose up -d --wait`
returns almost instantly. Docker Desktop itself must be running.

---

## Step 9 — Commit

```bash
cd ..
uv --directory backend run ruff check .
git add .
git commit -m "feat(backend): async database layer, Application model, first migration"
git push
```

(`uv --directory backend run …` runs a command as if you were in
`backend/` — handy from the root.)

✅ The hook starts (or finds) the database and shows `2 passed`. The
migration file is committed — that's how a teammate builds the same
table.

---

## Explain it back

**1. In plain words: what does `await` do, and what goes wrong if you
forget it?**

<details>
<summary>Answer</summary>

`await` pauses the current function until the operation (like a
database query) finishes, letting the server do other work meanwhile.
Forget it, and the operation never actually runs — you get a coroutine
object instead of a result, plus a "never awaited" warning.
</details>

**2. The tests use `tiro_test` while the app uses `tiro`. Why?**

<details>
<summary>Answer</summary>

Tests create and drop tables constantly. A separate database means they
can never damage data in the development database, and leftover
development data can never affect a test.
</details>

**3. Your tests create tables with `Base.metadata.create_all`. The real
database uses Alembic. Why not use `create_all` for the real one too?**

<details>
<summary>Answer</summary>

`create_all` only creates tables that don't exist; it can't change an
existing table (add a column, rename one) and keeps no history.
Migrations record every change in order, can be undone, and let every
teammate reach the exact same structure.
</details>

**4. What would go wrong if `env.py` didn't import `app.models`?**

<details>
<summary>Answer</summary>

`Base.metadata` would be empty, so autogenerate would think you have no
tables — and it might even generate a migration that drops them.
</details>

**5. Why `expire_on_commit=False`?**

<details>
<summary>Answer</summary>

By default, objects "forget" their values after a commit and reload
them on the next read. In async code that reload can't happen
automatically, so reading a value after a commit would raise an error.
Keeping the values avoids that.
</details>

---

## Checkpoint

- ✅ `uv run pytest` → `2 passed`
- ✅ `\d applications` in `psql` (database `tiro`) shows your columns
- ✅ `uv run alembic current` shows your migration's ID with `(head)`
- ✅ Committing starts the database if needed and runs the tests

Docs for going deeper:

- Python `async`/`await` (the official tutorial):
  <https://docs.python.org/3/library/asyncio-task.html>
- FastAPI's friendly explanation of async:
  <https://fastapi.tiangolo.com/async/>
- SQLAlchemy asyncio:
  <https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html>
- SQLAlchemy ORM quick start:
  <https://docs.sqlalchemy.org/en/20/orm/quickstart.html>
- Alembic tutorial:
  <https://alembic.sqlalchemy.org/en/latest/tutorial.html>
- pytest fixtures:
  <https://docs.pytest.org/en/stable/how-to/fixtures.html>
- pytest-asyncio: <https://pytest-asyncio.readthedocs.io/>

Next: [06 — CRUD endpoints with TDD](06-crud-endpoints.md)
