---
name: everyday
description: >
  Daily work with Comind.work - search, browse, filter records, view details,
  check history, navigate across apps and workspaces. Trigger when the user asks
  to find records, view data, track changes, create or update records, or do
  routine tasks in Comindwork.
---

# Comind.work Everyday Use

You help users navigate and work with their Comind.work data: finding records, viewing details, filtering lists, checking history, and creating or updating records.

## Core Principles

1. **Discover before assuming.** You don't know what workspaces, apps, or fields exist until you look. Always orient first.
2. **Show slugs, not URLs.** Display record references as `WORKSPACE/APP123` in tables and lists. Only construct a full URL when the user asks for a clickable link (use `whoami` to get the system URL).
3. **Respect the user's timezone.** Call `whoami` to learn their timezone, then present all datetime values accordingly.
4. **Right tool for the job.** Use `list_records` for browsing/filtering, `search_records` for keyword queries, `get_record` only for single-record deep dives. Never loop `get_record` over many records.
5. **Any record is writable.** If the user can create or edit a record in the web UI, they can do it via MCP too - tasks, deals, users, participants, any record type. The only exception: **delete is not exposed via MCP** (use the web UI instead).

---

## Orientation

When you don't know the workspace or app structure yet:

```
whoami                        -> your identity, timezone, system URL, recent activity
list_workspaces               -> discover available workspaces
get_workspace(alias)          -> apps, members, record counts
get_app_schema(app, ws)       -> field names, types, lookup values, actions, saved lists
```

The user may refer to things by display names ("IT Support", "the CRM", "deals"). Match fuzzy against workspace aliases and app names.

---

## Finding Records

### Keyword Search
Use `search_records` when the user gives you a keyword or phrase:
```
search_records(query="network outage", workspace="IT")
search_records(query="\"exact phrase\"")              # quotes for exact match
search_records(query="laptop -replacement")            # negation with -
```
No wildcards supported. Don't use search for browsing - use `list_records` instead.

### Browsing & Filtering
Use `list_records` to browse with structured filters:
```
list_records(workspace="CRM", app="DEAL", filter='state="active"', sort="updated desc", limit=20)
list_records(workspace="IT", app="TASK", filter='keeper="{current-user}"', sort="priority")
list_records(workspace="IT", app="TASK", list="THISWEEK")   # use saved list aliases
```

**Filter syntax (RLX):**

| Pattern | Example |
|---------|---------|
| Exact match | `state="active"` |
| Numeric comparison | `total>1000`, `c_amount<=500` |
| Date range | `creation_date>="2026-01-01"` |
| Not equal | `state!="closed"` |
| In-list | `state in "open,in_progress,new":string[]` |
| Current user | `keeper="{current-user}"` |
| Link field | `parent="IT/TASK123"` |
| Link-list field | `tag_list="docs"` |
| People-list field | `assignees_list="John Smith"` (MCP resolves names; raw RLX needs GUIDs) |
| AND | `state="open" AND keeper="{current-user}"` |
| OR | `state="open" OR state="in_progress"` |
| Grouping | `(state="open" OR state="new") AND priority="high"` |

**Sorting:** `updated`, `created`, `priority`, or any field db name + `asc`/`desc`.

**Pagination:** Use `skip` for paging (e.g., `skip=10, limit=10` for page 2).

### Record Details
Use `get_record` for full details on a single record:
```
get_record(reference="IT/TICKET42")
```
Returns: all fields, description, history log (first 10 + last 40 entries), and allowed workflow actions.

---

## Understanding App Structure

Before filtering by custom fields, pull the schema first:
```
get_app_schema(workspace="IT", app="TASK")
```

This reveals:
- **Field db names** - use these in filters (e.g., `c_priority`, not "Priority")
- **Lookup values** - valid dropdown values (e.g., state: submitted/active/closed)
- **Field controls** - date vs datetime, person vs text, richtext vs plain
- **Saved lists** - pre-configured filters (aliases are app-specific; check the schema to see what's available)
- **Actions/transitions** - available workflow actions (approve, close, reopen, etc.)

---

## Checking History & Changes

### Individual Changes
```
list_history(workspace="IT", filter='version_timestamp>="2026-03-01"', limit=20)
list_history(workspace="IT", app="TASK", filter='version_account="{current-user}"')
list_history(workspace="IT", filter='transition="comment"')
```
Shows: who changed what, when, with field diffs and comments.

**Filter restrictions:** History filters only support metadata fields: `version_account`, `version_timestamp`, `transition`, `comment`. You cannot filter history by app-specific fields (state, title, etc.).

### Change Summaries
```
aggregate_history(workspace="IT", dimensions=["version_account", "version_timestamp__pivotbymonth"])
```
For "who did what this month" or "how active is this workspace" type questions.

---

## Viewing Files & Attachments

When records reference images or files:
```
get_asset(content_id="e50c90bc-...")          # image/file UUID from [image: cmwid:<uuid>] in descriptions
get_asset(content_id="IT/ATTACHMENT31")        # attachment slug from file fields
```
Images display inline. Text files show content. PDFs/DOCX get text extracted.

---

## Creating Records

Any record the user can create in the web UI can be created via MCP:
```
create_record(workspace="IT", app="TASK", fields={
  "Title": "New laptop request",
  "Priority": "High",
  "Keeper": "John Smith",
  "Description": "<p>Need a new laptop for the new hire.</p>"
})
```

**Rules:**
- **Always confirm with the user before creating.** Show the field table and wait for approval.
- Field names accept captions or db names ("Priority" or "priority")
- Lookup fields accept caption values ("High", not the db id)
- Person fields accept full names (MCP resolves names to IDs automatically; raw RLX requires GUIDs)
- Link fields accept slugs ("IT/TASK123")
- Richtext fields accept HTML, markdown, or plain text (HTML is preferred; markdown and plain text are auto-converted)
- Datetime fields must include timezone offset (e.g., `2026-03-20T09:30:00+02:00`)
- Use `get_app_schema` first to see available fields and valid lookup values
- After success, always show the full URL from the response

## Updating Records

Any record the user can edit in the web UI can be updated via MCP:
```
update_record(reference="IT/TASK42", action="edit", fields={"Priority": "Critical"})
update_record(reference="IT/TASK42", action="Approve", comment="Looks good, approved.")
```

**Rules:**
- **Always confirm with the user before updating.** Show old vs new values.
- Pick the action that best matches intent from `get_record`'s allowed actions (e.g., "Approve", "Mark done"). Fall back to "edit" only when no better action fits.
- Only include fields you want to change - omitted fields stay untouched.
- Use the `comment` parameter for notes/messages - not the description field.
- **Delete is not exposed via MCP.** Use the web UI to delete records.
- After success, always show the full URL from the response.

---

## Counting & Aggregating

```
aggregate_records(workspace="IT", app="TASK", dimensions=["state"])                          # by state
aggregate_records(workspace="IT", app="TASK", dimensions=["keeper_id"], filter='state!="closed"')  # open per person
aggregate_records(workspace="IT", app="TASK", dimensions=["creation_date__pivotbymonth"])     # monthly volume
aggregate_records(workspace="IT", app="TASK", dimensions=["creation_date__pivotbyweek"])      # weekly volume
```
Pass all dimensions in a single call - don't make separate single-dimension queries.

For numeric metrics (totals, averages), add `metric_field`:
```
aggregate_records(workspace="CRM", app="DEAL", dimensions=["state"], metric_field="c_total_amount")
```

---

## Common Patterns

**"What's assigned to me?"**
```
list_records(workspace="...", app="...", filter='keeper="{current-user}" AND state!="closed"', sort="priority")
```

**"What changed recently?"**
```
list_history(workspace="...", filter='version_timestamp>="2026-03-25"', limit=30)
```

**"Show me this month's records"**
```
list_records(workspace="...", app="...", filter='creation_date>="2026-03-01"', sort="created desc")
```

**"How many open items by type?"**
```
aggregate_records(workspace="...", app="...", dimensions=["title"], filter='state!="closed"')
```

**"Update the status of this record"**
```
get_record(reference="IT/TASK42")                           # check current state and allowed actions
update_record(reference="IT/TASK42", action="Mark done")    # use the appropriate action
```
