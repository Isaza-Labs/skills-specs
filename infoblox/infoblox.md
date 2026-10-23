# Infoblox — WAPI v2.12

API prefix: `infoblox:`
Base path: `/wapi/v2.12/` (compatible with v2.10, v2.11, v2.12)
Domain: Everything inside the Infoblox WAPI: DNS (views, zones, records), IPAM/DHCP (IPv4 and IPv6), Grid, global search, and Sites/Stores.

## Tool naming convention

All WAPI REST calls are made through `discover` / `detail` / `execute` using the `operationId` defined in `infoblox.yaml`. **Never invent tool names** — always call `discover(keyword="...", api="infoblox")` first.

| Intent | operationId |
|---|---|
| Health check (lab mock extension) | `health` |
| Grid info | `grid_read` |
| Global search | `search` |
| All DNS records | `allrecords` |
| List DNS views | `view_list` |
| DNS view CRUD | `view_create`, `view_read`, `view_update`, `view_patch`, `view_delete` |
| Authoritative zones | `zone_auth_list/create/read/update/patch/delete` |
| Delegated zones | `zone_delegated_list/create/read/update/patch/delete` |
| Forward zones | `zone_forward_list/create/read/update/patch/delete` |
| A record | `record_a_list/create/read/update/patch/delete` |
| AAAA record | `record_aaaa_list/create/read/update/patch/delete` |
| CNAME record | `record_cname_list/create/read/update/patch/delete` |
| Host record | `record_host_list/create/read/update/patch/delete` |
| NS record | `record_ns_list/create/read/update/patch/delete` |
| PTR record | `record_ptr_list/create/read/update/patch/delete` |
| MX record | `record_mx_list/create/read/update/patch/delete` |
| TXT record | `record_txt_list/create/read/update/patch/delete` |
| SRV record | `record_srv_list/create/read/update/patch/delete` |
| Network view | `networkview_list/create/read/update/patch/delete` |
| Network (IPv4) | `network_list/create/read/update/patch/delete` |
| Next available IP | `network_next_available_ip` |
| Network container | `networkcontainer_list/create/read/delete` |
| Next available network | `networkcontainer_next_available_network` |
| IPv4 address (RO) | `ipv4address_list`, `ipv4address_read` |
| Fixed address (IPv4) | `fixedaddress_list/create/read/update/patch/delete` |
| DHCP range (IPv4) | `range_list/create/read/update/patch/delete` |
| Lease (RO) | `lease_list` |
| Network (IPv6) | `ipv6network_list/create/read/update/patch/delete` |
| IPv6 address (RO) | `ipv6address_list`, `ipv6address_read` |
| Fixed address (IPv6) | `ipv6fixedaddress_list/create/read/update/patch/delete` |
| DHCP range (IPv6) | `ipv6range_list/create/read/update/patch/delete` |
| Site (lab mock extension) | `site_list/create/read/update/delete` |
| Site provisioning (lab mock extension) | `site_provision` |
| Site summary (lab mock extension) | `site_summary` |

The `health` check and the `/site` endpoints are extensions of the lab mock, not part of the standard Infoblox WAPI.

## Authentication

All requests require **HTTP Basic Auth**. Credentials are provided by the integration configuration — never hardcode or guess them.
- Real Infoblox appliances serve the WAPI over HTTPS; the lab mock may serve plain HTTP.
- Only Basic Auth is supported here: no client certificates, SAML, or tokens.

## Global query parameters

These are accepted by almost every `_list` / `_read` endpoint:

| Parameter | Purpose |
|---|---|
| `_return_fields` | Explicit list of fields (replaces the defaults) |
| `_return_fields+` | Adds fields to the defaults (encode `+` as `%2B`) |
| `_max_results` | Result limit (default 1000) |
| `_paging` | Enables paging; requires `_return_as_object=1` |
| `_return_as_object` | Wraps the response in `{"result": [...]}` |
| `_page_id` | Continues a previous paged query |

## Search modifiers (field-name suffixes)

| Suffix | Semantics | Example |
|---|---|---|
| `=` | Exact match | `name=server1.example.com` |
| `~=` | Regex / contains | `name~=^web` |
| `:=` | Case-insensitive exact | `name:=SERVER1` |
| `~:=` | Case-insensitive like | `comment~:=production` |
| `!=` | Negation | `status!=UNUSED` |
| `<=` / `>=` | Numeric range | `utilization>=90` |
| `*Attr=` | By extensible attribute | `*Site=AWS` |

When building URLs in code, **encode `+` as `%2B`**, and remember `_return_as_object=1` whenever you enable `_paging=1`.

## CRITICAL: how objects are identified

Infoblox does NOT use numeric IDs — every object is identified by its **`_ref`** (an opaque base64-like string, e.g. `record:a/ZG5zLmJpbmRfYSQu...:server1.example.com/default`).

- NEVER assume or invent a `_ref`.
- ALWAYS call the matching `*_list` first, filtering by `name` / `fqdn` / `network` / `ipv4addr`, to resolve the `_ref`.
- Then use that exact `_ref` in `*_read` / `*_update` / `*_patch` / `*_delete`.
- The `_ref` must be URL-encoded if it contains special characters (`:`, `/`, etc.).
- If the `*_list` returns 0 results, tell the user the object does not exist — do not try to create it or guess.

## PUT vs PATCH

- `PUT` replaces the whole object — every required field must be in the body.
- `PATCH` is a partial update — only the fields to change.
- **Default: prefer `PATCH`** unless the user explicitly asks for a full replacement. It carries less risk of losing fields (TTL, extattrs, comment).

## Global search (`/search`)

A single endpoint that searches across all DNS object types at once. Use it when:
- The user does not know the exact resource type ("find everything containing `mail`").
- You want to cover several types in one call (`objtype=record:a,record:cname`).

Types supported by `/search`: `record:a`, `record:aaaa`, `record:cname`, `record:host`, `record:mx`, `record:txt`, `record:ptr`, `record:srv`, `record:ns`, `zone_auth`, `zone_delegated`, `zone_forward`.

If the user wants IPs or networks → do NOT use `/search` (it does not cover IPAM); use `ipv4address_list` / `network_list` directly.

## Allrecords

`allrecords` returns all DNS records, optionally filtered by `zone` and/or `view`. Useful for:
- Auditing an entire zone.
- Exporting all RRs in a view.
Avoid it if you already know the type (it is more expensive than `record:<type>_list`).

## Special functions (`_function=...`)

Some operations are not plain CRUD but **functions invoked with `POST` + `?_function=...`** on a `_ref`:

### `next_available_ip` (on `network`)
```
POST /network/{_ref}?_function=next_available_ip
Body: {"num": 5, "exclude": ["192.168.1.10"]}
```
Reserves N free IPs from the network. Use it before creating a `fixedaddress` / `record:a` when the IP is not predetermined.

### `next_available_network` (on `networkcontainer`)
```
POST /networkcontainer/{_ref}?_function=next_available_network
Body: {"cidr": 24, "num": 3}
```
Returns N free subnets of size `cidr` inside the container. Use it before `network_create` when you delegate allocation to IPAM.

**Rule**: function arguments go in the JSON body, NOT in the query string (except `_function=` itself).

## Read-only objects

These types do **not** accept `POST`/`PUT`/`PATCH`/`DELETE` — they are views computed by the WAPI:
- `ipv4address` / `ipv6address`: IP state derived from records + ranges + fixedaddresses.
- `lease`: active/expired DHCP leases.

If the user asks to "create an ipv4address" → explain that it is created via `record:a`, `record:host`, `fixedaddress`, or `range`, and the state then shows up in `ipv4address_list`.

## Extensible Attributes (extattrs)

- Key/value pairs defined by the organization (`Site`, `Owner`, `Environment`, etc.).
- To filter: `*<AttrName>=<value>` in the query (e.g. `*Site=AWS`).
- To read them: add `extattrs` to `_return_fields+`.
- To write them in POST/PATCH: an `extattrs` field formatted as `{"Site": {"value": "AWS"}, "Owner": {"value": "neteng"}}`.

## Network views vs DNS views

Do not confuse them:
- `networkview` → logical separation of the IP space (IPAM/DHCP). Default: `"default"`.
- `view` → logical separation of the DNS space (split-horizon). Default: `"default"`.
Both coexist. If the user works with overlapping networks (`10.0.0.0/8` in two contexts), use `networkview`.

## Sites/Stores (lab mock extension)

The `/site` endpoints are NOT standard Infoblox WAPI — they are an extension of the lab mock that simulates retail store provisioning.

- A `site` wraps a `networkcontainer` and supports automatic subnet provisioning via `POST /site/{id}/provision?subnet_cidr=24&num_subnets=2`.
- `GET /site/{id}/summary` returns aggregated utilization.
- There is no `PATCH` — only `PUT` for updates.
- The identifier is `site_id` (a string with the store code, e.g. `CO-MDE-001`), not a `_ref`.

## Important Rules

1. **Before creating/modifying/deleting → confirm with the user** (POST, PUT, PATCH, DELETE). GET/LIST operations do not require confirmation.
2. **After each action → show the results**: `_ref`, `name`, HTTP status, relevant fields (IP, FQDN, network).
3. **At the end → a summary** of every affected resource (created, updated, deleted).
4. **Validate required inputs before the POST**:
   - `record:a` → `name` + `ipv4addr`
   - `record:aaaa` → `name` + `ipv6addr`
   - `record:cname` → `name` + `canonical`
   - `record:host` → `name` + `ipv4addrs[]` (or `ipv6addrs[]`)
   - `record:ptr` → `ptrdname` + `ipv4addr`/`ipv6addr`
   - `record:mx` → `name` + `mail_exchanger` + `preference`
   - `record:srv` → `name` + `target` + `port` + `priority` + `weight`
   - `zone_auth` → `fqdn`
   - `zone_delegated` → `fqdn` + `delegate_to[]`
   - `zone_forward` → `fqdn` + `forward_to[]`
   - `network` → `network` (CIDR)
   - `fixedaddress` → `ipv4addr` + `mac`
   - `range` → `start_addr` + `end_addr` + `network`
   - `ipv6fixedaddress` → `ipv6addr` + `duid`

## Main Workflows

### 1. Create a simple A record
1. (Optional) `zone_auth_list(fqdn="example.com")` to check that the zone exists.
2. `record_a_create({name, ipv4addr, comment?, view?, ttl?, extattrs?})`.
3. Show the returned `_ref`.

### 2. Create a multi-NIC host with DHCP reservation
1. `record_host_create({name, ipv4addrs: [{ipv4addr, mac, configure_for_dhcp: true}, ...]})`.
2. Verify with `ipv4address_list(network="...")` that the IPs show as `USED`.

### 3. Assign the next free IP
1. `network_list(network="192.168.1.0/24")` → get the `_ref`.
2. `network_next_available_ip(_ref, {num: 1})` → get the IP.
3. `record_a_create({name, ipv4addr: <ip>})` or `fixedaddress_create({ipv4addr: <ip>, mac})`.

### 4. Create a subnet automatically inside a container
1. `networkcontainer_list(network="10.0.0.0/8")` → `_ref`.
2. `networkcontainer_next_available_network(_ref, {cidr: 24, num: 1})` → free CIDR.
3. `network_create({network: <cidr>, comment, extattrs})`.
4. (Optional) `range_create(...)` + `record_a_create(...)`.

### 5. Provision a retail store (lab mock extension)
1. `site_create({site_id, name, location, region, network_container})`.
2. `site_provision(site_id, subnet_cidr=24, num_subnets=2)`.
3. `site_summary(site_id)` to check utilization.

### 6. Audit an entire zone
1. `zone_auth_list(fqdn="corp.local")` → confirm it exists.
2. `allrecords(zone="corp.local", view="default", _max_results=1000)`.
3. Group by the `_ref` type to report (A, CNAME, MX...).

### 7. Fuzzy search (type unknown)
1. `search(search_string~="mail", objtype="record:a,record:cname,record:mx", _max_results=100)`.
2. If the user searches by partial IP: `search(search_string~="192.168.1")`.
3. If nothing is found → suggest `ipv4address_list` (bare IPs are not covered by `/search`).

### 8. Update a record's TTL
1. `record_a_list(name="server1.example.com")` → `_ref`.
2. `record_a_patch(_ref, {ttl: 7200, use_ttl: true})`.
3. Re-read with `record_a_read(_ref)` to confirm.

### 9. Clean up a dangling record
1. `record_a_list(name="old.example.com")` → `_ref`.
2. Confirm with the user (DELETE is irreversible unless a backup exists).
3. `record_a_delete(_ref)`.

### 10. Page through large result sets
1. First page: `record_a_list(_paging=1, _max_results=100, _return_as_object=1)`.
2. Read `next_page_id` from the response.
3. Next page: `record_a_list(_paging=1, _max_results=100, _return_as_object=1, _page_id=<id>)`.
4. Repeat until no `next_page_id` is returned.

## Lookup before Update/Delete

- NEVER assume a `_ref`.
- ALWAYS call the matching `*_list` first to resolve it by `name` / `fqdn` / `network`.
- If the `*_list` returns 0 → tell the user the object does not exist.
- If it returns >1 → show the candidates and ask the user to choose.

## IP status (ipv4address.status)

Possible values: `USED`, `UNUSED`, `ACTIVE`, `MATCH`, `UNMATCHED`.
- `USED`: assigned (record, fixedaddress, lease).
- `UNUSED`: free for assignment.
- `ACTIVE`: recent traffic seen (if discovery is enabled).

## Lease binding state

Values of `lease.binding_state`: `ACTIVE`, `FREE`, `EXPIRED`, `RELEASED`, `RESET`, `BACKUP`, `ABANDONED`.

## Paging and limits

- Exhaustive queries ("all A records in zone X") → `_max_results=1000` + paging.
- Searches by name → `_max_results=50`.
- Lists for user selection → `_max_results=100`.
- NEVER rely on the default (1000) for large volumes without paging: timeouts are likely.

## Lab mock limitations (differences from real Infoblox)

Tell the user if they ask for any of these while working against the lab mock:
- Mandatory HTTPS → the mock may serve plain HTTP.
- Grid replication, DDNS updates, CSV import, Scheduling, Approval workflow → NOT implemented.
- Certificate / SAML authentication → NOT supported.
- `Discovered data` → partial.

## When an operation fails

Diagnostic sequence:
1. Check the HTTP status: `400` → invalid payload; `401` → auth; `404` → `_ref` does not exist; `409` → conflict (duplicate); `500` → server error.
2. On `400`: check that the required fields are present (see Important Rules #4) and that types match (`ipv4addr` is a string, `preference` is an int, `ipv4addrs` is an array of objects).
3. On `404` from a `*_update`/`*_delete`: refresh the `_ref` with `*_list`.
4. On `409` when creating: most likely a duplicate — run `*_list` on the unique key and report the existing object.
5. Show the raw WAPI error (`text/plain` with the message) to the user.

## Compatibility with `infoblox-client` (Python)

When the user wants to generate client code:
```python
from infoblox_client import connector
opts = {'host': '<grid-master-host>', 'username': '<username>',
        'password': '<password>', 'wapi_version': '2.12', 'ssl_verify': True}
conn = connector.Connector(opts)
```
- `conn.create_object('record:a', {...})` → POST
- `conn.get_object('record:a', {'name~':'...'})` → GET with filters
- `conn.call_func('next_available_ip', ref, {'num':3})` → `_function=` functions

## Resetting state

Real Infoblox has no WAPI call to reset state. If the user asks to wipe the lab mock, that is an environment-specific, destructive operation outside the WAPI — always confirm first and follow the lab's own reset procedure.
