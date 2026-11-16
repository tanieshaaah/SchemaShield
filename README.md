# SchemaShield

**Detect PostgreSQL schema drift and pinpoint the Python code that could break because of it.**

SchemaShield compares a saved baseline schema with the current PostgreSQL database, scans your application code, identifies the Python files, functions, and SQL queries that may be affected, assigns severity levels, and produces a downloadable drift report.

**Tech stack:** Python 3.11+ · FastAPI · Streamlit · PostgreSQL 16 (Docker)

---

## Features

- **Schema drift detection**: compares a saved baseline against the live database and reports added, removed, renamed, and modified tables and columns
- **Code impact analysis**: scans Python files and statically detectable SQL to find the files, functions, and queries that depend on changed schema objects
- **Severity levels**: every finding is rated High, Medium, or Low
- **Downloadable report**: export the complete result as a JSON drift report
- **Interactive dashboard**: Streamlit UI backed by a FastAPI service
- **Reproducible demo**: the demo database is stored as SQL files in Git and recreated locally with Docker

## How it works

```mermaid
flowchart LR
    A[Baseline schema JSON] --> C[Compare schemas]
    B[Current PostgreSQL schema] --> C
    C --> D[Schema changes]
    D --> E[Scan Python code and SQL]
    E --> F[Affected code with severity]
    F --> G[Drift report JSON]
```

## Project structure

```text
SchemaShield/
├── backend/
│   └── api.py                      # FastAPI backend
├── frontend/
│   └── app.py                      # Streamlit dashboard
├── samples/
│   ├── database/
│   │   ├── baseline_schema.sql     # Original demo schema
│   │   ├── modified_schema.sql     # Changed demo schema
│   │   ├── expected_changes.json   # Expected schema changes
│   │   └── expected_impacts.json   # Expected code impacts
│   └── python_project/             # Sample Python + SQL code using the original schema
├── evaluation_report.json          # Recorded evaluation results
└── requirements.txt
```

---

## Requirements

- Git (optional if you download the ZIP)
- Python 3.11 or newer
- Docker Desktop

PostgreSQL does not need to be installed separately because the project uses PostgreSQL 16 through Docker. No GPU or HPC resources are required.

## Quick start

Download the project from GitHub (**Code → Download ZIP**, or clone it with Git) and open a terminal in the project folder, then run:

```bash
python -m venv .venv
# activate the environment (see step 2 for your OS), then:
python -m pip install -r requirements.txt
python -m pytest -q
```

Then follow the [full walkthrough](#full-demo-walkthrough) to start the database, backend, and dashboard.

---

## Full demo walkthrough

The repository contains everything needed for a controlled demonstration. The SQL files are the demo database stored in Git; a running PostgreSQL database or Docker container is **not** committed. Every teammate recreates the same database locally by loading these SQL files into a PostgreSQL Docker container.

Run all commands from the project folder.

### 1. Download or update the project

Download the project from GitHub using **Code → Download ZIP** (or clone it with Git), extract it, and open a terminal in the project folder. The folder may be named `SchemaShield-main` if you downloaded the ZIP.

If you already cloned the project, get the latest changes with:

```bash
git pull
```

### 2. Create the Python environment

**macOS / Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

**Windows (PowerShell)**

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

> If PowerShell blocks the activation script, run `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` and try again.

Install the required packages:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

When the environment is active, the terminal prompt normally contains `(.venv)`.

### 3. Verify the project

```bash
python -m pytest -q
```

Expected result:

```text
30 passed
```

A deprecation warning from the test libraries does not stop the project and can currently be ignored.

### 4. Start the demo PostgreSQL database

Open Docker Desktop and wait until Docker is running, then create the PostgreSQL container.

**macOS / Linux**

```bash
docker run \
  --name schemashield-db \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=schemashield_demo \
  -p 55432:5432 \
  -v "$PWD:/project:ro" \
  -d postgres:16-alpine
```

**Windows (PowerShell)**

```powershell
docker run `
  --name schemashield-db `
  -e POSTGRES_PASSWORD=postgres `
  -e POSTGRES_DB=schemashield_demo `
  -p 55432:5432 `
  -v "${PWD}:/project:ro" `
  -d postgres:16-alpine
```

> **Demo only:** the `postgres` password above is for the local demo database. Never use it for a real database.

The first run downloads PostgreSQL and may take a few minutes. Successful output is a long container ID.

Check that PostgreSQL is ready:

```bash
docker exec schemashield-db pg_isready -U postgres -d schemashield_demo
```

Expected result:

```text
/var/run/postgresql:5432 - accepting connections
```

If Docker reports that the container name already exists, start the existing container instead of creating a new one, then run the readiness check again:

```bash
docker start schemashield-db
```

### 5. Load the original database schema

Load the baseline schema **before** capturing the baseline in SchemaShield.

**macOS / Linux**

```bash
docker exec schemashield-db \
  psql -v ON_ERROR_STOP=1 \
  -U postgres \
  -d schemashield_demo \
  -f /project/samples/database/baseline_schema.sql
```

**Windows (PowerShell)**

```powershell
docker exec schemashield-db `
  psql -v ON_ERROR_STOP=1 `
  -U postgres `
  -d schemashield_demo `
  -f /project/samples/database/baseline_schema.sql
```

Expected output contains `DROP SCHEMA`, `CREATE SCHEMA`, `SET`, and three `CREATE TABLE` lines. A notice that the schema does not exist is normal on the first run. The baseline contains the `customers`, `orders`, and `products` tables.

### 6. Start the FastAPI backend

Keep the database running. In **Terminal 1**, from the project folder:

```bash
# macOS / Linux: source .venv/bin/activate
# Windows:       .venv\Scripts\Activate.ps1
python -m uvicorn backend.api:app --reload
```

Expected output includes:

```text
Application startup complete
Uvicorn running on http://127.0.0.1:8000
```

Verify the backend by opening <http://127.0.0.1:8000/health>. The response should be:

```json
{"status":"ok"}
```

Keep this terminal running.

### 7. Start the Streamlit dashboard

Open **Terminal 2**, enter the project folder, activate the environment, and run:

```bash
python -m streamlit run frontend/app.py
```

Expected output includes:

```text
Local URL: http://localhost:8501
```

Open <http://localhost:8501> if the dashboard does not open automatically. Keep this terminal running.

### 8. Capture the baseline

Enter these values in the dashboard:

| Field | Value |
|---|---|
| FastAPI URL | `http://localhost:8000` |
| PostgreSQL URL | `postgresql+psycopg://postgres:postgres@localhost:55432/schemashield_demo` |
| Schema name | `schemashield_demo` |
| Python project folder | `samples/python_project` |
| Baseline JSON file | `baseline.json` |

Click **Capture baseline**. Expected message:

```text
Saved 3 tables to .../SchemaShield/baseline.json
```

SchemaShield has now saved the original database structure. **Do not capture the baseline again after loading the modified schema.**

### 9. Apply the controlled database changes

Open **Terminal 3** and run, from the project folder:

**macOS / Linux**

```bash
docker exec schemashield-db \
  psql -v ON_ERROR_STOP=1 \
  -U postgres \
  -d schemashield_demo \
  -f /project/samples/database/modified_schema.sql
```

**Windows (PowerShell)**

```powershell
docker exec schemashield-db `
  psql -v ON_ERROR_STOP=1 `
  -U postgres `
  -d schemashield_demo `
  -f /project/samples/database/modified_schema.sql
```

Expected output again contains `DROP SCHEMA`, `CREATE SCHEMA`, `SET`, and three `CREATE TABLE` lines. A `drop cascades to 3 other objects` notice is normal.

The modified schema intentionally makes these nine changes:

| # | Change |
|---|---|
| 1 | Adds the `audit_logs` table |
| 2 | Removes the `products` table |
| 3 | Renames `customers.full_name` to `customers.name` |
| 4 | Removes `customers.phone` |
| 5 | Changes the maximum length of `customers.email` from 150 to 200 |
| 6 | Adds `customers.is_active` |
| 7 | Renames `orders.order_total` to `orders.total_amount` |
| 8 | Changes the maximum length of `orders.status` from 20 to 30 |
| 9 | Adds `orders.updated_at` |

### 10. Run drift detection

Return to the Streamlit dashboard and click **Run drift detection**.

The **Schema changes** section should show all nine changes listed above (display order may differ).

The **Affected code** section should include:

| Files | Affected Python functions |
|---|---|
| `customer_repository.py` | `get_customer_profile`, `create_customer` |
| `order_repository.py` | `list_customer_orders`, `mark_order_complete` |
| `product_repository.py` | `list_available_products` |
| `report_queries.sql` | (SQL file) |

Expected severity totals:

| Severity | Findings |
|---|---|
| High | 4 |
| Medium | 2 |
| Low | 3 |

The controlled test contains 15 expected database-to-code impact links. The sample project uses valid static SQL, so an empty warnings list is a correct result.

Click **Download drift report** to download the complete result as JSON.

### 11. View the evaluation results

```bash
cat evaluation_report.json        # macOS / Linux
type evaluation_report.json       # Windows
```

Key controlled-test results:

| Metric | Result |
|---|---|
| `evaluation_passed` | `true` |
| True positives | 9 |
| False positives | 0 |
| False negatives | 0 |
| Accuracy / Precision / Recall | 1.0 / 1.0 / 1.0 |
| False positive rate | 0.0 |
| Matched impact links | 15 of 15 |

> These measurements apply to the supplied controlled test cases and should not be read as universal accuracy on every real-world project.

### 12. Verify the no-change result

Restore the original schema by re-running the step 5 command, then click **Run drift detection** again **without** capturing a new baseline. The dashboard should show:

```text
No schema drift detected
```

This confirms that SchemaShield does not report changes when the baseline and current database match.

### 13. Stop and restart the project

Press `Ctrl + C` in the FastAPI and Streamlit terminals, then stop PostgreSQL:

```bash
docker stop schemashield-db
```

For the next demonstration, start the same container again:

```bash
docker start schemashield-db
```

Load `baseline_schema.sql` again before beginning a new demonstration, then start FastAPI, start Streamlit, and capture a fresh baseline.

### Optional cleanup

Remove the stopped demo container only if it is no longer needed:

```bash
docker rm schemashield-db
```

Removing the container does not remove the SQL files stored in Git. Running the Docker setup command recreates the same demo database.

---

## Troubleshooting

<details>
<summary><code>ModuleNotFoundError: No module named 'backend'</code></summary>

Run commands from the project root and start pytest through Python:

```bash
cd SchemaShield
python -m pytest -q
```

</details>

<details>
<summary><code>Cannot connect to the Docker daemon</code></summary>

Open Docker Desktop and wait until it finishes starting.

</details>

<details>
<summary>The container name <code>schemashield-db</code> is already in use</summary>

Start the existing container:

```bash
docker start schemashield-db
```

</details>

<details>
<summary>Port <code>55432</code> is already in use</summary>

Choose another unused port, such as `55433`, in the `-p` option of the `docker run` command (`-p 55433:5432`), then use port `55433` in the dashboard PostgreSQL URL.

</details>

<details>
<summary>The dashboard cannot reach FastAPI</summary>

Confirm the backend terminal is still running and that <http://127.0.0.1:8000/health> returns `{"status":"ok"}`.

</details>

<details>
<summary>Baseline capture reports zero tables</summary>

Load `baseline_schema.sql` again and make sure the dashboard schema name is exactly `schemashield_demo`.

</details>

<details>
<summary>The dashboard shows unexpected changes</summary>

Restore `baseline_schema.sql`, capture a new baseline, apply `modified_schema.sql`, and run drift detection again, in that order.

</details>

---

## Project scope and limitations

- PostgreSQL is the supported database.
- Python source code and statically detectable SQL are supported.
- Dynamic and ORM-generated SQL may not be fully resolved.
- Rename detection is heuristic and can be ambiguous in some cases.
- SchemaShield reports *potentially* affected code; it does **not** automatically modify application code or perform database migrations.
