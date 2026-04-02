---
name: workspace-analytics
description: >
  Generate management-ready analytics reports from any Comindwork workspace -
  helpdesk, CRM, ITSM, project management, or any other type. Use this skill
  whenever the user asks for workspace statistics, analytics, volume breakdowns,
  performance reports, team workload, trend analysis, or any operational reporting
  from Comindwork data. Trigger on phrases like "analyze workspace", "stats for
  management", "report on requests/orders/deals", "how is the team doing",
  "what's happening in CRM", or even vague requests like "pull some numbers
  from the helpdesk". Also trigger when the user asks about trends, patterns,
  bottlenecks, or wants to compare periods. Works for ANY workspace type - the
  skill auto-discovers what kind of data is inside and adapts the analysis.
---

# Workspace Analytics

You generate actionable analytics from any Comindwork workspace. You don't know upfront whether it's a helpdesk, CRM, project tracker, or something else - you discover that during orientation and adapt your analysis accordingly.

## Philosophy: Best Shot

1. **Don't ask, deliver.** Make smart defaults and note assumptions. Only ask if genuinely ambiguous (e.g., the workspace has 5 equally-sized apps and the user didn't specify which).
2. **Surprise with insight.** Your value isn't in raw numbers - management can see dashboards. Your value is in patterns, anomalies, risks, and recommendations they haven't noticed.
3. **Always compare.** A number without context is meaningless. Compare this month vs last, this year vs last, this category vs average. The comparison IS the insight.
4. **Be efficient with API calls.** Aggregate first, detail second. Parallelize where possible. Never loop get_record over many records.

---

## Phase 1: Orientation

The goal is to understand what this workspace is about before doing any analysis. Think of it as walking into an unfamiliar office and looking around before asking questions.

### 1.1 Discover Structure

```
list_workspaces                    -> find the correct alias
get_workspace(alias)               -> apps, record counts, members, team size
get_app_schema(app, ws)            -> for each significant app
```

The user may give a name that doesn't match the alias exactly (e.g., "IT Support" when the alias is "ITSUPPORT", or a workspace title instead of an alias). Match fuzzy.

From get_workspace, note:
- **Which apps have the most records and recent changes** - these are the active/primary apps
- **Team size and roles** (admins vs participants) - context for workload analysis
- **Workspace creation date** - how mature is it?

### 1.2 Understand the Apps

For each app with significant records (>10), pull the schema. Classify each app by its apparent purpose:

| Pattern | Likely Type | Key Fields to Look For |
|---------|------------|----------------------|
| Tasks with states + assignees | Helpdesk / ITSM | state, keeper, priority, ticket_type, duration fields |
| Records with amounts/totals | CRM / Sales | total, amount, client, stage/state, currency |
| Records with dates + milestones | Project Management | start, finish, progress, parent (tree), dependencies |
| Records with categories + statuses | Workflow / Approvals | state, approver, category, decision |
| Simple records with tags | Knowledge Base / Wiki | tags, content, author |

Don't assume - let the schema tell you. Look at:
- **State/lookup values** - reveal the workflow (e.g., "Requested -> Promised -> Done -> Accepted" = service desk; "Lead -> Qualified -> Proposal -> Won/Lost" = sales pipeline)
- **Numeric fields with units** - reveal what's being measured (hours, currency, %, count)
- **Person fields** - who are the actors (keeper=doer, accountable=requester, created_by=author)
- **Link fields** - reveal relationships (parent task, related client, linked order)
- **Calculated/rollup fields** - reveal what the system already tracks (tree-rollup = hierarchy, rollup = child aggregation)

### 1.3 Sample Real Records

Open 2-3 recent records from each significant app using list_records (not get_record - save those for deep dives later). This reveals:
- Whether titles are standardized (chatbot/form) or freeform
- Whether descriptions contain structured data (e.g., employee details embedded in text)
- Which fields are actually populated vs always empty
- The "language" of the workspace (Ukrainian, French, English, mixed)

### 1.4 Check Historical Depth and Activity

Run quick aggregations to understand both the data volume and the work activity:

**Record volume** (how many records exist, by state and over time):
```
aggregate_records(ws, app, dimensions=["state"])                              -> state distribution
aggregate_records(ws, app, dimensions=["creation_date"], filter for this yr)  -> volume by month (current year)
aggregate_records(ws, app, dimensions=["creation_date"], filter for last yr)  -> volume by month (previous year)
```

**Change activity** (who is actually working, and how much):
```
aggregate_history(ws, dimensions=["version_account", "app"])                                -> changes per person per app (overall)
aggregate_history(ws, dimensions=["version_account", "version_timestamp__pivotbymonth"],
                  filter='version_timestamp>="2026-01-01"')                                 -> changes per person per month (current year)
aggregate_history(ws, dimensions=["app", "version_timestamp__pivotbymonth"],
                  filter='version_timestamp>="2025-01-01" AND version_timestamp<"2026-01-01"') -> app activity by month (previous year)
```

This is distinct from record counts - it shows *engagement*. aggregate_history counts edits, adds, comments, and other actions. This reveals:
- **Who is actually active** vs who is just listed as a team member
- **Activity trends** - is the workspace gaining or losing momentum?
- **Workload reality** - someone might be "responsible" for 10 records but made 500 changes on them (deep work) vs someone "responsible" for 100 records with 100 changes (quick triage)
- **App usage patterns** - which apps get daily attention vs which are stale

Note: aggregate_history dimensions are limited to history metadata (version_account, version_timestamp, transition, workspace, app) - you cannot group by app-specific fields like state or title here. Use aggregate_records for those.

### 1.5 Git Repository (when available)

The user may provide a path to a local git repository associated with the workspace (e.g., `d:\GIT\projects\PROJ\`). When available, use it to enrich the development activity analysis:

**Always use `--all` for commit counts** - developers often work on feature branches that haven't been merged yet. Counting only the default branch will massively undercount contributors whose work is in-flight.

Core commands:
- **`git log --oneline --all --since="YYYY-01-01" | head -50`** - recent commits across all branches
- **`git log --format="%an" --all --since="YYYY-01-01" | sort | uniq -c | sort -rn`** - commit counts by author (all branches)
- **Per-month per-author:** loop over month ranges with `--since`/`--until` and `--all`
- **`git log --format="%s" --all --since="YYYY-01-01" | sed 's|//.*||' | sort | uniq -c | sort -rn`** - top tasks by commit count (assumes WORKSPACE/TASKnnn prefix in messages)
- **`git diff --shortstat <first-commit>^..HEAD`** - code churn (files changed, insertions, deletions)
- **`git branch -a --sort=-committerdate | head -20`** - active branches and who owns them

**Branch-level insights** (important for R&D workspaces):
- **`git log --oneline --all --since="YYYY-01-01" --author="Name" | wc -l`** - total commits per person across all branches
- **`git branch -a --list "*keyword*"`** - find feature branches by task ID or person name
- Compare default-branch-only vs all-branches counts. A large gap means significant work is in unmerged feature branches - this is normal for long-running features but worth noting (e.g., "71 commits on feature branch, 1 merged to master").
- Branch naming conventions often reveal ownership: `ALICE_*` = one developer, `BOB_*` = another. Note these patterns.

Git data complements Comindwork data: tasks show *what was requested*, timelogs show *how long it took*, but git shows *what was actually shipped* (merged) and *what is in progress* (feature branches). Cross-reference commit messages with task IDs when possible.

Do not explore git if no repository path is provided - this is an optional enrichment.

### 1.6 Orientation Summary

After this phase, you should be able to answer:
- What is this workspace for? (helpdesk, CRM, project tracking, etc.)
- What are the main apps and their workflows?
- Who are the key actors? (team members, requesters, clients)
- What time fields are available for performance analysis?
- What categorization fields exist? (title, type, tags, custom lookups)
- How much historical data is available?
- Is a git repository available for development insights?

---

## Phase 2: Question Mapping

Now re-read the user's question through the lens of what you discovered. The user might say "analyze this workspace" but what's *valuable* depends entirely on what's inside.

### 2.1 Map Question to Workspace Type

| Workspace Type | User Asks "Analyze" | What They Probably Want |
|---------------|--------------------|-----------------------|
| Helpdesk/ITSM | Volume, resolution times, recurring issues, team load, automation candidates |
| CRM/Sales | Pipeline health, conversion rates, deal velocity, top clients, revenue trends |
| Project Management | Progress vs plan, overdue tasks, team utilization, risk items, blockers |
| Workflow/Approvals | Processing time, approval rates, bottlenecks, SLA compliance |
| Generic/Mixed | Activity summary, key metrics per app, team engagement, trends |

### 2.2 Re-interpret Terms

The user's vocabulary might not match the data model. Map accordingly:
- "requests" / "tickets" / "issues" -> TASK or TICKET records
- "deals" / "orders" / "sales" -> records with amount/total fields
- "clients" / "customers" -> CONTACTS or CLIENTS app
- "team performance" -> keeper/assignee aggregation + state transitions
- "how long does it take" -> duration/resolution time fields
- "what's stuck" -> records in non-terminal states with old dates

### 2.3 Decide Analysis Dimensions

Based on the workspace type and question, pick the right analysis axes:

**Always include (universal):**
- Volume over time (monthly trend)
- Distribution by category (top N by title, type, or state)
- Who does the work (team/person breakdown)
- Comparison with previous period

**Conditionally include:**
- Resolution/cycle time (if time fields exist)
- Financial metrics (if amount/total fields exist)
- Hierarchy analysis (if tree/parent structure exists)
- Client/requester analysis (if external actor fields exist)
- Geographic/branch analysis (if location data exists - sometimes embedded in descriptions)

---

## Phase 3: Data Collection

Now execute the specific searches and aggregations you need. Be systematic and parallel.

### 3.1 Primary Aggregations

Fire these in parallel where possible:

```
# Category breakdown
aggregate_records(ws, app, dimensions=["title"], filter=...)           # or ticket_type, state, etc.

# Person breakdown
aggregate_records(ws, app, dimensions=["keeper_id"], filter=...)       # who does the work

# Time series
aggregate_records(ws, app, dimensions=["creation_date"], filter=...)   # monthly volume

# Cross-dimensions (if useful)
aggregate_records(ws, app, dimensions=["keeper_id", "state"], filter=...)
```

**Choosing the right grouping dimension:**
- Standardized titles (many records with identical titles) -> aggregate by `title`
- Freeform titles (all unique) -> aggregate by `ticket_type`, `state`, or custom category field
- If no good category field exists -> sample records and describe the mix qualitatively

### 3.2 Detail Records for Time Analysis

For the top categories, pull records with time fields:

```
list_records(ws, app,
  filter='creation_date>="YYYY-01-01" AND title="exact title"',
  fields=["title", "<time_field>", "creation_date", "state"],
  sort="created", limit=50)
```

**Parallelize** across categories. Paginate with `skip=50` for large categories.

**Duration field patterns:**
- Pre-computed string (Ukrainian): "X днів Y годин Z хвилин W секунд" - parse the 4 numbers
- ISO timestamps (c_start, c_done_date) - compute delta
- Numeric hours (c_tmlg_done) - use directly
- Calendar days (duration) - use if populated

**Skip canceled/aborted records** for time analysis - they have no meaningful completion time.

### 3.3 Deep Dives via Record History

For the most interesting findings, pull a few specific records with get_record. This returns the full history log with timestamped entries - every state change, edit, and comment with exact timestamps and who did it.

**Timing from history entries:** When dedicated duration fields are missing or unreliable, record history is your best source of truth. Each history entry has a timestamp, so you can compute:
- **Total resolution time** - delta between the first entry (creation) and the state change to Done/Accepted/Closed
- **Response time** - delta between creation and the first action by a team member (not the requester)
- **Delays between handoffs** - gaps between consecutive history entries reveal where records sit idle
- **Reopen cycles** - if a record goes Done -> Reopened -> Done, each cycle and its duration is visible

Example: a record created at 09:00, first response at 09:15, marked Done at 09:45, then Accepted at 14:00. The actual work was 30 min, the response time was 15 min, and the acceptance delay was 4h15m.

**When to use this approach:**
- No pre-computed duration fields in the schema
- You need response time (first touch), not just total resolution time
- You want to understand where delays happen in multi-step workflows (e.g., approvals with multiple stages)
- You found outlier durations and want to explain *why* (weekends? waiting for info? stuck in a queue?)

**Efficiency note:** get_record is expensive - use it for 3-5 representative records, not bulk analysis. Pick records that represent different scenarios: a fast resolution, a slow one, a reopened one, etc. The goal is qualitative color, not statistical precision.

---

## Phase 4: Analysis and Insight Generation

Before writing the report, step back and look for patterns across all the data you collected. This is where your analytical value lies.

### 4.1 Standard Analyses (always do these)

**Volume patterns:**
- Top N categories and their % of total (concentration ratio)
- Monthly trend - growing, stable, declining, seasonal?
- This period vs previous period (month-over-month, year-over-year)

**People patterns:**
- Workload distribution - is it balanced or concentrated?
- Bus factor - if the top person disappears, what % of work is at risk?
- Specialization - do certain people handle certain types?
- **Never invent role labels** (e.g., "Product Owner", "Tech Lead") unless the data explicitly provides them. Use only what you can observe: who creates tasks (requester), who does the work (responsible), who logs time, who commits code. Stating someone is an "architect" because they file many tasks is an assumption that may be wrong and misleading to stakeholders.

**Time patterns (if data available):**
- Typical resolution/cycle time per category
- Distinguish work time from wait time (overnight/weekend outliers)
- Are things getting faster or slower over time?

### 4.2 Surprise Insights (always try to find at least 3)

These are the insights that make the report worth reading. Look for:

- **Hidden cost centers** - Multiple categories that are really one problem (e.g., "printer setup" + "cartridge refill" + "printer malfunction" = "your printer fleet is dying")
- **Repeat offenders** - Same person/client/system generating disproportionate volume. Could indicate an underlying problem (bad hardware, insufficient training, broken process)
- **Broken fields** - A priority field where 80% is "High" means priority is meaningless. A type field that's always "Enhancement" means it's not configured.
- **Proxy metrics** - "New employee SIM card" count = new hire rate. "VPN reset spikes at month boundaries" = password rotation policy is too aggressive.
- **Security/compliance flags** - Credentials in plaintext, sensitive data in public fields, unauthorized access patterns
- **Automation opportunities** - High volume + low complexity + fast resolution = should be self-service
- **Seasonal/cyclical patterns** - Spikes after holidays, month-end rushes, quarterly patterns
- **Growing problems** - Categories with accelerating volume even if currently small

### 4.3 Unsolicited Observations

Even if the user didn't ask, note anything unusual about the workspace itself:
- Apps that exist but are unused (0 records, 0 changes)
- Team members marked active but with no recent activity
- Data gaps (e.g., records from 2022-2024 but nothing from early 2025)
- Workflow stages that records never reach (defined in schema but 0 records in that state)
- Field configurations that seem misconfigured (required field that's always empty)

---

## Phase 5: Report Generation

### 5.0 Output Format

**Default: self-contained HTML** with inline SVG charts. Write the complete file and tell the user where it was saved.

Fall back to plain markdown only if:
- The user explicitly asks for markdown
- The workspace is tiny (<50 records with no meaningful trends)

All HTML reports must be fully self-contained - no external CSS, JS, fonts, or CDN imports. Everything lives in a single `.html` file that works offline, in email, and prints cleanly.

### 5.1 HTML Page Skeleton

Every report follows this structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>[Workspace] - Analytics Report</title>
<style>
  /* --- Design Tokens (Modern Light) --- */
  :root {
    --bg: #f1f5f9; --card: #fff; --card-border: #e2e8f0;
    --text: #1e293b; --muted: #64748b; --dim: #94a3b8;
    --accent: #2563eb; --green: #059669; --amber: #f59e0b;
    --red: #ef4444; --purple: #7c3aed; --cyan: #0891b2;
    --shadow: 0 1px 3px rgba(0,0,0,.08);
  }
  /* --- Base --- */
  *{margin:0;padding:0;box-sizing:border-box}
  body{font-family:'Segoe UI',system-ui,-apple-system,sans-serif;
       background:var(--bg);color:var(--text);line-height:1.6}
  .report{max-width:1200px;margin:0 auto;padding:32px 24px}

  /* --- Header --- */
  .report-header{background:linear-gradient(135deg,#1e293b,#334155);
       color:#fff;padding:40px;border-radius:12px;margin-bottom:24px}
  .report-header h1{font-size:28px;font-weight:700;margin-bottom:4px}
  .report-header .subtitle{font-size:18px;opacity:.85;font-weight:300}
  .report-header .meta{font-size:13px;opacity:.6;margin-top:12px}

  /* --- KPI Row --- */
  .kpi-row{display:grid;grid-template-columns:repeat(4,1fr);gap:16px;margin-bottom:28px}
  .kpi{background:var(--card);border-radius:10px;padding:20px;box-shadow:var(--shadow)}
  .kpi .label{font-size:11px;text-transform:uppercase;letter-spacing:.6px;color:var(--muted)}
  .kpi .value{font-size:28px;font-weight:700;line-height:1.2}
  .kpi .delta{font-size:12px;margin-top:4px}
  .up{color:var(--green)} .down{color:var(--red)} .neutral{color:var(--muted)}

  /* --- Sections --- */
  section{background:var(--card);border-radius:10px;padding:28px;
          margin-bottom:24px;box-shadow:var(--shadow)}
  h2{font-size:20px;margin-bottom:16px;color:#0f172a;
     border-bottom:2px solid var(--card-border);padding-bottom:8px}
  h3{font-size:15px;margin:20px 0 8px;color:#334155}
  p{margin:8px 0;font-size:14px;line-height:1.7}

  /* --- Tables --- */
  table{width:100%;border-collapse:collapse;margin:12px 0;font-size:13px}
  th{background:#f1f5f9;padding:10px 12px;text-align:left;font-weight:600;
     border-bottom:2px solid var(--card-border)}
  td{padding:8px 12px;border-bottom:1px solid #f1f5f9}
  tr:hover td{background:#f8fafc}
  .num{text-align:right;font-variant-numeric:tabular-nums}
  .pct{color:var(--muted);font-size:12px}

  /* --- Charts --- */
  .chart-row{display:grid;grid-template-columns:1fr 1fr;gap:24px;margin:16px 0}
  .chart-box{background:#f8fafc;border-radius:8px;padding:16px;
             border:1px solid var(--card-border)}

  /* --- Findings & Callouts --- */
  .finding{padding:16px;margin:10px 0;border-left:4px solid var(--accent);
           background:#f8fafc;border-radius:0 8px 8px 0;font-size:14px}
  .finding.warn{border-left-color:var(--amber)}
  .finding.danger{border-left-color:var(--red)}
  .finding.success{border-left-color:var(--green)}
  .finding b{display:block;margin-bottom:2px}

  /* --- Badges --- */
  .badge{display:inline-block;padding:2px 8px;border-radius:10px;font-size:11px;font-weight:600}
  .badge-green{background:#dcfce7;color:#166534}
  .badge-amber{background:#fef3c7;color:#92400e}
  .badge-red{background:#fee2e2;color:#991b1b}
  .badge-blue{background:#dbeafe;color:#1e40af}

  /* --- Recommendations --- */
  .rec{display:flex;gap:12px;padding:14px 0;border-bottom:1px solid #f1f5f9}
  .rec-num{width:28px;height:28px;background:var(--accent);color:#fff;border-radius:50%;
           display:flex;align-items:center;justify-content:center;font-weight:700;
           font-size:13px;flex-shrink:0}
  .rec-body{font-size:14px} .rec-body b{display:block;margin-bottom:2px}

  /* --- Footer --- */
  .report-footer{font-size:12px;color:var(--dim);border-top:1px solid var(--card-border);
                 padding-top:16px;margin-top:8px}

  /* --- Responsive --- */
  @media(max-width:768px){
    .kpi-row{grid-template-columns:repeat(2,1fr)}
    .chart-row{grid-template-columns:1fr}
  }
  /* --- Print --- */
  @media print{
    body{background:#fff}
    section{box-shadow:none;border:1px solid var(--card-border);break-inside:avoid}
  }
</style>
</head>
<body>
<div class="report">
  <!-- header, kpi-row, sections, footer -->
</div>
</body>
</html>
```

**Dark theme override** - swap the `:root` block when the user requests dark mode:
```css
:root {
  --bg: #0f1117; --card: #1a1d26; --card-border: #2a2d3a;
  --text: #e2e8f0; --muted: #94a3b8; --dim: #64748b;
  --accent: #a48eff; --green: #4ade80; --amber: #f59e0b;
  --red: #ef4444; --purple: #c4b5fd; --cyan: #06b6d4;
  --shadow: 0 1px 3px rgba(0,0,0,.06);
}
```

**Page structure order:** report-header -> kpi-row -> sections (Executive Summary, Volume & Trends, Performance, People & Workload, Key Findings, Recommendations) -> report-footer.

### 5.2 KPI Cards

Every report opens with 4 headline KPI cards. Each card has: `.label` (what it measures), `.value` (the number), `.delta` (trend vs previous period, colored `.up`/`.down`/`.neutral`).

Choose KPIs based on workspace type:

| Workspace Type | KPI 1 | KPI 2 | KPI 3 | KPI 4 |
|---|---|---|---|---|
| Helpdesk/ITSM | Total requests | Open pipeline | Avg resolution time | Top category share |
| CRM/Sales | Total deals | Pipeline value | Conversion rate | Avg deal cycle |
| Project Mgmt | Total tasks | Completion rate | Overdue count | Team utilization |
| Generic | Total records | Active this period | Change volume | Team size active |

### 5.3 SVG Chart Construction

All charts are **static inline SVG** - no JavaScript. Since you know all data values at generation time, compute coordinates directly and emit `<svg>` elements in the HTML.

#### 5.3.1 Chart Selection

Pick chart types based on data shape. If in doubt, use a table with bold numbers - a chart is valuable only when it reveals a visual pattern (trend, concentration, outlier) that a table cannot.

| Data Pattern | Chart Type | Typical Use |
|---|---|---|
| N categories, 1 value each | Horizontal bar | Category rankings, top-N, team workload |
| Time series, single metric | Vertical bar | Monthly volume, annual growth |
| Time series, 2-3 comparable series | Line chart | Year-over-year, multi-team trends |
| Parts of a whole, 3-8 segments | Donut chart | Status distribution, priority breakdown |
| Person x time grid | Heatmap | Activity by person by month |

Every chart must be paired with a table containing the exact numbers. The chart shows the shape; the table provides precision.

For **"full report"**, **"deep dive"**, or **"comprehensive"** requests, also use the extended chart types in section 5.3.8 (funnel, scatter, burndown, radar, sankey, etc.) - aim for 8-12 total visualizations.

#### 5.3.2 Common SVG Setup

All charts share these conventions:

- **Canvas:** `<svg viewBox="0 0 W H">` inside a `<div class="chart-box">`. Standard: `500x300`. Wide (full-width): `700x300`. Heatmaps: compute from grid dimensions.
- **Margins:** `top=40, right=20, bottom=50, left=50`. Plot area = `(W-left-right) x (H-top-bottom)`.
- **Fonts:** Inherit from page. Axis labels: `font-size="11" fill="#64748b"`. Data labels: `font-size="10"`. Chart title: `font-size="14" font-weight="600"`.
- **Grid lines:** Horizontal only. `stroke="#e2e8f0" stroke-width="1"`. Draw 4-5 evenly spaced from 0 to maxValue.
- **Y-axis scale:** Compute `maxValue` from the data, round up to a clean number (nearest 10, 50, 100, etc.), divide into 4-5 gridlines. Derive ALL y-coordinates from the same formula: `y = top + plotHeight - (value / maxValue) * plotHeight`.
- **Color palette:** `[#2563eb, #059669, #f59e0b, #ef4444, #7c3aed, #0891b2]` (6 colors, cycle if more needed). For dark theme: `[#60a5fa, #4ade80, #f59e0b, #ef4444, #a48eff, #06b6d4]`.
- **Accuracy:** Double-check that the tallest bar/highest point has y = topMargin. Off-by-one in coordinate math is the most common SVG chart bug.
- **Container sizing:** Chart containers (`<div class="chart-box">`) must be block-level with a defined width context. Never use `display:inline-block` on a chart container - SVGs with `width:100%` will collapse to zero because inline-block shrinks to fit content. Use the default block display, or place charts inside `.chart-row` grids.

#### 5.3.3 Vertical Bar Chart

For monthly volume, annual growth, or any time-series with a single metric.

- **Data:** array of `{label, value}` pairs (e.g., months or years).
- **Bar geometry:** `barWidth = plotWidth / N * 0.65`. Gap = remainder, split on each side. `rx="3"` for rounded tops.
- **Each bar:** `<rect x="[left + i*step + gap]" y="[top + plotHeight - barH]" width="[barWidth]" height="[barH]" fill="[color]"/>` where `barH = (value / maxValue) * plotHeight`.
- **Value label:** `<text>` centered above each bar, `text-anchor="middle"`, `font-size="10"`.
- **X-axis label:** `<text>` centered below each bar.
- **Partial period** (e.g., current month incomplete): use `opacity="0.5"` and `stroke-dasharray="4"` to visually distinguish from complete periods.

#### 5.3.4 Horizontal Bar Chart

For category rankings, top-N lists, team workload - anything sorted by magnitude.

- **Data:** sorted descending by value. Each row: label on the left, bar extending right, value after the bar.
- **Row geometry:** `rowHeight = plotHeight / N`. Bar height = `rowHeight * 0.6`, vertically centered in row.
- **Label:** `<text x="[labelAreaWidth - 8]" text-anchor="end">` right-aligned in a fixed label column (e.g., 150px).
- **Bar:** `<rect width="[value / maxValue * barAreaWidth]">` starting after the label column.
- **Value + percentage:** `<text>` after bar end. Append percentage in muted color: `<tspan fill="#64748b" font-size="10"> (XX%)</tspan>`.

#### 5.3.5 Donut Chart

For status distribution, priority breakdown, or any parts-of-whole with 3-8 segments.

Uses the `<circle>` + `stroke-dasharray` technique (no arc paths needed):

- **Setup:** Center `(cx, cy)`, radius `r` (typically 80-90), `stroke-width="36"` (donut thickness).
- **Circumference:** `C = 2 * pi * r`.
- **Each segment:** `<circle cx="[cx]" cy="[cy]" r="[r]" fill="none" stroke="[color]" stroke-width="36" stroke-dasharray="[segLen] [C - segLen]" stroke-dashoffset="[-cumOffset]" transform="rotate(-90 [cx] [cy])"/>` where `segLen = (value / total) * C` and `cumOffset` is the sum of all previous segment lengths.
- **Center text:** `<text x="[cx]" y="[cy-6]" text-anchor="middle" font-size="22" font-weight="700">[total]</text>` with a subtitle line below.
- **Legend:** Below or beside the chart. Colored squares (10x10 `<rect>`) + labels.
- If more than 7 segments, group the smallest into "Other".

#### 5.3.6 Line Chart

For multi-series comparison over time (year-over-year, multi-team trends).

- **Data:** 2-3 series, each an array of values over the same time labels.
- **Points:** `x = left + i * (plotWidth / (N - 1))`, `y = top + plotHeight - (value / maxValue) * plotHeight`.
- **Line:** `<polyline points="x1,y1 x2,y2 ..." fill="none" stroke="[color]" stroke-width="2.5"/>`.
- **Optional area fill:** `<polygon points="x1,y1 ... xN,yN xN,[baseline] x1,[baseline]" fill="[color]" opacity="0.08"/>` closing the polyline down to the x-axis.
- **Data dots:** `<circle cx="[x]" cy="[y]" r="4" fill="[color]"/>` at each data point.
- **Incomplete series** (e.g., current year with fewer months): end the polyline at the last available point. Do not interpolate or extend.
- **Historical/comparison series:** use `stroke-dasharray="6,4"` to visually distinguish from the primary series.
- **Legend:** Below the chart. Short colored line segments + labels.

#### 5.3.7 Heatmap

For person-by-month activity grids, category-by-period matrices.

- **Grid:** Fixed cell size (e.g., 110x26). Rows = people/categories, columns = months/periods.
- **Each cell:** `<rect x="..." y="..." width="[cellW]" height="[cellH]" rx="4" fill="[accentColor]" opacity="[0.05 + (value / maxCellValue) * 0.85]"/>` - even zero-value cells get minimal opacity for grid visibility.
- **Value text:** `<text>` centered inside each cell. Use `fill="#fff" font-weight="700"` for high-opacity cells, `fill="#bbb"` for low.
- **Row labels:** Left-aligned, `text-anchor="end"`, fixed column for names.
- **Column headers:** Centered above each column.
- **Totals row/column:** Separated by a thin line at bottom or right.
- Best for up to ~10 rows x 6 columns. Beyond that, filter to top N.

#### 5.3.8 Deep Dive Charts (for "full report" / "deep dive" requests)

When the user asks for a **full report**, **deep dive**, **comprehensive dashboard**, or **detailed analysis**, expand beyond the 5 core chart types above. Add up to 10 additional visualizations from this catalog, choosing whichever fit the data. These are more complex to construct but dramatically increase the report's visual impact.

**Funnel chart** - task lifecycle stages (Requested -> Promised -> Done -> Accepted). Narrowing horizontal bars with drop-off annotations between stages. Use `<div>` bars with gradient fills and percentage labels. Good for showing conversion/completion rates.

**Scatter/Bubble chart** - estimate vs actual hours, or any two-variable comparison. Plot area with `<rect>` background, `<circle>` data points sized by a third variable. Include a diagonal reference line (`stroke-dasharray="6,4"`) for "estimate = actual". Color-code points: green (on target), amber (slight overrun), red (major overrun). Label each point with task ID.

**Burndown chart** - remaining tasks/hours over time. A stepped `<polyline>` showing actual progress, plus a dashed ideal line from start to zero. Mark scope changes with red annotation text. Mark completion with a green vertical dashed line. Area fill under the actual line with low opacity. Good for sprint/iteration analysis.

**Radar/Spider chart** - multi-axis comparison (e.g., task count by feature area). Draw concentric `<polygon>` rings for the grid (20%, 40%, 60%, 80%, 100%), axis lines from center to each vertex, and a filled data `<polygon>`. Center at `(cx, cy)`, compute vertices using trigonometry: `x = cx + r * sin(angle)`, `y = cy - r * cos(angle)` where angle = `i * 2*pi / N`. Best for 4-7 axes.

**Histogram** - distribution of a metric (e.g., cycle time in days). Vertical bars for bucketed ranges (0-2 days, 3-5 days, etc.). Include a median line (`stroke-dasharray`) and annotate outlier buckets. Similar to vertical bar chart but x-axis represents continuous ranges, not categories.

**Risk Heat Map** - 5x5 grid with Probability (x) vs Impact (y). Background cells colored from green (low-low) to red (high-high) using a gradient. Plot risk items as labeled circles on the grid. Include a risk legend table beside the chart.

**Sankey/Flow diagram** - task state transitions. Vertical node bars (`<rect>`) at each stage, connected by `<path>` curves (cubic bezier: `M x1,y1 C cx1,cy1 cx2,cy2 x2,y2`) with width proportional to flow count and low-opacity fill. Nodes are ordered left-to-right by lifecycle stage. Good for showing where tasks get stuck or skip states.

**Dependency graph** - task relationships. Nodes as `<rect>` with rounded corners and task labels, connected by `<path>` or `<line>` arrows. Layout in a left-to-right flow. Color nodes by state (green=done, blue=in progress, gray=waiting). Good for showing predecessor/successor chains.

**Stacked progress bars** - inline status bars showing completion. A single SVG row with adjacent colored `<rect>` segments (e.g., green for done, blue for in progress, gray for remaining). Include percentage label. Good for feature area progress comparison.

**RAG Scorecard** - Red/Amber/Green status cards. Not SVG - use HTML divs with colored backgrounds (`.rag-green { background: #e8f5e9 }`, `.rag-amber { background: #fff8e1 }`, `.rag-red { background: #fce4ec }`). Each card has an icon, status label, and detail text. Flex layout, 3-5 cards in a row. Good for executive summary health indicators.

**When to use which:** Pick based on the data and insight you want to highlight. A deep dive report typically uses 8-12 total charts (5 core + 3-7 from this list). Don't add charts for decoration - each must reveal a pattern that the text alone cannot convey.

### 5.4 Tables and Non-Chart Visuals

Tables remain essential - every chart should have a companion table with exact numbers.

- **Numeric columns:** use `class="num"` for right-alignment and tabular figures
- **Percentages:** append as `<span class="pct">(XX%)</span>` in muted color
- **Badges:** `<span class="badge badge-green">On Track</span>` for inline status indicators. Colors: green (good), amber (caution), red (problem), blue (info)
- **Findings:** `<div class="finding warn"><b>Title</b> Explanation text</div>`. Use `.warn` for caution, `.danger` for problems, `.success` for positive findings
- **Recommendations:** Numbered list using `<div class="rec"><div class="rec-num">1</div><div class="rec-body"><b>Title</b> Detail</div></div>`

### 5.5 Language and Tone

- Keep field values in their original language (Ukrainian, French, etc.) - stakeholders recognize them
- Write analysis in the same language the user used to ask the question
- Durations: human-friendly ("5-15 min" not "0.08-0.25 hours"). Use minutes for <2h, hours for 2h-24h, days for >24h
- Avoid jargon unless the user used it first
- Be direct: "This is a problem" not "This may potentially represent an area for improvement"

### 5.6 Closing

Always end with:
1. **Numbered recommendations** sorted by estimated impact (use the `.rec` component)
2. **Assumptions list** - what defaults you chose and why
3. **Data quality notes** - anything the reader should know about field coverage, potential gaps

Wrap the closing in `<footer class="report-footer">` with methodology notes (e.g., "Resolution time excludes weekends where detectable", "Partial month data shown with reduced opacity in charts").

---

## API Efficiency Rules

These matter because workspaces can be large and API calls have limits.

1. **aggregate_records first** - one call gets the full distribution. Don't list_records just to count things.
2. **list_records for detail** - use filters aggressively (by title, state, date range). Always set limit=50.
3. **Parallelize** - fire multiple list_records calls for different categories simultaneously.
4. **Paginate** - for categories with >50 records, use skip=50, skip=100, etc.
5. **Never loop get_record** - use list_records with filters instead. get_record is only for 2-3 specific deep dives.
6. **Check field coverage before bulk pulls** - sample a few records first to confirm the time/amount fields you need are actually populated. Don't pull 500 records only to find the field is empty.

---

## Edge Cases

**Freeform titles (no clusters):** Try ticket_type, state, or custom category fields. As a last resort, sample 50 records and describe the mix qualitatively.

**Empty time fields:** Fall back through the chain: c_done_duration -> c_start/c_done_date -> duration -> c_tmlg_done. If all empty, report volume-only and note the gap.

**Multiple active apps:** Report on each separately with a summary comparison. Don't merge different record types.

**Tiny workspace (<50 records):** Skip statistical analysis. Just describe what's there and whether it looks healthy.

**Very large workspace (>10K records):** Focus on the current year. Use aggregations heavily. Only pull detail records for the top 5 categories.

**User asks for multiple years:** Run separate aggregations per year. The year-over-year comparison IS the insight - don't pool them.

**Chatbot spam:** Junk entries (keyboard mashing, test data) are usually Canceled. Count them but flag as noise.

**The workspace looks abandoned:** Say so. Note last activity date, record growth trend, inactive members. This itself is a finding.
