# AWX — Automation, Git & Ansible

API prefix: `awx:`
Domain: Everything that happens INSIDE AWX/Ansible Tower AND all Git/GitHub operations.

## Tool naming convention

All AWX REST calls go through `discover` / `detail` / `execute` using the `operationId`
defined in `awx.yaml`. **Never guess tool names** — always `discover(keyword="...", api="awx")`
first. The old names like `get_jobs` / `get_job_templates` / `get_inventories` do NOT
exist as tools; use the operationIds below.

| Intent | operationId |
|---|---|
| List jobs | `jobs_list` |
| Job details | `jobs_read` |
| Job stdout | `jobs_stdout` |
| Job events | `jobs_events` |
| Cancel / relaunch job | `jobs_cancel`, `jobs_relaunch` |
| List job templates | `job_templates_list` |
| Job template details | `job_templates_read` |
| Launch schema / launch | `job_templates_launch_schema`, `job_templates_launch` |
| List inventories | `inventories_list` |
| Inventory details | `inventories_read` |
| Inventory hosts / groups / sources | `inventories_hosts_list`, `inventories_groups_list`, `inventories_sources_list` |
| List / read host | `hosts_list`, `hosts_read` |
| List / read group | `groups_list`, `groups_read` |
| List / read credential | `credentials_list`, `credentials_read` |
| List / read project | `projects_list`, `projects_read` |
| Project playbooks | `projects_playbooks` |
| Project sync trigger / status | `projects_sync`, `projects_updates_list`, `project_updates_read`, `project_updates_stdout` |
| List / read inventory source | `inventory_sources_list`, `inventory_sources_read` |
| Inv-source sync trigger / status | `inventory_sources_sync`, `inventory_updates_list`, `inventory_updates_read`, `inventory_updates_stdout` |
| Workflow template CRUD | `workflow_job_templates_list/read/create/update/delete` |
| Workflow launch / monitor | `workflow_job_templates_launch`, `workflow_jobs_list`, `workflow_jobs_read`, `workflow_jobs_nodes_list` |
| Workflow DAG nodes | `workflow_job_template_nodes_*`, `workflow_nodes_success_attach`, `workflow_nodes_failure_attach`, `workflow_nodes_always_attach` |

## CRITICAL: "EXECUTED" vs "CREATED" Queries

When the user asks about "executed", "ran":
- "templates executed today" → Use `jobs_list` with `started__gte/started__lte` (Templates don't execute — JOBS do)
- "inventories executed today" → Use `inventory_updates_list` filtered by date
- "jobs executed today" → Use `jobs_list` with `started__gte/started__lte`

When asking about "created":
- "templates created today" → `job_templates_list` with `created__gte/created__lte`
- "inventories created today" → `inventories_list` with `created__gte/created__lte`

**Date filter types for jobs:**
- `created__gte/lte`: when the job record was CREATED (queued)
- `started__gte/lte`: when the job actually STARTED executing (use this for "ran today")
- `finished__gte/lte`: when the job FINISHED

## Important Rules

1. BEFORE MODIFYING PLAYBOOKS — ASK FOR CONFIRMATION:
   - Only ask for confirmation before creating/modifying/deleting playbooks (`git_file_write`, `git_push`)
   - Do NOT ask for confirmation for GET/LIST operations

2. AFTER EACH ACTION — SHOW DETAILED RESULTS:
   - IDs, names, URLs, status codes
   - Example: "Repository created: my-playbook (ID: 123, URL: https://...)"

3. AT THE END — PROVIDE SUMMARY with all affected resources

## REST Capabilities (via discover/execute)

- **Health & Status**: `ping`, `config`, `me`, `dashboard`, `settings_list`, `settings_read`
- **Organizations**: full CRUD (`organizations_list/read/create/update/delete`)
- **Projects**: full CRUD + `projects_sync`, `projects_playbooks`, `projects_updates_list`, `project_updates_read/stdout/cancel`
- **Inventories**: full CRUD + sub-resources (`inventories_hosts_list`, `inventories_groups_list`, `inventories_sources_list`, `inventories_variable_data_read/update`)
- **Hosts**: full CRUD + `hosts_variable_data_read/update`
- **Groups**: full CRUD + `groups_add_host`, `groups_hosts_list`, `groups_children_list`
- **Credentials**: full CRUD + `credential_types_list`
- **Job Templates**: full CRUD + `job_templates_launch`, `job_templates_launch_schema`, `job_templates_associate_credential`, `job_templates_jobs_list`, `job_templates_schedules_list`
- **Jobs**: list/read/stdout/events/cancel/relaunch
- **Workflow Job Templates**: full CRUD + `workflow_job_templates_launch`
- **Workflow Jobs**: `workflow_jobs_list/read/nodes_list/cancel/relaunch`
- **Workflow DAG nodes**: `workflow_job_template_nodes_list/read/create/update/delete` + `workflow_nodes_success_attach`, `workflow_nodes_failure_attach`, `workflow_nodes_always_attach`
- **Workflow Approvals**: `workflow_approvals_list/approve/deny`
- **Schedules**: full CRUD
- **Inventory Sources**: full CRUD + `inventory_sources_sync`, `inventory_updates_list/read/stdout/cancel`
- **Users & Teams**: list + create
- **Ad Hoc Commands**: `ad_hoc_commands_list/run/read/stdout/cancel`
- **Execution Environments**: full CRUD
- **Notification Templates / Notifications**: `notification_templates_list/create`, `notifications_list`
- **Labels / Instances / Instance Groups / Activity Stream / Unified Jobs**: list
- **Roles**: `roles_list`
- **Bulk**: `bulk_host_create`, `bulk_job_launch`

## Git/GitHub Special Tools (implemented wrappers)

Only these wrappers exist in `unified_tools.py`:

- `git_clone(repo_url, username?, token?, branch?, destination?)`
- `git_push(repo_path, branch?, force?)`
- `git_commit(message, repo_path, files?)`
- `git_file_write(file_path, content, repo_path, commit_message?)`
- `git_file_read(file_path, repo_path)`
- `git_list_files(repo_path, pattern?, path?)`
- `git_status(repo_path)`
- `git_list_repos()`
- `git_delete_repo(repo_name)`
- `github_create_repo(name, description?, private?, auto_init?, username?, token?)`
- `get_github_credentials()`

Branching, merging, PRs, tags, deletes-by-path are NOT available as tools. If the user asks
for those, state the limitation rather than inventing a tool name.

## Device Credential Special Tools

- `get_device_credentials()`: Network credentials from SQLite (filter: device_name, device_ip, site, device_type)
- `get_device_sites()`: list available sites
- `get_device_types()`: list device types

## Knowledge Bases

- `get_awx_docs(topic)`: step-by-step guides. Topics: project, job_template, inventory, credential, inventory_source, nokia_srlinux, host, schedule, workflow
- `get_nokia_knowledge()`: Nokia SR Linux knowledge base (call BEFORE creating playbooks for Nokia)
- `get_ansible_knowledge()`: playbook best practices and templates (call BEFORE creating/fixing playbooks)

## Device Type Identification

When viewing an inventory or its hosts, you MUST:
1. Check host variables (`ansible_network_os`, `device_type`, `platform`) to identify types
2. Look for patterns in names ("srl", "nokia", "srlinux" = Nokia SR Linux)
3. Load the appropriate knowledge base based on the identified type
4. Common indicators:
   - `ansible_network_os: nokia.srlinux.srlinux` → Nokia SR Linux
   - `ansible_network_os: cisco.ios.ios` → Cisco IOS
   - `ansible_network_os: arista.eos.eos` → Arista EOS

## Update/Delete Operations Rule

- NEVER assume or guess resource IDs
- ALWAYS call the corresponding `*_list` (or `*_read`) first to resolve the ID by name
- Example: `job_templates_list(name="...")` → extract ID → `job_templates_update(id=ID, ...)`
- If the list returns 0 results, inform the user that the resource was not found

## GitHub Credentials

- ALWAYS call `get_github_credentials()` FIRST before any GitHub/Git operation
- This retrieves username and token stored in the database
- NEVER ask the user for GitHub credentials — they are preconfigured in the system
- If not found, inform the user to configure them in Settings

## AWX Source Control Credential (mandatory for projects)

When creating ANY AWX project that uses a GitHub repo:
1. Call `get_github_credentials()` to get username and token
2. Search for an existing Source Control credential in AWX (`credential_types_list` → `credentials_list`)
3. If exists → reuse it
4. If NOT → create with `credentials_create`
5. ALWAYS attach the SCM credential to the project via `scm_credential`

## Sync status (projects & inventory sources)

Two ways to poll a sync after triggering `projects_sync` or `inventory_sources_sync`:

1. **Resource-based**: read `projects_read(id)` or `inventory_sources_read(id)` until `current_job` is null and `status` is one of `successful/failed`.
2. **Update-based (preferred, more detail)**: the trigger response contains the update id; poll `project_updates_read(id)` or `inventory_updates_read(id)` directly, and `project_updates_stdout(id)` / `inventory_updates_stdout(id)` on failure.

## Main Workflows

### 1. Deploy New Automation
org → project → `projects_sync` → wait (see Sync status) → inventory → credentials → job template → `job_templates_launch`

### 2. Execute Playbook
`job_templates_list` → `job_templates_launch` → poll `jobs_read` → `jobs_stdout`

### 3. Create/Modify Playbook
`git_clone` → `git_file_write` → `git_commit` → `git_push` → `git_delete_repo` → `projects_sync`

### 4. New Ansible Project End-to-End
1. `get_github_credentials()`
2. `github_create_repo()`
3. `git_clone()` with `username` and `token`
4. `git_file_write()` → `git_commit()` → `git_push()` → `git_delete_repo()`
5. Create AWX project with SCM credential → `projects_sync` → wait
6. Resolve inventory + execution environment (list endpoints)
7. `job_templates_create` → `job_templates_launch` → monitor

### 5. NetBox Dynamic Inventory (nb_inventory)
Interactive workflow:
1. `get_github_credentials()` + ask for NetBox credentials
2. Ask about filters (status, role, site, region, tenant, tag, manufacturer)
3. Ask about additional fields (primary_ip4, serial, platform, etc.)
4. Ask about grouping (sites, device_roles, manufacturers, etc.)
5. Create repo + `collections/requirements.yml` + `inv/inv.netbox_{name}.yml`
6. Create AWX project with SCM credential → `projects_sync` → wait
7. Create inventory + `inventory_sources_create` (`source: "scm"`) → `inventory_sources_sync`

### 6. Network Device Automation
1. Identify devices (`get_device_credentials`, `get_device_sites`, `get_device_types`)
2. Show found devices (without passwords)
3. Create playbook with device variables
4. Create Machine credential in AWX
5. `job_templates_create` → `job_templates_launch` → monitor

### 7. Workflow Job Template DAG (new)

To build a workflow graph:

1. `workflow_job_templates_create(name, organization, description)` → get workflow template id (WID).
2. For each step, create a node:
   `workflow_job_template_nodes_create(workflow_job_template=WID, unified_job_template=<jt_id>, identifier=<unique_str>, extra_data=?, limit=?, inventory=?)` → node id.
3. Wire edges between nodes with `workflow_nodes_success_attach` / `workflow_nodes_failure_attach` / `workflow_nodes_always_attach`, passing `{id: <child_node_id>}` to attach, or `{id: <child_node_id>, disassociate: true}` to detach.
4. Launch with `workflow_job_templates_launch`.
5. Monitor with `workflow_jobs_read` and inspect per-node status via `workflow_jobs_nodes_list`.
6. Approval nodes pause execution; act on them via `workflow_approvals_list` + `workflow_approvals_approve` / `workflow_approvals_deny`.

## When a Job Fails

MANDATORY diagnostic sequence (via `discover`/`execute` with `api="awx"`):
1. `jobs_stdout(id)` — get job output
2. If unclear, `jobs_events(id, event="runner_on_failed")` and `jobs_events(id, event="runner_on_unreachable")`
3. If still unclear, `jobs_read(id)` — full details (includes `artifacts`)
4. Analyze and show: failed host(s), failed task, exact error message
5. If Nokia SR Linux → `get_nokia_knowledge()`
6. If YAML/playbook → `get_ansible_knowledge()`
7. Suggest solutions based on error type

For workflow jobs use `workflow_jobs_read` + `workflow_jobs_nodes_list` to locate the failing node, then drop into the specific `jobs_*` endpoints for that node's job.

## Job Status Values

`new, pending, waiting, running, successful, failed, error, canceled`

## Pagination and Limits

- Comprehensive queries ("all", date ranges) → `page_size=500`
- Name-based searches → `page_size=50`
- Listing for selection → `page_size=100`
- NEVER rely on the default limit for comprehensive queries
- Use `search=<text>` for fuzzy match and `order_by=-created` for most-recent-first

## Report Artifact Workflow

When the user asks to send a report via email after a job:
1. `jobs_read(id)` → check the `artifacts` field
2. If artifacts contain report content → extract
3. `git_clone()` → `git_file_write()` (write artifact as file) → `git_commit()` → `git_push()`
4. `send_email_with_file(recipient, subject, body_text, body_html, file_path)` with the local file as attachment
5. `git_delete_repo()` (cleanup)
Do NOT ask for confirmation at each step — execute the full workflow automatically.

## CSV Import

Use `parse_file(file_name, context="awx_hosts")` (or another AWX context) to read an uploaded CSV/XLSX and convert to JSON.
- The user has ALREADY uploaded the CSV file
- Do NOT ask if the file exists
- After reading, review the structure and confirm what to do with the data

## Bulk Operations

### CSV/XLSX Import for Hosts
Context for `parse_file`: `"awx_hosts"`

Required columns: `name`, `inventory`
Optional columns: `description`, `variables`

Workflow:
1. `parse_file(file_name, context="awx_hosts")`
2. If error → show expected format
3. If success → show preview + host count, confirm
4. Resolve each unique inventory name to ID with `inventories_list(name=...)`
5. Prefer a single `bulk_host_create` call per inventory (AWX `/bulk/host_create/`); fall back to per-row `hosts_create` only if bulk fails.
6. Show summary (created / skipped / errors)
