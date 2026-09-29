# 01 — Install your tools

**Goal:** every tool installed, verified, and ready. At the end, one
command per tool prints a version number.

If a tool is already installed, just run its check command and move on.

---

## Before you start: where to keep your code

Keep your repos in a folder that is **not** synced by OneDrive, iCloud,
Dropbox, or Google Drive. For example:

- Windows: `C:\Users\<you>\dev\`
- macOS / Linux: `~/dev/`

**Why:** `node_modules` and `.venv` contain tens of thousands of small
files. Sync tools try to upload all of them and can lock files mid-sync,
which causes random install failures. Your code is already backed up on
GitHub; sync adds nothing but trouble.

---

## 1. Git

**What it is:** version control. It records snapshots (commits) of your
code so you can see history and undo mistakes.

- **Windows:** install from <https://git-scm.com/downloads>. Accept the
  defaults. This also installs **Git Bash**, a terminal you'll use.
- **macOS:** run `git --version` in Terminal. If it's missing, macOS
  offers to install it.
- **Linux:** use your package manager (e.g. `sudo apt install git`).

**Check:**

```bash
git --version
```

✅ You should see something like `git version 2.x.x`.

---

## 2. VS Code

**What it is:** the code editor used throughout this guide.

Install from <https://code.visualstudio.com/>.

### Windows only: make Git Bash your terminal

The commands in this guide are written for a Bash-style terminal. On
Windows, set VS Code to use Git Bash so they work exactly as written:

1. Open VS Code.
2. Open the Command Palette: `Ctrl+Shift+P`.
3. Type **Terminal: Select Default Profile** and press Enter.
4. Choose **Git Bash**.
5. Open a new terminal: `` Ctrl+` `` (the key above Tab).

The terminal tab should now say `bash`.

---

## 3. Node.js and npm

**What it is:** Node.js runs JavaScript outside a browser. The frontend
tools (Vite, Vitest) run on it. **npm** comes with Node and installs
JavaScript packages.

Install the **LTS** ("long-term support") version from
<https://nodejs.org/>.

**Check (in a new terminal):**

```bash
node --version
npm --version
```

✅ `node` should print `v22` or higher. `npm` prints its own version.

> **"Command not found" right after installing?** Close and reopen your
> terminal (or all of VS Code). Terminals only read the list of
> installed programs when they start.

---

## 4. uv (and Python)

**What it is:** a tool that installs Python itself, creates a
**virtual environment** (an isolated folder of packages for one
project), and installs packages into it. It replaces `pip`, `venv`, and
`pyenv` with one fast tool.

**Why a virtual environment?** Two projects might need different
versions of the same package. Each project gets its own `.venv` folder
so they never collide.

- **Windows (PowerShell, not Git Bash):**

  ```powershell
  powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
  ```

  Or, if you use winget: `winget install --id=astral-sh.uv -e`

- **macOS / Linux:**

  ```bash
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```

Close and reopen your terminal, then **check:**

```bash
uv --version
```

✅ You should see `uv 0.x.x`.

Now install Python through uv:

```bash
uv python install 3.12
```

✅ uv downloads Python 3.12. You don't need to install Python any
other way.

Docs: <https://docs.astral.sh/uv/getting-started/installation/>

---

## 5. VS Code extensions

Open this repo's folder in VS Code (**File → Open Folder…**). VS Code
reads [`.vscode/extensions.json`](../.vscode/extensions.json) and shows
a popup: *"Do you want to install the recommended extensions?"* Click
**Install**.

If you missed the popup: open the Extensions sidebar
(`Ctrl+Shift+X` / `Cmd+Shift+X`), type `@recommended` in the search box,
and install everything under **Workspace Recommendations**.

What each one does for you:

| Extension | What you'll notice |
| --- | --- |
| **Python** | A test beaker icon in the sidebar; "Run" buttons above tests |
| **Pylance** | Autocomplete and red squiggles for Python type errors |
| **Ruff** | Warnings for unused imports, formats Python on save |
| **Even Better TOML** | Color highlighting in `pyproject.toml` |
| **ESLint** | Squiggles for TypeScript/React mistakes |
| **Prettier** | Formats TypeScript on save |
| **Vitest** | Frontend tests appear in the Testing sidebar |
| **Coverage Gutters** | Green/red bars beside each line showing test coverage |
| **SQLite Viewer** | Click a `.db` file to see its tables |
| **Pretty TypeScript Errors** | Turns long TypeScript errors into readable ones |

---

## Explain it back

**1. Why does each Python project get its own `.venv` folder?**

<details>
<summary>Answer</summary>

So each project's packages are isolated. Project A can use one version
of a library while Project B uses another, without conflict.
</details>

**2. You installed Node but `node --version` says "command not found".
What's the first thing to try?**

<details>
<summary>Answer</summary>

Close and reopen the terminal (or VS Code). The terminal only picks up
newly installed programs when it starts.
</details>

---

## Checkpoint

All five commands print a version:

```bash
git --version
node --version
npm --version
uv --version
uv python list --only-installed
```

✅ The last one lists a `cpython-3.12` entry.

Commit nothing yet — there's nothing to commit. Move on.

Next: [02 — Scaffold the repo](02-scaffold-repo.md)
