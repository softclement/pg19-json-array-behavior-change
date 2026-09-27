# PoC: `json_array()` empty-result behavior change (PostgreSQL 18 → 19)

## Objective

Validate the PostgreSQL 19 behavior change where `json_array()` called with a
query that returns **no rows** now returns an **empty JSON array (`[]`)**
instead of `NULL`.

- Commit: [8d829f5a0](https://postgr.es/c/8d829f5a0) (Richard Guo)
- Old behavior (PG ≤ 18): `json_array(SELECT ... FROM ... WHERE <no rows>)` → `NULL`
- New behavior (PG ≥ 19): same call → `[]`

Environment: WSL2 + Podman, two containers running PostgreSQL 18 and
PostgreSQL 19 beta side by side on different host ports.

## Why this matters in real-world scenarios

Application code and ETL/reporting pipelines that build JSON with
`json_array()` often treat the result as "the list of items" and feed it
straight into JSON parsing, API responses, or downstream aggregation. When
the source rows are empty:

- **Before (NULL):** code has to explicitly check for `NULL` before parsing
  or iterating, or it fails with a null-pointer / "cannot iterate null"
  error. Any `NOT NULL` column or `json_array() IS NOT NULL` check would
  incorrectly filter these records out.
- **After (`[]`):** downstream consumers (JSON parsers, APIs, JS
  `array.map()`, etc.) can treat "no rows" and "empty array" the same way,
  with no special-casing.

So on upgrade to PG19, any code that relied on `json_array()` returning
`NULL` for empty results (e.g. `WHERE json_array(...) IS NULL` or
`COALESCE(json_array(...), '[]')` workarounds) needs review — the workaround
becomes redundant, and a `NULL`-check branch will silently stop triggering.

---

## 1. Prerequisites

- WSL2 with Podman installed and working (`podman version`)
- Ports `5018` and `5019` free on the WSL host

---

## 2. Create the PoC directory

```bash
mkdir -p ~/pg-json-array-poc
cd ~/pg-json-array-poc
```

---

## 3. Start the two containers

```bash
# PostgreSQL 19 beta
podman run -d \
  --name pg19 \
  -e POSTGRES_PASSWORD=postgres \
  -p 5019:5432 \
  postgres:19beta4

# PostgreSQL 18
podman run -d \
  --name pg18 \
  -e POSTGRES_PASSWORD=postgres \
  -p 5018:5432 \
  postgres:18
```

---

## 4. Create the test SQL script

This script builds an empty table, then calls `json_array()` against a
`SELECT` that matches zero rows — the exact case the commit changes.

```bash
cat > test_json_array.sql << 'EOF'
DROP TABLE IF EXISTS empty_source;
CREATE TABLE empty_source (id int, val text);

SELECT json_array(SELECT val FROM empty_source WHERE id = 999) AS result;
EOF
```

Copy it into both containers:

```bash
podman cp test_json_array.sql pg18:/tmp/test_json_array.sql
podman cp test_json_array.sql pg19:/tmp/test_json_array.sql
```

---

## 5. Run the test on PostgreSQL 18 (expect `NULL`)

```bash
podman exec -it pg18 psql -U postgres -f /tmp/test_json_array.sql
```

Expected output:

```
 result
--------
 (null)
```

---

## 6. Run the test on PostgreSQL 19 (expect `[]`)

```bash
podman exec -it pg19 psql -U postgres -f /tmp/test_json_array.sql
```

Expected output:

```
 result
--------
 []
```

---

## 7. Clean up the environment

```bash
# Stop and remove containers
podman stop pg18 pg19
podman rm pg18 pg19

# (Optional) remove the images to reclaim disk space
podman rmi postgres:18 postgres:19beta4

# (Optional) remove the PoC directory
cd ~
rm -rf ~/pg-json-array-poc
```

---

## Summary

| PostgreSQL version | `json_array()` on empty subquery |
|---------------------|-----------------------------------|
| 18 and earlier       | `NULL`                            |
| 19 (beta4+)           | `[]` (empty JSON array)           |

This confirms commit
[8d829f5a0](https://postgr.es/c/8d829f5a0) changes `json_array()`'s
zero-row behavior starting in PostgreSQL 19.
