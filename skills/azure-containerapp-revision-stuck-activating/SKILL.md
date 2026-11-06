---
name: azure-containerapp-revision-stuck-activating
description: "Diagnose Azure Container Apps revision problems (e.g. Tricipta asset-api after CD): revision stuck Activating/Unhealthy (CrashLoopBackOff vs boot hang, PG extension allow-list, boot DDL lock) OR old revisions/replicas not shutting down and latest deploy getting no traffic."
---

# ACA revision diagnostics

First distinguish: (a) new revision stuck Activating, vs (b) many revisions still active / old replicas not shutting down / latest deploy not serving. Always run both commands in step 1.

## 1. Inspect app + revisions
```
az containerapp show -n APP -g RG --query "{mode:properties.configuration.activeRevisionsMode,latest:properties.latestRevisionName,ready:properties.latestReadyRevisionName,scale:properties.template.scale,traffic:properties.configuration.ingress.traffic}" -o json
az containerapp revision list -n APP -g RG --query "[].{name:name,active:properties.active,running:properties.runningState,health:properties.healthState,traffic:properties.trafficWeight,replicas:properties.replicas,created:properties.createdTime,image:properties.template.containers[0].image}" -o table
```
- `latest` != `ready` -> latest revision stuck: go to 2.
- Many active revisions each with replicas -> go to 5.

## 2. Replica state decides branch
```
az containerapp replica list -n APP -g RG --revision REV --query "[].{name:name,c:properties.containers[0].{ready:ready,restarts:restartCount,state:runningState,detail:runningStateDetails}}" -o json
```
- `CrashLoopBackOff`, restarts > 0: process exits. Go to 3.
- `Running`, 0 restarts, `ready:false` for minutes: boot hung. Go to 4.

## 3. Crash: find fatal line
`az containerapp logs show ... --revision REV --container NAME --tail 300 --format text` (300 max). Tail often is echoed migration SQL; grep `Failed|ERROR|panic`.
- `extension "X" is not allow-listed ... (SQLSTATE 0A000)`: `az postgres flexible-server parameter show -g RG -s SERVER -n azure.extensions`; fix with `parameter set --value A,B` keeping existing values. Server-wide: affects every env DB on that server.
- Map image tag SHA to commit only after `git fetch`.

## 4. Hang: usually DDL waiting on a lock
Get secrets (`az containerapp secret show --secret-name db-host|db-user|db-password|db-name`), psql with PGSSLMODE=require, SQL in a file:
```sql
select pid, usename, client_addr, state, now()-xact_start xact_age, wait_event_type, wait_event,
       pg_blocking_pids(pid) blocked_by, left(regexp_replace(query,'\s+',' ','g'),100) q
from pg_stat_activity where datname=current_database() and pid<>pg_backend_pid()
  and (state<>'idle' or xact_start is not null) order by xact_start nulls last;
```
- Head blocker has empty `blocked_by`. Map client_addr via `az network vnet list` (Tricipta 10.0.5.0/26 = Databricks host subnet).
- `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` still takes ACCESS EXCLUSIVE; a waiting ALTER blocks new readers too. Every restart queues another ALTER.
- Options: wait, or `pg_cancel_backend(pid)` with owner consent. Durable fix: check information_schema first, or `SET lock_timeout` on boot migrations.

## 5. Old revisions/replicas not shutting down
Causes (check all):
- `activeRevisionsMode: Multiple`: old revisions stay active until explicitly deactivated.
- `minReplicas >= 1`: 0%-traffic revisions still hold replicas (and cost).
- CD only does `az containerapp update --image`: creates revision, never deactivates or shifts traffic.
- Ingress traffic pinned to a named revision/label instead of `latestRevision: true`: new deploys get 0% and the env silently serves an older image despite green CD. Check `az monitor activity-log list -g RG --offset 3d --query "[?contains(resourceId,'APP') && operationName.value=='Microsoft.App/containerApps/write']"` for who changed it.
Fix (confirm manual test revisions are done with their owner first):
```
az containerapp ingress traffic set -n APP -g RG --revision-weight latest=100
az containerapp revision deactivate -n APP -g RG --revision OLD_REV
az containerapp revision set-mode -n APP -g RG --mode single   # durable
```
Or keep Multiple and add CD step: traffic latest=100 + deactivate non-latest.

## Tricipta specifics
- asset-api: `ca-prod-ai-np-{dev,staging,uat}-asset-api`, rg `rg-production-ai-nonprod-{env}`; all three in Multiple mode, minReplicas 1, max 3. Shared PG `psql-production-ai-nonprod` in `rg-production-ai-nonprod-shared`. Probes TCP :80, startup 240s.
- Deploy: `.github/workflows/cd.yml` `az containerapp update` only.
- Boot migrations: `internal/database/connection.go`, `internal/database/migrations/`; re-run every start.
