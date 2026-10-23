# NetBox — Inventory & CMDB

API: `netbox` — operations are called as `netbox:<operation_id>`.
Domain: everything that happens inside the NetBox application (devices, sites, interfaces, cables, IPAM, circuits, tenants, tags, custom fields).
Target version: NetBox 4.4.

## NetBox REST API basics

- **GET** lists or retrieves, **POST** creates, **PATCH** updates part of a record (needs the integer ID in the path).
- Every list returns `{"count": N, "next": "...", "previous": "...", "results": [...]}`.
- References between objects are **integer IDs**. To create a device you need the IDs of its `site`, `device_type` and `role`; if you only have a name, GET the referenced object first.

## Finding and calling operations

- `discover_operations(keyword="device", api="netbox", method="GET")` → finds `dcim_devices_list`
- `operation_detail(operation_id="netbox:dcim_devices_list")` → parameters and body schema
- `execute_operation(operation_id="netbox:dcim_devices_list", query_params={"site": "atlanta", "limit": 100})`
- `execute_operation(operation_id="netbox:dcim_devices_partial_update", path_params={"id": 25774}, body={"status": "active"})`

Query values can be strings, numbers or booleans. A list repeats the key: `{"status": ["active", "planned"]}`.
GET calls are read-only; POST and PATCH change NetBox and go inside the plan the user confirms.

## Operations by resource

| Resource | Operations |
| --- | --- |
| Devices | `dcim_devices_list`, `dcim_devices_retrieve`, `dcim_devices_create`, `dcim_devices_partial_update` |
| Interfaces | `dcim_interfaces_list`, `dcim_interfaces_retrieve`, `dcim_interfaces_create`, `dcim_interfaces_partial_update` |
| Cables | `dcim_cables_list`, `dcim_cables_retrieve`, `dcim_cables_create`, `dcim_cables_partial_update` |
| Sites | `dcim_sites_list`, `dcim_sites_retrieve`, `dcim_sites_create`, `dcim_sites_partial_update` |
| Device types | `dcim_device_types_list`, `dcim_device_types_retrieve`, `dcim_device_types_create`, `dcim_device_types_partial_update` |
| Device roles | `dcim_device_roles_list`, `dcim_device_roles_retrieve`, `dcim_device_roles_create`, `dcim_device_roles_partial_update` |
| Manufacturers | `dcim_manufacturers_list`, `dcim_manufacturers_retrieve`, `dcim_manufacturers_create`, `dcim_manufacturers_partial_update` |
| Platforms | `dcim_platforms_list`, `dcim_platforms_retrieve`, `dcim_platforms_create`, `dcim_platforms_partial_update` |
| IP addresses | `ipam_ip_addresses_list`, `ipam_ip_addresses_retrieve`, `ipam_ip_addresses_create`, `ipam_ip_addresses_partial_update` |
| Prefixes | `ipam_prefixes_list`, `ipam_prefixes_retrieve`, `ipam_prefixes_create`, `ipam_prefixes_partial_update` |
| IP ranges | `ipam_ip_ranges_list`, `ipam_ip_ranges_retrieve`, `ipam_ip_ranges_create`, `ipam_ip_ranges_partial_update` |
| VLANs | `ipam_vlans_list`, `ipam_vlans_retrieve`, `ipam_vlans_create`, `ipam_vlans_partial_update` |
| VRFs | `ipam_vrfs_list`, `ipam_vrfs_retrieve`, `ipam_vrfs_create`, `ipam_vrfs_partial_update` |
| Tenants | `tenancy_tenants_list`, `tenancy_tenants_retrieve`, `tenancy_tenants_create`, `tenancy_tenants_partial_update` |
| Circuits | `circuits_circuits_list`, `circuits_circuits_retrieve`, `circuits_circuits_create`, `circuits_circuits_partial_update` |
| Circuit types | `circuits_circuit_types_list`, `circuits_circuit_types_retrieve`, `circuits_circuit_types_create`, `circuits_circuit_types_partial_update` |
| Providers | `circuits_providers_list`, `circuits_providers_retrieve`, `circuits_providers_create`, `circuits_providers_partial_update` |
| Tags | `extras_tags_list`, `extras_tags_retrieve`, `extras_tags_create`, `extras_tags_partial_update` |
| Custom fields | `extras_custom_fields_list`, `extras_custom_fields_retrieve`, `extras_custom_fields_create`, `extras_custom_fields_partial_update` |

There are no DELETE operations: removing data from NetBox is done by a person.
To bring NetBox devices into the Nashira device inventory, use `sync_netbox_inventory` instead of copying them one by one.

## Writing to NetBox

Every resource supports create (`*_create`) and partial update (`*_partial_update`). Writes change
NetBox: they go in the plan the user confirms. Before creating, check the object does not already exist
(list by name, slug, address or prefix); NetBox refuses duplicates of unique fields with a 400.

**Cable between two interfaces.** Get both interface IDs with `dcim_interfaces_list` (`device` + `name`),
check neither has a cable (`cable` is null), then:

    execute_operation(operation_id="netbox:dcim_cables_create", body={"a_terminations": [{"object_type": "dcim.interface", "object_id": 21941}], "b_terminations": [{"object_type": "dcim.interface", "object_id": 21943}], "status": "connected", "label": "srl1-srl2"})

**IP address on an interface, and primary IP.** Create the address assigned to the interface, then PATCH
the device with the new IP's ID as `primary_ip4` (it must already be assigned to one of the device's interfaces):

    execute_operation(operation_id="netbox:ipam_ip_addresses_create", body={"address": "10.0.12.0/31", "assigned_object_type": "dcim.interface", "assigned_object_id": 21941, "status": "active"})
    execute_operation(operation_id="netbox:dcim_devices_partial_update", path_params={"id": 25774}, body={"primary_ip4": <ip address id>})

**Prefix, VLAN, VRF.** `ipam_prefixes_create` with `prefix` (CIDR) and optionally `scope_type` + `scope_id`
(for example `dcim.site` + site ID), `vrf`, `vlan`, `status`. `ipam_vlans_create` needs `vid` (1-4094) and
`name`. `ipam_vrfs_create` needs `name`; `rd` is optional.

**Interface.** `dcim_interfaces_create` needs `device` (ID), `name` and `type` (for example `1000base-t`,
`10gbase-x-sfpp`, `virtual`).

**Update.** `*_partial_update` sends only the fields that change, with the object's integer ID in the path.
For lists such as `tags`, send the full new list: what you send replaces what was there.

## Filters

- `name=router-01` exact match (case-sensitive); `q=router` free-text search, case-insensitive.
- `site`, `role`, `platform`, `tenant`, `tag` filter by **slug**; `site_id`, `role_id`, `device_id` by integer ID.
- `status=active` (devices: active, planned, staged, failed, inventory, decommissioning, offline).
- Providers in NetBox 4.x: `accounts` and `asns` are lists of IDs; the old `account`, `asn`, `portal_url` and `noc_contact` fields no longer exist.

## Pagination

- `limit` defaults to 50; the server maximum is 1000. `offset` skips records.
- For "every device", send `limit=1000` and compare the number of results with `count`; if `count` is larger, call again with `offset` until you have them all. Never assume one page holds everything.
- When you report a list, say how many were returned out of `count`.

## Recipes

**What is connected to a port.** `dcim_interfaces_list` with `device=<name>` and `name=<interface>`. In the result, `link_peers` holds the far-end interface and its `device`; an empty `link_peers` (or `cable: null`) means NetBox has no cable for that port.

**Tag a device without losing its tags.** `dcim_devices_retrieve` (or list by name), take its current `tags` slugs, add the new one and remove any tag it replaces, then `dcim_devices_partial_update` with the full list as `[{"slug": "..."}]`. If the tag does not exist yet, check `extras_tags_list` with `slug=...` and create it with `extras_tags_create`.

**Primary IP of a device.** `primary_ip4.address` on the device (CIDR form, for example `10.0.0.1/24`); it is null when no primary IP is set.

**Reports.** For a table (CSV, XLSX, PDF…) use `export_table` with `columns` and one object per row; use `save_report` to list it in /reports. To read a CSV or XLSX the user uploaded, use `parse_file`, show a preview and the row count, and wait for confirmation before creating anything.

## Custom fields — mandatory rules

Custom fields are declared per object type (`dcim.device`, `dcim.site`, ...). Sending an undeclared key fails the whole create or update with `400 {"__all__": ["Unknown field name 'X' in custom field data."]}`.

Before any payload with `custom_fields`:

1. `execute_operation(operation_id="netbox:extras_custom_fields_list", query_params={"object_type": "dcim.device", "limit": 1000})`.
2. Keep keys that are declared.
3. For each undeclared key:
   - write tasks (onboarding, the user expects the data to land): create it with `netbox:extras_custom_fields_create` (`name`, `type`, `object_types`) and continue;
   - read-only or sync tasks: drop the key and report `custom_field 'X' not registered; skipped`.
4. Never create a custom field that already exists.

| Sample value | NetBox `type` |
| --- | --- |
| `"v23.10.2"` | `text` |
| multi-line output, configs | `longtext` |
| `42` | `integer` |
| `True` / `False` | `boolean` |
| `"2026-04-29"` | `date` |
| `dict` or `list` | `json` |

When the value comes from a device command with structured parsing, use the parsed field (for example the software version), not the raw multi-line output, unless the field is `longtext` and the goal is to keep the full command result.

## Creating devices

Required: `name`, `device_type`, `role`, `site` (IDs). Optional: `status`, `tenant`, `platform`, `serial`, `asset_tag`, `tags`, `custom_fields`.
If a referenced device type, role or site does not exist, tell the user which one is missing and ask whether to create it first; create dependencies before the device.
