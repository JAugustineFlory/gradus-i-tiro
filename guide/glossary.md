# Glossary

Every new term in Tiro, in plain words. The lesson where it first
appears is in brackets.

---

**Accessible name** [08] — What a screen reader announces for an
element. For a button, usually its text, or its `aria-label` if it has
one. Testing Library's `getByRole(..., { name })` searches by it.

**Alembic** [05] — The tool that creates and runs **migrations** for a
SQLAlchemy database.

**API** [00] — Application Programming Interface. Here: the set of URLs
(endpoints) the backend offers, and what each one accepts and returns.

**Arrange, Act, Assert** [00] — The three parts of a test: set up, do
the thing, check the result.

**ASGI** [06] — The standard interface between Python async web apps
(like FastAPI) and the programs that run or test them.

**`async def` / `await`** [05] — `async def` defines a function that's
allowed to pause; `await` pauses it until a slow operation (like a
database query) finishes, letting other work run meanwhile. The
frontend uses the same keywords for the same idea [08].

**`AsyncClient`** [06] — httpx's async HTTP client. With an
`ASGITransport`, it sends requests straight to your app inside the
test's own event loop.

**asyncpg** [05] — The driver SQLAlchemy uses to talk to PostgreSQL
asynchronously.

**`AsyncSession`** [05] — SQLAlchemy's async session: database work on
it is awaited.

**Callback prop** [08] — A function passed to a component as a prop,
which the component calls when something happens (`onDelete`,
`onSubmit`).

**Component** [07] — A function that returns what should appear on
screen. The building block of a React UI.

**Container** [04] — A running instance of a Docker image.

**Controlled input** [08] — A form field whose value comes from React
state, and which updates that state on every change.

**Conventional Commits** [02] — A commit message style:
`type: description` (e.g. `feat: add delete button`).

**`conftest.py`** [05] — A special pytest file whose fixtures are shared
with every test in its folder.

**Coroutine** [05] — What calling an `async def` function gives you: a
paused piece of work that runs when awaited.

**CORS** [08] — Cross-Origin Resource Sharing. Browsers block a page
from reading responses from a different **origin** unless the server
sends a header allowing it.

**Coverage** [00] — Which lines of code ran during tests.

**CRUD** [06] — Create, Read, Update, Delete — the four basic operations
on stored data.

**Database** [04] — A program built to store data safely and answer
questions about it. Here, **PostgreSQL**.

**Decorator** [03] — A line starting with `@` above a Python function
that adds behavior to it. `@app.get("/health")` registers the function
as a route.

**Dependency (package)** [03] — A package your project needs.
**Dev dependencies** are needed only for development (tests, linters).

**Dependency injection** [06] — Declaring what a function needs
(`Depends(get_db)`) and letting the framework supply it.

**Dependency override** [06] — Telling FastAPI to supply something
different for a dependency, usually in tests.

**Destructuring** [08] — Pulling named values out of an object:
`{ applications, onDelete }` from props.

**Docker** [04] — A tool that runs software in **containers**, the same
way on every machine.

**Docker Compose** [04] — A file (`compose.yaml`) describing a
project's containers, started together with `docker compose up`.

**Driver** [05] — The library an ORM uses to speak a particular
database's network protocol (here, asyncpg).

**Endpoint / route** [03] — One method + path the API responds to, like
`GET /health`.

**Engine** [05] — SQLAlchemy's object that knows where the database is
and keeps a pool of connections to it.

**Event loop** [05] — The part of Python that tracks every paused async
function and resumes each when what it's waiting for is ready.

**f-string** [06] — Python text with `{expressions}` filled in:
`f"/applications/{id}"`.

**Fixture** [05] — A pytest function that prepares something a test
needs, passed in by naming it as a parameter.

**Generator / async generator** [05] — A function that uses `yield` to
hand out a value and pause, then resume later. `get_db` is an async
generator.

**Git hook** [02] — A script Git runs automatically at a certain moment,
like just before a commit.

**Health check** [03, 04] — A quick "are you alive and ready?" check:
`GET /health` for the backend, `pg_isready` for PostgreSQL.

**Husky** [02] — A tool that installs Git hooks for everyone who clones
the repo.

**Image** [04] — A packaged, read-only snapshot of software (like
`postgres:17`), used to start containers.

**JSON** [00] — A text format for data: `{"company": "Acme"}`. What the
frontend and backend send each other.

**jsdom** [07] — A fake browser that runs inside Node, used by tests.

**JSX / TSX** [07] — HTML-like syntax inside JavaScript/TypeScript.

**`key`** [08] — A unique, stable identifier React needs on each item in
a rendered list.

**Linter** [03] — A tool that reads code and flags likely mistakes and
style problems (Ruff, ESLint).

**Lockfile** [02] — `uv.lock` / `package-lock.json`: exact versions of
every installed package. Always commit them.

**Middleware** [08] — Code that runs on every request and response,
around your routes.

**Migration** [05] — A script that changes a database's structure, with
an `upgrade` and a `downgrade`.

**Mock function** [08] — A fake function (`vi.fn()`) that records how it
was called.

**Model** [05] — A Python class representing a database table.

**ORM** [05] — Object-Relational Mapper. Translates between objects in
code and rows in a database. SQLAlchemy is one.

**Origin** [08] — Scheme + host + port, like `http://localhost:5173`.

**Package (Python)** [03] — A folder of modules with an `__init__.py`,
importable by name.

**Path parameter** [06] — A variable part of a URL:
`/applications/{application_id}`.

**PEP 8** [02] — Python's official style guide (4-space indentation,
naming conventions, and more).

**Port mapping** [04] — Connecting a port on your computer to a port
inside a container: `"5433:5432"`.

**PostgreSQL** [04] — The database used in this project: a server that
stores data and speaks SQL.

**Primary key** [04, 05] — The column that uniquely identifies each row
(`id`).

**Promise** [08] — A JavaScript value that will be available later
(like a network response). JavaScript's counterpart to a coroutine.

**Props** [08] — The inputs to a React component.

**`psql`** [04] — PostgreSQL's command-line client.

**Pure function** [07] — A function whose output depends only on its
input, with no side effects.

**Pydantic schema** [06] — A class describing the shape of data crossing
the API, used to validate input and shape output.

**Red → green → refactor** [00] — The TDD loop: failing test, passing
code, clean up.

**Refactor** [00] — Changing the structure of code without changing its
behavior.

**Session** [05] — SQLAlchemy's conversation with the database: stage
changes, then `commit`.

**Spread** [08] — `...` copies the contents of an array or object into a
new one.

**SQL** [04] — The language databases speak: `SELECT`, `INSERT`,
`CREATE TABLE`, and so on.

**State** [08] — Data a React component remembers, which re-draws the
component when changed.

**Status code** [06] — The number in an HTTP response saying what
happened (`200`, `404`…).

**TDD** [00] — Test-Driven Development: write the test first.

**Template literal** [08] — JavaScript text with `${expressions}`:
`` `Delete ${company}` ``.

**Test database** [04] — A separate database (`tiro_test`) that tests
can create and wipe freely, so they never touch development data.

**Transaction** [05] — A group of database changes that succeed or fail
together.

**Type / type annotation** [05, 07] — A description of what kind of
value something is (`Mapped[int]`, `status: string`), checked before the
code runs.

**User story** [00] — A feature described from the user's point of
view: *As a…, I want…, so that…*.

**Utility type** [08] — A TypeScript type built from another, like
`Omit<Application, 'id'>`.

**uv** [01] — The tool that installs Python, creates the virtual
environment, and manages packages.

**Virtual environment (`.venv`)** [01] — A folder holding one project's
Python packages, isolated from every other project.

**Volume** [04] — Docker-managed storage that survives when a container
is removed. Where PostgreSQL keeps its data.

**Watch mode** [07] — A test runner that re-runs tests every time you
save.

---

## Latin

| Word | Meaning |
| --- | --- |
| *gradus* | step, rank |
| *tiro* | recruit, beginner |
| *miles* | soldier |
| *veteranus* | veteran |
