# Foundry2 Lab
## Position-Centric Regenerative Software for SMBs

**Status:** Working draft, high-level architecture milestone  
**Audience:** Founders, product, architecture, engineering, consulting  
**Purpose:** Capture the current thesis behind Foundry2, the motivation for the product, the governing principles, and the high-level architecture direction.

---

## 1. Executive summary

Foundry2 is an attempt to build a new kind of ERP for SMBs.

Instead of shipping one large, generic ERP application and asking the business to adapt to it, Foundry2 proposes a different unit of software: the **position app**. A position app is a role-specific business application tailored to the work, events, decisions, and responsibilities of one position in the company.

The economic premise is that this only becomes viable if software creation and change are heavily automated. Consultants and users should not be manually writing detailed specifications or engineering every change by hand. Instead, Foundry2 aims to use agents to capture business intent, build structured models, perform impact analysis, generate software, test it, deploy it, and continuously evolve it.

The current architectural view has three parts:

1. **Semantic architecture** defines the business meaning that software should reflect.
2. **Regenerative architecture** defines what must remain durable and trusted while software is continuously regenerated.
3. **Compiler architecture** defines how structured business intent becomes running software.

The thesis of Foundry2 is not merely that AI can generate code. The thesis is that SMB software can be built as a **position-centric, constraint-bound, regenerative software system** that starts from local business pain points, embraces incompleteness, evolves continuously, and delivers **personalized business apps within a stable and bounded UI grammar**.

---

## 2. The problem Foundry2 is trying to solve

Traditional ERP has several structural problems for SMBs:

- It is usually **module-centric** rather than **position-centric**.
- It is expensive to implement and expensive to change.
- It often assumes the business should conform to the software.
- It is optimized for large, upfront analysis and long implementation cycles.
- It produces software that becomes operationally rigid once deployed.

This is especially misaligned with SMB reality. SMB businesses are rarely static, fully documented, or architecturally clean. They change quickly, operate under uncertainty, and are often managed through constant firefighting by the CEO and a small set of trusted operators.

The core Foundry2 belief is that SMB software should:

- start with a real pain point,
- start from any business unit or position,
- work with incomplete knowledge,
- deliver useful software early,
- and evolve continuously after first release.

---

## 3. The Foundry2 product thesis

### 3.1 What Foundry2 is

Foundry2 is a system for generating and evolving **personalized business applications** for SMB positions.

Each position app is a role-specific projection over the business:

- what work reaches that person,
- what decisions they make,
- what entities they monitor,
- what actions they initiate,
- what events matter to them,
- what approvals they give or receive,
- and what AI assistance they can invoke.

This personalization is not intended to be free-form UI improvisation. It should be delivered through **bounded, typed, and regenerable variation** inside a stable product grammar.

### 3.2 What Foundry2 is not

Foundry2 is not:

- a generic low-code builder,
- a one-time code generator,
- one backend per position,
- a free-form agent that writes arbitrary software however it wants,
- a free-form UI system that allows arbitrary per-user structural forks,
- or a traditional ERP rollout that requires complete enterprise analysis before implementation begins.

---

## 4. Foundry2 personality and operating stance

The product personality matters because it drives the architecture.

Foundry2 assumes:

- **start anywhere**: a department, a team, a workflow, or even one position,
- **pain-point first**: automation begins where business urgency is highest,
- **incompleteness is acceptable**: the model can start partial and become richer later,
- **co-creation is expected**: the CEO and operators can shape the system through conversation and usage,
- **change is continuous**: the first release is the beginning, not the end,
- **local before global**: the system grows slice by slice, not through a big-bang rollout,
- **agents do most of the work**: human intervention is reserved for true decisions, approvals, and exception handling,
- **personalization matters**: the position app should feel like the user's own work surface,
- **bounded variation beats free-form customization**: users should experience ownership without sacrificing trust, coherence, or regenerability.

This means Foundry2 cannot rely on a semantic process that assumes exhaustive enterprise discovery up front. Its semantic approach must be incremental, composable, and tolerant of ambiguity.

---

## 5. Core concepts

### 5.1 Position app
A front-end application specialized for one position in the company. It is the user's work surface.

### 5.2 Bounded context
A domain boundary that owns business meaning, rules, and mutation authority. It is a backend/domain boundary, not a UX boundary.

### 5.3 Progressive semantic slice
The minimum business meaning required to safely generate the next useful software slice.

### 5.4 Regenerative grain
The smallest unit that can be safely deleted, regenerated, tested, and redeployed without destabilizing the whole system.

### 5.5 Conservation layer
A layer that should change more slowly because users build trust, habits, and operational confidence on top of it. In Foundry2, the user-facing app patterns, navigation logic, and action semantics should behave this way.

### 5.6 Provenance
The causal record of why and how a change happened: conversations, assumptions, approvals, generated plans, evaluations, and resulting artifacts.

### 5.7 UI pattern
A reusable interaction structure that defines how a class of business problems is presented and handled inside a position app. Patterns can exist at page, flow, and component levels. They include structure, behavior, state expectations, and usage rules.

### 5.8 Pattern variant
An explicitly modeled variation of a UI pattern. Variants may affect presentation, emphasis, density, layout regions, or limited structural choices, but only within approved bounds.

### 5.9 Bounded personalization
The doctrine that position apps should feel personalized while remaining stable, generatable, and supportable. Personalization is achieved through approved pattern variants, view preferences, themes, defaults, and scoped configuration rather than arbitrary UI invention.

---

## 6. Three-pronged architecture

| Architecture | Purpose | Main question |
|---|---|---|
| Semantic architecture | Define business meaning | What does the business actually mean? |
| Regenerative architecture | Govern safe continuous change | What must survive regeneration and how is trust established? |
| Compiler architecture | Turn meaning into software | How does structured intent become running code and systems? |

These are complementary, not competing.

- The **semantic architecture** prevents meaningless automation.
- The **regenerative architecture** prevents unsafe automation.
- The **compiler architecture** prevents ad hoc automation.

The UI/UX layer cuts across all three:

- in **semantic architecture**, it appears as the **position projection** and the mapping of business slices into UI patterns,
- in **regenerative architecture**, it appears as a **conservation layer** with stable UI patterns and bounded personalization,
- in **compiler architecture**, it appears as a **deterministic generation target** expressed through patterns, variants, and UI-facing schemas.

Its center of gravity is regenerative architecture, but it depends on the other two.

---

## 7. Semantic architecture for Foundry2

### 7.1 The key correction

Foundry2 should not use a traditional, enterprise-wide, big-bang semantic process.

It should use **progressive business semantics**.

That means:

- model only the business slice needed for the next useful release,
- allow missing information and placeholders,
- version and refine meaning over time,
- and let the enterprise model grow by accretion.

### 7.2 Semantic architecture should be slice-based

The right question is not:

> Do we understand the entire enterprise?

The right question is:

> Do we understand enough of this business slice to generate a useful and safe app/service set?

### 7.3 Minimum semantic slice

A semantic slice for Foundry2 should usually contain the following artifacts.

| Artifact | Purpose |
|---|---|
| Problem frame | What pain point is being solved now? |
| Local org slice | Which org unit, positions, and nearby collaborators matter? |
| Work slice | What events, actions, decisions, handoffs, and exceptions define the work? |
| Vocabulary and rules | What terms, facts, and business rules must be understood correctly? |
| Context boundary | Which business boundary owns this capability and its mutation authority? |
| Position projection | How does this work appear inside one or more position apps, including mapping to UI patterns and personalization surfaces? |
| Success criteria | What business outcomes or user outcomes would count as improvement? |

### 7.4 How enterprise frameworks are used

Foundry2 should borrow rigor from enterprise frameworks without inheriting their rollout style.

- **DDD** is useful for business language and bounded contexts.
- **EventStorming** is useful for discovering events, commands, and handoffs.
- **BPMN / CMMN / DMN** are useful where process, case, or decision logic needs formalization.
- **SBVR** is useful as inspiration for vocabulary and rules.
- **Business architecture ideas** such as capabilities, value streams, and roles are useful when the slice needs them.
- **Fiori-style role-based thinking** is useful for shaping the position app experience and initial UI pattern grammar.

These are inputs to Foundry2 semantics, not prerequisites for a traditional ERP program.

### 7.5 Semantic architecture compatibility verdict

Foundry2 personality and semantic architecture are **compatible** only if the semantic architecture is:

- partial,
- local,
- event-centered,
- position-centered,
- regenerable,
- personalization-aware,
- and progressive.

They are incompatible if semantic architecture is interpreted as enterprise-wide upfront completeness.

---

## 8. Regenerative architecture for Foundry2

The regenerative architecture is the operating doctrine for continuous, agentic change.

Its role is to answer questions that the semantic model and compiler do not answer by themselves:

- What is durable and what is disposable?
- Where does trust come from in an unmanned system?
- How do we choose the right regeneration boundary?
- What should change quickly and what should change slowly?
- What records replace human code review as the causal history of change?

### 8.1 Durable assets

For Foundry2, the durable assets are:

- business meaning,
- context boundaries,
- contracts,
- evaluations,
- policies,
- provenance,
- stable UI patterns,
- approved pattern variants,
- and personalization policies and defaults.

### 8.2 Disposable assets

The disposable assets are primarily:

- generated code,
- generated configuration,
- generated tests that are derivable from durable specs,
- and deployable artifacts.

### 8.3 Key regenerative principles

- **compile to architecture** rather than to loose files,
- **evaluations are first-class assets**,
- **provenance replaces code diffs as the main causal record**,
- **regeneration must happen at a safe grain**,
- **different layers should move at different speeds**,
- **UI should act as a conservation layer**,
- **personalization must be bounded, typed, and versioned**,
- **agents are trusted through constraints, policy, and evaluation, not through hope**.

### 8.4 Why this matters for Foundry2

Without this doctrine, Foundry2 could still generate software, but it would struggle to govern ongoing change safely.

Without bounded personalization, Foundry2 would either feel too rigid for its product promise or become too fragmented to regenerate safely.

---

## 9. Compiler architecture for Foundry2 v1

### 9.1 Guiding principle

The compiler is not the agent.

- The **agent** gathers, proposes, restructures, and patches business intent.
- The **compiler** deterministically turns valid structured intent into intermediate representations and software artifacts.

This distinction is central to predictability.

### 9.2 Target stack

Current baseline decisions:

- **Frontend target:** React + TypeScript
- **Backend target:** Python + FastAPI
- **Contract layer:** Pydantic + JSON Schema + OpenAPI
- **Policy/control:** OPA
- **Orchestration and durable execution:** Temporal
- **Client generation:** OpenAPI Generator for TypeScript clients
- **Generation core:** custom deterministic compiler and templates

### 9.3 Why this stack fits Foundry2

- React + TypeScript suits web-native, component-based position apps.
- FastAPI suits AI-native backend systems and works naturally with typed contracts.
- Pydantic, JSON Schema, and OpenAPI provide a machine-validatable representation layer.
- OPA allows policies and change permissions to be codified and enforced.
- Temporal supports durable, long-running, partially automated workflows.
- Deterministic generation keeps the lower layers repeatable and diffable.

### 9.4 High-level compiler flow

```text
conversation / discovery / feedback
    -> progressive semantic slice
    -> structural validation
    -> policy validation
    -> normalization
    -> UI pattern and variant resolution
    -> intermediate representation
    -> generated React apps
    -> generated FastAPI services
    -> generated contracts, tests, and deployment artifacts
    -> evaluations
    -> deploy / reject / request approval
```

### 9.5 What the compiler should own

The compiler should own:

- normalization of business slices,
- derivation of architecture and contracts,
- mapping of position projections into UI patterns and approved variants,
- generation of code and configuration,
- generation of evaluation scaffolding,
- and artifact packaging.

It should not own open-ended business interpretation without semantic or policy constraints.

---

## 10. UX doctrine

### 10.1 Position-centric UX

The UI unit in Foundry2 is the **position app**, not the horizontal enterprise module.

Each app should reflect the daily operating surface of one position.

### 10.2 Consumer-grade usability, enterprise-grade rigor

Foundry2 targets SMB workers who are highly trained by consumer software to expect:

- low friction,
- obvious next steps,
- minimal learning curve,
- fast feedback,
- and forgiving interactions.

The goal is not to make enterprise work trivial. The goal is to make complex business work feel as approachable as the best consumer software while preserving enterprise needs such as traceability, approvals, and correctness.

### 10.3 UI patterns as the core abstraction

Foundry2 should define a constrained and reusable set of **UI patterns**.

A **UI pattern** is a reusable interaction structure that defines how a class of business problems is presented and handled in a position app.

Patterns are not just layouts. They include:

- structure,
- behavior,
- state,
- and rules for when they should or should not be used.

The system should generate applications **by selecting and composing patterns**, not by inventing arbitrary UI for each case.

### 10.4 Pattern hierarchy

UI patterns exist at multiple levels.

#### 1. Page patterns
High-level structures for complete screens.

Examples:
- Collection View
- Record Page
- Overview / Monitor
- Search / Results
- Exception Resolution View
- Approval / Review View

#### 2. Flow patterns
Structures for multi-step or task-oriented interactions.

Examples:
- Create/Edit Flow
- Approval Flow
- Exception Resolution Flow
- Assisted Setup Flow

#### 3. Component patterns
Reusable substructures within pages and flows.

Examples:
- Activity Timeline
- Related Records Section
- Bulk Action Bar
- Status Banner
- Task Context Panel

This layered approach allows Foundry2 to balance consistency and flexibility while keeping the system generatable.

### 10.5 Pattern selection and agent alignment

The model should not directly specify UI implementation details. Instead, it should reference **UI patterns**.

The agent is responsible for mapping:

- business intent,
- position projection,
- and user goals

into appropriate UI patterns.

Each pattern must therefore define:

- when to use it,
- when not to use it,
- required inputs from the model,
- allowed actions,
- expected states,
- and approved variant surfaces.

This makes patterns usable both by humans and by the agent during generation.

### 10.6 Bounded personalization inside the conservation layer

Foundry2 should not choose between stability and personalization. It should deliver **stable interaction grammar with bounded personalization**.

The invariants should remain stable:

- navigation logic,
- action semantics,
- state semantics,
- validation behavior,
- approval behavior,
- and the core pattern grammar.

Variation should happen only within approved bounds.

### 10.7 Personalization surfaces

The main personalization surfaces should be:

- **visual personalization**: theme, density, emphasis, spacing feel, and similar token-driven adjustments,
- **view-state personalization**: saved filters, column choices, sort order, pinned views, default sections, and preferred entry points,
- **slot-level and emphasis variants**: approved differences in presentation or emphasis within a pattern family,
- **limited structural variants**: only where explicitly modeled and supportable.

Foundry2 should prefer more freedom in visual and view-state personalization, some freedom in emphasis and slot variants, and much tighter control over structural variants.

### 10.8 Fiori compatibility and delta

Foundry2 should treat Fiori-style role-based UX as a valid source of inspiration.

The compatible ideas are:

- role/position focus,
- stable UI patterns,
- coherent layouts,
- consistent workflows,
- and a repeatable UX grammar.

The deliberate Foundry delta is:

- lower default density,
- fewer visible actions at once,
- stronger primary-action emphasis,
- more progressive disclosure,
- bounded personalization as a first-class capability,
- search-first and action-first flows where appropriate,
- clearer empty states and contextual help,
- and softer, less intimidating interaction patterns.

### 10.9 Practical implication

Foundry2 should generate **within a constrained set of UI patterns and approved variants**, not produce arbitrary UI inventions for every release.

---

## 11. End-to-end operating loop

```text
CEO / operator pain point
    -> consultant + agent discovery
    -> progressive semantic slice
    -> compiler generation
    -> position apps + backend services + evaluations
    -> deployment
    -> user feedback / telemetry / requests
    -> impact analysis + estimate + approval
    -> regenerated slice
```

This is not a one-time implementation model. It is a living operating loop.

---

## 12. Foundry2 invariants

These are the current non-negotiable architectural beliefs.

1. **Business meaning must be explicit enough to compile.**
2. **Code is not the primary asset; the system and its meaning are.**
3. **Generation must be constraint-bound, not free-form.**
4. **The agent should primarily shape models, not handcraft arbitrary code.**
5. **Bounded contexts own business meaning and mutation authority.**
6. **Position apps own business experience, not backend ownership.**
7. **Change should be local by default and global only when necessary.**
8. **Evaluations are required for trust in unmanned regeneration.**
9. **Provenance is required for governance, debugging, and learning.**
10. **The UX should be stable enough to build user trust and habit.**
11. **That stable UX should still support bounded personalization so the app feels owned by the user.**
12. **The semantic model must tolerate incompleteness and evolve over time.**
13. **The system must support continuous post-release change as a first-class mode.**

---

## 13. Requirements

### 13.1 Business requirements

- Make SMB automation economically viable.
- Reduce dependence on large upfront ERP programs.
- Support pain-point-first adoption.
- Let the CEO and operators co-create software progressively.
- Shorten the cycle time from need to software change.

### 13.2 Product requirements

- Generate role-specific business apps.
- Support continuous change requests after release.
- Provide AI-native app behaviors where useful.
- Preserve stable, low-friction UI patterns.
- Keep the learning curve low for SMB workers.
- Provide bounded personalization so each position app can feel owned by the user without fragmenting the product grammar.

### 13.3 Architecture requirements

- Accept incomplete and evolving business models.
- Support deterministic generation below the semantic layer.
- Allow policy-gated and approval-gated change workflows.
- Track provenance for every meaningful change.
- Support local regeneration of slices rather than full-system rewrites.
- Enforce evaluations before trust is increased.
- Model UI patterns and approved variants explicitly enough for deterministic compilation and safe regeneration.

---

## 14. Non-goals

Foundry2 is not trying to:

- complete a global enterprise model before first value delivery,
- mirror a classic ERP implementation methodology,
- generate arbitrary software without reusable constraints,
- assign one backend or one database per position app,
- permit free-form UI customization that bypasses the pattern grammar,
- create one-off structural UI forks per individual user by default,
- or maximize visual novelty in the UI.

---

## 15. Current decisions captured in this draft

### 15.1 Product direction

- Foundry2 is position-centric.
- The system starts from local pain points.
- Continuous evolution after release is core, not optional.
- Personalized business apps are part of the value proposition, but within bounded variation.

### 15.2 Architecture direction

- Three-pronged architecture is the right current framing.
- Semantic architecture must be progressive, not exhaustive.
- Regenerative architecture should be treated as governance doctrine.
- Compiler architecture should remain deterministic below the model layer.
- UI/UX is a cross-cutting layer with its center of gravity in regenerative architecture.

### 15.3 Technical direction

- React + TypeScript is the frontend target.
- FastAPI + Python is the backend target.
- Pydantic + JSON Schema + OpenAPI are the v1 machine-contract baseline.
- OPA and Temporal remain part of the v1 control plane.

### 15.4 UX direction

- Foundry2 should take inspiration from Fiori-like role-based UX.
- UX should aim for consumer-grade usability and enterprise-grade rigor.
- UI should be treated as a conservation layer.
- UI patterns are the core generation abstraction.
- Personalization should be first-class, but bounded and explicitly modeled.

---

## 16. Risks and tensions

### 16.1 Over-modeling risk
If semantic rigor is interpreted in a big-enterprise way, Foundry2 will become too slow and too top-heavy.

### 16.2 Under-modeling risk
If semantic rigor is ignored, the compiler will generate software from unstable or ambiguous meaning.

### 16.3 Free-form generation risk
If agents are allowed to write arbitrary code without strong constraints, the system will become inconsistent and hard to evolve.

### 16.4 UI churn risk
If every regeneration changes the user-visible surface significantly, user trust and usability will collapse.

### 16.5 Variant-sprawl risk
If patterns support too many uncontrolled variants, the test matrix, support burden, and upgrade burden will grow too quickly.

### 16.6 Conservation-versus-personalization risk
If personalization is too weak, the product will feel generic and impersonal. If it is too strong, the system will lose coherence and regenerability.

### 16.7 Grain-selection risk
If the regenerative grain is too large, change becomes expensive. If it is too small, coherence is lost.

---

## 17. Open questions for the next milestone

1. What is the exact Foundry2 canonical schema for a progressive semantic slice?
2. What is the right intermediate representation between semantic slice and generated artifacts?
3. What is the right regenerative grain for Foundry2 in practice?
4. What are the required evaluation categories for trust escalation?
5. What are the minimal stable UI patterns, approved variants, and primitives for position apps?
6. Which personalization surfaces should operate at user, role, tenant, or global scope?
7. What provenance model should be captured for every change request and regeneration?
8. How should impact analysis and costing be modeled?
9. How should AI-native interactions appear inside stable, role-based, personalization-aware UX patterns?

---

## 18. Recommended next steps

1. Define the **Foundry2 progressive semantic slice schema**.
2. Define the **Foundry2 UI pattern grammar and bounded personalization model** for position apps.
3. Define the **evaluation taxonomy** for generated slices.
4. Define the **provenance model** for conversational and agentic changes.
5. Define the **compiler IR** and generation boundaries.
6. Select one pilot department and run the full loop on a narrow slice.

---

## 19. Closing statement

Foundry2 is best understood not as an AI coding product, and not as a classical ERP product, but as a **position-centric regenerative software system for SMBs**.

Its distinctive move is to combine:

- progressive business semantics,
- regenerative governance,
- and deterministic compilation,

so that software can start from one local pain point, become useful quickly, continue to evolve safely over time, and produce **stable yet personalized business apps** for the people who use them every day.
