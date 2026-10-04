# 04 — Docker and PostgreSQL

> **Where this fits.** **US-5**: *"As a job seeker, I want my
> applications to still be there tomorrow."* Data kept in the backend's
> memory disappears every time the server restarts. To keep it, you need
> a **database** — a program built to store data safely on disk. This
> lesson gets that database running. In the big-picture diagram from
> lesson 00, it's the box on the far right.

**Goal:** PostgreSQL running in a Docker container, a separate test
database beside it, and enough SQL and `psql` to look inside both.

Work at the **repo root** for this whole lesson.

---

## Concepts first

### Why a database server?

A **database** stores data so it survives restarts, lets many
requests read and write at once without corrupting anything, and answers
questions about the data quickly ("every application with status
*offer*").

**PostgreSQL** (often "Postgres") is a free, widely used, industrial-
strength database. It's a **server**: a program that runs continuously
and accepts connections, just like your FastAPI backend does — but it
speaks **SQL** instead of HTTP.

### Why Docker?

Installing PostgreSQL directly on your computer works, but every
operating system does it differently, versions clash between projects,
and "works on my machine" becomes a real problem. **Docker** runs
PostgreSQL inside a **container**: a sealed box with the program and
everything it needs, identical on Windows, macOS, and Linux.

| Term | Plain English |
| --- | --- |
| **Image** | A packaged, read-only snapshot of software — e.g. `postgres:17`. Downloaded once from Docker Hub, the public library of images. |
| **Container** | A *running* instance of an image. You can start, stop, and delete containers; the image doesn't change. Think: the image is the recipe, the container is the meal. |
| **Volume** | Storage that lives *outside* the container, managed by Docker. Delete the container and the volume — your data — survives. |
| **Port mapping** | Containers are sealed, so their ports aren't reachable from your computer unless you **map** them: `"5433:5432"` means "port 5433 on my computer leads to port 5432 inside the container." |
| **Docker Compose** | A file (`compose.yaml`) that describes the containers a project needs, so one command starts them all the same way every time. |

---

## Step 1 — Describe the database in `compose.yaml`

**Connect the dots.** To run PostgreSQL, Docker needs to know:

- **which image** to run (and which version);
- **the username, password, and database name** to create;
- **which port** on your computer should lead to it;
- **where to keep the data** so it survives;
- **how to tell when it's ready** to accept connections.

*Before reading on: which of these would you expect to change if two
projects on one computer each needed their own PostgreSQL?*

📄 **File:** `compose.yaml` — **new**, at the **repo root**:

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: tiro
      POSTGRES_PASSWORD: tiro
      POSTGRES_DB: tiro
    ports:
      - "5433:5432"
    volumes:
      - tiro-data:/var/lib/postgresql/data
      - ./docker/postgres-init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tiro -d tiro"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  tiro-data:
```

Line by line:

- **Line 1, `services:`** — the list of containers this project needs.
  Just one here.
- **Line 2, `db:`** — the service's name. You'll type it in commands
  (`docker compose logs db`).
- **Line 3, `image: postgres:17`** — the official PostgreSQL image,
  **pinned** to major version 17. `postgres:latest` would silently jump
  to new versions; newer versions can store data differently and refuse
  to start on your old volume.
- **Lines 4–7, `environment:`** — settings the image reads **the first
  time it starts**: create a user `tiro` with password `tiro`, and a
  database named `tiro`. This is your **development database**.
- **Lines 8–9, `ports:`** — `"5433:5432"`: PostgreSQL listens on 5432
  inside the container; you reach it on **5433** on your computer. We
  use 5433 (not the usual 5432) so it can't collide with a PostgreSQL
  you might already have installed. The later GRADUS tiers use 5434 and
  5435, so all three can run at once.
- **Line 11** — mounts the **named volume** `tiro-data` where PostgreSQL
  keeps its files. This is what makes data survive (US-5).
- **Line 12** — mounts your repo's `docker/postgres-init/` folder into a
  special folder in the container. Any `.sql` file there runs **once**,
  when the database is first created. `:ro` means read-only. You'll use
  it in Step 2.
- **Lines 13–17, `healthcheck:`** — every 5 seconds, Docker runs
  `pg_isready` (a PostgreSQL tool) inside the container to ask "are you
  accepting connections?" Until it says yes, the container is
  **starting**, not **healthy**.
- **Lines 19–20** — declares the `tiro-data` volume so Docker creates
  and manages it.

> 🔒 **These credentials are for a local, throwaway development
> database only.** `tiro`/`tiro` is fine on your laptop and would be a
> serious mistake anywhere real. GRADUS II moves settings like these out
> of code.

---

## Step 2 — A separate test database

**Connect the dots.** Your tests will create data, delete it, and wipe
tables between tests. Pointing them at your **development** database
would destroy anything you'd entered by hand. Tests get their **own
database** on the same server: `tiro_test`.

The image only creates one database from `POSTGRES_DB`. To create a
second, use the init folder from line 12.

📄 **File:** `docker/postgres-init/01-create-test-database.sql` —
**new** (create the `docker/` and `docker/postgres-init/` folders too):

```sql
-- Runs once, the first time the database container starts
-- with an empty volume. Creates the database the tests use.
CREATE DATABASE tiro_test;
```

- **Lines 1–2** are SQL **comments** (they start with `--`).
- **Line 3** is a SQL **statement**. SQL keywords are traditionally
  written in capitals; statements end with `;`.
- The `01-` prefix: init scripts run in alphabetical order. Numbering
  them keeps the order obvious if you add more later.

---

## Step 3 — Start it

```bash
docker compose up -d --wait
```

| Part | Meaning |
| --- | --- |
| `docker compose up` | Create and start every service in `compose.yaml` |
| `-d` | **Detached**: run in the background and give you your terminal back |
| `--wait` | Don't return until every service is **healthy** |

✅ The first time, Docker downloads the `postgres:17` image (a minute or
two). The command ends with the `db` container reported as **Healthy**.

Look at it:

```bash
docker compose ps
```

✅ One row: the `db` service, with **STATUS** showing `Up … (healthy)`
and **PORTS** showing `0.0.0.0:5433->5432/tcp` — your port mapping.

See what PostgreSQL has been saying:

```bash
docker compose logs db
```

✅ Near the end: `database system is ready to accept connections`.
Earlier, you'll see a line mentioning your
`01-create-test-database.sql` — proof the init script ran.

---

## Step 4 — Look inside with `psql`

**`psql`** is PostgreSQL's command-line client: you type SQL, it shows
results. It's already inside the container, so you don't install
anything:

```bash
docker compose exec db psql -U tiro -d tiro
```

| Part | Meaning |
| --- | --- |
| `docker compose exec db` | Run a command **inside** the running `db` container |
| `psql` | The command to run |
| `-U tiro` | Connect as user `tiro` |
| `-d tiro` | Connect to the database `tiro` |

✅ Your prompt changes to `tiro=#`. You're now talking to PostgreSQL.

`psql` has two kinds of commands:

- **SQL statements**, which end with `;`
- **Backslash commands** (meta-commands), which don't

| Command | Does |
| --- | --- |
| `\l` | List databases |
| `\dt` | List tables in the current database |
| `\d tablename` | Describe one table's columns |
| `\c dbname` | Connect to a different database |
| `\q` | Quit `psql` |

Try `\l`. ✅ Among others, you'll see **`tiro`** and **`tiro_test`**.

### A five-minute SQL experiment

You'll rarely write raw SQL in this project — SQLAlchemy writes it for
you — but seeing a little makes everything later less mysterious. Type
each statement (end each with `;`):

```sql
CREATE TABLE scratch (id SERIAL PRIMARY KEY, note TEXT);
INSERT INTO scratch (note) VALUES ('hello'), ('postgres');
SELECT * FROM scratch;
```

- **`CREATE TABLE`** defines a table with two **columns**. `SERIAL
  PRIMARY KEY` means "a whole number that PostgreSQL fills in
  automatically — 1, 2, 3… — and that uniquely identifies each row."
- **`INSERT`** adds **rows**.
- **`SELECT *`** reads every column of every row.

✅ A small table with ids 1 and 2.

Run `\dt` — ✅ `scratch` is listed. Then quit with `\q`.

---

## Step 5 — Prove the data survives (US-5)

Stop and **remove** the container:

```bash
docker compose down
```

✅ The container is gone (`docker compose ps` shows nothing). The
**volume** is not — `down` keeps volumes unless you add `-v`.

Start it again and look:

```bash
docker compose up -d --wait
docker compose exec db psql -U tiro -d tiro -c "SELECT * FROM scratch;"
```

(`-c "…"` runs one statement and exits, instead of opening the prompt.)

✅ Both rows are still there. The container was brand new; the data
lived in the volume. **That's US-5, at the infrastructure level.**

Clean up the experiment:

```bash
docker compose exec db psql -U tiro -d tiro -c "DROP TABLE scratch;"
```

| Command | Effect on data |
| --- | --- |
| `docker compose stop` | Pauses the container. Data kept. |
| `docker compose down` | Removes the container. **Data kept** (in the volume). |
| `docker compose down -v` | Removes the container **and the volume**. **Data deleted.** Init scripts run again on the next `up`. |

> **When to use `down -v`:** only when you deliberately want a fresh,
> empty database — for example, if you change the init script. Never
> by habit.

---

## Step 6 — Commit

```bash
git add .
git commit -m "chore: run PostgreSQL in Docker with a test database"
git push
```

✅ `compose.yaml` and `docker/postgres-init/01-create-test-database.sql`
are committed. The database's **data** isn't in the repo — it's in
Docker's volume, private to your machine.

---

## Explain it back

**1. What's the difference between an image and a container?**

<details>
<summary>Answer</summary>

An image is a read-only package (the recipe). A container is a running
instance of it (the meal). You can delete and recreate containers from
the same image as often as you like.
</details>

**2. You ran `docker compose down`, then `up`. Why was your data still
there?**

<details>
<summary>Answer</summary>

PostgreSQL's files live in the named volume `tiro-data`, which `down`
doesn't remove. The new container mounted the same volume and found the
data. Only `down -v` deletes volumes.
</details>

**3. Why does the test database exist, and how was it created?**

<details>
<summary>Answer</summary>

So tests can create and wipe data freely without touching the
development database. It was created by
`docker/postgres-init/01-create-test-database.sql`, which the image ran
automatically the first time the database started with an empty volume.
</details>

**4. What does `"5433:5432"` mean, and why 5433?**

<details>
<summary>Answer</summary>

Port 5433 on your computer leads to port 5432 (PostgreSQL's standard
port) inside the container. Using 5433 avoids a clash with any
PostgreSQL already installed on 5432, and leaves 5434/5435 for the other
GRADUS tiers.
</details>

---

## Checkpoint

- ✅ `docker compose ps` shows `db` as **healthy**
- ✅ `\l` in `psql` lists both `tiro` and `tiro_test`
- ✅ Data survived a `down` / `up`

> **Every time you sit down to work** on this project, start the
> database first: `docker compose up -d --wait`. (In lesson 05 the
> pre-commit hook will start it for you, too.) Docker Desktop must be
> running.

Docs for going deeper:

- Docker overview: <https://docs.docker.com/get-started/docker-overview/>
- Compose quickstart: <https://docs.docker.com/compose/gettingstarted/>
- The `postgres` image (see "Initialization scripts"):
  <https://hub.docker.com/_/postgres>
- `psql`: <https://www.postgresql.org/docs/current/app-psql.html>
- SQL tutorial: <https://www.postgresql.org/docs/current/tutorial-sql.html>

Next: [05 — Async database layer and migrations](05-database-and-migrations.md)
