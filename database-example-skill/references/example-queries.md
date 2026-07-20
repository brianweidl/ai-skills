# Example queries — a single day for one region (orders-service)

Goal of the skill: bring back **orders**, **payments** (settlements) and **validations** for one day for a region, filterable by columns.

> **Generic example — replace placeholders.** App name, schemas, table/column names and the region are placeholders. Adapt to your data model.

Parameters to replace:
- `:DAY` → the day, format `YYYY-MM-DD` (e.g. `2026-07-15`).
- `:NEXT_DAY` → the following day, `YYYY-MM-DD` (e.g. `2026-07-16`). Half-open range `>= :DAY AND < :NEXT_DAY` (uses the index; works for `date` and `datetime` columns).
- Region: `region = 'US'` (currency `'USD'`).

Run with the wrapper (staging or prod). Staging example:
```sh
db-connect --env staging ORDERSSTG --table -e "<query>"
```
Prod is identical with `--env prod ORDERSPRD` (read-only).

---

## 1) Orders of the day (US)

```sql
SELECT /*+ MAX_EXECUTION_TIME(15000) */
  id, status, region, currency, amount, flow_id, origin_app, brand, is_test,
  origin_payment_date, start_payment_date, effective_payment_date
FROM orders
WHERE region = 'US'
  AND start_payment_date >= ':DAY' AND start_payment_date < ':NEXT_DAY'
ORDER BY start_payment_date
LIMIT 200;
```
Filter by day on `start_payment_date` (alternative: `origin_payment_date`). Filterable by `status`, `flow_id`, `brand`, `is_test`, etc.

## 2) Payments / settlements of the day (US)

```sql
SELECT /*+ MAX_EXECUTION_TIME(15000) */
  id, idempotency_id, status, type, region, currency, amount, bank,
  flow_id, is_test, order_id, payment_kind, network, payment_date
FROM payments
WHERE region = 'US'
  AND payment_date >= ':DAY' AND payment_date < ':NEXT_DAY'
ORDER BY payment_date
LIMIT 200;
```
Note: `payments.region` was added late and is nullable — very old records could have `region IS NULL`. For a recent day it already comes as `'US'`. If rows are missing, widen to `(region = 'US' OR region IS NULL)` and validate by `flow_id`/`order_id`.

## 3) Validations of the day (US)

```sql
SELECT /*+ MAX_EXECUTION_TIME(15000) */
  id, name, is_valid, manually_approved, region, currency, origin_app,
  flow_id, is_test, payment_id, date
FROM validations
WHERE region = 'US'
  AND date = ':DAY'
ORDER BY name
LIMIT 200;
```
`validations.date` is a `date` type, so `date = ':DAY'` is exact. `payment_id` references `payments.id` (you can join to see which payment it validated). Filterable by `is_valid`, `name`, `flow_id`, `manually_approved`.

---

## Structured output (to process the data)

So the agent can use the data, also run in batch mode (TSV with header) and dump to a git-ignored file under `output_dir/<app>/<env>/`:
```sh
OUT="$HOME/db-example-results/orders-service/staging"; mkdir -p "$OUT"
db-connect --env staging ORDERSSTG --batch -e "<query>" > "$OUT/db-result-orders-$(date +%Y%m%dT%H%M%S).tsv"
```
