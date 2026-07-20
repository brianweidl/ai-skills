---
name: database-example-skill
category: infrastructure
description: Run SQL queries against the MySQL databases of your apps, across environments (local Docker / staging / prod), with hard guardrails suited to an AI agent. Reads on all envs; writes only on staging (with explicit per-statement confirmation); prod is SELECT-only (double-gated: agent + wrapper). Excludes PII vault DBs by default. Returns a markdown table plus a structured file the agent can process. Connects via a SQL proxy (staging master / prod read replica) using an OS-keychain-stored credential. Use when the user says "/database-example-skill", "query the DB", "query app X's database", "get me the orders/payments/records for a day", "SQL to staging", "SQL to prod", "run a query against <app>'s database", "check data in the database". THIS IS A GENERIC EXAMPLE SKILL — all hosts, ports, schemas and app names are placeholders; adapt them to your own infrastructure before use.
---

# Database (example) — SQL queries against your apps' databases

Runs SQL against the MySQL databases of your apps, in local / staging / prod. It is designed for use by an agent: **the guardrails are hard — they do not rely on operator discipline.**

> **This is a generic, redistributable example.** Every host, port, schema, credential name and app name below is a **placeholder** (`db-master.internal.example.com`, `ORDERSSTG`, `orders-service`, …). Replace them with your real infrastructure. It exists to demonstrate *how* an agentic skill is structured, not to run as-is.

**Before anything**, read these references:
- `references/db-access.md` — connection substrate (8.4 client, keychain, wrapper, proxy hosts). Setup and troubleshooting.
- `references/app-registry.md` — map app → env → {host, schema, vault?, segmentation}. This is the source for resolving the connection.
- `references/example-queries.md` — ready-to-run queries for the base case: orders + payments + validations for a single day (orders-service).

---

## Prerequisites (verify on first use)

1. **Corporate VPN** connected (if your DBs live behind one).
2. **MySQL 8.4 client:** `brew --prefix mysql-client@8.4` responds. If not, go to `references/db-access.md` §1.
3. **`~/.my.cnf`** exists with `[client]` (replica) and `[clientstg]` (master). If not, §2.
4. **Password in the OS keychain:** `security find-generic-password -s db_example_password -w` responds (do not print the value). If not, §3.
5. **Wrapper on PATH:** `command -v db-connect`. If not, §4.
6. **Local config** `~/.db-example-skill-config.json` (non-secret). If it does not exist, create it with defaults:
   ```json
   { "db_user": "<your DB user>", "default_row_limit": 200, "statement_timeout_ms": 15000, "output_dir": "/Users/<user>/db-example-results" }
   ```
   `output_dir` should point to a **git-ignored** directory, so query results (business data) are never committed.

If any part of the substrate is missing, guide the setup via `references/db-access.md` instead of improvising.

---

## Flow of a query

### Step 1 — Resolve app + environment

From the user's request, identify:
- **app** (e.g. `orders-service`) → look it up in `references/app-registry.md`.
- **env** (`local` / `staging` / `prod`). If not stated, **ask** — do not assume prod or staging.
- **schema** = the one from the registry for that app+env. **vault** only if the user explicitly asks.

If the app is not in the registry, offer to add it (section "How to add a new app") before continuing.

### Step 2 — Build the SQL

- Build the `SELECT` with the requested columns and filters. Use the real names from the registry (tables/columns of `orders`, `payments`, etc.).
- **Filter by day:** half-open range `>= 'YYYY-MM-DD' AND < 'YYYY-MM-DD+1'` on the correct date column (`start_payment_date` for orders, `payment_date` for payments). Do not use `DATE(col) = ...` (breaks the index).
- **Segmentation:** if the app segments (e.g. by `region`), include the filter unless the user wants all.
- Filter by an indexed column; avoid `SELECT *` and full scans, especially in prod.

### Step 3 — Guardrails (GATE — always apply before executing)

1. **Classify the statement** by its first verb:
   - Read: `SELECT`, `SHOW`, `EXPLAIN`, `DESCRIBE`, `WITH`.
   - Write: `INSERT`, `UPDATE`, `DELETE`, `REPLACE`, `ALTER`, `DROP`, `CREATE`, `TRUNCATE`, `GRANT`, etc.
2. **Apply the per-env policy:**

   | env | read | write |
   |-----|------|-------|
   | prod | ✅ allowed | ❌ **hard reject** — hand the SQL back as text for the DBA. (The wrapper blocks it too.) |
   | staging | ✅ allowed | ⚠️ **explicit per-statement confirmation** before executing |
   | local | ✅ allowed | ✅ allowed (Docker, no risk) |

   For writes on staging: show the exact statement and ask "do you confirm I run this on staging?" — do not execute without an explicit yes.
3. **LIMIT:** if it is a `SELECT` without `LIMIT`, add `LIMIT <default_row_limit>` (from config) and mention it.
4. **Timeout:** add the hint `/*+ MAX_EXECUTION_TIME(<statement_timeout_ms>) */` right after `SELECT` (the wrapper also passes `--connect-timeout`).
5. **Vault / PII:** the `*V*` schemas (e.g. `ORDERSVSTG/PRD`) are **excluded by default**. Only touch them with the user's explicit opt-in; when you do, **mask** sensitive columns (account/card/token) with `LEFT(col,6)` or similar. **Never** print PII in the clear or dump it to the output file.
6. **Large scans:** if the query looks like a heavy scan/aggregation, suggest `EXPLAIN` first and consider your data warehouse instead.
7. **Never** echo the password or the wrapper line that contains it.

### Step 4 — Execute

- **staging / prod:** via the wrapper. For table + structured output, run two formats (or parse one):
  ```sh
  # readable table
  db-connect --env <staging|prod> <SCHEMA> --table -e "<SQL>"
  # structured for processing (TSV/JSON): --batch gives TSV with header
  db-connect --env <staging|prod> <SCHEMA> --batch -e "<SQL>"
  ```
- **local:** `docker exec <container> mysql -u root -p<pw> <SCHEMA> -e "<SQL>"` (container/pw from the registry). Does not use the wrapper.

### Step 5 — Output

1. **Markdown table** in the chat (convert the `--table` / TSV output to markdown). If there are many rows, show the first N and state the total.
2. **Structured file** under `output_dir/<app>/<env>/db-result-<table>-<timestamp>.<ext>` (e.g. `db-example-results/orders-service/staging/db-result-payments-20260716T1627.tsv`). It lives in a git-ignored directory. Create the `<app>/<env>` subfolders if they do not exist. The agent can freely read/process that file afterwards.

   **Pick the format automatically** (TSV is denser in tokens; JSONL is robust to complex content):
   - **TSV by default** (`--batch`, extension `.tsv`) — for flat columns (numbers, dates, enums, short strings). The normal case.
   - **JSONL** (extension `.jsonl`) — use when the `SELECT` projects a `json` or text column that may contain tabs / newlines (`properties`, `metadata`, `*payload*`, `request_payload`), because they would break the TSV. Emit one JSON object per row by wrapping the columns in `JSON_OBJECT(...)` and dumping with `--batch --skip-column-names`:
     ```sh
     db-connect --env staging ORDERSSTG --batch --skip-column-names \
       -e "SELECT JSON_OBJECT('id',id,'name',name,'properties',properties) FROM validations WHERE ... LIMIT 200;" \
       > "$OUT/db-result-validations-<ts>.jsonl"
     ```
   If in doubt between the two for a mixed query, use JSONL (it does not lose data).
3. Report: rows returned, whether `LIMIT` was applied, the chosen format (and why, if JSONL), and the file path.

---

## Notes

- **Authoritative control of prod = grants + replica, not the regex.** The real "SELECT-only" comes from the SELECT-only grants of the DB user on prod and the read replica (see `db-access.md` §Security model). The agent gate (Step 3) and the wrapper gate are defense in depth; they are not the guarantee.
- **Quoting literals:** when building SQL from natural language, escape/quote the values (dates, regions, IDs) so a user string cannot break the query. Prefer values quoted with single quotes and validate the format (e.g. date `YYYY-MM-DD`).
- **Double gate in prod:** the agent rejects writes in Step 3 and the `db-connect` wrapper blocks them again (verb + stdin + multi-statement). Defense in depth.
- **Output format (TSV vs JSONL):** TSV by default because it is token-dense and easy for an LLM to consume on flat tabular data; JSONL only when there are `json`/multi-line text columns that would break the TSV (see Step 5).
- **Extensibility:** adding a new app is just editing `references/app-registry.md`; the wrapper does not change.
- **Security:** credentials only in the OS keychain; config and result dumps git-ignored; VPN + DB access are external prerequisites the skill does not grant (the user requests them from the DBA / infra team).
- This skill is for ad-hoc / diagnostic queries. Schema changes go through your migration tool; large aggregations through your data warehouse.
