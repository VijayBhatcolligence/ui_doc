# Foundry UX Doctrine v0

**Status:** Revised v0 — patch round 1  
**Workstream:** UI/UX — Work Product 10.1  
**References:** Foundry UI/UX Workstream Charter v1, Foundry2 Position-Centric Regenerative Software for SMBs v1  
**Purpose:** Define the governing interaction principles, rules, and invariants for all Foundry position apps.

This document is a first-class artifact. All subsequent work products — UI pattern library, semantic component library, personalization model, primitive mapping — must be consistent with it.

**Scope:** This doctrine applies to all UI patterns, flows, and components generated for position apps. It is the authority against which all pattern specifications, component designs, and regenerated releases are evaluated.

**Doctrine as generation constraint:** This doctrine is both a product artifact and a generation constraint. It governs how agents and downstream compiler stages may map position-facing business intent into UI. It constrains pattern selection, approved variant selection, semantic component composition, regeneration review, and downstream evaluation gates. Any generated UI that falls outside this doctrine is invalid by default. It does not define lower-level implementation detail unless explicitly marked as invariant.

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
The user can see what changed, when, who initiated it, and what approved it. Any entity that participates in a business workflow must expose change history and state history in a consistent, reviewable form. The exact component realization belongs to pattern and semantic-component specifications.

**2.2 Correctness guarantees**  
Validation happens at both field and form levels. Submitting invalid data is prevented, not silently allowed. The system actively prevents conflicting states.

**2.3 Approval as a first-class pattern**  
Approval workflows are first-class UX patterns. The user can always see what is pending, who must act, and the consequence of the approval. Approval is never an afterthought added to an existing screen.

**2.4 State before action**  
See P4.

**2.5 Recoverability**  
The user can recover from errors. Where rollback is technically possible, it must be surfaced. Errors are explained in plain language that names the problem and states what to do.

**2.6 Role-boundary enforcement**  
Position apps do not surface actions or data the user is not authorized to access. Unauthorized items are absent, not merely disabled. Exceptions apply only when the user needs to see that an action exists but requires a prerequisite.

**2.7 Observability and auditability**  
Key decision points, state transitions, flow completions, abandonments, and exceptional outcomes must be observable and auditable. Exact event schemas, hook names, and hook-key compatibility rules belong to the evaluation and instrumentation layer, not to this doctrine.

**2.8 Progressive incompleteness and provisional-state UX**  
Foundry position apps must remain usable even when the semantic slice is partial. Foundry2 explicitly assumes start-anywhere, pain-point-first, incomplete-knowledge-acceptable operation. The UX must reflect that operating model.

- **Provisional state must be explicit.** Unknown, provisional, or not-yet-modeled state must be communicated clearly to the user.
- **Unknown must not masquerade as zero, empty, complete, or approved.** The absence of data and the confirmed emptiness of data are distinct states.
- **Partial slices must remain operable for the in-scope pain point.** A position app may be incomplete in scope; it must not be ambiguous in state.
- **Missing downstream capability must surface a next-best action, not a dead end.** When a required downstream workflow is not yet modeled, the UI must offer the user a next-best path rather than a silent failure or blank screen.
- **Generated UX may be incomplete in scope, but never ambiguous in state.**

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

### 3.4 Personalization vs approved variants vs pattern migration

These three categories are distinct and must not be conflated.

**User personalization** is limited to approved surfaces within the current pattern and variant. It must not change the interaction grammar.

**Approved pattern variants** are versioned product artifacts. They may change structure only within explicitly modeled bounds. They are not user personalization and are not governed by user preferences.

**Pattern-family changes** are not personalization. A screen moving from one pattern family to another (for example, Collection View to Overview/Monitor) requires explicit versioned migration and must not occur silently across regeneration. See I4.

### 3.5 Conflict resolution

Personalization must be precedence-defined, migration-safe, and loss-aware. When a saved preference is invalidated by regeneration or a scope-level default change, the system must: (1) preserve what remains valid, (2) discard only what cannot be applied without producing an inconsistent state, (3) notify the user in plain language, and (4) offer a direct path to reconfigure. Silent loss of personalization is not acceptable.

*Detailed conflict scenarios — orphaned preferences, role-override collision, tenant-default change, structural-pattern change — belong to Pattern Variability and Personalization Model v0.*

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
The same business action uses the same label in every screen, context, and position app. Synonyms for the same concept are not permitted — Submit, Save, and Confirm must not be used interchangeably for the same semantic action. This applies within a position app and across all position apps. See also I8.

**P9 — Accessibility is a floor, not a feature**  
All patterns and components meet WCAG 2.1 AA as the minimum. Keyboard navigation, focus management, and screen reader annotations are required. Accessibility must survive regeneration.

**P10 — Observability**  
Workflows and outcomes must be observable and auditable. See §2.7. Exact event schemas, hook names, and hook-key compatibility rules are instrumentation-layer concerns, not doctrine.

**P11 — AI-assisted actions are explainable, reversible, and distinguishable**  
AI-assisted actions are clearly distinguished from user-initiated actions at the point of presentation. The user can understand why the system made a suggestion, reverse any AI-initiated action, and override any AI recommendation without penalty. AI must not silently alter business state.

**P12 — User retains final control**  
The system may guide, suggest, and recommend actions. Final decision authority remains with the user unless the user has explicitly delegated an action class to automation. Autonomous system actions that affect business state must be surfaced, logged, and reversible.

**P13 — Pattern and variant legitimacy**  
All generated UI must resolve to an approved pattern and, where applicable, an approved variant. Structures not declared as an approved pattern or variant are invalid generation output.

**Exception handling**  
Exceptions to these principles require explicit pattern-level justification and must be declared as approved variants. An undeclared exception is a compliance failure, not a design choice.

---

## 5. Default density rules

**D1 — Low density by default**  
The default density is comfortable for the primary desktop operating surface. Generous spacing is the baseline. Power-user density is not the default.

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

*These are v0 doctrine-level defaults. They must be validated against pilot workflows before being treated as permanent invariants.*

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
- Validation should occur early enough to avoid late surprise and close enough to the affected field to support correction. Exact trigger timing belongs to the pattern specification.
- Inline validation errors appear adjacent to the affected field, not only at the top of the form.
- Long forms use progressive disclosure: the relevant section is expanded; the rest are collapsed or stepped.
- Create flows and edit flows may differ in structure. Edit flows may include read-only context fields not present in create flows.
- Autosave with drafts is preferred for long forms. If not supported, the user is warned before data loss occurs.

### 7.4 Approvals

- Approval patterns always show: what is being approved, who initiated it, when, and why.
- Approve and Reject actions are both visible and clearly distinct from each other.
- Rejection or denial paths must capture rationale where the decision materially changes business state, auditability, or hand-off clarity. Exact input mechanics belong to the pattern specification.
- Approval history is visible on the record.
- Pending approvals appear in the approver's position app work surface. Email notification is supplementary, not the primary surface.

### 7.5 Exceptions

Exceptions are not equal. All exceptions are classified into one of three severity tiers. Severity must be declared by the pattern or component specification; ad hoc severity assignment at the implementation level is not permitted.

| Tier | Definition | Surfacing | Dismissal | Escalation | Auditability |
|---|---|---|---|---|---|
| **Informational** | Awareness only; no immediate action required | Inline notice or low-prominence alert on the relevant record or view | Dismissible without required action | None | Not required |
| **Warning** | Action recommended; will have business impact if unresolved | Persistent alert on affected record and in the exception queue; includes what happened, which entity is affected, and the recommended action | Resolvable in place | Escalates to Critical after the pattern-defined window lapses | Optional |
| **Critical** | Immediate action required; business continuity, financial integrity, or compliance at risk | Prominent on the user's primary work surface, exception queue, and affected record simultaneously; includes consequence of non-resolution and who else can act | Cannot be silently dismissed; requires explicit resolution or escalation action | Names who else can act if current user cannot | Required regardless of resolution outcome |

**General exception rules**

- Every exception must communicate: what happened, which entity is affected, and the recommended action. This is true at all tiers.
- Exceptions that cannot be resolved by the current user must name who can resolve them and provide a direct path to hand off.
- After resolution, the exception is dismissed or updated in place at all surfaces where it appeared.

---

## 8. Invariants that must survive regeneration

These invariants are enforced on every regenerated release of a position app. Any change to these properties is a breaking change requiring review.

**I1 — Navigation structure**  
Navigation model must not change between regenerations without a deliberate versioned migration. Users must not find their navigation reorganized unexpectedly.

**I2 — Primary action semantics**  
Primary action label and semantic meaning for a given record state must remain stable. If a model change requires a primary action to change, this is a breaking change requiring explicit review.

**I3 — State labels**  
Business state labels (pending, approved, rejected, in-progress, and equivalent) must remain stable. Regeneration must not introduce synonyms or alternates.

**I4 — Pattern-family stability**  
A screen must not change pattern family silently across regeneration. Pattern-family changes require explicit versioned migration. See §3.4.

**I5 — Personalization preservation**  
User-saved view preferences, saved filters, and named views must be preserved where valid across regeneration, and migrated or explicitly invalidated where not. See §3.5.

**I6 — Accessibility**  
Accessibility properties — focus management, aria labels, keyboard navigation — must not be degraded between regenerations.

**I7 — Action budget compliance**  
No regenerated screen may exceed the action budget defined in section 6. Automated audit of action counts is a required part of the generation evaluation pipeline.

**I8 — Cross-position semantic consistency**  
Business action labels and semantics must remain consistent across all position apps between regenerations. A change that alters the label or meaning of a shared business action in one position app without aligning all others is a breaking change requiring coordinated review. See P8.

**I9 — Provisional-state honesty**  
Regeneration must not convert unknown, provisional, or not-yet-modeled business state into false precision, false completeness, or misleading emptiness. See §2.8.

*Note: Telemetry event-name and hook-key stability is an instrumentation-layer invariant, not a doctrine-level invariant. It belongs to the evaluation and instrumentation specification.*

---

## 9. Fiori / Carbon / Fluent reference delta

Foundry takes deliberate positions relative to its three primary enterprise UI references. Each is used for a different purpose and carries a different delta. This matrix is the shared language for evaluating all subsequent pattern and component specifications.

- **SAP Fiori**: reference for ERP semantics, role-based thinking, list/detail/task patterns, and stateful business workflows.
- **IBM Carbon**: reference for information architecture discipline, structured layout, and content hierarchy.
- **Microsoft Fluent**: reference for approachable interaction patterns, accessibility defaults, and familiarity for business workers.

Foundry borrows rigor from all three without inheriting big-bang rollout assumptions. Progressive, pain-point-first adoption is a design premise, not a migration choice.

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
| Adoption model | Big-bang module deployment | Admin-governed rollout | Suite-wide deployment | Pain-point-first; progressive semantic slices |

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
| Is search immediately visible where required? | P3, §7.1 |
| Are all error messages in plain language? | P7, §7.3 |
| Is progressive disclosure applied? | §1.4, D3 |
| Is the empty state communicative with an action path? | §1.7, §7.1 |
| Are approvals first-class? | §2.3, §7.4 |
| Is density compliant with default or compact mode? | section 5 |
| Does personalization stay within bounded surfaces? | section 3 |
| Are user personalization, approved variants, and pattern migration clearly distinguished? | §3.4 |
| Does this survive the regeneration invariants? | section 8 |
| Is incomplete or provisional business state made explicit without false precision? | §2.8, I9 |
| Does the generated UI resolve only to approved patterns and approved variants? | P13 |
| Is the Foundry delta from Fiori maintained? | §9.2 |
| Is the Carbon information architecture contribution applied correctly? | §9.3 |
| Is the Fluent approachability contribution applied correctly? | §9.4 |
| Is the AI action explainable, reversible, and distinguishable? | P11 |
| Does the user retain final control? | P12 |
| Are semantic labels consistent within and across all position apps? | P8, I8 |
| If a personalization preference is invalidated, is conflict resolution applied? | §3.5 |
| Is each exception assigned a severity tier? | §7.5 |
| Is Critical exception behavior non-dismissible and audit-logged? | §7.5 |
| If a principle is violated, is it declared as an approved variant? | Exception handling, §4 |

---

*This document is version 0, patch round 1. It supersedes the first working draft. All subsequent work products should be evaluated against this revised version.*
