# Foundry UX Doctrine v0

**Status:** First working draft  
**Workstream:** UI/UX — Work Product 10.1  
**References:** Foundry UI/UX Workstream Charter v1, Foundry2 Position-Centric Regenerative Software for SMBs v1  
**Purpose:** Define the governing interaction principles, rules, and invariants for all Foundry position apps.

This document is a first-class artifact. All subsequent work products — UI pattern library, semantic component library, personalization model, primitive mapping — must be consistent with it.

**Scope:** This doctrine applies to all UI patterns, flows, and components generated for position apps. It is the authority against which all pattern specifications, component designs, and regenerated releases are evaluated.

---

## 1. Consumer-grade ease — operational definition

Consumer-grade ease means SMB workers encounter no learning curve simply from the product's interaction model. The reference bar is not a competing ERP. It is the best apps those workers already use daily.

**1.1 Obvious next step**  
At every point in a workflow, the user can identify the single next action without reading instructions. One action is visually dominant. All others are accessible but not competing.

**1.2 Navigation they already know**  
Navigation follows patterns established by consumer apps. Novel navigation models require explicit justification and must be validated before adoption.

**1.3 Minimal friction on entry**  
The most-needed input — a search box, a primary form, an action button — is immediately reachable without opening a menu or navigating to a sub-page first.

**1.4 Progressive disclosure by default**  
Advanced options, secondary fields, and edge-case controls are hidden until needed. No screen presents every possible field or action upfront.

**1.5 Fast and visible feedback**  
Every user action produces immediately perceptible feedback. Loading, saving, and processing states are communicative. Silent waits are not acceptable. No action should feel unacknowledged; exact timing targets are defined at the pattern specification level, not here.

**1.6 Forgiving interactions**  
Undo is available for destructive actions. Confirmations are proportional to risk — not overused for minor actions, always required before irreversible ones. Validation messages identify the problem and suggest a fix; they do not punish the user.

**1.7 Purposeful empty states**  
An empty view always explains why it is empty and gives the user a direct path to take action. Blank tables and lists are not acceptable.

---

## 2. Enterprise-grade rigor — UX definition

Enterprise-grade rigor means position apps can be trusted in operational business contexts: workflows are traceable, states are clear, errors can be recovered from, and approvals are first-class.

**2.1 Traceability**  
The user can see what changed, when, who initiated it, and what approved it. Activity timelines and state history are required components for any entity that participates in a business workflow.

**2.2 Correctness guarantees**  
Validation happens at both field and form levels. Submitting invalid data is prevented, not silently allowed. The system actively prevents conflicting states.

**2.3 Approval as a first-class pattern**  
Approval workflows are first-class UX patterns. The user can always see what is pending, who must act, and the consequence of the approval. Approval is never an afterthought added to an existing screen.

**2.4 State before action**  
The current state of a record or workflow is always visible before action controls appear. Users do not encounter action buttons without a clear view of current state.

**2.5 Recoverability**  
The user can recover from errors. Where rollback is technically possible, it must be surfaced. Errors are explained in plain language that names the problem and states what to do.

**2.6 Role-boundary enforcement**  
Position apps do not surface actions or data the user is not authorized to access. Unauthorized items are absent, not merely disabled. Exceptions apply only when the user needs to see that an action exists but requires a prerequisite.

**2.7 Auditability hooks**  
The UX supports audit trails at all key decision and action points. Every primary action, flow completion, flow abandonment, error encounter, and empty-state view requires a telemetry hook. Telemetry is part of the component contract, not an optional addition.

---

## 3. Bounded personalization — product definition

Bounded personalization means a position app feels like the user's own work surface while remaining stable, generatable, and supportable. Personalization is delivered through approved surfaces, not through free-form structural invention.

### 3.1 What is personalized

| Surface | Scope | Examples |
|---|---|---|
| Visual layer | Token-driven | Theme, density mode, color emphasis, spacing feel |
| View state | Saved per user | Filters, sort order, column choices, pinned views, default sections |
| Emphasis variants | Approved slot differences | Which content is prominent within a defined pattern slot |
| Limited structural variants | Explicitly modeled only | Alternate layout regions where explicitly specified and tested |

### 3.2 What is not personalized

- Navigation logic and structure
- Action semantics and placement rules
- State semantics and validation behavior
- Approval and confirmation interaction patterns
- Core pattern grammar, slot definitions, and composition rules

### 3.3 Personalization scope and precedence

Precedence applies from lowest to highest. Higher-scope settings override lower-scope defaults.

1. **Global** — Foundry product defaults
2. **Tenant** — per-company configuration
3. **Role** — per-position defaults
4. **User** — per-user saved preferences

### 3.4 The invariant rule

Personalization must never change the interaction grammar. A personalized app and the default app must be semantically identical. Only presentation, emphasis, density, and saved view state may differ.

### 3.5 Conflict resolution

Saved personalization can become invalid after a regeneration or a scope-level default change. Four scenarios must be handled explicitly.

**Scenario A — Orphaned preference (field or filter no longer exists after regeneration)**  
The saved preference silently refers to a field, filter dimension, or view that the regenerated model no longer exposes. Resolution: discard the orphaned portion of the preference. Do not apply it partially in a way that produces an inconsistent view. Notify the user with a plain-language explanation of what was reset and why. Offer a direct path to reconfigure.

**Scenario B — Role-level default overrides a user preference**  
A role default introduced or changed after a regeneration covers a surface that the user has already personalized. Resolution: role defaults do not retroactively overwrite user preferences unless the user preference references something that no longer exists (see Scenario A). The user's saved preference is preserved where it remains valid. If the role default introduces a new surface the user has not yet configured, the role default fills that gap.

**Scenario C — Tenant default changes after regeneration**  
A tenant-level configuration change affects defaults below it. Resolution: role and user preferences above the tenant level are preserved where still valid. The new tenant default fills any surfaces not covered by role or user preferences.

**Scenario D — Structural pattern change invalidates a saved view**  
The underlying pattern for a screen changes (for example, a Collection View becomes an Overview/Monitor). Saved view state tied to the old pattern's slots is now structurally invalid. Resolution: treat as Scenario A. Discard the view state that cannot be applied. Notify the user. Do not apply partial or mismatched view state silently.

**General conflict resolution rule**  
When a conflict cannot be resolved silently with full fidelity, the system must: (1) notify the user, (2) explain what changed in plain language, (3) preserve what is still valid, (4) offer a path to reconfigure what was reset. Silent loss of personalization is not acceptable.

---

## 4. Non-negotiable interaction principles

These rules apply to every position app, every UI pattern, and every regenerated release.

**P1 — One primary action per context**  
Every screen or modal has one visually dominant primary action. Multiple competing primary actions are not permitted.

**P2 — Action-budget discipline**  
The visible action count on any screen or modal is bounded. The budget is defined in section 6. It is a hard limit, not a guideline.

**P3 — Search-first on collection views**  
Any collection view with more than one page of data provides immediate, visible search. Filtering follows the same rule when multiple filter dimensions exist.

**P4 — State before action**  
The user sees current state before action controls. Action controls do not appear before state is communicated.

**P5 — Confirmation proportional to risk**  
Low-risk actions execute immediately. Medium-risk actions use inline confirmation. High-risk or irreversible actions require a distinct confirmation step with an explicit description of the consequence.

**P6 — No orphaned flows**  
Every multi-step flow has a clear exit at every step, with explicit handling for save draft, cancel, and abandon.

**P7 — Error messages in plain language**  
Error messages name the problem, identify the field or action involved, and suggest resolution. System error codes and technical identifiers are not exposed to the user.

**P8 — Consistent semantic labels**  
The same business action uses the same label in every position app. Synonyms for the same concept are not permitted (for example, Submit, Save, and Confirm must not be used interchangeably for the same semantic action).

**P9 — Accessibility is a floor, not a feature**  
All patterns and components meet WCAG 2.1 AA as the minimum. Keyboard navigation, focus management, and screen reader annotations are required. Accessibility must survive regeneration.

**P10 — Telemetry hooks are part of the contract**  
Every primary action, flow completion, flow abandonment, error encounter, and empty-state view has a telemetry hook. Hook names are stable and treated as a breaking change if modified.

**P11 — AI-assisted actions are explainable, reversible, and distinguishable**  
AI-assisted actions are clearly distinguished from user-initiated actions at the point of presentation. The user can understand why the system made a suggestion, reverse any AI-initiated action, and override any AI recommendation without penalty. AI must not silently alter business state.

**P12 — User retains final control**  
The system may guide, suggest, and recommend actions. Final decision authority remains with the user unless the user has explicitly delegated an action class to automation. Autonomous system actions that affect business state must be surfaced, logged, and reversible.

**P13 — Semantic consistency across position apps**  
The same business action carries the same semantic meaning and uses the same label across all position apps. An action that means "Submit for approval" in one position app must not be labeled or behave differently in another. Cross-position consistency is an invariant, not a goal.

**Exception handling**  
Exceptions to these principles require explicit pattern-level justification and must be declared as approved variants. An undeclared exception is a compliance failure, not a design choice.

---

## 5. Default density rules

**D1 — Low density by default**  
The default density is comfortable on a 1080p or higher desktop screen. Generous spacing is the baseline. Power-user density is not the default.

**D2 — Two explicit modes only**  
Supported density modes: Default (comfortable) and Compact (higher information density). No other modes are permitted.

**D3 — Compact mode preserves interaction grammar**  
Compact mode reduces spacing, adjusts font size within range, and reduces whitespace. It must not increase visible actions beyond the action budget, expose additional fields, or change the interaction grammar.

**D4 — Mobile scope is explicit and limited**  
Primary device target is desktop. Mobile support is scoped to monitoring, approval review, and assisted workflows. Full data-entry flows are desktop-only unless a mobile-specific pattern is explicitly specified.

**D5 — No raw data grids by default**  
Lists are presented as semantic record lists, not raw spreadsheet grids. Grid-style views are a compact structural variant, available only where explicitly modeled and supported.

---

## 6. Action-budget rules

| Slot | Limit |
|---|---|
| Primary actions per context | 1 |
| Visible secondary actions | 3 maximum |
| Total visible actions before overflow | 5 |
| Overflow trigger | Any count above 5 |

**A1 — One primary action per context**  
One primary action per screen or modal. This is the single most important action from this context.

**A2 — Secondary actions: max 3 visible**  
Secondary actions may appear alongside the primary. No more than three are visible without overflow or menu expansion.

**A3 — Overflow is a last resort**  
Overflow menus are not a design convenience. They exist to enforce the budget, not to hide actions from discovery planning.

**A4 — Bulk actions replace individual actions**  
Bulk action controls appear only when items are selected. During selection state, bulk actions replace or suppress individual-item actions.

**A5 — Destructive actions are isolated**  
Destructive or irreversible actions are visually separated from the primary action group. Adjacency to the primary action button is not permitted.

**A6 — Context-sensitive actions are hidden, not disabled**  
Actions that are currently unavailable are hidden. Exceptions apply only when the user needs to understand an action exists but requires a prerequisite. In that case, the action is disabled with an explanatory tooltip.

---

## 7. Guidance for search, filtering, forms, approvals, and exceptions

### 7.1 Search

- Immediate, visible search input on all collection views with more than one page of records.
- Search is always visible. It is never hidden behind a filter panel toggle.
- Default search scope covers the most relevant fields: typically name, identifier, and status.
- Matched terms in results are highlighted.
- Empty search results provide a clear message and an action path to reset or modify the query.

### 7.2 Filtering

- Filters supplement search; they do not replace it.
- Active filter count is always visible when filters are applied.
- A single-action clear-all affordance is always present when filters are active.
- Filter state is saveable as a named view where the user performs this task regularly.
- Filter panels are progressive: common filters are visible by default; advanced filters are behind expansion.

### 7.3 Forms

- Forms use guided section structure, not a flat field list.
- Required fields are clearly marked.
- Validation triggers on field exit (blur), not only on form submit.
- Inline validation errors appear adjacent to the affected field, not only at the top of the form.
- Long forms use progressive disclosure: the relevant section is expanded; the rest are collapsed or stepped.
- Create flows and edit flows may differ in structure. Edit flows may include read-only context fields not present in create flows.
- Autosave with drafts is preferred for long forms. If not supported, the user is warned before data loss occurs.

### 7.4 Approvals

- Approval patterns always show: what is being approved, who initiated it, when, and why.
- Approve and Reject actions are both visible and clearly distinct from each other.
- Rejection always requires a reason via a text input.
- Approval history is visible on the record.
- Pending approvals appear in the approver's position app work surface. Email notification is supplementary, not the primary surface.

### 7.5 Exceptions

Exceptions are not equal. A missing optional field, a failed payment, and a compliance flag have different urgency and require different surfacing behavior. All exceptions are classified into one of three severity tiers.

#### Severity tiers

| Tier | Definition | Business impact |
|---|---|---|
| **Informational** | Awareness only. No immediate action required. | None at present; monitor. |
| **Warning** | Action recommended. Not immediately urgent but will have business impact if unresolved within a defined window. | Partial or degraded; recoverable without escalation. |
| **Critical** | Immediate action required. Business continuity, financial integrity, or compliance is at risk. | Blocked or at risk; escalation may be required. |

#### Tier behavior

**Informational**
- Surfaced as an inline notice or low-prominence alert on the relevant record or view.
- Does not interrupt the user's current workflow.
- Dismissible without a required action.
- Does not appear in the primary exception work-surface queue unless the user has opted in.

**Warning**
- Surfaced as a persistent alert on the affected record and in the user's exception work-surface queue.
- Includes: what happened, which entity is affected, and the recommended action.
- Resolution is actionable from the exception surface without navigating away.
- If unresolved beyond the pattern-defined window, escalates automatically to Critical.
- After resolution, dismissed or updated in place.

**Critical**
- Surfaced prominently on the user's primary work surface, the exception queue, and the affected record simultaneously.
- Includes: what happened, which entity is affected, the consequence of non-resolution, the recommended action, and who else can act if the current user cannot.
- Resolution is actionable from the exception surface without navigating away.
- Cannot be silently dismissed. Requires an explicit resolution action or an explicit escalation/hand-off action.
- Generates an audit entry regardless of resolution outcome.

#### General exception rules

- Every exception must communicate: what happened, which entity is affected, and the recommended action. This is true at all tiers.
- Exceptions that cannot be resolved by the current user must name who can resolve them and provide a direct path to hand off.
- After resolution, the exception is dismissed or updated in place at all surfaces where it appeared.
- Severity must be declared by the pattern or component specification. Ad hoc severity assignment at the implementation level is not permitted.

---

## 8. Invariants that must survive regeneration

These invariants are enforced on every regenerated release of a position app. Any change to these properties is a breaking change requiring review.

**I1 — Navigation structure**  
Navigation model must not change between regenerations without a deliberate versioned migration. Users must not find their navigation reorganized unexpectedly.

**I2 — Primary action semantics**  
Primary action label and semantic meaning for a given record state must remain stable. If a model change requires a primary action to change, this is a breaking change requiring explicit review.

**I3 — State labels**  
Business state labels (pending, approved, rejected, in-progress, and equivalent) must remain stable. Regeneration must not introduce synonyms or alternates.

**I4 — Pattern grammar**  
A screen regenerated as a Collection View must remain a Collection View. Pattern type is not permitted to change silently during regeneration.

**I5 — Personalization data**  
User-saved view preferences, saved filters, and named views are user-owned data. Regeneration must not overwrite or invalidate personalization data without a migration path.

**I6 — Accessibility**  
Accessibility properties — focus management, aria labels, keyboard navigation — must not be degraded between regenerations.

**I7 — Telemetry hook keys**  
Telemetry event names and hook keys remain stable between regenerations. A change in hook key is a breaking change and must be logged with a migration record.

**I8 — Action budget compliance**  
No regenerated screen may exceed the action budget defined in section 6. Automated audit of action counts is a required part of the generation evaluation pipeline.

**I9 — Cross-position semantic consistency**  
Business action labels and semantics must remain consistent across all position apps between regenerations. A change that alters the label or meaning of a shared business action in one position app without aligning all other position apps is a breaking change requiring coordinated review.

---

## 9. Fiori / Carbon / Fluent reference delta

Foundry takes deliberate positions relative to its three primary enterprise UI references. Each is used for a different purpose and carries a different delta. This matrix is the shared language for evaluating all subsequent pattern and component specifications.

- **SAP Fiori**: reference for ERP semantics, role-based thinking, list/detail/task patterns, and stateful business workflows.
- **IBM Carbon**: reference for information architecture discipline, structured layout, and content hierarchy.
- **Microsoft Fluent**: reference for approachable interaction patterns, accessibility defaults, and familiarity for business workers.

### 9.1 Comparative delta matrix

| Dimension | SAP Fiori | IBM Carbon | Microsoft Fluent | Foundry position |
|---|---|---|---|---|
| Primary purpose borrowed | ERP role/task semantics | Information architecture structure | Approachable interaction patterns | All three, selectively |
| Default density | High; data-dense | High; structured grids | Medium; comfortable for Office users | Lower; SMB comfort baseline |
| Action visibility | Many toolbar actions visible | Actions in structured panels | Context menus and command bars | Strict budget; overflow enforced |
| Progressive disclosure | Limited; most fields visible | Moderate; structured tabs | Moderate; wizard patterns | Required by default |
| Navigation model | Role-based launchpad + Fiori tiles | Structured sidebar + breadcrumbs | Pivot/tab and navigation pane | Position app as primary unit |
| Form structure | Flat field layout common | Structured sections with type scale | Wizard and panel patterns | Guided sections; progressive field reveal |
| Search placement | Variable; top-of-page or filtered | Global search prominent | Command bar search | Always immediate; always visible |
| Empty states | Minimal; often absent | Structured; sometimes generic | Present but variable | Always communicative with action path |
| Personalization | Rigid; limited user control | Rigid; tenant/admin controlled | Some view preferences | Bounded personalization as first-class |
| Mobile | Responsive but not mobile-first | Responsive grid; not mobile-first | Moderate mobile consideration | Explicitly scoped to approve/monitor/assist |
| Interaction tone | Enterprise formal | Enterprise structured | Familiar; Office-adjacent | Approachable; consumer-grade ease |
| Accessibility | Partial; improving | Strong foundation | Strong by design | Required floor at WCAG 2.1 AA |

### 9.2 SAP Fiori — keep / change / discard

**Keep**
- Role- and position-based app as the primary structural unit
- List-detail navigation as a foundational pattern
- Stateful business workflows with explicit state management
- Approval and task surfaces as first-class elements

**Change**
- Dense multi-action toolbars → action-budget-compliant surfaces
- Flat field forms → guided section forms with progressive disclosure
- Implicit or absent empty states → communicative empty states with action paths
- Module-centric launchpad as primary entry → position app work surface as primary entry

**Discard**
- Everything-visible action bars on collection and detail views
- Technical or code-based validation and error messages exposed to the user
- Big-bang module onboarding; replace with pain-point-first progressive adoption

### 9.3 IBM Carbon — keep / change / discard

**Keep**
- Strong information hierarchy: clear typographic scale, consistent header-body-metadata distinction
- Grid discipline: structured layout regions that avoid visual chaos
- Content-first layout: the primary content area is dominant; chrome is minimal
- Structured use of panels and sections for complex record pages

**Change**
- High-density defaults calibrated for technical users → lower density calibrated for SMB operators
- Deep information architecture with many nested levels → shallower hierarchy matched to SMB task complexity
- Admin-controlled rigidity → bounded personalization with user-level surfaces
- Generic enterprise component vocabulary → semantic business-aware components

**Discard**
- Data table as the default list presentation; replace with semantic record lists
- Enterprise-first component complexity (multi-level structured menus, dense data grids) for SMB work surfaces
- Carbon's assumption that information architecture is set by admins and fixed; Foundry allows user-level view state

### 9.4 Microsoft Fluent — keep / change / discard

**Keep**
- Approachable interaction patterns: familiar affordances that SMB workers recognize from Office and everyday tools
- Accessibility-first defaults: focus management, keyboard navigation, and screen reader support as designed-in properties
- Motion and feedback: purposeful transitions that communicate state change rather than decorate
- Familiar business worker mental models: list, detail, task, and panel structures that Office users already understand

**Change**
- Windows-native visual grammar → neutral web-native visual language without OS-specific affordances
- Office-style ribbon and menu bars → strict action budget with overflow enforcement
- Desktop application component metaphors → web-first patterns appropriate for browser-delivered position apps
- Microsoft-brand color and type defaults → Foundry token system with bounded personalization

**Discard**
- Ribbon bars and command bars with many simultaneously visible action categories
- Complex cascading menu hierarchies; replace with progressive disclosure and inline contextual actions
- Desktop application structural patterns (task panes, split views with many simultaneous panels) where they conflict with the SMB simplicity target

---

## 10. Shared evaluation language

All subsequent specifications in this workstream — UI patterns, semantic components, personalization model, prototype workflows — must be evaluated against the following questions derived from this doctrine.

| Evaluation question | Derived from |
|---|---|
| Is there one primary action? | P1, A1 |
| Is the action budget within limit? | P2, section 6 |
| Does the user see state before action? | P4 |
| Is search immediately visible where required? | P3, 7.1 |
| Are all error messages in plain language? | P7, 7.3 |
| Is progressive disclosure applied? | 1.4, D3 |
| Is the empty state communicative with an action path? | 1.7, 7.1 |
| Are approvals first-class? | 2.3, 7.4 |
| Is density compliant with default or compact mode? | section 5 |
| Does personalization stay within bounded surfaces? | section 3 |
| Does this survive the regeneration invariants? | section 8 |
| Is the Foundry delta from Fiori maintained? | §9.2 |
| Is the Carbon information architecture contribution applied correctly? | §9.3 |
| Is the Fluent approachability contribution applied correctly? | §9.4 |
| Is the AI action explainable, reversible, and distinguishable? | P11 |
| Does the user retain final control? | P12 |
| Are semantic labels consistent across all position apps? | P13, I9 |
| If a personalization preference is invalidated, is conflict resolution applied? | §3.5 |
| Is each exception assigned a severity tier? | §7.5 |
| Is Critical exception behavior non-dismissible and audit-logged? | §7.5 |
| If a principle is violated, is it declared as an approved variant? | Exception handling, §4 |

---

*This document is version 0. It is the first executable version sufficient to evaluate subsequent work products. It is expected to be refined as the pattern library and component library are validated against real workflows.*
