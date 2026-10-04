# 07 — Frontend: first test

> **Where this fits.** The backend can now do everything in US-1
> through US-4 — but only through `/docs`. A job seeker needs a real
> screen: a form to record applications and a list to see them. That's
> the **frontend**, the left-hand box in the lesson 00 diagram. This
> lesson sets up its toolchain and proves it with one small test; lesson
> 08 builds the screens.

**Goal:** a React + TypeScript app scaffolded with Vite, Vitest and React
Testing Library installed and configured, your first frontend red →
green cycle, coverage in the editor, and frontend tests in your
pre-commit hook.

---

## Concepts first

| Term | Plain English |
| --- | --- |
| **React** | A library for building a UI out of **components** — functions that return what should appear on screen |
| **JSX / TSX** | HTML-like syntax inside JavaScript/TypeScript. `.tsx` files are TypeScript files that contain JSX |
| **TypeScript** | JavaScript plus **types**. It checks, before your code runs, that you're not passing a number where a string belongs |
| **Vite** | Runs a dev server that reloads your page instantly on save, and builds the final app |
| **Vitest** | A test runner built on Vite — it understands TypeScript and JSX with no extra setup. The frontend's pytest |
| **jsdom** | A fake browser that runs in Node, so tests can render components without opening a real browser |
| **React Testing Library (RTL)** | Renders components in jsdom and lets you find things the way a *user* would — by visible text, labels, and roles |

The backend and frontend toolchains line up neatly — useful to keep in
mind:

| Job | Backend | Frontend |
| --- | --- | --- |
| Manage packages | uv (`pyproject.toml`, `uv.lock`) | npm (`package.json`, `package-lock.json`) |
| Run tests | pytest | Vitest |
| Measure coverage | pytest-cov → `coverage.xml` | `@vitest/coverage-v8` → `lcov.info` |
| Lint / format | Ruff | ESLint / Prettier |
| Dev server | `fastapi dev` (port 8000) | `npm run dev` (port 5173) |

---

## Step 1 — Scaffold with Vite

From the **repo root**:

```bash
npm create vite@latest frontend -- --template react-ts
```

- **`npm create vite@latest`** runs Vite's project generator.
- **`frontend`** is the folder name.
- **`--`** passes the rest to the generator (not to npm).
- **`--template react-ts`** chooses React with TypeScript.

If it asks extra questions (for example, about an experimental bundler,
or whether to install and start now), answer **No** — you'll do those
steps yourself so you see what each one does.

```bash
cd frontend
npm install
npm run dev
```

✅ The terminal shows `Local: http://localhost:5173/`. Open it: a Vite +
React page with a counter button. Stop the server with `Ctrl+C`.

**From now on in this lesson, stay in `frontend/`.**

### A quick tour

| File | Job |
| --- | --- |
| `frontend/index.html` | The one HTML page. React fills in the `<div id="root">` |
| `frontend/src/main.tsx` | Entry point: finds `#root` and renders `<App />` into it |
| `frontend/src/App.tsx` | The top-level component |
| `frontend/package.json` | Dependencies and **scripts** (`dev`, `build`, `lint`…) |
| `frontend/vite.config.ts` | Vite's settings — you'll add test settings here |
| `frontend/tsconfig*.json` | TypeScript's settings |
| `frontend/eslint.config.js` | Linting rules |

---

## Step 2 — Clean out the demo

**1.** Delete `frontend/src/App.css` and the `frontend/src/assets/`
folder.

**2.** 📄 **File:** `frontend/src/App.tsx` — **edit**: replace the
whole file with:

```tsx
export default function App() {
  return (
    <main>
      <h1>Job tracker</h1>
    </main>
  )
}
```

- A component is just a function that returns JSX.
- **`export default`** makes `App` the file's main export, which is how
  `main.tsx` imports it.

**3.** 📄 **File:** `frontend/src/index.css` — **edit**: replace the
whole file with a small readable baseline:

```css
:root {
  font-family: system-ui, sans-serif;
  line-height: 1.5;
  color: #1f2328;
  background: #ffffff;
}

body {
  margin: 0;
}

main {
  max-width: 40rem;
  margin: 0 auto;
  padding: 2rem 1rem;
}
```

**4.** 📄 **File:** `frontend/.prettierrc` — **new**. It matches Prettier
to Vite's code style (single quotes, no semicolons):

```json
{
  "semi": false,
  "singleQuote": true
}
```

Without this, format-on-save would rewrite every file with double
quotes and semicolons, and your code would stop matching this guide.

Run `npm run dev` again: the page just says **Job tracker**. Stop it.

---

## Step 3 — Install the test tools

Still in `frontend/`:

```bash
npm install -D vitest @vitest/coverage-v8 jsdom
```

```bash
npm install -D @testing-library/react @testing-library/dom
```

```bash
npm install -D @testing-library/user-event @testing-library/jest-dom
```

(`-D` is short for `--save-dev`.)

| Package | What it adds |
| --- | --- |
| `vitest` | The test runner |
| `@vitest/coverage-v8` | Coverage measurement |
| `jsdom` | The fake browser |
| `@testing-library/react` | `render` and `screen` for components |
| `@testing-library/dom` | The query engine RTL is built on |
| `@testing-library/user-event` | Realistic clicks and typing |
| `@testing-library/jest-dom` | Extra assertions like `toBeInTheDocument()` |

---

## Step 4 — Configure Vitest

📄 **File:** `frontend/vite.config.ts` — **edit**. It looks roughly
like this now (leave the `plugins` line exactly as Vite generated it):

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

Make two changes:

1. Add a **reference line** as the very first line of the file.
2. Add a **`test` block** after the `plugins` line.

The file becomes:

```ts
/// <reference types="vitest/config" />
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: './src/setupTests.ts',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      include: ['src/**/*.{ts,tsx}'],
      exclude: [
        'src/main.tsx',
        'src/setupTests.ts',
        'src/**/*.test.{ts,tsx}',
      ],
    },
  },
})
```

- **Line 1** tells TypeScript that `defineConfig` accepts a `test`
  section. Without it, you'd get a red squiggle under `test`.
- **Line 8, `environment: 'jsdom'`** — run tests in the fake browser.
- **Line 9, `setupFiles`** — a file that runs before every test file.
- **Line 12, `reporter: ['text', 'lcov']`** — print a table *and* write
  `coverage/lcov.info`, which Coverage Gutters reads.
- **Lines 13–18, `include` / `exclude`** — measure your source files, but
  not the entry point, the setup file, or the tests themselves.

### The setup file

📄 **File:** `frontend/src/setupTests.ts` — **new**:

```ts
import '@testing-library/jest-dom/vitest'
import { cleanup } from '@testing-library/react'
import { afterEach } from 'vitest'

afterEach(() => {
  cleanup()
})
```

- **Line 1** adds jest-dom's matchers (like `toBeInTheDocument`) to
  Vitest's `expect`.
- **Lines 5–7** remove whatever a test rendered after it finishes, so the
  next test starts with a blank page. (RTL does this automatically only
  if Vitest's "globals" mode is on. We import everything explicitly
  instead, so you can always see where a function comes from.)

### Scripts

📄 **File:** `frontend/package.json` — **edit**: inside `"scripts"`,
add three lines (keep the existing ones):

```json
"test": "vitest",
"test:run": "vitest run",
"coverage": "vitest run --coverage"
```

| Script | Does |
| --- | --- |
| `npm test` | **Watch mode**: runs tests, then re-runs them every time you save. Stop with `q` or `Ctrl+C` |
| `npm run test:run` | Runs once and exits. Used by the Git hook |
| `npm run coverage` | Runs once with coverage |

> **JSON is strict.** Every line in `"scripts"` except the last needs a
> comma after it. A missing or extra comma breaks every `npm` command.

---

## Step 5 — 🔴 Red: the first frontend test

**Connect the dots.**

- The backend stores statuses in lowercase: `applied`, `interviewing`.
- The UI should *display* them capitalized: `Applied`, `Interviewing` —
  that's part of US-2 (seeing your applications clearly).
- That's a **pure function** — same input, same output, nothing else
  touched. Pure functions are the easiest things to test, so they're a
  good first step.

*Before reading on: which two or three inputs would you test?*

📄 **File:** `frontend/src/utils/formatStatus.test.ts` — **new** (create
the `utils/` folder too):

```ts
import { describe, expect, it } from 'vitest'
import { formatStatus } from './formatStatus'

describe('formatStatus', () => {
  it('capitalizes the first letter', () => {
    expect(formatStatus('applied')).toBe('Applied')
  })

  it('leaves an empty string empty', () => {
    expect(formatStatus('')).toBe('')
  })
})
```

- **`describe`** (line 4) groups related tests under a name.
- **`it`** (lines 5 and 9) is one test. Read it as a sentence: "*it*
  capitalizes the first letter."
- **`expect(actual).toBe(expected)`** is the assertion — the frontend's
  `assert`.
- **Line 2** imports a file that doesn't exist yet.

```bash
npm test
```

🔴 **Expected:** `Failed to resolve import "./formatStatus"`. Leave watch
mode running.

📄 **File:** `frontend/src/utils/formatStatus.ts` — **new**, with a
deliberately *wrong* version, so you see the test fail for the right
reason:

```ts
export function formatStatus(status: string): string {
  return status
}
```

- **`export`** makes the function importable from other files.
- **`(status: string): string`** — takes a string, returns a string.
  That's TypeScript's type annotation, like Python's type hints.

Save. Watch mode re-runs automatically.

🔴 **Expected:** the first test fails —
`expected 'applied' to be 'Applied'`. The second passes (an empty string
is still empty).

---

## Step 6 — 🟢 Green

📄 **File:** `frontend/src/utils/formatStatus.ts` — **edit**: replace
line 2 so the file reads:

```ts
export function formatStatus(status: string): string {
  return status.charAt(0).toUpperCase() + status.slice(1)
}
```

- **`charAt(0)`** — the first character (`'a'`), or `''` for an empty
  string.
- **`.toUpperCase()`** — `'A'`.
- **`slice(1)`** — everything from index 1 on: `'pplied'`.

Save.

🟢 **Expected:** `2 passed`. Press `q` to leave watch mode.

---

## Step 7 — Coverage and the Vitest extension

```bash
npm run coverage
```

✅ A coverage table prints, and `frontend/coverage/lcov.info` is
created. Open `frontend/src/utils/formatStatus.ts` with Coverage Gutters
watching (or run **Coverage Gutters: Display Coverage**): green lines.

> **`App.tsx` shows 0%?** Expected — no test renders it yet. Lesson 08
> explains why `App.tsx` stays untested in Tiro, and GRADUS II teaches
> the technique that covers it.

Now open the **Testing** sidebar in VS Code (the beaker icon). You'll see
both the backend tests (from the Python extension) and the frontend
tests (from the Vitest extension). You can run any single test from
there with its ▶ button.

---

## Step 8 — Add frontend tests to the pre-commit hook

📄 **File:** `.husky/pre-commit` (repo root) — **edit**: add the last
two lines, so the whole file reads:

```sh
echo "Starting the database..."
docker compose up -d --wait

echo "Running backend tests..."
(cd backend && uv run pytest -q)

echo "Running frontend tests..."
(cd frontend && npm run test:run)
```

---

## Step 9 — Commit

```bash
cd ..
git add .
git commit -m "feat(frontend): scaffold Vite app with Vitest"
git push
```

✅ Both test suites run before the commit is created.

Check `git status` afterward: `frontend/node_modules/` and
`frontend/coverage/` should **not** have been committed.

---

## Explain it back

**1. What's the difference between `npm test` and `npm run test:run`,
and why does the Git hook use the second one?**

<details>
<summary>Answer</summary>

`npm test` runs Vitest in watch mode — it never exits on its own, it
waits for file changes. The hook needs a command that runs once and
exits with pass/fail, so it uses `vitest run`.
</details>

**2. Why is `formatStatus` a good *first* thing to test?**

<details>
<summary>Answer</summary>

It's a pure function: output depends only on input, and it doesn't
touch the screen, the network, or anything else. No setup is needed —
just call it and check the result.
</details>

**3. What does jsdom let your tests do?**

<details>
<summary>Answer</summary>

Render components and interact with a DOM (the page structure) without
opening a real browser, so tests run fast inside Node.
</details>

**4. Why does `setupTests.ts` call `cleanup()` after each test?**

<details>
<summary>Answer</summary>

To remove what the previous test rendered, so each test starts with an
empty page and can't accidentally find elements left over from another
test. (The backend equivalent: the `engine` fixture dropping tables.)
</details>

---

## Checkpoint

- ✅ `npm run test:run` (in `frontend/`) → `2 passed`
- ✅ `npm run dev` shows **Job tracker**
- ✅ Committing starts the database and runs backend *and* frontend
  tests

Docs for going deeper:

- Vite guide: <https://vite.dev/guide/>
- Vitest guide: <https://vitest.dev/guide/>
- React — describing the UI: <https://react.dev/learn/describing-the-ui>
- TypeScript for JavaScript programmers:
  <https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html>

Next: [08 — Components with TDD](08-components.md)
