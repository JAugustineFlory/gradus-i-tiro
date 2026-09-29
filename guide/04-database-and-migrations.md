# 04 — Database and migrations

**Goal:** an `Application` model tested against a throwaway database,
and an Alembic migration that creates the real `applications` table in
`tiro.db`.

Work in `backend/` for this whole lesson.

---

## The concepts first

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

### The four SQLAlchemy pieces you'll use

| Piece | Plain-English job |
| --- | --- |
| **Engine** | Knows *where* the database is and how to connect to it |
| **Session** | A conversation with the database: you add/change objects, then `commit` to save them all at once |
| **Base** | The parent class every model inherits from; it keeps a list (`Base.metadata`) of every table |
| **Model** | One class per table, like `Application` |

### What's a migration?

Your models describe what the tables *should* look like. The database
file has what they *actually* look like. A **migration** is a small
script that changes the actual database to match — "create this table,"
"add this column."

**Alembic** creates and runs migrations. Each migration has:

- **`upgrade()`** — apply the change
- **`downgrade()`** — undo it

Migrations run in order, like commits for your database structure. A new
teammate runs `alembic upgrade head` and gets the exact same tables you
have.

---

## Step 1 — Install SQLAlchemy and Alembic

```bash
uv add sqlalchemy alembic
```

✅ Both appear in `dependencies` in `pyproject.toml`.

---

## Step 2 — The database setup file

This file is **plumbing**, not behavior — it connects things. We write it
first so there's something to test models against.

Create `backend/app/database.py`:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

DATABASE_URL = "sqlite:///./tiro.db"

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False},
)

SessionLocal = sessionmaker(bind=engine, autoflush=False)


class Base(DeclarativeBase):
    pass


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

Line by line:

- **Line 4:** the **connection URL**. `sqlite:///` means "SQLite file";
  `./tiro.db` means "a file named `tiro.db` in the current folder."
- **Lines 6–9:** create the **engine**. `check_same_thread=False` is a
  SQLite-only setting: FastAPI may handle a request on a different
  thread than the one that opened the connection, and SQLite blocks that
  by default.
- **Line 11:** `sessionmaker` builds a **factory** — calling
  `SessionLocal()` gives you a new session. `autoflush=False` means
  nothing is sent to the database until you ask.
- **Lines 14–15:** `Base`. Every model will inherit from it.
- **Lines 18–23:** `get_db` opens a session, **`yield`s** it (hands it to
  whoever asked, then pauses), and when they're done, the `finally`
  block closes it — even if an error happened. FastAPI will use this in
  lesson 05.

---

## Step 3 — 🔴 Red: test the model

**Connect the dots.**

- We need a table named `applications` with columns: `id`, `company`,
  `role`, `status`, `applied_on`.
- `status` should default to `"applied"` when not given.
- To test without touching `tiro.db`, we'll use an **in-memory
  database**: the URL `sqlite://` (nothing after the slashes) creates a
  database that lives only in RAM and vanishes when the test ends.
- `Base.metadata.create_all(engine)` creates every table `Base` knows
  about.

*Before reading on: what's the smallest test that proves "a new
application's status defaults to applied"?*

### Fixtures

A **fixture** is a function that prepares something a test needs. You
list it as a parameter of your test, and pytest runs the fixture first
and passes in what it `yield`s. Code after the `yield` runs after the
test — perfect for cleanup.

Create `backend/tests/test_models.py`:

```python
from datetime import date

import pytest
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from app.database import Base
from app.models import Application


@pytest.fixture
def session():
    engine = create_engine("sqlite://")
    Base.metadata.create_all(engine)
    with Session(engine) as db:
        yield db
    Base.metadata.drop_all(engine)


def test_new_application_defaults_to_applied(session):
    application = Application(
        company="Acme",
        role="Junior Developer",
        applied_on=date(2026, 10, 1),
    )
    session.add(application)
    session.commit()

    saved = session.scalars(select(Application)).one()

    assert saved.id is not None
    assert saved.company == "Acme"
    assert saved.status == "applied"
```

- **Lines 11–17:** the fixture. Fresh database → create tables → hand
  over a session → drop tables afterward.
- **Line 20:** `session` as a parameter tells pytest to run the fixture.
- **Lines 21–27** (arrange + act): build an object and save it. `add`
  stages it; `commit` writes it.
- **Line 29:** `select(Application)` builds a query for all rows.
  `scalars(...)` runs it and returns model objects. `.one()` insists on
  exactly one row.
- **Lines 31–33:** the assertions. We never set `id` or `status` — the
  database and the model's default should fill them in.

```bash
uv run pytest
```

🔴 **Expected:** `ModuleNotFoundError: No module named 'app.models'`.

---

## Step 4 — 🟢 Green: write the model

Create `backend/app/models.py`:

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

- **Line 9:** inheriting from `Base` registers the table.
- **Line 10:** `__tablename__` is the table's name in the database.
- **`Mapped[int]`** is a **type hint**: "this column holds an `int`."
  SQLAlchemy reads it to pick the SQL column type.
- **Line 12:** `primary_key=True` makes `id` the unique identifier for
  each row. The database assigns it automatically: 1, 2, 3…
- **`String(100)`** limits text to 100 characters.
- **Lines 15–18:** `default="applied"` fills in the status on insert
  when none is given.
- **Line 19:** no `mapped_column` needed — `Mapped[date]` is enough for
  SQLAlchemy to make a date column.

```bash
uv run pytest
```

🟢 **Expected:** `2 passed`.

Look at the coverage table: `app/database.py` shows some **missing**
lines — the body of `get_db`. No test calls it yet. You'll fix that in
lesson 05. Open `app/database.py` with Coverage Gutters watching to see
them in red.

---

## Step 5 — Set up Alembic

```bash
uv run alembic init migrations
```

✅ Creates:

- `alembic.ini` — Alembic's settings
- `migrations/env.py` — the script Alembic runs to connect to your
  database and find your models
- `migrations/versions/` — where migration files will go (empty)

### Tell Alembic where the database is

Open `backend/alembic.ini`. Find the line starting with
`sqlalchemy.url` (use `Ctrl+F`). Replace it with:

```ini
sqlalchemy.url = sqlite:///./tiro.db
```

> This duplicates `DATABASE_URL` from `app/database.py`. Keeping one
> source of truth is something you'll fix in GRADUS II.

### Tell Alembic about your models

Open `backend/migrations/env.py`. Near the top, after the existing
imports, add:

```python
from app import models  # noqa: F401
from app.database import Base
```

Then find this line (use `Ctrl+F` — it's around line 21):

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
- **`# noqa: F401`** tells Ruff "I know this import looks unused; it's on
  purpose."

> **How can `env.py` import `app`?** `alembic.ini` contains
> `prepend_sys_path = .`, which adds the `backend/` folder to Python's
> import path when Alembic runs.

### Keep Ruff out of generated files

Alembic writes its own files in its own style. Add one line to the
`[tool.ruff]` section of `pyproject.toml`:

```toml
[tool.ruff]
line-length = 88
extend-exclude = ["migrations"]
```

---

## Step 6 — Generate and apply the first migration

**Autogenerate** compares your models to the database and writes a
migration for the difference:

```bash
uv run alembic revision --autogenerate -m "create applications table"
```

✅ Output ends with something like:

```text
Detected added table 'applications'
Generating .../migrations/versions/xxxx_create_applications_table.py ...  done
```

**Always read a generated migration before running it.** Open the new
file in `migrations/versions/`. You'll find:

- `upgrade()` calling `op.create_table('applications', ...)` with your
  five columns
- `downgrade()` calling `op.drop_table('applications')`

Autogenerate is a helper, not an oracle. It can miss things (like
renamed columns, which it sees as "drop one, add another"). Reading the
file is your job.

Apply it:

```bash
uv run alembic upgrade head
```

- **`head`** means "the newest migration."

✅ A `tiro.db` file appears in `backend/`.

### Look inside

Click `backend/tiro.db` in VS Code. SQLite Viewer shows two tables:

- **`applications`** — your table, with its five columns and no rows
- **`alembic_version`** — one row holding the ID of the last migration
  applied. That's how Alembic knows where you are.

### Try undoing it

```bash
uv run alembic downgrade -1
```

Refresh the viewer — `applications` is gone. Bring it back:

```bash
uv run alembic upgrade head
```

Useful commands:

| Command | Does |
| --- | --- |
| `alembic current` | Show which migration the database is at |
| `alembic history` | List all migrations |
| `alembic upgrade head` | Apply everything not yet applied |
| `alembic downgrade -1` | Undo the last migration |

(Prefix each with `uv run`.)

---

## Step 7 — Commit

```bash
uv run ruff check .
cd ..
git add .
git commit -m "feat(backend): add Application model and first migration"
git push
```

✅ The hook runs `2 passed`. `tiro.db` is **not** committed (it's
ignored), but the migration file **is** — that's how teammates build
the same table.

---

## Explain it back

**1. The test uses `sqlite://` but the app uses `sqlite:///./tiro.db`.
Why use a different database in tests?**

<details>
<summary>Answer</summary>

The in-memory database starts empty for every test and disappears
afterward. Tests can't be affected by leftover data, and they never
damage your real data.
</details>

**2. Your test creates tables with `Base.metadata.create_all`. The real
database uses Alembic. Why not use `create_all` for the real one too?**

<details>
<summary>Answer</summary>

`create_all` only creates tables that don't exist; it can't change an
existing table (add a column, rename one) and has no history. Migrations
record every change in order, can be undone, and let every teammate
reach the exact same structure.
</details>

**3. What would go wrong if `env.py` didn't import `app.models`?**

<details>
<summary>Answer</summary>

`Base.metadata` would be empty, so autogenerate would think you have no
tables — and it might even generate a migration that drops them.
</details>

**4. What does `yield` do in the fixture and in `get_db`?**

<details>
<summary>Answer</summary>

It hands a value to the caller and pauses the function. When the caller
is done, the function resumes after the `yield` and runs its cleanup
(dropping tables, closing the session).
</details>

---

## Checkpoint

- ✅ `uv run pytest` → `2 passed`
- ✅ `tiro.db` exists and contains an empty `applications` table
- ✅ `uv run alembic current` shows your migration's ID with `(head)`

Docs for going deeper:

- SQLAlchemy ORM quick start:
  <https://docs.sqlalchemy.org/en/20/orm/quickstart.html>
- Alembic tutorial:
  <https://alembic.sqlalchemy.org/en/latest/tutorial.html>
- Alembic autogenerate:
  <https://alembic.sqlalchemy.org/en/latest/autogenerate.html>
- pytest fixtures:
  <https://docs.pytest.org/en/stable/how-to/fixtures.html>

Next: [05 — CRUD endpoints with TDD](05-crud-endpoints.md)
