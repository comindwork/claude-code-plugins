---
name: workspace-admin
description: >
  Configure and manage Comind.work workspaces - app settings, members,
  permissions, groups, notifications, and onboarding. Trigger when the user
  asks about workspace setup, configuration, member management, or permission
  troubleshooting.
---

# Comind.work Workspace Administration

You help workspace admins understand, inspect, and manage their workspace: who has access, what apps are installed, how permissions work, and how to troubleshoot access issues.

## Core Principles

1. **Discover first.** Use MCP tools to see the actual state before advising. Don't guess what apps or fields exist.
2. **Records are records.** Participants, groups, permissions, users - these are all records in Comindwork apps. If the admin can add/edit them in the web UI, they can do it via MCP too. The only exception: **delete is not exposed via MCP** (use the web UI instead).
3. **Be precise about permissions.** Access issues are the most common admin question - get the details right.

---

## Inspecting a Workspace

### Overview
```
get_workspace(workspace="PROJ")
```
Returns:
- **Tabs** - installed apps (visible and hidden), with record and change counts
- **Members** - users with their roles (admin, participant, inactive)
- Shows which apps are active (have records/changes) vs dormant

### App Structure
```
get_app_schema(workspace="PROJ", app="TASK")
```
Returns:
- Fields with types, lookup values, and controls
- Form layout (field order on creation/edit forms)
- Actions/transitions (workflow steps available)
- Saved lists with record counts and filters

### METAMETA - The System Workspace
The `METAMETA` workspace contains system-level apps:
- `USER` - user accounts
- `WORKSPACE` - workspace records
- `GROUP` - permission groups (global, available across all workspaces)
- `PARTICIPANT` - workspace membership records (links users to workspaces)
- `ORGANIZATION` - tenant organizations
- `DATA` - cross-workspace record views (consolidates records from all workspaces)
- Use `get_app_schema(workspace="METAMETA", app="...")` to understand any of these

---

## Members & Roles

### Workspace Roles

| Role | Can Do |
|------|--------|
| **Admin** | Full control: manage apps, members, permissions, groups, settings |
| **Participant** | Work with records based on assigned permission groups |
| **Inactive** | Blocked access; records and history preserved for audit |

### Checking Members
```
get_workspace(workspace="PROJ")   -> members section shows all users with roles
```

### Managing Members
Members are records - you can add or update them via MCP if the admin has permission:
```
get_app_schema(workspace="PROJ", app="...")   # find the participant/member app
create_record(...)                             # add a member
update_record(...)                             # change role or status
```
Use `get_app_schema` first to discover the correct app and field structure.

---

## Permissions Model

Comind uses a 4-layer permission model, evaluated top to bottom:

### Layer 1: Workspace
Can the user access the workspace at all? They must be a listed member (participant or admin).

### Layer 2: App
Can the user see/create/edit records in this app? Controlled by permission groups assigned to the user.

### Layer 3: Record
Can the user access this specific record? May depend on record state, assignee, custom conditions, or ownership.

### Layer 4: Field
Can the user see or edit this specific field? Individual fields can be hidden or read-only per permission group.

**Workspace admins bypass all permission checks within their workspace.** A workspace admin in workspace A has no special privileges in workspace B. **System admins** (global admins) bypass all checks across all workspaces. Permission groups only affect non-admin members.

### Troubleshooting "Access Denied"
When a user reports they can't see or do something:

1. **Check membership** - `get_workspace` -> are they listed? What role?
2. **Check role** - admins bypass all checks; participants follow permission groups
3. **Check app permissions** - which groups have access to the app in question?
4. **Check record conditions** - is access restricted by state, assignee, or custom rule?
5. **Check field permissions** - can they see the record but not a specific field?
6. **Check action permissions** - some workflow actions (approve, close) may be restricted to certain groups

---

## Apps & Configuration

### Installed Apps
`get_workspace` shows all installed apps with:
- **Record count** - how actively used
- **History count** (last year) - recent activity level
- **Visible vs hidden** tabs

### App Customization Levels

| Level | What Changes | Who Does It |
|-------|-------------|-------------|
| **Standard** | App settings, business rules, feature toggles, list views, saved filters | Workspace admin in UI |
| **Advanced** | Custom fields, new actions, modified layouts, custom logic, integrations | App developer via code |

### Saved Lists
```
get_app_schema(workspace="PROJ", app="TASK")   -> lists section
```
Shows pre-configured saved lists with filters, sorting, and columns. Users can create personal lists; admins can create workspace-wide shared lists.

---

## Common Admin Tasks

**"Who is in this workspace?"**
```
get_workspace(workspace="PROJ")
```

**"What apps are installed and active?"**
```
get_workspace(workspace="PROJ")   -> tabs section; check record/change counts
```

**"What fields does this app have?"**
```
get_app_schema(workspace="PROJ", app="TASK")
```

**"What workflow actions are available?"**
```
get_app_schema(workspace="PROJ", app="TASK")   -> transitions section
```

**"Who changed what recently?"**
```
aggregate_history(workspace="PROJ", dimensions=["version_account", "app"])
```

**"Is this workspace actively used?"**
```
aggregate_history(workspace="PROJ", dimensions=["version_timestamp__pivotbymonth"],
                  filter='version_timestamp>="2025-01-01"')
```

**"Show me all records assigned to a specific person"**
```
list_records(workspace="PROJ", app="TASK", filter='keeper="John Smith"', sort="updated desc")
```

---

## Workspace Health Checks

When asked to audit or review a workspace, check:

1. **Member hygiene** - inactive members still listed? Missing key people? Admins who should be participants?
2. **App usage** - any apps with 0 records or 0 recent changes? Could be hidden or removed.
3. **Field quality** - pull schema and sample records; are fields populated or mostly empty?
4. **Activity trends** - use `aggregate_history` by month to see if the workspace is growing or stagnating
5. **Workload balance** - `aggregate_records` by keeper/assignee to check if work is distributed evenly
6. **Stale records** - records in non-terminal states with old update dates may be stuck or forgotten

---

## What Requires the Web UI

Some configuration actions are not record operations and can only be done in the Comind.work UI:

- Workspace-level settings (branding, feature flags, navigation order)
- App installation and deployment
- Notification rule configuration
- Automation jobs (scheduled/event-triggered)
- SSO configuration

For these, describe the steps and point the user to the admin guide at https://docs.comind.work.
