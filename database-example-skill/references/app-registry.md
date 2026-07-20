# App Registry — map of apps → environments → connection

This registry tells the skill, per app and environment, **which host and which schema** to use, whether there is a **vault (PII)**, and whether the app is **segmented**. It is the only file you need to touch to add a new app.

> **Generic example — replace placeholders.** App names (`orders-service`, `payments-connector`), schema codes (`ORDERS*`, `PAYCONN*`), hosts and ports are all placeholders. Swap them for your real ones.

The access pattern **is not uniform** across apps (here we model two families: Makefile-driven and runtime `settingsMap`), and the vault / segmentation are optional. That is why each app is described with its own structure.

---

## Conventions

- **Hosts** (same for all apps, resolved by the `db-connect` wrapper):
  - `prod` → `db-replica.internal.example.com:6033` (read replica).
  - `staging` → `db-master.internal.example.com:6033` (master).
  - `local` → the app's Docker (`docker exec <container> mysql -u root -p<pw> ...`), does **not** use the wrapper.
- **Schemas**: convention `<CODE>LOC` / `<CODE>STG` / `<CODE>PRD`. The vault (PII) is `<CODE>V*` (optional).
- **Segmentation**: most apps do not segment. `orders-service` segments by the **`region` column** at runtime (the `ORDERSSTGUS/EU/…` folders are migration folders, not distinct schemas). Some other app might segment by *project* instead of region.

---

## Apps (v1)

### orders-service

Orders, payments and settlements. Reference app. Has a vault (PII) and is region-segmented by column.

| env | host | schema | vault schema (opt-in, PII) |
|-----|------|--------|----------------------------|
| local | docker `app.orders.db` (`:3306`, root/`secret`) | `ORDERSLOC` | `ORDERSVLOC` (docker `app.orders.vault`, `:3307`) |
| staging | db-master:6033 | `ORDERSSTG` | `ORDERSVSTG` |
| prod | db-replica:6033 | `ORDERSPRD` | `ORDERSVPRD` |

- **Segmentation:** `region` column ∈ {`US`, `EU`, `LATAM`, `APAC`}. Base example region = `US` (currency `'USD'`). Filter with `WHERE region = 'US'`.
- **Key tables (data model):**
  - `orders` — PK `internal_id`, public `id`; cols `status`, `region`, `currency`, `amount`, `flow_id`, `origin_app`, `is_test`, `brand`, `customer_id`; dates `origin_payment_date`, `start_payment_date`, `effective_payment_date`. **Filter by day:** `start_payment_date` (or `origin_payment_date`).
  - `payments` (settlements) — PK `id`; cols `idempotency_id`, `status`, `type` (settlement/cancellation/funding), `region` (nullable), `currency`, `amount`, `bank`, `flow_id`, `is_test`, `order_id`, `payment_kind`, `network`. **Filter by day:** `payment_date`.
  - `validations` — PK `id`; cols `origin_app`, `region`, `is_valid` (tinyint), `name`, `properties` (json), `currency`, `is_test`, `flow_id`, `payment_id` (FK → `payments.id`), `manually_approved`. **Filter by day:** `date` (type `date`).
  - `orders_payments_ref` — junction: `order_id` → `orders.internal_id`, `payment_id` → `payments.id`.
- **Vault (`ORDERSV*`):** PII = bank account numbers. Excluded by default; explicit opt-in + masking.

### payments-connector

Payment connector. No vault, no segmentation.

| env | host | schema | vault |
|-----|------|--------|-------|
| local | docker `app.payments-connector.db` (`:3306`, root/`secret`) | `PAYCONNLOC` | — |
| staging | db-master:6033 | `PAYCONNSTG` | — |
| prod | db-replica:6033 | `PAYCONNPRD` | — |

---

## How to add a new app

1. Confirm the app's `<CODE>` and its schemas per env (check the app's `Makefile` and `internal/**/settings.go`: secret names typically follow a `DB_MYSQL_<DBHOST>_<SCHEMA>_ENDPOINT` pattern).
2. Add a section with the `env → host → schema (+ optional vault)` table.
3. If it has a vault, note the `<CODE>V*` schema and the PII columns to mask.
4. If it segments, say by which dimension (`region` column, `project`, etc.) and how to filter.
5. For `local`, note the Docker container name and the root port/credential (they vary: e.g. some apps use `:3316/:3317`, others use root/`root` with an empty password).

You do not need to touch the `db-connect` wrapper — only this file.

### Candidates to add (out of v1)

- `reporting-service` → `REPORTING*` (no vault).
- `ledger-connector` → `LEDGER*` + vault `LEDGERV*` (`:3316/:3317` local).
- `multi-tenant-connector` → segmented by *project*, multiple schemas + vault. The most complex.
- `account-connector` → `ACCTCONN*` (no vault, no local DB).
