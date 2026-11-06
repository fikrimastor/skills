---
name: aca-revision-stuck-activating
description: "Diagnose an Azure Container Apps revision stuck in 'Activating' (latestRevision != latestReadyRevision), e.g. a Tricipta asset-api deploy that never goes live — replica CrashLoopBackOff, startup-migration failures, Azure PostgreSQL extension allow-list errors."
---

# ACA revision stuck in Activating

## Steps
1. Find the stuck app: in `az containerapp list --query "[].{name:name,rg:resourceGroup,rev:properties.latestRevisionName,ready:properties.latestReadyRevisionName}" -o table`, look for a row where rev != ready.
2. List its revisions with `az containerapp revision list -n APP -g RG --query "[].{name:name,running:properties.runningState,health:properties.healthState,traffic:properties.trafficWeight,image:properties.template.containers[0].image}"`. In Single mode the old revision keeps serving, so the outage is "new code not live", not downtime.
3. Check the replica state with `az containerapp replica list -n APP -g RG --revision REV`. `CrashLoopBackOff` means the app exits; `started:false` with no restarts points to probes or the port.
4. Read the logs with `az containerapp logs show -n APP -g RG --revision REV --container NAME --tail 300 --format text`. Don't use a small tail: echoed migration SQL bodies push the real error out of view. Search for `Failed to initialize` / `ERROR:`.
5. Map the image SHA to a commit, and check whether the last good revision's SHA contains it (`git merge-base --is-ancestor`). Run `git fetch` first, because the SHA may not exist locally.

## Known cause: Azure PG extension allow-list
The error is `extension "X" is not allow-listed ... (SQLSTATE 0A000)`.
- Check it: `az postgres flexible-server parameter show -g rg-production-ai-nonprod-shared -s psql-production-ai-nonprod -n azure.extensions`.
- Fix it: `parameter set ... --value <existing>,X`. Keep any existing values. This is a shared server, so the change hits dev, staging and uat. Get the user's approval before running it.
- Then run `az containerapp revision restart -n APP -g RG --revision REV`.
- Warn the user that staging and uat will crash the same way when the change is promoted if the extension isn't allow-listed first.
