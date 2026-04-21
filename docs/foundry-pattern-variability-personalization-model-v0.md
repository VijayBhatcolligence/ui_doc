# Foundry Pattern Variability and Personalization Model v0

**Status:** First working draft  
**Workstream:** UI/UX — Work Product 10.4  
**References:** Foundry UX Doctrine v0, Foundry Position Projection Schema v0, Foundry UI Pattern Library v0, Foundry UI/UX Workstream Charter v1, Foundry2 Position-Centric Regenerative Software for SMBs v1  
**Purpose:** Define the complete model for how variation is permitted within Foundry position apps — what can vary, at what scope, through what mechanism, under what governance, and what can never be changed.

This document is the authoritative source for the personalization and variability model. It expands the doctrine principles (UX Doctrine §3) into an operational specification. All pattern variant declarations (10.3), schema personalization hooks (10.2 §4.15), and runtime personalization behaviors must be consistent with this model.

---

## 1. Purpose and position in the system

This document sits between the UI Pattern Library (10.3) and the Semantic Component Library (10.5) in the workstream sequence. Its role is to govern how much flexibility is permitted inside the patterns that 10.3 defines, and how that flexibility is expressed, stored, applied, and protected.

```
UX Doctrine (10.1) ————— governing principles
Position Projection Schema (10.2) ————— PersonalizationHooks schema
UI Pattern Library (10.3) ————— approved variant surfaces per pattern
[This document — 10.4] ————— complete variability model and governance
Semantic Component Library (10.5) ————— component-level surfaces
Golden Workflow Prototypes (10.6) ————— validation against real workflows
```

Without this model, three failure modes persist:

1. Variants are added inconsistently across patterns without a governing taxonomy.
2. Structural variants are introduced informally and erode the conservation layer.
3. User preferences accumulate, become orphaned after regeneration, and silently degrade the app experience.

This document prevents all three.

---

## 2. Variation taxonomy

Foundry permits four and only four types of variation inside position apps. All variation must be classified under one of these types. Any unlisted form of variation is a compliance failure, not a design choice.

### 2.1 Four variation types

| Type | What it changes | Who sets it | Mechanism | Governance level |
|---|---|---|---|---|
| **Token-driven** | Visual appearance only: theme, density feel, color emphasis, spacing | Tenant / role / user | CSS custom properties via design token system | Low — no structural impact |
| **View-state** | Saved operational state: filters, sort order, column choices, pinned views, section expansion | User only | Runtime preference store keyed to position + view | Moderate — must survive regeneration; invalidated on schema change |
| **Emphasis variant** | Presentation and emphasis within a slot: which content is prominent, where emphasis falls, information depth | Role / user (as declared per variant in §4) | Approved variant id resolved at compile time or saved to preference store | Moderate — must be a listed approved variant; does not change structure |
| **Structural variant** | Layout regions, component substitution, compositional differences | Tenant / role only (never user) | Compiled into PositionProjection; declared in ViewSpec.patternVariant | High — requires approval, documentation, test coverage, and compiler support |

### 2.2 Type rules

**Token-driven rules**
- Token-driven variation never changes the number, order, or type of rendered elements.
- It changes only visual properties: color, spacing, typography scale within allowed range, density feel.
- Token-driven variation does not require an entry in the approved variant registry (§4).
- All token surfaces must be resolvable to a specific CSS custom property or design token (§5). Undeclared token surfaces are not permitted.

**View-state rules**
- View-state is always user-owned. Regeneration must not overwrite it (UX Doctrine I5).
- View-state is keyed by a compound key defined in §6.1. Changing any component of that key requires migration or invalidation.
- View-state is scoped to user scope only. It cannot be set or overridden by tenant or role.
- View-state that references a field, filter, or view that no longer exists after regeneration is orphaned and must be handled by the conflict resolution algorithm (§7).

**Emphasis variant rules**
- An emphasis variant must be declared as an approved variant in the pattern that contains it (10.3) and must be listed in the approved variant registry in this document (§4).
- Emphasis variants do not change the interaction grammar, action semantics, or state semantics.
- Emphasis variants may affect: which slot a component renders in, the depth of information shown within a slot, or the visual prominence of one area over another.
- Emphasis variants may be role-scoped (compiled into the projection) or user-scoped (saved in the preference store), as declared per variant in §4.

**Structural variant rules**
- A structural variant changes the layout structure, composition, or component boundaries of a pattern.
- Structural variants require all five of the following: (1) registry entry in §4, (2) approved variant surface in the pattern library (10.3), (3) `isStructuralVariant=true` in PersonalizationSurface, (4) dedicated test coverage per §11.2, (5) compiler-level IR support.
- Structural variants may be scoped to tenant or role. They may never be scoped to user.
- Introducing a new structural variant is a governed process defined in §8.

### 2.3 Boundary disambiguation: emphasis variant vs view-state

These two types share a common symptom (the user's app looks different) but are governed differently. The following rules clarify the boundary.

**The core distinction:**

> An emphasis variant answers "how should this type of content be presented?" — it is answerable without knowing the current data or the user's operational context.
> A view-state preference answers "what am I looking at right now?" — it encodes the user's current operational choices about which data to see and how.

**Disambiguation table:**

| Ambiguous case | Classification | Why |
|---|---|---|
| `column_choice` vs `key_fact_count` | `column_choice` = view-state; `key_fact_count` = emphasis | Column choice is a per-session operational decision about which data columns to show. Key fact count is a role-level presentation decision about information density in the standard card. |
| `pinned_view` vs nav structure | `pinned_view` = view-state; nav structure = invariant (I1) | Pinning a shortcut to a view is an operational preference. The navigation structure itself cannot be personalized at any scope. |
| `section_expansion` vs `section_grouping` | `section_expansion` = view-state; `section_grouping` = layout-driven (role) | Expansion state is transient — the user's current collapsed/expanded choice. Section grouping is a structural presentation decision about how sections are labeled and ordered. |
| `result_grouping` vs `saved_filter` | `result_grouping` = emphasis; `saved_filter` = view-state | Result grouping is a presentation decision about how results are organized in the UI. A saved filter is a data-selection decision about which records to show. |
| `ai_panel_visibility` vs `ai_assist_mode` | `ai_panel_visibility` = user preference (view-state category); `ai_assist_mode` = structural variant | Panel visibility is a per-user runtime toggle. AI assist mode is a structural change that adds or removes a flow step — set at tenant scope, not user scope. |

**Decision rule when classification is unclear:**

1. Does the variation require a change to the IR node tree? → Structural variant.
2. Does the variation change which data the user is viewing this session? → View-state.
3. Does the variation change the presentation of a content type universally for this position? → Emphasis variant.
4. Does the variation change only visual appearance without affecting elements? → Token-driven.

If more than one rule applies, classify at the higher governance level. When in doubt between emphasis and structural: apply the IR test (rule 1).

---

## 3. Scope system

### 3.1 Four scopes

| Scope | Definition | Who manages it | Applies to |
|---|---|---|---|
| **Global** | Foundry product defaults — the baseline when nothing else is configured | Foundry platform team | All tenants, all positions, all users |
| **Tenant** | Per-company configuration — overrides global for this company | Tenant admin | All positions and users within the tenant |
| **Role** | Per-position defaults — overrides tenant for this position | Position model / compiler-generated from semantic slice | All users assigned to this position |
| **User** | Per-user saved preferences — overrides role for this individual | User at runtime | This user only |

### 3.2 What each scope owns

**Global scope owns:**
- Default design token values (theme, density, spacing, motion, shadow)
- Default interaction grammar for all patterns
- Default action budget enforcement rules
- Default density mode: `default`
- Default conflict policy behavior

**Tenant scope owns:**
- Company-level design token overrides (brand colors, typography adjustments within bounds)
- Structural variant selections for positions within the tenant
- Tenant-level view hint overrides (e.g., different default sort for a specific entity type across all positions)
- Approved token-driven theme selection within the token vocabulary

**Role scope owns:**
- Default view hints for this position: default sort, default filter, default expanded sections (from `DefaultViewHints` in the projection)
- Emphasis variant selections for this position (e.g., `key_fact_count` for the RecordPage header)
- Density mode default for this position (overrides global; does not override user preference)
- Structural variants where scope permits

**User scope owns:**
- Saved filter state per view
- Saved sort preference per view
- Column choices per view
- Pinned views and sections
- Density mode override (within the two permitted modes only)
- Emphasis variant overrides where explicitly declared as user-scope in §4
- AI panel visibility where applicable
- Recent search behavior preferences

### 3.3 Precedence algorithm

The precedence order is: **global < tenant < role < user**. A higher-scope setting overrides a lower-scope default for the same surface.

**Full algorithm:**

```
For each personalization surface S in a position app view:

  1. Start with the global default for S.
  2. If tenant has a configuration for S and tenant scope ≤ S.scope: apply tenant value.
  3. If role has a configuration for S and role scope ≤ S.scope: apply role value.
  4. If user has a saved preference for S and user scope ≤ S.scope: apply user value.
  5. The final resolved value is the highest-scope value present.

Special cases:

  a. If a scope level has no configuration for S, the next lower scope's value persists unchanged.

  b. If the user has a saved preference for S but S.scope = role (user not permitted for this surface),
     the user preference is silently ignored. It is not deleted — it becomes dormant and reactivates
     if S.scope is later expanded to include user.

  c. Structural variants (isStructuralVariant = true) cap at role scope regardless of any stored 
     user preference. The algorithm stops at role scope for structural variants.
```

**Fallback behavior:**
When no scope has a configuration for a surface, the pattern's built-in default behavior applies. The pattern default is the pattern's own baseline from 10.3 — it is not a global configuration value. If the global default must deviate from the pattern baseline, it must be explicitly set at global scope.

### 3.4 Cross-position variant consistency

Variant values are per-position by default. Different positions within the same tenant may have different variant values for the same variant id. This is correct and intentional — variant values govern presentation, not semantics. UX Doctrine I9 (cross-position semantic consistency) governs action labels and semantics, not presentation variants.

The rules are:

**Role-scope variants (emphasis or structural):**
Each position independently declares its value. Position A may have `key_fact_count=3` and Position B may have `key_fact_count=4`. No cross-position review is required. These are position-specific presentation decisions.

**Tenant-scope structural variants:**
A tenant-scope structural variant is a single tenant-level setting that applies uniformly to all positions within the tenant. A tenant cannot have `autosave=enabled` for one position and `autosave=disabled` for another — the tenant-scope value propagates to all positions unless a role-scope override exists for a specific position. Role-scope values for the same variant id override the tenant-scope value for that position only, following the precedence algorithm (§3.3).

**Variant values and I9:**
Variant values never affect action labels or action semantics, so they cannot violate I9 by definition. However, a proposed structural variant that would alter how an action is presented or labeled in a way that differs across positions would constitute a design error, not a variant — it would be redesigned before approval (§8.2).

**When cross-position review is warranted:**
If a tenant-scope structural variant is applied to a pattern that appears in more than three positions, the workstream owner should review whether the variant value produces a coherent tenant-wide experience. This review is advisory, not a blocking gate — coherence is a quality concern, not a compliance requirement.

---

## 4. Approved variant registry

This is the authoritative list of all approved variants across all Foundry UI patterns. No variant may be used in a position app that does not appear here. A variant applied without a registry entry is an undeclared structural or emphasis change and is a compliance failure.

Each entry defines: the variant identifier, its classification type, the maximum scope at which it can be set, the default value, and the permitted alternate values.

---

### 4.1 CollectionView (P1)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `list_style` | Structural | Role | `semantic_list` | `compact_grid` | Compact grid requires explicit compiler declaration; replaces semantic record rows with a grid; aligns with doctrine D5 (grid is a compact structural variant only) |
| `filter_placement` | Emphasis | User | `rail_sidebar` | `chip_bar_inline` | Rail sidebar preferred; chip bar for narrow content contexts |
| `row_density` | Density mode | User | `default` | `compact` | Two modes only per doctrine D2 |
| `key_fact_count` | Emphasis | Role | `4` | `2`, `3` | Max 4 in list rows per pattern spec; min 2 |

---

### 4.2 RecordPage (P2)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `sidebar_position` | Structural | Role | `right` | `below_content` | Below-content collapses to single-column; use in compact or space-constrained position layouts |
| `section_grouping` | Layout-driven | Role (compiler) | Determined by `LayoutHints.groupedSections` | N/A | Not a user choice; governed entirely by ViewSpec.layoutHints; listed here for registry completeness |
| `key_fact_count` | Emphasis | Role | `8` | `4`, `5`, `6`, `7` | Range 4–8 per pattern spec; cannot exceed 8 |
| `timeline_placement` | Emphasis | Role | `inline_below_sections` | `tab` | Tab placement reduces visual weight; use when timeline is secondary to the primary work |
| `header_density` | Density mode | User | `default` | `compact` | Applies to RecordHeader region only |

---

### 4.3 OverviewMonitor (P3)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `summary_strip_layout` | Structural | Tenant | `tile_row` | `card_grid` | Card grid increases visual weight per metric; for positions where the summary strip is the primary daily work surface |
| `alert_list_position` | Emphasis | Role | `below_summary` | `sidebar` | Sidebar position gives alerts persistent visibility alongside summaries |
| `ai_panel_visibility` | User preference | User | `hidden` | `visible` | User can toggle; hidden by default per progressive disclosure (doctrine 1.4) |
| `pinned_summaries` | View-state | User | None pinned | Any `StateSummarySpec` id(s) | User-owned view-state; orphaned if referenced StateSummarySpec is removed |

---

### 4.4 SearchResults (P4)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `result_grouping` | Emphasis | User | `grouped_by_entity_type` | `flat_unified` | Flat unified useful when position's search is scoped to a single entity type |
| `scope_selector_placement` | Emphasis | Role | `above_results` | `sidebar` | Sidebar for positions that regularly narrow by entity type before viewing results |
| `recent_searches` | User preference | User | `shown` | `hidden` | User can suppress; default shown per doctrine 1.3 (minimal friction on entry) |

---

### 4.5 ExceptionResolutionView (P5)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `context_section_depth` | Emphasis | Role | `6_facts` | `3_facts` | 3-fact minimal for positions where exception type is self-explanatory from the header alone |
| `timeline_inclusion` | Emphasis | Role | `hidden` | `inline` | Inline timeline for positions that regularly investigate exception history before resolving |
| `exception_list_mode` | Structural | Role | `single_exception` | `list_with_drill_in` | List mode for positions that review many exceptions per session; adds a list-to-detail sub-navigation layer |

---

### 4.6 ApprovalReviewView (P6)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `supporting_detail_placement` | Emphasis | Role | `below_context` | `sidebar` | Sidebar when supporting documents are central to the approval decision for this position |
| `approval_history_placement` | Emphasis | Role | `inline` | `collapsible_section` | Collapsible when history is rarely consulted during active review |
| `consequence_section_prominence` | Emphasis | Role | `standard` | `highlighted_banner` | Banner for high-stakes approval types where consequence visibility is critical |

---

### 4.7 CreateEditFlow (F1)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `form_layout` | Structural | Role (compiler-determined) | `single_section` | `stepped_multi_section` | Determined by LayoutHints.groupedSections count; not a runtime user choice |
| `review_step` | Emphasis | Role (compiler-determined) | `omit` | `include` | Required for high-stakes forms; declared in ActionSpec context; not a user preference |
| `autosave` | Structural | Tenant | `disabled` | `enabled` | Requires backend draft API support; must be declared at tenant scope before compiler can enable |
| `section_navigator_style` | Emphasis | Role | `sidebar_list` | `top_stepper` | Top stepper for flows with ≤3 sections; sidebar list for 4+ sections |

---

### 4.8 ApprovalFlow (F2)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `approver_selection` | Structural | Tenant | `prefilled` | `user_selectable` | User-selectable from a constrained set only — not free-form; requires tenant-level approver set configuration |
| `supporting_context_step` | Emphasis | Role | `shown` | `hidden` | Hide only when the subject entity already provides full context to the approver |

---

### 4.9 ExceptionResolutionFlow (F3)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `impact_preview_step` | Severity-governed | N/A | `omit` (severity=warning) | `include` (severity=critical, required) | Not a configurable variant — fully determined by AlertSpec.severity. Listed here for registry completeness and to prevent mis-classification as user preference. |
| `resolution_input_layout` | Emphasis | Role | `single_section` | `multi_section` | Multi-section for complex resolution inputs spanning more than 5 fields or multiple decision areas |

---

### 4.10 AssistedSetupFlow (F4)

| Variant id | Type | Max scope | Default | Alternate(s) | Notes |
|---|---|---|---|---|---|
| `ai_assist_mode` | Structural | Tenant | `suggestions_only` | `auto_fill_with_review` | Auto-fill mode requires AI service support and adds an explicit AI review step before completion |
| `skip_availability` | Emphasis | Role (compiler-determined) | `per_step_skip` | `no_skip` | No-skip for mandatory configuration surfaces; governed by PersonalizationSurface.configurationSchema required field flags; not a user runtime choice |
| `welcome_step` | Emphasis | Role | `full_welcome` | `minimal_header` | Minimal header for re-entry flows or positions where setup context is already established |

---

### 4.11 Structural variant IR transformation table

This table defines exactly what changes in the IR node tree for each structural variant. It is the binding contract between the approved variant registry and the compiler. The output contract IR trees in 10.3 define the default node structure; this table defines the delta when the variant is active.

The compiler must implement each row here as a distinct IR branch. A structural variant without an entry in this table is not compiler-ready regardless of whether it passes the §8.2 approval requirements.

| Variant id | Pattern | Default IR structure | Variant IR structure | Exact node changes |
|---|---|---|---|---|
| `list_style = compact_grid` | CollectionView | `RecordList > RecordRow[]` each row: Identifier, StatusBadge, KeyFactsStrip, InlineActions | `RecordGrid > GridItem[]` each item: Identifier, StatusBadge, 2 facts max | RecordList node → RecordGrid container; RecordRow → GridItem; KeyFactsStrip in GridItem capped at 2 facts (not the row max of 4); InlineActions removed from GridItem; BulkActionBar and PaginationControl remain unchanged |
| `sidebar_position = below_content` | RecordPage | Two-column layout: ContentSections (main) + sidebar (right): TaskContextPanel, AIAssistPanel | Single-column: ContentSections → sidebar below in DOM order | Layout wrapper changes from CSS two-column grid to single-column stack; TaskContextPanel and AIAssistPanel move from right column to a stacked section appended after ContentSections; no child node changes — only the container layout node and DOM order change |
| `summary_strip_layout = card_grid` | OverviewMonitor | `SummaryStrip > MetricTile[]` horizontal flex row | `SummaryCardGrid > MetricCard[]` CSS grid layout | SummaryStrip node → SummaryCardGrid; MetricTile → MetricCard (expanded format: larger value display, optional trend indicator, more vertical space per item); grid column count responsive; NavigationLink and AlertIndicator remain present on MetricCard |
| `exception_list_mode = list_with_drill_in` | ExceptionResolutionView | Single exception: ExceptionHeader → ExceptionDetail → KeyFactsStrip → ActionPanel → optional EscalationPanel | `ExceptionListContainer > ExceptionListItem[] + ExceptionDetailPane` | Adds ExceptionListContainer as root wrapper; adds ExceptionListItem (severity badge + type label + entity reference, one per active alert) as a navigable list; ExceptionDetailPane replaces the prior single-exception layout and renders the selected exception's full detail on selection; ActionPanel and EscalationPanel move inside ExceptionDetailPane |
| `form_layout = stepped_multi_section` | CreateEditFlow | `ActiveSection > SmartFormSection` (single, no navigator); ProgressIndicator inactive | `StepNavigator + ActiveSection > SmartFormSection` (stepped with navigator); ProgressIndicator active with step count | Adds StepNavigator node (sidebar list or top stepper per `section_navigator_style` variant); FlowHeader.ProgressIndicator activates with current-step/total display; FlowNavigationBar.NextAction adds step-level validation before advancing; Submit action appears only on final step |
| `autosave = enabled` | CreateEditFlow | DraftSaveIndicator node absent; CancelAction has single discard path | DraftSaveIndicator node present and active; CancelAction adds "Save draft" option | DraftSaveIndicator changes from conditional-absent to conditionally-visible (shown when unsaved changes exist); CancelAction modal gains "Save draft" branch before discard; backend autosave API hook activates on field-blur events |
| `approver_selection = user_selectable` | ApprovalFlow | Step: Approver confirmation renders `ApproverList` as a static read-only display of required approvers | Step: Approver confirmation renders `ApproverList` as an interactive multi-select bounded to the approved approver set | ApproverList node changes from static display to interactive selector; validation added to the Approver confirmation step (must select ≥1 from the constrained set before Next is enabled); the constrained approver set is resolved from the ApprovalSpec at compile time — not free-form input |
| `ai_assist_mode = auto_fill_with_review` | AssistedSetupFlow | No AI review step between Configuration and Review steps; AI suggestions shown inline per configuration step | Adds `Step: AI review > ReviewSummary` between Configuration steps and final Review step | Adds an AI review step node after all Configuration steps; AI-suggested values are held in pending state (not applied) until this step; the step renders ReviewSummary showing only AI-suggested values with per-item Accept/Edit controls; values accepted here are applied before the final Review step |

---

### 4.12 PersonalizationSurface schema cross-reference

This table maps the `surfaceType` vocabulary from Position Projection Schema v0 §4.15 (10.2) to the variation taxonomy in this document (§2.1). It makes explicit how the schema type maps to governance rules.

The `PersonalizationSurface` schema is defined in 10.2 §4.15. This document does not re-define it — it governs how each type behaves.

| 10.2 `surfaceType` value | Variation type (§2.1) | Max scope allowed | Stored in preference service | Compile-time or runtime |
|---|---|---|---|---|
| `visual_theme` | Token-driven | Tenant | No — resolved from token system | Compile-time (token resolution) |
| `density_mode` | Token-driven + User preference | User | Yes — user's density choice | Runtime (user toggle) |
| `saved_filter` | View-state | User | Yes | Runtime |
| `sort_preference` | View-state | User | Yes | Runtime |
| `column_choice` | View-state | User | Yes | Runtime |
| `pinned_view` | View-state | User | Yes | Runtime |
| `section_expansion` | View-state | User | Yes | Runtime |
| `emphasis_variant` | Emphasis variant | Role or User (per §4 registry) | Yes when user-scope; no when role-scope (compiled) | Role: compile-time; User: runtime |
| `structural_variant` | Structural variant | Role (never user) | No — compiled into PositionProjection | Compile-time only |

**Key rule from this mapping:** Any `PersonalizationSurface` with `surfaceType=structural_variant` must have `isStructuralVariant=true`, must have max scope ≤ role, and must never appear in the runtime preference store. If a structural_variant entry is found in the preference store for a user, it is stale data and must be discarded silently.

---

## 5. Token system integration

Token-driven variation does not appear in the approved variant registry (§4) because it does not change structure or composition. It is governed separately here.

### 5.1 Token categories

| Category | What it governs | Configurable scope |
|---|---|---|
| `color.brand` | Primary brand color and its tonal scale | Tenant |
| `color.semantic` | Status and severity colors: success, warning, error, info | Global only (tenant override not permitted — see §5.3) |
| `color.surface` | Background, border, and container surface colors | Tenant |
| `color.text` | Text colors: primary, secondary, muted, inverse | Tenant |
| `spacing.base` | Base spacing unit; all other spacing values derive from this | Global |
| `spacing.density.default` | Spacing multipliers for default density mode | Global |
| `spacing.density.compact` | Spacing multipliers for compact density mode | Global |
| `typography.scale` | Font size scale: display, heading, body, caption | Tenant (within bounds) |
| `typography.family` | Font family: primary and monospace | Tenant |
| `shadow.elevation` | Box shadow levels: 0 flat, 1 raised, 2 overlay, 3 modal | Global |
| `motion.duration` | Animation duration scale: instant, fast, moderate | Global |
| `motion.easing` | Easing curves for transitions | Global |
| `radius.base` | Border radius for interactive elements | Tenant |

### 5.2 Scope of token configuration

- Users do not set token values directly. Token variation is resolved from global → tenant → role using the precedence algorithm (§3.3).
- Density mode preference (`density_mode` surface type) maps to the `spacing.density.default` or `spacing.density.compact` token set. Switching density mode is a surface preference, not a token override — it resolves to a pre-built token set. The implementation path is: user sets a `density_mode` preference in the preference store (view-state layer) → at render time the preference service reads the stored value → the UI resolves the named mode to the corresponding pre-built spacing token set. Implementers must not treat this as a direct token mutation; the token values themselves do not change, only which pre-built set is active.
- Role-level token configuration is limited to selecting a pre-approved theme or density default from a constrained set — not setting arbitrary token values.

### 5.3 What tokens cannot change

Token variation cannot alter:
- Semantic color tokens (`color.semantic`): severity colors (Warning, Critical, Informational) must remain consistent and distinct across all tenants (invariant V1).
- Focus ring appearance: focus indicator visibility and contrast must satisfy WCAG 2.1 AA under any token configuration (P9, I6).
- The visual distinction between AI-generated content and user-entered content (invariant V2, P11).
- The visual isolation of destructive actions from primary actions (invariant V3, A5).

Typography scale overrides by tenant are bounded: body text floor is 12px, ceiling is 20px. No tenant token override may produce a text/background combination that fails WCAG 2.1 AA contrast.

---

## 6. View-state storage model

View-state preferences are user-owned runtime data. They are not schema artifacts — they are stored in a runtime preference service keyed to schema-level identifiers.

### 6.1 Storage key structure

Each saved view-state preference is stored under a compound key:

```
{tenantId} / {positionId} / {viewId} / {surfaceId} / {userId}
```

- `tenantId`: from PositionIdentity.tenantId
- `positionId`: from PositionIdentity.positionId — invariant across regenerations
- `viewId`: from ViewSpec.viewId — invariant across regenerations
- `surfaceId`: from PersonalizationSurface.surfaceId
- `userId`: runtime user identity

Any change to any key component (positionId, viewId, surfaceId) requires migration or invalidation. These identifiers are invariants — the schema versioning rules from schema §6.2 apply to all of them.

### 6.2 View-state surface types and stored values

| Surface type | What is stored | Invalidation trigger |
|---|---|---|
| `saved_filter` | FilterKey + applied value(s) | FilterKey removed from filterSources; viewId changed; positionId changed |
| `sort_preference` | FieldKey + direction | FieldKey removed from entity model; viewId changed |
| `column_choice` | Ordered list of visible column keys | Any referenced column key removed from entity model |
| `pinned_view` | ViewId reference | ViewId removed from workSurface.views |
| `section_expansion` | SectionKey + expanded boolean | SectionKey removed from LayoutHints.groupedSections |
| `emphasis_variant` | Variant id | Variant id removed from the approved registry (§4) or S.scope changed to exclude user |
| `density_mode` | `default` or `compact` | Never invalidated — always one of two valid values |
| `pinned_summaries` | List of StateSummarySpec ids | Referenced StateSummarySpec id removed from projection |
| User preference (boolean/enum: ai_panel_visibility, recent_searches) | The selected value | Never invalidated unless the surfaceId itself is removed |

### 6.3 View-state lifecycle

```
User saves a preference
    → stored in preference service under compound key

Regeneration occurs (new positionVersion compiled)
    → on next app load: preference service detects new positionVersion
    → validate all stored preferences for this positionId against new PositionProjection
    → for each stored preference:
        - compound key resolves fully → apply (no change, no notification)
        - any key component orphaned → conflict resolution algorithm §7
    → record new positionVersion in preference service after successful validation
```

### 6.4 View-state and regeneration

The preference service must be regeneration-aware. When a new `positionVersion` is detected on app load:

1. All stored preferences for that `positionId` are validated against the new projection.
2. Preferences that fully resolve are applied without user notification.
3. Preferences with orphaned key components trigger the conflict resolution algorithm (§7).
4. The validated `positionVersion` is recorded after successful validation.

A regeneration that results in zero orphaned preferences must produce zero notifications. Notification only on actual data loss.

### 6.5 Performance constraints

These constraints apply to the preference service and validation pipeline. They are not implementation prescriptions — they are requirements the implementation must satisfy.

**Preference loading:**
- Preferences for the current `positionId` are loaded eagerly on app init, before any view renders.
- All preferences for a positionId are loaded in a single batch fetch — not per-view on demand.
- The loaded preference set is cached in-memory for the session. Individual views read from the in-memory cache, not from the preference service.

**Validation timing:**
- Version validation (detecting a new `positionVersion`) runs once at app init.
- If no new version is detected, validation is skipped entirely for that session. Per-view lazy validation is not permitted — it would create inconsistent state within a session.
- If a new version is detected, all preferences for the positionId are validated in a single synchronous pass before any view renders. No view may render with an unvalidated preference set.

**Cache invalidation triggers:**
- `positionVersion` changes (detected at app init)
- User explicitly saves a new preference (immediate cache update; preference service write is async)
- Conflict resolution discards a preference (immediate cache update)
- Session end — cache is not persisted between sessions; preferences are reloaded from the preference service on next init

**Storage size guidance (soft limits):**
- Preference entries per `positionId` per `userId`: warn at 40; log at 50. Exceeding 50 entries suggests variant sprawl or personalization surface overuse.
- Stored value size per preference entry: no individual preference value should exceed 2 KB. Larger values indicate the wrong data is being stored (e.g., full entity records rather than ids or keys).

**Lazy loading exception:**
- Emphasis variant preferences that are user-scope may be validated lazily when the view that contains them is first rendered, provided that: (a) the view is not the defaultLandingViewId, and (b) the emphasis variant does not affect navigation or action semantics. This is the only permitted exception to the single-pass validation rule.

---

## 7. Conflict resolution algorithm

This section expands UX Doctrine §3.5 (Scenarios A–D) into an operational algorithm. The algorithm runs on every app load where a new `positionVersion` is detected for the current user's position.

### 7.1 Full algorithm

```
Input: stored_preferences[], new_PositionProjection

For each preference P in stored_preferences:

  Step 1 — Check viewId
    Does P.viewId exist in new_PositionProjection.workSurface.views?
      NO and the view was removed entirely → Scenario A → goto DISCARD (full)
      NO and the view's pattern changed → Scenario D → goto DISCARD (full)
      YES → continue

  Step 2 — Check surfaceId
    Does P.surfaceId exist in new_PositionProjection.personalizationHooks.allowedSurfaces?
      NO → Scenario A (surface removed) → goto DISCARD (full)
      YES → continue

  Step 3 — Check value validity
    Is P.value still valid for P.surfaceId? Apply these checks by surface type:
      saved_filter: does P.filterKey exist in the view's filterSources?
      sort_preference: does P.fieldKey exist in the current entity model?
      column_choice: do all referenced column keys still exist?
      pinned_view: does the referenced viewId still exist?
      section_expansion: does the referenced sectionKey exist in LayoutHints?
      emphasis_variant: is the variant id still in the registry (§4)?
      density_mode: always valid
      user preferences (boolean/enum): always valid if surfaceId exists
      pinned_summaries: do all referenced StateSummarySpec ids still exist?
        → any orphaned ids → DISCARD (partial): remove orphaned ids only; retain valid ids

      INVALID (full) → Scenario A → goto DISCARD (full)
      INVALID (partial) → goto DISCARD (partial) for the invalid portion
      VALID → goto APPLY

  APPLY
    Apply the preference. No user notification.

  DISCARD (full)
    Remove the entire preference record.
    Apply the conflictPolicy from PersonalizationHooks:
      preserve_user: silently discard — implementation must log per minimum log schema below; no user notification
      reset_to_role: discard; apply the role-scope default for this surface
      notify_and_preserve: discard; notify user (see §7.2); apply role-scope default

    Minimum log record for preserve_user discard (all five fields required):
      {
        positionId:   string  — from PositionIdentity.positionId
        userId:       string  — runtime user identity
        viewId:       string  — from ViewSpec.viewId (or null if view was removed)
        surfaceId:    string  — from PersonalizationSurface.surfaceId
        reason:       enum    — one of: view_removed | surface_removed | field_orphaned |
                                        variant_removed | pattern_changed | structural_stale
      }
    The orphaned value itself must not be logged (may contain user data). Log the reason only.

  DISCARD (partial)
    Remove only the orphaned portion. Retain the valid remainder.
    Example: saved_filter with 3 filter keys, one key orphaned →
      remove orphaned key; retain the 2 valid filter keys; apply the trimmed preference.
    Apply notification per conflictPolicy (notify only about the discarded portion).
```

### 7.2 User notification requirements

When a preference is discarded (full or partial) and `conflictPolicy = notify_and_preserve` (recommended):

The notification must include all of the following:
1. **What changed** — plain language. E.g., "Your saved filter for 'Supplier Region' was removed because this filter is no longer available in this view."
2. **What was reset** — the surface name and what it was reset to.
3. **A direct reconfiguration path** — a link or action reaching the relevant view's personalization settings.

Notification timing: shown once on the first app load after the change is detected. Not repeated on subsequent loads. The notification uses the StatusBanner component at informational severity — it does not interrupt the workflow.

Silent loss of personalization is not acceptable. If `conflictPolicy = preserve_user`, all discards must still be written to a system log even if not surfaced to the user.

### 7.3 Scenario reference

| Scenario | Trigger | Resolution |
|---|---|---|
| **A — Orphaned preference** | Field, filter, view, or surface referenced by a preference no longer exists in the new projection | Discard orphaned portion; notify per conflictPolicy; apply role-scope default |
| **B — Role default introduced over user preference** | A new role default covers a surface the user has already personalized | Preserve user preference where it remains valid; role default fills only unconfigured surfaces — it does not overwrite existing valid user preferences |
| **C — Tenant default changes** | Tenant-level configuration changes after regeneration | Preserve role and user preferences above tenant level where still valid; new tenant default fills surfaces not covered by role or user configuration |
| **D — Pattern change on a view** | ViewSpec.pattern changes for a view (itself a breaking change per UX Doctrine I4 and schema §6.2) | Treat as Scenario A: discard all view-state preferences for that view; notify user; apply the new pattern's defaults |

### 7.4 Navigation graph changes and personalization

The navigation graph (NavigationSpec in 10.2 §4.2a) is schema-level. It is governed by invariant I1 (navigation structure does not change without a versioned migration) and is not a personalization surface. However, navigation changes can invalidate user-scope view-state preferences. This section defines the interaction.

**When nav items are added (new view added to workSurface.views):**
- No stored preference is affected — the new view has no prior preferences.
- The new view uses role-scope defaults and pattern defaults on first load.
- No conflict resolution triggered.

**When a nav item is removed (viewId removed from workSurface.views):**
- Any `pinned_view` preference referencing the removed viewId is orphaned → Scenario A.
- Any `saved_filter`, `sort_preference`, `column_choice`, or `section_expansion` preferences keyed to the removed viewId are also orphaned → Scenario A.
- Conflict resolution discards all preferences for that viewId and notifies per conflictPolicy.
- This is a breaking change — it must bump `positionVersion` per schema §6.2.

**When `navOrder` changes (view reordered in navigation):**
- `navOrder` changes are non-breaking per schema §6.2 (non-versioned change for display-order changes).
- No personalization conflict is triggered.
- Users will see their navigation items in the new order on next app load. This is by design — the user does not "own" the nav order, only shortcuts to views.

**When `isHiddenFromNav` changes (view hidden from nav but still reachable):**
- The viewId still exists in workSurface.views — no orphan.
- `pinned_view` preferences for that viewId remain valid and still work as shortcuts.
- The view is reachable via its shortcut even if not in the main nav.
- No conflict resolution triggered.

**When `defaultLandingViewId` changes:**
- `defaultLandingViewId` is a schema-level field in DefaultViewHints — it is not a personalization surface in v0.
- The new landing view takes effect on next app load for all users.
- If a user had a `pinned_view` shortcut to the old landing view, it remains valid (the view still exists).
- There is no user-scope "saved landing view" preference in v0. If this is added in a future version, it must handle the interaction with `defaultLandingViewId` changes explicitly.

**When `navStyle` changes (sidebar → topbar → tabs):**
- `navStyle` is declared as an optional enum in NavigationSpec (10.2 §4.2a) and must be an approved variant.
- This is a structural variant on the navigation container — it is schema-level and compile-time.
- It cannot be set by users. No user-scope preference is affected.
- If `navStyle` changes between regenerations, it is a structural change to the navigation layout. It must follow the structural variant governance process (§8) and constitutes a breaking change requiring a `positionVersion` bump.

**General rule:**  
Personalization operates within the navigation graph — it can save shortcuts to views that exist in the graph. It cannot modify the graph itself. Any personalization preference that references a navigation element (viewId, navLabel, navOrder) is governed by what the current compiled NavigationSpec declares. When the NavigationSpec changes, the conflict resolution algorithm (§7) handles the downstream impact on view-state preferences.

---

## 8. Structural variant governance

Structural variants carry the highest governance burden. Introducing one without proper process erodes the conservation layer.

### 8.1 What qualifies as structural

A variation is structural if it meets any of the following conditions:
- It changes the number, identity, or order of named layout regions in a pattern.
- It substitutes one component for a different component in a slot.
- It changes the parent-child composition of the pattern's IR node tree.
- It adds or removes a primary content area.
- It would require the IR output contract in 10.3 to change.

A variation is NOT structural if:
- It changes only visual appearance, spacing, or typography within a region.
- It changes which data fields are displayed within an existing slot.
- It changes the visual prominence or ordering of content within a slot.
- It shows or hides an optional region that does not affect required regions.

When in doubt: if the IR output contract (10.3) would need to change, the variation is structural.

### 8.2 Approval requirements

All five conditions must be met before a structural variant is considered approved:

| Requirement | Description |
|---|---|
| **Registry entry** | Documented in §4 of this document with variant id, type, max scope, default, alternate, and notes |
| **Pattern library entry** | Declared as an approved variant surface in the corresponding pattern spec (10.3) |
| **Schema declaration** | Declared with `isStructuralVariant=true` in the relevant PersonalizationSurface in the PositionProjection |
| **Test coverage** | Dedicated test scenarios written and passing per §11.2 |
| **IR support** | Compiler team has declared and implemented the IR branch for this variant before it is enabled in production |

A structural variant used in production without all five conditions met is a compliance failure.

### 8.3 Structural variant scope constraint

Structural variants may be set at:
- **Tenant scope**: applies the structural variant to all positions within the tenant where it is declared
- **Role scope**: applies the structural variant to all users of a specific position

Structural variants may NOT be set at:
- **User scope**: a user cannot change the structural layout of their position app
- **Runtime** (by the user in-session): structural variants are compile-time declarations in the PositionProjection

### 8.4 Introducing a new structural variant

The process for introducing any new structural variant:

1. Document the business need and the position context that requires the variant.
2. Confirm the variation is structural per §8.1.
3. Confirm max scope is tenant or role.
4. Propose the variant entry in §4 of this document (registry entry).
5. Update the affected pattern spec in 10.3 with the approved variant surface entry.
6. Declare `isStructuralVariant=true` in the PersonalizationSurface schema.
7. Write test scenarios per §11.2.
8. Compiler team implements the IR branch.
9. Review and approve all five conditions before enabling.

### 8.5 Deprecating a structural variant

When a structural variant must be removed:

1. Mark the variant as deprecated in §4 with a deprecation note and the version at which deprecation begins.
2. Notify tenants and roles currently using the deprecated variant.
3. Provide a migration path: map to the default behavior or to a replacement variant.
4. Observe a dual-support window of minimum two regeneration cycles per schema §6.4.
5. After the window closes: remove from the pattern library (10.3), remove from this registry (§4), and remove IR support.

---

## 9. Migration and compatibility

### 9.1 PositionProjection major version bump

When a `positionVersion` major bump occurs:
- All stored view-state preferences for the affected position are validated per §7 on next app load.
- Emphasis variant preferences referencing a removed variant id are treated as orphaned (Scenario A).
- Structural variant declarations may differ from the prior version; user-scope data (which should not exist for structural variants — see §8.3) is discarded if found.

### 9.2 Approved variant deprecation

When a variant in §4 is deprecated (§8.5):
- Active configurations using the deprecated variant (at any scope: tenant, role, or stored user preference) are migrated forward per the deprecation migration path before the window closes.
- If an automated mapping exists (old variant id → new variant id): migration applied silently; no notification.
- If no mapping exists: setting reset to default; affected admin or user notified per §7.2 notification requirements.

### 9.3 Token system changes

Token changes do not affect view-state preference keys or values. Visual changes take effect on next app load without migration.

Exception: if a token change affects semantic color tokens, AI-action distinguishability, or destructive action isolation, it must be reviewed as a potential breaking change to invariants V1, V2, V3 before deployment.

### 9.4 Schema-level surface removal

When a `PersonalizationSurface.surfaceId` is removed from a projection:
- All stored preferences keyed to that surfaceId are orphaned and discarded per §7 (Scenario A).
- Removing a surfaceId is a breaking change to the personalization contract.
- It must be declared as a major version bump in `positionVersion` per schema §6.2.

---

## 10. Invariants — the non-negotiable boundary

The following properties of a position app can never be changed through any personalization mechanism, at any scope, by any variation type. Any proposed variant that would alter any of these properties is not a variant — it is a breaking change requiring full review.

### 10.1 From UX Doctrine invariants (I1–I9)

| Invariant | What it protects |
|---|---|
| I1 | Navigation structure — nav model does not change between regenerations without a versioned migration |
| I2 | Primary action semantics — primary action label and meaning for a given record state |
| I3 | State labels — business state labels remain stable across regenerations |
| I4 | Pattern grammar — a view's pattern type does not change silently during regeneration |
| I5 | Personalization data — regeneration does not overwrite user-saved preferences |
| I6 | Accessibility — focus management, aria labels, and keyboard navigation not degraded |
| I7 | Telemetry hook keys — event names treated as breaking change if modified |
| I8 | Action budget compliance — no variant may exceed 1 primary + max 3 secondary + overflow |
| I9 | Cross-position semantic consistency — same business action, same label, across all positions |

### 10.2 From UX Doctrine §3.2

These surfaces are explicitly excluded from personalization at any scope:
- Navigation logic and structure
- Action semantics and placement rules
- State semantics and validation behavior
- Approval and confirmation interaction patterns
- Core pattern slot definitions and composition rules

### 10.3 Additional invariants declared by this model

| Invariant | What it protects |
|---|---|
| V1 | Severity color semantics — Warning and Critical visual treatment cannot be altered by any token override at any scope |
| V2 | AI-action distinguishability — the visual treatment that distinguishes AI-generated content from user-entered content cannot be removed by any variant or token change |
| V3 | Destructive action isolation — the visual separation of destructive actions from primary actions (A5) cannot be removed by any variant |
| V4 | Structural variants are never user-scope — no user preference may alter the structural layout of a position app |
| V5 | Empty states are always communicative — no variant may replace an EmptyState component with a blank region |
| V6 | Approval controls are always both visible — no variant may hide either the Approve or Reject action from ApprovalReviewView while a pending decision exists |

---

## 11. Testing model

### 11.1 Emphasis and view-state variant testing

For every emphasis variant and view-state surface declared in §4, the following test scenarios are required before the variant is enabled in production:

| Test type | What to verify |
|---|---|
| Default state | Pattern renders correctly with no variant applied |
| Variant applied | Pattern renders correctly with the variant applied at its declared max scope |
| Variant + compact density | Variant does not break under compact density mode |
| Accessibility in variant | Focus management, aria labels, and keyboard navigation preserved in variant state |
| Action budget in variant | Action count does not exceed the budget (UX Doctrine §6) in the variant state |
| Conflict resolution | When variant preference is orphaned, conflict resolution applies correctly and default is restored |
| Regeneration stability | Variant preference survives a non-breaking regeneration; is correctly invalidated by a breaking regeneration |

### 11.2 Structural variant testing (additional requirements)

Structural variants require all tests from §11.1 plus:

| Test type | What to verify |
|---|---|
| IR correctness | Generated IR node tree matches the declared structural variant output contract from 10.3 |
| Default → variant | UI renders correctly when the structural variant is active in the compiled projection |
| Invariant compliance | All invariants from §10 hold in the variant state |
| Cross-pattern composition | Pattern composition rules (10.3 §2.2) that involve this pattern still hold in the variant state |
| Back navigation | Back navigation, scroll restoration, and filter state preservation work correctly in the variant layout |
| No user-scope leakage | Confirm that user-scope preferences cannot override the structural variant at runtime |

### 11.3 Token system testing

| Test type | What to verify |
|---|---|
| Contrast compliance | All text/background combinations under any tenant token override pass WCAG 2.1 AA |
| Severity color invariance | V1 — Warning and Critical severity colors are not altered by any tenant token override |
| AI-action distinguishability | V2 — AI-generated content remains visually distinguishable under every valid token combination |
| Density mode switch | Switching between Default and Compact produces correct spacing and font scaling without layout breakage |
| Focus ring invariance | Focus ring is visible and WCAG 2.1 AA compliant under every valid token combination |

---

## 12. Open seams

| Seam | Connects to | Current status at v0 |
|---|---|---|
| Design token system | Token implementation layer (CSS custom properties, theme provider) | Token categories defined here (§5.1); implementation substrate defined in 10.7 (Primitive Mapping and Token Strategy) |
| Runtime preference store | Backend preference service | Storage key structure defined here (§6.1); service API, persistence model, and TTL policy are out of scope for this document |
| Structural variant IR branches | Compiler intermediate representation | Variant ids and types defined here; IR node tree branches defined in 10.3 output contracts; IR compilation targets are a compiler workstream concern |
| Component-level personalization surfaces | Semantic Component Library v0 (10.5) | Component types referenced in PersonalizationSurface.targetComponentTypes; component-level variant surfaces will be defined in 10.5 and must be consistent with this model |
| Conflict resolution notification UI | StatusBanner or notification system | Notification requirements defined here (§7.2); StatusBanner component spec defined in 10.5 |
| Personalization settings UI | A dedicated settings surface within the position app | Not modeled at v0; the surface is acknowledged as necessary; pattern and entry point are deferred to 10.6 workflow prototypes. Likely candidates: a RecordPage variant (for viewing and editing saved preferences as a structured record) or a lightweight AssistedSetupFlow-style settings flow (for guided reconfiguration). 10.5 authors specifying configuration-related components (SmartFormSection, ReviewSummary, ConfigurationField renderers) should plan for both candidates until 10.6 resolves the pattern choice. |
| Tenant admin configuration | Admin portal for tenant-scope variant and token configuration | Tenant scope declared here (§3.1); admin UI is outside the UI/UX workstream scope |

---

## 13. Milestone 4 exit criteria — self-assessment

| Exit criterion | Status | Evidence |
|---|---|---|
| Invariants versus variable surfaces are explicit | Met | §10 exhaustive invariant list (I1–I9, doctrine §3.2, V1–V6); §2 defines all variation types and their limits |
| Precedence and fallback rules are defined | Met | §3.3 full precedence algorithm with special cases and fallback behavior explicitly specified |
| Structural variants are tightly bounded | Met | §8 governance process with all five approval requirements; §4 registry as sole authority; V4 invariant (never user-scope); §8.1 structural qualification test |
| The model is supportable and testable | Met | §11 testing model per variation type; §6 view-state storage model with defined lifecycle and invalidation; §7 operational conflict resolution algorithm with step-by-step logic |

---

*This document is version 0. It is the first executable version sufficient to govern pattern variation and personalization within Foundry position apps. It is expected to be refined once the Semantic Component Library (10.5) introduces component-level personalization surfaces, and once the Golden Workflow Prototypes (10.6) exercise this model against real workflows.*
