# Today I Learned

Short, single-discovery notes. Things worth remembering but too small for a full post.

***

## 2026-04-16 — `rm-rf` is a bad idea

`rm -rf` is a bad idea.
It's a bad idea because it's a bad idea. It's a bad idea because it's a bad idea.
It's a bad idea because it's a bad idea. It's a bad idea because it's a bad idea.
It's a bad idea because it's a bad idea. It's a bad idea because it's a bad idea.

***

## 2026-04-16 — ClickHouse empty password disables network access

Setting `CLICKHOUSE_PASSWORD=""` (empty string) in Docker/k8s env vars causes ClickHouse to
silently disable network access for the `default` user. It won't throw an error at startup —
it just refuses all connections. Always set a non-empty password.

***

## 2026-04-16 — k3d nginx routes to node port, not NodePort

k3d's auto-generated nginx config routes to the container port directly (e.g. `:8123`),
not the Kubernetes NodePort (e.g. `:30123`). If your Service is NodePort type, you need
to manually patch the nginx upstream inside the LB container after cluster creation.

***

## 2026-04-01 — MySQL: wrapping column in function breaks index usage

`WHERE CONVERT_TZ(created_at, '+00:00', '+07:00') >= '2026-04-01'` cannot use an index on
`created_at`. Convert the filter value to UTC instead and compare directly against the column.

***

## 2026-04-01 — MySQL CASCADE + SET NULL on same row: second constraint is moot

If two FK constraints both target the same row — one with `ON DELETE CASCADE` and one with
`ON DELETE SET NULL` — the CASCADE fires first and deletes the row. The SET NULL constraint
then has nothing to act on. MySQL doesn't error; it just skips it silently.

***

## 2026-03-30 — Airflow DAG version churn: sort your config dict

If `get_dynamic_dag_variable()` returns a dict with non-deterministic key ordering,
Airflow generates a new DAG version on every parse cycle. Fix: wrap the result with
`{k: v for k, v in sorted(dags.items())}`. Fixed properly in Airflow 3.x.
