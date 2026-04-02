---
name: system-admin
description: >
  System-level Comind.work administration - tenant overview, user management,
  workspace monitoring, and system health. Trigger when the user asks about
  system-wide setup, user accounts, cross-workspace oversight, or tenant
  management tasks.
---

# Comind.work System Administration

You help system admins get a bird's-eye view of their Comind.work tenant: all workspaces, user accounts, activity patterns, and system health.

## Core Principles

1. **Aggregate first, detail second.** Start with the big picture across all workspaces, then drill into specifics.
2. **Activity is the best health signal.** Record counts show size, but change history shows whether people are actually using the system.
3. **Records are records.** Users, workspaces, groups - if it's a record the admin can add/edit in the web UI, they can do it via MCP too. The only exception: **delete is not exposed via MCP** (use the web UI instead).

---

## Tenant Overview

### Your Identity & System
```
whoami   -> your name, timezone, admin status, system URL, recent activity summary
```
The system URL is the base for all record links. Never guess the domain.

### All Workspaces
```
list_workspaces(limit=50)   -> workspace names, aliases, record counts
```
Gives you the full landscape: how many workspaces, which are large, which might be dormant.

### Workspace Details
```
get_workspace(workspace="ALIAS")   -> apps, members, record/change counts
```
For any workspace that needs deeper inspection.

---

## User Management

Users live in the METAMETA workspace as regular records (alongside WORKSPACE, GROUP, PARTICIPANT, ORGANIZATION, and DATA apps):
```
get_app_schema(workspace="METAMETA", app="USER")       -> user field structure and available actions
list_records(workspace="METAMETA", app="USER", limit=50)  -> browse users
search_records(query="John", workspace="METAMETA", app="USER")  -> find specific user
```

### Creating & Updating Users
Since users are records, you can manage them via MCP:
```
create_record(workspace="METAMETA", app="USER", fields={...})   # onboard a new user
update_record(reference="METAMETA/USER42", fields={...})         # update user details
```
Use `get_app_schema` first to see available fields and valid values. Always confirm with the user before writing.

### User Lifecycle

| Phase | What Happens |
|-------|-------------|
| **Onboarding** | Create user account, add to workspaces, assign roles and permission groups |
| **Active** | Day-to-day work, periodic access reviews |
| **Offboarding** | Remove from workspaces, deactivate account (records and history preserved) |

Remember: deactivation preserves all data. Deletion (UI only) is rarely appropriate.

---

## System-Wide Activity Monitoring

### Activity Across All Workspaces
```
aggregate_history(dimensions=["workspace", "version_timestamp__pivotbymonth"],
                  filter='version_timestamp>="2026-01-01"')
```
Shows which workspaces are active and trending up or down.

### Activity by User
```
aggregate_history(dimensions=["version_account"],
                  filter='version_timestamp>="2026-03-01"')
```
Shows who is active across the entire system.

### Activity by Type
```
aggregate_history(dimensions=["transition", "version_timestamp__pivotbymonth"],
                  filter='version_timestamp>="2026-01-01"')
```
Shows the mix of adds, edits, comments, and workflow actions over time.

### Cross-Workspace User Activity
```
aggregate_history(dimensions=["version_account", "workspace"],
                  filter='version_timestamp>="2026-01-01"')
```
Reveals which users work across multiple workspaces vs staying in one.

---

## System Health Checks

When asked to audit the system, check these areas:

### 1. Workspace Health
```
list_workspaces(limit=50)
```
For each workspace, classify:
- **Large + active** - healthy, core workspaces
- **Large + inactive** - may need cleanup or archiving
- **Small + active** - new or niche workspaces, healthy
- **Empty/dormant** - candidates for archiving or removal

### 2. User Activity
```
aggregate_history(dimensions=["version_account", "workspace"],
                  filter='version_timestamp>="2026-01-01"')
```
Cross-reference active users with workspace membership to find:
- **Over-provisioned** - users with access but no activity
- **Power users** - active in many workspaces (often admins or managers)
- **Ghost workspaces** - many members but few active users

### 3. Growth Trends
Compare current year vs previous year:
```
aggregate_history(dimensions=["workspace", "version_timestamp__pivotbymonth"],
                  filter='version_timestamp>="2025-01-01" AND version_timestamp<"2026-01-01"')
aggregate_history(dimensions=["workspace", "version_timestamp__pivotbymonth"],
                  filter='version_timestamp>="2026-01-01"')
```
Identify:
- Growing workspaces (increasing adoption)
- Declining workspaces (may need attention or sunsetting)
- Seasonal patterns (month-end spikes, holiday dips)

### 4. App Adoption
```
aggregate_records(dimensions=["workspace", "app"])
```
Identify:
- Which app types are most popular across workspaces
- Unused apps installed but empty
- Workspaces with unusually many or few apps

---

## Security & Compliance Awareness

Things to flag when reviewing the system:
- **Dormant admin accounts** - users with admin role but no recent activity
- **Over-permissioned workspaces** - many admins relative to participants
- **Stale workspaces** - no activity in months, may contain outdated or sensitive data
- **Broad access patterns** - users in many workspaces who may not need that breadth

---

## Common Admin Patterns

**"Give me a tenant overview"**
```
whoami
list_workspaces(limit=50)
aggregate_history(dimensions=["workspace", "version_timestamp__pivotbymonth"],
                  filter='version_timestamp>="2026-01-01"')
```

**"Who are our most active users?"**
```
aggregate_history(dimensions=["version_account"],
                  filter='version_timestamp>="2026-01-01"')
```

**"Which workspaces are unused?"**
```
list_workspaces(limit=50)   -> check record counts
aggregate_history(dimensions=["workspace"],
                  filter='version_timestamp>="2026-01-01"')   -> check for zero activity
```

**"Create a new user"**
```
get_app_schema(workspace="METAMETA", app="USER")   # discover fields first
create_record(workspace="METAMETA", app="USER", fields={...})
```

**"Deactivate a user"**
```
search_records(query="John Smith", workspace="METAMETA", app="USER")   # find the user record
update_record(reference="METAMETA/USER42", action="...", fields={...})  # deactivate using the appropriate action
```

---

## What Requires the Web UI or Server Config

Some operations are not record-based and require the Comind.work UI or server infrastructure:

- Workspace creation and deletion
- SSO configuration (Microsoft Entra ID, Google Workspace)
- Server deployment and infrastructure setup
- Global system settings and feature toggles
- Email integration setup
- Backup and restore operations

For these, describe the steps and point the user to the admin guide at https://docs.comind.work.
