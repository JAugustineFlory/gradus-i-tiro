# 06 — CRUD endpoints with TDD

> **Where this fits.** This is where the backend starts delivering the
> user stories:
>
> | Story | Endpoint you'll build |
> | --- | --- |
> | **US-1** record an application | `POST /applications` |
> | **US-2** see all applications | `GET /applications` (and one: `GET /applications/{id}`) |
> | **US-3** update its status | `PATCH /applications/{id}` |
> | **US-4** delete a mistake | `DELETE /applications/{id}` |
>
> In the lesson 00 diagram, you're building the "FastAPI routes" box and
> connecting it to the models from lesson 05. The frontend will call
> these same endpoints in lesson 08.

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

Make sure the database is running (`docker compose up -d --wait` at the
repo root). Work in `backend/` for this whole lesson. It's the longest
backend lesson — take a break between cycles if you need one.

---

## Concepts first

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
db: Annotated[AsyncSession, Depends(get_db)]
```

Read it as: "`db` is an `AsyncSession`, and FastAPI should get it by
calling `get_db`." For each request, FastAPI runs `get_db` up to its
`yield`, passes the session in, and after the response finishes the
rest of `get_db` — closing the session.

The payoff comes in testing: you can tell FastAPI "whenever something
asks for `get_db`, use *this other function* instead" — and point it at
the test database. That's a **dependency override**.

### Async endpoints

Endpoints that talk to the database are **`async def`**, and every
database call inside them is **awaited** — the async waiter from
lesson 05. (Your `health` endpoint stays a plain `def`: it doesn't wait
on anything, and FastAPI happily runs both kinds.)

### Testing an async app: `AsyncClient`

In lesson 03 you used `TestClient`. It works by running your app in a
**separate thread with its own event loop**. That was fine for
`/health`, which never touches the database. But now your test fixtures
open database connections **in the test's event loop**, and a
connection can only be used by the loop that opened it. Mix the two and
you get errors like *"attached to a different loop."*

So from here on, tests use **httpx's `AsyncClient`** with an
**`ASGITransport`**. It calls your app *directly*, inside the test's own
event loop — no thread, no second loop. Every request is awaited:

```python
response = await client.post("/applications", json={...})
```

(**ASGI** is the standard interface between Python async web apps and
the things that run them; FastAPI is an ASGI app. httpx came with
`fastapi[standard]`.)

---

## Step 1 — Add a `client` fixture

**Connect the dots.**

- Model tests (lesson 05) need a **session** on the test database.
- API tests need a **client** whose requests use that **same** test
  database — so the app's `get_db` must be overridden.
- Both should start from empty tables every test. The `engine` fixture
  already does that, so `client` should depend on `engine`.

📄 **File:** `backend/tests/conftest.py` — **edit**: replace the whole
file with this. New lines are **2**, **7–8**, and **31–46**; the rest is
unchanged from lesson 05.

```python
import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine
from sqlalchemy.pool import NullPool

from app import models  # noqa: F401
from app.database import Base, get_db
from app.main import app

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


@pytest.fixture
async def client(engine):
    TestingSession = async_sessionmaker(engine, expire_on_commit=False)

    async def override_get_db():
        async with TestingSession() as db:
            yield db

    app.dependency_overrides[get_db] = override_get_db
    transport = ASGITransport(app=app)
    async with AsyncClient(
        transport=transport,
        base_url="http://test",
    ) as test_client:
        yield test_client
    app.dependency_overrides.clear()
```

- **Line 2** imports the async client and its transport.
- **Line 7** now also imports `get_db` (to override it); **line 8**
  imports your app.
- **Line 33** — a session factory bound to the **test** engine.
- **Lines 35–37** — a replacement for `get_db` that yields test-database
  sessions instead.
- **Line 39** — the **dependency override**: "wherever the app asks for
  `get_db`, call `override_get_db`."
- **Line 40** — a transport that delivers requests straight to `app`.
- **Lines 41–45** — the client. `base_url` is required by httpx but
  never actually contacted; `"http://test"` is the convention.
- **Line 46** — after the test, remove the override so it can't leak
  into other tests.

---

## Step 2 — Move the health test onto the new client

**Connect the dots.** `test_health.py` still makes its own
`TestClient`. Now that every test has a `client` fixture, the health test
should use it too — one way of testing, everywhere.

📄 **File:** `backend/tests/test_health.py` — **edit**: replace the
whole file with:

```python
async def test_health_returns_ok(client):
    response = await client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

- It's now **`async def`** and takes `client` as a parameter.
- **Line 2** — `await client.get(...)`: the async client's requests must
  be awaited.
- No imports needed: the fixture provides everything.

```bash
uv run pytest
```

✅ Still `2 passed`. You changed *how* the test runs without changing
*what* it checks — a small **refactor**, proven safe by staying green.

---

## Step 3 — Schemas

📄 **File:** `backend/app/schemas.py` — **new**:

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
| `ApplicationCreate` (lines 6–10) | Body of `POST` | No `id` — the database assigns it. `status` is optional with a default. |
| `ApplicationUpdate` (lines 13–17) | Body of `PATCH` | *Every* field optional — you might only change `status`. `str \| None` means "a string, or nothing." |
| `ApplicationRead` (lines 20–27) | Every response | Includes `id`. **`from_attributes=True`** (line 21) lets Pydantic read a SQLAlchemy object's attributes directly. |

Each class inherits from Pydantic's **`BaseModel`**, and each field is a
**type hint**. Pydantic uses those hints to check incoming data and
convert it — the text `"2026-10-01"` becomes a real `date`.

---

## Cycle 1 — Create an application (US-1)

> **Story:** *As a job seeker, I want to record each application I send.*
> **Test it means:** `POST /applications` with valid data returns `201`
> and the saved row, including the `id` the database assigned and the
> default status.

### 🔴 Red

📄 **File:** `backend/tests/test_applications.py` — **new**:

```python
SAMPLE = {
    "company": "Acme",
    "role": "Junior Developer",
    "applied_on": "2026-10-01",
}


async def test_create_application_returns_201_and_the_row(client):
    response = await client.post("/applications", json=SAMPLE)

    assert response.status_code == 201
    body = response.json()
    assert body["id"] == 1
    assert body["company"] == "Acme"
    assert body["status"] == "applied"
    assert body["applied_on"] == "2026-10-01"
```

- **Lines 1–5** — sample data, shared by many tests in this file. Dates
  travel as text in `YYYY-MM-DD` format.
- **Line 9** — **`json=SAMPLE`** sends the dictionary as a JSON body.
- **Line 12** — `response.json()` turns the response's JSON back into a
  Python dictionary.
- **Line 13** — `id` is `1` because every test starts with freshly
  created tables, so the numbering starts over.
- **Line 15** checks the default status was applied, even though we
  never sent one.

```bash
uv run pytest
```

🔴 **Expected:** `assert 404 == 201`. The route doesn't exist.

### 🟢 Green

📄 **File:** `backend/app/main.py` — **edit**: replace the whole file
with:

```python
from typing import Annotated

from fastapi import Depends, FastAPI
from sqlalchemy.ext.asyncio import AsyncSession

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
async def create_application(
    payload: schemas.ApplicationCreate,
    db: Annotated[AsyncSession, Depends(get_db)],
):
    application = models.Application(**payload.model_dump())
    db.add(application)
    await db.commit()
    await db.refresh(application)
    return application
```

- **Lines 1–7** — new imports: `Annotated` and `Depends` for dependency
  injection, `AsyncSession` for the type, and your own `models`,
  `schemas`, and `get_db`.
- **Lines 17–21** — `response_model` tells FastAPI to shape the return
  value with `ApplicationRead`. `status_code=201` replaces the default
  `200`.
- **Line 22** — `async def`: this endpoint awaits the database.
- **Line 23** — a parameter typed as a Pydantic schema means "read this
  from the JSON body and validate it."
- **Line 24** — the injected database session.
- **Line 26** — `payload.model_dump()` turns the schema into a
  dictionary: `{"company": "Acme", "role": ..., ...}`. The **`**`**
  *unpacks* it into keyword arguments, so this equals
  `models.Application(company="Acme", role=..., ...)`.
- **Line 27** — `add` stages the object (no `await`: nothing is sent
  yet).
- **Line 28** — `await db.commit()` sends the `INSERT` and saves it.
- **Line 29** — `await db.refresh(...)` reloads the row from the
  database, so the object has everything the database filled in.
- **Line 30** — return the model object; FastAPI converts it to JSON
  through `ApplicationRead`.

```bash
uv run pytest
```

🟢 **Expected:** `3 passed`.

Commit (from the repo root; `cd ..` first, then come back):

```bash
cd ..
git add .
git commit -m "feat(backend): create applications"
cd backend
```

---

## Cycle 2 — Reject bad input (US-1)

> **Story, continued:** recording an application is only useful if the
> record is complete. **Test it means:** a `POST` without `company`
> returns `422`.

### 🔴 Red?

📄 **File:** `backend/tests/test_applications.py` — **edit**: add this
test at the **end of the file** (two blank lines after the previous
test):

```python
async def test_create_without_company_returns_422(client):
    payload = {
        "role": "Junior Developer",
        "applied_on": "2026-10-01",
    }

    response = await client.post("/applications", json=payload)

    assert response.status_code == 422
```

```bash
uv run pytest
```

🟢 **It passes immediately.** Pydantic already rejects the missing field.

This happens in real TDD: sometimes the framework already does what you
want. But a test that has never failed hasn't proven anything. So
**prove it can fail**:

1. 📄 In `backend/app/schemas.py`, **line 7**, temporarily change
   `company: str` to `company: str = "Unknown"`.
2. Run `uv run pytest`. 🔴 The new test fails with `assert 201 == 422`.
3. Change line 7 back. 🟢 Green again.

Now you *know* this test guards against someone making `company`
optional by accident.

Commit with message `test(backend): reject applications without a
company`.

---

## Cycle 3 — List applications (US-2)

> **Story:** *As a job seeker, I want to see all my applications in one
> list.* **Test it means:** `GET /applications` returns `200` and every
> application, oldest first — or an empty list when there are none.

### 🔴 Red

Many tests from here on need an application to already exist. A small
helper saves repeating the `POST` every time.

📄 **File:** `backend/tests/test_applications.py` — **edit**: add this
helper **directly below `SAMPLE`** (after line 5, with two blank lines
before and after it):

```python
async def create_sample(client, **overrides):
    payload = {**SAMPLE, **overrides}
    response = await client.post("/applications", json=payload)
    return response.json()
```

- **`async def`** — it awaits the client, so callers must `await` it:
  `created = await create_sample(client)`.
- **`**overrides`** collects any extra keyword arguments into a
  dictionary. `create_sample(client, company="Globex")` gives
  `overrides = {"company": "Globex"}`.
- **`{**SAMPLE, **overrides}`** merges two dictionaries; later values
  win. So you get the sample data with `company` swapped.

📄 **File:** `backend/tests/test_applications.py` — **edit**: add these
two tests at the **end of the file**:

```python
async def test_list_applications_starts_empty(client):
    response = await client.get("/applications")

    assert response.status_code == 200
    assert response.json() == []


async def test_list_returns_every_application_in_order(client):
    await create_sample(client, company="Acme")
    await create_sample(client, company="Globex")

    response = await client.get("/applications")

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

📄 **File:** `backend/app/main.py` — **edit**, two changes:

**1.** Add a new import as **line 4**, between the FastAPI import
(line 3) and the `AsyncSession` import:

```python
from sqlalchemy import select
```

Lines 3–5 now read:

```python
from fastapi import Depends, FastAPI
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
```

**2.** Add this route at the **end of the file**:

```python
@app.get(
    "/applications",
    response_model=list[schemas.ApplicationRead],
)
async def list_applications(
    db: Annotated[AsyncSession, Depends(get_db)],
):
    query = select(models.Application).order_by(models.Application.id)
    result = await db.scalars(query)
    return result.all()
```

- **`list[schemas.ApplicationRead]`** — the response is a *list* of
  applications.
- **`select(models.Application)`** builds a query for every row;
  **`.order_by(...id)`** sorts by id. Databases don't promise any order
  unless you ask — without this, the test might pass today and fail
  tomorrow.
- **`await db.scalars(query)`** runs it (database work, so awaited) and
  returns model objects; **`.all()`** collects them into a list.

```bash
uv run pytest
```

🟢 **Expected:** `6 passed`.

Commit with message `feat(backend): list applications`.

---

## Cycle 4 — Get one application (US-2)

> **Story:** seeing one application on its own is part of US-2 — and
> the building block for updating and deleting it. **Test it means:**
> `GET /applications/{id}` returns that application, or `404` with a
> clear message if it doesn't exist.

### 🔴 Red

📄 **File:** `backend/tests/test_applications.py` — **edit**: add at
the **end of the file**:

```python
async def test_get_application_by_id(client):
    created = await create_sample(client)

    response = await client.get(f"/applications/{created['id']}")

    assert response.status_code == 200
    assert response.json() == created


async def test_get_missing_application_returns_404(client):
    response = await client.get("/applications/999")

    assert response.status_code == 404
    assert response.json() == {"detail": "Application not found"}
```

- **`f"..."`** is an **f-string**: `{created['id']}` is replaced with the
  value, giving `/applications/1`.
- The second test checks the *unhappy path* — asking for something that
  doesn't exist. Always test both.

🔴 **Expected:** both fail (`405`).

### 🟢 Green

📄 **File:** `backend/app/main.py` — **edit**, two changes:

**1.** On **line 3**, add `HTTPException` to the FastAPI import:

```python
from fastapi import Depends, FastAPI, HTTPException
```

**2.** Add this route at the **end of the file**:

```python
@app.get(
    "/applications/{application_id}",
    response_model=schemas.ApplicationRead,
)
async def get_application(
    application_id: int,
    db: Annotated[AsyncSession, Depends(get_db)],
):
    application = await db.get(models.Application, application_id)
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
- **`await db.get(Model, id)`** looks up one row by primary key and
  returns `None` if there isn't one.
- **`raise HTTPException`** stops the function and sends an error
  response with that status and `detail`.

🟢 **Expected:** `8 passed`. Commit with message `feat(backend): get one
application`.

---

## Cycle 5 — Update an application (US-3)

> **Story:** *As a job seeker, I want to update an application's status
> as it moves along.* **Test it means:** `PATCH` changes only the fields
> sent, leaves the rest alone, and returns `404` for a missing
> application.

### 🔴 Red

📄 **File:** `backend/tests/test_applications.py` — **edit**: add at
the **end of the file**:

```python
async def test_update_changes_only_the_fields_sent(client):
    created = await create_sample(client)

    response = await client.patch(
        f"/applications/{created['id']}",
        json={"status": "interviewing"},
    )

    assert response.status_code == 200
    body = response.json()
    assert body["status"] == "interviewing"
    assert body["company"] == "Acme"


async def test_update_missing_application_returns_404(client):
    response = await client.patch(
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
client wants to change. Pydantic fills every *missing* field of
`ApplicationUpdate` with its default, `None`. If we copied every field,
we'd set `company` to `None`! We need only the fields the client
**actually sent**. `model_dump(exclude_unset=True)` gives exactly that.

📄 **File:** `backend/app/main.py` — **edit**: add this route at the
**end of the file**:

```python
@app.patch(
    "/applications/{application_id}",
    response_model=schemas.ApplicationRead,
)
async def update_application(
    application_id: int,
    payload: schemas.ApplicationUpdate,
    db: Annotated[AsyncSession, Depends(get_db)],
):
    application = await db.get(models.Application, application_id)
    if application is None:
        raise HTTPException(
            status_code=404,
            detail="Application not found",
        )
    changes = payload.model_dump(exclude_unset=True)
    for field, value in changes.items():
        setattr(application, field, value)
    await db.commit()
    await db.refresh(application)
    return application
```

- **`changes.items()`** gives each `(field, value)` pair. For our test,
  just `("status", "interviewing")`.
- **`setattr(obj, "status", "interviewing")`** is the same as writing
  `obj.status = "interviewing"`, but lets the field name come from a
  variable.
- Changing an attribute on a loaded object is enough: SQLAlchemy notices
  the change and sends an `UPDATE` on commit.

**Try it:** remove `exclude_unset=True`, run the tests, and read the
failure — PostgreSQL refuses to store a `null` company. Then put it
back.

🟢 **Expected:** `10 passed`. Commit with message `feat(backend): update
applications`.

---

## Cycle 6 — Delete an application (US-4)

> **Story:** *As a job seeker, I want to delete an application I logged
> by mistake.* **Test it means:** `DELETE` returns `204`, the
> application is really gone afterwards, and a missing one gives `404`.

### 🔴 Red

📄 **File:** `backend/tests/test_applications.py` — **edit**: add at
the **end of the file**:

```python
async def test_delete_removes_the_application(client):
    created = await create_sample(client)

    response = await client.delete(f"/applications/{created['id']}")

    assert response.status_code == 204
    follow_up = await client.get(f"/applications/{created['id']}")
    assert follow_up.status_code == 404


async def test_delete_missing_application_returns_404(client):
    response = await client.delete("/applications/999")

    assert response.status_code == 404
```

**Lines 7–8** check the *result*, not just the status code: after
deleting, the application really is gone.

🔴 **Expected:** both fail (`405`).

### 🟢 Green

📄 **File:** `backend/app/main.py` — **edit**: add this route at the
**end of the file**:

```python
@app.delete(
    "/applications/{application_id}",
    status_code=204,
)
async def delete_application(
    application_id: int,
    db: Annotated[AsyncSession, Depends(get_db)],
):
    application = await db.get(models.Application, application_id)
    if application is None:
        raise HTTPException(
            status_code=404,
            detail="Application not found",
        )
    await db.delete(application)
    await db.commit()
```

- **No `return`** — a `204` response has no body.
- **`await db.delete(...)`** — unlike `add`, `delete` *is* awaited in
  async SQLAlchemy, because deleting can require loading related rows
  first.

🟢 **Expected:** `12 passed`. Commit with message `feat(backend): delete
applications`.

---

## 🔵 Refactor — remove the duplication

Look at `get_application`, `update_application`, and
`delete_application` in `backend/app/main.py`. The same six lines appear
in all three: look up the row, raise `404` if it's missing. Duplicated
code means a future fix has to happen in three places.

📄 **File:** `backend/app/main.py` — **edit**: add this helper **below
line 10** (`app = FastAPI(...)`), with two blank lines before and after
it:

```python
async def find_application_or_404(
    db: AsyncSession,
    application_id: int,
) -> models.Application:
    application = await db.get(models.Application, application_id)
    if application is None:
        raise HTTPException(
            status_code=404,
            detail="Application not found",
        )
    return application
```

- It's **`async`** because it awaits the database — so callers must
  `await` it too.
- **`-> models.Application`** is a **return type hint**: it tells
  readers (and Pylance) what the function gives back.

Then, in each of the three routes, replace the six duplicated lines with
one:

```python
application = await find_application_or_404(db, application_id)
```

(In `get_application`, you can go one step further and
`return await find_application_or_404(db, application_id)`.)

```bash
uv run pytest
```

🟢 **Still `12 passed`.** That's the whole point of refactoring under
test: you changed the structure and the tests confirm the behavior is
identical.

### Your `backend/app/main.py` should now look like this

```python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app import models, schemas
from app.database import get_db

app = FastAPI(title="Tiro Job Tracker")


async def find_application_or_404(
    db: AsyncSession,
    application_id: int,
) -> models.Application:
    application = await db.get(models.Application, application_id)
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
async def create_application(
    payload: schemas.ApplicationCreate,
    db: Annotated[AsyncSession, Depends(get_db)],
):
    application = models.Application(**payload.model_dump())
    db.add(application)
    await db.commit()
    await db.refresh(application)
    return application


@app.get(
    "/applications",
    response_model=list[schemas.ApplicationRead],
)
async def list_applications(
    db: Annotated[AsyncSession, Depends(get_db)],
):
    query = select(models.Application).order_by(models.Application.id)
    result = await db.scalars(query)
    return result.all()


@app.get(
    "/applications/{application_id}",
    response_model=schemas.ApplicationRead,
)
async def get_application(
    application_id: int,
    db: Annotated[AsyncSession, Depends(get_db)],
):
    return await find_application_or_404(db, application_id)


@app.patch(
    "/applications/{application_id}",
    response_model=schemas.ApplicationRead,
)
async def update_application(
    application_id: int,
    payload: schemas.ApplicationUpdate,
    db: Annotated[AsyncSession, Depends(get_db)],
):
    application = await find_application_or_404(db, application_id)
    changes = payload.model_dump(exclude_unset=True)
    for field, value in changes.items():
        setattr(application, field, value)
    await db.commit()
    await db.refresh(application)
    return application


@app.delete(
    "/applications/{application_id}",
    status_code=204,
)
async def delete_application(
    application_id: int,
    db: Annotated[AsyncSession, Depends(get_db)],
):
    application = await find_application_or_404(db, application_id)
    await db.delete(application)
    await db.commit()
```

Commit with message `refactor(backend): extract find_application_or_404`.

---

## Close the coverage gap

Look at the coverage table (or `backend/app/database.py` with Coverage
Gutters watching). Lines 16–17 — the body of `get_db` — are still red.
Every API test *overrides* `get_db`, so the real one never runs.

**Connect the dots.** `get_db` is an **async generator** (it uses
`yield` inside `async def`).

- **`await anext(generator)`** runs it up to the `yield` and returns the
  session.
- **`await generator.aclose()`** resumes it so the `async with` block
  finishes and the session closes.

📄 **File:** `backend/tests/test_database.py` — **new**:

```python
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db


async def test_get_db_yields_a_session_and_closes_it():
    generator = get_db()

    db = await anext(generator)

    assert isinstance(db, AsyncSession)
    await generator.aclose()
```

🟢 **Expected:** `13 passed`, and `app/database.py` at 100%.

> This test never touches the database: a session only connects when it
> runs a query, and this one never does.

---

## See it for real

From `backend/`:

```bash
uv run alembic upgrade head
uv run fastapi dev app/main.py
```

Open <http://127.0.0.1:8000/docs>. Use **Try it out** to walk through
the user stories:

1. **US-1:** `POST /applications` with a real application you've sent
   (or a made-up one).
2. **US-2:** `GET /applications` to see it listed.
3. **US-3:** `PATCH` its status to `interviewing`.
4. In a second terminal, at the repo root:

   ```bash
   docker compose exec db psql -U tiro -d tiro -c "SELECT * FROM applications;"
   ```

   ✅ Your row is there, in the **development** database.
5. **US-5:** stop the server (`Ctrl+C`), start it again, and `GET` —
   the data is still there.

Commit:

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
to the object. PostgreSQL would refuse to store `null` in the required
`company`, `role`, and `applied_on` columns, so the request fails with an
error instead of updating the status.
</details>

**3. Why did tests switch from `TestClient` to `AsyncClient`?**

<details>
<summary>Answer</summary>

`TestClient` runs the app in a separate thread with its own event loop.
The test database connections belong to the test's event loop, and a
connection can't be used from a different loop. `AsyncClient` with
`ASGITransport` runs the app inside the test's own loop.
</details>

**4. Which calls on the session are awaited, and which aren't? Why?**

<details>
<summary>Answer</summary>

Anything that talks to the database is awaited: `commit`, `refresh`,
`get`, `scalars`, `delete`. `add` isn't — it only records the object in
the session; nothing is sent until `commit`.
</details>

**5. What is a dependency override, and why is it useful?**

<details>
<summary>Answer</summary>

It tells FastAPI to call a different function wherever a dependency is
requested. In tests, it swaps the real database session for one
connected to the test database — without changing any app code.
</details>

---

## Checkpoint

- ✅ `uv run pytest` → `13 passed`, 100% coverage
- ✅ Every endpoint works in `/docs`, and data survives a server restart
- ✅ Your commit history shows one commit per cycle

Docs for going deeper:

- Path parameters: <https://fastapi.tiangolo.com/tutorial/path-params/>
- Request body: <https://fastapi.tiangolo.com/tutorial/body/>
- Dependencies: <https://fastapi.tiangolo.com/tutorial/dependencies/>
- Partial updates with `PATCH`:
  <https://fastapi.tiangolo.com/tutorial/body-updates/>
- Async tests: <https://fastapi.tiangolo.com/advanced/async-tests/>
- Testing dependencies with overrides:
  <https://fastapi.tiangolo.com/advanced/testing-dependencies/>

Next: [07 — Frontend: first test](07-frontend-init.md)
