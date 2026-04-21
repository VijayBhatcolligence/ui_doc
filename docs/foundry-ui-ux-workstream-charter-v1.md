# Foundry UI/UX Workstream Charter v1

## 1. Purpose

This workstream exists to define the **stable UI grammar and bounded personalization model** that Foundry's compiler will target for position apps.

The goal is **not** to design arbitrary screens, and **not** to build a generic frontend component library. The goal is to specify the UI/UX layer that can be:

- generated deterministically from a model,
- implemented in React,
- stable enough to act as a conservation layer,
- rigorous enough for enterprise work,
- simple enough for SMB users trained by consumer software,
- and personal enough that a position app feels like the user's own work surface.

This workstream is intended to progress **independently** of still-evolving layers such as the full business ontology, final policy model, provenance model, and other compiler details. It should expose clean seams for those layers to integrate later.

---

## 2. Context

Foundry is building position-centric regenerative software for SMBs. The frontend target is React + TypeScript, and the product is expected to generate role-specific business apps from structured models via a compiler path that eventually reaches React code.

Within this broader architecture, the UI is a **conservation layer**. It should not churn heavily every time deeper semantic, policy, or service layers evolve. Instead, the visible user experience should be constrained to a stable and repeatable grammar.

At the same time, Foundry's product stance is not generic enterprise software. Position apps should feel owned by the user. That requires personalization, but in a form that remains generatable, supportable, and regenerable.

The relevant interpretation for this workstream is:

**Model -> UI-facing projection -> pattern selection -> variant selection -> semantic component composition -> React implementation**

This means the workstream should focus on the parts of the UI stack that can be specified and stabilized early:

- UX doctrine
- architectural placement of UI across the three-pronged architecture
- position-oriented work surfaces
- UI patterns
- bounded personalization and variability model
- semantic component library
- primitive mapping
- token strategy

---

## 3. Architectural placement inside Foundry

This workstream spans all three parts of Foundry's architecture, but its center of gravity is in the regenerative layer.

### 3.1 Semantic architecture contribution

The UI workstream contributes the **position-facing projection** of business meaning:

- how work appears for a role,
- how actions and decisions surface,
- how events and states are made visible,
- and how business slices map into UI patterns.

### 3.2 Regenerative architecture contribution

The UI workstream defines part of the **conservation layer**:

- stable UI patterns,
- stable navigation logic,
- stable action semantics,
- approved pattern variants,
- and bounded personalization rules.

This is the primary architectural home of the workstream.

### 3.3 Compiler architecture contribution

The UI workstream defines a **deterministic generation target**:

- a UI-facing schema the model can target,
- a pattern vocabulary,
- a variant vocabulary,
- semantic component composition rules,
- and a structure that can be compiled into React.

---

## 4. Why this workstream matters

Without a defined UI grammar, Foundry risks one of three failure modes:

1. **Ad hoc generation**: the compiler emits arbitrary screens that are inconsistent and hard to evolve.
2. **Premature low-level implementation**: the team starts from primitives or CSS and loses the business semantics the generator must target.
3. **No meaningful personalization**: the product feels like generic enterprise software rather than a position app that feels owned by the user.

A deliberate UI/UX workstream avoids all three problems.

It gives Foundry:

- a stable target for deterministic generation,
- a product-level answer to "enterprise rigor + consumer ease",
- a bounded answer to personalization,
- a clean seam for other evolving layers,
- and a concrete project that can be assigned to a developer and taken to completion.

---

## 5. Core objective

Define the first version of Foundry's **UI meta-model, pattern system, and bounded personalization model** for position apps.

This includes:

1. the UX doctrine,
2. the position-facing UI projection schema,
3. the UI pattern taxonomy and specifications,
4. the pattern-variant and personalization model,
5. the semantic component library specifications,
6. the primitive-component mapping for React implementation,
7. and a provisional token/style direction that reflects the intended product personality.

---

## 6. Product stance for this workstream

The workstream should be guided by the following product stance:

- **Position-centric**: the unit of experience is the position app, not the horizontal module.
- **Consumer-grade ease**: low friction, obvious next steps, progressive disclosure, forgiving interactions.
- **Enterprise-grade rigor**: traceability, correctness, approvals, statefulness, recoverability.
- **Bounded personalization**: the app should feel personal without becoming free-form.
- **Deterministic composition**: the generator should target stable semantics, not raw low-level widgets.
- **Stable visible patterns**: the UI should absorb less churn than deeper layers.
- **Progressive semantics**: the UI contract must work even while the broader semantic architecture is incomplete.

---

## 7. Starting references and deliberate delta

### 7.1 Reference systems

The workstream should use multiple reference systems, but for different purposes:

- **SAP Fiori**: reference for ERP semantics, role-based thinking, list/detail/task patterns, stateful business workflows.
- **IBM Carbon**: reference for enterprise structure and information architecture.
- **Microsoft Fluent**: reference for approachable interaction patterns and familiarity.
- **Consumer apps (for example WhatsApp/YouTube level simplicity)**: reference for ease, clarity, low friction, strong action guidance, and ownership feel.

### 7.2 The Foundry delta

Foundry should **not** lift Fiori as-is.

It should preserve ERP rigor while changing the experience in the following directions:

- lower default density,
- fewer visible actions at once,
- stronger primary action emphasis,
- more progressive disclosure,
- search-first and action-first flows,
- clearer empty states and contextual help,
- guided create/edit flows instead of exposing every field upfront,
- softer, less intimidating interaction patterns,
- bounded personalization as a first-class capability.

This delta should be made explicit in the specifications.

---

## 8. Scope

### In scope

- UX doctrine for position apps
- architectural placement of UI across semantic, regenerative, and compiler architecture
- position-facing projection schema for the UI layer
- UI pattern taxonomy
- UI pattern specifications
- pattern variants and personalization surfaces
- user, role, and tenant personalization boundaries
- semantic component library specifications
- composition rules and allowed child structures
- state and action models at the UI layer
- React implementation strategy for semantic components
- primitive-component mapping
- provisional token strategy
- prototype of 2-3 golden workflows

### Out of scope for now

- full enterprise-wide business ontology
- final bounded-context decomposition
- final backend service contracts
- full provenance model
- full policy/control integration details
- final AI interaction patterns
- complete visual branding system
- free-form per-user structural UI customization
- production-hardening of every component

---

## 9. The key architectural decision

The workstream should proceed **top-down**, not bottom-up.

The order should be:

1. UX doctrine
2. position-facing UI projection schema
3. UI patterns and invariants
4. pattern variants and personalization model
5. semantic component library
6. prototype workflows
7. primitive mapping
8. tokens/style layer
9. implementation plan

### Why this order

The generator should eventually target **business UI grammar**, not primitive UI atoms.

Therefore the first-class artifacts for this workstream are:

- UI patterns,
- approved variants,
- personalization rules,
- semantic components,
- composition rules,
- and UI-facing schemas.

Not button variants, raw tables, CSS scales, or generic primitives.

---

## 10. Work products

### 10.1 UX doctrine v0

A concise document that defines:

- what "consumer-grade ease" means in operational terms,
- what "enterprise-grade rigor" means in UX terms,
- what "bounded personalization" means in product terms,
- the non-negotiable interaction principles,
- default density rules,
- action-budget rules,
- guidance for search, filtering, forms, approvals, and exceptions,
- and the invariants that must survive regeneration.

### 10.2 Position Projection Schema v0

A UI-facing schema describing what a position app needs from the model.

Expected areas:

- tasks
- decisions
- monitored entities
- alerts/events
- actions
- approvals
- related collaborators/hand-offs
- state summaries
- permissions hooks
- AI-assist hooks
- default view hints
- personalization hooks and scopes

### 10.3 UI Pattern Library v0

The first set of page and flow types that the model can refer to.

Candidate page patterns:

- Collection View
- Record Page
- Overview / Monitor
- Search / Results
- Exception Resolution View
- Approval / Review View

Candidate flow patterns:

- Create/Edit Flow
- Approval Flow
- Exception Resolution Flow
- Assisted Setup Flow

Each pattern should specify:

- purpose,
- when to use,
- when not to use,
- required inputs,
- optional inputs,
- regions/slots,
- navigation rules,
- state behavior,
- loading/empty/error behavior,
- responsive behavior,
- allowed semantic children,
- invariants,
- approved variant surfaces.

### 10.4 Pattern Variability and Personalization Model v0

A model for how variation is allowed without breaking the conservation layer.

It should define:

- scopes of variation: global, tenant, role, user,
- personalization surfaces,
- precedence rules and fallbacks,
- which variations are token-driven,
- which are saved-view driven,
- which are approved structural variants,
- migration and compatibility expectations,
- and what remains non-negotiable.

### 10.5 Semantic Component Library v0

A library of business-aware UI building blocks that live under the patterns.

Candidate components:

- RecordHeader
- KeyFactsStrip
- RelatedRecordsSection
- SmartFormSection
- StatusBanner
- BulkActionBar
- FilterRail
- ActionPanel
- ActivityTimeline
- ApprovalPanel
- AttachmentPanel
- EmptyState
- ValidationSummary
- TaskContextPanel
- Comment/ConversationPanel

Each component spec should include:

- purpose,
- inputs,
- slots,
- states,
- actions,
- composition rules,
- accessibility considerations,
- telemetry hooks,
- and allowed personalization surfaces.

### 10.6 Golden Workflow Prototypes

Prototype 2-3 high-value workflows to validate the grammar.

Suggested workflow types:

- one create/edit flow,
- one approve/review flow,
- one monitor-and-resolve-exception flow.

### 10.7 Primitive Mapping and Token Strategy v0

After semantic validation, map the system into:

- React primitives/wrappers,
- Radix usage,
- custom structural primitives,
- spacing/typography/color roles,
- density modes,
- motion/elevation principles,
- and theming/personalization hooks.

---

## 11. Milestones and exit criteria

### Milestone 0 — Charter approved

**Output**: this charter reviewed and accepted.

**Exit criteria**:
- scope is agreed,
- the workstream owner is assigned,
- review cadence is agreed,
- immediate next deliverables are locked.

### Milestone 1 — UX doctrine and reference delta

**Output**: UX doctrine v0 + Fiori/Carbon/Fluent/reference delta matrix.

**Exit criteria**:
- explicit rules for ease vs rigor exist,
- the meaning of bounded personalization is documented,
- Fiori-derived patterns to keep/change/discard are documented,
- there is a shared language for evaluating later specs.

### Milestone 2 — Position Projection Schema v0

**Output**: UI-facing schema for what the model provides to the UI layer.

**Exit criteria**:
- schema covers tasks, entities, actions, events, approvals, and state,
- personalization hooks and scopes are identified,
- schema is sufficiently narrow to remain stable,
- open seams for policy/provenance/AI are identified without overcommitting.

### Milestone 3 — UI Pattern Library v0

**Output**: specifications for the first 6-10 UI patterns.

**Exit criteria**:
- patterns cover the main work surfaces of position apps,
- each pattern has invariants and composition rules,
- each pattern can be expressed as a target in the future IR,
- approved variant surfaces are explicit.

### Milestone 4 — Pattern Variability and Personalization Model v0

**Output**: rules for allowed variation across global, tenant, role, and user scopes.

**Exit criteria**:
- invariants versus variable surfaces are explicit,
- precedence and fallback rules are defined,
- structural variants are tightly bounded,
- the model is supportable and testable.

### Milestone 5 — Semantic Component Library v0

**Output**: component specs under the patterns.

**Exit criteria**:
- semantic components are defined with inputs/states/slots,
- overlap and redundancy are reduced,
- components are business-aware rather than generic UI widgets,
- component-level personalization hooks stay within pattern constraints.

### Milestone 6 — Golden Workflow Validation

**Output**: 2-3 end-to-end prototypes.

**Exit criteria**:
- representative workflows can be expressed with the patterns and components,
- usability issues are surfaced early,
- personalization assumptions are exercised,
- generator implications are visible.

### Milestone 7 — Primitive Mapping and Token Strategy

**Output**: implementation substrate decision and initial token system.

**Exit criteria**:
- semantic components can be mapped to React implementation patterns,
- required primitives are known,
- token/style direction is compatible with the UX doctrine,
- personalization hooks are implementable without breaking invariants.

### Milestone 8 — Handoff to compiler integration planning

**Output**: a clear input contract for the compiler/UI IR work.

**Exit criteria**:
- the semantic UI system can be named and targeted by a future IR,
- pattern and variant vocabularies are explicit,
- integration questions are logged,
- remaining dependencies on other Foundry layers are explicit.

---

## 12. Additional inputs required

The workstream can begin now, but it will benefit from these inputs as they become available:

1. Top 10 business workflows to support first
2. Roles/positions most likely to be first pilots
3. Action/state matrix for those workflows
4. List sizes, filter needs, bulk-action needs, and data density expectations
5. Device assumptions (desktop first, mobile assisted, etc.)
6. Compliance/audit constraints that affect visible UX
7. Brand/tone direction beyond "consumer-grade ease"
8. Personalization expectations by role, tenant, and user
9. Success metrics for usability, adoption, and task completion

These are helpful inputs, not blockers for starting the workstream.

---

## 13. Risks

### Risk 1 — Starting too low in the stack
If the team starts with raw components, tokens, or CSS, the generator contract will remain unclear.

### Risk 2 — Copying Fiori too literally
If Foundry inherits Fiori's density and enterprise gravity, the SMB usability goal will be weakened.

### Risk 3 — Over-coupling to unfinished backend layers
If the UI waits for the full semantic or policy architecture, progress will stall unnecessarily.

### Risk 4 — Excessive abstraction
If the workstream tries to solve every future variation upfront, the semantic library will become vague and unusable.

### Risk 5 — No validation against real workflows
If the workstream stays purely conceptual, weak assumptions will survive too long.

### Risk 6 — Variant sprawl
If too many variants are allowed without discipline, the test matrix, support burden, and upgrade burden will grow too quickly.

### Risk 7 — Personalization undermines the conservation layer
If personalization changes core interaction grammar too freely, trust and learnability will collapse.

### Risk 8 — Under-personalization
If the system is too rigid, Foundry will fail to deliver the "this is my app" feeling that is central to the product promise.

---

## 14. Recommended immediate next step

Create the first working document after this charter:

**Foundry UI Grammar v1**

It should contain:

1. UX doctrine
2. Reference delta matrix (especially against Fiori)
3. Position Projection Schema v0
4. UI Pattern Library v0
5. Pattern Variability and Personalization Model v0

Only after that should the workstream draft the semantic component library in detail.

---

## 15. Suggested near-term sequencing

### Week 1
- Finalize charter
- Gather and summarize reference patterns
- Draft UX doctrine
- Produce Fiori keep/change/discard matrix
- Draft personalization guardrails and invariants

### Week 2
- Draft Position Projection Schema v0
- Draft first UI pattern set
- Draft variability/personalization model
- Review with founder/product

### Week 3
- Draft semantic component library v0
- Validate against 2-3 workflows

### Week 4
- Refine based on feedback
- Draft primitive mapping and token direction
- Prepare compiler-facing handoff notes

---

## 16. Working mode

This should be treated as a standalone project with:

- one directly responsible owner,
- weekly review checkpoints,
- explicit decision logs,
- and milestone-based outputs rather than open-ended exploration.

The developer should be asked to produce specifications first and implementation second.

---

## 17. Definition of success

This workstream is successful when Foundry has:

- a documented UI grammar for position apps,
- a bounded personalization model that preserves the conservation layer,
- a clear UI-facing schema the model can target,
- a UI pattern library that captures the core work surfaces,
- a semantic component library that sits under those patterns,
- and enough validation to begin integration with the compiler and React implementation.
