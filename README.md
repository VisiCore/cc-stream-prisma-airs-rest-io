# Prisma AIRS Collector

Pulls Palo Alto Networks **Prisma AIRS** (AI Runtime Security) scan activity into Cribl Stream on a schedule, with no inbound firewall changes. The pack ships a REST Collector, an event breaker, a null-removal pipeline, and a route. One event is produced per scan action (a prompt, model response, or code scan inside an AI session).

## About this Pack

- Polls the Strata Cloud Manager (SCM) Management API `dashboard/v2/sessions` endpoints, the same backend the SCM AI Security dashboard uses.
- Discovers every session active in the last hour, then collects each session's scan actions with a one-day window and offset paging.
- Strips null and empty values from `_raw` with a Parser function in reserialize mode (field filter `value != null && value !== '' && value !== undefined`), so events carry only populated fields. No other transformation is applied; flatten or enrich downstream as needed.
- Each event keeps the source record as-is: `scan_id`, `session_id`, `transaction_id`, `application_id`, `application_name`, `profile_name`, `status`, `kind[]`, `start_time`, `scan_sub_req_id`, `violation_breakdown{}` (critical/high/medium/low/total) and `detection_type_violation_breakdown[]` (one entry per detector). `_time` comes from `start_time`.
- Collector state tracking plus the run filter `_time > __collectible.state.latestTime` prevents duplicates across overlapping runs.

### Why not the documented Scan Logs endpoint

The documented `POST /v1/mgmt/scanlogs` endpoint returned HTTP 502 `Unexpected EOF at target` for every combination of window, page size, and filter during development (September 2026). The dashboard session endpoints are the working alternative. If Palo Alto fixes `scanlogs`, it offers richer per-detector verdicts and could replace the collect step.

The per-application Scan API key (`x-pan-token`) cannot list scan history. It only submits scans and looks up results by an id you already hold, so it is not used here.

## Deployment

### 1. Create an SCM service account (Palo Alto side)

In Strata Cloud Manager go to **Identity & Access**, create a service account, and assign it a role on the tenant service group (TSG) where Prisma AIRS is activated. A read-only custom role with the Prisma AIRS AI Runtime (`airs_api.*`) permissions is sufficient; Superuser on that TSG also works. Note the **client id**, **client secret**, and the numeric **TSG id**.

Verify before configuring the pack:

```bash
ACCESS=$(curl -sS -u "$CLIENT_ID:$CLIENT_SECRET" -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&scope=tsg_id:$TSG_ID" \
  -X POST https://auth.apps.paloaltonetworks.com/oauth2/access_token | python3 -c 'import json,sys;print(json.load(sys.stdin)["access_token"])')
curl -sS -H "Authorization: Bearer $ACCESS" \
  "https://api.sase.paloaltonetworks.com/aisec/v1/mgmt/dashboard/v2/sessions/sessionsoverview?time_interval=1&time_unit=hour&limit=25&offset=0"
```

A 403 with the response header `x-opa-decision: false` means the role lacks Prisma AIRS permissions on that TSG.

### 2. Create the group secret (Cribl side)

Packs cannot contain secrets. In the Worker Group, create a **text secret** with id `vct-prisma-airs-rest-secret` holding the SCM client secret. The id must match the `vct_prisma_airs_secret_id` pack variable.

### 3. Set the pack variables

| Variable | Value |
|---|---|
| `vct_prisma_airs_client_id` | SCM service account client id (quoted string) |
| `vct_prisma_airs_tsg_id` | Tenant service group id (quoted string) |
| `vct_prisma_airs_mgmt_host` | `https://api.sase.paloaltonetworks.com/aisec` (default; `api.apps.paloaltonetworks.com/aisec` also works) |
| `vct_prisma_airs_secret_id` | `vct-prisma-airs-rest-secret` (informational; the collector references the secret directly) |

Variable values are JavaScript expressions, so keep the single quotes around strings.

### 4. Point the route at a destination

The pack route `vct_prisma_airs_session_actions` matches `datatype=='prisma_airs_session_action'` and sends to `devnull` by default. Change the output to your SIEM or Lake destination.

### 5. Run once, then enable the schedule

Open the collector `vct_prisma_airs_session_actions` inside the pack, click **Run** in preview mode, and confirm events appear with `_time` matching `start_time`. Then enable the schedule (default every 15 minutes), commit, and deploy.

### Collector settings worth knowing

- Discovery limit is 500 sessions per run. The sessions overview accepts hour, day, and days units.
- The session detail endpoint **requires** `offset` and only accepts an hour window of exactly 1, so the collect step uses `time_interval=1&time_unit=day`. Pagination is configured with a starting offset of 0 so Cribl always sends it.
- The OAuth token lives 15 minutes; the collector fetches a fresh one every run.
- The transaction endpoint (`sessions/sessiontransaction`) adds latency, token counts, model name, and environment per scan, at the cost of one extra call per action. It is not collected in this version.

## Upgrades

Upgrading in place keeps your pack variables and route output. The `vct_prisma_airs_session_actions` pipeline contains only the Parser null-removal step; if you add your own functions, copy the pipeline rather than editing it so upgrades do not overwrite your changes.

## Authors

- Andrew Hendrix <Andrewh@visicoretech.com>
- VisiCore Tech <CriblPacks@VisiCoreTech.com>

## Release Notes

### Version 0.1.0 - 2026-09-09

- Initial release: session-action REST collector, `_vct_prisma_airs_eb` event breaker, null/empty-removal pipeline, placeholder route, pack logo.
