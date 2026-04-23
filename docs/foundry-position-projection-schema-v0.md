# Foundry Position Projection Schema v0

**Status:** Final v0  
**Workstream:** UI/UX — Work Product 10.2  
**References:** Foundry UX Doctrine v0, Foundry UI/UX Workstream Charter v1, Foundry2 Position-Centric Regenerative Software for SMBs v1  
**Purpose:** Define the UI-facing schema that the compiler must resolve to generate a position app.

This schema is the contract between the model/compiler layer and the UI generation layer. It describes exactly what information a position app needs from the model to render its views, actions, and interactions correctly.

This schema is **not** a database schema, not a backend service contract, and not a full business model. It is narrowly scoped to what the UI generator needs. Stability of this schema is a first-class goal.

---

## 1. Purpose and position in the compiler flow

The Position Projection Schema sits between the semantic normalization step and the UI generation step.

```text
progressive semantic slice
    -> structural validation
    -> policy validation
    -> normalization
    -> [PositionProjection resolution]  ← this schema defines this layer
    -> UI pattern and variant resolution
    -> React position app generation
```

The compiler resolves a validated semantic slice into a `PositionProjection` object. The UI generator consumes this object to select patterns, compose views, configure personalization surfaces, and wire up action and permission boundaries.

A semantic slice can be partial. Foundry2 explicitly supports pain-point-first, start-anywhere adoption — a position app may be generated before all of its capabilities are modeled. The UI generator must distinguish between an extension section that is absent because the position genuinely lacks that capability, and one that is absent because the slice is not yet complete. The `completeness` section (§4.16) provides this distinction. An absent extension section that is not explicitly declared as out of scope is treated as deferred, consistent with UX Doctrine §2.8. Provisional state must be explicit; unknown must not masquerade as empty, zero, or complete.

**Schema design goals:**
- Narrow enough to remain stable across minor semantic changes
- Complete enough to drive all UI pattern selection decisions without further model queries
- Decoupled from backend service contracts, policy internals, and provenance records
- Explicit about open seams where policy, provenance, instrumentation, and AI integration will connect later

---

## 2. Schema architecture principles

### 2.1 Core sections vs extension sections

The schema has two tiers. **Core sections** are required for any position app. **Extension sections** are conditionally present based on what kind of work the position does. Extension sections that are absent must not cause a pattern failure.

| Tier | When present |
|---|---|
| Core | Always — required for every generated position app |
| Extension | Only when the position's semantic slice includes that work type |

### 2.2 Static schema vs runtime data

This schema is a **compiled, static artifact**. It describes types, structures, and rules — not live instances or resolved values. Where a field must exist at schema level but is only resolved at runtime, it is explicitly marked `(runtime-resolved)`.

| Schema level (this document) | Runtime level (not in this document) |
|---|---|
| `EntityTypeRef` — entity type + display template | Live entity instance with id and resolved display name |
| `AlertSpec` — alert type + body template | Active alert instance with resolved values |
| `FilterSpec` — field key + description of default | Applied filter with resolved value for a specific user |
| `TaskSpec` — task type + state model | Active task instance with current state and assignee |

Blurring this boundary is a schema defect. If a field feels like a runtime value, it should be a template, a type reference, or a seam — not a resolved value.

### 2.3 Canonical ownership and validation scopes

Two additional rules govern the schema:

1. **One fact, one owner.**  
   Any meaningful fact in the projection has exactly one canonical owner field. Duplicate ownership is a schema defect.

2. **Every rule has a validation scope.**  
   Constraints in this document fall into one of three scopes:

| Validation scope | Meaning | Typical examples |
|---|---|---|
| Projection-local | Can be validated from one `PositionProjection` instance in isolation | Reference resolution, action-budget compliance, required linked sections |
| Compiler-global | Requires registry state, cross-projection comparison, or deployment context | Action label consistency across projections, pattern/variant legitimacy |
| Downstream seam | Validated by an adjacent layer or later work product | OPA policy evaluation, instrumentation semantics, component-library identifiers |

If a rule depends on information outside one projection, it must be marked compiler-global or downstream. It must not be presented as a local schema check.

---

## 3. Top-level schema structure

| Section | Tier | Type | Required | Purpose |
|---|---|---|---|---|
| `schemaVersion` | Core | string | Required | Version of the Position Projection schema definition |
| `identity` | Core | `PositionIdentity` | Required | Who this position is and its organizational context |
| `workSurface` | Core | `WorkSurfaceComposition` | Required | Navigation structure, view registry, and pattern bindings |
| `actions` | Core | `list<ActionSpec>` | Required | Full action vocabulary for this position |
| `permissionsHooks` | Core | `PermissionsHooks` | Required | Permission scopes and boundaries |
| `defaultViewHints` | Core | `DefaultViewHints` | Required | Generator-time defaults for non-structural presentation |
| `personalizationHooks` | Core | `PersonalizationHooks` | Required | Personalization surfaces and scopes |
| `completeness` | Core | `ProjectionCompletenessSpec` | Required | Slice completeness state and capability resolution status |
| `tasks` | Extension | `list<TaskSpec>` | Optional | Discrete work items — present when position handles task-oriented work |
| `decisions` | Extension | `list<DecisionSpec>` | Optional | Structured choice points — present when position makes explicit decisions |
| `monitoredEntities` | Extension | `list<MonitoredEntitySpec>` | Optional | Watched entities — present when position has monitoring responsibility |
| `alerts` | Extension | `list<AlertSpec>` | Optional | Events and exceptions — present when position receives exception alerts |
| `approvals` | Extension | `list<ApprovalSpec>` | Optional | Approval workflows — present when position initiates or reviews approvals |
| `collaborators` | Extension | `list<CollaboratorSpec>` | Optional | Hand-off relationships — present when position has explicit cross-position flows |
| `stateSummaries` | Extension | `list<StateSummarySpec>` | Optional | Overview indicators — present when position has an Overview/Monitor view |
| `aiAssistHooks` | Extension | `list<AIAssistHook>` | Optional | AI assistance — present when position has AI-assisted workflows |

### 3.1 Schema-level metadata

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `schemaVersion` | string | Required | Version of the Position Projection schema definition | Semver; canonical owner of structural schema version; not semantic-slice data |

**Schema-level rules:**
- Every `PositionProjection` must include all Core sections, including `schemaVersion` and `completeness`.
- Extension sections that are absent must not cause a UI pattern failure. Whether an absent extension section renders as hidden, empty, or provisional depends on `completeness`. See §4.16 and C12.
- The schema must be fully resolvable from a validated semantic slice without querying live backend services.
- `schemaVersion` and `identity.positionVersion` are distinct. `schemaVersion` tracks the structure of this schema. `positionVersion` tracks one position's compiled projection.

---

## 4. Section specifications

### 4.1 PositionIdentity

Identifies and scopes the position. These fields are stable invariants unless explicitly versioned and migrated.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `positionId` | id | Required | Unique identifier for this position | Invariant across regenerations |
| `positionTitle` | string | Required | Display name of the position | Shown in app header and navigation |
| `positionSlug` | string | Required | URL-safe identifier | Lowercase, hyphenated; invariant |
| `tenantId` | id | Required | The company this position belongs to | |
| `boundedContextId` | id | Required | The bounded context that owns this position's primary data | Seam: bounded context definition lives in semantic layer |
| `orgUnit` | string | Optional | The org unit or department | Display only; not a routing key |
| `positionVersion` | string | Required | Version of this compiled position projection | Semver; not interchangeable with `schemaVersion` |

**Rules:**
- `positionId` and `positionSlug` are invariants. They must not change when the position is regenerated without explicit migration.
- `positionVersion` changes when the compiled projection changes materially, even if `schemaVersion` does not.
- `schemaVersion` is the canonical owner of schema structure version. `positionVersion` is the canonical owner of one projection instance's version.

---

### 4.2 WorkSurfaceComposition

`workSurface` is split into two distinct sub-concerns: **NavigationSpec** (how the user moves between views) and **ViewRegistry** (what each view is and what it provides to its pattern). These are kept as sub-sections of `workSurface` but are independently owned.

| Field | Type | Required | Description |
|---|---|---|---|
| `navigation` | `NavigationSpec` | Required | App-level navigation structure |
| `views` | `list<ViewSpec>` | Required | View definitions and pattern bindings |

#### 4.2a NavigationSpec

Owns navigation structure only. Navigation concerns (order, labels, icons, landing) are fully contained here and do not bleed into `ViewSpec`.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `defaultLandingViewId` | id | Required | The view shown on first load | Canonical owner of landing view; must reference a valid `viewId` in `views` |
| `navItems` | `list<NavItem>` | Required | All navigation entries | Min 1 |
| `navStyle` | `enum(sidebar, topbar, tabs)` | Optional | Navigation presentation hint | Must be an approved variant; default is `sidebar` |

**NavItem**

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `viewId` | id | Required | References a `ViewSpec` in the view registry | |
| `navLabel` | string | Required | Display label shown in navigation | |
| `navOrder` | number | Required | Position in nav list | 1-indexed; unique |
| `navIcon` | string | Optional | Icon identifier | Must reference approved icon vocabulary |
| `isHiddenFromNav` | boolean | Optional | Reachable but not in main nav | Default: `false` |

**Rules:**
- `defaultLandingViewId` is the only canonical owner of the landing-view decision.
- Landing-view defaults must not be duplicated elsewhere in the projection. `DefaultViewHints` intentionally does not own landing view.

#### 4.2b ViewRegistry — ViewSpec

Each `ViewSpec` owns its pattern binding and layout hints. Navigation concerns remain in `NavigationSpec`.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `viewId` | id | Required | Unique view identifier within this projection | Invariant across regenerations |
| `pattern` | `PatternId` | Required | The UI pattern this view maps to | Validated against UI Pattern Library v0; see vocabulary below |
| `patternVariant` | string | Optional | An approved variant name | Compiler-global validation against UI Pattern Library v0 |
| `primaryEntityType` | string | Optional | The entity type this view centers on | Seam: entity definitions live in semantic layer |
| `patternInputBinding` | `PatternInputBinding` | Required | What this view provides to its pattern | See §4.2c |
| `layoutHints` | `LayoutHints` | Optional | How content is grouped within this view | See §4.2d |
| `accessConditionHookId` | string | Optional | ConditionalAccessHook id controlling visibility | Seam: evaluated at runtime by permissions layer |

**PatternId vocabulary for v0**

For v0, `PatternId` is a **closed vocabulary** validated against UI Pattern Library v0 (work product 10.3). This is intentionally explicit at v0. A later version may move to registry-backed extensible identifiers without changing the ownership of the `pattern` field.

Page patterns:
- `CollectionView`
- `RecordPage`
- `OverviewMonitor`
- `SearchResults`
- `ExceptionResolutionView`
- `ApprovalReviewView`

Flow patterns:
- `CreateEditFlow`
- `ApprovalFlow`
- `ExceptionResolutionFlow`
- `AssistedSetupFlow`

**ViewSpec rules:**
- `viewId` is invariant.
- Changing a view's `pattern` is a breaking change and requires versioned migration.
- Pattern and variant legitimacy are **compiler-global/library validations**, not assumptions made from one projection in isolation.
- Each pattern value must correspond to a pattern defined in UI Pattern Library v0 (10.3).

#### 4.2c PatternInputBinding

Declares what this view provides to its pattern. This is the per-view side of the pattern contract. The authoritative required-vs-optional input requirements per pattern type are defined in UI Pattern Library v0 (10.3); this binding is validated against those requirements at compile time.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `patternType` | `PatternId` | Required | Must match the parent `ViewSpec.pattern` | Must be identical to parent view pattern |
| `entityTypeRef` | `EntityTypeRef` | Optional | Primary entity for this view | See §4.3 |
| `listSourceDescription` | string | Optional | Plain-language description of the data source | Required for list-based patterns |
| `filterSources` | `list<FilterSourceSpec>` | Optional | Available filter dimensions for this view | |
| `actionSlots` | `ActionSlotBinding` | Optional | Which actions go in which slots | Validated by projection-local rule C11 |
| `aiAssistHookIds` | `list<id>` | Optional | AI hooks available in this view | References `aiAssistHooks` section |

**ActionSlotBinding**

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `primaryActionId` | id | Optional | The single primary action for this context | One or zero; validated by C11 |
| `secondaryActionIds` | `list<id>` | Optional | Visible secondary actions in this context | Budget validated by C11 |
| `overflowActionIds` | `list<id>` | Optional | Actions placed in overflow | Used when visible budget is exceeded |
| `bulkActionIds` | `list<id>` | Optional | Actions available in bulk-select state | Bulk behavior validated by pattern contract |

**FilterSourceSpec**

| Field | Type | Required | Description |
|---|---|---|---|
| `filterKey` | string | Required | Unique filter identifier |
| `filterLabel` | string | Required | Display label |
| `filterType` | `enum(text, select, date_range, status, boolean)` | Required | Filter input type |
| `isDefaultVisible` | boolean | Required | Shown by default or behind expansion |

**Rules:**
- `PatternInputBinding` must satisfy the consuming pattern's input contract from UI Pattern Library v0.
- Action-slot math and action-budget compliance are enforced by projection-local composition rule C11.
- Pattern-specific slot semantics belong to the pattern library, not to this schema.

#### 4.2d LayoutHints

Declares how content is grouped within this view. Optional — if absent, the pattern's default layout applies.

| Field | Type | Required | Description |
|---|---|---|---|
| `primaryRegion` | string | Optional | Semantic label for the main content area |
| `sidebarRegion` | string | Optional | Semantic label for sidebar content |
| `headerRegion` | string | Optional | What appears in the record or page header |
| `groupedSections` | `list<SectionGroup>` | Optional | How form or detail sections are grouped |

**SectionGroup**

| Field | Type | Required | Description |
|---|---|---|---|
| `groupKey` | string | Required | Unique group identifier |
| `groupLabel` | string | Required | Display label for this group |
| `sectionKeys` | `list<string>` | Required | Ordered section keys within this group |
| `isCollapsedByDefault` | boolean | Optional | Default collapse state; default: `false` |

---

### 4.3 Shared sub-types

These types are reused across multiple sections. They are defined once here.

#### EntityTypeRef

`EntityTypeRef` is a **schema-level type reference** — not an instance reference. It names an entity type and provides a display template. Instance ids and resolved display values are runtime data and do not belong in this schema.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `entityType` | string | Required | The entity type being referenced | Seam: entity model lives in semantic layer |
| `displayNameTemplate` | string | Required | Template string for display (for example `{{name}} — {{status}}`) | Resolved at runtime; not a static value |
| `viewLink` | id | Optional | The `viewId` in this projection that shows this entity's detail | |

> **Static/runtime rule:** Do not include `entityId` or resolved display strings in an `EntityTypeRef`. Those are runtime values resolved by the UI from live backend queries, not schema declarations.

**viewLink accessibility rule:**  
`viewLink` is optional. When present, the generator renders it as a navigable link. When the linked view is inaccessible to the current user — either because the `viewId` is absent from this projection or because its `accessConditionHookId` evaluates to denied at runtime — the generator must render the entity reference as a plain non-navigable label. The entity name remains visible; only the navigation affordance is suppressed.

#### InputFieldSpec

Reused in `ActionSpec` and `DecisionSpec`.

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | Unique field identifier |
| `fieldLabel` | string | Required | Display label |
| `inputType` | `enum(text, number, currency, date, select, multiselect, textarea, file)` | Required | Input type |
| `isRequired` | boolean | Required | Whether mandatory |
| `validationRules` | `list<string>` | Optional | Plain-language validation descriptions |

---

### 4.4 TaskSpec

A task is a discrete, bounded unit of work assigned to or owned by this position.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `taskSpecId` | id | Required | Unique identifier for this task type | |
| `taskType` | string | Required | Business classification (for example `InvoiceReview`, `OnboardingStep`) | Must match vocabulary in semantic slice |
| `titleTemplate` | string | Required | Plain-language task title template | May include entity name placeholder |
| `entityTypeRef` | `EntityTypeRef` | Optional | The entity type this task is about | See §4.3 |
| `stateModel` | `list<string>` | Required | The valid state values for this task type | Example: `[open, in_progress, blocked, completed, cancelled]` |
| `priorityLevels` | `list<string>` | Required | The valid priority values | Example: `[low, normal, high, urgent]` |
| `hasDueDate` | boolean | Required | Whether this task type carries a due date | |
| `availableActionIds` | `list<id>` | Optional | `ActionSpec` ids that apply to tasks of this type | References `actions` section |
| `aiAssistHookIds` | `list<id>` | Optional | AI assist hook ids available for this task type | References `aiAssistHooks` section |

**Rules:**
- `availableActionIds` must only reference actions defined in the `actions` section of the same projection.
- `TaskSpec` is schema-level. It describes what kinds of tasks this position handles. Live task instances are runtime data, not schema data.

---

### 4.5 DecisionSpec

A decision is a choice point where the position holder selects from defined options with concrete business consequences. Decisions differ from approvals: an approval is binary (`approve`/`reject`); a decision selects from a defined option set and may trigger different downstream actions per option.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `decisionSpecId` | id | Required | Unique identifier | |
| `decisionType` | string | Required | Business classification (for example `SupplierSelection`, `PricingException`) | |
| `titleTemplate` | string | Required | Plain-language description of what must be decided | |
| `contextFields` | `list<ContextField>` | Required | Information the user needs to see before deciding | Min 1 |
| `options` | `list<DecisionOption>` | Required | Available choices | Min 2 |
| `hasDueDate` | boolean | Required | Whether this decision type has a deadline | |
| `entityTypeRef` | `EntityTypeRef` | Optional | The entity type this decision concerns | See §4.3 |
| `linkedActionIds` | `list<id>` | Optional | Actions that become available or are triggered after this decision | References `actions` section |
| `postDecisionBehavior` | `enum(stay, navigateTo, dismiss)` | Required | UI behavior after the decision is made | Aligns with `ActionSpec.postActionBehavior` |
| `postDecisionNavigateToViewId` | id | Optional | Target `viewId` if `postDecisionBehavior=navigateTo` | |
| `aiAssistHookIds` | `list<id>` | Optional | AI assist hooks for decision support | |

**ContextField**

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | Data field identifier |
| `fieldLabel` | string | Required | Display label |
| `displayFormat` | `enum(text, number, currency, date, status, badge)` | Required | How to render this value |
| `isHighlighted` | boolean | Optional | Whether this field receives visual emphasis |

**DecisionOption**

| Field | Type | Required | Description |
|---|---|---|---|
| `optionId` | id | Required | Unique identifier within this decision |
| `label` | string | Required | Display label |
| `consequence` | string | Required | Plain-language description of what this choice causes |
| `isDefault` | boolean | Optional | Whether this option is pre-selected |
| `requiresConfirmation` | boolean | Required | Whether choosing this triggers a confirmation step |
| `triggersActionId` | id | Optional | The `ActionSpec` id initiated when this option is chosen | References `actions` section; allows per-option action routing |

**Rules:**
- `linkedActionIds` declares the full set of actions this decision can unlock or trigger.
- `triggersActionId` declares the specific action a given option initiates.
- `postDecisionBehavior=navigateTo` requires a non-null `postDecisionNavigateToViewId`.

---

### 4.6 MonitoredEntitySpec

Monitored entities are things the position watches continuously. They may not require immediate action, but the position needs ongoing visibility into their state.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `entityType` | string | Required | The entity type being monitored | |
| `displayLabel` | string | Required | Label for this entity class (for example `Open Orders`) | |
| `keyFacts` | `list<KeyFactSpec>` | Required | The fields this position cares about | Min 1 |
| `monitoredStates` | `list<string>` | Optional | State values significant to this position | |
| `alertConditions` | `list<AlertConditionRef>` | Optional | Conditions that generate an alert for this entity | References `alerts` section |
| `linkedViewId` | id | Optional | The view that shows the full entity collection | |

**KeyFactSpec**

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | The data field |
| `fieldLabel` | string | Required | Display label |
| `displayFormat` | `enum(text, number, currency, date, status, badge)` | Required | Render format |
| `isAlertable` | boolean | Optional | Whether this field can trigger an alert condition |

**AlertConditionRef**

| Field | Type | Required | Description |
|---|---|---|---|
| `alertSpecId` | id | Required | References an entry in the `alerts` section |
| `conditionDescription` | string | Required | Plain-language condition description |

**Rules:**
- `keyFacts` is a semantic declaration, not a promise that all facts are simultaneously visible in every pattern. Surface capacity is validated against the consuming pattern contract in UI Pattern Library v0.
- If a consuming pattern cannot surface the declared `keyFacts` set within its approved bounds, pattern validation fails or requires an approved variant.

---

### 4.7 AlertSpec

Alerts are events and exceptions surfaced to this position. Severity must align with UX Doctrine §7.5 tiers.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `alertSpecId` | id | Required | Unique identifier for this alert type | |
| `alertType` | string | Required | Business classification (for example `PaymentFailed`, `ComplianceFlagRaised`) | |
| `severity` | `enum(informational, warning, critical)` | Required | Must align with UX Doctrine §7.5 | |
| `titleTemplate` | string | Required | Display title template | May include entity name placeholder |
| `bodyTemplate` | string | Required | Plain-language body template | No technical codes; plain language only |
| `entityTypeRef` | `EntityTypeRef` | Optional | The entity type associated with this alert type | See §4.3 |
| `resolvableByThisPosition` | boolean | Required | Whether the current position can resolve this alert | |
| `escalationTargetPositionId` | id | Optional | Who to hand off to if not resolvable here | Required when `resolvableByThisPosition=false` |
| `resolutionActionIds` | `list<id>` | Optional | `ActionSpec` ids that resolve this alert | References `actions` section |
| `linkedViewId` | id | Optional | The view that handles resolution | |
| `generateAuditEntry` | boolean | Required | Whether resolution generates an audit record | Must be `true` for `severity=critical` |
| `warningEscalationHours` | number | Optional | Hours before a warning auto-escalates to critical | Only applicable when `severity=warning` |

**Rules:**
- `severity=critical` requires `generateAuditEntry=true`.
- `severity=critical` requires either `resolvableByThisPosition=true` with at least one `resolutionActionId`, or a non-null `escalationTargetPositionId`.
- `resolvableByThisPosition=false` requires a non-null `escalationTargetPositionId`.
- `warningEscalationHours` defines the auto-escalation window for warnings consistent with UX Doctrine §7.5.

---

### 4.8 ActionSpec

Actions are things the position can initiate. This section defines the full action vocabulary. Which actions appear in which context is controlled by each view's pattern and the action-budget rules from UX Doctrine §6.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `actionId` | id | Required | Unique identifier | Invariant across regenerations |
| `actionType` | string | Required | Business classification (for example `SubmitInvoice`, `ApprovePO`) | |
| `label` | string | Required | Display label for this action type | Compiler-global consistency rule; see below |
| `targetEntityType` | string | Optional | The entity type this action operates on | |
| `preconditions` | `list<PreconditionSpec>` | Optional | Conditions that must be true for this action to be available | |
| `requiredInputs` | `list<InputFieldSpec>` | Optional | Fields the user must provide | |
| `confirmationType` | `enum(none, inline, full)` | Required | Confirmation behavior — aligns with UX Doctrine P5 | |
| `isDestructive` | boolean | Required | Whether this action cannot be undone | |
| `requiresApproval` | boolean | Required | Whether this action enters an approval workflow | |
| `postActionBehavior` | `enum(stay, navigateTo, dismiss)` | Required | UI behavior after action completes | |
| `postActionNavigateToViewId` | id | Optional | Target `viewId` if `postActionBehavior=navigateTo` | |
| `aiAssistHookIds` | `list<id>` | Optional | AI assist hooks available for this action | |
| `telemetryKey` | string | Required | Opaque instrumentation reference for this action | Semantics owned by instrumentation layer |

**PreconditionSpec**

| Field | Type | Required | Description |
|---|---|---|---|
| `conditionDescription` | string | Required | Plain-language description of what must be true |
| `displayWhenNotMet` | `enum(hide, disable_with_tooltip)` | Required | Aligns with UX Doctrine A6 |
| `tooltipText` | string | Optional | Required when `displayWhenNotMet=disable_with_tooltip` |

**Rules:**
- `label` consistency for the same `actionType` across projections is a **compiler-global validation**, not a single-projection rule. Enforcement mechanism: Action Label Registry seam note below.
- `isDestructive=true` requires `confirmationType=full`.
- Actions with `requiresApproval=true` must have a corresponding `ApprovalSpec` in the `approvals` section.
- `telemetryKey` is an opaque reference into the instrumentation layer. The existence and compatibility of the referenced hook are governed by the evaluation/instrumentation specification, not by this schema alone.

**Action Label Registry — seam note**  
The constraint that `label` is identical for the same `actionType` across all projections cannot be enforced within a single projection in isolation. Enforcement requires a **cross-projection action label registry** — a tenant-scoped index of `actionType -> canonical label` pairs maintained by the compiler. When the compiler resolves a projection, it checks the registry: if the `actionType` already exists with a different `label`, compilation fails with a label conflict error. If the `actionType` is new, the `label` is registered. This registry is a compiler-layer concern, not a UI-layer concern.

---

### 4.9 ApprovalSpec

Defines approval workflows this position participates in — either as initiator, approver, or observer.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `approvalSpecId` | id | Required | Unique identifier | |
| `approvalType` | string | Required | Business classification (for example `PurchaseOrderApproval`) | |
| `subjectEntityType` | string | Required | The entity type being approved | |
| `thisPositionRole` | `enum(initiator, approver, observer)` | Required | The role this position plays | |
| `requiredApproverPositionIds` | `list<id>` | Optional | Which positions must approve | Required when `thisPositionRole=initiator` |
| `approvalDeadlineHours` | number | Optional | Hours before the request escalates | |
| `showConsequenceOnReview` | boolean | Required | Whether the approval view shows business consequence | `false` requires explicit justification downstream |
| `rejectionRequiresReason` | boolean | Required | Whether rejection requires a reason input | Must be `true` — UX Doctrine §7.4 |
| `approvalHistoryVisible` | boolean | Required | Whether approval history is shown on the record | Must be `true` — UX Doctrine §7.4 |
| `linkedActionId` | id | Optional | The `ActionSpec` that initiates this approval | References `actions` section |
| `linkedViewId` | id | Optional | The view that handles approval review | Must reference a view with `pattern=ApprovalReviewView` |

**Rules:**
- `rejectionRequiresReason` must always be `true`.
- `approvalHistoryVisible` must always be `true`.
- Positions with `thisPositionRole=approver` must have at least one view with `pattern=ApprovalReviewView` in `workSurface.views`.

---

### 4.10 CollaboratorSpec

Defines related positions and hand-off relationships relevant to this position's work.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `collaboratorPositionId` | id | Required | The related position's id | |
| `collaboratorTitle` | string | Required | Display name of the related position | |
| `relationshipType` | `enum(receives_from, sends_to, shared_visibility, escalation_target)` | Required | Nature of the relationship | |
| `handOffEntityTypes` | `list<string>` | Optional | Entity types that flow between positions | |
| `handOffActionIds` | `list<id>` | Optional | Actions that initiate hand-offs to this collaborator | References `actions` section |
| `visibleToCurrentPosition` | boolean | Required | Whether this collaborator's work summary is visible here | |

---

### 4.11 StateSummarySpec

State summaries are high-level indicators shown on the Overview/Monitor view. They communicate operational health and volume at a glance.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `summaryId` | id | Required | Unique identifier | |
| `summaryType` | `enum(count, amount, ratio, status)` | Required | Kind of value being shown | |
| `label` | string | Required | Display label (for example `Open Invoices`, `Pending Approvals`) | |
| `entityType` | string | Optional | The entity type this summary is computed from | |
| `filterDescription` | string | Optional | Plain-language description of the filter applied | |
| `alertThreshold` | `AlertThresholdSpec` | Optional | Condition that converts this summary into an alert | |
| `linkedViewId` | id | Optional | The view to navigate to when the user clicks this summary | |
| `displayOrder` | number | Required | Order in the summary strip | 1-indexed; unique within projection |

**AlertThresholdSpec**

| Field | Type | Required | Description |
|---|---|---|---|
| `thresholdDescription` | string | Required | Plain-language threshold description |
| `severity` | `enum(informational, warning, critical)` | Required | Severity when threshold is crossed |

**Rules:**
- Any projection with a non-empty `stateSummaries` section must have at least one view with `pattern=OverviewMonitor`.

---

### 4.12 PermissionsHooks

Defines the permission boundaries for this projection. These are seams into the policy layer. The UI generator uses them as routing instructions; evaluation happens in OPA or an equivalent policy engine.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `permissionsVersion` | string | Required | Policy schema version this projection was compiled against | Seam: matched against policy layer at runtime |
| `allowedActionIds` | `list<id>` | Required | Action ids this position is permitted to initiate | |
| `visibleEntityTypes` | `list<string>` | Required | Entity types this position can see | |
| `dataFilterPolicies` | `list<DataFilterPolicy>` | Optional | Named filters applied to entity data for this position | Seam: filter logic lives in policy layer |
| `conditionalAccessHooks` | `list<ConditionalAccessHook>` | Optional | View or action conditions evaluated at runtime | |

**DataFilterPolicy**

| Field | Type | Required | Description |
|---|---|---|---|
| `policyId` | id | Required | Unique identifier |
| `description` | string | Required | Plain-language description of the filter |
| `appliesTo` | string | Required | Entity type or `viewId` this policy applies to |

**ConditionalAccessHook**

| Field | Type | Required | Description |
|---|---|---|---|
| `hookId` | id | Required | Unique identifier — referenced in `ViewSpec.accessConditionHookId` |
| `description` | string | Required | Plain-language condition description |
| `evaluationTiming` | `enum(compile_time, runtime)` | Required | When the condition is evaluated |

**Rules:**
- The generator must not surface any action absent from `allowedActionIds`.
- `dataFilterPolicies` are opaque to the UI layer. The UI passes the `policyId` through; it does not implement filter logic.
- The UI layer is responsible for routing and affordance shaping — not policy enforcement.

---

### 4.13 AIAssistHook

Defines where and how AI assistance is available within this position app. All rules align with UX Doctrine P11 and P12.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `hookId` | id | Required | Unique identifier — referenced from tasks, decisions, actions, and views | |
| `hookType` | `enum(suggestion, draft_generation, anomaly_detection, summarization, auto_complete, classification)` | Required | Type of AI assistance | |
| `label` | string | Required | Plain-language label shown to the user | Example: `Suggest vendor`, `Summarize activity` |
| `triggerContext` | `enum(on_view_load, on_field_focus, on_user_request, on_exception_surface)` | Required | When the hook activates | |
| `outputType` | `enum(inline_suggestion, panel_result, field_fill, notification)` | Required | How the AI output is presented | |
| `isReversible` | boolean | Required | Whether the user can undo the AI output | Must be `true` unless `outputType=notification` |
| `requiresUserConfirmation` | boolean | Required | Whether the user must explicitly accept before output is applied | Aligns with P11, P12 |
| `affectsBusinessState` | boolean | Required | Whether accepting this output changes business state | If `true`, `requiresUserConfirmation` must be `true` |
| `explainabilityAvailable` | boolean | Required | Whether the user can ask why this suggestion was made | Seam: explainability service binding defined separately |

**Rules:**
- `affectsBusinessState=true` requires `requiresUserConfirmation=true`.
- `isReversible=false` is only permitted when `outputType=notification`.
- The user must always be aware that an AI action has occurred. Silent AI-applied changes are not permitted.

---

### 4.14 DefaultViewHints

Generator-time hints that configure the default experience. These are static schema declarations, not runtime decisions.

> **Ownership note:** `defaultLandingViewId` is intentionally absent here. Landing view is owned by `workSurface.navigation.defaultLandingViewId`.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `defaultDensityMode` | `enum(default, compact)` | Required | Initial density mode | Aligns with UX Doctrine D2 |
| `defaultSortSpecs` | `list<SortSpec>` | Optional | Default sort orders per view | |
| `defaultFilterSpecs` | `list<FilterSpec>` | Optional | Default filters per view | |
| `defaultExpandedSections` | `list<SectionHint>` | Optional | Which record sections are expanded by default | |
| `pinnedSummaryIds` | `list<id>` | Optional | State summary ids to pin at the top of the overview | References `stateSummaries` |

**SortSpec**

| Field | Type | Required | Description |
|---|---|---|---|
| `viewId` | id | Required | Which view this applies to |
| `fieldKey` | string | Required | The field to sort on |
| `direction` | `enum(asc, desc)` | Required | Sort direction |

**FilterSpec**

| Field | Type | Required | Description |
|---|---|---|---|
| `viewId` | id | Required | Which view this applies to |
| `fieldKey` | string | Required | The field to filter on |
| `filterDescription` | string | Required | Plain-language description of the filter value |

**SectionHint**

| Field | Type | Required | Description |
|---|---|---|---|
| `viewId` | id | Required | Which view this applies to |
| `sectionKey` | string | Required | The section identifier within that view |

---

### 4.15 PersonalizationHooks

Defines which surfaces can vary within this position app, at what scope, and what configuration they accept. Aligns with UX Doctrine §3.

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `allowedSurfaces` | `list<PersonalizationSurface>` | Required | All surfaces that can vary in this projection | |
| `conflictPolicy` | `enum(preserve_user, reset_to_role, notify_and_preserve)` | Required | Default behavior when a user preference is invalidated | Aligns with UX Doctrine §3.5 |

**PersonalizationSurface**

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `surfaceId` | id | Required | Unique identifier | |
| `surfaceType` | enum | Required | Type of variability surface | See vocabulary below |
| `scope` | `enum(global, tenant, role, user)` | Required | Maximum scope for configuration | Aligns with UX Doctrine §3.3 |
| `description` | string | Required | Plain-language description of what this surface controls | |
| `isStructuralVariant` | boolean | Required | Whether this surface changes layout structure | Structural variants require approved pattern variants |
| `targetViewIds` | `list<id>` | Optional | Which views this surface applies to | Empty = applies app-wide |
| `targetComponentTypes` | `list<string>` | Optional | Opaque downstream semantic-component identifiers this surface may target | Forward seam only; see rules |
| `configurationSchema` | `list<ConfigurationField>` | Optional | The shape of configuration this surface accepts | Defines what the configuration layer can change |

**ConfigurationField**

| Field | Type | Required | Description |
|---|---|---|---|
| `fieldKey` | string | Required | Unique identifier for this configuration option |
| `fieldLabel` | string | Required | Display label shown to the configurator |
| `valueType` | `enum(string, boolean, enum, number)` | Required | Type of the configuration value |
| `allowedValues` | `list<string>` | Optional | Enum values — required when `valueType=enum` |
| `defaultValue` | string | Optional | The default value at this scope |

**Surface type vocabulary**
- `visual_theme`
- `density_mode`
- `saved_filter`
- `sort_preference`
- `column_choice`
- `pinned_view`
- `section_expansion`
- `emphasis_variant`
- `structural_variant`

**Rules:**
- `surfaceType=structural_variant` may not be introduced without a corresponding approved variant in UI Pattern Library v0 (10.3).
- `isStructuralVariant=true` declares a compiled product artifact — an approved pattern variant — not a user-modifiable preference. Structural variant surfaces must have `scope=tenant` or `scope=role`. `scope=user` is not permitted.
- `conflictPolicy` applies globally to this projection. Per-surface conflict policies are not supported at v0.
- Precedence order (`global -> tenant -> role -> user`) governs override order at runtime. `conflictPolicy` governs what happens when a saved user preference is invalidated.
- `targetComponentTypes` is a **forward seam**, not a hard dependency. At v0 these are opaque semantic-component identifiers that the compiler may pass through. Hard validation against Semantic Component Library v0 (10.5) begins only once that work product exists.

---

### 4.16 ProjectionCompletenessSpec

Declares the completeness state of this projection's semantic slice. Required in every projection. Aligns with UX Doctrine §2.8: provisional state must be explicit; unknown must not masquerade as empty, zero, or complete.

The UI generator uses this section to distinguish between four states for any extension capability:

| State | Meaning | Generator behavior |
|---|---|---|
| Listed in `resolvedCapabilities` | Fully modeled and present in this projection | Normal rendering |
| Listed in `deferredCapabilities` | In scope for this position but not yet modeled in this slice | Render as provisional / "not yet available"; see C12 |
| Listed in `outOfScopeCapabilities` | Confirmed absent — this position genuinely does not need this capability | Render as absent / hidden |
| Not listed anywhere | Unknown | Treat as deferred; emit warning; see C12 |

| Field | Type | Required | Description | Constraints |
|---|---|---|---|---|
| `sliceStatus` | `enum(full, partial, provisional)` | Required | Overall completeness state of the semantic slice | Canonical source is semantic normalization step |
| `resolvedCapabilities` | `list<string>` | Required | Extension section names fully resolved and present in this projection | Example: `["tasks", "approvals"]` |
| `deferredCapabilities` | `list<string>` | Optional | Extension section names known to be in scope but not yet modeled | |
| `outOfScopeCapabilities` | `list<string>` | Optional | Extension section names confirmed not applicable to this position | |

**Rules:**
- If `sliceStatus=full`, `deferredCapabilities` must be empty or absent.
- `sliceStatus=provisional` signals that field names, state labels, and action semantics in the projection may still change. The generator must render the app in a provisional frame consistent with UX Doctrine §2.8.
- An extension section that is absent from the projection and not declared in `deferredCapabilities` or `outOfScopeCapabilities` is treated as deferred by the generator. The generator must not assume it is definitively absent.
- The union of `resolvedCapabilities`, `deferredCapabilities`, and `outOfScopeCapabilities` should account for all known extension section names. Missing known capability names cause a compilation warning and deferred treatment.
- `sliceStatus` is read-only to the UI layer. Its canonical source is semantic normalization, not UI generation.

---

## 5. Composition rules

Violations in this section are schema errors unless explicitly marked otherwise.

### 5.1 Projection-local composition rules

**C1 — Action references must resolve**  
Any `actionId` referenced in `tasks.availableActionIds`, `alerts.resolutionActionIds`, `collaborators.handOffActionIds`, `approvals.linkedActionId`, `decisions.linkedActionIds`, or `decisions.options[].triggersActionId` must have a corresponding entry in `actions`.

**C2 — View references must resolve**  
Any `viewId` referenced in `navigation.defaultLandingViewId`, `EntityTypeRef.viewLink`, `alerts.linkedViewId`, `approvals.linkedViewId`, `stateSummaries.linkedViewId`, `decisions.postDecisionNavigateToViewId`, or `actions.postActionNavigateToViewId` must match a `viewId` in `workSurface.views`.

**C3 — AI hook references must resolve**  
Any `aiAssistHookId` referenced in `tasks`, `decisions`, `actions`, or `patternInputBinding.aiAssistHookIds` must have a corresponding entry in `aiAssistHooks`.

**C4 — View-scoped hint references must resolve**  
Any `viewId` referenced in `defaultSortSpecs`, `defaultFilterSpecs`, or `defaultExpandedSections` must match a `viewId` in `workSurface.views`. Any `pinnedSummaryId` must resolve to a `summaryId` in `stateSummaries`.

**C5 — Permissions gate actions**  
Actions in `actions` that are absent from `permissionsHooks.allowedActionIds` must be treated as hidden by the generator. They must not be surfaced regardless of other references.

**C6 — Critical alerts require audit**  
Any `AlertSpec` with `severity=critical` must have `generateAuditEntry=true`.

**C7 — Approvers require approval views**  
Any projection with an `ApprovalSpec` where `thisPositionRole=approver` must have at least one view with `pattern=ApprovalReviewView`.

**C8 — State summaries require an overview view**  
Any projection with a non-empty `stateSummaries` section must have at least one view with `pattern=OverviewMonitor`.

**C9 — Destructive actions require full confirmation**  
Any `ActionSpec` with `isDestructive=true` must have `confirmationType=full`.

**C10 — Approval-gated actions require approval specs**  
Any `ActionSpec` with `requiresApproval=true` must have a corresponding `ApprovalSpec` referencing it via `linkedActionId`.

**C11 — Action slot binding must satisfy the action budget**  
In any `PatternInputBinding.actionSlots`: visible secondary actions must be no more than three; total visible actions (`1 primary + visible secondary count`) must be no more than five; any additional actions must go to `overflowActionIds`. Violations are projection-local schema errors.

**C12 — Deferred capabilities must not render as definitively absent**  
When an extension section name appears in `completeness.deferredCapabilities`, or when an extension section is absent from the projection and not declared in `completeness.outOfScopeCapabilities`, the generator must render the corresponding UI surface in a provisional or "not yet available" state. It must not render the surface as empty, zero, definitively absent, or complete.

**C13 — Structural variant surfaces must not be user-scoped**  
Any `PersonalizationSurface` with `isStructuralVariant=true` must have `scope=tenant` or `scope=role`. `scope=user` on a structural variant is a projection-local schema error.

### 5.2 Compiler-global validation rules

These checks require more than one projection, or access to compiler-maintained registries or adjacent work-product registries.

**G1 — Pattern and variant legitimacy**  
`ViewSpec.pattern` and `ViewSpec.patternVariant` must resolve against the approved pattern and variant registry defined by UI Pattern Library v0 (10.3) for the current `schemaVersion`. This is not validated from one projection in isolation.

**G2 — Cross-projection action label consistency**  
For the same tenant and `actionType`, the canonical `label` must be identical across projections. This is enforced by the compiler's Action Label Registry. A mismatch is a compiler-global validation failure.

**G3 — Breaking invariant changes require migration review**  
When a regenerated projection changes an invariant such as `viewId`, `positionId`, `positionSlug`, or a view's pattern family, the compiler must require a versioned migration plan before deployment. This requires comparison against prior projection history, not just the current projection instance.

---

## 6. Versioning strategy

### 6.1 Version types

| Version type | Scope | Tracks | Canonical owner |
|---|---|---|---|
| `schemaVersion` | Entire `PositionProjection` structure | Changes to the schema definition itself | Top-level `schemaVersion` |
| `positionVersion` | One specific position's compiled projection | Changes to that position's resolved projection | `identity.positionVersion` |

### 6.2 Breaking vs non-breaking changes

**Breaking — requires major version bump and migration plan**
- Removing or renaming a required field
- Renaming a field marked as invariant or canonical (`viewId`, `actionId`, `positionId`, `positionSlug`)
- Changing the type of an existing field
- Removing or renaming a value in an existing enum
- Changing a view's `pattern` family
- Changing the canonical label for the same `actionType` within a tenant

**Non-breaking — minor version bump**
- Adding a new optional field to any section
- Adding a new value to an existing enum
- Adding a new extension section
- Adding a new approved `PatternId` value together with corresponding pattern-library and generator support
- Reordering navigation items when the overall navigation model and canonical landing view remain stable

**Non-versioned — no schema bump required**
- Changes to descriptions or template strings that do not alter semantics
- Changes to display ordering that do not break invariants
- Changes to downstream seam implementations that preserve the same schema contract

**Notes:**
- Changing the value of an opaque downstream reference such as `telemetryKey` may require coordinated migration in the downstream layer, but is not by itself a schema-structure change unless the schema meaning or type changes.
- `positionVersion` may change without `schemaVersion` changing.

### 6.3 Backward compatibility rules

- A generator built against schema version `N` must consume a projection compiled against schema version `N-1` without failure.
- Unknown optional fields must be silently ignored for forward compatibility on minor bumps.
- Breaking changes require a migration plan before the regenerated projection is deployed.
- Per-section compatibility tracking may be introduced later if extension section lifecycles diverge materially from core section lifecycles.

### 6.4 Deprecation windows for major version bumps

When a major schema version bump occurs, live tenants may be running projections compiled against the previous major version. The following rules define the operational window:

| Phase | Definition | Duration |
|---|---|---|
| Dual-support window | Generator supports both version `N` and `N-1` projections simultaneously | Minimum 2 full regeneration cycles for the affected tenant; defined per deployment |
| Migration-required notice | Tenants are notified that `N-1` projections must be regenerated against `N` | Issued at the start of the dual-support window |
| Hard cutoff | Generator drops support for `N-1` projections | After the dual-support window closes; `N-1` projections are rejected at compile time |

**Operational rules:**
- A tenant's position app continues to run the `N-1` compiled artifact during the dual-support window. No forced downtime.
- The compiler must refuse to generate new `N-1` projections once the major bump is live. Only existing `N-1` artifacts remain valid during the window.
- The migration plan for a major bump must include: a list of affected fields, the transformation required, and automated migration tooling where feasible.
- Exact window durations are deployment decisions, not schema decisions. This section defines the minimum structure; specific timelines are set per release by the workstream owner.

---

## 7. Open seams

Integration points where other Foundry layers connect. Identified here to prevent over-engineering at v0 while preserving the connection points for later.

| Seam | Connects to | Current status at v0 |
|---|---|---|
| `boundedContextId` in `PositionIdentity` | Semantic layer bounded context model | Placeholder id; full context definition is out of scope |
| `primaryEntityType` in `ViewSpec` and `entityType` in `EntityTypeRef` | Full entity model in semantic layer | Entity types are named here; field definitions and schema live in the semantic layer |
| `pattern` and `patternVariant` in `ViewSpec` | UI Pattern Library v0 (10.3) | Closed v0 vocabulary here; legitimacy and approved variants validated against 10.3 |
| `PatternInputBinding` per `ViewSpec` | UI Pattern Library v0 (10.3) | Per-view binding declared here; authoritative pattern input contracts defined in 10.3 |
| `targetComponentTypes` in `PersonalizationSurface` | Semantic Component Library v0 (10.5) | Opaque forward seam; hard validation deferred until 10.5 exists |
| `accessConditionHookId` in `ViewSpec` | Policy layer | Opaque to UI layer; `hookId` passed at runtime for policy evaluation |
| `dataFilterPolicies` in `PermissionsHooks` | Policy layer | Opaque to UI layer; `policyId` passed through; filter logic is not implemented in UI |
| `permissionsVersion` | Policy schema/versioning | Version matched at compile time; runtime enforcement by policy layer |
| `telemetryKey` in `ActionSpec` | Evaluation/instrumentation layer | Opaque instrumentation reference; semantics and lifecycle owned by instrumentation spec |
| `aiAssistHooks` | AI service layer | Hook type and trigger declared here; service binding defined separately |
| `explainabilityAvailable` in `AIAssistHook` | AI explainability service | Declared here; implementation and response format live outside this schema |
| `fieldKey` values in `KeyFactSpec`, `SortSpec`, `FilterSourceSpec`, `ContextField` | Semantic entity field model | Field keys named here; field types and validation live in the semantic layer |
| Cross-projection Action Label Registry | Compiler layer | Tenant-scoped `actionType -> canonical label` index enforced by compiler-global validation |
| Event and lifecycle modeling | Future event/lifecycle layer | Not modeled in this schema at v0; alerts cover exception surfacing only |
| `completeness.sliceStatus` origin | Semantic normalization step | `sliceStatus` is populated by normalization and consumed read-only by UI generation |
| Provenance | Provenance layer | Every projection change should carry provenance; captured externally, not in this schema |
| Alert escalation execution | Workflow/orchestration layer | `warningEscalationHours` declared here; escalation execution handled by orchestration |

---

## 8. Milestone 2 exit criteria — self-assessment

| Exit criterion | Status | Evidence |
|---|---|---|
| Schema covers tasks, entities, actions, events, approvals, and state | Met | `TaskSpec`, `MonitoredEntitySpec`, `ActionSpec`, `AlertSpec`, `ApprovalSpec`, `StateSummarySpec` |
| Personalization hooks and scopes are identified | Met | `PersonalizationHooks`; scopes align with UX Doctrine §3.3 and §3.4 |
| Schema is sufficiently narrow to remain stable | Met | UI-facing only; excludes backend contracts, policy logic, provenance internals, and runtime instances |
| Open seams for policy/provenance/AI/instrumentation are identified without overcommitting | Met | §7 Open seams lists each integration point and ownership boundary |
| Schema can represent partial and provisional semantic slices explicitly | Met | `ProjectionCompletenessSpec` + C12 |
| Validation scopes are explicit | Met | §2.3 + split between projection-local and compiler-global rules in §5 |
| Canonical ownership is explicit | Met | `schemaVersion`, `positionVersion`, and `defaultLandingViewId` ownership clarified |
| Pattern vocabulary is stable enough for v0 without over-freezing later evolution | Met | Closed `PatternId` vocabulary for v0 with explicit migration path to registry-backed identifiers later |
| Personalization surface scoping is aligned with UX Doctrine §3.4 three-way distinction | Met | Structural variants limited to tenant/role scopes; user personalization kept distinct |

---

*This document is Final v0. It preserves the full field-level projection contract while clarifying canonical ownership, validation scope, and compiler boundaries. It is intended to govern UI pattern selection and position app generation until replaced by a versioned successor.*
