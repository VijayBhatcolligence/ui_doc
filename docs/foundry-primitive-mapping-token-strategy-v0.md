# Foundry Primitive Mapping and Token Strategy v0

**Status:** First working draft  
**Workstream:** UI/UX — Work Product 10.7  
**References:** Foundry UX Doctrine v0 (10.1), Foundry Position Projection Schema v0 (10.2), Foundry UI Pattern Library v0 (10.3), Foundry Pattern Variability and Personalization Model v0 (10.4), Foundry Semantic Component Library v0 (10.5), Foundry Golden Workflow Prototypes v0 (10.6), Foundry UI/UX Workstream Charter v1  
**Purpose:** Translate the semantic component contracts from 10.5 into React implementation patterns. Define the token system the compiler targets when generating styles. Specify the two deferred primitives (StatusBadge, InputField). Establish how personalization hooks are implementable without breaking doctrine invariants.

This document sits at the bottom of the grammar stack. It is the bridge from business-language specifications to the React + TypeScript implementation substrate.

```
Semantic Component Library (10.5) ————— defines component contracts in business language
[This document — 10.7] ————— maps those contracts to React + Radix + CSS tokens
Compiler integration (10.8) ————— uses this mapping to generate position app code
```

10.7 does not produce production-ready React code. It produces:
- A primitive selection decision (which Radix primitives to use and why)
- A mapping of each semantic component to a React component structure
- A complete token system with named values
- Implementation strategies for all four personalization variant types (10.4 §2)
- An invariant implementation map ensuring each of the I1–I9 invariants survives to code

---

## 1. Primitive vocabulary

Three levels of primitive exist in the Foundry component system:

| Level | Name | Definition | Example |
|---|---|---|---|
| 1 | **Radix primitive** | Headless, accessible, unstyled component from `@radix-ui/*`. Used where complex accessibility behavior (keyboard, ARIA, focus management) is required. | `@radix-ui/react-dialog` for modals |
| 2 | **Structural primitive** | Custom React component built on top of HTML elements or minimal DOM wrappers. No Radix equivalent; too complex for plain HTML. | `StatusBadge`, `InputField`, `RecordRow` |
| 3 | **Semantic component** | The 10.5 library components. Assembled from Radix primitives + structural primitives + tokens. The compiler targets these by name. | `SmartFormSection`, `ActionPanel` |

**The compiler targets level 3 only.** Radix primitives (level 1) and structural primitives (level 2) are implementation details hidden below the semantic layer. The compiler is not aware of which Radix primitives back a semantic component.

---

## 2. Radix UI primitive selection

### 2.1 Selected Radix primitives

| Radix package | Used for | Components that use it |
|---|---|---|
| `@radix-ui/react-dialog` | Full confirmation modals (`confirmationType=full`), ApprovalFlow rejection step container, AttachmentPanel delete confirmation when the entity needs a full explanation | ActionPanel (full confirm), ApprovalFlow (F2) |
| `@radix-ui/react-alert-dialog` | Destructive bulk action confirmation, any action where `isDestructive=true` and `confirmationType=full` | BulkActionBar (C7), ActionPanel (C5) destructive zone |
| `@radix-ui/react-popover` | Inline confirmation rendering (`confirmationType=inline`) anchored below the triggering action button | ActionPanel (C5) inline confirm |
| `@radix-ui/react-dropdown-menu` | ActionPanel overflow menu, row-level action menus in RecordRow | ActionPanel (C5), RecordRow (structural) |
| `@radix-ui/react-tooltip` | Disabled action tooltips (A6 rule), `at_limit` AttachmentPanel tooltip, field-level hints in InputField | ActionPanel (C5), AttachmentPanel (C11), InputField |
| `@radix-ui/react-select` | SmartFormSection select-type fields, preferred_supplier pre-fill field | InputField (structural) |
| `@radix-ui/react-checkbox` | BulkActionBar select-all, CollectionView row selection checkboxes, filter chip checkboxes (chip-bar variant) | BulkActionBar (C7), FilterRail (C6) |
| `@radix-ui/react-collapsible` | SmartFormSection section collapse/expand toggle, ApprovalPanel `approval_history_placement=collapsible_section` variant | SmartFormSection (C3), ApprovalPanel (C9) |
| `@radix-ui/react-tabs` | ActivityTimeline `timeline_placement=tab` variant, RecordPage multi-section tab navigation | ActivityTimeline (C8) |
| `@radix-ui/react-scroll-area` | Long ActivityTimeline event lists, RelatedRecordsSection group record lists, long SmartFormSection content | ActivityTimeline (C8), RelatedRecordsSection (C10), CommentConversationPanel (C15) |
| `@radix-ui/react-label` | InputField label binding (`htmlFor` association for accessibility) | InputField (structural) |
| `@radix-ui/react-separator` | Section dividers between SmartFormSections, between filter groups in FilterRail, between timeline events | FilterRail (C6), SmartFormSection (C3) |
| `@radix-ui/react-progress` | AttachmentPanel per-file upload progress bar, multi-step flow progress indicator in FlowHeader | AttachmentPanel (C11), FlowHeader (S5) |

### 2.2 Radix primitives NOT selected

| Radix package | Reason not used |
|---|---|
| `@radix-ui/react-navigation-menu` | Foundry navigation is position-app-centric, not global site navigation. Navigation model is defined in 10.2 and is invariant (I1). No Radix nav primitive needed. |
| `@radix-ui/react-radio-group` | Not in the Foundry form vocabulary. Select fields replace radio groups for all single-choice form fields. |
| `@radix-ui/react-slider` | No component in 10.5 requires a slider input. |
| `@radix-ui/react-context-menu` | Right-click context menus are not in the Foundry interaction vocabulary. All actions surface via ActionPanel or row actions (UX Doctrine §6). |
| `@radix-ui/react-menubar` | No menubar pattern exists in 10.3 or 10.5. |
| `@radix-ui/react-accordion` | Collapsible covers all collapse use cases. Accordion implies multiple-simultaneous collapse behavior which is not modeled in Foundry. |
| `@radix-ui/react-toggle-group` | Not needed for v0. Could be revisited for filter chip implementation, but Checkbox handles that use case adequately. |

---

## 3. Structural primitives

These are custom React components required by the system but not available as Radix primitives. Each is specified with its DOM structure, required props, and key accessibility properties.

### SP1. StatusBadge

**Deferred from 10.5 §7 — now fully specified.**

**Purpose:** Renders a small labelled badge with a semantic role. Used in two distinct roles that have different ARIA semantics: entity-type classification and record status.

**Props:**

| Prop | Type | Required | Description |
|---|---|---|---|
| `variant` | enum(`entity-type`, `status`) | Required | Drives ARIA role and color role |
| `label` | string | Required | Display text |
| `semantic` | enum(`neutral`, `positive`, `warning`, `critical`, `inactive`) | Conditional | Required when `variant=status` |
| `entityTypeKey` | string | Conditional | Required when `variant=entity-type`; used to select entity-type color |

**DOM structure:**
```html
<!-- variant=entity-type -->
<span role="note" aria-label="{label} (entity type)" class="foundry-badge foundry-badge--entity-type">
  {label}
</span>

<!-- variant=status -->
<span role="status" aria-label="{label}" class="foundry-badge foundry-badge--status foundry-badge--status-{semantic}">
  {label}
</span>
```

**Token bindings:**

| Property | Token |
|---|---|
| Padding (horizontal) | `--foundry-space-2` |
| Padding (vertical) | `--foundry-space-1` |
| Font size | `--foundry-text-xs` |
| Font weight | `--foundry-weight-medium` |
| Border radius | `--foundry-radius-sm` |
| Background (entity-type) | `--foundry-color-badge-entity-bg` |
| Text (entity-type) | `--foundry-color-badge-entity-text` |
| Background (status) | `--foundry-color-status-{semantic}-bg` |
| Text (status) | `--foundry-color-status-{semantic}-text` |

**Invariant:** The `role="note"` / `role="status"` distinction must survive all token overrides and regeneration (10.5 §7 open seam requirement). These ARIA roles are hardcoded to each variant — they are not configurable via props.

---

### SP2. InputField

**Deferred from 10.5 §7 — now fully specified.**

**Purpose:** A single form input field with label, validation state, inline error, and hint. The atomic building block of SmartFormSection.

**Props:**

| Prop | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | Unique id within the form; used to generate ARIA ids |
| `label` | string | Required | Display label |
| `inputType` | enum(`text`, `textarea`, `number`, `currency`, `date`, `select`) | Required | Controls rendered input element |
| `isRequired` | boolean | Required | `aria-required` and visual required marker |
| `value` | string \| number | Required (runtime) | Controlled value |
| `onChange` | function | Required | Controlled change handler |
| `errorMessage` | string \| null | Required | Null when valid; string drives aria-invalid + error display |
| `hint` | string | Optional | Supporting guidance shown below input; not an error |
| `isReadOnly` | boolean | Required | True for read-only context fields |

**DOM structure:**
```html
<div class="foundry-field-group" data-invalid="{!!errorMessage}">
  <Label htmlFor="{fieldKey}-input">
    {label}
    {isRequired && <span aria-hidden="true" class="foundry-field-required-marker">*</span>}
  </Label>

  <!-- inputType=select uses Radix Select; all others use native input/textarea -->
  <input
    id="{fieldKey}-input"
    aria-required="{isRequired}"
    aria-describedby="{fieldKey}-error {fieldKey}-hint"
    aria-invalid="{!!errorMessage}"
    aria-readonly="{isReadOnly}"
    class="foundry-input"
  />

  {hint && (
    <span id="{fieldKey}-hint" class="foundry-field-hint">{hint}</span>
  )}

  {errorMessage && (
    <span id="{fieldKey}-error" role="alert" class="foundry-field-error">{errorMessage}</span>
  )}
</div>
```

**Token bindings:**

| Property | Token |
|---|---|
| Input border radius | `--foundry-radius-md` |
| Input border color (default) | `--foundry-color-border-default` |
| Input border color (error) | `--foundry-color-status-error` |
| Input border color (focus) | `--foundry-color-border-focus` |
| Focus ring | `2px solid var(--foundry-color-border-focus)` |
| Label font size | `--foundry-text-sm` |
| Input font size | `--foundry-text-base` |
| Error text color | `--foundry-color-status-error-text` |
| Hint text color | `--foundry-color-text-secondary` |
| Spacing (label to input) | `--foundry-space-1` |
| Spacing (input to error/hint) | `--foundry-space-1` |

**Invariant:** `aria-required`, `aria-describedby`, and `aria-invalid` are hardcoded to `isRequired` and `errorMessage` props. They cannot be overridden via token or variant. These are accessibility requirements from 10.5 §7 open seam and must survive regeneration (I6).

---

### SP3. RecordRow

**Purpose:** A single row in a CollectionView record list. Renders entity identity, KeyFactsStrip, status indicator, and row-level actions.

**Props:** `entityTypeRef`, `displayName`, `status`, `statusSemantic`, `keyFacts`, `rowActions`, `isSelected`, `onSelect`

**DOM:** `<li role="row">` containing checkbox (when bulkActionIds present), `<a>` for record navigation, `KeyFactsStrip`, row action button(s).

**Key rules:** Row actions use DropdownMenu when more than one action present. `isSelected` drives `aria-selected` on the `<li>`. Row navigation uses `<a>` with `href` — not an `onClick` button — for correct browser history and accessibility behavior.

---

### SP4. DraftSaveIndicator

**Purpose:** Unobtrusive autosave status indicator inside SmartFormSection. Three states: `saving`, `saved`, `error`.

**DOM:** `<span role="status" aria-live="polite" aria-atomic="true">` — screen reader announces state transitions automatically. Visible text: "Saving…" / "Saved" / "Could not save — retrying". No visual prominence — low hierarchy text below the form section header.

---

### SP5. ActiveFilterIndicator

**Purpose:** Badge inside FilterRail showing the count of active filters. Drives "clear-all" visibility.

**DOM:** `<span aria-live="polite" class="foundry-active-filter-count">` — count change announced politely. Hidden when count is 0.

---

### SP6. SummaryCardGrid

**Purpose:** The SummaryStrip layout when `summary_strip_layout=card_grid` structural variant is active. Renders MetricTile instances in a 3-column responsive grid instead of a horizontal strip.

**DOM:** `<div role="region" aria-label="Summary metrics" class="foundry-summary-card-grid">` containing MetricTile instances.

**Token bindings:** `--foundry-grid-summary-columns: 3`, `--foundry-grid-summary-gap: var(--foundry-space-4)`.

---

### SP7. StepIndicator

**Purpose:** The step progress dots or numbered step list at the top of FlowHeader for multi-step flows.

**DOM:** `<ol aria-label="Form steps" class="foundry-step-indicator">` with `<li aria-current="step">` for the current step.

**Note:** Distinct from StepNavigator (S6). StepIndicator is the compact summary in FlowHeader. StepNavigator is the full sidebar/stepper list used in longer flows.

---

### SP8. ImpactPreviewPanel (deferred stub)

**Purpose:** Referenced in 10.3 F3 ExceptionResolutionFlow output contract. Business logic for computing impact preview is domain-specific. Structural primitive is registered here as a named stub for the compiler to target.

**Status:** Stub only at v0. Full spec deferred pending domain model for exception impact calculation.

---

## 4. Semantic component → React implementation mapping

For each of the 15 canonical components and 7 supporting components from 10.5, this section specifies: the React component name, key Radix primitives used, structural primitives used, root DOM element, and special implementation notes.

### 4.1 Canonical components

| # | Semantic component | React name | Key Radix | Key structural | Root DOM | Notes |
|---|---|---|---|---|---|---|
| C1 | RecordHeader | `<RecordHeader>` | — | StatusBadge (×2), KeyFactsStrip | `<header>` | `displayName` renders as `<h1>`. StatusBadge (status variant) must precede ActionPanel in DOM order — hardcoded in component structure (XC5, P4). |
| C2 | KeyFactsStrip | `<KeyFactsStrip>` | — | — | `<dl>` | Each fact: `<dt>` (label) + `<dd>` (value). `isHighlighted` facts get `aria-label="key fact"`. Max 2 highlighted enforced in component logic. |
| C3 | SmartFormSection | `<SmartFormSection>` | Collapsible | InputField[], DraftSaveIndicator | `<fieldset>` | `<legend>` for section label. Collapsible wraps section body. `autosaveEnabled` activates DraftSaveIndicator. AI-assist slot renders AIAssistPanel when `aiAssistHookIds` present. `additional-content` slot available when `isReadOnly=true` (GW-G3 amendment). |
| C4 | StatusBanner | `<StatusBanner>` | — | — | `<div>` | `role="alert"` for warning/error; `role="status"` for informational/success. Stacking: rendered in a `<div class="foundry-banner-stack">` container; sort order enforced by component (error→warning→success→informational). 8px gap via `--foundry-space-2` between stacked banners. System-replace: `isPersistent=true` banners accept a `onSystemUpdate` prop for severity/message change without unmount (GW-G4 amendment). |
| C5 | ActionPanel | `<ActionPanel>` | DropdownMenu (overflow), Popover (inline confirm), AlertDialog (full confirm), Tooltip (disabled actions) | — | `<div role="group">` | `destructive-zone` slot uses `aria-describedby` to visually separate destructive actions. Budget enforcement: TypeScript prop type limits `secondaryActions.length ≤ 3`. |
| C6 | FilterRail | `<FilterRail>` | Checkbox (chip-bar variant), Separator | ActiveFilterIndicator | `<aside>` | `aria-label="Filters"`. SearchBar always first child (enforced in component structure, not configurable). `filter_placement=chip_bar` structural variant: FilterRail renders as `<div role="toolbar">` instead of `<aside>`. |
| C7 | BulkActionBar | `<BulkActionBar>` | Checkbox (select-all), AlertDialog (destructive confirm) | — | `<div role="toolbar">` | `aria-label="Bulk actions"`. Select-all checkbox: `aria-label="Select all {entityType} records"`. Replaces ActionPanel in page-header slot — mutual exclusion enforced by CollectionView parent component. |
| C8 | ActivityTimeline | `<ActivityTimeline>` | Tabs (`timeline_placement=tab`), ScrollArea | — | `<section>` | Events in `<ol>` (chronological order meaningful). Each event: `<li>` + `<time datetime="...">`. AI events: `aria-label="AI-initiated action"` (V2, XC2). `Tabs` wrapper activated by `timeline_placement=tab` variant. |
| C9 | ApprovalPanel | `<ApprovalPanel>` | Collapsible (`approval_history_placement=collapsible_section`) | — | `<section>` | `aria-label="Approval status"`. Current state badge uses StatusBadge (`variant=status`). Rejection reason: `aria-label="Rejection reason"`. Deadline indicator: `role="alert"` when approaching. |
| C10 | RelatedRecordsSection | `<RelatedRecordsSection>` | ScrollArea | — | `<section>` | Max 3 groups visible by default. "Show more related types" expander for additional groups (10.5 §C10 rule). Inaccessible viewLinks render as `<span>` (non-navigable). Navigable records: `<a>` with `aria-label`. |
| C11 | AttachmentPanel | `<AttachmentPanel>` | AlertDialog (delete confirm), Progress (upload) | — | `<section>` | `aria-label="Attachments"`. Upload area: `role="button"` or `<input type="file">` with `<label>`. `at_limit` state: upload area rendered as `<button disabled aria-describedby="upload-limit-tooltip">` with Tooltip. Progress: one `<Progress>` per in-flight file. |
| C12 | EmptyState | `<EmptyState>` | — | — | `<div>` | `role="alert"` for error context; `role="status"` for positive_confirmation; `role="region" aria-label="..."` for all others. Never blank — enforced by all parent components (XC3). |
| C13 | ValidationSummary | `<ValidationSummary>` | — | — | `<div role="alert">` | Announced immediately on render. Error items: `<a>` elements linking to field id via `href="#{fieldKey}-input"`. Focus moves programmatically to ValidationSummary on blocked submit. |
| C14 | TaskContextPanel | `<TaskContextPanel>` | — | StatusBadge | `<section>` | `aria-label="Task context"`. Task state badge: `variant=status`. Overdue indicator: `aria-label="Task overdue"`. Due date: `<time datetime="...">`. `no_task_instance` state: renders EmptyState (passive — no action path). Aggregate mode (OverviewMonitor): only `active` / `no_task_instance` states rendered (GW-G2 amendment). |
| C15 | CommentConversationPanel | `<CommentConversationPanel>` | ScrollArea | — | `<section>` | `aria-label="Comments"`. Comments in `<ol>`. Compose area: `aria-label="Add a comment"`. Submit: `aria-disabled="true"` when empty. AI-generated comments: visually-hidden `aria-label="AI-generated"` annotation (V2, XC2). AI summary: on-demand only (`on_user_request` trigger — P12). |

### 4.2 Supporting components

| # | Semantic component | React name | Key Radix | Root DOM | Notes |
|---|---|---|---|---|---|
| S1 | SearchBar | `<SearchBar>` | — | `<div role="search">` | `<input aria-label="Search {entityType}">`. Always first in FilterRail — enforced by FilterRail component structure. Matched terms highlighted via `<mark>` elements in results. |
| S2 | AIAssistPanel | `<AIAssistPanel>` | Popover or inline slot | `<aside>` | Suggestion container: `aria-label="AI suggestion"`. Accept and Edit buttons clearly labeled. `affectsBusinessState=true` actions require AlertDialog confirmation before applying (P11). |
| S3 | MetricTile | `<MetricTile>` | — | `<button>` | `role="button"` — navigates to `linkedViewId` on click. `alertThresholdCrossed` drives visual indicator via `data-alert="true"` attribute + token. `aria-label="{label}: {value}{unit}"`. |
| S4 | AlertList | `<AlertList>` | — | `<section>` | `aria-label="Active alerts"`. Alert items in `<ul>`. Ordering: Critical → Warning → Informational — hardcoded in component (not configurable, not personalizable). EmptyState positive_confirmation when empty. |
| S5 | FlowHeader | `<FlowHeader>` | Progress | StepIndicator | `<header>` | Cancel/exit: always visible at every step — enforced in component (P6). StepIndicator rendered only when `totalSteps > 1`. |
| S6 | StepNavigator | `<StepNavigator>` | — | `<nav>` | `aria-label="Form steps"`. Current step: `aria-current="step"`. `section_navigator_style=sidebar_list` (default): renders as `<ol>` in sidebar. `section_navigator_style=top_stepper`: renders as horizontal stepper in page header area. |
| S7 | ReviewSummary | `<ReviewSummary>` | — | `<section>` | `aria-label="Review your entries"`. Sections in `<dl>` per section. Edit links: `<a>` navigating back to step index. Read-only — no inline editing. |

---

## 5. Token system

Token naming convention: `--foundry-{category}-{variant}-{property}` as CSS custom properties. All components reference tokens only — no hardcoded color or spacing values anywhere in component implementations.

### 5.1 Spacing scale

Base unit: **4px**. Scale follows a 4px grid.

| Token | Value | Primary usage |
|---|---|---|
| `--foundry-space-1` | 4px | Minimum gap, badge internal padding, icon-to-label gap |
| `--foundry-space-2` | 8px | Tight spacing, banner-stack gap, field group label-to-input gap |
| `--foundry-space-3` | 12px | Form field internal padding, compact row padding |
| `--foundry-space-4` | 16px | Standard gap, section padding, comfortable row padding |
| `--foundry-space-6` | 24px | Section-to-section gap, card padding |
| `--foundry-space-8` | 32px | Page-level horizontal margins, modal padding |
| `--foundry-space-12` | 48px | Page header vertical padding, flow-header height |
| `--foundry-space-16` | 64px | Major page breaks, hero area padding |

### 5.2 Typography scale

| Token | Size / Line-height | Weight usage | Primary usage |
|---|---|---|---|
| `--foundry-text-xs` | 11px / 1.4 | medium (500) | StatusBadge labels, timestamps, metadata |
| `--foundry-text-sm` | 13px / 1.4 | regular (400) | Field labels, fact labels, secondary content |
| `--foundry-text-base` | 15px / 1.5 | regular (400) | Body text, field values, messages, descriptions |
| `--foundry-text-lg` | 17px / 1.4 | medium (500) | Section headings, prominent field labels |
| `--foundry-text-xl` | 21px / 1.3 | semibold (600) | RecordHeader display names, page titles |
| `--foundry-text-2xl` | 27px / 1.2 | semibold (600) | FlowHeader titles |
| `--foundry-weight-regular` | 400 | Body, labels, descriptive text |
| `--foundry-weight-medium` | 500 | Key facts, section headings, badge text |
| `--foundry-weight-semibold` | 600 | Display names, primary action labels, headings |

**Typography rule (from UX Doctrine §9.1 delta matrix):** The Foundry product position is approachable, not enterprise-formal. No additional type weights beyond these three. No decorative type treatments. All type is set in a single humanist sans-serif system font stack to reduce load weight and maximize OS-native rendering.

**Font stack:**
```css
--foundry-font-sans: -apple-system, BlinkMacSystemFont, "Segoe UI", system-ui, sans-serif;
```

### 5.3 Color roles

Components reference semantic roles only — never palette literals. Palette values live in theme files (§6).

**Surface roles:**

| Token | Semantic meaning |
|---|---|
| `--foundry-color-surface-page` | Main page background |
| `--foundry-color-surface-card` | Card and panel background (elevation-1 surfaces) |
| `--foundry-color-surface-form` | Form field input background |
| `--foundry-color-surface-overlay` | Modal and dialog backdrop |
| `--foundry-color-surface-hover` | Row and item hover state |
| `--foundry-color-surface-selected` | Row and item selected state |

**Border roles:**

| Token | Semantic meaning |
|---|---|
| `--foundry-color-border-default` | Standard border on cards, inputs, separators |
| `--foundry-color-border-focus` | Focus ring on interactive elements |
| `--foundry-color-border-error` | Input border in error state |

**Text roles:**

| Token | Semantic meaning |
|---|---|
| `--foundry-color-text-primary` | Body text, headings |
| `--foundry-color-text-secondary` | Labels, metadata, placeholder |
| `--foundry-color-text-disabled` | Disabled element text |
| `--foundry-color-text-on-action` | Text on primary action button (contrast: ≥4.5:1) |
| `--foundry-color-text-link` | Navigable record links, action links |

**Action roles:**

| Token | Semantic meaning |
|---|---|
| `--foundry-color-action-primary` | Primary action button fill |
| `--foundry-color-action-primary-hover` | Primary action hover state |
| `--foundry-color-action-primary-active` | Primary action pressed state |
| `--foundry-color-action-secondary` | Secondary action button border/fill |
| `--foundry-color-action-secondary-hover` | Secondary action hover state |
| `--foundry-color-action-destructive` | Destructive action color (isolated zone — A5) |

**Status roles (used by StatusBanner, StatusBadge, AlertList, ExceptionResolutionView):**

| Token | Semantic meaning |
|---|---|
| `--foundry-color-status-informational-bg` | Informational banner/badge background |
| `--foundry-color-status-informational-text` | Informational banner/badge text |
| `--foundry-color-status-informational-border` | Informational banner/badge border |
| `--foundry-color-status-warning-bg` | Warning severity |
| `--foundry-color-status-warning-text` | Warning severity text |
| `--foundry-color-status-warning-border` | Warning severity border |
| `--foundry-color-status-error-bg` | Error/critical severity |
| `--foundry-color-status-error-text` | Error/critical severity text |
| `--foundry-color-status-error-border` | Error/critical severity border |
| `--foundry-color-status-success-bg` | Success/positive severity |
| `--foundry-color-status-success-text` | Success/positive severity text |
| `--foundry-color-status-success-border` | Success/positive severity border |
| `--foundry-color-status-neutral-bg` | Neutral status state |
| `--foundry-color-status-neutral-text` | Neutral status text |
| `--foundry-color-status-inactive-bg` | Inactive status state |
| `--foundry-color-status-inactive-text` | Inactive status text |

**AI content distinction role (invariant V2, P11, XC2):**

| Token | Semantic meaning |
|---|---|
| `--foundry-color-ai-indicator-bg` | Background tint for AI-generated content |
| `--foundry-color-ai-indicator-border` | Left-border accent for AI-generated content |
| `--foundry-color-ai-indicator-label` | Color of "AI" label pill |

**Constraint:** The AI indicator tokens must not be overridden by tenant theme configuration. AI-generated content must always be visually distinguishable regardless of theme. The compiler must treat these tokens as invariant (I6, XC2, P11). They are exempt from tenant color overrides.

### 5.4 Elevation and border radius

| Token | Value | Usage |
|---|---|---|
| `--foundry-elevation-0` | none | Flat surfaces — page background, table rows, inline form areas |
| `--foundry-elevation-1` | 0 1px 3px rgba(0,0,0,0.08) | Cards, SmartFormSection, FilterRail panels |
| `--foundry-elevation-2` | 0 4px 8px rgba(0,0,0,0.10) | Popovers (inline confirm), DropdownMenu, Tooltip |
| `--foundry-elevation-3` | 0 8px 16px rgba(0,0,0,0.14) | Dialog overlays (AlertDialog, Dialog) |
| `--foundry-radius-sm` | 4px | StatusBadge, filter chips, tags |
| `--foundry-radius-md` | 6px | InputField, buttons, row actions |
| `--foundry-radius-lg` | 8px | Cards, SmartFormSection, FilterRail, panel components |
| `--foundry-radius-xl` | 12px | Modals, dialogs, full-screen flow containers |

### 5.5 Motion tokens

Motion is functional. It orients the user (showing that a panel is accessible via back navigation, confirming a save operation, indicating progress) rather than providing decoration. No motion should delay interaction — all animations are interruptible.

| Token | Value | Usage |
|---|---|---|
| `--foundry-duration-fast` | 100ms | Hover states, icon changes, badge color transitions |
| `--foundry-duration-default` | 200ms | Expand/collapse (Collapsible), tooltip appear, inline confirm show |
| `--foundry-duration-slow` | 300ms | Dialog entrance, page-level transitions, flow step transitions |
| `--foundry-easing-default` | cubic-bezier(0.4, 0, 0.2, 1) | General transitions |
| `--foundry-easing-enter` | cubic-bezier(0, 0, 0.2, 1) | Elements entering the viewport |
| `--foundry-easing-exit` | cubic-bezier(0.4, 0, 1, 1) | Elements leaving the viewport |

**Reduced motion:** All transitions respond to `prefers-reduced-motion: reduce`. When active, all `--foundry-duration-*` tokens collapse to `0ms` except focus indicators. Focus ring transitions are preserved (1ms) so keyboard users always see clear focus state.

```css
@media (prefers-reduced-motion: reduce) {
  :root {
    --foundry-duration-fast: 0ms;
    --foundry-duration-default: 0ms;
    --foundry-duration-slow: 0ms;
  }
}
```

---

## 6. Density mode implementation

Density mode (UX Doctrine D1–D3, 10.4 §5.2) is implemented as a CSS-only approach. No JavaScript re-render is required to switch density.

### 6.1 Two density presets

| Mode | Data attribute | Description |
|---|---|---|
| `comfortable` (default) | `data-density="comfortable"` | Full spacing scale — consumer-grade baseline (D1) |
| `compact` | `data-density="compact"` | 75% of spacing scale — higher information density for power users |

### 6.2 Implementation mechanism

A single density multiplier CSS custom property controls all density-responsive spacing:

```css
:root[data-density="comfortable"] {
  --foundry-density-multiplier: 1;
}

:root[data-density="compact"] {
  --foundry-density-multiplier: 0.75;
}
```

Components that respond to density use `calc()` for their spacing:

```css
.foundry-record-row {
  padding: calc(var(--foundry-space-4) * var(--foundry-density-multiplier));
  gap: calc(var(--foundry-space-2) * var(--foundry-density-multiplier));
}
```

Components that do not respond to density (interactive target sizes, badge padding, modal padding) use the absolute space tokens directly, without the multiplier.

### 6.3 Density invariants

The following properties must NOT be scaled by the density multiplier (UX Doctrine D3):

| Property | Reason |
|---|---|
| Typography size | Text must remain legible at compact density. Font size tokens are fixed. |
| Interactive touch/click target minimum | WCAG minimum touch target 44×44px must hold in compact mode (I6, P9). |
| Focus ring width | Keyboard accessibility floor — focus ring must remain visually clear (I6). |
| Modal padding | Dialogs are elevated surfaces where density reduction would feel claustrophobic. |
| AI content distinction | `--foundry-color-ai-indicator-*` tokens must not be scaled or removed by density (V2, XC2). |

### 6.4 Density resolution path

Density mode is a token-driven variant (10.4 §2.1). Resolution follows the four-scope precedence:

1. Global default: `comfortable`
2. Tenant config: may set a default different from global
3. Role default: may override tenant for a specific position
4. User preference: sets `density_mode` in the preference store → applies `data-density` attribute on `<html>` at session load

The compiler writes the initial `data-density` value based on the resolved scope. The client-side preference store may update it during the session if the user changes density preference.

---

## 7. Theming architecture

A theme is a named set of palette values assigned to the semantic color roles defined in §5.3. The token system exposes only roles; themes provide palette values.

### 7.1 Theme structure

```css
/* foundry-light.css — default theme */
:root[data-theme="foundry-light"] {
  --foundry-color-surface-page: #FAFAFA;
  --foundry-color-surface-card: #FFFFFF;
  --foundry-color-surface-form: #FFFFFF;
  --foundry-color-border-default: #E2E8F0;
  --foundry-color-border-focus: #3B82F6;
  --foundry-color-text-primary: #1A202C;
  --foundry-color-text-secondary: #718096;
  --foundry-color-action-primary: #2563EB;
  --foundry-color-action-primary-hover: #1D4ED8;
  --foundry-color-text-on-action: #FFFFFF;
  --foundry-color-status-error-bg: #FEF2F2;
  --foundry-color-status-error-text: #DC2626;
  --foundry-color-status-error-border: #FECACA;
  --foundry-color-status-warning-bg: #FFFBEB;
  --foundry-color-status-warning-text: #D97706;
  --foundry-color-status-warning-border: #FDE68A;
  --foundry-color-status-success-bg: #F0FDF4;
  --foundry-color-status-success-text: #16A34A;
  --foundry-color-status-success-border: #BBF7D0;
  --foundry-color-status-informational-bg: #EFF6FF;
  --foundry-color-status-informational-text: #2563EB;
  --foundry-color-status-informational-border: #BFDBFE;
  /* AI indicator — INVARIANT: cannot be overridden by tenant theme */
  --foundry-color-ai-indicator-bg: #F5F3FF;
  --foundry-color-ai-indicator-border: #7C3AED;
  --foundry-color-ai-indicator-label: #6D28D9;
}
```

### 7.2 Theme application

Themes are applied via `data-theme` attribute on `<html>`:

```html
<html data-theme="foundry-light" data-density="comfortable" lang="en">
```

Both attributes may be changed at runtime without a page reload — purely CSS-driven.

### 7.3 Dark theme

A `foundry-dark` theme is a required deliverable for production but is **deferred from v0**. The dark theme color palette is not specified here. The token architecture is ready for it — dark theme provides an alternate set of palette values for all roles in §5.3. The AI indicator tokens remain fixed even in dark theme.

### 7.4 Tenant color customization

Tenants may provide a custom color override file that sets a subset of action and brand color roles:

- Permitted overrides: `--foundry-color-action-primary`, `--foundry-color-action-primary-hover`, `--foundry-color-border-focus` (brand accent)
- Prohibited overrides: all status role tokens, all AI indicator tokens, all text contrast tokens
- The prohibition on status overrides prevents a tenant from inadvertently making error/warning states indistinguishable (contrast requirement under I6/P9)

---

## 8. Personalization hook implementation

10.4 §2 defines four variation types. Each type maps to a distinct implementation mechanism.

### 8.1 Token-driven variants

**What changes:** Visual appearance only — theme, density feel, color emphasis. No element count or structure change.

**Implementation:**
- `density_mode` → `data-density="comfortable|compact"` attribute on `<html>`. Pure CSS.
- `theme` → `data-theme="foundry-light|foundry-dark"` attribute on `<html>`. Pure CSS.
- No React re-render required for token-driven changes. The compiler writes the initial attribute values. The client updates them if the user changes preferences.

**Invariant:** Token-driven changes must not affect: element count, ARIA roles, telemetry hook keys, AI content distinction, or action semantics. These are verified by the compiler at generation time (I6, I7).

### 8.2 View-state variants

**What changes:** Saved operational state — active filters, sort order, section expansion, pinned views.

**Implementation:**
- Persisted to a client-side preference store keyed by `{tenantId}:{positionId}:{viewId}` (10.4 §6.1 compound key)
- Loaded on view mount; applied as initial prop values to FilterRail (`activeFilters`), RecordRow column visibility, SmartFormSection `isCollapsedByDefault`
- No IR impact — purely runtime state, not compiled into the component tree
- Orphan detection: on view mount, the preference store is validated against the current ViewSpec. Keys referencing removed fields or filters are discarded with an orphan log record (10.4 §7.1)

### 8.3 Emphasis variants

**What changes:** Component prop values — which content is prominent, information depth, slot selection.

**Implementation:**
- Resolved at compile time (for role-scoped emphasis) or at session load (for user-scoped emphasis)
- Written into generated component props: `<KeyFactsStrip maxFacts={5} />` (from `key_fact_count=5`)
- No CSS changes required — all prop-driven
- The compiler reads resolved variant values from the PositionProjection and emits them as static prop values in the generated component tree

### 8.4 Structural variants

**What changes:** The IR — a different component tree is generated.

**Implementation:**
- Compiler generates different React component trees per approved structural variant
- The active variant value is resolved before code generation
- Example: `form_layout=stepped_multi_section` → CreateEditFlow renders `<StepNavigator>` + multiple `<SmartFormSection>` instances; `form_layout=single_page` → single `<SmartFormSection>` without `<StepNavigator>`
- IR transformation table from 10.4 §4.11 maps each structural variant to exact component tree changes

**Governance gate:** Structural variants are tenant/role-scoped only (never user-scoped). The compiler must validate that the resolved value has passed the governance gate (10.4 §5.3) before generating the structural variant tree. An unapproved structural variant value is a compile error.

### 8.5 Personalization resolution order in the compiler

The compiler resolves personalization in this order before generating any view:

```
1. Read allowedSurfaces from PersonalizationHooks (10.2 §4.15)
2. For each surface in allowedSurfaces:
   a. Check if structural variant → validate governance gate first
   b. Collect values from all four scopes (global → tenant → role → user)
   c. Apply conflictPolicy (role_overrides_user OR user_overrides_role)
   d. If user value cannot be applied (out of scope, governance failure, orphaned) → log orphan record; use lower-scope value
3. Emit resolved values as component props (emphasis) or data attributes (token-driven) or IR branch (structural)
```

---

## 9. Invariant implementation map

Each of the nine regeneration invariants from UX Doctrine §8 (I1–I9) must be implemented as a concrete constraint at the React layer.

| Invariant | Implementation constraint |
|---|---|
| **I1 — Navigation structure** | Navigation is driven from `workSurfaceComposition.navigationConfig` in the PositionProjection. The nav component is generated once per position app. Re-generation only changes nav when the projection's navigation config changes. Nav is not configurable via React props at runtime — it is compiled-in. |
| **I2 — Primary action semantics** | ActionSpec `actionId` and `label` are stable schema values. The compiler emits them as static strings. No runtime override mechanism exists for action labels. Changing an action label is a schema change (I9 coordinate). |
| **I3 — State labels** | StatusBadge `label` prop values come from the entity's `stateModel` in the schema. State labels are schema-stable strings. No runtime label override. Synonyms are rejected at schema level. |
| **I4 — Pattern grammar** | The generated component tree's root pattern is fixed by `patternType` in ViewSpec. The compiler never changes a pattern type silently. A `patternType` change is a breaking migration (emit migration record). |
| **I5 — Personalization data** | View-state preferences are keyed by `{tenantId}:{positionId}:{viewId}`. Regeneration does not touch the preference store. Orphaned preferences are detected and logged on mount; they are discarded, not silently applied. |
| **I6 — Accessibility** | Accessibility properties (ARIA roles, `aria-required`, `aria-describedby`, `aria-invalid`, `role="alert"`, focus management) are hardcoded in structural primitives (StatusBadge, InputField) and semantic components. They are not configurable via token or variant. The compiler cannot emit a component without them. Verified by automated accessibility test suite post-generation. |
| **I7 — Telemetry hook keys** | All telemetry hook keys (e.g., `action_initiated`, `banner_shown`, `filter_applied`) are exported as named TypeScript constants from each component file. They are never derived from props. Changing a constant is a TypeScript breaking change — caught at compile time. |
| **I8 — Action budget compliance** | ActionPanel TypeScript prop type enforces: `secondaryActions.length` max 3 (compile-time type error), `primaryAction` max 1 field (not array). Budget is enforced at the type level, not just convention. Automated audit of compiled component trees checks total visible action count post-generation. |
| **I9 — Cross-position semantic consistency** | `actionId` and `label` values are sourced from a shared ActionSpec registry in the schema. The same `actionId` always resolves to the same label across all position apps. The compiler validates cross-position consistency as part of the schema validation step — not at the React layer. |

---

## 10. Open seams

| Seam | Connects to | Status at v0 |
|---|---|---|
| Dark theme palette | Foundry brand/design system | Token architecture is ready. `foundry-dark` palette values are not specified. Required before production launch. |
| Icon system | Foundry design system / icon library | All components reference icons by semantic name (e.g., `icon.severity.warning`, `icon.action.dismiss`). The actual icon files and the icon component are not specified here. An icon vocabulary document is needed before React implementation begins. |
| AI rendering details | AI service layer | AIAssistPanel contract is specified. The visual design of the suggestion UI (inline suggestion vs panel vs field overlay) requires design validation. AI service response schema is outside this workstream. |
| Mobile-responsive token overrides | Responsive breakpoint spec | Density rule D4 scopes mobile to monitoring, approval, and assisted workflows. Breakpoints and responsive token adjustments for `max-width: 768px` are not specified at v0. |
| Accessibility test harness | CI/CD pipeline | WCAG 2.1 AA compliance is required (P9, I6) and must be verified post-generation. Integration with axe-core or equivalent automated accessibility testing is not specified here — it is a CI/CD pipeline concern. |
| Animation library selection | Frontend build toolchain | Motion tokens are defined (§5.5). No specific animation library is mandated — CSS transitions cover the use cases at v0. Framer Motion or React Spring may be adopted later for more complex flow-step transitions. |
| Compiler-generated data attributes | Compiler integration (10.8) | The compiler needs to emit `data-density` and `data-theme` attributes on `<html>`. The exact mechanism for this in the compiler IR is outside this document — it is a 10.8 concern. |
| SP8 ImpactPreviewPanel | Domain model for exception impact | Registered as a named stub (§3 SP8). Full spec requires domain-specific exception impact calculation logic from the business model layer. |

---

## 11. Milestone 7 exit criteria — self-assessment

| Exit criterion | Status | Evidence |
|---|---|---|
| Semantic components can be mapped to React implementation patterns | Met | §4 — all 15 canonical components and 7 supporting components mapped. Root DOM, key Radix primitives, structural primitives, and special invariant notes specified for each. |
| Required primitives are known | Met | §2 — 13 Radix primitives selected with rationale. §3 — 8 structural primitives specified (SP1–SP8). StatusBadge and InputField fully specified (deferred from 10.5). |
| Token/style direction is compatible with the UX doctrine | Met | §5 — spacing scale (4px base), typography scale, and color role taxonomy all consistent with D1–D5 (density rules) and §9.1 delta matrix (approachable, not enterprise-formal). AI indicator tokens marked as invariant (V2, P11, XC2). §6 — density mode implemented as CSS-only, preserving all D3 constraints. |
| Personalization hooks are implementable without breaking invariants | Met | §8 — all four variant types have implementation strategies. Token-driven variants use CSS-only data attributes. Emphasis variants use compiled static props. Structural variants use IR branching with governance gate. View-state uses preference store with orphan detection. §9 — all I1–I9 invariants have explicit React-layer implementation constraints. |

---

*This document is version 0. It is the implementation substrate specification sufficient to begin React component development and compiler IR integration. It is expected to be updated when: the dark theme palette is finalized, the icon vocabulary is defined, mobile breakpoints are specified, and the AI rendering design is validated. The 10.8 compiler integration handoff will consume this document alongside 10.1–10.6 to define the complete compiler contract.*
