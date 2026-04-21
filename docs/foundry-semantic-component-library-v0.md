# Foundry Semantic Component Library v0

**Status:** First working draft  
**Workstream:** UI/UX — Work Product 10.5  
**References:** Foundry UX Doctrine v0, Foundry Position Projection Schema v0, Foundry UI Pattern Library v0, Foundry Pattern Variability and Personalization Model v0, Foundry UI/UX Workstream Charter v1  
**Purpose:** Define the authoritative specifications for all semantic UI building blocks that live under the UI patterns. These components are the business-aware primitives the compiler assembles to fill pattern slots.

This library sits between the pattern layer (10.3) and the React implementation layer (10.7). Components here are specified as business contracts — not React code. Every component name referenced in 10.3 output contracts and in 10.4 `PersonalizationSurface.targetComponentTypes` must resolve to a spec in this library.

---

## 1. Purpose and position in the system

```
UI Pattern Library (10.3) ————— defines pattern slots and allowed children
[This document — 10.5] ————— defines what fills those slots
Primitive Mapping (10.7) ————— maps these specs to React + Radix + tokens
```

Patterns define structure. Components define content and behavior within that structure. The compiler selects a pattern, then selects and configures components to fill its slots. Without this library, the compiler cannot complete the final composition step.

**What a semantic component is:**  
A reusable, business-aware UI building block. It knows about business concepts — records, approvals, exceptions, tasks, timelines. It is not a generic widget (not a button, not a table row). It encodes business semantics into its input contract, state model, and behavior.

**What a semantic component is not:**  
- A React component (that is 10.7's concern)
- A UI pattern (patterns are full screens or flows; components fill slots within them)
- A design token or style primitive (those are 10.7)

---

## 2. How to read this library

Each component is specified with ten properties. All ten are required before a component is considered fully specified.

| Property | What it answers |
|---|---|
| Purpose | What business problem this component solves |
| When to use / when not to use | Correct placement and anti-patterns |
| Inputs | The data contract: what the component receives |
| Slots | Named areas within the component that accept child content |
| States | The meaningful states the component moves through |
| Actions | What the component can do (user-initiated or system-triggered) |
| Composition rules | What it can contain; what can contain it; nesting constraints |
| Accessibility | Required ARIA roles, focus behavior, keyboard navigation |
| Telemetry hooks | Required event names and when they fire |
| Allowed personalization surfaces | Which surfaces from 10.4 §4 can target this component |

---

## 3. Component composition matrix

This matrix shows which patterns each component appears in and whether the placement is required or conditional.

| Component | CollectionView | RecordPage | OverviewMonitor | SearchResults | ExceptionResolutionView | ApprovalReviewView | CreateEditFlow | ApprovalFlow | ExceptionResolutionFlow | AssistedSetupFlow |
|---|---|---|---|---|---|---|---|---|---|---|
| RecordHeader | — | Required | — | — | — | Required | — | — | — | — |
| KeyFactsStrip | Conditional (row) | Required (header) | — | Conditional (row) | Required | Required | — | — | Required | — |
| SmartFormSection | — | Conditional | — | — | — | — | Required | Conditional | Required | Required |
| StatusBanner | Conditional | Conditional | Conditional | Conditional | Conditional | Conditional | Conditional | Conditional | Conditional | Conditional |
| ActionPanel | Required | Required | Conditional | — | Required | Required | Required | Required | Required | Required |
| FilterRail | Required | — | — | — | — | — | — | — | — | — |
| BulkActionBar | Conditional | — | — | — | — | — | — | — | — | — |
| ActivityTimeline | — | Conditional | — | — | Conditional | Conditional | — | — | — | — |
| ApprovalPanel | — | Conditional | — | — | — | Required | — | — | — | — |
| RelatedRecordsSection | — | Conditional | — | — | — | — | — | — | — | — |
| AttachmentPanel | — | Conditional | — | — | — | Conditional | Conditional | Conditional | — | — |
| EmptyState | Required | Required | Required | Required | — | Conditional | — | — | — | — |
| ValidationSummary | — | — | — | — | — | — | Required | — | Conditional | — |
| TaskContextPanel | — | Conditional | Conditional | — | — | — | — | — | — | — |
| CommentConversationPanel | — | Conditional | — | — | — | — | — | — | — | — |

**Key:** Required = must be present when this pattern is instantiated. Conditional = present only when the relevant projection data exists.

---

## 4. Component specifications

---

### C1. RecordHeader

**Purpose**  
Displays the identity, classification, status, and key facts of a single entity at the top of a RecordPage or ApprovalReviewView. Establishes the record's context before action controls appear (UX Doctrine P4).

**When to use**  
- As the first region of any RecordPage or ApprovalReviewView.
- Whenever a single entity instance is the primary subject of a view.

**When not to use**  
- Inside list rows — use KeyFactsStrip directly.
- Inside flow steps — flows do not have a RecordHeader; they have a FlowHeader.
- When no entity is the primary subject (e.g., OverviewMonitor).

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `entityType` | string | Required | From EntityTypeRef.entityType — used for the entity type badge |
| `entityTypeBadgeLabel` | string | Required | Display label for the entity type badge |
| `displayName` | string | Required | Runtime-resolved display name from EntityTypeRef.displayNameTemplate |
| `statusLabel` | string | Required | Current record status label (from TaskSpec.stateModel or ActionSpec state model) |
| `statusSemantic` | enum(neutral, positive, warning, critical, inactive) | Required | Maps status to a visual severity treatment |
| `keyFacts` | KeyFactSpec[] | Required | 4–8 key facts; count governed by `key_fact_count` variant |

**Slots**

| Slot | Child component | Required |
|---|---|---|
| `entity-type-badge` | StatusBadge (entity type variant) | Required |
| `status-badge` | StatusBadge (status variant) | Required — rendered before actions in DOM order |
| `display-name` | text heading | Required |
| `key-facts` | KeyFactsStrip | Required |
| `action-group` | ActionPanel | Required when non-read-only state |

**States**

| State | Behavior |
|---|---|
| `loading` | Skeleton for name, status badge, and key facts strip; entity type badge renders immediately |
| `populated` | Full content visible |
| `error` | StatusBanner inline; display name shows fallback placeholder; key facts show dash placeholders |

**Actions**  
None. RecordHeader delegates all actions to the ActionPanel in its `action-group` slot.

**Composition rules**  
- Must be the first child node of RecordPage or ApprovalReviewView.
- Cannot be nested inside another RecordHeader.
- Cannot appear inside a flow pattern.
- StatusBadge in the `status-badge` slot must precede ActionPanel in both DOM order and visual order (invariant P4).
- KeyFactsStrip in `key-facts` is constrained to the `key_fact_count` variant value for this view (4–8).

**Accessibility**  
- `displayName` renders as `<h1>` — the page's primary landmark heading.
- `statusLabel` StatusBadge has `aria-label` describing the semantic meaning of the status, not just the label text.
- KeyFactsStrip renders as `<dl>` — see C2.
- The entity type badge renders as a visually distinct `<span>` with `role="note"`.

**Telemetry hooks**  
None directly — telemetry on actions is owned by ActionPanel (C5).

**Allowed personalization surfaces**  
| Surface | Variant id | Scope | Effect |
|---|---|---|---|
| Emphasis variant | `key_fact_count` | Role | Controls how many facts KeyFactsStrip renders (4–8) |
| Density mode | `header_density` | User | Applies compact spacing to the header region |

---

### C2. KeyFactsStrip

**Purpose**  
Displays a concise, labeled set of key facts about an entity. Used wherever a quick summary of an entity's most important attributes is needed — in record headers, list rows, exception contexts, and approval review.

**When to use**  
- Inside RecordHeader for entity detail views.
- Inside RecordRow on CollectionView (max 4 facts).
- Inside ExceptionResolutionView context section (max 6 facts).
- Inside ApprovalReviewView subject context (max 8 facts).
- Inside SearchResultRow (max 3 facts).

**When not to use**  
- As a standalone page-level component without a parent context.
- To display more than 8 facts — exceeding 8 is a pattern selection error.
- To display actions or interactive controls — KeyFactsStrip is display-only.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `facts` | KeyFactSpec[] | Required | The facts to display; count capped by `maxFacts` |
| `maxFacts` | number | Required | Context-determined: 2–4 in rows, 4–8 in headers, max 6 in exception, max 3 in search |
| `orientation` | enum(horizontal, vertical) | Optional | Default: horizontal. Vertical for sidebar or narrow contexts |

Each `KeyFactSpec`:

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | Data field identifier |
| `fieldLabel` | string | Required | Display label |
| `displayFormat` | enum(text, number, currency, date, status, badge) | Required | How to render the value |
| `isHighlighted` | boolean | Optional | Receives visual emphasis; max 2 highlighted per strip |
| `value` | string | Required (runtime) | Resolved at runtime — not a schema value |

**Slots**  
None — renders facts directly as a definition list.

**States**

| State | Behavior |
|---|---|
| `populated` | All facts rendered |
| `partial` | Some values loading — skeleton dashes for unresolved values |
| `overflow` | More facts than `maxFacts` — silently truncates; no "show more"; excess is a data error |

**Actions**  
None — display-only component.

**Composition rules**  
- Allowed inside: RecordHeader, RecordRow, ExceptionResolutionView context section, ApprovalReviewView subject context, SearchResultRow.
- Cannot stand alone at page level.
- Cannot contain interactive elements.
- `isHighlighted` may be set on at most 2 facts per strip. If more than 2 are marked, only the first 2 receive emphasis treatment.

**Accessibility**  
- Renders as `<dl>` (definition list): each fact is a `<dt>` (label) + `<dd>` (value) pair.
- `isHighlighted` facts have `aria-label="key fact"` or equivalent visually-hidden annotation.
- Status and badge format values have `role="status"` or descriptive `aria-label`.

**Telemetry hooks**  
None — passive display component.

**Allowed personalization surfaces**  
| Surface | Variant id | Scope | Effect |
|---|---|---|---|
| Emphasis variant | `key_fact_count` | Role | How many facts to display when context allows fewer than the max |

---

### C3. SmartFormSection

**Purpose**  
A labeled, collapsible form section with guided field layout, blur-triggered inline validation, and section-level error aggregation. The primary building block of all form input — used in CreateEditFlow, ExceptionResolutionFlow, AssistedSetupFlow, and edit states on RecordPage.

**When to use**  
- Whenever the user must enter or edit structured data.
- One SmartFormSection per logical group of related fields.
- Use multiple SmartFormSections composed by CreateEditFlow's StepNavigator for multi-section forms.

**When not to use**  
- For display-only data — use KeyFactsStrip or a read-only section instead.
- For single-field inline edits on a RecordPage — use an inline edit control directly.
- For nested forms — SmartFormSections do not nest.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `sectionKey` | string | Required | Unique identifier within the form |
| `sectionLabel` | string | Required | Displayed section heading |
| `sectionDescription` | string | Optional | Supporting guidance shown below the label |
| `fields` | InputFieldSpec[] | Required | The fields in this section |
| `isCollapsedByDefault` | boolean | Required | Default expansion state; from ViewSpec.layoutHints |
| `isReadOnly` | boolean | Required | True for context-only sections in edit flows |
| `autosaveEnabled` | boolean | Required | From the `autosave` structural variant (10.4 §4.7) |

**Slots**

| Slot | Child | Required |
|---|---|---|
| `section-header` | Label text + collapse toggle | Required |
| `field-group` | InputField[] | Required |
| `validation-summary` | ValidationSummary (C13) | Conditional — visible when section has errors |
| `ai-assist` | AIAssistPanel | Conditional — when aiAssistHookIds present for this section |

**States**

| State | Behavior |
|---|---|
| `empty` | No fields filled; required fields marked; no errors shown |
| `partial` | Some fields filled; blur-validation active; no section-level error yet |
| `valid` | All required fields filled and valid; section-level indicator positive |
| `error` | One or more fields invalid; ValidationSummary visible; section-level error indicator |
| `read-only` | Fields rendered as labeled values, not inputs; no validation |
| `loading` | Edit mode only — skeleton inputs while field values are fetched |
| `saving` | Autosave in progress — unobtrusive indicator; fields remain interactive |

**Actions**

| Action | Trigger | Behavior |
|---|---|---|
| `field-blur` | User leaves a field | Triggers per-field validation for that field only |
| `collapse-toggle` | User clicks section header | Expands or collapses the section body |
| `autosave-trigger` | Field value changes (when autosave enabled) | Queues a debounced autosave write |

**Composition rules**  
- One SmartFormSection per logical section — never nest SmartFormSections inside each other.
- Multiple SmartFormSections are composed and stepped by CreateEditFlow's StepNavigator.
- `autosave` trigger only fires when the `autosave` structural variant is enabled at tenant scope (10.4 §4.7).
- ValidationSummary in the `validation-summary` slot is in addition to per-field inline errors — not instead of them (UX Doctrine §7.3).
- AI-generated field fills must be visually distinguished from user-entered values and require explicit acceptance (P11, P12).

**Accessibility**  
- Section label renders as `<fieldset>` + `<legend>`.
- Required fields have `aria-required="true"`.
- Inline field errors are linked to their field via `aria-describedby`.
- Collapse toggle has `aria-expanded` reflecting current state.
- In `saving` state, a visually hidden `aria-live="polite"` region announces "Saving…" and "Saved".

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `section_expand` | Section is expanded |
| `section_collapse` | Section is collapsed |
| `validation_error_shown` | Section enters error state (includes sectionKey; no field values) |
| `autosave_triggered` | Autosave write fires |

**Allowed personalization surfaces**  
None at component level. Section grouping and expansion defaults are governed by `ViewSpec.layoutHints.groupedSections` (role-scope, schema level).

---

### C4. StatusBanner

**Purpose**  
Displays a contextual system message at informational, warning, error, or success severity. Used for loading errors, conflict resolution notifications, action feedback, and inline alerts.

**When to use**  
- When the system needs to communicate a state change, error, or notification within a view without interrupting the workflow.
- For conflict resolution notifications (10.4 §7.2).
- For retry affordances after load or mutation errors.

**When not to use**  
- As the sole content of a modal — use a full error state instead.
- For persistent business exceptions — use ExceptionResolutionView.
- More than 2 StatusBanners in a single region simultaneously — this indicates a design error.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `severity` | enum(informational, warning, error, success) | Required | Drives icon, color, and ARIA role |
| `message` | string | Required | Plain language — always explains what happened |
| `actionLabel` | string | Optional | Label for an inline action (retry, reconfigure, etc.) |
| `actionId` | id | Optional | References ActionSpec or a local callback |
| `isDismissible` | boolean | Required | Whether the user can dismiss |
| `isPersistent` | boolean | Required | If true, dismiss control is absent |

**Slots**

| Slot | Child | Required |
|---|---|---|
| `icon` | Severity icon (derived from severity; not overridable) | Required |
| `message` | Text content | Required |
| `action` | Inline action button | Conditional |

**States**

| State | Behavior |
|---|---|
| `visible` | Rendered |
| `dismissed` | Removed from DOM after user dismissal |
| `persistent` | Visible; no dismiss control |

**Actions**

| Action | Condition | Behavior |
|---|---|---|
| `dismiss` | `isDismissible=true` and `isPersistent=false` | Removes banner from DOM |
| `inline-action` | `actionLabel` and `actionId` present | Triggers the associated action |

**Composition rules**  
- Allowed in any pattern at any level.
- Cannot be nested inside another StatusBanner.
- At most 2 StatusBanners per region simultaneously. Multiple banners stack vertically.
- Must not be used as the sole content of a modal.

**Accessibility**  
- `role="alert"` for `severity=warning` and `severity=error` (live region — announced immediately).
- `role="status"` for `severity=informational` and `severity=success` (polite announcement).
- Dismiss button has `aria-label="Dismiss [severity] message"`.

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `banner_shown` | Banner becomes visible (includes severity, viewId) |
| `banner_dismissed` | User dismisses the banner |
| `banner_action_clicked` | Inline action is triggered |

**Allowed personalization surfaces**  
None.

---

### C5. ActionPanel

**Purpose**  
Renders the action controls for a pattern or component, enforcing the action budget from UX Doctrine §6. The single source of truth for action rendering in any context. Every action the user can take in a view passes through an ActionPanel.

**When to use**  
- In the `action-group` slot of RecordHeader.
- In the page header of CollectionView and OverviewMonitor.
- In every flow step's primary control area.
- As the decision control in ApprovalReviewView.

**When not to use**  
- For row-level inline actions — those are rendered directly in RecordRow, not via ActionPanel.
- For bulk actions — BulkActionBar (C7) handles those.
- More than one ActionPanel per context.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `primaryAction` | ActionSpec | Optional | The single dominant action; absent in read-only states |
| `secondaryActions` | ActionSpec[] | Optional | Max 3 visible secondary actions |
| `overflowActions` | ActionSpec[] | Optional | Actions placed in overflow menu when total > 5 |
| `orientation` | enum(horizontal, vertical) | Optional | Default: horizontal |
| `context` | enum(page, modal, inline, flow-step) | Required | Affects sizing and visual weight |

**Slots**

| Slot | Content | Constraint |
|---|---|---|
| `primary` | Primary action button | Max 1 |
| `secondary` | Secondary action buttons | Max 3 visible |
| `overflow` | Overflow menu trigger + items | Only when total visible > 5 |
| `destructive-zone` | Visually isolated destructive actions | Must not be adjacent to `primary` slot (A5) |

**States**

| State | Behavior |
|---|---|
| `default` | All permitted actions available |
| `loading` | Primary action in loading state; all other actions disabled |
| `disabled-partial` | One or more actions disabled per PreconditionSpec (A6: hidden or disabled-with-tooltip) |

**Actions**  
Delegates to each ActionSpec. Confirmation behavior per `ActionSpec.confirmationType` (none, inline, full).

**Composition rules**  
- One ActionPanel per context — never two ActionPanels in the same slot.
- Destructive actions (`isDestructive=true`) must be placed in the `destructive-zone` slot, not in `primary` or `secondary`.
- Budget is enforced at compile time via `ActionSlotBinding` (schema C10): max 1 primary, max 3 secondary, max 5 total visible, overflow for any beyond 5.
- Actions absent from `permissionsHooks.allowedActionIds` must not be rendered by ActionPanel regardless of their presence in the binding (schema C4).
- Actions with unmet preconditions: `displayWhenNotMet=hide` → hidden; `displayWhenNotMet=disable_with_tooltip` → disabled with tooltip (A6).

**Accessibility**  
- All action buttons are `<button>` elements with descriptive `aria-label`.
- Overflow menu trigger has `aria-haspopup="menu"` and `aria-expanded`.
- Disabled actions have `aria-disabled="true"` and tooltip linked via `aria-describedby`.
- After a modal or flow completes, focus returns to the triggering ActionPanel button.

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `action_initiated` | Any action button is clicked (includes actionId, actionType, context) |
| `overflow_opened` | Overflow menu is opened |
| `action_confirmed` | Confirmation step completes (includes actionId) |
| `action_cancelled` | Confirmation step is cancelled |

**Allowed personalization surfaces**  
None. Action presence is governed by `permissionsHooks.allowedActionIds`; budget is invariant (I8).

---

### C6. FilterRail

**Purpose**  
Renders the filter dimensions for a CollectionView. Manages filter state, communicates active filter count, and provides clear-all affordance. Search is always its first element.

**When to use**  
- As the filter surface inside CollectionView only.
- Always present when `filterSources` is non-empty on the view.

**When not to use**  
- Outside of a CollectionView context.
- As a standalone global filter panel.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `filterSources` | FilterSourceSpec[] | Required | Available filter dimensions from the view's PatternInputBinding |
| `activeFilters` | AppliedFilter[] | Required (runtime) | Currently applied filter values |
| `defaultVisibleCount` | number | Optional | Filters shown before expansion; default 3 |

**Slots**

| Slot | Child | Required |
|---|---|---|
| `search-bar` | SearchBar | Required — always first |
| `filter-groups` | FilterGroup[] | Required — one per isDefaultVisible=true filterSource |
| `expanded-filters` | FilterGroup[] | Conditional — isDefaultVisible=false filters |
| `active-filter-indicator` | ActiveFilterIndicator | Conditional — when any filter is applied |
| `clear-all` | Clear-all control | Conditional — when any filter is applied |

**States**

| State | Behavior |
|---|---|
| `idle` | No filters applied; default groups visible |
| `active` | One or more filters applied; ActiveFilterIndicator shows count; clear-all visible |
| `expanded` | Advanced filters revealed |
| `loading` | Filter option lists loading — skeleton chips |

**Actions**

| Action | Behavior |
|---|---|
| `apply-filter` | Adds or changes a filter value; list updates in place |
| `clear-filter` | Removes one filter dimension |
| `clear-all` | Removes all active filters simultaneously |
| `expand-filters` | Reveals isDefaultVisible=false filter groups |

**Composition rules**  
- Allowed only inside CollectionView.
- SearchBar must always be the first slot — never hidden, never below filter groups (UX Doctrine P3, §7.1).
- Common filters (`isDefaultVisible=true`) are visible by default; advanced filters are behind expansion (§7.2).
- The `filter_placement` variant governs whether this renders as a sidebar rail or an inline chip bar (10.4 §4.1).

**Accessibility**  
- Each filter group is a `<fieldset>` with a `<legend>`.
- Active filter count is announced via `aria-live="polite"` region.
- Clear-all button has `aria-label="Clear all filters"`.
- Filter chips (when chip-bar variant) have `aria-pressed` state.

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `filter_applied` | A filter is applied (includes filterKey; not the value) |
| `filter_cleared` | A single filter is cleared |
| `filters_cleared_all` | Clear-all is triggered |
| `filter_rail_expanded` | Advanced filters section is revealed |

**Allowed personalization surfaces**

| Surface | Variant id | Scope | Effect |
|---|---|---|---|
| Emphasis variant | `filter_placement` | User | Rail sidebar (default) vs inline chip bar |
| View-state | `saved_filter` | User | Which filter keys and values are saved per view |

---

### C7. BulkActionBar

**Purpose**  
Replaces the page header ActionPanel during selection state on a CollectionView. Shows selected count and available bulk actions. Enforces isolation of destructive bulk actions.

**When to use**  
- Only when items are selected in a CollectionView with `bulkActionIds` defined.

**When not to use**  
- Outside of CollectionView selection state.
- When no `bulkActionIds` are declared in the view's ActionSlotBinding.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `selectedCount` | number | Required (runtime) | Currently selected item count |
| `totalCount` | number | Required (runtime) | Total items in current result set |
| `bulkActions` | ActionSpec[] | Required | From ActionSlotBinding.bulkActionIds |

**Slots**

| Slot | Content | Constraint |
|---|---|---|
| `selection-indicator` | "N of M selected" + select-all + deselect-all | Required |
| `bulk-actions` | Non-destructive bulk action buttons | Required |
| `destructive-zone` | Destructive bulk actions | Visually isolated from bulk-actions slot (A5) |

**States**

| State | Behavior |
|---|---|
| `active` | One or more items selected; replaces PageHeader ActionPanel |
| `inactive` | Zero items selected; BulkActionBar not rendered |

**Actions**

| Action | Behavior |
|---|---|
| `select-all` | Selects all items in the current result set |
| `deselect-all` | Clears all selections |
| `bulk-action` | Triggers action on all selected items; confirmation per ActionSpec.confirmationType |

**Composition rules**  
- Allowed only inside CollectionView.
- Replaces ActionPanel in the page header slot during selection state — they do not coexist (A4, invariant in 10.3 P1).
- Destructive bulk actions must be in the `destructive-zone` slot, visually isolated (A5).

**Accessibility**  
- Selection count is announced via `aria-live="polite"` on change.
- Select-all checkbox has `aria-label="Select all [entity type] records"`.
- Bulk action buttons include selected count in `aria-label` (e.g., "Delete 5 records").

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `bulk_action_initiated` | A bulk action is triggered (includes actionId, selectedCount) |
| `bulk_select_all` | Select-all is triggered |
| `bulk_deselect_all` | Deselect-all is triggered |

**Allowed personalization surfaces**  
None.

---

### C8. ActivityTimeline

**Purpose**  
Renders a chronological audit history of significant events, state changes, actions, and approvals on a specific entity. Required on any entity participating in a business workflow (UX Doctrine §2.1 traceability).

**When to use**  
- On RecordPage for any entity with a TaskSpec or ApprovalSpec.
- On ApprovalReviewView to show prior approval history.
- On ExceptionResolutionView when `timeline_inclusion=inline` variant is active.

**When not to use**  
- For entities with no workflow involvement (no TaskSpec, ApprovalSpec, or AlertSpec).
- Inside SmartFormSection or KeyFactsStrip.
- As a standalone page element without a parent record context.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `entityType` | string | Required | Entity type for labeling |
| `entityId` | id | Required (runtime) | Used to fetch events |
| `events` | TimelineEvent[] | Required (runtime) | Ordered newest-first |
| `maxVisible` | number | Optional | Default 5; expand to see earlier events |

Each `TimelineEvent`:

| Field | Type | Required | Description |
|---|---|---|---|
| `eventId` | id | Required | |
| `eventType` | enum(state_change, action_taken, approval_given, approval_rejected, comment_added, ai_action, system_event) | Required | |
| `timestamp` | datetime | Required | Displayed in a `<time>` element |
| `actorLabel` | string | Required | Who performed the action |
| `description` | string | Required | Plain language; no technical codes |
| `isAIInitiated` | boolean | Required | Drives visual AI-action distinction (V2, P11) |

**Slots**

| Slot | Content | Required |
|---|---|---|
| `timeline-header` | Label + expand/collapse control | Required |
| `event-list` | TimelineEvent[] | Required |
| `load-more` | Load earlier events control | Conditional |

**States**

| State | Behavior |
|---|---|
| `loading` | Skeleton event rows |
| `populated` | Events rendered; newest first |
| `empty` | EmptyState: "No activity recorded yet" |
| `error` | StatusBanner with retry; last-loaded events preserved |

**Actions**

| Action | Behavior |
|---|---|
| `expand` | Shows all events beyond maxVisible |
| `collapse` | Returns to maxVisible |
| `load-more` | Fetches earlier events from backend |

**Composition rules**  
- Allowed inside: RecordPage, ApprovalReviewView, ExceptionResolutionView (variant only).
- Cannot be nested inside SmartFormSection, KeyFactsStrip, or BulkActionBar.
- AI-initiated events (`isAIInitiated=true`) must be visually distinguished from human-initiated events (V2, P11). The distinction must survive token changes.

**Accessibility**  
- Renders as `<ol>` (ordered list — chronological order is meaningful).
- Each event is an `<li>` with a `<time datetime="...">` element.
- AI-initiated events have `aria-label="AI-initiated action"` or visually-hidden annotation.
- Expand/collapse button has `aria-expanded`.

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `timeline_expanded` | User expands to see all events |
| `timeline_collapsed` | User collapses to maxVisible |
| `timeline_load_more` | Older events are loaded |

**Allowed personalization surfaces**

| Surface | Variant id | Scope | Effect |
|---|---|---|---|
| Emphasis variant | `timeline_placement` | Role | inline_below_sections (default) vs tab (RecordPage only) |

---

### C9. ApprovalPanel

**Purpose**  
Displays the approval lifecycle of an entity — current state, pending approvers, rejection reason, and full approval event history. Implements the `approvalHistoryVisible=true` requirement from every ApprovalSpec (UX Doctrine §7.4).

**When to use**  
- On RecordPage for any entity participating in an ApprovalSpec.
- On ApprovalReviewView.

**When not to use**  
- On entities without an ApprovalSpec.
- Inside flow steps.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `approvalType` | string | Required | From ApprovalSpec.approvalType |
| `currentState` | enum(pending, approved, rejected, expired, withdrawn) | Required | Runtime |
| `approvalEvents` | ApprovalEvent[] | Required (runtime) | Full history |
| `pendingApprovers` | ApproverRef[] | Optional (runtime) | Present when currentState=pending |
| `rejectionReason` | string | Optional (runtime) | Present when currentState=rejected |
| `deadlineLabel` | string | Optional | Formatted deadline from approvalDeadlineHours |

Each `ApprovalEvent`:

| Field | Type | Required | Description |
|---|---|---|---|
| `eventType` | enum(submitted, approved, rejected, withdrawn, expired, resubmitted) | Required | |
| `actorLabel` | string | Required | |
| `timestamp` | datetime | Required | |
| `comment` | string | Optional | Present on rejection events |

**Slots**

| Slot | Content | Required |
|---|---|---|
| `approval-status` | Current state badge | Required |
| `pending-approvers` | List of approvers yet to act | Conditional |
| `deadline-indicator` | Deadline proximity warning | Conditional |
| `event-history` | ApprovalEvent[] newest-first | Required |
| `rejection-reason` | Rejection reason text | Conditional |

**States**

| State | Behavior |
|---|---|
| `pending` | Current state badge; pending approvers visible |
| `approved` | Approved state; full history visible; pending section absent |
| `rejected` | Rejection reason prominently shown; full history visible |
| `expired` | Expired state badge; escalation information shown |
| `withdrawn` | Withdrawn by initiator; history shows withdrawal event |

**Actions**  
None — display only. Actions live in ActionPanel.

**Composition rules**  
- Allowed inside: RecordPage, ApprovalReviewView.
- `approvalHistoryVisible` must be `true` — this component implements that requirement (ApprovalSpec rule).
- Cannot appear in a view unless an ApprovalSpec exists for the entity type.
- Rejection reason must be visually prominent when `currentState=rejected` — it is not hidden behind expansion.

**Accessibility**  
- Current state badge has `role="status"`.
- Rejection reason has `aria-label="Rejection reason"`.
- Approval history is an `<ol>` with `<time>` elements.
- Deadline indicator has `role="alert"` when deadline is approaching.

**Telemetry hooks**  
None directly — passive display component.

**Allowed personalization surfaces**

| Surface | Variant id | Scope | Effect |
|---|---|---|---|
| Emphasis variant | `approval_history_placement` | Role | inline (default) vs collapsible_section (ApprovalReviewView only) |

---

### C10. RelatedRecordsSection

**Purpose**  
Displays grouped lists of related entity references with navigation links. Surfaces the relationships between the current record and other entities that this position cares about.

**When to use**  
- On RecordPage when the entity has related EntityTypeRef entries with `viewLink` defined.

**When not to use**  
- As a standalone page element.
- When there are no related entity references in the projection.
- Inside flow patterns.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `relatedGroups` | RelatedGroup[] | Required | One group per related entity type |

Each `RelatedGroup`:

| Field | Type | Required | Description |
|---|---|---|---|
| `groupLabel` | string | Required | E.g., "Related Purchase Orders" |
| `entityTypeRef` | EntityTypeRef | Required | The related entity type |
| `records` | RelatedRecord[] | Required (runtime) | Each with displayName, status, viewLink |
| `maxVisible` | number | Optional | Default 3; expand for more |

**Slots**

| Slot | Content | Required |
|---|---|---|
| `group-header` | Group label + record count | Required per group |
| `record-list` | RelatedRecord[] as navigable links | Required |
| `load-more` | Show more records control | Conditional |

**States**

| State | Behavior |
|---|---|
| `loading` | Skeleton list per group |
| `populated` | Records rendered |
| `empty` | Per-group EmptyState with message only (no action path — this is a contextual empty, not a primary empty) |
| `inaccessible` | EntityTypeRef.viewLink unavailable for current user — renders as plain non-navigable label per schema §4.3 |

**Actions**

| Action | Behavior |
|---|---|
| `navigate` | Clicking a related record navigates to its RecordPage via EntityTypeRef.viewLink |

**Composition rules**  
- Allowed inside RecordPage only.
- Inaccessible viewLinks must render as `<span>` (non-navigable label), not a broken `<a>` tag and not hidden (schema §4.3 rule).
- Cannot contain ActionPanel, SmartFormSection, or ActivityTimeline.

**Accessibility**  
- Each group is a `<section>` with a `<h3>` heading.
- Navigable related records are `<a>` elements with `aria-label` including entity type and name.
- Inaccessible records render as `<span>` with no interactive affordance.

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `related_record_navigated` | A related record link is clicked (includes entityType, target viewId) |

**Allowed personalization surfaces**  
None.

---

### C11. AttachmentPanel

**Purpose**  
Displays and manages file attachments associated with a record or flow step. Supports viewing and — when permitted — uploading supporting documents.

**When to use**  
- On RecordPage when the entity type carries supporting documents.
- In ApprovalReviewView for initiator-submitted supporting context.
- In CreateEditFlow and ApprovalFlow when the ActionSpec permits file attachments.

**When not to use**  
- When no file attachments are part of the entity model.
- As a standalone page element.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `attachments` | Attachment[] | Required (runtime) | Current file list |
| `allowUpload` | boolean | Required | From ActionSpec permissions |
| `maxFiles` | number | Optional | Tenant-configured limit |
| `acceptedTypes` | string[] | Optional | MIME type restrictions |

Each `Attachment`:

| Field | Type | Required | Description |
|---|---|---|---|
| `attachmentId` | id | Required | |
| `fileName` | string | Required | |
| `fileType` | string | Required | |
| `uploadedBy` | string | Required | Display name of uploader |
| `uploadedAt` | datetime | Required | |
| `isViewable` | boolean | Required | Whether the current user can open this file |

**Slots**

| Slot | Content | Required |
|---|---|---|
| `attachment-list` | Attachment rows | Required |
| `upload-area` | Drag-drop or file picker | Conditional — `allowUpload=true` only |
| `empty-state` | EmptyState | Required when list is empty |

**States**

| State | Behavior |
|---|---|
| `loading` | Skeleton list |
| `populated` | Attachment list rendered |
| `empty` | EmptyState with upload affordance when `allowUpload=true`; without affordance when not |
| `uploading` | Upload in progress — progress indicator per file |

**Actions**

| Action | Condition | Behavior |
|---|---|---|
| `view` | `isViewable=true` | Opens file in new tab or preview panel |
| `upload` | `allowUpload=true` | Triggers file picker or drag-drop |
| `delete` | `isDestructive=true`; `confirmationType=inline` | Removes attachment after inline confirmation |

**Composition rules**  
- Allowed inside: RecordPage, ApprovalReviewView, CreateEditFlow, ApprovalFlow.
- Upload action only rendered when `allowUpload=true` and the action is in `permissionsHooks.allowedActionIds`.
- Delete is a destructive action — must use `confirmationType=inline` minimum; must be visually isolated from view action (A5).

**Accessibility**  
- File list is a `<ul>`.
- Upload area has `role="button"` or uses a visible `<input type="file">` with associated `<label>`.
- Upload progress is announced via `aria-live="polite"`.
- Delete button has `aria-label="Delete [fileName]"`.

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `attachment_viewed` | A file is opened (file type only; not file name) |
| `attachment_uploaded` | Upload completes (file type only) |
| `attachment_deleted` | An attachment is deleted |

**Allowed personalization surfaces**  
None.

---

### C12. EmptyState

**Purpose**  
Communicates why a region is empty and provides a direct action path where appropriate. Required on every region that can be empty — a blank region is never acceptable (UX Doctrine 1.7, XP2, V5).

**When to use**  
- In any list, collection, or content region that has zero items.
- In any region where data could not be loaded.
- For access-denied and not-found conditions at page level.
- For the positive "no alerts" confirmation in OverviewMonitor (a positive-confirmation EmptyState, not an error state).

**When not to use**  
- As a loading state — use skeleton content instead.
- More than one EmptyState per region simultaneously.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `context` | enum(no_records, no_search_results, no_alerts, no_tasks, no_attachments, access_denied, not_found, error, positive_confirmation) | Required | Drives the message template and illustration |
| `message` | string | Required | Plain language — always provided; explains why the region is empty |
| `actionLabel` | string | Optional | Direct path action label |
| `actionId` | id | Optional | References ActionSpec or local callback |
| `illustration` | enum(none, default_for_context) | Optional | Default uses context-appropriate illustration |

**Slots**

| Slot | Content | Required |
|---|---|---|
| `illustration` | Optional visual | Conditional |
| `message` | Required text | Required |
| `action` | Optional action button | Conditional |

**States**

| State | Behavior |
|---|---|
| `standard` | Message + optional action |
| `positive` | Positive confirmation variant — "No active alerts — all operational" |

**Actions**

| Action | Condition | Behavior |
|---|---|---|
| `primary-action` | `actionLabel` and `actionId` present | Triggers the direct path action |

**Composition rules**  
- May appear in any pattern region that can be empty.
- Must never be replaced with a blank region (V5).
- `context=access_denied` and `context=not_found` appear at full-page level only — not inside a component sub-region.
- `context=positive_confirmation` is used only in OverviewMonitor's alert list region when no active alerts exist.
- `context=no_search_results` must display the submitted query so the user understands what produced no results.

**Accessibility**  
- `role="status"` for `positive_confirmation` context.
- `role="alert"` for `error` context.
- `role="region"` with `aria-label` for all other contexts.
- Action button has descriptive `aria-label`.

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `empty_state_viewed` | EmptyState is rendered (includes context type, viewId) |

**Allowed personalization surfaces**  
None.

---

### C13. ValidationSummary

**Purpose**  
Aggregates and displays field-level validation errors at section or form level. Provides jump-to-field navigation. Appears in addition to per-field inline errors — not as a replacement.

**When to use**  
- Inside SmartFormSection when the section enters error state.
- At the top of a CreateEditFlow when form-submit is blocked by errors.

**When not to use**  
- As a replacement for per-field inline errors.
- Outside of a form context.
- When there are no errors — ValidationSummary must not be rendered with an empty error list.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `errors` | ValidationError[] | Required | One entry per invalid field |
| `level` | enum(section, form) | Required | Section-level renders inline; form-level renders at the top of the form |

Each `ValidationError`:

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | Identifies the field |
| `fieldLabel` | string | Required | Display label |
| `message` | string | Required | Plain language: names the problem and suggests the fix (P7) |

**Slots**

| Slot | Content | Required |
|---|---|---|
| `error-list` | ValidationError[] as linked items | Required |

**States**

| State | Behavior |
|---|---|
| `hidden` | No errors — component not rendered |
| `visible` | One or more errors — rendered and announced |

**Actions**

| Action | Behavior |
|---|---|
| `navigate-to-field` | Clicking an error item focuses the corresponding InputField and scrolls it into view |

**Composition rules**  
- Allowed inside SmartFormSection (section level) and CreateEditFlow (form level).
- Per-field inline errors must also appear adjacent to their field — ValidationSummary does not replace them.
- Cannot render with an empty `errors` array.
- Focus moves to ValidationSummary when form submit is blocked by errors.

**Accessibility**  
- `role="alert"` — announced immediately when errors appear.
- Each error item is an `<a>` linking to the corresponding field via `id` reference.
- Focus is programmatically moved to ValidationSummary on blocked form submit.

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `validation_error_shown` | ValidationSummary becomes visible (includes error count; no field values) |

**Allowed personalization surfaces**  
None.

---

### C14. TaskContextPanel

**Purpose**  
Surfaces the task context relevant to the current record — task type, state, priority, due date, and available task-specific actions. Placed in the contextual sidebar of RecordPage or in OverviewMonitor's pending tasks section.

**When to use**  
- In the RecordPage sidebar when a TaskSpec is linked to the entity type.
- In OverviewMonitor's pending tasks section.

**When not to use**  
- When no TaskSpec is linked to the entity.
- At page level without a parent record context.
- Inside flow patterns.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `taskSpec` | TaskSpec | Required | The task type definition from the projection |
| `taskInstance` | TaskInstance | Required (runtime) | Current state, priority, due date, assignee |
| `availableActions` | ActionSpec[] | Required | From TaskSpec.availableActionIds (filtered by permissions) |

`TaskInstance` (runtime):

| Field | Type | Required | Description |
|---|---|---|---|
| `taskId` | id | Required | |
| `currentState` | string | Required | From TaskSpec.stateModel |
| `priority` | string | Required | From TaskSpec.priorityLevels |
| `dueDate` | datetime | Optional | Present when TaskSpec.hasDueDate=true |
| `assigneeLabel` | string | Optional | |

**Slots**

| Slot | Content | Required |
|---|---|---|
| `task-header` | Task type label + current state badge | Required |
| `task-meta` | Priority, due date, assignee | Required |
| `action-group` | ActionPanel with available task actions | Required |
| `ai-assist` | AIAssistPanel | Conditional — when aiAssistHookIds present |

**States**

| State | Behavior |
|---|---|
| `loading` | Skeleton |
| `active` | Task in progress — standard rendering |
| `blocked` | Visually distinct from active — blocked indicator |
| `completed` | Task done — read-only; no action controls |
| `overdue` | Due date passed — visual urgency indicator on task-meta |

**Actions**  
Delegates to ActionPanel using `TaskSpec.availableActionIds`.

**Composition rules**  
- Allowed inside: RecordPage sidebar, OverviewMonitor pending tasks section.
- Cannot stand alone at page level.
- Cannot be nested inside SmartFormSection.
- `availableActions` must be filtered through `permissionsHooks.allowedActionIds` — TaskContextPanel does not render actions the user cannot take.

**Accessibility**  
- Task state badge has `role="status"`.
- Overdue indicator has `aria-label="Task overdue"`.
- Due date renders in a `<time>` element with `datetime` attribute.

**Telemetry hooks**  
Telemetry for task actions is owned by ActionPanel (C5).

**Allowed personalization surfaces**  
None at component level.

---

### C15. CommentConversationPanel

**Purpose**  
Displays threaded comments and conversation history on a record. Supports human commentary and, where configured, AI-generated summaries. AI-generated content must be visually distinguished and require explicit user request.

**When to use**  
- On RecordPage when the entity type benefits from threaded annotation or collaborative commentary.

**When not to use**  
- Inside flow patterns.
- For structured approval commentary — use ApprovalPanel's rejection reason field instead.
- When no comments are expected or permitted on the entity type.

**Inputs**

| Field | Type | Required | Description |
|---|---|---|---|
| `entityType` | string | Required | |
| `entityId` | id | Required (runtime) | |
| `comments` | Comment[] | Required (runtime) | |
| `allowComment` | boolean | Required | From permissions |
| `aiSummaryHookId` | id | Optional | When an AI summarization hook is configured |

Each `Comment`:

| Field | Type | Required | Description |
|---|---|---|---|
| `commentId` | id | Required | |
| `authorLabel` | string | Required | |
| `timestamp` | datetime | Required | |
| `body` | string | Required | |
| `isAIGenerated` | boolean | Required | Drives visual AI-content distinction (V2, P11) |
| `isEditable` | boolean | Required | Can the current user edit this comment |

**Slots**

| Slot | Content | Required |
|---|---|---|
| `comment-list` | Comment[] ordered newest-first | Required |
| `compose-area` | Text input + submit | Conditional — `allowComment=true` |
| `ai-summary` | AI-generated summary | Conditional — `aiSummaryHookId` present |

**States**

| State | Behavior |
|---|---|
| `loading` | Skeleton comments |
| `populated` | Comments rendered |
| `empty` | EmptyState "No comments yet" + compose area if `allowComment=true` |
| `composing` | Compose area focused; submit enabled when non-empty |
| `submitting` | Comment being posted — loading indicator; compose area locked |

**Actions**

| Action | Condition | Behavior |
|---|---|---|
| `add-comment` | `allowComment=true` | Posts comment; clears compose area on success |
| `edit-comment` | `isEditable=true` | Opens inline edit for that comment |
| `ai-summarize` | `aiSummaryHookId` present | Requests AI summary on `on_user_request` trigger (P12) |

**Composition rules**  
- Allowed inside RecordPage only.
- AI-generated comments (`isAIGenerated=true`) must be visually distinguished from human comments — this is invariant V2 and must survive any token override.
- AI summaries are only generated on explicit user request (`on_user_request` trigger) — never on view load (P12).
- Submit button must be disabled when compose area is empty — no empty comment submissions.

**Accessibility**  
- Comment list is `<ol>` (chronological order is meaningful).
- Compose area has `aria-label="Add a comment"`.
- AI-generated content has visually-hidden annotation "AI-generated" via `aria-label` or `::before` with `aria-label`.
- Submit button has `aria-disabled="true"` when compose is empty.

**Telemetry hooks**

| Hook key | Fires when |
|---|---|
| `comment_added` | Comment is posted |
| `comment_edited` | Comment is edited |
| `ai_summary_requested` | User requests AI summary |

**Allowed personalization surfaces**  
None.

---

## 5. Supporting components

These components appear in 10.3 output contracts as sub-components within patterns. They are specified here at reduced depth — inputs and key rules only. Full specs are deferred to a future library version or 10.7 implementation substrate work.

### S1. SearchBar

**Purpose:** The always-visible search input on CollectionView and SearchResults. Always the first element in FilterRail.

**Inputs:** `queryBinding` (string), `placeholder` (string), `isActive` (boolean, runtime)  
**Rules:** Always visible; never hidden behind a toggle (UX Doctrine P3, §7.1). Matched terms in results are highlighted. Empty search state returns to default list.  
**Accessibility:** `role="search"`, `aria-label="Search [entity type]"`.  
**Telemetry:** `search_submitted` (includes result count; not the query string).

---

### S2. AIAssistPanel

**Purpose:** Renders AI suggestions, draft generations, or field fills within a pattern or SmartFormSection. All AI output is labeled, held pending until accepted, and reversible.

**Inputs:** `hookId` (from AIAssistHook), `outputType` (inline_suggestion, panel_result, field_fill), `suggestion` (runtime), `isAccepted` (boolean, runtime)  
**Rules:** AI output is visually labeled. Accept or Edit required before applying. `affectsBusinessState=true` requires explicit confirmation (P11, P12). `isReversible=true` always except `outputType=notification`.  
**Accessibility:** AI suggestion container has `aria-label="AI suggestion"`. Accept and Edit buttons are clearly labeled.  
**Telemetry:** `ai_suggestion_shown`, `ai_suggestion_accepted`, `ai_suggestion_rejected`.

---

### S3. MetricTile

**Purpose:** Displays a single summary metric (count, amount, ratio, or status) in the OverviewMonitor SummaryStrip.

**Inputs:** `label`, `summaryType` (count, amount, ratio, status), `value` (runtime), `alertThresholdCrossed` (boolean, runtime), `linkedViewId`  
**Rules:** Capped at 8 tiles per SummaryStrip (compiler warning if exceeded). Alert indicator shown when threshold crossed. Click navigates to linkedViewId.  
**Telemetry:** `summary_tile_clicked` (includes summaryId, linkedViewId).

---

### S4. AlertList

**Purpose:** Displays active AlertSpec instances in OverviewMonitor, ordered by severity (Critical → Warning → Informational).

**Inputs:** `alerts` (AlertItem[], runtime), each with severity, label, entityRef, linkedViewId  
**Rules:** Cannot be blank when alerts exist. Critical before Warning before Informational (invariant). Empty state shows positive confirmation "No active alerts — all operational" (not blank).  
**Telemetry:** `alert_item_clicked` (includes alertSpecId, severity).

---

### S5. FlowHeader

**Purpose:** The header bar of all flow patterns. Shows flow title, step progress indicator, and cancel/exit action.

**Inputs:** `flowTitle` (string), `currentStep` (number), `totalSteps` (number), `cancelAction` (ActionSpec or local callback)  
**Rules:** Cancel/exit always present at every step (P6). ProgressIndicator active on multi-step flows only.  
**Telemetry:** `flow_started` (on FlowHeader mount), `flow_abandoned` (on cancel at any step with step number).

---

### S6. StepNavigator

**Purpose:** The sidebar list or top stepper that shows the step structure of a multi-section CreateEditFlow or multi-step AssistedSetupFlow.

**Inputs:** `steps` (StepItem[], from LayoutHints.groupedSections), `currentStepIndex` (number, runtime)  
**Rules:** Shown only in multi-section flows. Variant `section_navigator_style` controls sidebar list vs top stepper (10.4 §4.7).  
**Accessibility:** `role="navigation"`, `aria-label="Form steps"`. Current step has `aria-current="step"`.

---

### S7. ReviewSummary

**Purpose:** Displays a read-only summary of all entries made across prior flow steps before final submit. Used in CreateEditFlow (optional), ApprovalFlow, ExceptionResolutionFlow, AssistedSetupFlow.

**Inputs:** `sections` (ReviewSection[], accumulated from prior steps), `isEditable` (boolean — each section links back to its step)  
**Rules:** All prior inputs visible. Each section has an edit link back to its step. Read-only display only — no inline editing.

---

## 6. Cross-component rules

**XC1 — One ActionPanel per context**  
No context (page, modal, flow step, component) may have more than one ActionPanel. The action budget invariant (I8) is enforced per ActionPanel instance.

**XC2 — AI-generated content is always distinguishable**  
Any component that renders AI-generated content (ActivityTimeline, CommentConversationPanel, AIAssistPanel, SmartFormSection with AI fills) must visually distinguish that content from human-generated content. This distinction is invariant V2 and must survive token changes and regeneration (P11).

**XC3 — EmptyState is required in every region that can be empty**  
No component that renders a list, collection, or data-dependent region may render a blank state. EmptyState (C12) is required. The sole exception is a sub-region inside a component (e.g., a single RelatedGroup within RelatedRecordsSection) — these use a per-group message without a full EmptyState component.

**XC4 — Destructive actions are always isolated**  
In any ActionPanel or BulkActionBar, destructive actions must be in the `destructive-zone` slot and visually separated from the primary action. This is invariant A5 and V3 and must survive any variant.

**XC5 — State before action in every component**  
In any component that renders both state information and actions (RecordHeader, TaskContextPanel, ApprovalPanel), the state indicators (status badge, state label) must precede the action controls in both DOM order and visual rendering (P4, XP6).

**XC6 — Telemetry hooks survive regeneration**  
All telemetry hook keys defined per component in §4 are invariant. Changing a hook key is a breaking change (I7, P10). No regeneration may silently drop a telemetry hook.

**XC7 — Accessibility is a floor for every component**  
Every component meets WCAG 2.1 AA as a minimum. Accessibility properties must survive regeneration (P9, I6). No component may be shipped without the accessibility requirements defined in its spec.

---

## 7. Open seams

| Seam | Connects to | Current status at v0 |
|---|---|---|
| React implementation of each component | Primitive Mapping and Token Strategy v0 (10.7) | Component contracts defined here; React primitives, Radix usage, and CSS token bindings defined in 10.7 |
| `InputField` primitive (used inside SmartFormSection) | 10.7 primitive mapping | Referenced in SmartFormSection.field-group slot; field-level implementation is a 10.7 concern |
| `StatusBadge` primitive (used in RecordHeader, RecordRow, TaskContextPanel) | 10.7 primitive mapping | Used extensively; spec deferred to 10.7 as a primitive-level component |
| `PersonalizationSurface.targetComponentTypes` validation | This library | Component names in 10.4 §4.12 targetComponentTypes must reference names defined here; validation active once this library is complete |
| AI service binding for AIAssistPanel | AI service layer | Hook type and trigger declared in AIAssistHook schema (10.2 §4.13); service response format and error handling defined in AI service layer |
| Personalization settings UI components | Deferred to 10.6 + 10.7 | SmartFormSection and ReviewSummary will be used in the settings surface; pattern choice (RecordPage variant vs AssistedSetupFlow-like) deferred to 10.6 per 10.4 §12 |
| Component-level telemetry infrastructure | Observability / telemetry layer | Hook keys defined here; event schema, routing, and ingestion are outside this workstream |
| `ImpactPreviewPanel` (ExceptionResolutionFlow step) | Future component spec | Referenced in 10.3 F3 output contract; business logic for computing impact preview is domain-specific; deferred from v0 |
| `ConsequencePanel` (ApprovalFlow, ApprovalReviewView) | Future component spec | Referenced in 10.3 F2 and P6 output contracts; simple display component; deferred from v0 |

---

## 8. Milestone 5 exit criteria — self-assessment

| Exit criterion | Status | Evidence |
|---|---|---|
| Semantic components are defined with inputs / states / slots | Met | §4 — all 15 canonical components fully specified with all 10 required properties |
| Overlap and redundancy are reduced | Met | §3 composition matrix makes shared usage explicit; cross-component rules (§6) prevent duplicate action rendering, empty state omission, and AI-content conflation |
| Components are business-aware rather than generic UI widgets | Met | Every component knows about business concepts (approvals, tasks, exceptions, timelines, entities) — none are generic buttons, tables, or cards |
| Component-level personalization hooks stay within pattern constraints | Met | Each component's allowed personalization surfaces reference variant ids from 10.4 §4; no component introduces new variant surfaces not declared in 10.4 |

---

*This document is version 0. It is the first executable version sufficient to drive compiler component selection and slot filling for position app generation. It is expected to be validated against the Golden Workflow Prototypes (10.6) and refined once the Primitive Mapping and Token Strategy (10.7) resolves implementation substrate decisions.*
