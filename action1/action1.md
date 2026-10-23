# Action1 — Patch Management & RMM

API prefix: `action1:`
Domain: everything that happens INSIDE the Action1 cloud console — managed
endpoints, patching, vulnerabilities, software inventory, deployments,
automations and reports.

## Action1 Native REST API

Base URL (this deployment): `https://app.eu.action1.com/api/3.0`
Other regions: `app.action1.com` (NA), `app.na-2.action1.com` (NA-2),
`app.au.action1.com` (AU). Never mix regions inside one run.

HTTP methods:
- **GET** to list/retrieve (with query parameters for filtering)
- **POST** to create
- **PATCH** to partially update
- **DELETE** to remove

Every collection GET returns a **ResultPage** envelope:

```json
{
  "id": "1",
  "type": "ResultPage",
  "name": "",
  "self": "https://app.eu.action1.com/api/3.0/organizations?from=0&limit=50",
  "items": [ ... ],
  "total_items": "324",
  "limit": "50",
  "next_page": "...",
  "prev_page": ""
}
```

Note `items` (not `results`), and note that `total_items` and `limit` are
returned as **strings**.

---

# ⛔ RULE ZERO — A RAW LIST RESPONSE NEVER ENTERS THE CONVERSATION

Read this before the pagination rules, because it is the failure that
actually bites. Everything below assumes it.

A collection call to this API succeeds with `200 OK` and then dies anyway,
because the body is too large for whatever sits between the API and the
model: transcripts truncate, tool-result buffers clip, serializers give up.
The HTTP call was fine. The data is gone. And nothing in the response says
so — which is worse than an error, because a truncated body looks like a
short list, and a short list looks like an answer.

**So: a list operation's body is data to be processed, never text to be
read.** Concretely:

1. Run every list call inside a code step (`python_snippet` or equivalent).
   Drain the pages, keep the rows in a variable, write them to a file.
2. Return to the conversation only: the coverage line, aggregates, and at
   most the first ~20 rows. Never the raw page.
3. If a runtime offers no code step and only single-call tools, then a list
   call is a **counting instrument, not a retrieval instrument** — read
   `total_items`, then fetch the objects you actually need one at a time
   with the single-object GET.
4. Never reason about the shape of the data from a body you watched get
   truncated. Re-fetch it into a file first.

Corollaries, both learned the hard way:

- **A truncated body is not evidence about the backend.** If a page size or
  a filter looks ignored, you may simply be reading a clipped rendering of a
  correct response — or a query string that never reached Action1. Before
  concluding "Action1 ignores this parameter", run the A/B in
  *Verifying a parameter actually works* below.
- **`self` is not proof.** It is a string in the payload. In an agent
  runtime it may be reconstructed from the request by the integration layer,
  so `self` showing `limit=1` tells you nothing about what the server did.
  `len(items)` is the only honest measure.

---

# ⛔ PAGINATION — THE HARD RULE

**The Action1 API returns only 50 items per page by default.** If a call is
made without `from` and `limit`, everything past item 50 is silently
discarded: no error, no warning, just a short list. This is the single most
common way an Action1 integration produces wrong answers ("we only have 50
computers", "there are only 50 missing patches").

### Non-negotiable rules

1. **Never call a list operation once and report the result.** A single
   `execute(...)` on a collection endpoint is ALWAYS one page, never the
   answer.
2. **Always send `from` and `limit` explicitly.** They are declared
   `required: true` in the spec for exactly this reason. Do not rely on
   defaults.
3. **The page size ceiling is 200, not 1000.** Action1's own PowerShell
   module sets `$Action1_PagedGetRequestDefaultLimit = 200` and rejects
   anything larger (`Test-Action1PageSize` throws: *"Limit must be from 1
   through 200"*). Use **100–200**. Asking for 1000 is outside what the
   vendor's own client emits, and combined with `fields=*` it is the single
   most reliable way to produce a body nothing downstream can hold.
   More pages of 200 is cheaper than one page you cannot read.
4. **Loop until drained.** Stop when `items` is empty, when
   `len(collected) >= int(page["total_items"])`, or when the offset stops
   advancing.
5. **Cast before comparing.** `total_items` and `limit` are strings.
   `len(items) < page["total_items"]` compares an int to a str and is a
   silent bug (and a `TypeError` in Python 3). Always `int(...)`.
6. **Step by what the response actually returned.** Not by the limit you
   sent, and not by `page["limit"]` either. Compute
   `next_from = int(page.get("from", current_from)) + len(page["items"])`.
   This is what PSAction1 does, and its own comment says why: *"some
   endpoints may normalize the requested limit, update total_items between
   requests, or return unexpected paging values. This also prevents infinite
   loops when the returned offset does not advance."*
7. **Guard against a stalled offset.** If `next_from <= previous_from`,
   break and report the dataset as incomplete. Do not spin.
8. **`next_page` / `prev_page` are empty strings when absent**, not `null`
   and not missing. `if page["next_page"]:` is safe; `if "next_page" in page:`
   is not.
9. **State the totals in the answer.** Always say "247 of 247 endpoints
   (2 pages)". If the loop was capped, say so on the first line:
   "showing the first 500 of 3,214 — ask me to continue".
10. **`from` is a Python reserved word.** You cannot write
    `execute(..., from=0)`. Always pass it inside the `query_params` dict:
    `query_params={"from": 0, "limit": 200}`.

### Known quirk: `limit=1`

The organizations endpoint returns an **empty** first page for `limit=1`
sent without `from`; PSAction1 works around it by requesting `limit=2` on
the first page instead. Always send `from` explicitly, and treat `limit=1`
as a smoke test rather than a normal read.

### Canonical pagination helper (paste-ready for python_snippet steps)

```python
import time

def fetch_all(operation_id, path_params=None, query_params=None,
              page_size=200, max_items=None, pause=0.25):
    """Drain an Action1 ResultPage collection completely.

    Returns (items, total_items, pages_fetched, complete: bool).
    `complete` is False when the loop stopped early — propagate it to the
    user instead of presenting a partial set as the answer.
    """
    if not 1 <= page_size <= 200:
        raise ValueError("page_size must be 1..200 (Action1's own client "
                         "caps at 200)")

    path_params = path_params or {}
    base_q = dict(query_params or {})
    # Never let a caller's paging args fight with ours.
    base_q.pop("from", None)
    base_q.pop("limit", None)

    items, offset, pages, total, complete = [], 0, 0, None, False

    while True:
        q = dict(base_q)
        q["from"] = offset            # 'from' is a reserved word -> dict only
        q["limit"] = page_size
        page = execute(operation_id=operation_id,
                       path_params=path_params,
                       query_params=q)

        if not isinstance(page, dict) or "items" not in page:
            raise RuntimeError(f"{operation_id}: response is not a ResultPage")

        batch = page.get("items") or []
        pages += 1
        items.extend(batch)

        # Re-read paging metadata from EVERY page: endpoints may normalise the
        # limit or revise total_items between requests.
        if page.get("total_items") is not None:
            try:
                total = int(page["total_items"])
            except (TypeError, ValueError):
                pass

        if not batch:                                  # server returned nothing
            complete = True
            break
        if max_items and len(items) >= max_items:       # explicit user cap
            items = items[:max_items]
            break                                       # complete stays False
        if total is not None and len(items) >= total:   # drained
            complete = True
            break

        # Advance by what the server actually served, never by what we asked.
        try:
            served_from = int(page.get("from"))
        except (TypeError, ValueError):
            served_from = offset
        if served_from < 0:
            served_from = offset
        next_offset = served_from + len(batch)

        if next_offset <= offset:                       # stalled -> bail out
            break                                       # complete stays False
        if pages >= 2000:                               # runaway guard
            break

        offset = next_offset
        time.sleep(pause)                               # stay under 30 req/min

    return items, (total if total is not None else len(items)), pages, complete
```

If an endpoint ever ignores `from` (cursor-only behaviour), fall back to
following the envelope instead: keep requesting while `page["next_page"]`
is a non-empty string, parsing the `from`/`limit` values out of that URL.
The termination condition is unchanged: empty `items`, or
`len(collected) >= int(total_items)`.

Usage — note the projected `fields`, not `*`:

```python
endpoints, total, pages, complete = fetch_all(
    "action1:endpoints_managed_list",
    path_params={"organization_id": org_id},
    query_params={"fields": "name,status,OS,agent_version,last_seen"},
)
assert complete, f"partial: {len(endpoints)} of {total}"
# -> report: f"{len(endpoints)} of {total} endpoints ({pages} pages)"
```

### If the runtime cannot loop (single-call tools)

Then a list call is a **counting instrument**, not a retrieval one. Send
`limit=200`, read `int(total_items)`, and tell the user the real total. To
get objects, fetch them individually with `endpoints_managed_get` — a
single-object read returns a bounded body and is the only retrieval path
that never truncates. Never present a first page as the full set.

### Verifying a parameter actually works

Before reporting that Action1 ignores a `limit` or a filter, prove it with
an A/B that only that parameter varies — and read `len(items)`, never the
rendered body or `self`:

```python
a = execute(op, path_params=pp, query_params={"from": 0, "limit": 200})
b = execute(op, path_params=pp, query_params={"from": 0, "limit": 2})
# limit is honoured   -> len(b["items"]) == 2
# limit is ignored    -> len(b["items"]) == len(a["items"])
```

Same shape for a filter: identical calls with and without `name=`. If both
come back the same length, the query string is probably not reaching Action1
at all — suspect the integration layer before the API, and check whether the
runtime drops query params on operations that also take path params.

### Quick self-check before answering

- [ ] Did I send `from` and `limit` (limit ≤ 200)?
- [ ] Did the rows go through a code step and into a file, not into the chat?
- [ ] Did I compare `len(items)` with `int(total_items)`?
- [ ] Do the two match (or did I tell the user they don't)?

---

## Authentication

1. `POST action1:oauth2_token` with **form-encoded** body
   `client_id=...&client_secret=...`
   (`Content-Type: application/x-www-form-urlencoded` — JSON is rejected).
2. Response: `{"access_token": "...", "refresh_token": "...",
   "expires_in": 3600, "token_type": "bearer"}`.
3. Send `Authorization: Bearer <access_token>` on every other call.
4. The token lasts **1 hour**. Cache it for the run; re-request on `401`
   rather than on every call.
5. Credentials come from the Action1 console:
   *Configuration → Users & API Credentials*.

## Rate limits

Action1 recommends staying **below 30 requests per minute** per Enterprise.
Above that you get `429`.

- Sleep ~0.25 s between pages (already in `fetch_all`).
- On `429`: exponential backoff — 2 s, 4 s, 8 s, up to 3 retries.
- Prefer `limit=200` over twenty pages of 10 — but never go above 200 to
  save a round trip. A page you cannot read costs more than a second call.
- For very large reports prefer `report_export` (CSV, unpaginated) over
  `report_data_get`.

## Organization scoping — do this first

Almost every path contains `{organization_id}`. Resolve it once per run:

- `action1:me_get` → returns the caller's default organization.
- `action1:organizations_list` → drain it (yes, organizations paginate too)
  and cache the `name → id` map.
- Some endpoints accept the literal `all` in place of an organization id to
  span every org. Use it only when the user asked for a cross-org view.

If the user names an organization, resolve the name to its id before any
other call. Never pass a display name where an id is expected.

## How to Find Operations

- `discover(keyword="endpoint", api="action1", method="get")` → `endpoints_managed_list`
- `discover(keyword="update", api="action1", method="get")` → `missing_updates_list`
- `discover(keyword="vulnerability", api="action1")` → vulnerability operations
- `discover(keyword="policy", api="action1", method="post")` → `policies_create`

## Entity References

Action1 uses **opaque string IDs** (not integers, unlike NetBox). Never
build them by hand, never assume a format. Always obtain an id from a prior
GET. CVE ids are the exception — they are the natural key
(`CVE-2025-12345`) and go straight into the path.

## Filtering

Filtering is field-based: any field that appears inside an item can normally
be used as a query parameter.

- `?name=WKS-01` → exact match on endpoint name
- `?status=Connected` → only connected endpoints
- `?security_severity=Critical&approval_status=New` → combine with `&`
- `?endpoint_id=<id>` → scope updates to one machine

Action1 documents filters as exact `field=value` matching, ANDed together —
the docs state a multi-parameter call "returns records that match both
filtering criterias specified in the path". Operators, wildcards and negation
are not documented; assume they do not exist.

Filters and pagination are independent: **filtering does not remove the need
to paginate.** A filter that still matches 300 rows still returns 50.

**If a filter looks ignored, do not conclude that from one call.** Run the
A/B in *Verifying a parameter actually works* above. Two independent filters
failing on the same operation is the signature of a query string that never
reached Action1 — an integration layer dropping query params, not an API bug.
When exactness matters regardless, resolve the id from a list and read the
object with `endpoints_managed_get`.

## Extended data — the `fields` parameter

This is your main lever on response size. Use it deliberately.

| Value | Meaning |
|---|---|
| `fields=name,status,OS` | **project** — only these columns. Default choice for lists. |
| `fields=#` or omitted | default field set |
| `fields=*` (URL-encoded `%2A`) | everything, including extended data |

Extended fields available:
- `missing_critical_updates`, `missing_other_updates` — on managed endpoints
- `versions` — package versions in the software repository
- `outdated_endpoints` — endpoint counts per update

Supported on: `endpoints_managed_list`, `endpoints_managed_get`,
`endpoint_group_members_list`, `packages_list`, `packages_get_for_org`,
`package_versions_list`, `updates_catalog_list`, `missing_updates_list`.

### `fields=*` on a list is almost always the wrong call

On `endpoints_managed_list` it returns, per row: hardware, network,
`group_membership`, `custom` attributes, `missing_updates`, `vulnerabilities`,
`update_status`, `vulnerability_status`. That is hundreds of KB for a few
dozen machines, it costs server time, and it is the most common source of
bodies that break downstream.

Rules:

- **List + `fields=*` → only inside a code step that writes to a file.** Never
  as a call whose body you intend to read.
- **Project instead.** Pick the columns the question needs:

  | Question | `fields` |
  |---|---|
  | inventory / "what machines do we have" | `name,status,OS,agent_version,last_seen` |
  | patch compliance | `name,status,missing_critical_updates,missing_other_updates` |
  | vulnerability posture | `name,status,vulnerability_status` |
  | "just the ids" (phase 1 of a two-phase read) | `name` |

- **`fields=*` is fine on a single-object GET** (`endpoints_managed_get`) —
  one object is a bounded body, and this is the reliable way to see the full
  record for a machine.

### Don't compare rows fetched different ways

`fields=*` and the default set do not return the same fields, and a list row
is not always identical to the single-object read of the same machine
(`update_status` may read `UNDEFINED` in one and `SUCCESS` in the other).
Pick one read path and one parameter set for a given audit and stay on it.
Differences between heterogeneous calls are an artifact of the method, not a
finding about the fleet.

## Operations by Resource Type

- ORGANIZATIONS: `organizations_list`, `organizations_get`, `organizations_create`, `organizations_update`, `organizations_delete`
- IDENTITY: `me_get`, `users_list`, `enterprise_get`, `enterprise_update`
- ENDPOINTS: `endpoints_managed_list`, `endpoints_managed_get`, `endpoints_managed_update`, `endpoints_managed_delete`, `endpoints_status_list`, `endpoints_discovery_list`
- GROUPS: `endpoint_groups_list`, `endpoint_groups_get`, `endpoint_groups_create`, `endpoint_groups_update`, `endpoint_group_members_list`, `endpoint_group_members_add`, `endpoint_group_members_set`, `endpoint_group_members_remove`
- SOFTWARE INVENTORY: `installed_software_list`, `endpoint_software_list`, `installed_software_requery`
- REPOSITORY: `packages_list`, `packages_get_for_org`, `package_versions_list`, `scripts_list`
- UPDATES: `updates_catalog_list`, `missing_updates_list`, `installed_updates_list`, `installed_updates_requery`
- VULNERABILITIES: `vulnerabilities_list`, `vulnerabilities_get`, `vulnerability_endpoints_list`, `vulnerability_remediations_list`, `vulnerability_remediation_create`, `vulnerability_remediation_update`, `vulnerability_remediation_delete`
- POLICIES / DEPLOYMENTS: `policies_list`, `policies_get`, `policies_create`, `policy_results_list`
- AUTOMATIONS: `automations_list`, `automations_create`, `automations_update`, `automation_instances_list`
- REPORTS: `reports_list`, `report_data_get`, `report_export`, `report_requery`, `data_sources_list`
- OTHER: `logs_list`, `advanced_settings_list`, `search`

---

## Write Operations — Mandatory Safety Rules

Creating a policy instance **executes work on real production machines**
(installs software, patches, reboots). Before any `policies_create`,
`automations_create` or `vulnerability_remediation_create`:

1. Resolve and **name** the targets out loud: "this will run on 42 endpoints
   in group *Servers-EU*". Never silently expand to
   `[{"id": "ALL", "type": "EndpointGroup"}]`.
2. State the reboot behaviour explicitly.
3. Ask for confirmation. Wait for it.
4. After creation, report the policy id and poll `policy_results_list`
   (paginated!) for outcomes.

`DELETE` operations (`organizations_delete`, `endpoints_managed_delete`,
`vulnerability_remediation_delete`) always require explicit confirmation and
an echo of the object name.

### Body template — deploy software (`deploy_package`)

```json
{
  "name": "Deploy 7-Zip to Finance",
  "retry_minutes": "1440",
  "endpoints": [
    { "id": "<group_or_endpoint_id>", "type": "EndpointGroup" }
  ],
  "actions": [
    {
      "name": "Deploy Software",
      "template_id": "deploy_package",
      "params": {
        "display_summary": "",
        "packages": [ { "default": "default" } ],
        "reboot_options": { "auto_reboot": "no" }
      }
    }
  ]
}
```

### Body template — deploy updates (`deploy_update`)

```json
{
  "name": "Critical patches - EU servers",
  "retry_minutes": "1440",
  "endpoints": [
    { "id": "<group_or_endpoint_id>", "type": "EndpointGroup" }
  ],
  "actions": [
    {
      "name": "Deploy Update",
      "template_id": "deploy_update",
      "params": {
        "display_summary": "",
        "packages": [ { "default": "default" } ],
        "update_approval": "manual",
        "automatic_approval_delay_days": 7,
        "scope": "Specified",
        "reboot_options": {
          "auto_reboot": "yes",
          "show_message": "yes",
          "message_text": "Your computer requires maintenance and will be rebooted. Please save all work and reboot now to avoid losing any data.",
          "timeout": 240
        }
      }
    }
  ]
}
```

Targeting notes:
- `type` is `"Endpoint"` for a single machine, `"EndpointGroup"` for a group.
- `{"id": "ALL", "type": "EndpointGroup"}` means **every endpoint in the
  organization** — treat it as a blast radius, never as a default.
- `retry_minutes` is a **string**, not an int.

---

## Error Handling

| Status | Meaning | What to do |
|---|---|---|
| 400 | Malformed body / bad filter value | Echo the exact payload sent, fix the field, retry once |
| 401 | Token expired (1 h) or bad credentials | Re-run `oauth2_token`, retry once. Do not loop |
| 403 | Credential lacks permission for that organization | Tell the user which org and which operation |
| 404 | Wrong id, or wrong `organization_id` for that object | Re-resolve the id; ids are org-scoped |
| 405 | Wrong method (typical on `/requery` paths) | Retry the same path with POST |
| 429 | Rate limit (>30 req/min) | Exponential backoff 2/4/8 s, max 3 retries |
| 5xx | Server-side | Retry twice with backoff, then report |

Never silently swallow an error inside a pagination loop — a failed page
means the dataset is incomplete and the user must be told.

## Reporting Results to the User

Always lead with the coverage line:

```html
<div class="result-summary">
  <div class="stat stat-total">Endpoints: 247 / 247</div>
  <div class="stat stat-created">Pages fetched: 1</div>
  <div class="stat stat-skipped">Filters: status=Connected</div>
</div>
```

Endpoint listing:

```html
<strong>Managed endpoints (247 total)</strong>
<table>
  <thead><tr><th>Name</th><th>OS</th><th>Status</th><th>Agent</th><th>Last seen</th><th>Missing critical</th></tr></thead>
  <tbody>
    <tr><td><strong>WKS-EU-014</strong></td><td>Windows 11 Pro</td><td><span class="badge badge-active">Connected</span></td><td>5.2.1</td><td>2026-08-19 09:12</td><td><span class="badge badge-critical">3</span></td></tr>
  </tbody>
</table>
```

Single object detail:

```html
<div class="info-card">
  <strong>Endpoint Details</strong><br>
  <strong>Name:</strong> WKS-EU-014<br>
  <strong>OS:</strong> Windows 11 Pro<br>
  <strong>Status:</strong> <span class="badge badge-active">Connected</span><br>
  <strong>Agent:</strong> 5.2.1<br>
  <strong>Missing critical updates:</strong> 3
</div>
```

Rules:
- Never report a count without having drained pagination.
- If a result was capped, say so on the first line, not in a footnote.
- Round-trip the `self` URL of an object when the user may want to open it.

## Reports — `report_id` is NOT discoverable through the API

**Do not call `reports_list` hoping to find a report to export.** It returns
the report *category* tree (typically just the root `Built-in Reports`), not
concrete report definitions. `data_sources_list` is worse: it returns a huge
payload with long scripts embedded and no usable ids. Both are dead ends;
stop going down them.

Action1's own official example script,
`Action1Corp/APIScripts/ExportReportFromAllOrganizations.ps1`, does not call
`reports_list` at all. It does:

```powershell
$reportId = Read-Host -Prompt "Enter the Report ID"
```

and documents that the id is read out of the console URL:

```
https://app.eu.action1.com/console/reports/installed_software_1635264799139/summary
                                            └─────────── report_id ──────────┘
```

So a `report_id` is a **deployment-specific constant harvested once from the
console**, not something to discover at runtime.

### What to do

1. Keep the ids this tenant needs in a table right here in the skill, filled
   in once by a human:

   | Report | `report_id` |
   |---|---|
   | Managed endpoints | `<paste from console URL>` |
   | Installed software | `installed_software_<digits>` |
   | Missing updates | `<paste from console URL>` |

2. If an id is missing, **ask the user to copy it from the console URL**.
   That is a five-second request. Do not burn calls hunting for it.
3. `report_export` returns **raw CSV text, not JSON and not a ResultPage**.
   PSAction1 special-cases exactly two operations as non-JSON —
   `$Rawlist = @('ReportExport', 'Logs')` — and passes them through
   untouched. Read it as text and write it straight to a file; do not
   JSON-decode it, and do not paginate it.

## Exports

- For a report the user wants as a file, prefer `report_export` (CSV,
  unpaginated, one request) over draining `report_data_get` — provided you
  have a `report_id` from the console.
- For arbitrary collections: drain with `fetch_all` **inside a code step**,
  then `export_to_csv` / `export_to_xlsx`, then return the download link.
  The rows go to the file; only the counts go to the chat.
- Always include the row count in the message so the user can sanity-check
  it against the console, and say explicitly when `complete` was False.

## Recovering a full inventory when lists keep truncating

The reliable two-phase pattern, and the one to fall back on whenever a
runtime keeps clipping bodies:

1. **Phase 1 — ids only.** `endpoints_managed_list` with
   `fields=name`, `limit=200`, drained. Small rows, survives anything.
   Record `int(total_items)`.
2. **Phase 2 — one object at a time.** `endpoints_managed_get` per id with
   `fields=*`. Each response is a bounded, complete object.
3. **Consolidate in code**, write to CSV/XLSX, report
   `"N of total endpoints"` plus the file.

Phase 2 costs one request per endpoint — at 30 req/min that is fine for
hundreds of machines and wrong for thousands. Above ~500, get a `report_id`
from the console and use `report_export` instead.
