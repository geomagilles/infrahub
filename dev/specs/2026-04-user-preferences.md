---
Title: User & Global Preferences
Author:
  - Paul Lemesle
Status: draft
---

# User & Global Preferences

## Summary

Introduce persistent user and organisation-wide preferences, modelled as a **`StandardNode` object**
(the internal object type used by `Branch`/`Root`, `backend/infrahub/core/node/standard.py`) — **not**
a schema node.

A single **`Preference`** class serves both: each row is owned by a principal identified by
`owner_id` — an **account id** for a user's preferences, or the **Root node id** (`registry.id`) for
the organisation-wide (global) preferences. One class, one set of fields, no duplication.

Three custom GraphQL read fields (effective / user / global) and one write mutation make up the
entire surface — there is no auto-generated schema CRUD (like `Branch`), which is exactly what gives
us control over visibility. The **effective** field returns the caller's resolved view (global merged
with their personal overrides) so the frontend never merges itself.

V1 ships only two fields — `date_format` and `timezone` — to validate the model, query, mutations,
and UI plumbing end to end. Additional preferences (dark mode, etc.) land in follow-up tickets.

Three adjacent concerns get their own spec/ticket and are explicitly **not** part of V1: saved views
(`dev/specs/2026-04-saved-views.md`), saved filters (`dev/specs/2026-04-saved-filters.md`), and
"show extra fields" persistence (`dev/specs/2026-04-show-extra-fields.md`).

## Why `StandardNode` and not a schema node

This is the central design decision, reversing an earlier draft that used `CoreGlobalPreference` /
`CoreUserPreference` schema nodes.

A schema `Node` gets an **auto-generated generic GraphQL query** and is governed by **per-kind**
object permissions. Infrahub permissions cannot restrict reads **per row** — so any account with read
on the kind could query *every* user's preference row. A `StandardNode` has **no schema-registry
entry and no auto-generated GraphQL**: the only read/write paths are the resolvers we write by hand,
so visibility is **structural** — there is no generic query that could leak another user's
preferences, and each resolver binds to the calling account. Secondary benefits: it keeps the core
schema lean, and `StandardNode`s are global (not branch-versioned), which matches preferences exactly.

> **Known limitation (accepted for V1).** A `StandardNode` cannot declare a schema relationship with
> `on_delete: cascade` (that is a schema-`Node` feature), so `owner_id` is a plain string, not a graph
> link. Deleting an account therefore leaves its `Preference` row behind as unreachable dead data.
> Account ids are UUIDs and never reused, so such a row is permanently unreachable and benign. Cleanup
> is tracked in Jira and out of scope for V1.

## Solution Overview

One `StandardNode` class, owner-keyed; three custom read fields; one write mutation.

| Rows | Owner (`owner_id`) | Who writes | Who reads |
|---|---|---|---|
| Global (one) | the Root id (`registry.id`) | Holders of `manage_global_preferences` (super admins implicitly) | Any authenticated account (via the effective field); the raw global read is gated |
| Per user | the account id | The owning account only | The owning account only |

Effective resolution per field: **user value if set, else global value, else the client default**.
The backend stores `null` for "no opinion", and reads **never create** a row — a missing row simply
means "nothing set". On the web the client default is the **browser's own value** (browser locale
formatting; browser-resolved timezone); the backend does not render dates itself (clients do).

## Success Criteria — V1

- An admin (holder of `manage_global_preferences`) can set `date_format`/`timezone` once for the
  organisation; any user without a personal override sees those values.
- A user can override either field; the override takes effect on next load and is visible from any
  device. Clearing an override falls back to the global value, then the client default.
- **A user cannot read another user's preferences** — no generic query, and each resolver only ever
  touches the caller's `owner_id`.
- The effective field returns the value to render with — no client-side merging.

## V1 Fields

One `Preference` class; on a `StandardNode` these are plain pydantic fields, not schema attributes:

| Field | Type | Notes |
|---|---|---|
| `owner_id` | `str` | The account id (user preferences) or the Root id (global preferences). |
| `date_format` | `str \| None` | A **semantic format key** (see below), validated against the `DateFormat` enum — never an arbitrary string. |
| `timezone` | `str \| None` | IANA timezone name (`Europe/Paris`, `UTC`). Unset = the client's own zone. |

### Date format: semantic keys

`date_format` stores a **semantic key**, never a library-specific rendering pattern, so the stored
value is decoupled from any one client's formatter. The web app maps the key to a date-fns pattern; a
future non-web client would map it however it renders. A single Python enum
**`DateFormat`** (`core/preferences/constants.py`) is the source of truth: the GraphQL `DateFormat`
enum is derived from it (`Enum.from_enum`, `graphql/types/preferences.py`) and the `Preference` model
validates `date_format` against it, so an invalid key is rejected both at the model and the API.

| Key | Web (date-fns) | Example |
|---|---|---|
| `ISO_8601` | `yyyy-MM-dd'T'HH:mm:ssXXX` | `2026-07-01T14:30:00+02:00` |
| `ISO_DATETIME` *(default)* | `yyyy-MM-dd HH:mm` | `2026-07-01 14:30` |
| `ISO_DATETIME_SECONDS` | `yyyy-MM-dd HH:mm:ss` | `2026-07-01 14:30:00` |
| `EU_DATETIME` | `dd/MM/yyyy HH:mm` | `01/07/2026 14:30` |
| `US_12H` | `MM/dd/yyyy hh:mm a` | `07/01/2026 02:30 PM` |

Every preset includes date **and** time. The set is deliberately limited to formats that render
identically on every client with no locale library. Locale-dependent forms and a relative-time mode
were considered and **dropped** (locale forms aren't portable; relative time is a display mode, not a
format). The backend does **not** render dates from the key — there is no server-side renderer in V1;
clients render. If a server-side consumer (notifications, exports) ever needs it, a renderer is added
then.

## Backend

### Model (`StandardNode`)

`backend/infrahub/core/preferences/models.py` — one class following `Branch`/`Root`:

```python
class Preference(StandardNode):
    owner_id: str
    date_format: Optional[str] = None   # Optional[str] required by StandardNode.guess_field_type
    timezone: Optional[str] = None

    @field_validator("date_format")     # rejects any value not in DateFormat
    ...

    @classmethod
    async def get_for_owner(cls, db, owner_id) -> Preference | None: ...   # never creates
    @classmethod
    async def get_for_owners(cls, db, owner_ids) -> dict[str, Preference]: ...  # one query, for effective
```

- Persisted via the `StandardNode` Cypher queries; a small `PreferenceGetByOwnerQuery`
  (`core/query/preference.py`) fetches by `owner_id IN [...]` (targeted, never scans every row).
- **Reads never create a row**, and there is **no init seed** — a missing row means "nothing set".
  The row is created lazily only on the first write (the sole create path), under a per-`owner_id`
  distributed lock (namespace `PREFERENCE_LOCK_NAMESPACE`) so concurrent first-writes for the same
  owner can't duplicate and concurrent updates can't lose a field.
- **No schema definition, no generic CRUD, no graph migration.**
- Module hygiene: the `manage_global_preferences` permission constant lives in
  `core/preferences/permissions.py`; `core/preferences/__init__.py` is re-exports only.

### GraphQL surface (custom, `Branch`-style)

Shared GraphQL vocabulary (the `DateFormat` and `PreferenceSource` enums, the write-scope enum, and
the typed response objects) lives in a neutral `graphql/types/preferences.py`, imported by both the
query and mutation modules (the mutation does not import from the query module). Wired into the root
query/mutation in `graphql/schema.py`.

**Reads — three distinct typed fields** (each scope has its own shape and permission, rather than one
query whose meaning varies by a `scope` argument):

```graphql
query {
  InfrahubEffectivePreferences {          # caller's resolved view; any authenticated account
    date_format { value source }          # value: DateFormat | null; source: USER | GLOBAL | DEFAULT
    timezone    { value source }          # value: String    | null
  }
  InfrahubUserPreferences  { date_format timezone }   # caller's OWN raw values (null where unset)
  InfrahubGlobalPreferences { date_format timezone }  # org-wide raw values; gated
}
```

- **Effective** — per field: user value if set, else global, else `DEFAULT` (value `null` → the
  client applies its own default). The global row is read internally; raw org values are never exposed
  here. Reads both owner rows in ONE query.
- **User** — the caller's own raw values, bound to `account_session.account_id`; no account argument.
- **Global** — the org-wide raw values; **requires `manage_global_preferences`**, raised before any
  read (used by the Organisation-defaults editor, which needs the raw global).

The response is **typed** (`date_format` is the `DateFormat` enum, not a stringly `{key, value}`
list), so the schema is self-describing and the frontend needs no hardcoded key list.

**Write — one mutation, write-only scope enum:**

```graphql
mutation { InfrahubSetPreferences(scope: USER, date_format: EU_DATETIME, timezone: "Europe/Paris") { ok date_format timezone } }
```

- `PreferenceWriteScope` has only **`USER`** and **`GLOBAL`** — `EFFECTIVE` is not a member, so writing
  the resolved view is unrepresentable (no runtime guard needed).
- `scope: USER` → the caller's own row (`owner_id = account_id`); no account argument, so no path to
  write another user's row. `scope: GLOBAL` → the Root-owned row, gated on `manage_global_preferences`.
- Omitted arg = leave unchanged; explicit `null` = reset the field. `date_format` is the `DateFormat`
  enum (unknown key rejected at the GraphQL layer).

### Permissions

Enforced imperatively at each entry point, fail-closed (unauthenticated/anonymous rejected first):

| Operation | Allowed for | Mechanism |
|---|---|---|
| Effective read | Any authenticated account (own resolved view) | Binds to `account_session.account_id`; global read internally, no gate |
| User read/write | The owning account only | Bound to `account_session.account_id`; no account argument |
| Global read **and** write | Holders of `manage_global_preferences` (super admins implicitly) | `active_permissions.raise_for_permission(...)` before any raw read / write |

The global scope is gated on **read and write** by `GlobalPermissions.MANAGE_GLOBAL_PREFERENCES`,
checked imperatively (the `Branch`/global-permission idiom, `permissions/manager.py`) — not via the
object-permission pipeline (schema-`Node`-specific).

**Frontend gating signal.** The preferences reads do **not** return a permission flag. The frontend
determines whether the user may manage global preferences via the **generic `InfrahubPermissions`
query** (checking for the `manage_global_preferences` global permission) — the same mechanism used
elsewhere — rather than a bespoke boolean bolted onto a data query. The backend remains the source of
truth (the global read/write enforce the permission regardless).

## Frontend

### Data layer

- `useEffectivePreferences()` reads `InfrahubEffectivePreferences` and exposes the typed, resolved map
  `prefs.date_format` / `prefs.timezone` as `{ value, source }` (source `user`/`global`/`default`).
  A `source: "default"` value is `null`, so the consumer applies the browser value.
- `useGlobalPreferences()` reads `InfrahubGlobalPreferences` (raw org values), used only by the
  Organisation-defaults editor; gated server-side.
- A `useCanManageGlobalPreferences()` hook (backed by the generic `InfrahubPermissions` query) gates
  the Organisation-defaults tab.
- Writes go through `InfrahubSetPreferences(scope, …)` — the user card writes `scope: USER`, the org
  card `scope: GLOBAL`; success invalidates the effective (and global) queries.

### Preferences surfaces (account settings)

- **Personal preferences** — a "Preferences" card on the Profile tab (`/profile`), below the account
  details. Each field is pre-filled from the caller's own override; when they have none the control
  shows the placeholder **"Automatic (inherited)"** (the value is inheriting the org/browser default).
- **Organisation defaults** tab (`/profile/organisation-defaults`) — edits the raw global values;
  visible only when the caller may manage global preferences (via the permission hook above).
- **Clearing an override** uses the shared combobox's standard behaviour: re-selecting the currently
  selected value clears it (→ inherit). There is no bespoke "Automatic" option or reset button — this
  keeps the preferences dropdowns consistent with every other combobox in the app.
- **Source indicator.** An **(i) info icon** to the right of each field explains where the current
  effective value comes from: your preference / the organisation default / your browser.
- Both dropdowns use the same shared `ComboboxField` at the same fixed width; the date-format options
  are labelled by their format, with a live example beside the input. The timezone list ensures `UTC`
  is present (V8/Chrome omits it from `Intl.supportedValuesOf`).

### Date rendering — one preference-aware mechanism (PR3)

All user-facing dates render against the user's `date_format` + `timezone` preferences through a
**single mechanism**, so rendering is consistent app-wide without per-site preference plumbing:

- **`useFormatDate()`** (`shared/context/date-preferences-context.tsx`) is the one entry point:
  `formatDate(date, variant?)` with variants **`datetime`** (default — the user's full preferred
  pattern in their timezone), **`date`** (date-only, derived by stripping the pattern at the first
  time token), and **`relative`** ("x ago", timezone-independent). Use the hook when code needs a
  date *string*; use the `DateDisplay` component when rendering JSX.
- **Layering.** The hook reads a `DatePreferencesContext` defined in `shared` (so `shared` carries no
  dependency on `entities/preferences`); a `DatePreferencesProvider` in `entities/preferences` fills
  it from `useEffectivePreferences()` and is mounted app-wide in `app.tsx`. When no provider is
  mounted, or a preference's `source` is `"default"`, formatting falls back to the **browser locale
  and zone** (`toLocaleString`) — never a hardcoded pattern.
- **`DateDisplay`** keeps its historic look by default (relative "x ago" for recent dates, compact
  date otherwise), but its **tooltip** now shows the preferred full datetime+timezone, and a new
  **`variant="datetime"`** renders the full preferred timestamp inline. A `dateFormat` prop remains
  as an explicit escape hatch for the rare site that must pin a specific pattern.
- **Timezone.** date-fns v4 + the first-party **`@date-fns/tz`** (`TZDate`) render an instant in the
  chosen IANA zone; `patternForKey` (exported from `date-format-presets.ts`) maps the semantic key to
  the date-fns pattern.
- **Migration.** Every user-facing raw date render (ad-hoc `format()`/`toLocaleString()`, the four
  named sites `global-event.tsx`/`time-selector.tsx`/`duration-display.tsx`/`search-nodes.tsx`, plus
  metadata tooltips, token expiry, last-refresh, filter tags, …) routes through `DateDisplay`/the
  hook at its original granularity. **Not migrated:** form inputs/date-pickers, `toISOString()` values
  sent to APIs or used as keys, chart axes, and tests — those are machine/serialized, not
  preference-driven display.

## Future Preferences (out of V1)

Dark mode / theme, language, density, default landing page — each needs its own design pass; listed
as a backlog hint, not committed scope.

## Out of Scope (separate specs/tickets)

Saved views, saved filters, "show extra fields" persistence, `branch_delete_mode` (dropped —
persisting a destructive default is unsafe), cross-account import/export, schema-graph view state.

## Resolved Decisions

- **Model** — one owner-keyed `Preference` `StandardNode` (`owner_id` = account id, or the Root id for
  global), not two classes and not schema nodes. Reads/writes are custom GraphQL only.
- **Reads never create; no init seed** — a missing row means "nothing set"; the row is created lazily
  only on the first write. (Avoids a `get_*` that writes, and keeps reads routable to replicas.)
- **`date_format` is a `DateFormat` Python enum** (single source of truth) → GraphQL enum via
  `Enum.from_enum`; the model validates against it. No server-side date renderer in V1 (clients
  render).
- **Read API shape** — three typed fields (effective/user/global), not one `scope`-parameterized
  query returning a `{key,value,source}` string list. Permission info is **not** in the payload — the
  frontend uses the generic `InfrahubPermissions` query.
- **Write API shape** — one mutation with a write-only `PreferenceWriteScope` (`USER`/`GLOBAL`);
  `EFFECTIVE` is unrepresentable as a write.
- **Clearing an override** — re-select the current value (the shared combobox's standard reset); no
  bespoke "Automatic" option. A field with no override shows the "Automatic (inherited)" placeholder.
- **Global scope gated on read and write** — `manage_global_preferences`, checked imperatively.
- **Orphaned rows** on account deletion are accepted for V1 (StandardNode has no cascade) and tracked
  in Jira.
- **Module layout** — shared GraphQL vocab in `graphql/types/preferences.py`; permission constant in
  `core/preferences/permissions.py`; package `__init__` is re-exports only.

## Migration & Rollout

- Purely additive: one new `StandardNode`, three custom read fields + one mutation, new frontend
  hooks + surfaces. **No graph migration, no schema change, no init seed.** The feature is unreleased,
  so there is no data to migrate.
- Existing date-rendering code keeps working until each call site migrates to `DateDisplay` (PR3),
  incrementally.
