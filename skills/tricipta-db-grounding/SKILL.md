---
name: tricipta-db-grounding
description: Query real tricipta production/VFM/VPS data to ground sizing or data-availability decisions when the local dev DB is empty.
---

# Grounding tricipta decisions in real DB data

Use when a tricipta task needs real production data to size a limit, verify a column/table exists, or reconcile a mockup/spec against available data.

## Key fact: local dev DB is empty

The local docker DB (`edafy-db`, Postgres/Timescale on :5432) is **unseeded** — `ast_production_data` and friends return 0 rows. Do not conclude "no data" from the local DB.

The mounted postgres MCP (`xd://mcp__postgres_query`) has **no password configured** and fails with `SASL: ... client password must be a string`. Don't rely on it.

## Reach real data via the staging Azure DB

Credentials are in the commented `# ── Database staging ──` block of `tricipta-edafy-asset-api/.env` (read it with gitignore=false). Query read-only through the running container (it has psql + network egress):

```bash
docker exec edafy-db env PGPASSWORD='<staging-pass-from-.env>' PGSSLMODE=require \
  psql -h psql-production-ai-nonprod.postgres.database.azure.com -p 5432 \
  -U pgadmin -d dev_asset_db -tAc "<read-only SQL>"
```

Keep queries aggregate/read-only (counts, min/max, percentiles, `information_schema.columns`). Never hardcode the password into committed files, skills, or memories — reference its location.

## Data-source map (petroleum domain)

- Production rates/cumulatives/ratios → `ast_production_data` (monthly; 60 cols; **no pressure columns**).
- P50 rate forecasts → `ast_vfm_calculated` (`oil_rate_p50`, `water_rate_p50`, `gas_rate_p50`, `water_cut_p50`, `gas_oil_ratio_p50`).
- P50 pressure → `ast_vps_calculated` (`whp_p50`, `chp_p50`, `flp_p50`, ...).

So "vs P50 forecast" or THP/pressure insights need VFM/VPS, not production data — check the source before promising a mockup element is buildable.

## Typical grounding queries

```sql
-- totals + span
SELECT count(*), count(DISTINCT well_code), min(time)::date, max(time)::date FROM ast_production_data;
-- per-well / per-series distribution for sizing caps
SELECT max(cnt), round(avg(cnt)), round(percentile_cont(0.95) WITHIN GROUP (ORDER BY cnt))
FROM (SELECT well_code, count(*) cnt FROM ast_production_data GROUP BY well_code) t;
-- does a column exist?
SELECT column_name FROM information_schema.columns WHERE table_name='ast_vps_calculated' AND column_name ILIKE '%thp%';
```
