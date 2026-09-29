# 05 — CRUD endpoints with TDD

**Goal:** five fully tested endpoints that **C**reate, **R**ead,
**U**pdate, and **D**elete applications, built one red → green cycle at
a time.

| Method | Path | Does | Success code |
| --- | --- | --- | --- |
| `POST` | `/applications` | Create one | `201 Created` |
| `GET` | `/applications` | List all | `200 OK` |
| `GET` | `/applications/{id}` | Get one | `200 OK` |
| `PATCH` | `/applications/{id}` | Change some fields | `200 OK` |
| `DELETE` | `/applications/{id}` | Delete one | `204 No Content` |

Work in `backend/` for this whole lesson. This is the longest backend
lesson — take a break between cycles if you need one.

---

## The concepts first

### HTTP methods and status codes

An HTTP request has a **method** (what to do) and a **path** (what to do
it to). The response has a **status code** (what happened).

| Code | Name | When you'll see it |
| --- | --- | --- |
| `200` | OK | A request worked and returns data |
| `201` | Created | A new thing was created |
| `204` | No Content | It worked, and there's nothing to send back |
| `404` | Not Found | The path or the item doesn't exist |
| `405` | Method Not Allowed | The path exists, but not for that method |
| `422` | Unprocessable Content | The data you sent is the wrong shape |

Full list: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Status>

### Pydantic schemas

A **schema** describes the shape of data crossing the API boundary.
FastAPI uses **Pydantic** schemas to:

- **validate input** — reject a request missing `company`, or with
  `applied_on: "banana"`, before your code even runs (that's the `422`)
- **shape output** — control exactly which fields go back to the client

Why not use the SQLAlchemy model directly? Because what the client
*sends* and what the database *stores* differ. The client never sends
an `id`; the database always has one. Separate schemas keep those
straight.

### Dependency injection

Every endpoint needs a database session. Instead of each endpoint
opening its own, you **declare** that it needs one:

```python
db: Annotated[Session, Depends(get_db)]
```

This says "`db` is a `Session`, and FastAPI should get it by calling
`get_db`." FastAPI calls `get_db` for each request, passes in the
session, and closes it afterward (the code after `yield`).

The payoff comes in testing: you can tell FastAPI "whenever something
asks for `get_db`, use *this other function* instead" — and point it at
the in-memory test database. That's a **dependency override**.

---

## Step 1 — Shared test fixtures in `conftest.py`

**Connect the dots.**

- Model tests (lesson 04) need a **session** on a test database.
- API tests need a **client** whose requests use the *same* test
  database.
- Both should start from empty tables every test.
- pytest automatically loads fixtures from a file named
  **`conftest.py`** and shares them with every test in the folder.

*Before reading on: which fixture should create the tables — the session
fixture, the client fixture, or a third one both depend on?*

Create `backend/tests/conftest.py`:

```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker
from sqlalchemy.pool import StaticPool

from app import models  # noqa: F401
from app.database import Base, get_db
from app.main import app


@pytest.fixture
def engine():
    engine = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)


@pytest.fixture
def session(engine):
    with Session(engine) as db:
        yield db


@pytest.fixture
def client(engine):
    TestingSession = sessionmaker(
        bind=engine,
        autoflush=False,
    )

    def override_get_db():
        db = TestingSession()
        try:
            yield db
        finally:
            db.close()

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as test_client:
        yield test_client
    app.dependency_overrides.clear()
```

- **Line 7** registers the models with `Base`, so `create_all` knows
  about the `applications` table.
- **Lines 12–21:** a third fixture, `engine`, owns the database. Both
  `session` (line 25) and `client` (line 31) ask for it by name —
  **fixtures can use other fixtures**.
- **Line 16, `check_same_thread`:** `TestClient` runs your app on a
  different thread; SQLite needs permission for that.
- **Line 17, `StaticPool`:** an in-memory SQLite database exists only
  inside one connection. `StaticPool` makes every session reuse that
  *one* connection, so the test and the API see the same data.
- **Line 44** is the dependency override. **Line 47** removes it after
  the test so it can't leak into other tests.

Now open `backend/tests/test_models.py` and **delete** its `session`
fixture (the whole `@pytest.fixture` function), plus the imports it
used: `pytest`, `create_engine`, and `Base`. The test now gets `session`
from `conftest.py`. The top of the file should be:

```python
from datetime import date

from sqlalchemy import select

from app.models import Application
```

```bash
uv run pytest
```

✅ Still `2 passed`. You moved code without changing behavior — that's
a **refactor**, and the green tests prove it.

---

## Step 2 — Schemas

Create `backend/app/schemas.py`:

```python
from datetime import date

from pydantic import BaseModel, ConfigDict


class ApplicationCreate(BaseModel):
    company: str
    role: str
    status: str = "applied"
    applied_on: date


class ApplicationUpdate(BaseModel):
    company: str | None = None
    role: str | None = None
    status: str | None = None
    applied_on: date | None = None


class ApplicationRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    company: str
    role: str
    status: str
    applied_on: date
```

| Schema | Used for | Notice |
| --- | --- | --- |
| `ApplicationCreate` | Body of `POST` | No `id` — the database assigns it. `status` is optional with a default. |
| `ApplicationUpdate` | Body of `PATCH` | *Every* field optional — you might only change `status`. `str \| None` means "a string or nothing." |
| `ApplicationRead` | Every response | Includes `id`. **`from_attributes=True`** lets Pydantic read a SQLAlchemy object's attributes directly. |

---

## Cycle 1 — Create an application

### 🔴 Red

Create `backend/tests/test_applications.py`:

```python
SAMPLE = {
    "company": "Acme",
    "role": "Junior Developer",
    "applied_on": "2026-10-01",
}


def test_create_application_returns_201_and_the_row(client):
    response = client.post("/applications", json=SAMPLE)

    assert response.status_code == 201
    body = response.json()
    assert body["id"] == 1
    assert body["company"] == "Acme"
    assert body["status"] == "applied"
    assert body["applied_on"] == "2026-10-01"
```

- **`json=SAMPLE`** sends the dictionary as a JSON body. Dates travel as
  text in `YYYY-MM-DD` format.
- **Line 14** checks the default status was applied, even though we
  never sent one.

```bash
uv run pytest
```

🔴 **Expected:** `assert 404 == 201`. The route doesn't exist.

### 🟢 Green

Replace `backend/app/main.py` with:

```python
from typing import Annotated

from fastapi import Depends, FastAPI
from sqlalchemy.orm import Session

from app import models, schemas
from app.database import get_db

app = FastAPI(title="Tiro Job Tracker")


@app.get("/health")
def health():
    return {"status": "ok"}


@app.post(
    "/applications",
    response_model=schemas.ApplicationRead,
    status_code=201,
)
def create_application(
    payload: schemas.ApplicationCreate,
    db: Annotated[Session, Depends(get_db)],
):
    application = models.Application(**payload.model_dump())
    db.add(application)
    db.commit()
    db.refresh(application)
    return application
```

- **Lines 17–21:** `response_model` tells FastAPI to shape the return
  value with `ApplicationRead`. `status_code=201` replaces the default
  `200`.
- **Line 23:** a parameter typed as a Pydantic schema means "read this
  from the JSON body and validate it."
- **Line 24:** the injected database session.
- **Line 26:** `payload.model_dump()` turns the schema into a
  dictionary: `{"company": "Acme", "role": ..., ...}`. The **`**`**
  *unpacks* it into keyword arguments, so this equals
  `models.Application(company="Acme", role=..., ...)`.
- **Line 29:** `refresh` reloads the object from the database so it has
  its new `id`.

```bash
uv run pytest
```

🟢 **Expected:** `3 passed`.

```bash
cd ..
git add .
git commit -m "feat(backend): create applications"
cd backend
```

---

## Cycle 2 — Reject bad input

### 🔴 Red?

Add to `test_applications.py`:

```python
def test_create_without_company_returns_422(client):
    payload = {
        "role": "Junior Developer",
        "applied_on": "2026-10-01",
    }

    response = client.post("/applications", json=payload)

    assert response.status_code == 422
```

```bash
uv run pytest
```

🟢 **It passes immediately.** Pydantic already rejects the missing field.

This happens in real TDD: sometimes the framework already does what you
want. But a test that has never failed hasn't proven anything. So
**prove it can fail**:

1. In `app/schemas.py`, temporarily change `company: str` in
   `ApplicationCreate` (line 7) to `company: str = "Unknown"`.
2. Run `uv run pytest`. 🔴 The new test fails with `assert 201 == 422`.
3. Change it back. 🟢 Green again.

Now you *know* this test guards against someone making `company`
optional by accident.

---

## Cycle 3 — List applications

### 🔴 Red

First, a small helper so tests can create data in one line. Add this
**below `SAMPLE`** in `test_applications.py`:

```python
def create_sample(client, **overrides):
    payload = {**SAMPLE, **overrides}
    response = client.post("/applications", json=payload)
    return response.json()
```

- **`**overrides`** collects any extra keyword arguments into a
  dictionary. `create_sample(client, company="Globex")` gives
  `overrides = {"company": "Globex"}`.
- **`{**SAMPLE, **overrides}`** merges two dictionaries; later values
  win. So you get the sample data with `company` swapped.

Now the tests, at the bottom of the file:

```python
def test_list_applications_starts_empty(client):
    response = client.get("/applications")

    assert response.status_code == 200
    assert response.json() == []


def test_list_returns_every_application_in_order(client):
    create_sample(client, company="Acme")
    create_sample(client, company="Globex")

    response = client.get("/applications")

    companies = [item["company"] for item in response.json()]
    assert companies == ["Acme", "Globex"]
```

**Line 14** is a **list comprehension**: "for each item in the response,
take its `company`." It builds `["Acme", "Globex"]`.

```bash
uv run pytest
```

🔴 **Expected:** `assert 405 == 200`. **Not 404!** The path
`/applications` exists — for `POST`. It just doesn't accept `GET`. That's
`405 Method Not Allowed`.

### 🟢 Green

In `app/main.py`, add `select` to the SQLAlchemy imports (line 4):

```python
from sqlalchemy import select
from sqlalchemy.orm import Session
```

Add the route at the bottom of the file:

```python
@app.get(
    "/applications",
    response_model=list[schemas.ApplicationRead],
)
def list_applications(db: Annotated[Session, Depends(get_db)]):
    query = select(models.Application).order_by(
        models.Application.id
    )
    return db.scalars(query).all()
```

- **`list[schemas.ApplicationRead]`** — the response is a *list* of
  applications.
- **`order_by(...id)`**: databases don't promise any order unless you
  ask. Without this, the test might pass today and fail tomorrow.
- **`.all()`** returns every row as a list.

```bash
uv run pytest
```

🟢 **Expected:** `6 passed`.

```bash
cd ..
git add .
git commit -m "feat(backend): list applications"
cd backend
```

---

## Cycle 4 — Get one application

### 🔴 Red

```python
def test_get_application_by_id(client):
    created = create_sample(client)

    response = client.get(f"/applications/{created['id']}")

    assert response.status_code == 200
    assert response.json() == created


def test_get_missing_application_returns_404(client):
    response = client.get("/applications/999")

    assert response.status_code == 404
    assert response.json() == {"detail": "Application not found"}
```

- **`f"..."`** is an **f-string**: `{created['id']}` is replaced with the
  value, giving `/applications/1`.
- The second test checks the *unhappy path* — asking for something that
  doesn't exist. Always test both.

🔴 **Expected:** both fail (`405`).

### 🟢 Green

Add `HTTPException` to the FastAPI import (line 3):

```python
from fastapi import Depends, FastAPI, HTTPException
```

Add the route:

```python
@app.get(
    "/applications/{application_id}",
    response_model=schemas.ApplicationRead,
)
def get_application(
    application_id: int,
    db: Annotated[Session, Depends(get_db)],
):
    application = db.get(models.Application, application_id)
    if application is None:
        raise HTTPException(
            status_code=404,
            detail="Application not found",
        )
    return application
```

- **`{application_id}`** in the path is a **path parameter**. A function
  parameter with the same name receives it. The `int` type means FastAPI
  converts `"1"` to `1` — and returns `422` for `/applications/abc`.
- **`db.get(Model, id)`** looks up one row by primary key and returns
  `None` if there isn't one.
- **`raise HTTPException`** stops the function and sends an error
  response.

🟢 **Expected:** `8 passed`. Commit:

```bash
cd ..
git add .
git commit -m "feat(backend): get one application"
cd backend
```

---

## Cycle 5 — Update an application

### 🔴 Red

```python
def test_update_changes_only_the_fields_sent(client):
    created = create_sample(client)

    response = client.patch(
        f"/applications/{created['id']}",
        json={"status": "interviewing"},
    )

    assert response.status_code == 200
    body = response.json()
    assert body["status"] == "interviewing"
    assert body["company"] == "Acme"


def test_update_missing_application_returns_404(client):
    response = client.patch(
        "/applications/999",
        json={"status": "offer"},
    )

    assert response.status_code == 404
```

**Line 12** matters: it proves `PATCH` didn't wipe out the fields we
*didn't* send.

🔴 **Expected:** both fail (`405`).

### 🟢 Green

**Connect the dots.** The request body only contains the fields the
client wants to change. Pydantic fills every *missing* field with its
default, `None`. If we copied every field, we'd set `company` to `None`!
We need only the fields the client **actually sent**.
`model_dump(exclude_unset=True)` does exactly that.

```python
@app.patch(
    "/applications/{application_id}",
    response_model=schemas.ApplicationRead,
)
def update_application(
    application_id: int,
    payload: schemas.ApplicationUpdate,
    db: Annotated[Session, Depends(get_db)],
):
    application = db.get(models.Application, application_id)
    if application is None:
        raise HTTPException(
            status_code=404,
            detail="Application not found",
        )
    changes = payload.model_dump(exclude_unset=True)
    for field, value in changes.items():
        setattr(application, field, value)
    db.commit()
    db.refresh(application)
    return application
```

- **`changes.items()`** gives each `(field, value)` pair. For our test,
  just `("status", "interviewing")`.
- **`setattr(obj, "status", "interviewing")`** is the same as writing
  `obj.status = "interviewing"`, but lets the field name come from a
  variable.

**Try it:** remove `exclude_unset=True`, run the tests, and read the
failure. Then put it back.

🟢 **Expected:** `10 passed`. Commit with message
`feat(backend): update applications`.

---

## Cycle 6 — Delete an application

### 🔴 Red

```python
def test_delete_removes_the_application(client):
    created = create_sample(client)

    response = client.delete(f"/applications/{created['id']}")

    assert response.status_code == 204
    follow_up = client.get(f"/applications/{created['id']}")
    assert follow_up.status_code == 404


def test_delete_missing_application_returns_404(client):
    response = client.delete("/applications/999")

    assert response.status_code == 404
```

**Lines 7–8** check the result, not just the status code: after
deleting, the application really is gone.

🔴 **Expected:** both fail (`405`).

### 🟢 Green

```python
@app.delete(
    "/applications/{application_id}",
    status_code=204,
)
def delete_application(
    application_id: int,
    db: Annotated[Session, Depends(get_db)],
):
    application = db.get(models.Application, application_id)
    if application is None:
        raise HTTPException(
            status_code=404,
            detail="Application not found",
        )
    db.delete(application)
    db.commit()
```

No `return` — a `204` response has no body.

🟢 **Expected:** `12 passed`. Commit with message
`feat(backend): delete applications`.

---

## 🔵 Refactor — remove the duplication

Look at `get_application`, `update_application`, and
`delete_application`. The same six lines appear in all three: look up
the row, raise `404` if it's missing. Duplicated code means a future
fix has to happen in three places.

Add this helper to `app/main.py`, **above** the first route (below the
line `app = FastAPI(...)`):

```python
def find_application_or_404(
    db: Session,
    application_id: int,
) -> models.Application:
    application = db.get(models.Application, application_id)
    if application is None:
        raise HTTPException(
            status_code=404,
            detail="Application not found",
        )
    return application
```

The **`-> models.Application`** after the parameters is a **return type
hint**: it tells readers (and Pylance) what the function gives back.

Now, in each of the three routes, replace the six duplicated lines with
one:

```python
application = find_application_or_404(db, application_id)
```

```bash
uv run pytest
```

🟢 **Still `12 passed`.** That's the whole point of refactoring under
test: you changed the structure and the tests confirm the behavior is
identical.

### Your `app/main.py` should now look like this

```python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy import select
from sqlalchemy.orm import Session

from app import models, schemas
from app.database import get_db

app = FastAPI(title="Tiro Job Tracker")


def find_application_or_404(
    db: Session,
    application_id: int,
) -> models.Application:
    application = db.get(models.Application, application_id)
    if application is None:
        raise HTTPException(
            status_code=404,
            detail="Application not found",
        )
    return application


@app.get("/health")
def health():
    return {"status": "ok"}


@app.post(
    "/applications",
    response_model=schemas.ApplicationRead,
    status_code=201,
)
def create_application(
    payload: schemas.ApplicationCreate,
    db: Annotated[Session, Depends(get_db)],
):
    application = models.Application(**payload.model_dump())
    db.add(application)
    db.commit()
    db.refresh(application)
    return application


@app.get(
    "/applications",
    response_model=list[schemas.ApplicationRead],
)
def list_applications(db: Annotated[Session, Depends(get_db)]):
    query = select(models.Application).order_by(
        models.Application.id
    )
    return db.scalars(query).all()


@app.get(
    "/applications/{application_id}",
    response_model=schemas.ApplicationRead,
)
def get_application(
    application_id: int,
    db: Annotated[Session, Depends(get_db)],
):
    return find_application_or_404(db, application_id)


@app.patch(
    "/applications/{application_id}",
    response_model=schemas.ApplicationRead,
)
def update_application(
    application_id: int,
    payload: schemas.ApplicationUpdate,
    db: Annotated[Session, Depends(get_db)],
):
    application = find_application_or_404(db, application_id)
    changes = payload.model_dump(exclude_unset=True)
    for field, value in changes.items():
        setattr(application, field, value)
    db.commit()
    db.refresh(application)
    return application


@app.delete(
    "/applications/{application_id}",
    status_code=204,
)
def delete_application(
    application_id: int,
    db: Annotated[Session, Depends(get_db)],
):
    application = find_application_or_404(db, application_id)
    db.delete(application)
    db.commit()
```

Commit with message `refactor(backend): extract find_application_or_404`.

---

## Close the coverage gap

Look at the coverage table (or `app/database.py` with Coverage Gutters
watching). The body of `get_db` is still red. Every API test *overrides*
`get_db`, so the real one never runs.

**Connect the dots.** `get_db` is a **generator** (it uses `yield`).
Calling `next(generator)` runs it up to the `yield` and returns the
session. Calling `generator.close()` resumes it so the `finally` block
runs.

Create `backend/tests/test_database.py`:

```python
from sqlalchemy.orm import Session

from app.database import get_db


def test_get_db_yields_a_session_and_closes_it():
    generator = get_db()

    db = next(generator)

    assert isinstance(db, Session)
    generator.close()
```

🟢 **Expected:** `13 passed`, and `app/database.py` at 100%.

> This test doesn't create `tiro.db` — a session only connects when it
> runs a query, and this one never does.

---

## See it for real

```bash
uv run alembic upgrade head
uv run fastapi dev app/main.py
```

Open <http://127.0.0.1:8000/docs>. Use **Try it out** to:

1. `POST /applications` with a real application you've sent (or a
   made-up one).
2. `GET /applications` to see it listed.
3. `PATCH` its status to `interviewing`.
4. Open `tiro.db` in VS Code's SQLite Viewer — your row is there.

Stop the server with `Ctrl+C`, then commit:

```bash
uv run ruff check .
uv run ruff format .
cd ..
git add .
git commit -m "test(backend): cover get_db"
git push
```

---

## Explain it back

**1. `GET /applications` returned `405` while `GET /applications/1`
would have returned `404` at the same moment. Why the difference?**

<details>
<summary>Answer</summary>

`/applications` already existed as a path (for `POST`), so FastAPI knew
the path but not the method: `405 Method Not Allowed`.
`/applications/1` didn't match any route at all: `404 Not Found`.
</details>

**2. What would happen in `PATCH` without `exclude_unset=True`?**

<details>
<summary>Answer</summary>

Every field the client didn't send would be dumped as `None` and written
to the database, wiping out `company`, `role`, and `applied_on`.
(Actually the database would reject `None` for those required columns,
so you'd get an error — either way, broken.)
</details>

**3. Why does each API test get a fresh database instead of sharing
one?**

<details>
<summary>Answer</summary>

So tests can't affect each other. If they shared data, a test might
pass or fail depending on which tests ran before it — impossible to
debug.
</details>

**4. What is a dependency override, and why is it useful?**

<details>
<summary>Answer</summary>

It tells FastAPI to call a different function wherever a dependency is
requested. In tests, it swaps the real database session for one
connected to the in-memory test database — without changing any app
code.
</details>

**5. Why separate `ApplicationCreate` and `ApplicationRead`?**

<details>
<summary>Answer</summary>

The client never sends an `id` (the database creates it), but always
receives one. Separate schemas describe each direction exactly.
</details>

---

## Checkpoint

- ✅ `uv run pytest` → `13 passed`, 100% coverage
- ✅ Every endpoint works in `/docs`
- ✅ Your commit history shows one commit per cycle

Docs for going deeper:

- Path parameters: <https://fastapi.tiangolo.com/tutorial/path-params/>
- Request body: <https://fastapi.tiangolo.com/tutorial/body/>
- Dependencies: <https://fastapi.tiangolo.com/tutorial/dependencies/>
- Partial updates with `PATCH`:
  <https://fastapi.tiangolo.com/tutorial/body-updates/>
- Testing dependencies with overrides:
  <https://fastapi.tiangolo.com/advanced/testing-dependencies/>

Next: [06 — Frontend: first test](06-frontend-init.md)
