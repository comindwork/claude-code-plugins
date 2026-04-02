---
name: apps-dev
description: Skill for developing and customizing Comind.work (Comindwork or Comind) low-code apps. Trigger when the user asks to create, modify, or debug a Comind.work app - fields, actions, layouts, lists, view logic, transition logic, settings, plugins, or deployment.
argument-hint: [path]
allowed-tools: Read, Grep, Glob
---

# Comind.work App Development

## About this skill

- This skill is ~30KB (~8K tokens). In long conversations with large app code, context fills up - start a fresh conversation when Claude's answers degrade.
- **Updates:** If loaded via marketplace plugin, updates arrive automatically. If loaded manually (copied SKILL.md), re-download to get new versions - the file does not auto-update.

## Deep reference - docs.comind.work

For full API reference, complete field-option list, and how-to deep-dives:
- LLM-optimized index: https://docs.comind.work/llms.txt
- Full text concatenated: https://docs.comind.work/llms-full.txt
- Human docs: https://docs.comind.work

This skill carries the patterns, gotchas, and conventions you need at-a-glance. For exhaustive reference (every field option, every CLI flag, every server-side global), fetch the docs.

## Sample apps

For learning by example - real Comind apps showing fields, actions, layouts, view logic, and transitions wired together:

- Install: `npm i @comind/workspace-all`
- Browse: `node_modules/@comind/workspace-all/`

Read or grep these to see how production apps solve common patterns - usually faster than reading prose docs alone. Requires the private registry from Prerequisites.

## Target path

If a path argument is provided (`$0`), use it as the working context:

1. **Check if it's a single app** - look for `package.json` with a `comindApp` field
2. **If not, check if it's a workspace folder** - scan for subdirectories that are apps (each having `package.json` with `comindApp`)
3. **If no argument** - ask the user what app or folder to work with, or use the current working directory

When working with a workspace folder, list the available apps before proceeding.

## Skill directory

This skill's files are located at: `${CLAUDE_SKILL_DIR}`

When paths below mention `<SKILL_DIR>`, substitute this resolved path.

## Development environments

Claude Code is available as **CLI (recommended)**, Desktop app, IDE extensions, and `claude.ai/code`. The CLI runs inside whichever host environment you're using; the table below is about the host, not the Claude client.

Apps can be developed in three host environments. Detect the current one to adapt workflow:

| Environment | How to detect | Preview mechanism | Setup |
|---|---|---|---|
| **Codespaces (devcontainer)** | `CODESPACES=true` env var | Open URL in browser (VS Code Simple Browser or port forward) | Auto-provisioned, copy `.config/dev.env` |
| **Local devcontainer** | Docker + VS Code, no system Node | Open URL in browser | VS Code + Docker, copy `.config/dev.env` |
| **Local native** | Node.js on PATH | Open URL in browser | Node.js >=22, npm >=10, git, copy `.config/dev.env` |

All environments share the same codebase, CLI commands, and deploy workflow.

## Key concepts

1. **JSX is NOT React.** Layout `.tsx` files compile to XML strings via a custom `entity` JSX factory. No hooks, state, lifecycle, context, or portals.
2. **Same code runs in two places.** TypeScript compiles to JS for the browser and C# backend (via Jint). Action logic has no access to `fetch`, `document`, `window`, or Node APIs.
3. **Inheritance model.** `@comind/base` -> plugins -> parent app -> child app. Arrays **replace** (not concatenate) during deep merge.
4. **Field naming.** Custom fields use `c_` prefix. In layouts, camelCase props become kebab-case XML attributes. Field name values keep their original form.
5. **Settings tiers.** System-settings = compile-time (`import`). Workspace-settings = runtime (`ComindView.getVars()` or `ComindServer.getVars()`). **Never use `import` for workspace-level settings** - it returns the compile-time default, not the workspace value.
6. **Lookup field values.** Use db values from the schema, not display captions.

### Git as source of truth

Git is the canonical store for app code. Edit locally (or via Claude), run `npx comind install` to deploy and test, then commit to git.

## Prerequisites checklist

Before starting any app work, verify:

1. **Private registry configured** - `.npmrc` in the project root points to the org's private registry. `@comind/api` is not on public npm, so without this `npm i` will 404
2. **Dependencies installed** - `npm i` has been run in the project root (check: `node_modules/@comind/api` exists)
3. **Config file has REAL values** - `.config/dev.env` exists AND `COMIND_SECRET_AUTH_CODE_MANAGEMENT` / `COMIND_BASE_URL` are filled with values from your admin, not the template placeholders (`<token-from-admin>`, `YOUR-ORG.comindwork.com`). Placeholder values produce silent 401s with no hint at the cause
4. **Connectivity** - `npx comind i --dry-run` (or any small install) succeeds without auth errors
5. **Worktree awareness** - if Claude Code runs in a git worktree (e.g., Claude Code Desktop isolation mode), the worktree has its own working directory. Config files must exist in the worktree path, not just in the main repo root

## Development workflow

### Quick start - your first change in 60 seconds

Already set up? Try this to verify everything works:

1. Invoke the skill on any app: tell Claude to work with `apps/crm/app-org-task` (or whichever app folder you have)
2. Ask: *"Change the caption of the title field to 'Task name' and deploy to CRM"*
3. Claude will edit `fields/index.ts`, run `npx comind install`, and confirm the deployment
4. Open the app in your browser to see the change

That's the full cycle: read app code - modify - deploy - verify. Everything below explains each step in detail.

1. **Explore existing app** - Read `fields/`, `actions/`, `views/`, `settings/`
2. **Check parent/plugins** - If `inheritFrom` is set, check the parent app's code
3. **Follow existing patterns** - Match code style, field naming, module organization
4. **Use typed SDK** - Import from `@comind/api` for types (`AppField`, `AppAction`, `makeEntity()`, etc.)
5. **Test** - Run `npm test` at workspace level or per-app with Jest
6. **Deploy & preview:**
   - `npx comind i <app-folder>` - deploys the compiled app. Already-installed workspaces pick up the update.
   - `npx comind i <app-folder> WORKSPACE` - first time: deploys AND installs into that workspace.
   - `npx comind uni <app-folder> WORKSPACE` - uninstalls from a workspace.
   - Then show the updated app to the user (see Preview section)
7. **Push to production** - Commit to `main` (auto-deploys to dev site). When ready, PR from `main` to `prod` (auto-deploys to production, requires signed commits).

### Pre-production deploy checklist

1. **Verify on dev first** - confirm the change works on the dev environment before touching production
2. **Use a deploy user** - create a dedicated comind.work user for deploys so changes are clearly attributed in the audit trail (not mixed with your personal API user)
3. **Ignore API version warnings** - minor version mismatch errors on install (e.g., "API version X expected Y") are non-critical on dev environments and safe to ignore

## Environment setup

The CLI reads credentials from `.config/dev.env` (copied from `.config/dev.env.sample`):

```bash
COMIND_SECRET_AUTH_CODE_MANAGEMENT=<token-from-admin>  # NEVER commit
COMIND_BASE_URL=https://YOUR-ORG.comindwork.com
COMIND_API_V1_URL=https://api.comindwork.com
```

For multiple environments, set `COMIND_ENV=dev|uat|prod` in the root `.env` file - it loads the corresponding `.config/{env}.env` (dev.env, uat.env, prod.env).

### Switching environments step-by-step

1. Copy `.config/dev.env` to `.config/prod.env`
2. In `prod.env`, update `COMIND_SECRET_AUTH_CODE_MANAGEMENT` to the production API key and `COMIND_BASE_URL` to the production URL
3. Set `COMIND_ENV=prod` in the root `.env` file
4. Verify: run `npx comind i <small-app> <test-workspace>` - confirm it targets production
5. To switch back: set `COMIND_ENV=dev` (or remove the line - dev is default)

---

# App Engine Reference

> For build-time API rules and constraints, also check `node_modules/@comind/api/AGENTS.md` in the project.

## Overview

Apps are **TypeScript NPM packages** that compile into JavaScript objects running in two places:

- **Browser** - form rendering, calc fields, view logic, validation
- **C# backend** - via Jint (JS interpreter for .NET) for transitions, server-side logic

> **One app = one database table with advanced UI.** Fields define the data model, actions define transitions, layouts define the form UI.

## Package structure

```
app-name/
+-- package.json          # comindApp field, #typings alias, dependencies
+-- typings.ts            # makeEntity(fields) + JSX type declarations (required)
+-- fields/
|   +-- index.ts          # AppFieldArray, exported via `as const satisfies AppFieldArray` (can split into separate files)
|   +-- fields-*.ts       # Optional topical modules (core, dates, people, etc.)
|   +-- calc-fields.ts    # Calculated field functions (single file OR directory calc-fields/)
+-- actions/
|   +-- index.ts          # AppActionArray, exported via `as const satisfies AppActionArray`
|   +-- logic/            # Server-side transition code (optional)
|       +-- index.ts      # ActionLogic
|       +-- preconditions.ts
+-- views/
|   +-- layouts/
|   |   +-- index.ts      # AppLayout map
|   |   +-- default.tsx   # Form layout (JSX -> XML, NOT React)
|   +-- list-views.ts     # Grid view definitions (optional)
|   +-- logic/            # Client-side view logic (optional)
|       +-- index.ts      # ViewLogic
+-- settings/
    +-- index.ts          # AppSettings (backlinks, access, mutations)
    +-- secrets.ts        # COMIND_SECRET_* env variables (optional, import only from actions/logic/)
    +-- app-mutator.ts    # Schema post-processing (optional)
```

**Required:** `package.json`, `typings.ts`, `fields/index.ts`, `actions/index.ts`, `views/layouts/default.tsx`. System fields and system transitions are inherited from `@comind/base`.

**Common base fields** (inherited from `@comind/base`, available in every app): `title`, `state`, `creation_date`, `assignees_list`, `attachments`, `c_json_data` (arbitrary JSON storage), `description`, `id`. These do not need to be redeclared in your app's fields unless you want to override their options.

**Common base actions** (inherited): `add`, `edit`, `delete`, `copy`, `move`, `archive`, `finally`.

### typings.ts (required in every app)

```typescript
import type { FieldName, LayoutIntrinsicElements } from '@comind/api';
import { makeEntity } from '@comind/api';
import fields from './fields';

export const { entity } = makeEntity(fields);
export type EntityFieldName = FieldName<typeof entity>;
export declare namespace entity.JSX {
  type IntrinsicElements = LayoutIntrinsicElements<typeof entity>;
  type Element = string;
}
```

### package.json

```json
{
  "name": "app-acme-task",
  "comindApp": {
    "publishingAlias": "APPNAME",
    "icon": "icon-name",
    "titleSingular": "Item",
    "titlePlural": "Items",
    "inheritFrom": "@comind/app-task"
  },
  "dependencies": { "@comind/api": "*", "@comind/app-task": "*" },
  "imports": { "#typings": "./typings.ts" }
}
```

**Naming convention:** Use `app-<brand>-<entity>` (e.g. `app-acme-task`, `app-acme-crm-deal`) - one package per brand x entity. The entity segment can be multi-part for domain-scoped apps (e.g. `crm-deal`, `hr-leave-request`). Unscoped names only; scoped names like `@brand/task` break workspace install (see Common errors below).

### Inheritance (`inheritFrom`)

Every app inherits from `@comind/base` at minimum. Customer apps can extend standard apps via `inheritFrom` in `package.json` -> `comindApp`. The child only overrides or adds - everything else is inherited. Multi-level chains supported (C -> B -> A). Composition order: `@comind/base` -> plugins -> parent app -> child app.

**Merge rule:** Deep merge where arrays **replace** rather than concatenate. Last layer wins.

### Inter-app imports

Apps can import from sibling apps. The imported app must be in `dependencies`:

```typescript
import lookups from '@comind/app-pim-attribute/fields/lookups';
import { formatNumber } from '@comind/app-crm-quote/actions/logic/compose';
```

### Field splitting

Fields can be split into topical modules and merged in `fields/index.ts`:

```typescript
import { coreFields } from './fields-core';
import { dateFields } from './fields-dates';

const fields = [...coreFields, ...dateFields] as const satisfies AppFieldArray;
export default fields;
```

## Critical rules

- **Client-side (views/logic/):** Use `ComindView` global. NEVER use `Cmw.` (build warning). NEVER import from `@comind/api/server` (build error).
- **Server-side (actions/logic/):** Use `ComindServer` global. NEVER use `Cmw.` (build warning). NEVER use `ComindView` (build error). No `fetch`, `document`, `window`, or Node APIs - code runs on C# via Jint.
- **Field values in view logic:** Access via `entity.fieldName` - NOT `ComindView.record.get()` (deprecated).
- **Do NOT use `as const satisfies ViewLogic`** or `as const satisfies ActionLogic` - omit `as const` (build error).
- **Do NOT import `makeActionLogic`/`makePreconditionLogic`** - auto-injected during compilation.
- **Secrets:** Import `settings/secrets` only from `actions/logic/` (build error elsewhere - bundle exposes to browser).
- **Custom field prefix:** Always use `c_` prefix for custom fields or lookup fails silently.

## Field types

**DataType:** `text` . `lookup` . `lookupMulti` . `appsLookup` . `linkto` . `linkslist` . `fileslist` . `peoplelist` . `date` . `datetime` . `person` . `bool` . `number` . `object`

**CalcType:** `calcfield` (sync) . `rollup` . `datarollup` . `treerollup` . `shadow` . `tablecalcfield` (all async, processed by background Processor)

**TextType** (sub-type of `text`): `string` . `keyword` . `text` . `richtext`

**NumberType** (sub-type of `number`): `unknown` . `float` . `integer`

**Type utilities:** `FieldName<Entity>`, `EditableFieldName<Entity>`, `fieldName__resolved` (resolved lookup), `fieldName__list` (list accessor)

## Key field options

| Option | Purpose |
|---|---|
| `custom_form_control` | Override widget: `'search-box'`, `'drop-down-simple'`, `'color'`, `'link-list'`, etc. |
| `default_value_expression` | Default value for new records |
| `help_text` / `placeholder_text` | Tooltip / placeholder (LanguageTextObject) |
| `restrict_input` | Validation: `'email'`, `'percent'`, `'url'`, `'phone'`, `'int'` |
| `max_length` / `decimal_places` | Character limit / decimal precision |
| `number_min_value` / `number_max_value` | Numeric range |
| `gridview_renderer` | Grid cell renderer: `'circle_percent'`, `'indicator'`, `'rich_text_line'`, etc. |
| `plain_renderer` | Plain-view renderer: `'colorful-lookup'`, `'avatar-image'`, `'format-from-another-field'`, etc. |
| `gridview_column_max_width` / `gridview_column_min_width` | Column width constraints |
| `lookup_entries_formatting` | Per-entry colors, icons, text colors |
| `lookup_entries_localization` | Per-entry multi-language captions |
| `access` | Field access: `'workspaceAdmin'`, `'workspaceTeam'`, `'groups'` |
| `code_editor_enabled` / `code_editor_language` | Code editor with syntax highlighting |
| `clear_on_transition` | Clear value when a transition fires |
| `formatting_pre` / `formatting_post` | Text before/after value (e.g., `"$"`, `"%"`) |
| `formatting_external_url_base` | Base URL prepended to field value (renders as link) |
| `formatting_icon` | Font Awesome icon class for the field |
| `special_decimal_format` | Number format: `"{0:#,#%}"`, `'HH:MM:SS'` (seconds), `'HH:MM'` (hours) |
| `number_aggregate` | Grid grouping: `'sum'`, `'avg'`, `'max'`, `'min'`, `'none'` |
| `input_mask` | Input mask pattern (string or `{greedy, placeholder, autoUnmask, alias}`) |
| `file_extensions` / `file_max_size_mb` / `maxFiles` | File upload constraints |
| `formatting_style_for_edit` | CSS style when field is editable in grid (e.g., `'color:#0055A1;'` for blue text) |
| `formatting_style_for_changed` | CSS style when field value changed in grid (e.g., `'background-color:beige;'`) |
| `formatting_function` | Helper function name for custom cell rendering in grid |
| `copy_to_clipboard` | Show copy button on field |

Calc fields must have a corresponding function in `calc-fields.ts` (or `calc-fields/` directory). The `dependsOn` array must include data fields that trigger recalculation - system/technical fields like `id` are always available and should NOT be listed in `dependsOn`.

**Calc fields are always readonly** in layouts - do not add `mode="plain"` (it's redundant).

**Peoplelist calc field pattern:** To create a calculated `peoplelist`, define two fields as a pair:
- `c_myfield__list` (`type: 'object'`, `calcType: 'calcfield'`) - the calc function returns `[{id: userId}, ...]`
- `c_myfield` (`type: 'peoplelist'`, `calcType: 'calcfield'`) - the calc function returns a placeholder string `'c_myfield-' + entity.id`

The `__list` field holds the actual user array; the `peoplelist` field is the display wrapper.

**Calc field values are NOT available in action logic.** Calc fields are computed asynchronously by the background Processor, so `entity.c_myfield__list` will be stale or empty in server-side transition code. If action logic needs the same data, recompute it from the source fields (e.g., parse `c_json_data` directly).

## Actions

Actions define state transitions (e.g., "Approve", "Reject", "Submit"). Each action has a `uid`, `caption`, and optional `options`:

| Option | Purpose |
|---|---|
| `showAsButton` | Show as a button in the form toolbar |
| `visible` | Whether action appears in UI (set `false` for internal-only actions) |
| `skipAuditTrail` | No version created, no timestamp update |
| `skipCreatingVersion` | No version, but timestamps still update |
| `keepModalOpened` | Modal stays open after save |
| `showAsMassAction` | Enable in mass-edit mode (default `true`) |
| `viewMode` | Display mode: `'modal'`, `'backlink'`, `'local'`, etc. |
| `backlinkAppAlias` / `backlinkField` | For backlink-type actions: target app and field |
| `invokeFromEmail` | Action can be triggered from incoming emails |

## Server-side API (actions/logic/)

Available in transition logic (runs on C# backend via Jint). Import types from `@comind/api/server`.

**Record context:** `entity` (current, mutable), `entityOld` (previous, read-only), `records` (bulk ops array)

**`currentUser`:** `.id`, `.email`, `.name`, `.timeZone`, `.companyId`, `.isGlobalAdmin`, `.isWorkspaceAdmin`, `.isInGroup(name)`, `.isInAnyOfGroups(names)`, `.isInField(fieldName)`, `.isDefaultUnAuthUser()`, `.isWorkspaceAdminOf()`

**`ComindServer`:**

| Method | Purpose |
|---|---|
| `.action` | Current transition being executed |
| `.Acls.group(name)` / `.Acls.metaGroup(name)` | Access control list references |
| `.queryApp(wksApp, relex, fields?)` | Fetch single record |
| `.queryAppCached(wksApp, relex, fields?)` | Fetch single record (cached) |
| `.queryAppRecords(wksApp, relex, fields?: string[], sortBy?: string[])` | Fetch multiple records (also accepts params object with `limitRecords`, `skipRecords`) |
| `.queryAppRecordsCached(wksApp, relex, fields?: string[], sortBy?: string[])` | Fetch multiple records (cached) |
| `.createAppRecord(appAlias, data, runAsUser?)` | Create record |
| `.updateAppRecord(id, transition, values, runAsUser?)` | Update via transition |
| `.deleteAppRecord(id, runAsUser?)` | Delete record |
| `.bulkCreateAppRecords(wksApp, dataList)` | Bulk create |
| `.bulkDeleteAppRecords(ids)` | Bulk delete |
| `.importData({data, keyFields, ...})` | Import with duplicate handling |
| `.resolveLookup(fieldName)` / `.matchLookup(fieldName, caption)` | Lookup resolution |
| `.generateUniqueIdForPrefix(prefix)` | Auto-numbering |
| `.getAccessPass(...)` | Impersonation token |
| `.htmlToText(html)` | Strip HTML tags |
| `.sleep(ms)` | Pause execution |

**`RestApi`:** Fluent builder - `.createClient(url)` -> `.createRequest(path, method)` -> `.addHeader()` / `.addJsonBody()` / `.addParameter()` / `.addQueryParameter()` / `.addUrlSegment()` / `.addFile()` / `.setRequestTimeout(seconds)` -> `.executeRequest()` or `.downloadFile()`. Auth: `.setComindAuth()`, `.setHttpBasicAuth()`, `.setOAuth2AuthorizationHeaderAuth()`

**RestApi auth patterns** (for calling any external API from server-side action logic):

```typescript
// Basic Auth
RestApi.createClient("https://api.example.com");
RestApi.createRequest("/data", "GET");
RestApi.setHttpBasicAuth("username", "password");
const res = RestApi.executeRequest(); // { statusCode, content, headers }

// Bearer token (OAuth2 / API key)
RestApi.createClient("https://api.example.com");
RestApi.createRequest("/data", "POST");
RestApi.setOAuth2AuthorizationHeaderAuth(token);
RestApi.addJsonBody({ key: "value" });
const res = RestApi.executeRequest();

// Manual header (API key in custom header)
RestApi.createClient("https://api.example.com");
RestApi.createRequest("/data", "GET");
RestApi.addHeader("X-API-Key", apiKey);
const res = RestApi.executeRequest();
```

Always store credentials in `settings/secrets.ts` and retrieve via `AppSchema.getSecrets()` - never hardcode.

**`Files`:** `.getFileRef()`, `.persistFileRef()`, `.readExcelFileRef()`, `.writeExcelFileRef()`, `.readCsvFileRef()`, `.writeCsvFileRef()`, `.writePdfFileRef({html})`, `.mergePdfFileRefs()`, `.zipFileRefs()`, `.readFileRefAsBase64()`, `.useBase64DataAsFileRef()`

**`AppSchema`:** `.load()`, `.install()`, `.uninstall()`, `.getVars()`, `.getSecrets()`, `.getWorkspaceAdmins()`

**`Utils`:** `.formatString()`, `.formatDateTime()`, `.formatNumber()`, `.getDatesDiff()`, `.generateGuid()`, `.generateSha256()`, `.generateMd5()`, `.parseJwt()`, `.writeCsvString()`, `.parseCsvString()`, `.dateTimeToAnotherTimeZone()`

**`console`:** `.log()`, `.debug()`, `.warn()`, `.error()`, `.profile()` / `.profileEnd()`

**`LiquidParser`:** `.parse(template, vars)` - Liquid template engine. Variables accessed via `Vars.` prefix. Supports `{% if %}`, `{% for %}`, `{{ var | filter }}`, `{% assign %}`.

### ActionLogic export pattern

```typescript
import { entity } from '#typings';
import type { ActionLogic } from '@comind/api';

export default {
    approve,
    reject,
} satisfies ActionLogic<typeof entity>;

function approve() {
    entity.approved_by = currentUser.id;
    entity.approved_date = new Date();
}
```

Plugin logic combines in sequence: base runs first, then plugins in dependency order, then app's own logic.

### PreconditionLogic export pattern

Preconditions return `boolean` - `true` means the action is available:

```typescript
import { entity } from '#typings';
import type { PreconditionLogic } from '@comind/api';

export default {
    approve: () => entity.state === 'pending',
    reject: () => entity.state === 'pending',
} satisfies PreconditionLogic<typeof entity>;
```

### Notifications (`__sendNotifications`)

Return `NotifyContext` from a special `__sendNotifications` method in ActionLogic:

```typescript
import type { NotifyContext } from '@comind/api/server';

export default {
    approve,
    __sendNotifications: sendNotifications,
} satisfies ActionLogic<typeof entity>;

function sendNotifications() {
    const ctx: NotifyContext = {
        to: entity.assignees_list__list?.map(u => u.id) || [],
        cc: [entity.c_supervisor],
        subject: `Approved: ${entity.title}`,
        email_body: '<h2>Record approved</h2>',
        email_layout: 'SimpleLayoutNotification',
        attachment_fields: 'attachments',
    };
    return ctx;
}
```

**`email_layout`:** `'SimpleLayoutNotification'` (custom HTML) or `'RecordNotification'` (auto record details).

**Other `NotifyContext` options:** `bcc`, `replyTo`, `override_from_email`, `override_from_email_name`, `override_item_prefix`, `override_unsubscribe_text`, `email_note`, `returnPath`, `access_email_data_as`.

### PDF generation

```typescript
const pdfRef = Files.writePdfFileRef({ html: '<h1>Invoice</h1>...' });
const stored = Files.persistFileRef(pdfRef);
entity.attachments__list ??= [];
entity.attachments__list.push({ file_uid: stored.uploadedFileId, title: 'Invoice.pdf' });
```

File refs are **temporary** - always call `persistFileRef()` before attaching. Use inline CSS only (no external stylesheets). `Files.mergePdfFileRefs([ref1, ref2])` merges PDFs. `Files.zipFileRefs([refs])` creates ZIP archives. Combine with `LiquidParser.parse()` for data-driven templates.

### File download and attachment

```typescript
RestApi.createClient(fileUrl);
RestApi.createRequest("", "GET");
const res = RestApi.downloadFile(); // { statusCode, downloadedFileRef, contentLength }
const stored = Files.persistFileRef(res.downloadedFileRef);
entity.attachments__list ??= [];
entity.attachments__list.push({ file_uid: stored.uploadedFileId, title: 'file.pdf' });
```

### External API calls (e.g., AI-based actions)

```typescript
const { COMIND_OPEN_AI_API_KEY } = AppSchema.getSecrets();
RestApi.createClient("https://api.openai.com/v1/responses");
RestApi.createRequest("", "POST");
RestApi.addHeader("Authorization", `Bearer ${COMIND_OPEN_AI_API_KEY}`);
RestApi.addJsonBody({ input, instructions, model: "gpt-5-nano" });
const response = RestApi.executeRequest();
```

## Client-side API (views/logic/)

Available in view logic (runs in browser). Import types from `@comind/api/view`.

### ViewLogic export pattern

```typescript
import type { EntityFieldName } from '#typings';
import { entity } from '#typings';
import type { ViewLogic } from '@comind/api';

export default {
    onReady,
    getReadonlyFields,
    getRequiredFields,
} satisfies ViewLogic<typeof entity>;

function getReadonlyFields() {
    const fields: EntityFieldName[] = [];
    if (!ComindView.currentUser.isWorkspaceAdmin) fields.push('accountable_account_id');
    return fields;
}
```

### ViewLogic methods

| Method | Purpose |
|---|---|
| `onReady()` | Form loaded |
| `onRefresh()` | Form refreshed (entity not yet updated) |
| `onDestroy()` | Form closing |
| `onReset()` | Form reset |
| `onBeforeSave()` | Pre-save hook (return `false` or error string to block) |
| `onGridLoaded()` | Embedded grid finished loading |
| `onInputChange()` | Field value changed |
| `getInvisibleFields()` | Return field names to hide |
| `getRequiredFields()` | Return required field names |
| `getReadonlyFields()` | Return read-only field names |
| `getFilterKeys()` | Filter keys for lookups |
| `getFieldOverrides()` | Override field schema dynamically |
| `helperFunction()` | Register helper functions |

Plugin ViewLogic methods combine in sequence. If any `onBeforeSave` returns `false`, save is blocked. Engine re-evaluates field control methods whenever the record changes.

### ViewLogic cookbook

**Conditional visibility - hide field from non-admins:**

```typescript
function getInvisibleFields() {
    const fields: EntityFieldName[] = [];
    if (!ComindView.currentUser.isWorkspaceAdmin) {
        fields.push('c_internal_notes', 'c_cost');
    }
    return fields;
}
```

**Conditional required fields - require reason when rejecting:**

```typescript
function getRequiredFields() {
    const fields: EntityFieldName[] = [];
    if (entity.state === 'rejected' || ComindView.action === 'reject') {
        fields.push('c_rejection_reason');
    }
    return fields;
}
```

**Conditional readonly - lock fields after approval:**

```typescript
function getReadonlyFields() {
    const fields: EntityFieldName[] = [];
    if (entity.state === 'approved' || entity.state === 'closed') {
        fields.push('title', 'c_priority', 'c_due_date', 'c_assignee');
    }
    return fields;
}
```

**Validation before save - block save with error message:**

```typescript
function onBeforeSave() {
    if (entity.c_start_date && entity.c_end_date && entity.c_start_date > entity.c_end_date) {
        return 'End date must be after start date';
    }
    if (ComindView.action === 'approve' && !entity.c_reviewer) {
        return 'Reviewer is required before approval';
    }
}
```

All methods must be exported from the default object in `views/logic/index.ts`:

```typescript
export default {
    getInvisibleFields,
    getRequiredFields,
    getReadonlyFields,
    onBeforeSave,
} satisfies ViewLogic<typeof entity>;
```

### ComindView object

| Property/Method | Purpose |
|---|---|
| `.action` | Current action name (e.g. `'add'`, `'edit'`) |
| `.actionMode` | `'mass-edit'`/`'editable'`/`'editing'`/`'full-form'`/`'plain-with-comment'`/`'plain'` |
| `.workspace` | Workspace info: `.alias`, `.current_user_id`, `.group_participants`, `.tabs` |
| `.schema` | App schema object with `.publishing_alias`, `.lookups` |
| `.currentUser` | `.id`, `.email`, `.isWorkspaceAdmin`, `.isGlobalAdmin`, `.isInGroup(name)` |
| `.parentRecord(fieldName?)` | Get parent record (sync, from context) |
| `.childRecords(alias?, field?)` | Get child records (sync, from context) |
| `.siblingRecords()` | Get sibling records (sync, from context) |
| `.getLinkedRecordAsync(fieldName)` | Resolve linked record (async) |
| `.getRecordById(id)` | Fetch any record by ID (async) |
| `.request(url, config?)` | HTTP request from client |
| `.getVars()` | Workspace-level settings |
| `.getSetting(name)` | Get app setting value |
| `.addGridRow(values, alias?, field?)` | Add row to embedded grid |
| `.ui` | Property binding for custom layouts (use with `ngBindHtml`) |
| `.uiUtils` | `.getFgHexColorByBg()`, `.safeDigest()`, `.setTabCounter()` |

### Format functions for list display

Export a named function `fieldName_format_function` and register via `ComindView.addHelperFunction()`:

```typescript
export const c_image_url_format_function = (entity: typeof entityType) => {
    if (!entity.c_image_url) return null;
    return {
        text: '',
        text_style: 'display:inline-block;width:100px;height:38px;' +
            `background-image:url('${entity.c_image_url}');background-size:contain`,
        tooltip: entity.c_image_url,
    };
};
ComindView.addHelperFunction(c_image_url_format_function);
```

Requires `plain_renderer: 'format-from-another-field'` on the field.

## JSX Layout System

**TSX files compile to XML strings, NOT React.** The `entity` JSX factory converts elements to XML. CamelCase props become kebab-case attributes. Field name values keep their original form (`c_priority` stays `c_priority`).

### Layout structure and attributes

```tsx
/** @jsx entity */
import { entity } from '#typings';

export default (
<layout collapsedTabs="false">
  <section caption="Details" tab="Main" icon="info-square" sidebar="title"
           captionAboveField="true" position="100">
    <column>
      <column-heading caption="Subheading" />
      <field name="title" required="true" />
      <field name="c_priority" renderer="radio" />
      <field name="c_dates" name2="finish_on" renderer="date-range" />
      <field name="c_notes" mode="plain" widthFactor="3" />
    </column>
  </section>
</layout>
);
```

**Key field attributes:** `required`, `mode` (`"plain"` = read-only), `caption` (override), `captionAbove`, `renderer` (`"radio"`, `"checkboxes"`, `"date-range"`), `widthFactor` (column width proportion), `name2` (paired field for ranges).

**Section attributes:** `caption`, `tab` (tab grouping), `icon`, `sidebar`, `position`, `showCaptions`, `captionAboveField`, `visibility`.

**Layout-level `collapsedTabs`:** Set to `"false"` when the layout contains only tab-sections (no top-level field-bearing sections). Optionally also set `"false"` when you want tabs expanded by default on the `add` action. Omit on layouts with no tab-sections.

**Section visibility (non-obvious):** A section's render decision depends on its fields. A field can fail to render for several reasons: hidden by UI logic (conditional visibility rules), empty-and-readonly (no editing control needed), or blocked by permissions (field never loaded). When **all** the section's listed fields fail to render for any combination of these reasons, the UI treats the section as fully empty and skips it entirely - **even if the section contains a `<custom-layout>` with custom UI**. To force the section (and its custom-layout) to render anyway, set `visibility="true"` explicitly on the `<section>`.

Every layout `.tsx` file needs `/** @jsx entity */` pragma and the `entity` import.

### Multiple layouts and custom layouts

Apps can have multiple `.tsx` files in `views/layouts/`. The builder auto-discovers them - **do NOT create `views/layouts/index.ts`** unless you need conditional logic to select which layout to show for a given action (e.g., different form for `add` vs `edit`). Reusable fragments:

```tsx
<custom-layout name="custom_layout_totals" />
<custom-layout name="custom_layout_preview" widthFactor="3" />
```

**CSS best practices for custom layouts:**
- Use `<style jsx>` inside the layout TSX - it scopes styles to the app's form
- Prefix all class names with the app alias (e.g., `.balton-totals`) to avoid collisions with platform CSS
- Avoid changing `font-family` on the body or form container - platform CSS loads after custom styles, causing a visible flash/re-render. Target specific elements instead
- Inline styles on elements are safest for one-off overrides

Custom layouts can contain raw HTML, inline CSS, and Angular template expressions (`ngBindHtml` for dynamic HTML from `ComindView.ui`):

```tsx
/** @jsx entity */
import { entity } from '#typings';

export default (
    <layout>
        <style jsx>{`.totals { float: right; margin-right: 70px; }`}</style>
        <div class="totals">
            <div ngBindHtml="ui.movieLink"></div>
        </div>
    </layout>
);
```

### Editable grid

For spreadsheet-like multi-row editing within forms (e.g., invoice rows):

```tsx
<editable-grid
  caption="Line items"
  fieldName="c_parent"
  publishingAlias="INVOICEROW"
  columnSettings='[{"name":"title","width":300,"required":true},{"name":"c_amount","width":100}]'
  rowButtons='["open","insert","delete"]'
  showTotals="true"
  widthFactor="3"
/>
```

### Widgets

Embed data visualizations from other apps:

```tsx
<widget appAlias="HISTORY" widgetType="table" widgetLink="WKS/HISTORY" />
```

## Settings

| Tier | Who | Scope | Access in code |
|---|---|---|---|
| System-settings | System admin | All workspaces, triggers redeployment | `import` (compile-time) |
| Workspace-settings | Workspace admin | Single workspace, immediate effect | `ComindView.getVars()` or `ComindServer.getVars()` (runtime) |
| Personalization | End-user | Per-user | App-defined |

**Never use `import` for workspace settings** - it returns compile-time value, not workspace-specific.

### Common settings keys

**`vars`** - workspace-configurable variables accessed via `getVars()`:
```typescript
vars: {
    group_default_responsible: 'Task default assignees',
    org_units: 'IT, HR, Legal',
}
```

**`backlinks_settings`** - configure how linked records appear:
```typescript
backlinks_settings: [
    { app: 'TASK', field: 'refer_task_id', position: 10, visibility: true },
]
```

**`solution`** - group related apps:
```typescript
solution: {
    title: 'PM: Project and task management',
    apps: ['app-task', 'app-timelog', 'app-wiki'],
}
```

**`app-mutator.ts`** - dynamically modify app schema at runtime based on workspace settings:
```typescript
export default function (app: AppSchemaObject) {
    const types = AppSchema.getVars().app__available_types?.split(',') || [];
    app.fields.c_type.lookupOptions = types.map(t => ({ uid: t.trim(), caption: t.trim() }));
}
```

**Secrets** - accessed via `AppSchema.getSecrets()` (server-only, never sent to browser):
```typescript
const { COMIND_OPEN_AI_API_KEY } = AppSchema.getSecrets();
```

## CLI commands

| Command | Alias | Purpose |
|---|---|---|
| `npx comind install <app> [workspace] [position]` | `i` | Build + deploy (+ install to workspace if specified) |
| `npx comind uninstall <app> <workspace>` | `uni` | Uninstall from workspace |
| `npx comind build <app>` | | Compile to JS in `__upload/build/` |
| `npx comind promote <app>` | | Promote app to another environment |
| `npx comind createRecords ...` | | Create records from CSV/JSON |

**Type checking:** Run `npm run typecheck` in the project root to verify TypeScript types across all apps.

## List views

Define views in `views/list-views.ts` as `AppListView[]`. Each view specifies columns, filters, sorting, and grouping. Views can be organized into folders (set `is_folder: true`).

**Visualization modes:**

`gridTableView` (default) . `boardsView` . `boardCardsView` . `ganttView` . `calendarTimeView` . `calendarHeatMapView` . `feedView` . `pivotableReportsView` . `pivotChartsView` . `vegaView` . `googleMapsView` . `wordsCloudView` . `fileFoldersView` . `quickEditView`

**Cell renderers** (set via `gridview_renderer`): `circle_percent` (progress) . `indicator` (status) . `rich_text_line` . `colorful-lookup` . `avatar-image` . `work-progress-bar` . boolean checkboxes . currency . sparklines . priority indicators . linked records . file previews . comments count

**Renderer examples:**

```typescript
// Colored status indicator in grid
{ name: 'c_status', type: 'lookup', textType: 'keyword',
  options: {
    gridview_renderer: 'indicator',
    lookup_entries_formatting: {
      active:   { bgColor: '#4caf50', textColor: '#fff' },
      pending:  { bgColor: '#ff9800', textColor: '#fff' },
      closed:   { bgColor: '#9e9e9e', textColor: '#fff' },
    },
  }
}
// Progress bar in grid
{ name: 'c_progress', type: 'number', numberType: 'integer',
  options: { gridview_renderer: 'circle_percent' }
}
// Avatar image from URL field
{ name: 'c_photo', type: 'text', textType: 'string',
  options: { gridview_renderer: 'avatar-image' }
}
```

**Filtering:** Relational expression syntax - `state="open" AND priority>3`. Faceted filtering via lookup facets, multi-select, date range.

## Available plugins

`plugin-notification` . `plugin-follower` . `plugin-permission` . `plugin-reminder` . `plugin-recurrence` . `plugin-color` . `plugin-icon` . `plugin-cover-image` . `plugin-contact` . `plugin-country` . `plugin-custom-fields-section` . `plugin-rice-score` . `plugin-task-rate` . `plugin-google-drive` . `plugin-cloudflare` . `plugin-demo-data` . `plugin-admin-edit` . `plugin-pack-basic`

## Multi-language support

```typescript
caption: { en: 'Priority', uk: 'Пріоритет', de: 'Priorität' }
// Type: CaptionLanguage = 'en' | 'uk' | 'zh' | 'de' | 'fr' | 'es'
```

---

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `COMIND_SECRET_AUTH_CODE_MANAGEMENT is not set` | `.config/dev.env` missing or not in the working directory | Copy from `.config/dev.env.sample`, fill in token. If in a worktree, ensure config exists in the worktree path |
| Install fails with 401 / auth rejected | `.config/dev.env` contains template placeholders (`<token-from-admin>`, `YOUR-ORG.comindwork.com`) instead of real values | Replace placeholders with real values from your admin |
| `API version mismatch` (non-critical warnings on install) | Dev environment runs a slightly older API version than the SDK expects | Safe to ignore on dev. If on production, update the SDK |
| `Cannot find module '@comind/api'` | `npm install` not run in project root, or `.npmrc` missing the private registry | Run `npm i` in project root after ensuring `.npmrc` points to the org's private registry |
| `ERROR: These fields miss obligatory "type" attribute` | Field schema uses legacy `dataType:` key | Rename to `type:` (the `DataType` value names like `'text'`, `'lookup'` stay the same) |
| `Warning: Detected deprecated keyword "AppAction[]"` (or `"AppField[]"`) | Using the deprecated array type | Replace `const actions: AppAction[] = [...]` with `const actions = [...] as const satisfies AppActionArray` (same for fields / `AppFieldArray`) |
| Deploy succeeds but workspace install fails with `500 - Schema by provided appId cannot be loaded` | `package.json` uses a scoped name like `@brand/task` - the platform's AppId resolver doesn't bind scoped names to a workspace | Rename to an unscoped name following the `app-<brand>-<entity>` convention (e.g. `app-acme-task`), rerun `npm i` to refresh symlinks, retry |
| `Build error: Cannot use import statement` in actions/logic | Importing from `@comind/api/view` in server-side code (or vice versa) | Server code: import from `@comind/api/server`. Client code: import from `@comind/api/view` |
| `Lookup field returns undefined` | Custom field missing `c_` prefix, or using display caption instead of db value | Always prefix custom fields with `c_`. Use db values from schema, not captions |
| Claude refuses to edit `.env` files | Claude treats env files as security-sensitive | Approve the edit when prompted. Never paste secrets into the chat |
| CSS styles cause visible flash on form load | Custom `font-family` or broad selectors override platform CSS that loads later | Target specific elements with scoped class names instead of broad selectors |
