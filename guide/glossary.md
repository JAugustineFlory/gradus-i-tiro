# Glossary

Every new term in Tiro, in plain words. The lesson where it first
appears is in brackets.

---

**Accessible name** [07] — What a screen reader announces for an
element. For a button, usually its text, or its `aria-label` if it has
one. Testing Library's `getByRole(..., { name })` searches by it.

**Alembic** [04] — The tool that creates and runs **migrations** for a
SQLAlchemy database.

**API** [00] — Application Programming Interface. Here: the set of URLs
(endpoints) the backend offers, and what each one accepts and returns.

**Arrange, Act, Assert** [00] — The three parts of a test: set up, do
the thing, check the result.

**`async` / `await`** [07] — `await` pauses until a **Promise** has its
value. Any function using `await` must be marked `async`.

**Callback prop** [07] — A function passed to a component as a prop,
which the component calls when something happens (`onDelete`,
`onSubmit`).

**Component** [06] — A function that returns what should appear on
screen. The building block of a React UI.

**Conventional Commits** [02] — A commit message style:
`type: description` (e.g. `feat: add delete button`).

**`conftest.py`** [05] — A special pytest file whose fixtures are shared
with every test in its folder.

**Controlled input** [07] — A form field whose value comes from React
state, and which updates that state on every change.

**CORS** [07] — Cross-Origin Resource Sharing. Browsers block a page
from reading responses from a different **origin** unless the server
sends a header allowing it.

**Coverage** [00] — Which lines of code ran during tests.

**CRUD** [05] — Create, Read, Update, Delete — the four basic operations
on stored data.

**Decorator** [03] — A line starting with `@` above a Python function
that adds behavior to it. `@app.get("/health")` registers the function
as a route.

**Dependency (package)** [03] — A package your project needs.
**Dev dependencies** are needed only for development (tests, linters).

**Dependency injection** [05] — Declaring what a function needs
(`Depends(get_db)`) and letting the framework supply it.

**Dependency override** [05] — Telling FastAPI to supply something
different for a dependency, usually in tests.

**Destructuring** [07] — Pulling named values out of an object:
`{ applications, onDelete }` from props.

**Endpoint / route** [03] — One method + path the API responds to, like
`GET /health`.

**Engine** [04] — SQLAlchemy's object that knows where the database is
and how to connect.

**f-string** [05] — Python text with `{expressions}` filled in:
`f"/applications/{id}"`.

**Fixture** [04] — A pytest function that prepares something a test
needs, passed in by naming it as a parameter.

**Generator** [04] — A Python function that uses `yield` to hand out a
value and pause, then resume later.

**Git hook** [02] — A script Git runs automatically at a certain moment,
like just before a commit.

**Husky** [02] — A tool that installs Git hooks for everyone who clones
the repo.

**In-memory database** [04] — A SQLite database that lives only in RAM
(`sqlite://`) and disappears when closed. Ideal for tests.

**JSON** [00] — A text format for data: `{"company": "Acme"}`. What the
frontend and backend send each other.

**jsdom** [06] — A fake browser that runs inside Node, used by tests.

**JSX / TSX** [06] — HTML-like syntax inside JavaScript/TypeScript.

**`key`** [07] — A unique, stable identifier React needs on each item in
a rendered list.

**Linter** [03] — A tool that reads code and flags likely mistakes and
style problems (Ruff, ESLint).

**Lockfile** [02] — `uv.lock` / `package-lock.json`: exact versions of
every installed package. Always commit them.

**Middleware** [07] — Code that runs on every request and response,
around your routes.

**Migration** [04] — A script that changes a database's structure, with
an `upgrade` and a `downgrade`.

**Mock function** [07] — A fake function (`vi.fn()`) that records how it
was called.

**Model** [04] — A Python class representing a database table.

**ORM** [04] — Object-Relational Mapper. Translates between objects in
code and rows in a database. SQLAlchemy is one.

**Origin** [07] — Scheme + host + port, like `http://localhost:5173`.

**Package (Python)** [03] — A folder of modules with an `__init__.py`,
importable by name.

**Path parameter** [05] — A variable part of a URL:
`/applications/{application_id}`.

**Primary key** [04] — The column that uniquely identifies each row
(`id`).

**Promise** [07] — A JavaScript value that will be available later
(like a network response).

**Props** [07] — The inputs to a React component.

**Pure function** [06] — A function whose output depends only on its
input, with no side effects.

**Pydantic schema** [05] — A class describing the shape of data crossing
the API, used to validate input and shape output.

**Red → green → refactor** [00] — The TDD loop: failing test, passing
code, clean up.

**Refactor** [00] — Changing the structure of code without changing its
behavior.

**Session** [04] — SQLAlchemy's conversation with the database: stage
changes, then `commit`.

**Spread** [07] — `...` copies the contents of an array or object into a
new one.

**State** [07] — Data a React component remembers, which re-draws the
component when changed.

**Status code** [05] — The number in an HTTP response saying what
happened (`200`, `404`…).

**TDD** [00] — Test-Driven Development: write the test first.

**Template literal** [07] — JavaScript text with `${expressions}`:
`` `Delete ${company}` ``.

**Type / type annotation** [06] — A description of what kind of value
something is, checked before the code runs.

**Utility type** [07] — A TypeScript type built from another, like
`Omit<Application, 'id'>`.

**uv** [01] — The tool that installs Python, creates the virtual
environment, and manages packages.

**Virtual environment (`.venv`)** [01] — A folder holding one project's
Python packages, isolated from every other project.

**Watch mode** [06] — A test runner that re-runs tests every time you
save.

---

## Latin

| Word | Meaning |
| --- | --- |
| *gradus* | step, rank |
| *tiro* | recruit, beginner |
| *miles* | soldier |
| *veteranus* | veteran |
