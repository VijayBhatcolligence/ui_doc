# Foundry Position Projection Schema v0

**Status:** First working draft  
**Workstream:** UI/UX — Work Product 10.2  
**References:** Foundry UX Doctrine v0, Foundry UI/UX Workstream Charter v1, Foundry2 Position-Centric Regenerative Software for SMBs v1  
**Purpose:** Define the UI-facing schema that the compiler must resolve to generate a position app.

This schema is the contract between the model/compiler layer and the UI generation layer. It describes exactly what information a position app needs from the model to render its views, actions, and interactions correctly.

This schema is **not** a database schema, not a backend service contract, and not a full business model. It is narrowly scoped to what the UI generator needs. Stability of this schema is a first-class goal.

---

## 1. Purpose and position in the compiler flow

The Position Projection Schema sits between the semantic normalization step and the UI generation step.

```
progressive semantic slice
    -> structural validation
    -> policy validation
    -> normalization
    -> [PositionProjection resolution]  ← this schema defines this layer
    -> UI pattern and variant resolution
    -> React position app generation
```

The compiler resolves a validated semantic slice into a `PositionProjection` object. The UI generator consumes this object to select patterns, compose views, configure personalization surfaces, and wire up action and permission boundaries.

**Schema design goals:**
- Narrow enough to remain stable across minor semantic changes
- Complete enough to drive all UI pattern selection decisions without further model queries
- Decoupled from backend service contracts, policy internals, and provenance records
- Explicit about open seams where policy, provenance, and AI integration will connect later

---

## 2. Schema architecture principles

Two architectural rules govern the entire schema. All sections must respect both.

### 2.1 Core sections vs extension sections

The schema has two tiers. **Core sections** are required for any position app. **Extension sections** are conditionally present based on what kind of work the position does. Extension sections that are absent produce a correct empty or hidden state — they must not cause a pattern failure.

| Tier | When present |
|---|---|
| Core | Always — required for every generated position app |
| Extension | Only when the position's semantic slice includes that work type |

### 2.2 Static schema vs runtime data

This schema is a **compiled, static artifact**. It describes types, structures, and rules — not live instances or resolved values. Where a field must exist at schema level but is only resolved at runtime, it is explicitly marked `(runtime-resolved)`.

| Schema level (this document) | Runtime level (not in this document) |
|---|---|
| EntityTypeRef — entity type + display template | Live entity instance with id and resolved display name |
| AlertSpec — alert type + body template | Active alert instance with resolved values |
| FilterSpec — field key + description of default | Applied filter with resolved value for a specific user |
| TaskSpec — task type + state model | Active task instance with current state and assignee |

Blurring this boundary is a schema defect. If a field feels like a runtime value, it should be a template, a type reference, or a seam — not a resolved value.

---

## 3. Top-level schema structure

| Section | Tier | Type | Required | Purpose |
|---|---|---|---|---|
| `identity` | Core | PositionIdentity | Required | Who this position is and its organizational context |
| `workSurface` | Core | WorkSurfaceComposition | Required | Navigation structure, view registry, and pattern bindings |
| `actions` | Core | list\<ActionSpec\> | Required | Full action vocabulary for this position |
| `permissionsHooks` | Core | PermissionsHooks | Required | Permission scopes and boundaries |
| `defaultViewHints` | Core | DefaultViewHints | Required | Generator-time defaults for layout, sort, and filter |
| `personalizationHooks` | Core | PersonalizationHooks | Required | Personalization surfaces and scopes |
| `tasks` | Extension | list\<TaskSpec\> | Optional | Discrete work items — present when position handles task-oriented work |
| `decisions` | Extension | list\<DecisionSpec\> | Optional | Structured choice points — present when position makes explicit decisions |
| `monitoredEntities` | Extension | list\<MonitoredEntitySpec\> | Optional | Watched entities — present when position has monitoring responsibility |
| `alerts` | Extension | list\<AlertSpec\> | Optional | Events and exceptions — present when position receives exception alerts |
| `approvals` | Extension | list\<ApprovalSpec\> | Optional | Approval workflows — present when position initiates or reviews approvals |
| `collaborators` | Extension | list\<CollaboratorSpec\> | Optional | Hand-off relationships — present when position has explicit cross-position flows |
| `stateSummaries` | Extension | list\<StateSummarySpec\> | Optional | Overview indicators — present when position has an OverviewMonitor view |
| `aiAssistHooks` | Extension | list\<AIAssistHook\> | Optional | AI assistance — present when position has AI-assisted workflows |

**Schema-level rules:**

- Every `PositionProjection` must include all Core sections.
- Extension sections that are absent must not cause a UI pattern failure. An absent extension section produces a correct empty or hidden state.
- The schema must be fully resolvable from a validated semantic slice without querying live backend services.
- The `PositionProjection` is versioned. See §6 for versioning rules.

---

## 4. Section specifications

---

### 4.1 PositionIdentity

Identifies and scopes the position. These fields are stable invariants — they do not change between minor regenerations.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `positionId` | id | Required | Unique identifier for this position | Invariant across regenerations |
| `positionTitle` | string | Required | Display name of the position | Shown in app header and navigation |
| `positionSlug` | string | Required | URL-safe identifier | Lowercase, hyphenated; invariant |
| `tenantId` | id | Required | The company this position belongs to | |
| `boundedContextId` | id | Required | The bounded context that owns this position's primary data | Seam: bounded context definition lives in semantic layer |
| `orgUnit` | string | Optional | The org unit or department | Display only; not a routing key |
| `positionVersion` | string | Required | Schema version of this projection | Semver format; major bump = breaking change |

**Rules:**
- `positionId` and `positionSlug` are invariants. They must not change when the position is regenerated.
- `positionVersion` must be incremented whenever a field marked as invariant changes.

---

### 4.2 WorkSurfaceComposition

WorkSurfaceComposition is split into two distinct sub-concerns: **NavigationSpec** (how the user moves between views) and **ViewRegistry** (what each view is and what it provides to its pattern). These are kept as sub-sections of `workSurface` but are independently owned.

**WorkSurfaceComposition fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `navigation` | NavigationSpec | Required | App-level navigation structure |
| `views` | list\<ViewSpec\> | Required | View definitions and pattern bindings |

---

#### 4.2a NavigationSpec

Owns navigation structure only. Navigation concerns (order, labels, icons, landing) are fully contained here and do not bleed into ViewSpec.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `defaultLandingViewId` | id | Required | The view shown on first load | Must reference a valid `viewId` in `views` |
| `navItems` | list\<NavItem\> | Required | All navigation entries | Min 1 |
| `navStyle` | enum(sidebar, topbar, tabs) | Optional | Navigation presentation hint | Must be an approved variant; default is sidebar |

**NavItem:**

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `viewId` | id | Required | References a ViewSpec in the view registry | |
| `navLabel` | string | Required | Display label shown in navigation | |
| `navOrder` | number | Required | Position in nav list | 1-indexed; unique |
| `navIcon` | string | Optional | Icon identifier | Must reference approved icon vocabulary |
| `isHiddenFromNav` | boolean | Optional | Reachable but not in main nav | Default: false |

---

#### 4.2b ViewRegistry — ViewSpec

Each ViewSpec owns its pattern binding and layout hints. Navigation concerns have moved to NavigationSpec above.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `viewId` | id | Required | Unique view identifier within this projection | Invariant across regenerations |
| `pattern` | enum | Required | The UI pattern this view maps to | See pattern vocabulary below |
| `patternVariant` | string | Optional | An approved variant name | Must reference an approved variant; null = default |
| `primaryEntityType` | string | Optional | The entity type this view centers on | Seam: entity definitions live in semantic layer |
| `patternInputBinding` | PatternInputBinding | Required | What this view provides to its pattern | See §4.2c below |
| `layoutHints` | LayoutHints | Optional | How content is grouped within this view | See §4.2d below |
| `accessConditionHookId` | string | Optional | ConditionalAccessHook id controlling visibility | Seam: evaluated at runtime by permissions layer |

**Pattern vocabulary for `pattern` field:**

Page patterns: `CollectionView`, `RecordPage`, `OverviewMonitor`, `SearchResults`, `ExceptionResolutionView`, `ApprovalReviewView`

Flow patterns: `CreateEditFlow`, `ApprovalFlow`, `ExceptionResolutionFlow`, `AssistedSetupFlow`

**ViewSpec rules:**
- `viewId` is invariant. Changing a view's `pattern` is a breaking change (UX Doctrine I4).
- Each pattern value must correspond to a pattern defined in UI Pattern Library v0 (work product 10.3).

---

#### 4.2c PatternInputBinding

Declares what this view provides to its pattern. This is the per-view side of the pattern contract. The authoritative input requirements per pattern type are defined in UI Pattern Library v0 (10.3) — this binding is validated against those requirements at compile time.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `patternType` | enum | Required | Must match the ViewSpec's `pattern` | Must be identical to parent ViewSpec pattern |
| `entityTypeRef` | EntityTypeRef | Optional | Primary entity for this view | See EntityTypeRef in §4.3 |
| `listSourceDescription` | string | Optional | Plain-language description of the data source | Required for list-based patterns |
| `filterSources` | list\<FilterSourceSpec\> | Optional | Available filter dimensions for this view | |
| `actionSlots` | ActionSlotBinding | Optional | Which actions go in which slots | Enforces action budget at schema level |
| `aiAssistHookIds` | list\<id\> | Optional | AI hooks available in this view | References `aiAssistHooks` section |

**ActionSlotBinding** (enforces UX Doctrine §6 action-budget rules at schema level):

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `primaryActionId` | id | Optional | The single primary action | Max 1 — UX Doctrine P1, A1 |
| `secondaryActionIds` | list\<id\> | Optional | Visible secondary actions | Max 3 — UX Doctrine A2 |
| `overflowActionIds` | list\<id\> | Optional | Actions placed in overflow menu | Only when total > 5 — UX Doctrine A3 |
| `bulkActionIds` | list\<id\> | Optional | Actions available in bulk-select state | UX Doctrine A4 |

**FilterSourceSpec:**

| Field | Type | Required | Description |
|---|---|---|---|
| `filterKey` | string | Required | Unique filter identifier |
| `filterLabel` | string | Required | Display label |
| `filterType` | enum(text, select, date\_range, status, boolean) | Required | Filter input type |
| `isDefaultVisible` | boolean | Required | Shown by default or behind expansion |

**Composition rule for ActionSlotBinding:** `primaryActionIds` (1) + `secondaryActionIds` (max 3) + `overflowActionIds` must satisfy: total visible = 1 + secondary count ≤ 5. Any action beyond 5 visible goes to overflow. This is validated at compile time (see C8).

> **Seam note:** The complete required vs optional input contract per pattern type (CollectionView needs X, RecordPage needs Y) is defined in UI Pattern Library v0 (10.3). PatternInputBinding is the per-view declaration that 10.3's contracts validate against.

---

#### 4.2d LayoutHints

Declares how content is grouped within this view. Optional — if absent, the pattern's default layout applies.

| Field | Type | Required | Description |
|---|---|---|---|
| `primaryRegion` | string | Optional | Semantic label for the main content area |
| `sidebarRegion` | string | Optional | Semantic label for sidebar content |
| `headerRegion` | string | Optional | What appears in the record or page header |
| `groupedSections` | list\<SectionGroup\> | Optional | How form or detail sections are grouped |

**SectionGroup:**

| Field | Type | Required | Description |
|---|---|---|---|
| `groupKey` | string | Required | Unique group identifier |
| `groupLabel` | string | Required | Display label for this group |
| `sectionKeys` | list\<string\> | Required | Ordered section keys within this group |
| `isCollapsedByDefault` | boolean | Optional | Default collapse state; default: false |

---

---

### 4.3 Shared sub-types

These types are reused across multiple sections. They are defined once here.

#### EntityTypeRef

`EntityTypeRef` is a **schema-level type reference** — not an instance reference. It names an entity type and provides a display template. Instance ids and resolved display values are runtime data and do not belong in this schema.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `entityType` | string | Required | The entity type being referenced | Seam: entity model lives in semantic layer |
| `displayNameTemplate` | string | Required | Template string for display (e.g., `{{name}} — {{status}}`) | Resolved at runtime; not a static value |
| `viewLink` | id | Optional | The viewId in this projection that shows this entity's detail | |

> **Static/runtime rule:** Do not include `entityId` or resolved display strings in an `EntityTypeRef`. Those are runtime values resolved by the UI from live backend queries, not schema declarations.

**viewLink accessibility rule:**  
`viewLink` is optional. When present, the generator renders it as a navigable link. When the linked view is inaccessible to the current user (either because the `viewId` is absent from this projection, or because its `accessConditionHookId` evaluates to denied at runtime), the generator must render the entity reference as a **plain non-navigable label** — not a broken link, not a hidden element, and not an error state. The entity name remains visible; only the navigation affordance is suppressed. This rule applies wherever `EntityTypeRef.viewLink` appears in the rendered UI.

#### InputFieldSpec

Reused in ActionSpec and DecisionSpec.

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | Unique field identifier |
| `fieldLabel` | string | Required | Display label |
| `inputType` | enum(text, number, currency, date, select, multiselect, textarea, file) | Required | Input type |
| `isRequired` | boolean | Required | Whether mandatory |
| `validationRules` | list\<string\> | Optional | Plain-language validation descriptions |

---

### 4.4 TaskSpec

A task is a discrete, bounded unit of work assigned to or owned by this position.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `taskSpecId` | id | Required | Unique identifier for this task type | |
| `taskType` | string | Required | Business classification (e.g., `InvoiceReview`, `OnboardingStep`) | Must match vocabulary in semantic slice |
| `titleTemplate` | string | Required | Plain-language task title template | May include entity name placeholder |
| `entityTypeRef` | EntityTypeRef | Optional | The entity type this task is about | See §4.3 |
| `stateModel` | list\<string\> | Required | The valid state values for this task type | e.g., `[open, in_progress, blocked, completed, cancelled]` |
| `priorityLevels` | list\<string\> | Required | The valid priority values | e.g., `[low, normal, high, urgent]` |
| `hasDueDate` | boolean | Required | Whether this task type carries a due date | |
| `availableActionIds` | list\<id\> | Optional | ActionSpec ids that apply to tasks of this type | References `actions` section |
| `aiAssistHookIds` | list\<id\> | Optional | AI assist hook ids available for this task type | References `aiAssistHooks` section |

**Rules:**
- `availableActionIds` must only reference actions defined in the `actions` section of the same projection.
- TaskSpec is schema-level: it describes what kinds of tasks this position handles. Live task instances are runtime data, not schema data.

---

### 4.5 DecisionSpec

A decision is a choice point where the position holder selects from defined options with concrete business consequences. Decisions differ from approvals: an approval is binary (approve/reject); a decision selects from a defined option set and may trigger different downstream actions per option.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `decisionSpecId` | id | Required | Unique identifier | |
| `decisionType` | string | Required | Business classification (e.g., `SupplierSelection`, `PricingException`) | |
| `titleTemplate` | string | Required | Plain-language description of what must be decided | |
| `contextFields` | list\<ContextField\> | Required | Information the user needs to see before deciding | Min 1 |
| `options` | list\<DecisionOption\> | Required | Available choices | Min 2 |
| `hasDueDate` | boolean | Required | Whether this decision type has a deadline | |
| `entityTypeRef` | EntityTypeRef | Optional | The entity type this decision concerns | See §4.3 |
| `linkedActionIds` | list\<id\> | Optional | Actions that become available or are triggered after this decision | References `actions` section |
| `postDecisionBehavior` | enum(stay, navigateTo, dismiss) | Required | UI behavior after the decision is made | Aligns with ActionSpec.postActionBehavior |
| `postDecisionNavigateToViewId` | id | Optional | Target viewId if `postDecisionBehavior=navigateTo` | |
| `aiAssistHookIds` | list\<id\> | Optional | AI assist hooks for decision support | |

**ContextField:**

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | Data field identifier |
| `fieldLabel` | string | Required | Display label |
| `displayFormat` | enum(text, number, currency, date, status, badge) | Required | How to render this value |
| `isHighlighted` | boolean | Optional | Whether this field receives visual emphasis |

**DecisionOption:**

| Field | Type | Required | Description |
|---|---|---|---|
| `optionId` | id | Required | Unique identifier within this decision |
| `label` | string | Required | Display label |
| `consequence` | string | Required | Plain-language description of what this choice causes |
| `isDefault` | boolean | Optional | Whether this option is pre-selected |
| `requiresConfirmation` | boolean | Required | Whether choosing this triggers a confirmation step |
| `triggersActionId` | id | Optional | The ActionSpec id initiated when this option is chosen | References `actions` section; allows per-option action routing |

**Rules:**
- `linkedActionIds` declares the full set of actions this decision can unlock or trigger. `triggersActionId` per option declares which specific action that option initiates.
- `postDecisionBehavior=navigateTo` requires a non-null `postDecisionNavigateToViewId`.

---

### 4.6 MonitoredEntitySpec

Monitored entities are things the position watches continuously. They may not require immediate action, but the position needs ongoing visibility into their state.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `entityType` | string | Required | The entity type being monitored | |
| `displayLabel` | string | Required | Label for this entity class (e.g., `Open Orders`) | |
| `keyFacts` | list\<KeyFactSpec\> | Required | The fields this position cares about | Min 1; max 8 |
| `monitoredStates` | list\<string\> | Optional | State values significant to this position | |
| `alertConditions` | list\<AlertConditionRef\> | Optional | Conditions that generate an alert for this entity | References `alerts` section |
| `linkedViewId` | id | Optional | The view that shows the full entity collection | |

**KeyFactSpec:**

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | The data field |
| `fieldLabel` | string | Required | Display label |
| `displayFormat` | enum(text, number, currency, date, status, badge) | Required | Render format |
| `isAlertable` | boolean | Optional | Whether this field can trigger an alert condition |

**AlertConditionRef:**

| Field | Type | Required | Description |
|---|---|---|---|
| `alertSpecId` | id | Required | References an entry in the `alerts` section |
| `conditionDescription` | string | Required | Plain-language condition description |

**Rules:**
- `keyFacts` max of 8 enforces the key-facts strip design constraint. More than 8 key facts signals a pattern selection error, not a schema extension.

---

### 4.7 AlertSpec

Alerts are events and exceptions surfaced to this position. Severity must align with UX Doctrine §7.5 tiers.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `alertSpecId` | id | Required | Unique identifier for this alert type | |
| `alertType` | string | Required | Business classification (e.g., `PaymentFailed`, `ComplianceFlagRaised`) | |
| `severity` | enum(informational, warning, critical) | Required | Must align with UX Doctrine §7.5 | |
| `titleTemplate` | string | Required | Display title template | May include entity name placeholder |
| `bodyTemplate` | string | Required | Plain-language body template | No technical codes; plain language only |
| `entityTypeRef` | EntityTypeRef | Optional | The entity type associated with this alert type | See §4.3 |
| `resolvableByThisPosition` | boolean | Required | Whether the current position can resolve this alert | |
| `escalationTargetPositionId` | id | Optional | Who to hand off to if not resolvable here | Required when `resolvableByThisPosition=false` |
| `resolutionActionIds` | list\<id\> | Optional | ActionSpec ids that resolve this alert | References `actions` section |
| `linkedViewId` | id | Optional | The view that handles resolution | |
| `generateAuditEntry` | boolean | Required | Whether resolution generates an audit record | Must be `true` for `severity=critical` |
| `warningEscalationHours` | number | Optional | Hours before a warning auto-escalates to critical | Only applicable when `severity=warning` |

**Rules:**
- `severity=critical` requires `generateAuditEntry=true`. Hard constraint from UX Doctrine §7.5.
- `severity=critical` requires either `resolvableByThisPosition=true` with at least one `resolutionActionId`, or a non-null `escalationTargetPositionId`.
- `resolvableByThisPosition=false` requires a non-null `escalationTargetPositionId`.
- `warningEscalationHours` defines the auto-escalation window for warnings consistent with UX Doctrine §7.5.

---

### 4.8 ActionSpec

Actions are things the position can initiate. This section defines the full action vocabulary. Which actions appear in which context is controlled by each view's pattern and the action-budget rules from UX Doctrine §6.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `actionId` | id | Required | Unique identifier | Invariant across regenerations |
| `actionType` | string | Required | Business classification (e.g., `SubmitInvoice`, `ApprovePO`) | |
| `label` | string | Required | Display label — must be consistent across all position apps | P8, P13 |
| `targetEntityType` | string | Optional | The entity type this action operates on | |
| `preconditions` | list\<PreconditionSpec\> | Optional | Conditions that must be true for this action to be available | |
| `requiredInputs` | list\<InputFieldSpec\> | Optional | Fields the user must provide | |
| `confirmationType` | enum(none, inline, full) | Required | Confirmation behavior — aligns with UX Doctrine P5 | |
| `isDestructive` | boolean | Required | Whether this action cannot be undone | |
| `requiresApproval` | boolean | Required | Whether this action enters an approval workflow | |
| `postActionBehavior` | enum(stay, navigateTo, dismiss) | Required | UI behavior after action completes | |
| `postActionNavigateToViewId` | id | Optional | Target viewId if `postActionBehavior=navigateTo` | |
| `aiAssistHookIds` | list\<id\> | Optional | AI assist hooks available for this action | |
| `telemetryKey` | string | Required | Stable key for the telemetry hook | Invariant; changing is a breaking change |

**PreconditionSpec:**

| Field | Type | Required | Description |
|---|---|---|---|
| `conditionDescription` | string | Required | Plain-language description of what must be true |
| `displayWhenNotMet` | enum(hide, disable\_with\_tooltip) | Required | Aligns with UX Doctrine A6 |
| `tooltipText` | string | Optional | Required when `displayWhenNotMet=disable_with_tooltip` |

**Rules:**
- `label` must be identical for the same `actionType` across all projections in the same tenant. Two actions of the same type must not use different labels (P13, I9). Enforcement mechanism: see Action Label Registry seam in §7.
- `telemetryKey` is invariant. Changing it is a breaking change (UX Doctrine P10, I7).
- `isDestructive=true` requires `confirmationType=full`.
- Actions with `requiresApproval=true` must have a corresponding `ApprovalSpec` in the `approvals` section.

**Action Label Registry — seam note:**  
The constraint that `label` is identical for the same `actionType` across all projections cannot be enforced within a single projection in isolation. Enforcement requires a **cross-projection action label registry** — a tenant-scoped index of `actionType → canonical label` pairs maintained by the compiler. When the compiler resolves a projection, it checks the registry: if the `actionType` already exists with a different `label`, compilation fails with a label conflict error. If the `actionType` is new, the `label` is registered. This registry is a compiler-layer concern, not a UI-layer concern; it is an open seam listed in §7 pending compiler design in a later workstream.

---

### 4.9 ApprovalSpec

Defines approval workflows this position participates in — either as initiator, approver, or observer.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `approvalSpecId` | id | Required | Unique identifier | |
| `approvalType` | string | Required | Business classification (e.g., `PurchaseOrderApproval`) | |
| `subjectEntityType` | string | Required | The entity type being approved | |
| `thisPositionRole` | enum(initiator, approver, observer) | Required | The role this position plays | |
| `requiredApproverPositionIds` | list\<id\> | Optional | Which positions must approve | Required when `thisPositionRole=initiator` |
| `approvalDeadlineHours` | number | Optional | Hours before the request escalates | |
| `showConsequenceOnReview` | boolean | Required | Whether the approval view shows business consequence | Should be `true`; `false` requires justification |
| `rejectionRequiresReason` | boolean | Required | Whether rejection requires a reason input | Must be `true` — UX Doctrine §7.4 |
| `approvalHistoryVisible` | boolean | Required | Whether approval history is shown on the record | Must be `true` — UX Doctrine §7.4 |
| `linkedActionId` | id | Optional | The ActionSpec that initiates this approval | References `actions` section |
| `linkedViewId` | id | Optional | The view that handles approval review | Must reference a view with `pattern=ApprovalReviewView` |

**Rules:**
- `rejectionRequiresReason` must always be `true`. This is non-negotiable (UX Doctrine §7.4).
- `approvalHistoryVisible` must always be `true`. Same basis.
- Positions with `thisPositionRole=approver` must have at least one view with `pattern=ApprovalReviewView` in `workSurface.views`.

---

### 4.10 CollaboratorSpec

Defines related positions and hand-off relationships relevant to this position's work.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `collaboratorPositionId` | id | Required | The related position's id | |
| `collaboratorTitle` | string | Required | Display name of the related position | |
| `relationshipType` | enum(receives\_from, sends\_to, shared\_visibility, escalation\_target) | Required | Nature of the relationship | |
| `handOffEntityTypes` | list\<string\> | Optional | Entity types that flow between positions | |
| `handOffActionIds` | list\<id\> | Optional | Actions that initiate hand-offs to this collaborator | References `actions` section |
| `visibleToCurrentPosition` | boolean | Required | Whether this collaborator's work summary is visible here | |

---

### 4.11 StateSummarySpec

State summaries are high-level indicators shown on the Overview/Monitor view. They communicate operational health and volume at a glance.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `summaryId` | id | Required | Unique identifier | |
| `summaryType` | enum(count, amount, ratio, status) | Required | Kind of value being shown | |
| `label` | string | Required | Display label (e.g., `Open Invoices`, `Pending Approvals`) | |
| `entityType` | string | Optional | The entity type this summary is computed from | |
| `filterDescription` | string | Optional | Plain-language description of the filter applied | |
| `alertThreshold` | AlertThresholdSpec | Optional | Condition that converts this summary into an alert | |
| `linkedViewId` | id | Optional | The view to navigate to when the user clicks this summary | |
| `displayOrder` | number | Required | Order in the summary strip | 1-indexed; unique within projection |

**AlertThresholdSpec:**

| Field | Type | Required | Description |
|---|---|---|---|
| `thresholdDescription` | string | Required | Plain-language threshold description |
| `severity` | enum(informational, warning, critical) | Required | Severity when threshold is crossed |

**Rules:**
- Any projection with a non-empty `stateSummaries` section must have at least one view with `pattern=OverviewMonitor` (composition rule C7).

---

### 4.12 PermissionsHooks

Defines the permission boundaries for this projection. These are seams into the policy layer. The UI generator uses them as routing instructions; evaluation happens in OPA.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `permissionsVersion` | string | Required | Policy schema version this projection was compiled against | Seam: matched against OPA at runtime |
| `allowedActionIds` | list\<id\> | Required | Action ids this position is permitted to initiate | |
| `visibleEntityTypes` | list\<string\> | Required | Entity types this position can see | |
| `dataFilterPolicies` | list\<DataFilterPolicy\> | Optional | Named filters applied to entity data for this position | Seam: filter logic lives in policy layer |
| `conditionalAccessHooks` | list\<ConditionalAccessHook\> | Optional | View or action conditions evaluated at runtime | |

**DataFilterPolicy:**

| Field | Type | Required | Description |
|---|---|---|---|
| `policyId` | id | Required | Unique identifier |
| `description` | string | Required | Plain-language description of the filter |
| `appliesTo` | string | Required | Entity type or viewId this policy applies to |

**ConditionalAccessHook:**

| Field | Type | Required | Description |
|---|---|---|---|
| `hookId` | id | Required | Unique identifier — referenced in `ViewSpec.accessCondition` |
| `description` | string | Required | Plain-language condition description |
| `evaluationTiming` | enum(compile\_time, runtime) | Required | When the condition is evaluated |

**Rules:**
- The generator must not surface any action absent from `allowedActionIds`.
- `dataFilterPolicies` are opaque to the UI layer. The UI passes the `policyId` to the backend; it does not implement filter logic.
- The UI layer is responsible for routing — not enforcement. Policy enforcement belongs to the policy layer.

---

### 4.13 AIAssistHook

Defines where and how AI assistance is available within this position app. All rules align with UX Doctrine P11 and P12.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `hookId` | id | Required | Unique identifier — referenced from tasks, decisions, actions | |
| `hookType` | enum(suggestion, draft\_generation, anomaly\_detection, summarization, auto\_complete, classification) | Required | Type of AI assistance | |
| `label` | string | Required | Plain-language label shown to the user | e.g., `Suggest vendor`, `Summarize activity` |
| `triggerContext` | enum(on\_view\_load, on\_field\_focus, on\_user\_request, on\_exception\_surface) | Required | When the hook activates | |
| `outputType` | enum(inline\_suggestion, panel\_result, field\_fill, notification) | Required | How the AI output is presented | |
| `isReversible` | boolean | Required | Whether the user can undo the AI output | Must be `true` unless `outputType=notification` |
| `requiresUserConfirmation` | boolean | Required | Whether the user must explicitly accept before output is applied | Aligns with P11, P12 |
| `affectsBusinessState` | boolean | Required | Whether accepting this output changes business state | If `true`, `requiresUserConfirmation` must be `true` |
| `explainabilityAvailable` | boolean | Required | Whether the user can ask why this suggestion was made | Seam: explainability service binding defined separately |

**Rules:**
- `affectsBusinessState=true` requires `requiresUserConfirmation=true`. Hard constraint from P11 and P12.
- `isReversible=false` is only permitted when `outputType=notification`.
- The user must always be aware that an AI action has occurred. Silent AI-applied changes are not permitted (P11).

---

### 4.14 DefaultViewHints

Generator-time hints that configure the default experience. These are static schema declarations, not runtime decisions.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `defaultLandingViewId` | id | Required | View shown on first app load | Must reference a valid `viewId` in `workSurface` |
| `defaultDensityMode` | enum(default, compact) | Required | Initial density mode | Aligns with UX Doctrine D2 |
| `defaultSortSpecs` | list\<SortSpec\> | Optional | Default sort orders per view | |
| `defaultFilterSpecs` | list\<FilterSpec\> | Optional | Default filters per view | |
| `defaultExpandedSections` | list\<SectionHint\> | Optional | Which record sections are expanded by default | |
| `pinnedSummaryIds` | list\<id\> | Optional | State summary ids to pin at the top of the overview | References `stateSummaries` |

**SortSpec:**

| Field | Type | Required | Description |
|---|---|---|---|
| `viewId` | id | Required | Which view this applies to |
| `fieldKey` | string | Required | The field to sort on |
| `direction` | enum(asc, desc) | Required | Sort direction |

**FilterSpec:**

| Field | Type | Required | Description |
|---|---|---|---|
| `viewId` | id | Required | Which view this applies to |
| `fieldKey` | string | Required | The field to filter on |
| `filterDescription` | string | Required | Plain-language description of the filter value |

**SectionHint:**

| Field | Type | Required | Description |
|---|---|---|---|
| `viewId` | id | Required | Which view this applies to |
| `sectionKey` | string | Required | The section identifier within that view |

---

### 4.15 PersonalizationHooks

Defines which surfaces the user can personalize within this position app, at what scope, and what UI elements each surface actually controls. Aligns with UX Doctrine §3.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `allowedSurfaces` | list\<PersonalizationSurface\> | Required | All surfaces that can be personalized in this projection | |
| `conflictPolicy` | enum(preserve\_user, reset\_to\_role, notify\_and\_preserve) | Required | Default behavior when a conflict occurs | Aligns with UX Doctrine §3.5 |

**PersonalizationSurface:**

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `surfaceId` | id | Required | Unique identifier | |
| `surfaceType` | enum | Required | Type of personalization surface | See vocabulary below |
| `scope` | enum(global, tenant, role, user) | Required | Maximum scope for configuration | Aligns with UX Doctrine §3.3 |
| `description` | string | Required | Plain-language description of what this surface controls | |
| `isStructuralVariant` | boolean | Required | Whether this surface changes layout structure | Structural variants require an approved variant in the pattern library |
| `targetViewIds` | list\<id\> | Optional | Which views this surface applies to | Empty = applies app-wide |
| `targetComponentTypes` | list\<string\> | Optional | The semantic component types this surface affects | e.g., `FilterRail`, `RecordHeader`, `KeyFactsStrip` |
| `configurationSchema` | list\<ConfigurationField\> | Optional | The shape of configuration this surface accepts | Defines what the user can actually change |

**ConfigurationField** (describes what the user can set for this surface):

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | Unique identifier for this configuration option |
| `fieldLabel` | string | Required | Display label shown to the user |
| `valueType` | enum(string, boolean, enum, number) | Required | Type of the configuration value |
| `allowedValues` | list\<string\> | Optional | Enum values — required when `valueType=enum` |
| `defaultValue` | string | Optional | The default value at this scope |

**Surface type vocabulary:**

`visual_theme`, `density_mode`, `saved_filter`, `sort_preference`, `column_choice`, `pinned_view`, `section_expansion`, `emphasis_variant`, `structural_variant`

**Rules:**
- `surfaceType=structural_variant` may not be introduced without a corresponding approved variant in the UI Pattern Library (10.3).
- `conflictPolicy` applies globally to this projection. Per-surface conflict policies are not supported at v0.
- The precedence order (global → tenant → role → user) from UX Doctrine §3.3 governs conflict resolution at runtime. `conflictPolicy` governs what to do when a user preference is invalidated — see UX Doctrine §3.5.
- `targetComponentTypes` must reference component names that exist in the Semantic Component Library (work product 10.5). At v0, these are declared as string identifiers and validated once 10.5 is complete.

---

## 5. Composition rules

These rules govern how sections relate to each other and constrain the generator. Violations at compile time are schema errors, not warnings.

**C1 — Action references must resolve**  
Any `actionId` referenced in `tasks.availableActionIds`, `alerts.resolutionActionIds`, `collaborators.handOffActionIds`, `approvals.linkedActionId`, `decisions.linkedActionIds`, or `decisions.options[].triggersActionId` must have a corresponding entry in `actions`.

**C2 — View references must resolve**  
Any `viewId` referenced in `navigation.defaultLandingViewId`, `EntityTypeRef.viewLink`, `alerts.linkedViewId`, `approvals.linkedViewId`, `stateSummaries.linkedViewId`, `decisions.postDecisionNavigateToViewId`, or `actions.postActionNavigateToViewId` must match a `viewId` in `workSurface.views`.

**C3 — AI hook references must resolve**  
Any `aiAssistHookId` referenced in `tasks`, `decisions`, `actions`, or `patternInputBinding.aiAssistHookIds` must have a corresponding entry in `aiAssistHooks`.

**C4 — Permissions gate actions**  
Actions in `actions` absent from `permissionsHooks.allowedActionIds` must be treated as hidden by the generator. They must not be surfaced regardless of other references.

**C5 — Critical alerts require audit**  
Any `AlertSpec` with `severity=critical` must have `generateAuditEntry=true`. Validated at compile time.

**C6 — Approvers require approval views**  
Any projection with an `ApprovalSpec` where `thisPositionRole=approver` must have at least one view with `pattern=ApprovalReviewView`.

**C7 — State summaries require an overview view**  
Any projection with a non-empty `stateSummaries` section must have at least one view with `pattern=OverviewMonitor`.

**C8 — Destructive actions require full confirmation**  
Any `ActionSpec` with `isDestructive=true` must have `confirmationType=full`. Validated at compile time.

**C9 — Approval-gated actions require approval specs**  
Any `ActionSpec` with `requiresApproval=true` must have a corresponding `ApprovalSpec` referencing it via `linkedActionId`.

**C10 — Action slot binding must satisfy the action budget**  
In any `PatternInputBinding.actionSlots`: `secondaryActionIds.length` ≤ 3; total visible actions (`1 primary + secondary count`) ≤ 5; any action count above 5 must go to `overflowActionIds`. Validated at compile time. Violations are schema errors, not warnings.

---

## 6. Versioning strategy

### 6.1 Version types

| Version type | Scope | Tracks |
|---|---|---|
| `schemaVersion` | Entire `PositionProjection` structure | Changes to the schema definition itself |
| `positionVersion` | One specific position's compiled projection | Changes to that position's resolved data |

### 6.2 Breaking vs non-breaking changes

**Breaking — requires major version bump and migration plan:**
- Removing or renaming a required field
- Renaming a field marked as invariant (`viewId`, `actionId`, `positionId`, `telemetryKey`)
- Changing the type of an existing field
- Removing or renaming a value in an existing enum
- Changing a view's `pattern` (UX Doctrine I4)
- Changing `actionType` label for the same business action (UX Doctrine P13, I9)

**Non-breaking — minor version bump:**
- Adding a new optional field to any section
- Adding a new value to an existing enum
- Adding a new extension section
- Moving navigation items (unless it changes `navOrder` in a way that disrupts user expectations)

**Non-versioned — no bump required:**
- Changes to `description` or template strings that do not alter semantics
- Changes to `displayOrder` within a section that do not reorder invariant items

### 6.3 Backward compatibility rules

- A generator built against schema version N must consume a projection compiled against schema version N-1 without failure.
- Unknown optional fields must be silently ignored (forward compatibility for minor bumps).
- Breaking changes require a migration plan before the regenerated projection is deployed.
- Per-section compatibility tracking may be introduced in a future version when extension section lifecycles diverge significantly from core section lifecycles.

### 6.4 Deprecation windows for major version bumps

When a major schema version bump occurs, live tenants may be running projections compiled against the previous major version. The following rules define the operational window:

| Phase | Definition | Duration |
|---|---|---|
| Dual-support window | Generator supports both version N and N-1 projections simultaneously | Minimum 2 full regeneration cycles for the affected tenant; defined per deployment |
| Migration-required notice | Tenants are notified that N-1 projections must be regenerated against N | Issued at the start of the dual-support window |
| Hard cutoff | Generator drops support for N-1 projections | After the dual-support window closes; N-1 projections are rejected at compile time |

**Operational rules:**
- A tenant's position app continues to run the N-1 compiled artifact during the dual-support window. No forced downtime.
- The compiler must refuse to regenerate new N-1 projections once the major bump is live — only existing N-1 artifacts remain valid during the window.
- The migration plan for a major bump must include: a list of affected fields, the transformation required, and automated migration tooling where feasible.
- Exact window durations are deployment decisions, not schema decisions. This section defines the minimum structure; specific timelines are set per release by the workstream owner.

---

## 7. Open seams

Integration points where other Foundry layers connect. Identified here to prevent over-engineering at v0 while preserving the connection points for later.

| Seam | Connects to | Current status at v0 |
|---|---|---|
| `boundedContextId` in PositionIdentity | Semantic layer bounded context model | Placeholder id; full context definition is out of scope |
| `accessConditionHookId` in ViewSpec | OPA policy evaluation | Opaque to UI layer; `hookId` passed at runtime to policy layer |
| `dataFilterPolicies` in PermissionsHooks | OPA data-scoping policies | Opaque to UI layer; `policyId` passed to backend at runtime |
| `permissionsVersion` | OPA schema versioning | Version matched at compile time; runtime enforcement by policy layer |
| `primaryEntityType` in ViewSpec and `entityType` in EntityTypeRef | Full entity model in semantic layer | Entity types named here; field definitions and schema live in semantic layer |
| `PatternInputBinding` per ViewSpec | UI Pattern Library v0 (10.3) | Per-view binding declared here; authoritative pattern input contracts defined in 10.3 |
| `targetComponentTypes` in PersonalizationSurface | Semantic Component Library v0 (10.5) | Component names declared here; component specs defined in 10.5 |
| `aiAssistHooks` | AI service layer | Hook type and trigger declared here; AI service binding defined separately |
| `explainabilityAvailable` in AIAssistHook | AI model explainability service | Declared here; implementation and response format in AI service layer |
| `fieldKey` values in KeyFactSpec, SortSpec, FilterSourceSpec | Semantic entity field model | Field keys named here; field types and validation live in semantic layer |
| Cross-projection action label registry | Compiler layer | Tenant-scoped `actionType → canonical label` index enforced at compile time. Declared as a constraint in §4.8; registry mechanism is a compiler-layer design concern pending compiler workstream. |
| Event and lifecycle modeling | Future event/lifecycle layer | Not modeled in this schema at v0. Events, lifecycle hooks, and state-transition triggers are an open seam for a future extension. AlertSpec covers exception surfacing; deeper event modeling (EventStorming-derived) is out of scope until after 10.3 is complete. |
| Provenance | Provenance layer | Every projection change should carry provenance; captured externally — not in this schema |
| Alert escalation execution | Workflow/orchestration layer (Temporal) | `warningEscalationHours` declared here; escalation execution handled by orchestration layer |

---

## 8. Milestone 2 exit criteria — self-assessment

| Exit criterion | Status | Evidence |
|---|---|---|
| Schema covers tasks, entities, actions, events, approvals, and state | Met | §4.4 TaskSpec, §4.6 MonitoredEntitySpec, §4.8 ActionSpec, §4.7 AlertSpec, §4.9 ApprovalSpec, §4.11 StateSummarySpec |
| Personalization hooks and scopes are identified | Met | §4.15 PersonalizationHooks; scopes align with UX Doctrine §3.3 precedence |
| Schema is sufficiently narrow to remain stable | Met | Schema is UI-facing only; excludes backend contracts, policy logic, and provenance internals |
| Open seams for policy/provenance/AI are identified without overcommitting | Met | §7 Open Seams lists all integration points with explicit status |

---

*This document is version 0. It is the first executable version sufficient to drive UI pattern selection and position app generation. It is expected to be refined as the UI Pattern Library v0 (work product 10.3) is developed and as the first pilot workflows are validated against this schema.*
