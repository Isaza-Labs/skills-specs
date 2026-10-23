# Skill: ServiceNow — incidents, changes and the CMDB

Scoped to the ServiceNow integration: this text reaches the prompt only when that
integration is in scope, not on every turn.

Spec: `servicenow`. Reach it the normal way — `discover_operations(api="servicenow")`,
`operation_detail(operation_id)`, `execute_operation`. Everything below is what the spec
cannot tell you.

## The three shapes of a ServiceNow answer

Every Table API read returns `{"result": [...]}` for a listing and `{"result": {...}}`
for a single record. There is no envelope beyond that: no count, no next link, no total.
Pagination is `sysparm_limit` + `sysparm_offset` and nothing else, so "is there more?"
is answered by asking for one row past the end, not by reading a field.

## sysparm_query is an encoded query

Not SQL, not JSON, not a filter object. One string:

```
active=true^priority IN 1,2^assignment_groupISNOTEMPTY^ORDERBYDESCsys_updated_on
```

`^` is AND, `^OR` is OR, `^ORDERBY` / `^ORDERBYDESC` sort. Operators: `=`, `!=`, `<`,
`>`, `<=`, `>=`, `LIKE`, `STARTSWITH`, `ENDSWITH`, `IN` (comma-separated, no spaces
after the commas), `ISEMPTY`, `ISNOTEMPTY`. Dates take ServiceNow's own helpers —
`sys_updated_on>=javascript:gs.beginningOfToday()`.

Two failure modes worth knowing, because neither looks like the mistake it is:

- **A misspelled field is not an error.** ServiceNow drops the unrecognised clause and
  runs the rest, so `priorty=1` returns every incident and reads as "the filter found
  nothing to exclude". If a result set looks too broad, suspect the query before
  suspecting the data.
- **A reference field compares by sys_id, not by name.** `assignment_group=Network`
  matches nothing. Resolve the name first (`servicenow_group_list`), then filter on the
  sys_id — or use the dot-walk, `assignment_group.name=Network`, which does work.

## Always send sysparm_limit and sysparm_fields

Without `sysparm_limit` the default is **10000 rows**. Without `sysparm_fields` each row
carries ~90 columns, most of them empty strings, each reference field doubled by a
`link` URL. One unbounded read of `incident` will spend the whole context window on
nulls.

Ask for what you need:

```
sysparm_limit=25
sysparm_fields=number,short_description,state,priority,assignment_group,sys_updated_on
sysparm_exclude_reference_link=true
```

## display_value decides whether the answer is readable or usable

- `sysparm_display_value=false` (the default) — `state` is `2`, `assignment_group` is a
  32-hex sys_id. This is what you want when the value feeds another call.
- `=true` — `state` is `In Progress`, `assignment_group` is `Network Support`. This is
  what you want when a human reads the answer.
- `=all` — both, as `{value, display_value}` per field. Twice the payload; use it only
  when you genuinely need to show a label and act on the id in the same pass.

Reporting raw state codes to an operator is the most common way a correct answer reads
as wrong. Ask for display values when you are going to speak them out loud.

## Numbers are not ids

`INC0010023` is the `number` field. `sys_id` is the 32-hex primary key, and it is what
every `{sys_id}` path and every reference field wants. Going from one to the other is a
list call:

```
servicenow_incident_list  sysparm_query=number=INC0010023  sysparm_fields=sys_id,number  sysparm_limit=1
```

Do that lookup rather than guessing. A sys_id that does not exist is a 404, but a sys_id
belonging to a *different* record is a successful update of the wrong thing.

## Writes

- **PATCH, never PUT.** The spec exposes only PATCH for a reason: PUT replaces the
  record, so every field you did not send is cleared. That has quietly wiped assignment
  and work notes on people who only meant to move the state.
- **`work_notes` and `comments` append.** They are journal fields, not text columns —
  writing to one adds an entry, it never overwrites. That makes `work_notes` the right
  place for a workflow to record what it did.
- **State transitions carry required fields.** Resolving an incident (`state=6`) needs
  `close_code` and `close_notes`; putting one on hold (`state=3`) needs `hold_reason`.
  The instance rejects the transition without them, and the message names the field.
- **There is no delete.** Not exposed, on purpose. ServiceNow records get closed,
  resolved or cancelled — deleting an incident destroys the audit trail that was the
  reason for opening it. If someone asks to delete one, cancel it (`state=8`) and say
  that is what you did.

## Change requests gate the risky work

A network change that needs sign-off is a `normal` change: create it with
`servicenow_change_create`, then read `approval` back with `servicenow_change_get`
before anything touches a device. `approval=approved` and `state=-2` (Scheduled) is the
green light. Do not treat "the change was created" as "the change was approved" —
creating it is exactly the step that means it is not yet.

`standard` changes are pre-approved from a template and `emergency` shortcuts CAB. Use
what the operator asked for; do not upgrade a normal change to emergency to avoid a
wait.

Dates on a change are `YYYY-MM-DD HH:MM:SS` in **UTC**, with a space and no `Z`. An
ISO-8601 string with a `T` is rejected.

## The CMDB, and reconciling it with Nashira's inventory

`servicenow_netgear_list` is the network slice (`cmdb_ci_netgear`): switches, routers,
firewalls. Match against Nashira devices on `ip_address` first, `serial_number` second,
`name` last — names drift, serials do not. `install_status=1` means installed; anything
else is retired, on order, or in stock, and pinging it is a waste.

`servicenow_ci_list` reads the base `cmdb_ci` table and so returns every class. The
class is in `sys_class_name`, and reading a subclass table directly is cheaper than
filtering the base one.

Reconciliation is a **read** on both sides. Report the drift; do not create CIs or
devices to close the gap unless asked.

## Reading a table nobody enumerated

`servicenow_table_list` and `servicenow_table_get` take any table name — `sc_request`,
`cmdb_rel_ci`, `sys_journal_field`, a custom `u_` table. Reads only. When an operator
asks for something the named operations do not cover, use it rather than reporting that
the integration cannot do it.

Writes stay on the named operations. A generic write would be one action the risk gate
could say nothing useful about, and "PATCH some row in some table" is not a thing to ask
a human to confirm.

## When a call is refused

- **401, "User is not authenticated".** The credential was not accepted. On an instance
  with **Basic Auth Restriction** enabled — the default on new instances since 2026 —
  this is *also* what a correct username and password get when that user has no Basic
  Auth exception, which is why the UI login works and the API does not. Check the
  integration's auth method before anyone re-types the password. The fix is an OAuth
  client, not a new secret.
- **403.** Authenticated; an ACL refused the table, the row or a field. The integration's
  user needs a role — `itil` for incidents and changes, `cmdb_read` or `asset` for the
  CMDB. A different fault from 401 with a different fix.
- **400 on a write.** Almost always a field name the table does not have, or a reference
  that did not resolve. The body names the field.
- **A 200 with an empty `result`.** The query ran and matched nothing — which, given
  that a misspelled field is silently dropped, can also mean the query matched
  everything and something *else* excluded it. Re-read the query before concluding the
  record does not exist.
