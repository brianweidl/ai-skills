# MySQL prod/staging access setup (macOS)

Guide to get the connection to the MySQL databases of **staging** (master) and **prod** (read replica) working from your laptop, with the password stored in the **OS keychain** (never in plain text) and the `db-connect` wrapper that injects it automatically.

> **Generic example — replace placeholders.** All hostnames (`db-master.internal.example.com`, `db-replica.internal.example.com`), the port `6033`, schema names and the keychain service `db_example_password` are placeholders. Point them at your own infrastructure.

This setup **carries no one's credentials**. Each person uses their own DB user and their own password (the one your DBA / infra team enables for the SQL proxy).

---

## How it works (3 pieces)

1. **`mysql-client@8.4` (Homebrew)** — the MySQL binary. Some prod servers use the `mysql_native_password` auth plugin, removed in mysql 9.x. That is why the 8.4 line is pinned; with the 9.x client the connection fails with a cryptic plugin error. (Adjust to your server's auth plugin.)
2. **`~/.my.cnf`** — stores host/port/user per environment, **without a password** (perms 600).
   - Group `[client]` → **prod**, read replica (`db-replica.internal.example.com:6033`).
   - Group `[clientstg]` → **staging**, master (`db-master.internal.example.com:6033`).
3. **macOS keychain + `db-connect` wrapper** — the password lives encrypted in the keychain (item `service=db_example_password`). The wrapper (`scripts/db-connect`) reads it, passes it via `MYSQL_PWD` (so it stays out of the history and `ps`) and runs the 8.4 client. You never type the password by hand or store it on disk in the clear.

`mysql` on its own is left neutral on purpose (for local Docker / non-prod connections), so there is no risk of leaking the password to a local connection.

**Why the keychain and not `.mylogin.cnf` (login-path):** the login-path store is only *obfuscated* (decrypted with `my_print_defaults --show`); the keychain is real OS-level encryption, with per-application ACLs. On macOS `MYSQL_PWD` is not readable by other user processes, so the keychain + `MYSQL_PWD` pair is the most secure at-rest option.

---

## Prerequisites

- **Corporate VPN** connected (if your DBs are behind one).
- **macOS with Homebrew** (adapt for other OSes).
- **Proxy access granted to your DB user** (`db-replica.internal.example.com` / `db-master.internal.example.com`, port `6033`) and to the DBs you need. This is an infra grant: request it through your usual DB-access channel (DBA / infra team), like any access to a production database. The local setup does **not** grant access, it only configures the client.

---

## Step by step

In the examples, replace `YOUR_USER` with your DB user (the one enabled on the proxy).

### 1) Install the 8.4 client

```sh
brew install mysql-client@8.4
ls -l "$(brew --prefix mysql-client@8.4)/bin/mysql"   # verify
```

### 2) Create `~/.my.cnf` (without a password)

Copy the template and edit the user:

```sh
cp .claude/skills/database-example-skill/templates/my.cnf.example ~/.my.cnf
# edit user=YOUR_USER in both groups
chmod 600 ~/.my.cnf
```

### 3) Store the password in the keychain

Paste your password when prompted (it is not shown and does not stay in the history):

```sh
security add-generic-password -a "$(id -un)" -s db_example_password -w
```

Use your local mac user as the account. If your DB user differs from the local one, replace `"$(id -un)"` with your DB user and export `DB_EXAMPLE_ACCOUNT=your_db_user`.

To rotate the password later, add `-U`:

```sh
security add-generic-password -a "$(id -un)" -s db_example_password -w -U
```

### 4) Put the wrapper on PATH

The wrapper lives in `scripts/db-connect`. Link (or copy) it into a PATH dir and make it executable:

```sh
mkdir -p ~/.local/bin
ln -sf "$PWD/.claude/skills/database-example-skill/scripts/db-connect" ~/.local/bin/db-connect
chmod +x .claude/skills/database-example-skill/scripts/db-connect
# make sure ~/.local/bin is on PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

### 5) (First time) authorize keychain access

The first time the wrapper reads the keychain, macOS may ask for authorization. Choose **"Always Allow"** so it stays non-interactive in background sessions.

---

## Verification

```sh
db-connect --env staging ORDERSSTG -e 'SELECT 1'      # staging (master)
db-connect --env prod    ORDERSPRD -e 'SELECT NOW()'  # prod (read replica)
```

If they return a result, it is working.

---

## Security model — what guarantees what

The substrate has three controls, in order of authority:

1. **DB user grants (authoritative).** The real guarantee of "prod = read-only" is that **your proxy user has SELECT-only grants on prod** (and, ideally, no access to the vault schemas unless explicitly needed). Ask the DBA / infra team for it this way. No client-side control replaces this.
2. **Read replica (authoritative).** `--env prod` hits `db-replica` (a replica), which rejects writes at the server level.
3. **Wrapper gate (defense in depth, best-effort).** In `--env prod` the wrapper blocks anything that is not `SELECT/SHOW/EXPLAIN/DESCRIBE/WITH`, SQL via stdin, and multi-statements. It is a safety net in case something above fails — **do not rely on it alone.**

**`MYSQL_PWD`:** the password is injected via `MYSQL_PWD` only into the `mysql` process. On macOS other users cannot read it (no `/proc`), but your own user could see it with `ps eww` while the query runs. A known, acceptable trade-off on a single-user laptop; it never stays on disk or in the history.

## Operational rules (read them before running queries)

- **Prod = read-only.** The wrapper blocks writes in `--env prod`, but the control that matters is the grants (see above). Any change is handed back as text to be run by the DBA.
- **Staging = read + write**, but every write requires explicit confirmation (the agent asks for it, see `SKILL.md`).
- **Use the read replica** (`db-replica`, default of `[client]`) for prod diagnostics.
- **Always filter by an indexed column / partition key.** No full scans or `SELECT *` on large tables in prod. For large scans/aggregations use your data warehouse. When unsure whether it uses an index, validate with `EXPLAIN` first.
- **Never expose PII/secrets** in output or logs: the vault DBs (`*V*`, e.g. `ORDERSVSTG/PRD`) contain PII (account/card numbers). They are excluded by default; if you access them, mask (`LEFT(col,6)`). Never print passwords.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Authentication plugin 'mysql_native_password' cannot be loaded` | You are on the 9.x client. Use `db-connect` (forces 8.4), not `mysql`. |
| `no password in keychain` | Run step 3. Verify the account matches (`id -un` or `DB_EXAMPLE_ACCOUNT`). |
| `command not found: db-connect` | `~/.local/bin` is not on PATH. See step 4. |
| Connects but returns `Access denied` | Your DB user is not enabled on the proxy / that DB. Request access (prerequisites). |
| `cannot find mysql-client@8.4` | `brew install mysql-client@8.4`. |
| `BLOCKED. Only reads are allowed on prod` | You tried a DML/DDL on prod. Correct: hand the SQL to the DBA. |

---

## Recovery (new mac / lost keychain)

The `.my.cnf` has no password: the only local copy is the keychain. If you lose it, store it again (step 3) taking it from the team's canonical source (secrets vault / DBA / infra team). There is no plain-text backup on disk, on purpose.
