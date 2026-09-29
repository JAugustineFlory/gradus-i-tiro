# 07 — Components with TDD

**Goal:** two tested React components (a list and a form), a small API
module, and an `App` that wires everything to the backend. Along the
way you'll hit a real **CORS** error and fix it test-first.

Work in `frontend/` unless a step says otherwise.

> **Prettier reshapes code when you save.** It may join or split long
> lines differently from this guide. That's fine — the code is the same.
> Line numbers in this lesson assume the file as Prettier formats it.

---

## The concepts first

### Components and props

A **component** is a function that returns what should appear on
screen. **Props** are its inputs — like function arguments, passed as
attributes:

```tsx
<ApplicationList applications={items} onDelete={removeIt} />
```

Inside `ApplicationList`, `applications` and `onDelete` arrive as
props. A component should never change its own props.

### State

**State** is data a component remembers between renders, and changing it
makes React re-draw the component.

```tsx
const [form, setForm] = useState(EMPTY_FORM)
```

- `form` — the current value
- `setForm` — the function you call to change it
- `useState(...)` — the starting value

Never change state directly (`form.company = 'x'`). Always call the
setter with a **new** value.

### Effects

**`useEffect`** runs code *after* the component appears on screen —
for things outside React, like fetching data. The `[]` at the end means
"only once, when the component first appears."

### Passing functions down ("lifting state up")

The list and the form don't talk to the backend themselves. They receive
**callback props** like `onDelete` and call them. The parent (`App`)
decides what actually happens. That keeps the components simple — and
easy to test, because a test can pass in a fake callback and check it
was called.

### Testing like a user

React Testing Library encourages finding elements the way a person
would. In order of preference:

1. **`getByRole`** — by what the element *is* (`button`, `listitem`,
   `combobox` for a `<select>`), plus its accessible name
2. **`getByLabelText`** — a form field by its label
3. **`getByText`** — by visible text

If a test can't find an element by role or label, a screen reader user
probably can't either. **Testable and accessible are the same thing
here.**

Priority guide:
<https://testing-library.com/docs/queries/about/#priority>

### Mock functions

`vi.fn()` creates a **mock function**: a fake that records every call.
Pass it as a prop, click something, then ask:

```ts
expect(onDelete).toHaveBeenCalledWith(2)
```

---

## Step 1 — Shared types

**Connect the dots.** The backend's `ApplicationRead` schema (in
`backend/app/schemas.py`) defines what an application looks like in
JSON. The frontend needs the same shape as a TypeScript **type**, so the
editor can catch typos like `app.compnay`.

Create `frontend/src/types.ts`:

```ts
export const STATUSES = ['applied', 'interviewing', 'offer', 'rejected']

export type Application = {
  id: number
  company: string
  role: string
  status: string
  applied_on: string
}

export type NewApplication = Omit<Application, 'id'>
```

- **`type Application = {...}`** describes an object's shape. It exists
  only while writing code; it disappears when the code runs.
- **`applied_on: string`** — dates arrive from JSON as text
  (`'2026-10-01'`).
- **`Omit<Application, 'id'>`** is a **utility type**: "`Application`
  without `id`." That's exactly what the form sends — the backend
  assigns the `id`.

---

## Step 2 — `ApplicationList`, cycle by cycle

Start watch mode and leave it running for this whole step:

```bash
npm test
```

### Cycle 1 — the empty state

#### 🔴 Red

Create `frontend/src/components/ApplicationList.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, expect, it, vi } from 'vitest'
import type { Application } from '../types'
import { ApplicationList } from './ApplicationList'

const acme: Application = {
  id: 1,
  company: 'Acme',
  role: 'Junior Developer',
  status: 'applied',
  applied_on: '2026-10-01',
}

const globex: Application = {
  id: 2,
  company: 'Globex',
  role: 'QA Analyst',
  status: 'interviewing',
  applied_on: '2026-10-03',
}

function doNothing() {}

describe('ApplicationList', () => {
  it('invites you to add one when empty', () => {
    render(
      <ApplicationList
        applications={[]}
        onDelete={doNothing}
        onStatusChange={doNothing}
      />,
    )

    expect(
      screen.getByText(
        'No applications yet. Add your first one above.',
      ),
    ).toBeInTheDocument()
  })
})
```

- **Line 4, `import type`:** this project's TypeScript settings
  require type-only imports to say `type`. It tells the build tool
  "this import disappears at runtime."
- **Lines 7–21** are **test data**, shared by several tests.
- **Line 23:** tests that don't care about a callback pass one that
  does nothing.
- **`render(...)`** draws the component into jsdom. **`screen`** is
  how you search what was drawn.

🔴 **Expected:** `Failed to resolve import "./ApplicationList"`.

#### 🟢 Green

Create `frontend/src/components/ApplicationList.tsx`:

```tsx
import type { Application } from '../types'

type Props = {
  applications: Application[]
  onDelete: (id: number) => void
  onStatusChange: (id: number, status: string) => void
}

export function ApplicationList({ applications }: Props) {
  if (applications.length === 0) {
    return <p>No applications yet. Add your first one above.</p>
  }
  return null
}
```

- **Lines 3–7** type the props. `(id: number) => void` means "a
  function that takes a number and returns nothing useful."
- **Line 9, `{ applications }`:** **destructuring** — pull the
  `applications` prop out by name. We'll add the others when a test
  needs them.
- **`return null`** means "render nothing." It's the smallest thing that
  passes.

🟢 **Expected:** the test passes.

> **Empty text is direction, not decoration.** "No applications yet"
> alone leaves the user stuck. Telling them what to do next turns an
> empty screen into an invitation.

### Cycle 2 — show each application

#### 🔴 Red

Add inside the `describe` block, after the first test:

```tsx
  it('shows each application', () => {
    render(
      <ApplicationList
        applications={[acme, globex]}
        onDelete={doNothing}
        onStatusChange={doNothing}
      />,
    )

    expect(screen.getAllByRole('listitem')).toHaveLength(2)
    expect(screen.getByText('Acme')).toBeInTheDocument()
    expect(screen.getByText('QA Analyst')).toBeInTheDocument()
  })
```

- **`getAllByRole('listitem')`** finds every `<li>`. `getAll...` returns
  a list; `get...` expects exactly one match and fails otherwise.

🔴 **Expected:** `Unable to find an accessible element with the role
"listitem"`.

#### 🟢 Green

Replace `return null` (line 13 of `ApplicationList.tsx`) with:

```tsx
  return (
    <ul>
      {applications.map((application) => (
        <li key={application.id}>
          <strong>{application.company}</strong>
          <span>{application.role}</span>
        </li>
      ))}
    </ul>
  )
```

- **`{ ... }`** inside JSX means "run this JavaScript and show the
  result."
- **`.map(...)`** turns each application into an `<li>`.
- **`key={application.id}`** — React needs a unique, stable `key` on
  each item in a list to track which one is which when the list changes.
  Use the database `id`, never the array position.
- Company and role are in **separate elements** so each can be found by
  its own text.

🟢 **Expected:** `2 passed` for this file.

### Cycle 3 — delete

#### 🔴 Red

```tsx
  it('calls onDelete with the id', async () => {
    const user = userEvent.setup()
    const onDelete = vi.fn()
    render(
      <ApplicationList
        applications={[acme, globex]}
        onDelete={onDelete}
        onStatusChange={doNothing}
      />,
    )

    await user.click(
      screen.getByRole('button', { name: 'Delete Globex' }),
    )

    expect(onDelete).toHaveBeenCalledWith(2)
  })
```

- **`async` / `await`:** user-event actions take a moment (like a real
  user), so they return a **Promise** — a value that arrives later.
  `await` waits for it. A function that uses `await` must be marked
  `async`.
- **`userEvent.setup()`** creates a simulated user. Call it at the start
  of each test.
- **`{ name: 'Delete Globex' }`** — there are *two* Delete buttons; the
  accessible name tells them apart.

🔴 **Expected:** `Unable to find an accessible element with the role
"button" and name "Delete Globex"`.

#### 🟢 Green

Two changes in `ApplicationList.tsx`:

1. Line 9: destructure `onDelete` too:

```tsx
export function ApplicationList({ applications, onDelete }: Props) {
```

2. Add a button inside the `<li>`, after the `<span>`:

```tsx
          <button
            type="button"
            aria-label={`Delete ${application.company}`}
            onClick={() => onDelete(application.id)}
          >
            Delete
          </button>
```

- **`aria-label`** sets the button's accessible name. Sighted users see
  "Delete" next to the company; screen reader users hear "Delete
  Globex." Both know which one it removes.
- **`` `Delete ${...}` ``** is a **template literal** — TypeScript's
  version of Python's f-string.
- **`onClick={() => onDelete(application.id)}`** — an **arrow
  function** that runs on click. Writing `onClick={onDelete(...)}`
  (without `() =>`) would call it *immediately* during render — a
  classic bug.

🟢 **Expected:** `3 passed`.

### Cycle 4 — change status

#### 🔴 Red

```tsx
  it('calls onStatusChange with the id and new status', async () => {
    const user = userEvent.setup()
    const onStatusChange = vi.fn()
    render(
      <ApplicationList
        applications={[acme, globex]}
        onDelete={doNothing}
        onStatusChange={onStatusChange}
      />,
    )

    await user.selectOptions(
      screen.getByRole('combobox', { name: 'Status for Acme' }),
      'offer',
    )

    expect(onStatusChange).toHaveBeenCalledWith(1, 'offer')
  })
```

- A `<select>` has the role **`combobox`**.
- **`selectOptions`** picks an option by its value or text.

🔴 **Expected:** can't find the combobox.

#### 🟢 Green

1. Change the imports at the top of `ApplicationList.tsx` (line 1) to:

```tsx
import { STATUSES } from '../types'
import type { Application } from '../types'
import { formatStatus } from '../utils/formatStatus'
```

   (`STATUSES` is a real value, so it uses a normal `import`;
   `Application` is a type, so it keeps `import type`.)

2. Destructure `onStatusChange`:

```tsx
export function ApplicationList({
  applications,
  onDelete,
  onStatusChange,
}: Props) {
```

3. Add a `<select>` inside the `<li>`, between the `<span>` and the
   `<button>`:

```tsx
          <select
            aria-label={`Status for ${application.company}`}
            value={application.status}
            onChange={(event) =>
              onStatusChange(application.id, event.target.value)
            }
          >
            {STATUSES.map((status) => (
              <option key={status} value={status}>
                {formatStatus(status)}
              </option>
            ))}
          </select>
```

- **`value={application.status}`** makes this a **controlled** input:
  React decides what's selected, based on the data.
- **`event.target.value`** is the value of the option the user picked.
- **`formatStatus`** — your function from lesson 06 — turns `offer` into
  `Offer` for display, while the `value` stays lowercase for the
  backend.

🟢 **Expected:** `4 passed`.

### Your finished `ApplicationList.tsx`

```tsx
import { STATUSES } from '../types'
import type { Application } from '../types'
import { formatStatus } from '../utils/formatStatus'

type Props = {
  applications: Application[]
  onDelete: (id: number) => void
  onStatusChange: (id: number, status: string) => void
}

export function ApplicationList({
  applications,
  onDelete,
  onStatusChange,
}: Props) {
  if (applications.length === 0) {
    return <p>No applications yet. Add your first one above.</p>
  }
  return (
    <ul>
      {applications.map((application) => (
        <li key={application.id}>
          <strong>{application.company}</strong>
          <span>{application.role}</span>
          <select
            aria-label={`Status for ${application.company}`}
            value={application.status}
            onChange={(event) =>
              onStatusChange(application.id, event.target.value)
            }
          >
            {STATUSES.map((status) => (
              <option key={status} value={status}>
                {formatStatus(status)}
              </option>
            ))}
          </select>
          <button
            type="button"
            aria-label={`Delete ${application.company}`}
            onClick={() => onDelete(application.id)}
          >
            Delete
          </button>
        </li>
      ))}
    </ul>
  )
}
```

Commit (from the repo root, in a second terminal, or stop watch mode
with `q` first):

```bash
git add .
git commit -m "feat(frontend): add ApplicationList"
```

---

## Step 3 — `ApplicationForm`, cycle by cycle

### Cycle 1 — submit what was typed

#### 🔴 Red

**Connect the dots.**

- The form needs three fields: **Company**, **Role**, **Date applied**.
  Status always starts as `applied`, so it doesn't need a field.
- When submitted, it should call `onSubmit` with a `NewApplication`.
- The test should type into each field *by its label*, then click the
  button.

Create `frontend/src/components/ApplicationForm.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, expect, it, vi } from 'vitest'
import { ApplicationForm } from './ApplicationForm'

describe('ApplicationForm', () => {
  it('submits what the user typed', async () => {
    const user = userEvent.setup()
    const onSubmit = vi.fn()
    render(<ApplicationForm onSubmit={onSubmit} />)

    await user.type(screen.getByLabelText('Company'), 'Acme')
    await user.type(
      screen.getByLabelText('Role'),
      'Junior Developer',
    )
    await user.type(
      screen.getByLabelText('Date applied'),
      '2026-10-01',
    )
    await user.click(
      screen.getByRole('button', { name: 'Add application' }),
    )

    expect(onSubmit).toHaveBeenCalledWith({
      company: 'Acme',
      role: 'Junior Developer',
      status: 'applied',
      applied_on: '2026-10-01',
    })
  })
})
```

🔴 **Expected:** `Failed to resolve import "./ApplicationForm"`.

#### 🟢 Green

Create `frontend/src/components/ApplicationForm.tsx`:

```tsx
import { useState } from 'react'
import type { FormEvent } from 'react'
import type { NewApplication } from '../types'

type Props = {
  onSubmit: (application: NewApplication) => void
}

const EMPTY_FORM: NewApplication = {
  company: '',
  role: '',
  status: 'applied',
  applied_on: '',
}

export function ApplicationForm({ onSubmit }: Props) {
  const [form, setForm] = useState<NewApplication>(EMPTY_FORM)

  function updateField(field: keyof NewApplication, value: string) {
    setForm({ ...form, [field]: value })
  }

  function handleSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault()
    onSubmit(form)
  }

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Company
        <input
          value={form.company}
          onChange={(event) => updateField('company', event.target.value)}
          required
        />
      </label>
      <label>
        Role
        <input
          value={form.role}
          onChange={(event) => updateField('role', event.target.value)}
          required
        />
      </label>
      <label>
        Date applied
        <input
          type="date"
          value={form.applied_on}
          onChange={(event) => updateField('applied_on', event.target.value)}
          required
        />
      </label>
      <button type="submit">Add application</button>
    </form>
  )
}
```

- **Line 17, `useState<NewApplication>(EMPTY_FORM)`:** the form's
  state is one object holding every field. `<NewApplication>` tells
  TypeScript its shape.
- **Line 19, `keyof NewApplication`:** "one of the key names of
  `NewApplication`" — `'company' | 'role' | 'status' | 'applied_on'`.
  Pass `'compnay'` and TypeScript flags it.
- **Line 20, `{ ...form, [field]: value }`:** **spread** copies every
  field from the old state into a *new* object, then overwrites the one
  that changed. `[field]` uses the variable's value as the key.
- **Line 24, `event.preventDefault()`:** by default, submitting a form
  reloads the whole page. This stops that; React handles it instead.
- **Wrapping each `<input>` in its `<label>`** links them, so clicking
  the label focuses the field, screen readers announce it, and
  `getByLabelText` finds it.
- **`value` + `onChange`** make each input **controlled**: React's state
  is the single source of truth for what's in the box.
- **`required`** — the browser won't submit while a field is empty.

🟢 **Expected:** the test passes.

### Cycle 2 — clear after submit

#### 🔴 Red

**🔵 First, refactor the test file.** The next test needs the same
typing and clicking. Rather than copy nine lines, pull them into a
helper. Add this **above** the `describe` block, and add the extra
import:

```tsx
import type { UserEvent } from '@testing-library/user-event'
```

```tsx
async function fillAndSubmit(user: UserEvent) {
  await user.type(screen.getByLabelText('Company'), 'Acme')
  await user.type(
    screen.getByLabelText('Role'),
    'Junior Developer',
  )
  await user.type(
    screen.getByLabelText('Date applied'),
    '2026-10-01',
  )
  await user.click(
    screen.getByRole('button', { name: 'Add application' }),
  )
}
```

Replace the typing and clicking lines in the first test with
`await fillAndSubmit(user)`. The first test should still pass — that's
your proof the refactor was safe.

Now add the new test:

```tsx
  it('clears the fields after submitting', async () => {
    const user = userEvent.setup()
    render(<ApplicationForm onSubmit={vi.fn()} />)

    await fillAndSubmit(user)

    expect(screen.getByLabelText('Company')).toHaveValue('')
    expect(screen.getByLabelText('Role')).toHaveValue('')
  })
```

🔴 **Expected:** `Expected the element to have value: (empty) Received:
Acme`.

#### 🟢 Green

In `ApplicationForm.tsx`, add one line to `handleSubmit`, after
`onSubmit(form)` (line 25):

```tsx
    setForm(EMPTY_FORM)
```

🟢 **Expected:** both form tests pass.

Commit with message `feat(frontend): add ApplicationForm`.

---

## Step 4 — The API module

**Connect the dots.** The components are done, but nothing talks to the
backend yet. Each endpoint from lesson 05 needs a matching function:

| Function | Request |
| --- | --- |
| `fetchApplications()` | `GET /applications` |
| `createApplication(data)` | `POST /applications` |
| `updateStatus(id, status)` | `PATCH /applications/{id}` |
| `deleteApplication(id)` | `DELETE /applications/{id}` |

`fetch` is built into browsers. It returns a **Promise** of a
**Response**. `response.ok` is `true` for status codes 200–299.

Create `frontend/src/api.ts`:

```ts
import type { Application, NewApplication } from './types'

const API_URL = 'http://localhost:8000'

export async function fetchApplications(): Promise<Application[]> {
  const response = await fetch(`${API_URL}/applications`)
  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`)
  }
  return response.json()
}

export async function createApplication(
  application: NewApplication,
): Promise<Application> {
  const response = await fetch(`${API_URL}/applications`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(application),
  })
  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`)
  }
  return response.json()
}

export async function updateStatus(
  id: number,
  status: string,
): Promise<Application> {
  const response = await fetch(`${API_URL}/applications/${id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ status }),
  })
  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`)
  }
  return response.json()
}

export async function deleteApplication(id: number): Promise<void> {
  const response = await fetch(`${API_URL}/applications/${id}`, {
    method: 'DELETE',
  })
  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`)
  }
}
```

- **`Promise<Application[]>`** — "this returns, eventually, a list of
  applications."
- **`Content-Type: application/json`** tells FastAPI the body is JSON.
- **`JSON.stringify`** turns an object into JSON text.
- **`{ status }`** is shorthand for `{ status: status }`.
- The `if (!response.ok)` check is repeated four times. You'll remove
  that duplication in GRADUS II.

### Why no tests for this file?

Testing `api.ts` means faking the network — **mocking `fetch`** — which
is a whole technique of its own. That's the first new skill in GRADUS
II. For now, you'll verify it by hand in Step 6. Coverage Gutters will
show `api.ts` and `App.tsx` in red, and that's an honest picture: those
files are *not* automatically checked yet.

---

## Step 5 — Wire it up in `App`

Replace `frontend/src/App.tsx`:

```tsx
import { useEffect, useState } from 'react'
import {
  createApplication,
  deleteApplication,
  fetchApplications,
  updateStatus,
} from './api'
import { ApplicationForm } from './components/ApplicationForm'
import { ApplicationList } from './components/ApplicationList'
import type { Application, NewApplication } from './types'

const LOAD_ERROR =
  'Could not load applications. Is the backend running on port 8000?'

export default function App() {
  const [applications, setApplications] = useState<Application[]>([])
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    fetchApplications()
      .then((data) => setApplications(data))
      .catch(() => setError(LOAD_ERROR))
  }, [])

  async function handleCreate(newApplication: NewApplication) {
    const created = await createApplication(newApplication)
    setApplications([...applications, created])
  }

  async function handleDelete(id: number) {
    await deleteApplication(id)
    setApplications(applications.filter((item) => item.id !== id))
  }

  async function handleStatusChange(id: number, status: string) {
    const updated = await updateStatus(id, status)
    setApplications(
      applications.map((item) => (item.id === id ? updated : item)),
    )
  }

  return (
    <main>
      <h1>Job tracker</h1>
      {error && <p role="alert">{error}</p>}
      <ApplicationForm onSubmit={handleCreate} />
      <ApplicationList
        applications={applications}
        onDelete={handleDelete}
        onStatusChange={handleStatusChange}
      />
    </main>
  )
}
```

- **Lines 12–13:** the error message lives in a constant outside the
  component, so it's defined once and easy to find.
- **Lines 19–23:** load applications once when the page opens. `.then`
  runs when the data arrives; `.catch` runs if anything fails.
- **`string | null`** — the error is either a message or nothing.
- **Line 45, `{error && <p>...}`:** if `error` is `null`, render nothing;
  otherwise render the message. **`role="alert"`** makes screen readers
  announce it.
- **Each handler updates state with a *new* array:**
  - create: `[...applications, created]` — copy, then add to the end
  - delete: `.filter(...)` — keep every item *except* the deleted one
  - status: `.map(...)` — swap in the updated item, keep the rest

> **Known gap:** if create, delete, or status change fails, nothing
> tells the user. GRADUS II adds error handling for every action.

---

## Step 6 — Run it… and meet CORS

You need **two terminals** — one per server. In VS Code, click the
**split terminal** icon in the terminal panel.

**Terminal 1** (from the repo root):

```bash
cd backend
uv run alembic upgrade head
uv run fastapi dev app/main.py
```

**Terminal 2** (from the repo root):

```bash
cd frontend
npm run dev
```

Open <http://localhost:5173>.

❌ You'll see: **"Could not load applications. Is the backend running
on port 8000?"** But the backend *is* running.

Open the browser's **DevTools** (`F12`) → **Console** tab. You'll see a
red error containing **"blocked by CORS policy"** and **"No
'Access-Control-Allow-Origin' header"**.

### What's happening

An **origin** is scheme + host + port. Your frontend is at
`http://localhost:5173`; your backend is at `http://localhost:8000`.
Different ports → **different origins**.

By default, browsers block a page from reading responses from a
different origin. This is a security feature: without it, any website
you visit could quietly read data from your bank's site while you're
logged in.

**CORS** (Cross-Origin Resource Sharing) lets a server *opt in*: it sends
a header saying "I allow requests from this origin." The browser checks
that header before handing the response to your code.

The fix belongs on the **backend**. And it's behavior, so it gets a test
first.

### 🔴 Red (backend)

Stop the backend server (`Ctrl+C` in terminal 1). Create
`backend/tests/test_cors.py`:

```python
FRONTEND = "http://localhost:5173"


def test_frontend_origin_is_allowed(client):
    response = client.get("/health", headers={"Origin": FRONTEND})

    allowed = response.headers.get("access-control-allow-origin")
    assert allowed == FRONTEND


def test_other_origins_are_not_allowed(client):
    response = client.get(
        "/health",
        headers={"Origin": "http://unknown.example"},
    )

    assert "access-control-allow-origin" not in response.headers
```

- **`headers={"Origin": ...}`** imitates what a browser sends on a
  cross-origin request.
- The second test is a **guard**: it already passes, and it will fail if
  someone ever allows *every* origin.

```bash
cd backend
uv run pytest
```

🔴 **Expected:** the first test fails — `assert None ==
'http://localhost:5173'`. The second passes.

### 🟢 Green (backend)

In `backend/app/main.py`, add an import below the FastAPI import
(line 3):

```python
from fastapi.middleware.cors import CORSMiddleware
```

Then, directly below the line `app = FastAPI(title="Tiro Job Tracker")`,
add:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

- **Middleware** is code that runs on *every* request and response,
  before and after your routes. This one adds the CORS headers.
- **`allow_origins`** — only the frontend's origin. Never use `["*"]`
  (allow everyone) on an app that handles private data.
- **`allow_methods` / `allow_headers`** — allow `PATCH`, `DELETE`, and
  the `Content-Type` header our frontend sends.

```bash
uv run pytest
```

🟢 **Expected:** `15 passed`.

Start the backend again (`uv run fastapi dev app/main.py`) and reload
<http://localhost:5173>. The error is gone.

---

## Step 7 — Try it by hand

Work through this checklist in the browser. After each step, **reload
the page** to prove the change was saved in the database, not just on
screen.

- [ ] The empty message shows when there are no applications.
- [ ] Adding an application shows it in the list, and it survives a
      reload.
- [ ] The form clears after adding.
- [ ] Changing a status survives a reload.
- [ ] Deleting removes it, and it stays gone after a reload.
- [ ] Submitting with an empty field is blocked by the browser.

> **Two requests on page load?** In development, React's `StrictMode`
> (in `main.tsx`) runs effects twice on purpose, to help you spot bugs.
> It doesn't happen in a production build.

### Optional: a little layout

Add to `frontend/src/index.css`:

```css
form {
  display: grid;
  gap: 0.75rem;
  margin-bottom: 2rem;
}

label {
  display: grid;
  gap: 0.25rem;
}

ul {
  list-style: none;
  padding: 0;
}

li {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem 0;
  border-bottom: 1px solid #d0d7de;
}
```

---

## Step 8 — Commit

From the repo root:

```bash
git add .
git commit -m "feat: connect frontend to backend with CORS"
git push
```

---

## Explain it back

**1. Why do `ApplicationList` and `ApplicationForm` receive callbacks
instead of calling the API themselves?**

<details>
<summary>Answer</summary>

It keeps them simple and testable. A test can pass in `vi.fn()` and
check what was called, with no network involved. `App` owns the data and
decides what the callbacks do.
</details>

**2. Why does every state update create a *new* array or object instead
of changing the existing one?**

<details>
<summary>Answer</summary>

React compares the old and new values to decide whether to re-draw. If
you change the existing array in place, it's still the *same* array, so
React may not notice anything changed.
</details>

**3. Why did the CORS error appear even though both servers were
running?**

<details>
<summary>Answer</summary>

The frontend (port 5173) and backend (port 8000) are different origins.
The browser blocked the frontend from reading the backend's response
because the backend hadn't said that origin was allowed.
</details>

**4. What goes wrong with `onClick={onDelete(application.id)}`?**

<details>
<summary>Answer</summary>

It calls `onDelete` immediately while rendering (deleting everything as
soon as the list appears) and passes its return value as the click
handler. `onClick={() => onDelete(application.id)}` passes a function
that runs only on click.
</details>

**5. Why does `getByRole('button', { name: 'Delete Globex' })` work when
the button's visible text is only "Delete"?**

<details>
<summary>Answer</summary>

`aria-label` sets the button's accessible name to "Delete Globex."
Testing Library looks up buttons by accessible name, the same name a
screen reader announces.
</details>

---

## Checkpoint

- ✅ `npm run test:run` → `8 passed` (2 formatStatus, 4 list, 2 form)
- ✅ `uv run pytest` → `15 passed`
- ✅ Every item in the Step 7 checklist works

Docs for going deeper:

- React — managing state:
  <https://react.dev/learn/managing-state>
- React — `useEffect`:
  <https://react.dev/reference/react/useEffect>
- Testing Library — queries:
  <https://testing-library.com/docs/queries/about>
- user-event: <https://testing-library.com/docs/user-event/intro>
- MDN — CORS:
  <https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS>
- FastAPI — CORS: <https://fastapi.tiangolo.com/tutorial/cors/>

Next: [08 — Guardrails and debrief](08-guardrails-and-debrief.md)
