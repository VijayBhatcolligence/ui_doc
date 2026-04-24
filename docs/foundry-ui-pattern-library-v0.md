# Foundry UI Pattern Library v0

**Status:** Revised v0 — patch round 3  
**Workstream:** UI/UX — Work Product 10.3  
**References:** Foundry UX Doctrine v0, Foundry Position Projection Schema v0, Foundry UI/UX Workstream Charter v1, Foundry2 Position-Centric Regenerative Software for SMBs v1  
**Purpose:** Define the authoritative specification for all UI patterns available to the generator when composing position apps.

This library is the vocabulary the compiler uses to select and compose screens and flows. Every `ViewSpec.pattern` value in the Position Projection Schema must resolve to a pattern defined here. The `PatternInputBinding` declared per view is validated against the required inputs defined in each pattern spec.

This library is a **compiler constraint**, not reference documentation. A pattern that does not appear here cannot be used by the generator — the generator has no authority to invent patterns outside this vocabulary. Every pattern name, approved variant, and invariant defined here is enforced at compile time against the Position Projection Schema. Changes to this library that add or rename patterns are breaking changes to the schema.

---

## 1. How to read this library

Each pattern is specified with sixteen properties. All sixteen are required before a pattern is considered fully specified.

| Property | What it answers |
|---|---|
| Purpose | What this pattern is for |
| When to use | The conditions under which this pattern is the right choice |
| When not to use | Failure modes and anti-patterns |
| Required inputs | Fields from `PatternInputBinding` that must be present for this pattern to render |
| Optional inputs | Fields from `PatternInputBinding` that enhance but are not required |
| Regions / slots | The named layout areas of this pattern |
| Slot-to-component mapping | Which semantic component fills each slot and what data source it binds to |
| Navigation rules | How the user moves in, around, and out of this pattern |
| State behavior | How the pattern behaves across its meaningful states |
| Pattern lifecycle | Which lifecycle states this pattern passes through (see §2.3 for the standard lifecycle) |
| Loading / empty / error | How the pattern handles each edge-condition state |
| Responsive behavior | How the pattern adapts across density modes and the declared device profile (`desktop_primary`, `mobile_primary`, `dual_mode`); per UX Doctrine D8 |
| Allowed semantic children | Which semantic components (10.5) may be placed in this pattern |
| Invariants | Rules that must survive regeneration and personalization |
| Approved variant surfaces | Permitted variations, their type, and their scope |
| Output contract | The IR node tree the generator produces when it instantiates this pattern |

---

## 2. Pattern input contract

Each pattern defines its required and optional inputs from `PatternInputBinding`. The compiler validates a view's `PatternInputBinding` against these contracts at compile time. A missing required input is a schema error. A missing optional input produces a graceful default behavior.

| Pattern | `entityTypeRef` | `listSourceDescription` | `filterSources` | `actionSlots` |
|---|---|---|---|---|
| CollectionView | Required | Required | Required (min 1) | primaryActionId required |
| RecordPage | Required | Not used | Not used | primaryActionId required for non-read-only states |
| OverviewMonitor | Optional | Not used | Optional | primaryActionId optional |
| SearchResults | Optional | Required | Optional | Not used |
| ExceptionResolutionView | Required | Not used | Not used | At least one resolutionActionId required |
| ApprovalReviewView | Required | Not used | Not used | primaryActionId = Approve; secondaryActionIds includes Reject |
| CreateEditFlow | Required | Not used | Not used | primaryActionId = Submit |
| ApprovalFlow | Required | Not used | Not used | primaryActionId = Submit request |
| ExceptionResolutionFlow | Required | Not used | Not used | primaryActionId = Resolve |
| AssistedSetupFlow | Optional | Not used | Not used | primaryActionId = Complete setup |

---

## 2.1 Pattern selection rules

The generator uses these deterministic rules to select the landing pattern and assign patterns to views. Rules are ordered by priority; the first matching rule wins.

**Landing view selection**

| Priority | Condition | Landing pattern |
|---|---|---|
| 1 | Position has `StateSummarySpec` AND (`AlertSpec` entries ≥ 1 OR `TaskSpec` entries ≥ 3) | OverviewMonitor |
| 2 | Position's primary role is approver (`ApprovalSpec.thisPositionRole = approver`) with no broader operational scope | ApprovalReviewView |
| 3 | Position works with one primary entity type via list operations | CollectionView |
| 4 | Position works with a specific known entity instance, never a collection | RecordPage |
| 5 | Position's primary daily entry point is cross-entity discovery by name or id | SearchResults |

**Non-landing view assignment**

| View requirement | Pattern assigned |
|---|---|
| Entity collection browseable by this position | CollectionView |
| Single entity detail and action surface | RecordPage |
| Multi-entity or cross-type search | SearchResults |
| Unresolved exceptions or alerts are the target | ExceptionResolutionView |
| Pending approvals awaiting this position's decision | ApprovalReviewView |

**Flow assignment**

| Trigger condition | Flow assigned |
|---|---|
| `ActionSpec` with `requiredInputs` spanning more than one section | CreateEditFlow |
| `ActionSpec` with `requiresApproval = true` and `thisPositionRole = initiator` | ApprovalFlow |
| `AlertSpec` resolution action that has its own `requiredInputs` | ExceptionResolutionFlow |
| First launch with `PersonalizationHooks` requiring user configuration | AssistedSetupFlow |

**Conflict resolution:** When multiple rules match, the higher-priority rule wins. Conflicts unresolvable by this table are a schema error; the schema author must resolve them before generation proceeds.

---

## 2.2 Pattern composition rules

These rules govern how patterns combine into a coherent position app. They are compiler-enforced at generation time. Violations are compile errors — the generator must not produce a position app that violates any rule here.

| Rule | Condition | Required composition |
|---|---|---|
| PC1 | `ApprovalSpec` with `thisPositionRole = approver` exists | App must include at least one `ApprovalReviewView` |
| PC2 | `AlertSpec` with `severity = warning` or `severity = critical` exists | App must include at least one `ExceptionResolutionView` or `ExceptionResolutionFlow` |
| PC3 | `CollectionView` exists for entity type E | A `RecordPage` for entity type E must be reachable via `EntityTypeRef.viewLink` |
| PC4 | `CreateEditFlow` exists | A triggering `ActionSpec` with at least one entry in `requiredInputs` must exist |
| PC5 | `ApprovalFlow` exists | A corresponding `ApprovalReviewView` must also exist in the same app |
| PC6 | `AssistedSetupFlow` exists | `PersonalizationHooks` with at least one required configuration surface must be present |
| PC7 | `OverviewMonitor` is the landing view | At least one navigable link to a `CollectionView` or `RecordPage` must be present in the monitor — a monitor cannot be a terminal view |

---

## 2.3 Standard pattern lifecycle

All patterns follow this state machine unless the pattern specification declares lifecycle overrides. Each pattern's "Pattern lifecycle" property references this standard and notes any additions or deviations.

| Lifecycle state | Trigger | What renders |
|---|---|---|
| `init` | Pattern bound to view; data fetch not yet started | Blank container; bindings validated against schema |
| `loading` | Data fetch in progress | Skeleton version of all primary regions |
| `ready` | Data loaded; user may interact | Full pattern content |
| `interacting` | User engages: selects, filters, searches, types | Incremental updates within the ready state |
| `updating` | Mutation submitted (action, form save, approval decision) | Loading indicator on active action; form or list locked |
| `error` | Load or mutation failed | Error state per region or full-page error with retry affordance |
| `retry` | User initiates retry from error state | Transitions to `loading` (load error) or `updating` (mutation error) |
| `terminal` | Flow completed, action completed, or navigation away triggered | Navigation; no further state managed in this pattern instance |

**Transition diagram:**  
`init → loading → ready ↔ interacting → updating → terminal (success)`  
`updating → error → retry → loading | updating`

**Flow pattern additions:**  
Flow patterns (F1–F4) supplement the standard lifecycle with two step-level states applied per step:

| Step lifecycle state | Trigger | What renders |
|---|---|---|
| `step_complete` | Step-level validation passed | Step marked complete in progress indicator; next step enabled |
| `step_error` | Step-level validation failed | Inline errors per field; section-level error summary; next step blocked |

---

## 2.4 Structural variant registry

This section is the authoritative registry for all structural variants across all patterns in this library. The Position Projection Schema §4.15 and composition rule C13 require that any `PersonalizationSurface` with `isStructuralVariant=true` validate against this registry. A structural variant not listed here cannot be used in a projection — its presence is a compile-time error.

**Scope constraint:** All structural variants are restricted to `scope=tenant` or `scope=role`. `scope=user` is not permitted for any structural variant in this registry. This enforces UX Doctrine §3.4: structural variants are approved product artifacts versioned by the compiler, not user personalization preferences.

| Pattern | Variant name | Description | Permitted scope |
|---|---|---|---|
| CollectionView (P1) | `list_style` | Semantic record list (default) vs compact grid | tenant, role |
| RecordPage (P2) | `sidebar_position` | Right sidebar (default) vs below content | tenant, role |
| OverviewMonitor (P3) | `summary_strip_layout` | Tile row (default) vs card grid | tenant, role |
| ExceptionResolutionView (P5) | `exception_list_mode` | Single exception (default) vs list with drill-in | tenant, role |
| CreateEditFlow (F1) | `form_layout` | Single-section (default for short forms) vs stepped multi-section | tenant, role |
| CreateEditFlow (F1) | `autosave` | Disabled (default) vs enabled (requires backend support) | tenant, role |
| ApprovalFlow (F2) | `approver_selection` | Pre-filled (default) vs user-selectable from approved set | tenant, role |
| AssistedSetupFlow (F4) | `ai_assist_mode` | Suggestions only (default) vs auto-fill with explicit review step | tenant, role |

**How this registry is used at compile time:**
- The compiler validates every `PersonalizationSurface` with `isStructuralVariant=true` against this table by pattern and `surfaceType` (variant name).
- The compiler validates that `PersonalizationSurface.scope` is `tenant` or `role` for all entries here — `scope=user` on any structural variant is a compile-time schema error (C13).
- Structural variants absent from this registry are rejected at compile time, not silently ignored.
- Adding a new structural variant to any pattern requires a corresponding entry in this registry before the pattern variant can be referenced in a projection.

---

## 3. Page patterns

---

### P1. CollectionView

**Purpose**  
Display a navigable, searchable, filterable list of records of a single entity type. The primary entry point for a position that works with a set of records as its daily operational surface.

**When to use**  
- The position needs to browse, search, and select from a set of records.
- The position takes actions on individual records drawn from a list.
- The entity collection can grow beyond one screen of results.
- Filtering and sorting are regular tasks for this position.

**When not to use**  
- The entity collection is always fewer than 10 records (use OverviewMonitor with quick-links).
- The records are events or unresolved exceptions (use ExceptionResolutionView).
- The primary task is reviewing records for approval (use ApprovalReviewView).
- Only one specific record is ever accessed (navigate directly to RecordPage).

**Required inputs**  
- `entityTypeRef`: the entity type being listed
- `listSourceDescription`: plain-language description of the data source
- `actionSlots.primaryActionId`: the main action (typically "Create new [entity]")
- At least one `filterSources` entry

**Optional inputs**  
- `actionSlots.secondaryActionIds`: additional non-primary actions
- `actionSlots.bulkActionIds`: bulk actions for selection state
- `actionSlots.overflowActionIds`: overflow actions beyond the budget
- `aiAssistHookIds`: AI hooks for search suggestion or anomaly surfacing

**Regions / slots**

| Region | Description | Required |
|---|---|---|
| Page header | Page title, primary action, secondary actions (budget-compliant) | Required |
| Search bar | Full-width, immediately visible; never hidden | Required |
| Filter area | Filter rail (sidebar) or filter bar (inline); common filters visible, advanced behind expansion | Required when `filterSources` present |
| Active filter indicator | Count of active filters + clear-all affordance | Required when filters are applied |
| List region | The record list; semantic record rows | Required |
| Bulk action bar | Replaces page header actions during selection state | Required when `bulkActionIds` present |
| Empty state | Communicative message + action path; shown for empty list and no-results states | Required |
| Pagination / load-more | At bottom of list region | Required when records can exceed one page |

**Record row structure**  
- Primary identifier (name or id — always first)
- Status badge
- 2–4 key facts from the entity's KeyFactsStrip (max 4 in list rows)
- Contextual inline action(s) per row: max 2; neither may be the primary page action
- Selection checkbox (shown in bulk-select state; hidden otherwise)

**Navigation rules**  
- Clicking a record row navigates to the RecordPage for that entity (EntityTypeRef.viewLink).
- Primary action (e.g., "Create new") launches a CreateEditFlow.
- Back navigation from a RecordPage returns to this view, restoring filter, sort, and scroll position.
- Filter changes update the list in place — no full page navigation.

**State behavior**

| State | Behavior |
|---|---|
| Default | List rendered with default sort and filter from DefaultViewHints |
| Search active | List filtered by search query; matched terms highlighted in rows |
| Filters active | Active filter count shown; clear-all affordance visible |
| Selection active | Row checkboxes visible; BulkActionBar replaces page header actions |
| Empty — no records exist | EmptyState: explains why empty; provides primary create action |
| Empty — no results for query | EmptyState: explains no match; provides clear-search and clear-filter actions |

**Loading / empty / error behavior**  
- **Loading:** Skeleton rows in list region; search bar is active and usable during load.
- **Empty (no records):** EmptyState with message and primary action path. Never a blank list.
- **Empty (no search/filter results):** EmptyState with query displayed, clear actions offered.
- **Error:** Inline StatusBanner with retry action. List region shows last successful state or skeleton. Error does not wipe existing filter state.

**Responsive behavior**  
- **Desktop (primary):** Filter rail visible in sidebar; full row layout with all key facts.
- **Compact density:** Tighter row height; reduced whitespace; action budget unchanged.
- **Mobile (`mobile_primary` or `dual_mode` positions):** Single-column simplified rows; search visible; filter behind panel toggle; bulk selection disabled; inline row actions reduced to 1.

**Allowed semantic children**  
`FilterRail`, `BulkActionBar`, `StatusBanner`, `EmptyState`, `ValidationSummary`, `ActionPanel`

**Invariants**  
- Search is always visible and never hidden behind a toggle (UX Doctrine P3, §7.1).
- One primary action in the page header (P1, A1).
- Action budget enforced: max 3 secondary visible; overflow for any total above 5 (A2, A3).
- Selection state replaces header actions with BulkActionBar — they do not coexist (A4).
- Destructive bulk actions are visually isolated from non-destructive bulk actions (A5).
- Row click always navigates to the record detail — this behavior is invariant (I4).
- Active filter count shown whenever filters are applied (§7.2).
- Empty states are always communicative with an action path (UX Doctrine 1.7).
- Telemetry hooks required on: row click, primary action, bulk action, search submit, filter apply, empty state view (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `list_style` | Structural variant | Semantic record list (default) vs compact grid; requires approved variant declaration |
| `filter_placement` | Emphasis variant | Rail sidebar (default) vs inline chip bar |
| `row_density` | Density mode | Default vs compact (D2) |
| `key_fact_count` | Emphasis variant | 2, 3, or 4 facts per row |

> **Structural variant scope:** The `list_style` variant is a structural variant and requires `scope=tenant` or `scope=role`. `scope=user` is not permitted (§2.4, schema C13).

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Page header | `ActionPanel` | `actionSlots` (primaryActionId, secondaryActionIds) |
| Search bar | `SearchBar` | user input → query string |
| Filter area | `FilterRail` (sidebar) or `FilterBar` (inline) | `filterSources[]` |
| Active filter indicator | `ActiveFilterIndicator` | applied filter state (runtime) |
| List region | `RecordList` | `listSourceDescription` (runtime-resolved entity collection) |
| Record row | `RecordRow` | `EntityTypeRef` fields + runtime entity data |
| Bulk action bar | `BulkActionBar` | `actionSlots.bulkActionIds` |
| Empty state | `EmptyState` | pattern state (no-records or no-results) |
| Pagination / load-more | `PaginationControl` | list metadata (page count, cursor) |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
CollectionView
├── PageHeader
│   ├── PageTitle                    (entityTypeRef.displayNameTemplate, runtime-resolved)
│   └── ActionPanel                  (primaryAction, secondaryActions — budget-validated)
├── SearchBar                        (query binding; always rendered)
├── FilterRail | FilterBar           (filterSources[]; rendered when filterSources present)
│   └── ActiveFilterIndicator        (conditional: when filters applied)
├── RecordList
│   └── RecordRow[]                  (one per record in result set)
│       ├── Identifier               (primary display field)
│       ├── StatusBadge              (status field)
│       ├── KeyFactsStrip            (2–4 facts from entityTypeRef)
│       └── InlineActions            (0–2 row-level actions; neither = page primary action)
├── BulkActionBar                    (conditional: selection state only; replaces PageHeader actions)
├── EmptyState                       (conditional: no records or no query results)
└── PaginationControl                (conditional: multi-page result set)
```

**Pattern lifecycle**

Standard (§2.3). The `interacting` state has two named sub-states: `search_active` (query entered, list filtered) and `filter_active` (filters applied, active filter indicator shown). Both are sub-states of `ready` and do not block further interaction.

---

### P2. RecordPage

**Purpose**  
Display the full detail of a single entity instance. The canonical surface for reading, editing, acting on, and understanding one specific record in full context.

**When to use**  
- The user has selected a specific record and needs full detail.
- Single-entity actions (edit, submit, approve, resolve) are the primary task.
- The entity has related entities, workflow state, or history that must be surfaced.

**When not to use**  
- Multiple records need side-by-side comparison (use CollectionView with expanded rows).
- The entity is a lightweight lookup item with no meaningful detail.
- The primary task is approving the entity (use ApprovalReviewView).
- The entity is an unresolved exception requiring focus resolution (use ExceptionResolutionView).

**Required inputs**  
- `entityTypeRef`: the entity type being shown
- `actionSlots.primaryActionId`: the primary action for the current record state (may be absent in fully read-only states)

**Optional inputs**  
- `actionSlots.secondaryActionIds`
- Related entity references (EntityTypeRef.viewLink)
- `aiAssistHookIds` for summarization, anomaly detection, or field suggestion
- Activity timeline (when entity participates in a workflow)
- Approval history (when entity participates in an ApprovalSpec)

**Regions / slots**

| Region | Description | Required |
|---|---|---|
| Record header | Entity name, type badge, status badge, KeyFactsStrip (max 8 facts), primary action | Required |
| Action group | Primary + secondary actions (budget-compliant) | Required |
| Content sections | Grouped form-like sections; each collapsible; ordered by LayoutHints | Required |
| Related records section | Links to related entities with EntityTypeRef.viewLink | Optional |
| Activity timeline | Chronological history of changes, actions, and approvals | Required when entity participates in a workflow |
| Contextual sidebar | TaskContextPanel, AI assist panel, collaborator information | Optional |
| Approval panel | Approval history and current approval state | Required when entity is in an ApprovalSpec |
| Attachment panel | Supporting documents or files | Optional |

**Section structure rules**  
- Each content section has a label, optional description, and collapsible body.
- Section ordering: core entity facts first → supporting detail → history last.
- Default expansion state follows `ViewSpec.layoutHints.groupedSections`.
- One section may be expanded by default; all others start collapsed unless the LayoutHints specify otherwise.

**Navigation rules**  
- Breadcrumb or back link returns to the originating CollectionView, restoring filter, sort, and scroll state.
- **Direct URL arrival (no originating view):** When a RecordPage is reached via bookmark, external link, or notification deep link, there is no prior navigation state. In this case, back navigation targets the default CollectionView for the entity type defined by `EntityTypeRef.entityType`. Filter and sort state are not restored — the CollectionView opens with its `DefaultViewHints` defaults.
- Related record links use EntityTypeRef.viewLink — inaccessible links render as plain labels (schema §4.3).
- Primary action may open a CreateEditFlow or trigger an inline modal depending on confirmationType.
- After a destructive action (delete), navigates to CollectionView.
- After edit-submit, stays on RecordPage showing updated state.

**State behavior — primary action per record state**

| Record state | Primary action | Notes |
|---|---|---|
| Draft | Submit / Save for review | State determined by ActionSpec state model |
| Pending approval | Withdraw (if initiator) | Observer state if not initiator |
| Under review | None (observer state) | Read-only; primary action absent |
| Approved | Proceed / Archive | Context-dependent |
| Rejected | Revise / Resubmit | Links to CreateEditFlow |
| Completed | View only / Archive | |

The exact state model and its primary action mapping come from the entity's TaskSpec or ActionSpec. This table is a reference template — the schema is authoritative.

**Loading / empty / error behavior**  
- **Loading:** Record header skeleton renders first; content sections fade in progressively.
- **Not found:** Full-page not-found EmptyState with back navigation.
- **Permission denied:** Full-page access-denied EmptyState with explanation and contact path.
- **Error:** Inline StatusBanner with retry; last-known section content preserved.

**Responsive behavior**  
- **Desktop (primary):** Two-column layout — content sections (main) + contextual sidebar (right).
- **Compact density:** Single-column; sidebar moves below content sections.
- **Mobile (`mobile_primary` or `dual_mode` positions):** Single-column; sidebar collapsed; primary action fixed at bottom of screen.

**Allowed semantic children**  
`RecordHeader`, `KeyFactsStrip`, `SmartFormSection`, `RelatedRecordsSection`, `ActivityTimeline`, `ApprovalPanel`, `AttachmentPanel`, `TaskContextPanel`, `ActionPanel`, `StatusBanner`, `CommentConversationPanel`, `EmptyState`

**Invariants**  
- Status badge is always visible in the record header (UX Doctrine P4, §2.4).
- State is shown before actions — status badge precedes action controls in DOM and visual order (P4).
- Primary action corresponds to the current record state; this mapping is invariant across regenerations (I2).
- Activity timeline visible on any entity participating in a business workflow (§2.1 traceability).
- Back navigation returns to originating view preserving list state (I1).
- Approval history visible on any record in an ApprovalSpec (§7.4, ApprovalSpec.approvalHistoryVisible).
- Telemetry hooks required on: primary action, secondary actions, section expand/collapse, related record navigation (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `sidebar_position` | Structural variant | Right sidebar (default) vs below content |
| `section_grouping` | LayoutHints-driven | From ViewSpec.layoutHints.groupedSections |
| `key_fact_count` | Emphasis variant | 4–8 facts in the header KeyFactsStrip |
| `timeline_placement` | Emphasis variant | Inline below sections (default) vs tab |
| `header_density` | Density mode | Default vs compact (D2) |

> **Structural variant scope:** The `sidebar_position` variant is a structural variant and requires `scope=tenant` or `scope=role`. `scope=user` is not permitted (§2.4, schema C13).

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Record header | `RecordHeader` | `EntityTypeRef` + runtime entity fields |
| Action group | `ActionPanel` | `actionSlots` (state-dependent primary + secondary) |
| Content sections | `SmartFormSection[]` | entity field groups, ordered by `layoutHints.groupedSections` |
| Related records section | `RelatedRecordsSection` | `EntityTypeRef.viewLink` references |
| Activity timeline | `ActivityTimeline` | workflow event history (runtime) |
| Contextual sidebar | `TaskContextPanel`, `AIAssistPanel` | `aiAssistHookIds`, `TaskSpec` data |
| Approval panel | `ApprovalPanel` | `ApprovalSpec` + approval event history |
| Attachment panel | `AttachmentPanel` | file references |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
RecordPage
├── RecordHeader
│   ├── EntityTypeBadge              (entityTypeRef.entityType)
│   ├── StatusBadge                  (current record status — rendered before actions)
│   ├── DisplayName                  (entityTypeRef.displayNameTemplate, runtime-resolved)
│   ├── KeyFactsStrip                (4–8 facts)
│   └── ActionPanel                  (primary + secondary actions — state-dependent)
├── ContentSections
│   └── SmartFormSection[]           (one per section group from layoutHints)
│       └── FieldGroup[]
├── RelatedRecordsSection            (conditional: related EntityTypeRef entries present)
├── ActivityTimeline                 (conditional: entity participates in a workflow)
├── ApprovalPanel                    (conditional: entity is in an ApprovalSpec)
├── AttachmentPanel                  (conditional: file references present)
└── [sidebar]
    ├── TaskContextPanel             (conditional: TaskSpec linked to this entity)
    └── AIAssistPanel                (conditional: aiAssistHookIds present)
```

**Pattern lifecycle**

Standard (§2.3). In edit mode, the pattern transitions `ready → interacting (fields changed) → updating (submit) → ready (updated record rendered)`. On submit error, `updating → error → retry → updating`. On delete success, `updating → terminal (navigate to CollectionView)`.

---

### P3. OverviewMonitor

**Purpose**  
Provide the position holder a high-level operational dashboard of their work domain — current state summaries, active alerts, and pending tasks — without requiring navigation to individual records first.

**When to use**  
- The position has monitoring responsibility across multiple entity types or workflow areas.
- Quick operational status at a glance is the most valuable daily starting point.
- Alerts, pending tasks, and summary metrics are the primary surface for this position.
- The position is required to have a `StateSummarySpec` in its projection (schema C8).

**When not to use**  
- The position works with a single entity type in a list (use CollectionView as landing).
- Deep record interaction is always the first task (use RecordPage or CollectionView as landing).
- The position is primarily an approver (use ApprovalReviewView as landing).
- No StateSummarySpec is defined for this position.

**Required inputs**  
- At least one `StateSummarySpec` (from the `stateSummaries` section of the projection)
- `PatternInputBinding` with no `listSourceDescription` (this pattern is not a list)

**Optional inputs**  
- `actionSlots.primaryActionId` (overview is read-first; primary action is optional here)
- Active `AlertSpec` entries (surfaced in the alert list region)
- Pinned summary ids from `DefaultViewHints.pinnedSummaryIds`
- `aiAssistHookIds` for anomaly detection or operational summarization

**Regions / slots**

| Region | Description | Required |
|---|---|---|
| Summary strip | Metric tiles (count, amount, ratio, status) ordered by `StateSummarySpec.displayOrder`; max 8 tiles | Required |
| Alert list | Active alerts by severity: Critical first, Warning, then Informational | Required when AlertSpec entries exist |
| Pending tasks section | Actionable tasks in priority order; linked to task detail | Optional |
| Quick-links section | Links to related CollectionViews or RecordPages for frequent navigation | Optional |
| AI insight panel | Anomaly summary or AI-generated operational summary | Optional |

**Summary tile structure**  
- Label (from StateSummarySpec.label)
- Value (runtime-resolved: count, amount, ratio, or status)
- Alert indicator (if AlertThresholdSpec is crossed)
- Click navigates to StateSummarySpec.linkedViewId

**Navigation rules**  
- Summary tile click navigates to the linked CollectionView or RecordPage.
- Alert list items navigate to the ExceptionResolutionView or relevant RecordPage.
- Pending task items navigate to the TaskContextPanel on the relevant RecordPage.
- Quick-links navigate to the target view directly.

**State behavior**

| State | Behavior |
|---|---|
| Normal | Summary tiles show current values; no alert list or green confirmation |
| Alerts active | Alert list visible; Critical alerts shown at top with visual prominence |
| No alerts | Alert list shows positive confirmation: "No active alerts — all operational" |
| Loading | Summary tile skeletons; alert list skeleton |

**Loading / empty / error behavior**  
- **Loading:** Skeleton tiles in summary strip; alert list skeleton rows.
- **Empty alerts:** Positive confirmation state — never a blank alert section.
- **Empty tasks:** Communicative empty state in pending tasks section.
- **Error on tile:** Inline error per failing tile; other tiles remain functional. Tile shows error state, not blank.

**Responsive behavior**  
- **Desktop (primary):** Summary strip across top; alert list and tasks in two columns below.
- **Compact density:** Summary strip wraps to two rows; single-column below.
- **Mobile (`mobile_primary` or `dual_mode` positions):** Summary strip (tiles only); alert list; tasks and quick-links hidden.

**Allowed semantic children**  
`StatusBanner`, `EmptyState`, `TaskContextPanel`, `ActionPanel`

**Invariants**  
- Critical alerts always appear before Warnings; Warnings before Informational (§7.5 severity ordering).
- Alert list is never blank when alerts exist — it cannot be collapsed by default.
- Empty alert state is a positive confirmation, not a blank section (UX Doctrine 1.7).
- Summary tiles always link to their related view when `StateSummarySpec.linkedViewId` is present.
- Summary strip is capped at 8 tiles. If the position projection defines more than 8 `StateSummarySpec` entries, the generator selects the 8 with lowest `displayOrder` values and emits a compile warning listing the omitted entries. This is a compile-time check.
- Telemetry hooks required on: tile click, alert item click, task item click, AI insight view (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `summary_strip_layout` | Structural variant | Tile row (default) vs card grid |
| `alert_list_position` | Emphasis variant | Below summary (default) vs sidebar |
| `ai_panel_visibility` | User preference | Hidden (default) vs visible |
| `pinned_summaries` | View-state personalization | Which tiles are pinned per user |

> **Structural variant scope:** The `summary_strip_layout` variant is a structural variant and requires `scope=tenant` or `scope=role`. `scope=user` is not permitted (§2.4, schema C13).

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Summary strip | `MetricTile[]` | `StateSummarySpec[]` (runtime-resolved values) |
| Alert list | `AlertList` | `AlertSpec[]` (severity-ordered at runtime) |
| Pending tasks section | `TaskList` | `TaskSpec[]` (priority-ordered) |
| Quick-links section | `QuickLinkGroup` | `EntityTypeRef.viewLink` or explicit view ids |
| AI insight panel | `AIInsightPanel` | `aiAssistHookIds` (runtime-generated summary) |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
OverviewMonitor
├── SummaryStrip
│   └── MetricTile[]                 (one per StateSummarySpec, ordered by displayOrder)
│       ├── Label                    (StateSummarySpec.label)
│       ├── Value                    (runtime: count | amount | ratio | status)
│       ├── AlertIndicator           (conditional: AlertThresholdSpec crossed)
│       └── NavigationLink           (StateSummarySpec.linkedViewId)
├── AlertList                        (conditional: AlertSpec entries exist)
│   └── AlertItem[]                  (Critical → Warning → Informational)
│       ├── SeverityBadge
│       ├── AlertLabel
│       └── NavigationLink           (to ExceptionResolutionView or RecordPage)
├── PendingTaskList                  (optional: TaskSpec entries present)
├── QuickLinkGroup                   (optional: configured quick-links present)
└── AIInsightPanel                   (optional: aiAssistHookIds present)
```

**Pattern lifecycle**

Standard (§2.3). No `updating` state — this pattern is read-first; navigation is the primary interaction. `interacting` is not applicable; tile clicks and alert clicks immediately transition to `terminal` (navigation to linked view).

---

### P4. SearchResults

**Purpose**  
Present the results of a cross-entity or cross-view search. The primary entry pattern when the user finds records by name or identifier rather than browsing a list.

**When to use**  
- Global or cross-entity search is a primary workflow entry point for this position.
- The position regularly locates records by name, id, or keyword.
- Search results may span more than one entity type.

**When not to use**  
- Search is scoped to a single entity type within a CollectionView — use CollectionView's built-in search instead.
- The result is always a single entity — navigate directly to RecordPage.
- Browsing and filtering is the primary discovery mode — use CollectionView.

**Required inputs**  
- `listSourceDescription`: which entity types are in scope for this search
- Search input (always visible, focused on view load)

**Optional inputs**  
- `entityTypeRef`: scope to a primary entity type
- `filterSources`: scope filter (entity type selector)
- `aiAssistHookIds`: query suggestion or auto-complete

**Regions / slots**

| Region | Description | Required |
|---|---|---|
| Search input | Prominent; focused on load; never hidden | Required |
| Scope selector | Narrows results to one entity type; shown when multi-entity search | Optional |
| Recent searches | Shown before a query is entered; user preference | Optional |
| Results list | Grouped by entity type (multi-entity) or flat (single-entity) | Required |
| Result row | Entity type label, display name, key facts (max 3), status badge | Required |
| Empty / no-results state | Message + suggestions | Required |

**Navigation rules**  
- Each result row navigates to the RecordPage for that entity.
- Scope selector narrows results in place — no full page navigation.
- Clearing the search input returns to the recent searches or blank state.

**State behavior**

| State | Behavior |
|---|---|
| Default (no query) | Recent searches shown if available; otherwise blank with input focused |
| Query active | Results rendered; matched terms highlighted in result rows |
| Scope filtered | Results filtered to selected entity type; scope indicator active |
| No results | EmptyState: query shown; suggestions offered; scope reset available |

**Loading / empty / error behavior**  
- **Loading:** Skeleton result rows while query executes.
- **No results:** EmptyState with the query displayed and suggestions.
- **Error:** Inline StatusBanner with retry; search input remains active.

**Responsive behavior**  
- **Desktop:** Two-column result layout for multi-entity; single column for single-entity.
- **Compact:** Single-column; scope selector collapses to dropdown.
- **Mobile:** Single-column; scope selector hidden by default.

**Allowed semantic children**  
`EmptyState`, `StatusBanner`

**Invariants**  
- Search input is always visible and focused on entry (UX Doctrine P3, §7.1).
- Matched terms are highlighted in every result row (§7.1).
- No-results state is always communicative — never a blank page.
- **Partial-access results:** When search results span entity types the user has mixed access to (per `permissionsHooks`), the pattern renders only the records the user is permitted to see. Inaccessible records are silently omitted — their count and existence are not disclosed. The no-results EmptyState is shown only when the accessible result set is empty; it must not reference the total result count.
- Telemetry hooks required on: search submit, result click, scope change, no-results view (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `result_grouping` | Emphasis variant | Grouped by entity type (default) vs flat unified list |
| `scope_selector_placement` | Emphasis variant | Above results (default) vs sidebar |
| `recent_searches` | User preference | Shown (default) vs hidden |

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Search input | `SearchBar` | user input → query string |
| Scope selector | `ScopeSelector` | `listSourceDescription` entity type list |
| Recent searches | `RecentSearchList` | user preference store (runtime) |
| Results list | `SearchResultList` | query response (grouped by entity type) |
| Result row | `SearchResultRow` | entity fields + `EntityTypeRef` |
| Empty / no-results state | `EmptyState` | query state + suggestions |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
SearchResults
├── SearchBar                        (focused on load; query binding; always rendered)
├── ScopeSelector                    (optional: multi-entity search scope)
├── RecentSearchList                 (conditional: pre-query state, user preference enabled)
├── SearchResultList
│   └── ResultGroup[]                (one per entity type when multi-entity)
│       └── SearchResultRow[]        (one per result)
│           ├── EntityTypeLabel
│           ├── DisplayName          (matched terms highlighted)
│           ├── KeyFactsStrip        (max 3 facts)
│           └── StatusBadge
└── EmptyState                       (conditional: no query results)
```

**Pattern lifecycle**

Standard (§2.3). No `updating` state — this pattern is read-only; result rows navigate to `RecordPage`. `interacting` = query is active or scope is being filtered. Results update in place within `interacting`.

---

### P5. ExceptionResolutionView

**Purpose**  
Surface one or more active exceptions in a focused context that allows the position holder to understand, act on, and resolve each exception without navigating away.

**When to use**  
- The position is responsible for resolving exceptions, flags, or errors on a specific entity.
- An AlertSpec with `severity=warning` or `severity=critical` has been triggered.
- The user arrives from an OverviewMonitor alert or a task linked to an exception.
- The exception requires attention but does not require a multi-step resolution flow.

**When not to use**  
- The exception is Informational — surface inline as a StatusBanner on the RecordPage instead.
- The primary task is a structured binary approval (use ApprovalReviewView).
- Resolution requires multi-step data entry (use ExceptionResolutionFlow instead).

**Required inputs**  
- `entityTypeRef`: the entity the exception is about
- `AlertSpec` with `severity=warning` or `severity=critical`
- At least one `resolutionActionId` in AlertSpec

**Optional inputs**  
- Context fields from the related entity (KeyFactsStrip)
- Activity timeline for the exception history
- Escalation target information (when `resolvableByThisPosition=false`)
- `aiAssistHookIds` for resolution suggestion

**Regions / slots**

| Region | Description | Required |
|---|---|---|
| Exception header | Severity badge, exception type label, affected entity reference | Required |
| Exception detail | Body template resolved with entity context; plain language | Required |
| Context section | Key facts from the affected entity (max 6 facts) | Required |
| Resolution actions | Primary resolution action + secondary actions (budget-compliant) | Required |
| Escalation section | Escalation target name + hand-off action; shown when `resolvableByThisPosition=false` | Conditionally required |
| Activity timeline | History of this exception | Optional |

**Navigation rules**  
- Resolution action may navigate to an ExceptionResolutionFlow if data entry is required.
- After resolution: exception dismissed in place; return to OverviewMonitor or originating view.
- Escalation hand-off: logs the hand-off, dismisses exception from current user's surface, navigates back.
- Back without resolution: returns to originating view; exception remains active.

**State behavior**

| Exception state | Behavior |
|---|---|
| Active | Full exception detail + resolution actions visible |
| In progress | Progress indicator; cancel available |
| Resolved | Exception dismissed in place; brief success state; return navigation |
| Escalated | Exception marked escalated; resolution actions replaced with "Awaiting [position]" |
| Unresolvable by this user | Escalation section is the only actionable area; resolution actions hidden |

**Loading / empty / error behavior**  
- **Loading:** Exception header skeleton; context section skeleton.
- **No exceptions:** This view must not be navigated to with zero active exceptions. Alert routing in OverviewMonitor prevents this.
- **Error loading exception detail:** Inline StatusBanner with retry; entity reference preserved so the user knows which record is affected.

**Responsive behavior**  
- **Desktop:** Single main column; context section inline; resolution actions prominent.
- **Compact:** Same structure; spacing tightened.
- **Mobile:** Single-column; escalation section collapses below resolution actions.

**Allowed semantic children**  
`StatusBanner`, `EmptyState`, `ActionPanel`, `ActivityTimeline`, `KeyFactsStrip`

**Invariants**  
- Severity badge always visible at top of exception header (UX Doctrine §7.5).
- Resolution is actionable from this view — user must not navigate away to act (§7.5).
- After resolution, exception is dismissed in place at all surfaces where it appeared (§7.5).
- Critical exceptions cannot be dismissed without a resolution or explicit escalation action (§7.5).
- When `resolvableByThisPosition=false`, escalation target is always named (§7.5).
- `generateAuditEntry=true` for Critical exceptions is enforced at compile time (schema C6).
- Telemetry hooks required on: resolution action, escalation action, back without resolution (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `context_section_depth` | Emphasis variant | 3 facts (minimal) vs 6 facts (standard) |
| `timeline_inclusion` | Emphasis variant | Hidden (default) vs inline |
| `exception_list_mode` | Structural variant | Single exception (default) vs list of exceptions with drill-in |

> **Structural variant scope:** The `exception_list_mode` variant is a structural variant and requires `scope=tenant` or `scope=role`. `scope=user` is not permitted (§2.4, schema C13).

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Exception header | `ExceptionHeader` | `AlertSpec` (severity, type) + `EntityTypeRef` |
| Exception detail | `ExceptionDetail` | `AlertSpec.bodyTemplate` (runtime-resolved with entity context) |
| Context section | `KeyFactsStrip` | affected entity runtime fields (max 6 facts) |
| Resolution actions | `ActionPanel` | `AlertSpec.resolutionActionIds` |
| Escalation section | `EscalationPanel` | `AlertSpec` escalation target info |
| Activity timeline | `ActivityTimeline` | exception event history (runtime) |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
ExceptionResolutionView
├── ExceptionHeader
│   ├── SeverityBadge                (Critical | Warning — always first in DOM)
│   ├── ExceptionTypeLabel           (AlertSpec.alertType)
│   └── EntityReference              (EntityTypeRef, linked via viewLink)
├── ExceptionDetail                  (bodyTemplate resolved with entity context)
├── KeyFactsStrip                    (context, max 6 facts from affected entity)
├── ActionPanel                      (resolution actions; at least one required)
├── EscalationPanel                  (conditional: resolvableByThisPosition = false)
│   ├── EscalationTargetLabel
│   └── EscalationHandOffAction
└── ActivityTimeline                 (optional: exception event history)
```

**Pattern lifecycle**

Standard (§2.3). `updating` = resolution action in progress (form locked, loading indicator on action). `terminal` = exception resolved (dismissed at all surfaces) or escalated (marked escalated, navigation back). Critical exceptions must reach `terminal` via resolution or escalation — they cannot reach `terminal` via back navigation without action.

---

### P6. ApprovalReviewView

**Purpose**  
Present a single approval request in full context so the approver can understand what is being approved, see the business consequence, and act (approve/reject) without navigating away.

**When to use**  
- The position is an approver in an ApprovalSpec (`thisPositionRole=approver`).
- The user arrived from a pending approval item in their work surface.
- A record requires structured binary action (approve/reject) before proceeding.
- Required by schema composition rule C7.

**When not to use**  
- The decision has more than two meaningful options (model as a DecisionSpec, surface in RecordPage).
- The review is informational only with no action (use RecordPage in read-only state).
- The position is the initiator, not the approver (use ApprovalFlow for submission).

**Required inputs**  
- `ApprovalSpec` for this approval type
- `entityTypeRef` for the subject entity
- `actionSlots.primaryActionId` = Approve action
- `actionSlots.secondaryActionIds` includes the Reject action

**Optional inputs**  
- Activity timeline / prior approval history
- Related entity context (supporting documents, notes)
- `aiAssistHookIds` for summarization or anomaly detection

**Regions / slots**

| Region | Description | Required |
|---|---|---|
| Approval header | Approval type, subject entity reference, initiator, submitted date | Required |
| Consequence section | Plain-language statement of what approval and rejection each mean | Required when `showConsequenceOnReview=true` |
| Subject entity context | Key facts from the subject entity (max 8 facts) | Required |
| Supporting detail | Related entities, attachments, or initiator notes | Optional |
| Approval history | Prior approval/rejection events for this entity | Required (ApprovalSpec.approvalHistoryVisible) |
| Decision actions | Approve (primary) + Reject (secondary, isolated per A5) | Required |
| Rejection reason input | Shown inline when Reject is selected, before final confirmation | Required (ApprovalSpec.rejectionRequiresReason) |

**Navigation rules**  
- Approve: submits, dismisses view, returns to pending approvals list or OverviewMonitor.
- Reject: requires reason input → inline confirmation → submits → returns.
- Back without acting: returns to pending approvals list; approval remains pending.

**State behavior**

| Approval state | Behavior |
|---|---|
| Pending | Full context; Approve and Reject both visible |
| Approved by this user | Read-only view; approval record shown |
| Rejected by this user | Read-only view; rejection reason shown |
| Awaiting other approvers | Read-only; shows remaining approvers |
| Expired / deadline passed | Status shown; no action available; escalation information shown |

**Loading / empty / error behavior**  
- **Loading:** Approval header skeleton; context section skeleton.
- **Not found:** Full-page EmptyState — approval request no longer exists.
- **Error:** Inline StatusBanner with retry; decision actions remain visible and functional.

**Responsive behavior**  
- **Desktop:** Main column (header + context + actions) + optional supporting detail sidebar.
- **Compact:** Single-column; supporting detail collapses below decision actions.
- **Mobile:** Single-column; Approve and Reject fixed at bottom; context scrollable.

**Allowed semantic children**  
`RecordHeader`, `KeyFactsStrip`, `ApprovalPanel`, `ActivityTimeline`, `AttachmentPanel`, `ActionPanel`, `StatusBanner`, `EmptyState`

**Invariants**  
- Approve and Reject are always both visible and clearly distinct (UX Doctrine §7.4).
- Reject action is visually isolated from Approve (A5).
- Rejection always requires a reason input (§7.4; ApprovalSpec.rejectionRequiresReason = true).
- Approval history is always visible (§7.4; ApprovalSpec.approvalHistoryVisible = true).
- Consequence of the decision is shown when `showConsequenceOnReview=true` (§7.4).
- Telemetry hooks required on: approve, reject, back without action, deadline-expired view (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `supporting_detail_placement` | Emphasis variant | Below context (default) vs sidebar |
| `approval_history_placement` | Emphasis variant | Inline (default) vs collapsible section |
| `consequence_section_prominence` | Emphasis variant | Standard (default) vs highlighted banner |

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Approval header | `ApprovalHeader` | `ApprovalSpec` + initiator info (runtime) |
| Consequence section | `ConsequencePanel` | `ApprovalSpec.showConsequenceOnReview` |
| Subject entity context | `KeyFactsStrip` | subject entity runtime fields (max 8 facts) |
| Supporting detail | `AttachmentPanel`, notes | initiator-submitted content (runtime) |
| Approval history | `ApprovalPanel` | approval event history (runtime) |
| Decision actions | `ActionPanel` | primaryAction = Approve; secondaryAction = Reject (isolated) |
| Rejection reason input | `RejectionReasonInput` | user input (required when Reject engaged) |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
ApprovalReviewView
├── ApprovalHeader
│   ├── ApprovalTypeLabel            (ApprovalSpec.approvalType)
│   ├── SubjectEntityReference       (EntityTypeRef, linked via viewLink)
│   ├── InitiatorLabel               (runtime: who submitted)
│   └── SubmittedDate                (runtime)
├── ConsequencePanel                 (conditional: showConsequenceOnReview = true)
├── KeyFactsStrip                    (subject entity context, max 8 facts)
├── AttachmentPanel + Notes          (optional: initiator-submitted supporting detail)
├── ApprovalPanel                    (approval history; always visible)
├── ActionPanel
│   ├── ApproveAction                (primary)
│   └── RejectAction                 (secondary; visually isolated from Approve)
└── RejectionReasonInput             (conditional: rendered when RejectAction engaged)
```

**Pattern lifecycle**

Standard (§2.3). `updating` = approval or rejection decision being submitted. `terminal` = decision submitted (navigate back to pending approvals list or OverviewMonitor). Read-only states (approved by others, awaiting other approvers, expired) do not reach `updating` — they are `ready` with no actionable transitions.

---

## 4. Flow patterns

Flow patterns are multi-step interactions. They differ from page patterns in that they guide the user through a sequence of steps with explicit forward/back/cancel navigation. All flow patterns must satisfy P6 (no orphaned flows) at every step.

---

### F1. CreateEditFlow

**Purpose**  
Guide the user through creating a new entity record or editing an existing one via a structured, guided form. Ensures completeness, field-level validation, and safe submission.

**When to use**  
- A new entity record needs to be created.
- An existing record needs multi-field editing that benefits from guided sections.
- The ActionSpec that initiates this flow has `requiredInputs` with more than one section.

**When not to use**  
- A single field can be changed inline on the RecordPage.
- The entity is created entirely by the system without user input.
- The creation requires structured approval decision-making (use ApprovalFlow).

**Required inputs**  
- `entityTypeRef`
- `requiredInputs` from the initiating ActionSpec (defines the form fields)
- `actionSlots.primaryActionId` = the submit action
- `ActionSpec.confirmationType` (none, inline, or full) — governs final submit behavior

**Optional inputs**  
- `LayoutHints.groupedSections` for section structure
- `aiAssistHookIds` for auto-complete, draft generation, or field suggestion
- `autosaveDraft` hint from DefaultViewHints (if backend supports)

**Steps / regions**

| Step / region | Description | Required |
|---|---|---|
| Flow header | Task name ("Create Invoice", "Edit Supplier"); progress indicator for multi-section flows; Cancel | Required |
| Section navigator | Section list or stepper; shown for multi-section flows only | Conditional |
| Active section | Form fields for the current section (SmartFormSection); one section active at a time | Required |
| Validation summary | Field-level inline errors + section-level summary before advance | Required |
| Draft save indicator | Unobtrusive; shown only when autosave is enabled | Conditional |
| Section navigation | Next (advance), Back (previous), Cancel | Required |
| Review step | Summary of all inputs before final submit; for high-stakes forms | Optional |
| Submit action | Primary action; visible only on the final step | Required |

**Form section rules (UX Doctrine §7.3)**  
- Required fields are clearly marked.
- Validation triggers on field blur — not only on submit.
- Inline errors appear adjacent to the affected field.
- One section is active (expanded) at a time in multi-section flows.
- Long flows use a review step before final submit.

**Navigation rules**  
- **Cancel** (any step): inline confirmation ("Discard changes?"); returns to originating view. If draft save is enabled, offer "Save draft" before discarding.
- **Back**: moves to previous section; preserves all entered data.
- **Next**: validates current section before advancing; blocks advance on validation error.
- **Submit**: triggers `confirmationType` behavior from ActionSpec; on success, navigates per `postActionBehavior`.

**State behavior**

| Flow state | Behavior |
|---|---|
| New (create mode) | All fields empty; required fields marked |
| Edit (existing record) | Fields pre-populated; changed fields visually distinguished |
| Validation error | Inline error per field; section-level error summary; Next/Submit blocked |
| Draft saved | Draft save indicator updates; user can leave and return |
| Submitting | Submit button in loading state; form locked against input |
| Success | Navigate per `ActionSpec.postActionBehavior` |
| Submit error | StatusBanner with error; form unlocked; user may retry |

**Loading / empty / error behavior**  
- **Loading (edit mode):** SmartFormSection skeleton; fields progressively render as data loads.
- **Load error:** Full StatusBanner with retry; form not rendered until data is available.
- **Partial submit error:** Error banner specifying which fields or step failed.

**Responsive behavior**  
- **Desktop (primary):** Full form layout; section navigator in sidebar for multi-section.
- **Compact:** Single-column form; section navigator as top stepper.
- **Mobile:** Single-column; one field at a time for long forms; Submit fixed at bottom.

**Allowed semantic children**  
`SmartFormSection`, `ValidationSummary`, `StatusBanner`, `ActionPanel`, `AttachmentPanel`

**Invariants**  
- Cancel is always available at every step with draft/discard warning (P6).
- Validation errors appear inline at field level, not only at form submit (§7.3).
- Submit appears only when the user has reached the final step or review step.
- Progress indicator visible in all multi-section flows.
- AI-generated field fills are visually distinguishable from user-entered values (P11).
- AI field fills require explicit user acceptance before applying (P11, P12).
- Telemetry hooks required on: flow start, section advance, validation error, submit attempt, flow completion, flow abandonment (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `form_layout` | Structural variant | Single-section (default for short forms) vs stepped multi-section |
| `review_step` | Emphasis variant | Omit (default for simple forms) vs include (for high-stakes) |
| `autosave` | Structural variant | Disabled (default) vs enabled (requires backend support) |
| `section_navigator_style` | Emphasis variant | Sidebar list (default) vs top stepper |

> **Structural variant scope:** The `form_layout` and `autosave` variants are structural variants and require `scope=tenant` or `scope=role`. `scope=user` is not permitted for either (§2.4, schema C13).

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Flow header | `FlowHeader` | ActionSpec name; step count from `LayoutHints.groupedSections` |
| Section navigator | `StepNavigator` | `LayoutHints.groupedSections` (step labels and order) |
| Active section | `SmartFormSection` | `ActionSpec.requiredInputs` (fields for current section) |
| Validation summary | `ValidationSummary` | field-level validation state (runtime) |
| Draft save indicator | `DraftSaveIndicator` | autosave status (runtime; conditional) |
| Section navigation | `FlowNavigationBar` | step state (current step index, total steps) |
| Review step | `ReviewSummary` | all section inputs accumulated |
| Submit action | `ActionPanel` | `actionSlots.primaryActionId` (final step only) |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
CreateEditFlow
├── FlowHeader
│   ├── FlowTitle                    (ActionSpec name)
│   ├── ProgressIndicator            (multi-section only; current step / total)
│   └── CancelAction                 (available at every step)
├── StepNavigator                    (multi-section only; sidebar list or top stepper)
├── ActiveSection
│   └── SmartFormSection             (fields for current section from requiredInputs)
│       └── FieldGroup[]
│           └── InputField[]         (type from InputFieldSpec; AI fills labeled if present)
├── ValidationSummary                (conditional: validation errors present)
├── DraftSaveIndicator               (conditional: autosave enabled)
├── FlowNavigationBar
│   ├── BackAction                   (disabled on first step)
│   ├── NextAction                   (non-final steps; validates before advancing)
│   └── SubmitAction                 (final step only)
└── ReviewSummary                    (optional: pre-submit review step when configured)
```

**Pattern lifecycle**

Standard (§2.3) plus step lifecycle (§2.3 flow additions). `step_error` blocks `NextAction` until validation passes. `updating` = form submission in progress (form locked). `terminal` = submit succeeded (navigate per `ActionSpec.postActionBehavior`) or cancel confirmed (navigate to originating view).

---

### F2. ApprovalFlow

**Purpose**  
Guide the initiator through submitting an approval request — confirming what is being submitted, specifying approvers if configurable, adding context, and confirming submission.

**When to use**  
- An ActionSpec with `requiresApproval=true` has been initiated.
- The position is an initiator in an ApprovalSpec (`thisPositionRole=initiator`).
- A business event requires structured approval before the system can proceed.

**When not to use**  
- The position is the approver, not the initiator (use ApprovalReviewView).
- The action does not require approval (submit directly via CreateEditFlow or inline action).

**Required inputs**  
- `ApprovalSpec` with `thisPositionRole=initiator`
- `entityTypeRef` for the subject entity
- `requiredApproverPositionIds` (from ApprovalSpec; pre-filled or user-selectable)
- `actionSlots.primaryActionId` = submit the approval request

**Optional inputs**  
- Supporting notes or attachments
- Configurable deadline (`ApprovalSpec.approvalDeadlineHours`)

**Steps**

| Step | Description | Required |
|---|---|---|
| Context confirmation | Shows what is being submitted for approval and the business consequence | Required |
| Approver confirmation | Confirms required approvers; selectable when configured as variant | Required |
| Supporting context | Optional notes, attachments, or context fields for the approver | Optional |
| Review and submit | Summary of the request before sending | Required |

**Navigation rules**  
- Cancel: returns to RecordPage or originating view; no approval submitted.
- Back: moves to previous step; data preserved.
- Submit: sends approval request; navigates to RecordPage showing pending approval status.

**State behavior**

| State | Behavior |
|---|---|
| Submitting | Submit button loading; form locked |
| Success | Navigate to RecordPage with pending approval status visible |
| Error | StatusBanner; form unlocked; retry available |

**Responsive behavior**
- **Desktop (`desktop_primary`):** Full multi-step layout; context confirmation and approver list displayed with ample context; progress indicator in flow header.
- **Compact:** Single-column; section spacing reduced; progress indicator remains visible.
- **Mobile (`mobile_primary` or `dual_mode` positions):** Single-column; one step per screen; approver list scrollable; Submit request action fixed at bottom of final step.

**Invariants**  
- The consequence of the approval request is shown before submit (§7.4).
- Required approvers are confirmed before submission.
- Cancel is available at every step (P6).
- After submit, the initiator can see pending approval status on the RecordPage (ApprovalSpec.approvalHistoryVisible).
- Telemetry hooks required on: flow start, submit, cancellation (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `approver_selection` | Structural variant | Pre-filled (default) vs user-selectable from approved set |
| `supporting_context_step` | Emphasis variant | Optional step shown (default) vs hidden |

> **Structural variant scope:** The `approver_selection` variant is a structural variant and requires `scope=tenant` or `scope=role`. `scope=user` is not permitted (§2.4, schema C13).

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Context confirmation step | `ConsequencePanel` + `SmartFormSection` | `ApprovalSpec`, subject entity runtime fields |
| Approver confirmation step | `ApproverList` | `ApprovalSpec.requiredApproverPositionIds` |
| Supporting context step | `SmartFormSection` (notes, attachments) | user input |
| Review and submit step | `ReviewSummary` | all prior step inputs accumulated |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
ApprovalFlow
├── FlowHeader
│   ├── FlowTitle                    (e.g., "Submit for approval")
│   ├── ProgressIndicator
│   └── CancelAction                 (available at every step)
├── Step: Context confirmation
│   └── ConsequencePanel             (what is being submitted; approval/rejection consequences)
├── Step: Approver confirmation
│   └── ApproverList                 (required approvers; selectable when variant configured)
├── Step: Supporting context          (optional)
│   └── SmartFormSection             (notes, attachments)
├── Step: Review and submit
│   ├── ReviewSummary                (all prior step inputs)
│   └── ActionPanel                  (SubmitAction = primary)
└── FlowNavigationBar                (Back + Next; Submit on final step)
```

**Pattern lifecycle**

Standard (§2.3) plus step lifecycle (§2.3 flow additions). `updating` = approval request being submitted. `terminal` = approval request submitted (navigate to RecordPage showing pending approval status) or cancel (navigate to originating RecordPage; no request submitted).

---

### F3. ExceptionResolutionFlow

**Purpose**  
Guide the position holder through resolving a complex exception that requires multi-step data entry or decision-making beyond a single action button click.

**When to use**  
- An exception resolution requires data input before the system can close the exception.
- Resolution involves more than one step.
- The exception severity is Critical and resolution requires `confirmationType=full`.
- The ExceptionResolutionView's inline actions are insufficient for this exception type.

**When not to use**  
- The exception can be resolved with a single action button (use ExceptionResolutionView inline actions).
- The exception is Informational — no resolution flow needed.
- The resolution is delegated entirely to another position (use escalation in ExceptionResolutionView).

**Required inputs**  
- `AlertSpec` with at least one `resolutionActionId` that has `requiredInputs`
- `entityTypeRef` for the affected entity
- `actionSlots.primaryActionId` = Resolve action

**Optional inputs**  
- Impact preview data (recommended for Critical exceptions)
- `aiAssistHookIds` for resolution suggestion

**Steps**

| Step | Description | Required |
|---|---|---|
| Exception context | What happened, which entity is affected, severity | Required |
| Resolution input | Data entry or choice required to resolve | Required |
| Impact preview | Shows what the resolution will change | Required for `severity=critical`; optional for warning |
| Confirm and resolve | Explicit confirmation with consequence statement | Required |

**Navigation rules**  
- Cancel/Exit: available at every step; returns to ExceptionResolutionView or OverviewMonitor; exception remains active.
- Back: preserves entered data.
- Resolve: on success, exception dismissed at all surfaces where it appeared; navigate back.

**State behavior**

| State | Behavior |
|---|---|
| Resolving | Loading state on final step; form locked |
| Success | Exception dismissed; brief success state; return navigation |
| Error | StatusBanner; form unlocked; retry |

**Responsive behavior**
- **Desktop (`desktop_primary`):** Full step layout; exception context visible alongside resolution input in sequence; ImpactPreviewPanel rendered inline.
- **Compact:** Single-column; tightened spacing.
- **Mobile (`mobile_primary` or `dual_mode` positions):** Single-column; one step per screen; ImpactPreviewPanel scrollable; Resolve action fixed at bottom of final step.

**Invariants**  
- Exit available at every step (P6).
- Critical exceptions require `confirmationType=full` on the final step (§7.5).
- After resolution, exception dismissed at all surfaces where it appeared (§7.5).
- Audit entry generated for Critical exceptions (schema C6, AlertSpec.generateAuditEntry).
- Impact preview is required for `severity=critical` exceptions — it may not be omitted.
- Telemetry hooks required on: flow start, resolve, cancellation at each step (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `impact_preview_step` | Emphasis variant | Omit (for Warning) vs include (required for Critical) |
| `resolution_input_layout` | Emphasis variant | Single-section (default) vs multi-section (for complex resolutions) |

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Exception context step | `ExceptionHeader` + `KeyFactsStrip` | `AlertSpec`, affected `EntityTypeRef` runtime fields |
| Resolution input step | `SmartFormSection` | `AlertSpec.resolutionActionId.requiredInputs` |
| Impact preview step | `ImpactPreviewPanel` | computed from resolution inputs + entity state (runtime) |
| Confirm and resolve step | `ReviewSummary` + `ActionPanel` | resolution summary; ResolveAction = primary |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
ExceptionResolutionFlow
├── FlowHeader
│   ├── FlowTitle                    (e.g., "Resolve: [AlertSpec.alertType]")
│   ├── ProgressIndicator
│   └── ExitAction                   (available at every step; exception remains active on exit)
├── Step: Exception context
│   ├── ExceptionHeader              (SeverityBadge, ExceptionTypeLabel, EntityReference)
│   └── KeyFactsStrip                (affected entity, max 6 facts)
├── Step: Resolution input
│   └── SmartFormSection             (resolution data fields from resolutionActionId.requiredInputs)
├── Step: Impact preview              (required for severity = critical; optional for warning)
│   └── ImpactPreviewPanel           (what will change as a result of this resolution)
├── Step: Confirm and resolve
│   ├── ReviewSummary
│   ├── ConsequenceStatement         (plain-language summary of resolution effect)
│   └── ActionPanel                  (ResolveAction = primary)
└── FlowNavigationBar
```

**Pattern lifecycle**

Standard (§2.3) plus step lifecycle (§2.3 flow additions). `updating` = resolution being submitted (form locked). `terminal` = exception resolved (dismissed at all surfaces; audit entry generated for Critical; navigate back) or exit without resolution (navigate back; exception remains active).

---

### F4. AssistedSetupFlow

**Purpose**  
Guide a position holder through configuring a new position app, setting personalization defaults, or completing an onboarding sequence — with AI assistance available throughout to suggest initial values.

**When to use**  
- A new position app is being set up for the first time.
- The position holder needs to configure their personalization defaults before the app is operational.
- An onboarding sequence is required to make the position app useful on first launch.

**When not to use**  
- Setup is fully automated with no user configuration required.
- Configuration involves a single field or choice (use a form or inline action).
- The position app is already configured — this flow is for initial setup only.

**Required inputs**  
- `actionSlots.primaryActionId` = Complete setup
- At least one configuration step (from `PersonalizationSurface` entries in the projection)

**Optional inputs**  
- `aiAssistHookIds` for pre-filling configuration suggestions

**Steps**

| Step | Description | Required |
|---|---|---|
| Welcome | Explains what is being set up and why; sets expectations | Required |
| Configuration steps | One step per major configuration area from PersonalizationHooks | Required |
| AI assist steps | AI pre-fills suggestions for user review and acceptance | Optional (present when `aiAssistHookIds` exist) |
| Review | Summary of all configuration choices before finalising | Required |
| Completion | Confirmation and launch to `workSurface.navigation.defaultLandingViewId` | Required |

**Navigation rules**  
- Skip: available for optional configuration steps; skipped steps use the role or global default.
- Back: moves to previous step; all entries preserved.
- Cancel: not available after the welcome step for mandatory configurations; available for optional steps.
- Complete: navigates to the position app's `defaultLandingViewId`.
- **Mid-flow abandonment and resume:** For mandatory setup flows (Cancel not available), if the user closes the browser, loses connection, or is otherwise interrupted, partial configuration state is preserved as a draft tied to the position and user. On next launch, if setup is incomplete, the app re-enters the AssistedSetupFlow and resumes from the last completed step. Completed step state is shown in the progress indicator; already-entered values are pre-populated. The draft is cleared only on successful completion.

**State behavior**

| State | Behavior |
|---|---|
| AI suggestion shown | Suggestion is visually labeled; Accept or Edit options shown; not applied until accepted |
| Step complete | Step marked complete in progress indicator; advance available |
| Review | All configuration choices visible; each editable via Back |
| Completing | Submit loading; configuration persisted |
| Done | Navigate to landing view |

**Responsive behavior**
- **Desktop (`desktop_primary`):** Full step layout; StepNavigator as sidebar list for multi-configuration steps; AI suggestions displayed in a side panel alongside the active configuration field.
- **Compact:** Single-column; StepNavigator as top stepper.
- **Mobile (`mobile_primary` or `dual_mode` positions):** Single-column; one configuration step per screen; AI suggestions shown inline below the relevant field; Complete action fixed at bottom of final step.

**Invariants**  
- AI suggestions are clearly labeled and require explicit acceptance before applying (P11, P12).
- Skip is available for optional steps (P6).
- Completion navigates to `defaultLandingViewId` — this is invariant.
- All configured defaults can be changed later via personalization hooks — this must be communicated at the completion step.
- Telemetry hooks required on: flow start, each step completion, AI suggestion accepted/rejected, flow completion, flow abandonment (P10).
- When a capability used by this pattern is listed in `completeness.deferredCapabilities`, the generator renders a provisional frame in the affected region — never blank, empty, or absent (XP8, schema C12).

**Approved variant surfaces**

| Variant | Type | Description |
|---|---|---|
| `ai_assist_mode` | Structural variant | Suggestions only (default) vs auto-fill with explicit review step |
| `skip_availability` | Emphasis variant | Per-step skip (default) vs no-skip for mandatory configuration |
| `welcome_step` | Emphasis variant | Full welcome (default) vs minimal header only |

> **Structural variant scope:** The `ai_assist_mode` variant is a structural variant and requires `scope=tenant` or `scope=role`. `scope=user` is not permitted (§2.4, schema C13).

**Slot-to-component mapping**

| Slot | Component | Data source |
|---|---|---|
| Welcome step | `WelcomePanel` | setup flow metadata (flow name, purpose statement) |
| Configuration steps | `SmartFormSection` | `PersonalizationHooks.PersonalizationSurface.configurationSchema` |
| AI assist steps | `AIAssistPanel` + `SmartFormSection` | `aiAssistHookIds` (runtime suggestions; labeled) |
| Review step | `ReviewSummary` | all configured values accumulated |
| Completion step | `CompletionPanel` | setup result + `workSurface.navigation.defaultLandingViewId` |

**Output contract**

The generator instantiates this pattern as the following IR node tree:

```
AssistedSetupFlow
├── FlowHeader
│   ├── FlowTitle                    (e.g., "Set up [Position Name]")
│   └── ProgressIndicator
├── Step: Welcome
│   └── WelcomePanel                 (purpose statement; expectations; what will be configured)
├── Step[]: Configuration            (one per PersonalizationSurface requiring setup)
│   ├── SmartFormSection             (config fields from configurationSchema)
│   └── AIAssistPanel                (optional: labeled AI suggestions; Accept / Edit required)
├── Step: AI review                  (conditional: ai_assist_mode = auto-fill)
│   └── ReviewSummary                (AI-suggested values; each Accept/Edit)
├── Step: Review
│   └── ReviewSummary                (all configured values; each editable via Back)
├── Step: Completion
│   └── CompletionPanel              (confirmation; note that all defaults are editable later)
└── FlowNavigationBar                (Back + Next; Skip for optional steps; Complete on final step)
```

**Pattern lifecycle**

Standard (§2.3) plus step lifecycle (§2.3 flow additions). No `updating` between steps — per-step inputs are saved as draft only, not committed. `updating` = final Complete submission (configuration persisted to backend). `terminal` = setup complete (navigate to `defaultLandingViewId`). Skip transitions directly to `step_complete` for optional steps using role or global defaults. On re-entry after interruption, the flow resumes from `step_complete` at the last finished step; prior steps are in `step_complete` state and prior inputs are restored from draft.

---

## 5. Cross-pattern rules

These rules apply to all patterns without exception.

**XP1 — Action budget applies everywhere**  
Every pattern must comply with the action budget from UX Doctrine §6: max 1 primary, max 3 secondary visible, total ≤ 5 before overflow. This is enforced at compile time via `ActionSlotBinding` (schema C11).

**XP2 — Empty states are always communicative**  
Every region that can be empty must have an EmptyState component with a message explaining why it is empty and a direct action path where applicable (UX Doctrine 1.7).

**XP3 — Accessibility is a floor for every pattern**  
Every pattern, every region, every semantic child must meet WCAG 2.1 AA. Focus management, keyboard navigation, and aria annotations are required. Accessibility must survive regeneration (UX Doctrine P9, I6).

**XP4 — Telemetry hooks are part of every pattern contract**  
All primary actions, flow completions, flow abandonments, error encounters, and empty-state views carry telemetry hooks. Hook keys are invariant across regenerations (UX Doctrine §2.7, P10).

**XP5 — AI actions are always distinguishable**  
In any pattern where AI assist hooks are active, AI-generated content is visually labeled and requires explicit user acceptance before it affects business state (P11, P12).

**XP6 — State before action in every pattern**  
In page patterns, state (status badge, record state, exception severity) is always rendered before action controls in both DOM order and visual order (P4).

**XP7 — Pattern type is invariant across regenerations**  
A view assigned `CollectionView` must remain `CollectionView` across regenerations. A silent pattern change is a breaking change (UX Doctrine I4, schema §6.2).

**XP8 — Provisional frames for deferred capabilities**  
When the `completeness.deferredCapabilities` list in the position's projection includes an extension section name (e.g., `alerts`, `tasks`, `approvals`, `decisions`, `collaborators`, `stateSummaries`, `aiAssistHooks`), or when an extension section is absent from the projection without being declared in `outOfScopeCapabilities`, the generator must render the corresponding UI region as a **provisional frame** — not as absent, empty, hidden, or zero. A provisional frame must satisfy all four of the following requirements:

1. **Identify** the deferred capability by name in plain language (e.g., "Alerts are not yet set up for this position").
2. **Explain** what will be available once the capability is fully modeled (e.g., "When configured, alerts will surface here for active exceptions requiring your attention").
3. **Surface a next-best action** where applicable — navigation to a related available view, a contact path, or a fallback action. If no next-best action exists, the frame still renders with identification and explanation.
4. **Never** render as blank, zero-count, an empty list with no label, or a fully hidden region. Unknown must not masquerade as empty or absent.

The `DeferredCapabilityPlaceholder` component (defined in Semantic Component Library 10.5) is the standard component for provisional frames. Generator behavior for deferred capabilities is enforced by schema C12. Extension sections declared in `completeness.outOfScopeCapabilities` render as **absent or hidden** — they are confirmed not applicable and must not render as provisional.

Each pattern's Invariants section references this rule for the capabilities relevant to that pattern.

---

## 6. Pattern evaluation checklist

Use this checklist to evaluate any pattern specification, component placement, or regenerated view against this library and the UX Doctrine.

| Check | Doctrine / Schema ref |
|---|---|
| Is the pattern type the right choice for this work surface? | §3, §4 (when to use / when not to use) |
| Are all required PatternInputBinding fields present? | §2 Pattern input contract |
| Is the action budget compliant? (1 primary, ≤3 secondary, ≤5 total) | XP1, UX Doctrine §6, schema C11 |
| Is search immediately visible where required? | P3, §7.1 |
| Is state shown before action? | P4, XP6 |
| Are empty states communicative with an action path? | XP2, 1.7 |
| Are all error states handled (loading, empty, error)? | Per-pattern spec |
| Is the pattern accessible (WCAG 2.1 AA)? | XP3, P9 |
| Are telemetry hooks declared for all required events? | XP4, P10 |
| Are AI actions labeled, reversible, and confirmation-gated? | XP5, P11, P12 |
| Is the pattern type invariant (not changed from prior regeneration)? | XP7, I4 |
| Are invariants documented and enforced for this view? | Per-pattern invariants |
| Are all variant surfaces declared as approved variants? | Per-pattern approved variants |
| Are deferred capabilities rendered as provisional frames — not empty, absent, or zero? | XP8, schema C12, UX Doctrine §2.8 |
| Are all structural variants registered in §2.4 (not invented outside the approved registry)? | §2.4, schema C13, UX Doctrine §3.4, P13 |
| Are structural variant surfaces scoped to tenant or role — never user? | §2.4, schema C13, UX Doctrine §3.4 |

---

## 7. Open seams

| Seam | Connects to | Status |
|---|---|---|
| Allowed semantic children (component names) | Semantic Component Library v0 (10.5) | Component names declared here as strings; full specs defined in 10.5 |
| `PatternInputBinding` validation per pattern | Position Projection Schema v0 (10.2) | This library defines the authoritative input contracts; schema declares per-view bindings |
| Approved variant names in ViewSpec.patternVariant | This library (variant surfaces per pattern) | Variant vocabulary defined here; referenced as string ids in the schema |
| AI assist hook types per pattern | AI service layer | Hook trigger contexts declared here; AI service binding defined separately |
| Pattern-to-IR mapping | Compiler intermediate representation | Pattern names and slot vocabulary defined here; IR targeting is a compiler workstream concern |
| Prototype workflow validation | Golden Workflow Prototypes (10.6) | This library is validated against real workflows in 10.6 |
| Provisional frame behavior for deferred capabilities | Position Projection Schema v0 §4.16 (ProjectionCompletenessSpec) | `completeness.deferredCapabilities` and `outOfScopeCapabilities` determine whether absent extension sections render as provisional frames (XP8, C12) or as absent/hidden. This library defines the provisional frame contract (XP8); the schema declares the completeness state and enforces C11. |
| Structural variant scope and registry | Position Projection Schema v0 §4.15 (PersonalizationHooks) | `PersonalizationSurface.isStructuralVariant=true` requires validation against §2.4 of this library. The schema enforces C13 (scope=tenant or scope=role); this library is the authoritative registry of which structural variants are approved and for which patterns. |

---

## 8. Milestone exit criteria — self-assessment

| Exit criterion | Status | Evidence |
|---|---|---|
| All pattern names match `ViewSpec.pattern` vocabulary in the Position Projection Schema | Met | P1–P6 and F1–F4 correspond exactly to the schema's closed PatternId vocabulary |
| `PatternInputBinding` required/optional contracts defined for all 10 patterns | Met | §2 Pattern input contract table; each pattern's Required inputs and Optional inputs |
| Pattern selection is deterministic | Met | §2.1 priority-ordered selection rules; conflict resolution rule defined |
| Composition rules enforced at compile time | Met | §2.2 PC1–PC7; violations are compile errors |
| Standard pattern lifecycle defined and applied | Met | §2.3 standard lifecycle; each pattern references or notes deviations |
| Cross-pattern rules defined and enforced | Met | §5 XP1–XP8; all patterns reference applicable rules |
| Provisional rendering contract defined for deferred capabilities | Met (patch round 1) | XP8 added; all 10 patterns' Invariants sections reference XP8; aligns with schema C12 and UX Doctrine §2.8 |
| Structural variant registry present and scope-constrained | Met (patch round 1) | §2.4 Structural Variant Registry added with 8 entries; scope=tenant or scope=role enforced; aligns with schema C13 and UX Doctrine §3.4 |
| Structural variant scope notes added per pattern | Met (patch round 1) | All 7 patterns with structural variants have scope constraint notes on their variant tables |
| Library declared as compiler constraint, not reference documentation | Met (patch round 1) | Generation-constraint opener added to header |
| Open seams to dependent work products identified | Met (patch round 1) | §7 open seams; two seam entries added connecting to schema §4.15 and §4.16 |
| Stale doctrine cross-references resolved | Met (patch round 1) | XP4 I7 reference corrected to UX Doctrine §2.7, P10 |
| Schema constraint rule numbers aligned with Final v0 schema | Met (patch round 2) | New C4 insertion shifted C5→C6, C6→C7, C7→C8, C10→C11, C11→C12, C12→C13; `DefaultViewHints.defaultLandingViewId` corrected to `workSurface.navigation.defaultLandingViewId` in F4 slot mapping; PatternId vocabulary language aligned |
| Device-profile-aware responsive behavior declared for all 10 patterns | Met (patch round 3) | P1/P2/P3 "Mobile (assisted only)" label corrected to "`mobile_primary` or `dual_mode` positions" per D6/D9; F2/F3/F4 missing "Responsive behavior" sections added per D8; §1 property description updated to reference device profile |

---

*This document is version 0. It is the first executable version sufficient to drive UI pattern selection and position app generation. It is expected to be validated against the Golden Workflow Prototypes (work product 10.6) and refined once the Semantic Component Library (10.5) is drafted.*
