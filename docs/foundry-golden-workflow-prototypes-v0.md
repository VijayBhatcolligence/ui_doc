# Foundry Golden Workflow Prototypes v0

**Status:** First working draft  
**Workstream:** UI/UX — Work Product 10.6  
**References:** Foundry UX Doctrine v0 (10.1), Foundry Position Projection Schema v0 (10.2), Foundry UI Pattern Library v0 (10.3), Foundry Pattern Variability and Personalization Model v0 (10.4), Foundry Semantic Component Library v0 (10.5), Foundry UI/UX Workstream Charter v1  
**Purpose:** Validate the UI grammar by tracing three representative SMB business workflows end-to-end through the complete document chain. Prove that each workflow can be fully expressed using only the vocabulary defined in 10.1–10.5, and surface any gaps while correction is still cheap.

These are **grammar traces**, not visual mockups and not React code. Each prototype starts from a position app context and follows the grammar — selecting patterns, filling slots with components, applying variants, and making generator implications explicit.

---

## 1. Scope and validation method

### 1.1 Purpose of this document

10.6 is the validation milestone for the entire grammar. The three preceding specification layers (patterns, variability model, component library) were built top-down. This document tests the grammar bottom-up: start from business intent, flow forward through schema → pattern → variant → component, and verify that the system closes at each step.

**Four validation angles per workflow:**

| Angle | Question answered |
|---|---|
| Pattern coverage | Can the workflow's screens and flows be fully expressed using only patterns from 10.3? |
| Component coverage | Can every slot in every pattern be filled using only components from 10.5? |
| Variant exercise | Do the 10.4 personalization surfaces produce the expected behavior in this workflow? |
| Generator implications | What does the compiler need to do — and is the grammar sufficient to drive those decisions deterministically? |

### 1.2 Three workflows

| # | Workflow | Position | Primary patterns |
|---|---|---|---|
| W1 | Purchase Order Creation | Procurement Coordinator | CollectionView → CreateEditFlow |
| W2 | Purchase Order Approval | Finance Manager | CollectionView → ApprovalReviewView → ApprovalFlow |
| W3 | Low-Stock Exception Resolution | Warehouse Manager | OverviewMonitor → ExceptionResolutionView → ExceptionResolutionFlow |

All three share a company context: **Apex Components Ltd.**, a 45-person precision metal components manufacturer running Foundry on a single tenant deployment.

### 1.3 How to read a prototype

Each workflow is structured in eight sections:

| Section | Content |
|---|---|
| Context | Position, business purpose, trigger event, expected end state |
| PositionProjection excerpt | The 10.2 schema slice driving this workflow (JSON-like) |
| Pattern selection trace | Which patterns are selected and which 10.3 rule selects them |
| Screen sequence | Each screen or flow step: pattern, slots filled, component instances, state transitions |
| Variant surfaces exercised | Active 10.4 variants, their scope, and their effect |
| Personalization walk | One traced personalization scenario through 10.4 §5 resolution rules |
| Generator implications | What the compiler must do to produce this workflow |
| Issues surfaced | Specification gaps and tensions discovered during the trace |

### 1.4 PositionProjection excerpt alignment note

The JSON excerpts in §3.2, §4.2, and §5.2 use simplified field names for readability. Before these excerpts are used for compiler testing, they must be aligned to the authoritative 10.2 schema. Key deviations: excerpts use `positionLabel` (10.2 uses `positionTitle`), `primaryViews` (10.2 uses `workSurface.views`), `actionSpecs` (10.2 uses `actions`), `approvalSpecs` (10.2 uses `approvals`). This alignment is a pre-condition for compiler integration and is tracked as a compiler handoff item.

---

## 2. Shared company context

**Company:** Apex Components Ltd.  
**Industry:** Small manufacturing — precision metal components  
**Employees:** 45 across procurement, operations, warehouse, and finance  
**Foundry deployment:** Single tenant, SaaS  

**Positions exercised:**

| Position ID | Title | Primary work surface |
|---|---|---|
| `pos_procurement_coordinator` | Procurement Coordinator | Purchase Order management |
| `pos_finance_manager` | Finance Manager | PO approvals, spend oversight |
| `pos_warehouse_manager` | Warehouse Manager | Inventory monitoring, exception resolution |

---

## 3. Workflow 1 — Purchase Order Creation

### 3.1 Context

**Position:** Procurement Coordinator (`pos_procurement_coordinator`)  
**Business purpose:** Create a new Purchase Order for raw materials and submit it for Finance approval.  
**Trigger:** A low-stock alert surfaced in the Warehouse Manager's OverviewMonitor is communicated to the Procurement Coordinator. The coordinator navigates to the PO list and initiates creation.  
**Expected end state:** PO record created in "Pending Approval" state. Finance Manager receives an approval task. Coordinator lands on the PO RecordPage confirming the submission.

---

### 3.2 PositionProjection excerpt

The relevant schema slice for `pos_procurement_coordinator` that this workflow exercises:

```json
{
  "positionIdentity": {
    "positionId": "pos_procurement_coordinator",
    "positionLabel": "Procurement Coordinator",
    "tenantId": "apex_components"
  },
  "workSurfaceComposition": {
    "primaryViews": [
      {
        "viewId": "view_po_list",
        "patternType": "CollectionView",
        "entityTypeRef": {
          "entityType": "PurchaseOrder",
          "displayNameTemplate": "PO-{poNumber}",
          "viewLink": "/pos/procurement/po/{id}"
        },
        "defaultSortField": "createdAt",
        "defaultSortDirection": "desc"
      },
      {
        "viewId": "view_po_create",
        "patternType": "CreateEditFlow",
        "entityTypeRef": { "entityType": "PurchaseOrder" },
        "layoutHints": {
          "formLayout": "stepped_multi_section",
          "groupedSections": [
            { "sectionKey": "supplier_material",  "sectionLabel": "Supplier & Material",   "stepIndex": 1 },
            { "sectionKey": "delivery_terms",      "sectionLabel": "Delivery & Terms",      "stepIndex": 2 },
            { "sectionKey": "supporting_documents","sectionLabel": "Supporting Documents",  "stepIndex": 3 }
          ]
        }
      }
    ]
  },
  "actionSpecs": [
    {
      "actionId": "action_create_po",
      "actionType": "create",
      "label": "Create PO",
      "targetEntityType": "PurchaseOrder",
      "confirmationType": "none",
      "isDestructive": false
    },
    {
      "actionId": "action_submit_po",
      "actionType": "submit",
      "label": "Submit for Approval",
      "targetEntityType": "PurchaseOrder",
      "confirmationType": "inline",
      "preconditions": [
        {
          "conditionId": "pc_po_valid",
          "description": "All required fields must be filled and valid",
          "displayWhenNotMet": "disable_with_tooltip"
        }
      ]
    },
    {
      "actionId": "action_save_draft",
      "actionType": "save",
      "label": "Save Draft",
      "confirmationType": "none"
    },
    {
      "actionId": "action_cancel_create",
      "actionType": "cancel",
      "label": "Cancel",
      "confirmationType": "inline"
    }
  ],
  "approvalSpecs": [
    {
      "approvalType": "po_finance_approval",
      "label": "Finance Approval",
      "approvalDeadlineHours": 48,
      "approvalHistoryVisible": true
    }
  ],
  "personalizationHooks": {
    "allowedSurfaces": ["key_fact_count", "autosave"],
    "conflictPolicy": "role_overrides_user"
  }
}
```

---

### 3.3 Pattern selection trace

The compiler evaluates 10.3 §2.1 rules in priority order:

| Step | Rule applied | Result |
|---|---|---|
| Landing view | `workSurfaceComposition.primaryViews[0].patternType = CollectionView` | `view_po_list` → CollectionView (P1) |
| Create action trigger | `action_create_po` has `actionType=create` | Check flow assignment rules |
| Flow rule | `layoutHints.formLayout=stepped_multi_section` + 3 grouped sections | CreateEditFlow (F1) selected — ActionSpec with requiredInputs spanning more than one section |
| Review step | Multi-step flow default | ReviewSummary step auto-injected as final step |
| Post-flow landing | `entityTypeRef.viewLink` resolves to RecordPage | RecordPage (P2) rendered after flow exit |

**Patterns selected:** CollectionView (entry) → CreateEditFlow (workflow) → RecordPage (post-creation landing)

---

### 3.4 Screen sequence

#### Screen 1 — PO List (CollectionView)

**Pattern:** CollectionView (P1)  
**Entry state:** Populated with existing POs. No filters active. No items selected.

| Slot | Component | Configuration |
|---|---|---|
| `page-header / action-group` | ActionPanel (C5) | `context=page`. Primary: "Create PO" (action_create_po). Secondary: "Export", "Archive" (2 of 3 max). No overflow needed. |
| `filter-sidebar` | FilterRail (C6) | SearchBar first (S1). FilterGroups: status (isDefaultVisible=true), supplier (isDefaultVisible=true), date range (isDefaultVisible=false — behind expansion). ActiveFilterIndicator absent (idle state). |
| `record-list` | RecordRow[] | Each row: KeyFactsStrip (C2) — PO number, supplier, amount, status (4 facts; `key_fact_count=4` role variant). Row action: "View". |
| `empty-state` | EmptyState (C12) | `context=no_records`. Message: "No purchase orders yet." Action: "Create your first PO" (links to action_create_po). |
| `bulk-action-bar` | BulkActionBar (C7) | Inactive state — zero items selected. Replaces ActionPanel on first selection. Bulk actions: "Export selected", "Archive selected". |

**User action:** Clicks "Create PO" in ActionPanel.  
**Transition:** CreateEditFlow mounts. CollectionView is suspended (not destroyed — Back navigation will return to it).

---

#### Screen 2 — CreateEditFlow: Step 1 of 3 — Supplier & Material

**Pattern:** CreateEditFlow (F1)  
**Step:** 1 of 3 (Review is step 4)

| Slot | Component | Configuration |
|---|---|---|
| `flow-header` | FlowHeader (S5) | flowTitle: "Create Purchase Order". currentStep: 1. totalSteps: 3. cancelAction: action_cancel_create (confirmationType=inline). |
| `step-navigator` | StepNavigator (S6) | Steps: ["Supplier & Material", "Delivery & Terms", "Supporting Documents", "Review"]. currentStepIndex: 0. `section_navigator_style=sidebar_list` (default). |
| `form-body` | SmartFormSection (C3) | sectionKey=supplier_material. Fields: supplier (select, required), material_code (select, required), quantity (number, required), unit_price (currency, required), notes (text, optional). isCollapsedByDefault=false. autosaveEnabled=true (tenant structural variant). |
| `validation-area` | ValidationSummary (C13) | Hidden (`state=hidden`) — no errors yet. |
| `action-group` | ActionPanel (C5) | `context=flow-step`. Primary: "Next" (advances to step 2; disabled until section valid). Secondary: "Save Draft" (action_save_draft). No "Submit" — not the final step. |

**Autosave behavior:** `autosave=enabled` structural variant active at tenant scope (10.4 §4.7). DraftSaveIndicator node is active. On each field blur, a debounced autosave fires. `aria-live="polite"` region cycles: "Saving…" → "Saved". The save is unobtrusive — it does not interrupt the form.

**Validation path:** User fills supplier and material, leaves quantity blank, clicks "Next". Precondition `pc_po_valid` not met → "Next" is `disabled_with_tooltip`: "Complete all required fields to continue." ValidationSummary appears at section level with one error. Inline error on quantity field: `aria-describedby` links to the error message. User fills quantity → inline error clears → ValidationSummary hides → "Next" enabled.

---

#### Screen 3 — CreateEditFlow: Step 2 of 3 — Delivery & Terms

**Pattern:** CreateEditFlow (F1)  
**Step:** 2 of 3

| Slot | Component | Configuration |
|---|---|---|
| `flow-header` | FlowHeader (S5) | currentStep: 2. totalSteps: 3. Cancel still available. |
| `step-navigator` | StepNavigator (S6) | Step 1 shows completion indicator. Step 2 active (`aria-current="step"`). |
| `form-body` | SmartFormSection (C3) | sectionKey=delivery_terms. Fields: delivery_date (date picker, required), delivery_address (text, required), payment_terms (select, optional). isCollapsedByDefault=false. autosaveEnabled=true. |
| `action-group` | ActionPanel (C5) | Primary: "Next". Secondary: "Back" (returns to Step 1), "Save Draft". |

**Back navigation:** Step 1 values are preserved — autosave guarantees no data loss. DraftSaveIndicator shows "Saved" state before navigation occurs.

---

#### Screen 4 — CreateEditFlow: Step 3 of 3 — Supporting Documents

**Pattern:** CreateEditFlow (F1)  
**Step:** 3 of 3

| Slot | Component | Configuration |
|---|---|---|
| `flow-header` | FlowHeader (S5) | currentStep: 3. totalSteps: 3. |
| `step-navigator` | StepNavigator (S6) | Steps 1 and 2 show completion. Step 3 active. |
| `form-body` | SmartFormSection (C3) | sectionKey=supporting_documents. isReadOnly=true (context-only section — shows step 1+2 summary). AttachmentPanel occupies the `additional-content` slot within this section. **GW-G3:** this slot is not yet defined in 10.5 §C3 for `isReadOnly=true` sections — tracked as spec amendment: add `additional-content` slot to C3, available when isReadOnly=true, accepting AttachmentPanel and ActivityTimeline. |
| `attachment-panel` | AttachmentPanel (C11) | allowUpload=true. maxFiles=5 (tenant config). acceptedTypes: [pdf, xlsx, png]. Empty state: "No documents attached" with upload affordance. |
| `action-group` | ActionPanel (C5) | Primary: "Next → Review". Secondary: "Back", "Save Draft". |

**maxFiles behavior:** If 5 files are already attached, AttachmentPanel enters `at_limit` state. Upload area is rendered disabled with tooltip: "Maximum 5 attachments reached. Remove a file to add another." Upload area is not hidden (UX Doctrine A6: user must understand the limit and its prerequisite).

---

#### Screen 5 — CreateEditFlow: Review Step

**Pattern:** CreateEditFlow (F1 — review step, auto-injected for all multi-step flows)

| Slot | Component | Configuration |
|---|---|---|
| `flow-header` | FlowHeader (S5) | currentStep: 4 (Review). totalSteps: 3 + Review. |
| `step-navigator` | StepNavigator (S6) | All 3 prior steps show completion. Review highlighted. |
| `review-body` | ReviewSummary (S7) | Sections: Supplier & Material, Delivery & Terms, Supporting Documents. Each section has an edit link that navigates back to its step. Read-only display — no inline editing in Review. |
| `action-group` | ActionPanel (C5) | Primary: "Submit for Approval" (action_submit_po; `confirmationType=inline`; disabled until `pc_po_valid` passes). Secondary: "Back", "Save Draft". |

**Submit inline confirmation:** Renders below the primary button: "Submit PO for Finance approval? This cannot be undone once confirmed." Controls: "Confirm" | "Cancel". On Confirm: PO transitions to Pending Approval. Flow exits. User navigates to PO RecordPage.

---

#### Screen 6 — PO RecordPage (post-creation landing)

**Pattern:** RecordPage (P2)  
**Purpose:** Confirm the submission; show the PO in its live state.

| Slot | Component | Configuration |
|---|---|---|
| `page-header` | RecordHeader (C1) | displayName: "PO-1042". entityTypeBadgeLabel: "Purchase Order". statusLabel: "Pending Approval". statusSemantic: warning. KeyFactsStrip: supplier, amount, delivery date, submitted date (4 facts). StatusBadge (status variant) precedes ActionPanel in DOM order (P4). |
| `page-header / action-group` | ActionPanel (C5) | Primary: absent (PO is in approval — no create/edit action available). Secondary: "Withdraw Submission". Edit is hidden (not in allowedActionIds for pending-approval state). |
| `status-feedback` | StatusBanner (C4) | severity=success. message: "Purchase Order submitted for Finance approval." isDismissible=true. isPersistent=false. |
| `content / approval-section` | ApprovalPanel (C9) | currentState=pending. pendingApprovers: [Finance Manager]. deadlineLabel: "Due in 48 hours". eventHistory: [po_submitted_for_approval]. |
| `content / timeline` | ActivityTimeline (C8) | Events: [submitted_for_approval, draft_saved_x3, po_created]. maxVisible=5. isAIInitiated=false for all events. |
| `content / attachments` | AttachmentPanel (C11) | allowUpload=false (PO in approval — no upload permitted). Renders submitted attachments as view-only. |

---

### 3.5 Variant surfaces exercised

| Variant id | Type | Scope | Active in | Effect |
|---|---|---|---|---|
| `form_layout=stepped_multi_section` | Structural | Tenant | CreateEditFlow | StepNavigator IR node injected; groupedSections bound to step sequence |
| `autosave=enabled` | Structural | Tenant | SmartFormSection (all 3 steps) | DraftSaveIndicator node active; autosave_trigger action fires on field change |
| `key_fact_count=4` | Emphasis | Role | RecordRow (CollectionView), RecordHeader | 4 facts per row in PO list; 4 facts in RecordHeader |

---

### 3.6 Personalization walk

**Scenario:** The Procurement Coordinator has set a user-scope `filter_placement=chip_bar` preference (prefers the inline chip bar over the sidebar rail).

**Resolution path (10.4 §5.3):**

1. Compiler checks `personalizationHooks.allowedSurfaces` for `pos_procurement_coordinator`
2. `filter_placement` is **not listed** in `allowedSurfaces: ["key_fact_count", "autosave"]`
3. Surface is outside allowed scope — user preference cannot be applied
4. `filter_placement` falls back to role-scope default: `sidebar_rail`
5. conflictPolicy: `role_overrides_user` — no conflict needed, surface is simply not allowed

**Result:** FilterRail renders as sidebar rail. The chip-bar preference is silently discarded.  
**Orphan log (10.4 §7.1):** `{ positionId: "pos_procurement_coordinator", userId: "<id>", viewId: "view_po_list", surfaceId: "filter_placement", reason: "not_in_allowed_surfaces" }`. The orphaned value is not surfaced to the user.

---

### 3.7 Generator implications

| Task | Compiler action required |
|---|---|
| Landing pattern | Select CollectionView for `patternType=CollectionView` in primaryViews[0] |
| Create flow | On `action_create_po` trigger with `actionType=create` + groupedSections.length > 1 → select CreateEditFlow (F1) |
| Step binding | Map each `groupedSections` entry (ordered by stepIndex) to a SmartFormSection instance |
| StepNavigator | `form_layout=stepped_multi_section` active → inject StepNavigator IR node into CreateEditFlow |
| Review step | stepCount > 1 → auto-inject ReviewSummary as final step; bind from accumulated section data |
| Autosave | `autosave=enabled` structural variant → activate DraftSaveIndicator in all SmartFormSection instances; bind `autosave_trigger` action |
| Precondition enforcement | `action_submit_po.preconditions[0]` → ActionPanel renders "Submit" as disabled-with-tooltip until condition met |
| Post-flow navigation | On flow success → navigate to `entityTypeRef.viewLink`; pattern = RecordPage (P2) |
| ApprovalPanel binding | `approvalSpecs[0]` present → ApprovalPanel required on RecordPage; `approvalHistoryVisible=true` enforced |

---

### 3.8 Issues surfaced

| ID | Issue | Severity | Resolution path |
|---|---|---|---|
| W1-I1 | AttachmentPanel is inside a `isReadOnly=true` SmartFormSection in Step 3. The slot model in 10.5 §C3 does not explicitly declare an `additional-content` slot for interactive children within read-only sections. | Minor | Amend 10.5 §C3 SmartFormSection: add `additional-content` slot — available when `isReadOnly=true`, accepts interactive components (AttachmentPanel, ActivityTimeline). The read-only flag governs form fields, not slot content. |
| W1-I2 | ReviewSummary (S7) covers the Attachments step as a section with an edit link. Edit link navigates back to Step 3. But AttachmentPanel delete behavior in the context of ReviewSummary is undefined — can a user delete an attachment while in Review? | Minor | 10.5 §S7 ReviewSummary: edit links return to the step's SmartFormSection. Attachment management (delete, add) requires returning to Step 3. ReviewSummary does not render delete controls directly. |
| W1-I3 | "Back" action in the multi-step flow is not declared in `actionSpecs`. It is a flow-internal navigation action synthesized by the compiler. | Note | Back navigation is implicit in CreateEditFlow (F1). The compiler synthesizes back-navigation buttons between steps. No ActionSpec is required. Document this in 10.3 F1 as a compiler-synthesized flow action. |

---

## 4. Workflow 2 — Purchase Order Approval

### 4.1 Context

**Position:** Finance Manager (`pos_finance_manager`)  
**Business purpose:** Review a pending Purchase Order submitted for Finance approval; decide to approve or reject.  
**Trigger:** Finance Manager opens their approval queue (CollectionView pre-filtered to pending approvals). Selects PO-1042 submitted by the Procurement Coordinator.  
**Expected end states:**
- **Approve path:** PO transitions to "Approved". Coordinator is notified. Record becomes read-only.
- **Reject path:** PO transitions to "Rejected" with a mandatory rejection reason. Coordinator can revise and resubmit.

---

### 4.2 PositionProjection excerpt

```json
{
  "positionIdentity": {
    "positionId": "pos_finance_manager",
    "positionLabel": "Finance Manager",
    "tenantId": "apex_components"
  },
  "workSurfaceComposition": {
    "primaryViews": [
      {
        "viewId": "view_po_approvals",
        "patternType": "CollectionView",
        "entityTypeRef": {
          "entityType": "PurchaseOrder",
          "displayNameTemplate": "PO-{poNumber}",
          "viewLink": "/pos/finance/approvals/{id}"
        },
        "defaultFilterPreset": { "approval_state": "pending" }
      },
      {
        "viewId": "view_po_approval_detail",
        "patternType": "ApprovalReviewView",
        "entityTypeRef": { "entityType": "PurchaseOrder" }
      }
    ]
  },
  "actionSpecs": [
    {
      "actionId": "action_approve_po",
      "actionType": "approve",
      "label": "Approve",
      "confirmationType": "inline",
      "isDestructive": false
    },
    {
      "actionId": "action_reject_po",
      "actionType": "reject",
      "label": "Reject",
      "confirmationType": "full",
      "requiresComment": true,
      "isDestructive": false
    },
    {
      "actionId": "action_request_info",
      "actionType": "request_information",
      "label": "Request More Info",
      "confirmationType": "none"
    }
  ],
  "approvalSpecs": [
    {
      "approvalType": "po_finance_approval",
      "label": "Finance Approval",
      "approvalDeadlineHours": 48,
      "approvalHistoryVisible": true
    }
  ],
  "personalizationHooks": {
    "allowedSurfaces": ["key_fact_count", "approval_history_placement"],
    "conflictPolicy": "role_overrides_user"
  }
}
```

---

### 4.3 Pattern selection trace

| Step | Rule applied | Result |
|---|---|---|
| Landing view | `workSurfaceComposition.primaryViews[0].patternType = CollectionView` | `view_po_approvals` → CollectionView (P1), pre-filtered to pending approvals |
| Record selection | Finance Manager clicks a PO row | Navigate to `view_po_approval_detail` |
| Pattern selection | `patternType=ApprovalReviewView` + `approvalSpecs` non-empty | ApprovalReviewView (P6) selected |
| Approve action | `action_approve_po.confirmationType=inline` | Inline confirmation — no flow launched; decision taken on ApprovalReviewView |
| Reject action | `action_reject_po.confirmationType=full` + `requiresComment=true` | ApprovalFlow (F2) launched — full rejection step required before decision is committed |

**Patterns selected:** CollectionView (entry) → ApprovalReviewView (review) → ApprovalFlow (rejection path only)

---

### 4.4 Screen sequence

#### Screen 1 — PO Approvals Queue (CollectionView, pre-filtered)

**Pattern:** CollectionView (P1)  
**Pre-filter:** `defaultFilterPreset: { approval_state: "pending" }` — FilterRail loads with approval_state=pending pre-applied.

| Slot | Component | Configuration |
|---|---|---|
| `page-header / action-group` | ActionPanel (C5) | No primary action. Finance Manager does not create POs. Secondary: "Export". |
| `filter-sidebar` | FilterRail (C6) | SearchBar first (S1). Pre-filter: approval_state=pending active. ActiveFilterIndicator shows "1 filter". Additional filters: supplier, amount range, submitted date. Clear-all visible. |
| `record-list` | RecordRow[] | Each row: KeyFactsStrip (C2) — PO number, supplier, amount, submitted date (4 facts). Row action: "Review". |
| `empty-state` | EmptyState (C12) | `context=no_records`. Message: "No pending approvals — all caught up." Positive framing, no action path (Finance Manager has no create action). |

**User action:** Clicks "Review" on PO-1042 row.  
**Transition:** Navigate to `view_po_approval_detail` → ApprovalReviewView mounts.

---

#### Screen 2 — PO ApprovalReviewView

**Pattern:** ApprovalReviewView (P6)

| Slot | Component | Configuration |
|---|---|---|
| `page-header` | RecordHeader (C1) | displayName: "PO-1042". entityTypeBadgeLabel: "Purchase Order". statusLabel: "Pending Approval". statusSemantic: warning. KeyFactsStrip: supplier, amount, delivery date, initiator, submitted date (5 facts — `key_fact_count=5` role variant). StatusBadge (status) precedes ActionPanel in DOM order (P4, XP6, XC5). |
| `page-header / action-group` | ActionPanel (C5) | `context=page`. Primary: "Approve" (action_approve_po). Secondary: "Reject" (action_reject_po), "Request More Info" (action_request_info). Budget: 1+2 = 3 total — no overflow. **GW-G1:** action_request_info has no defined UI outcome — seam to CommentConversationPanel is an open amendment to 10.3 P6 and 10.5 §C15. |
| `content / approval-section` | ApprovalPanel (C9) | currentState=pending. pendingApprovers: [Finance Manager]. deadlineLabel: "Due in 32 hours". eventHistory: [submitted_by_coordinator]. Placement: `approval_history_placement=collapsible_section` (role variant — history collapsed by default; Finance Manager reviews many records and does not need to see history on every view load). |
| `content / timeline` | ActivityTimeline (C8) | Events: [submitted_for_approval, draft_saved_x3, po_created]. maxVisible=5. Placement: `timeline_placement=inline_below_sections`. |
| `content / attachments` | AttachmentPanel (C11) | allowUpload=false (Finance Manager cannot add attachments). Renders submitted quote attachment as view-only. |

**State invariant check:** RecordHeader `status-badge` slot renders before `action-group` slot in both DOM order and visual order. This is XC5 and must survive any token or variant change.

---

#### Screen 3a — Approve Path (inline confirmation on ApprovalReviewView)

**Trigger:** Finance Manager clicks "Approve" (primary action; `confirmationType=inline`).

**Inline confirmation** renders immediately below the primary button:  
*"Approve PO-1042 for £12,400? This will authorize the order."*  
Controls: "Confirm Approval" | "Cancel"

**On Confirm:**  
- PO state: Pending Approval → **Approved**  
- RecordHeader: statusLabel: "Approved", statusSemantic: positive  
- ApprovalPanel: currentState=approved; pendingApprovers section absent; history: [submitted, approved_by_finance_manager]  
- ActivityTimeline: new event [approved_by_finance_manager]  
- StatusBanner: severity=success, "Purchase Order approved. The Procurement Coordinator has been notified.", isDismissible=true  
- ActionPanel: "Approve" removed (already approved); "Reject" removed; read-only state on record  

**On Cancel:** Inline confirmation collapses. ApprovalReviewView returns to its default state. No state change.

---

#### Screen 3b — Reject Path (ApprovalFlow)

**Trigger:** Finance Manager clicks "Reject" (secondary action; `confirmationType=full`).

**Flow launch:** `confirmationType=full` triggers ApprovalFlow (F2) — rejection step must be completed before the decision is committed. This prevents accidental rejection without a reason.

**ApprovalFlow — Rejection Reason Step:**

| Slot | Component | Configuration |
|---|---|---|
| `flow-header` | FlowHeader (S5) | flowTitle: "Reject Purchase Order". currentStep: 1. totalSteps: 1. cancelAction: exits without rejecting — returns to ApprovalReviewView. |
| `context-panel` | KeyFactsStrip (C2) | PO reference: supplier, amount (read-only context; not part of the rejection form). |
| `form-body` | SmartFormSection (C3) | sectionKey=rejection_reason. sectionLabel: "Reason for rejection". sectionDescription: "Your reason will be visible to the submitter and stored in the approval record." Fields: rejection_reason (text area, required, min 10 chars). isReadOnly=false. autosaveEnabled=false. |
| `action-group` | ActionPanel (C5) | `context=flow-step`. Primary: "Confirm Rejection" (disabled until rejection_reason filled). Secondary: "Cancel" (exits without rejecting). Confirmation is already the action — no second confirmation step needed. |

**On Confirm Rejection:**  
- ApprovalFlow exits  
- User returns to ApprovalReviewView  
- PO state: Pending Approval → **Rejected**  
- RecordHeader: statusLabel: "Rejected", statusSemantic: critical  
- ApprovalPanel: currentState=rejected; rejectionReason displayed prominently (not collapsed, not hidden behind expansion — per 10.5 §C9 composition rule)  
- StatusBanner: severity=warning, "Purchase Order rejected. The Procurement Coordinator has been notified.", isDismissible=true  

**Two-banner scenario:** If the deadline-approaching warning StatusBanner was already visible before rejection ("Approval deadline in 4 hours"), two banners now coexist. Stacking order (10.5 §C4): warning above informational → warning (deadline) stacks above warning (rejection confirmed) → error first if any error were present. In this case: both are `warning` severity — the system-generated deadline warning appears above the action-feedback warning. 8px gap between banners.

---

### 4.5 Variant surfaces exercised

| Variant id | Type | Scope | Active in | Effect |
|---|---|---|---|---|
| `key_fact_count=5` | Emphasis | Role | RecordHeader (ApprovalReviewView) | 5 facts: supplier, amount, delivery date, initiator, submitted date |
| `approval_history_placement=collapsible_section` | Emphasis | Role | ApprovalPanel | Event history collapsed by default; Finance Manager can expand on demand |

---

### 4.6 Personalization walk

**Scenario:** Finance Manager has a user-scope preference `key_fact_count=3` ("I only need supplier and amount when reviewing — the rest is in the details below").

**Resolution path (10.4 §5.3):**

1. `key_fact_count` is in `allowedSurfaces` for `pos_finance_manager` — eligible surface
2. User-scope value exists: `3`
3. Role-scope value exists: `5`
4. conflictPolicy: `role_overrides_user`
5. **Role value wins (5)** — user preference is stored but not applied

**Result:** Finance Manager sees 5 facts despite their 3-fact preference. The preference is stored in the preference store but the effective value is role-scope.

**Illustrative counterfactual:** If conflictPolicy were `user_overrides_role` (not this tenant's config), the user's value of 3 would apply. KeyFactsStrip would truncate to 3 facts (supplier, amount, delivery date — the first 3 in the `keyFacts` array). `isHighlighted=true` on supplier and amount would still be respected, but facts 4 and 5 would be silently omitted (overflow rule in 10.5 §C2 — silent truncation, no "show more").

---

### 4.7 Generator implications

| Task | Compiler action required |
|---|---|
| Pattern selection | `patternType=ApprovalReviewView` + `approvalSpecs` non-empty → select P6 |
| ApprovalPanel binding | Bind to `approvalSpecs[0]`; `approvalHistoryVisible=true` → ApprovalPanel required |
| Action binding | Derive ActionPanel contents from `actionSpecs`; `actionType=approve` → primary; `actionType=reject` + `actionType=request_information` → secondary |
| Approve flow | `confirmationType=inline` → inline confirmation rendered within ApprovalReviewView; no flow launched |
| Reject flow | `confirmationType=full` + `requiresComment=true` → launch ApprovalFlow (F2); inject SmartFormSection for rejection reason |
| ApprovalFlow step | F2 is a single-step flow for rejection; StepNavigator not present (stepCount=1); FlowHeader present |
| Approval history variant | `approval_history_placement=collapsible_section` → ApprovalPanel IR node gains collapsible wrapper; default: collapsed |
| Pre-filter | `defaultFilterPreset: { approval_state: "pending" }` → FilterRail initializes with active filter; ActiveFilterIndicator visible on mount |

---

### 4.8 Issues surfaced

| ID | Issue | Severity | Resolution path |
|---|---|---|---|
| W2-I1 | ApprovalReviewView has no defined behavior when the current user is not in `pendingApprovers` (e.g., Finance Manager views a PO routed to a different approver). | Minor | 10.3 P6: if current user is not an active approver, ActionPanel hides "Approve" and "Reject" (not present in allowedActionIds); view renders as read-only RecordPage variant. |
| W2-I2 | `action_request_information` (`confirmationType=none`) has no defined UI outcome — no seam to CommentConversationPanel, no task creation, no notification. The action is declared but the response surface is undefined. | Minor | This is grammar gap GW-G1 (cross-workflow). Resolution: add to 10.3 P6: `actionType=request_information` triggers CommentConversationPanel to mount in compose state. Document in 10.5 §C15 open seams. |
| W2-I3 | After rejection, `action_approve_po` remains in ActionPanel (Finance Manager could immediately approve a record they just rejected). This is valid business behavior, but the sequence requires a second confirmationType=inline approval after a completed rejection, which feels ambiguous. | Minor | Action semantics are outside UI workstream scope — governed by ApprovalSpec.stateModel. Log as a compiler handoff note: if stateModel permits approve-after-reject, the UI presents the action unchanged. |

---

## 5. Workflow 3 — Low-Stock Exception Resolution

### 5.1 Context

**Position:** Warehouse Manager (`pos_warehouse_manager`)  
**Business purpose:** Monitor inventory levels, identify critical low-stock exceptions, investigate the affected SKU, and initiate a reorder to resolve the exception.  
**Trigger:** Warehouse Manager opens their position app. OverviewMonitor is the landing view (priority 1 landing rule: StateSummarySpec + AlertSpec with critical severity). A Critical alert reads: "Steel Sheet A-42: stock 23% below minimum threshold."  
**Expected end state:** Reorder PO created in Pending Approval state. Alert on Steel Sheet A-42 transitions to resolved. OverviewMonitor no longer shows a critical alert for this SKU.

---

### 5.2 PositionProjection excerpt

```json
{
  "positionIdentity": {
    "positionId": "pos_warehouse_manager",
    "positionLabel": "Warehouse Manager",
    "tenantId": "apex_components"
  },
  "workSurfaceComposition": {
    "primaryViews": [
      {
        "viewId": "view_inventory_monitor",
        "patternType": "OverviewMonitor",
        "summarySpecs": [
          {
            "summaryId": "sum_total_skus",
            "summaryType": "count",
            "label": "Total Active SKUs",
            "linkedViewId": "view_sku_list"
          },
          {
            "summaryId": "sum_below_reorder",
            "summaryType": "count",
            "label": "Below Reorder Point",
            "linkedViewId": "view_sku_exceptions",
            "alertThreshold": 1
          },
          {
            "summaryId": "sum_pending_reorders",
            "summaryType": "count",
            "label": "Pending Reorder Approvals",
            "linkedViewId": "view_reorder_queue"
          }
        ],
        "alertSpecs": [
          {
            "alertSpecId": "alert_low_stock",
            "severity": "critical",
            "label": "Low Stock Alert",
            "entityType": "InventorySKU",
            "linkedViewId": "view_sku_exception_detail",
            "triggerCondition": "current_stock < minimum_threshold"
          },
          {
            "alertSpecId": "alert_expiring_stock",
            "severity": "warning",
            "label": "Expiring Stock",
            "entityType": "InventorySKU",
            "linkedViewId": "view_sku_exception_detail"
          }
        ],
        "taskSpecs": [
          {
            "taskSpecId": "task_reorder_approval_watch",
            "taskType": "PendingReorderApproval",
            "label": "Pending Reorder Approvals",
            "stateModel": ["pending", "approved", "rejected"],
            "priorityLevels": ["normal", "urgent"],
            "hasDueDate": false,
            "availableActionIds": ["action_view_reorder_queue"]
          }
        ]
      },
      {
        "viewId": "view_sku_exception_detail",
        "patternType": "ExceptionResolutionView",
        "entityTypeRef": {
          "entityType": "InventorySKU",
          "displayNameTemplate": "{skuCode} — {description}",
          "viewLink": "/pos/warehouse/skus/{id}/exceptions"
        }
      }
    ]
  },
  "actionSpecs": [
    {
      "actionId": "action_initiate_reorder",
      "actionType": "create",
      "label": "Initiate Reorder",
      "targetEntityType": "PurchaseOrder",
      "confirmationType": "none"
    },
    {
      "actionId": "action_adjust_threshold",
      "actionType": "update",
      "label": "Adjust Threshold",
      "targetEntityType": "InventorySKU",
      "confirmationType": "inline"
    },
    {
      "actionId": "action_dismiss_alert",
      "actionType": "dismiss",
      "label": "Dismiss Alert",
      "confirmationType": "inline",
      "isDestructive": false
    }
  ],
  "personalizationHooks": {
    "allowedSurfaces": ["summary_strip_layout", "timeline_placement"],
    "conflictPolicy": "user_overrides_role"
  }
}
```

---

### 5.3 Pattern selection trace

| Step | Rule applied | Result |
|---|---|---|
| Landing view | Priority 1 landing rule: `StateSummarySpec` present + `alertSpecs` with critical severity ≥ 1 | `view_inventory_monitor` → OverviewMonitor (P3) |
| Alert click | User clicks Critical alert "Steel Sheet A-42" | Navigate to `view_sku_exception_detail` (linkedViewId from alertSpec) |
| Pattern selection | `patternType=ExceptionResolutionView` + alert linkage | ExceptionResolutionView (P5) selected |
| Resolution action | `action_initiate_reorder` with `actionType=create` + requiredInputs | ExceptionResolutionFlow (F3) launched |

**Patterns selected:** OverviewMonitor (landing) → ExceptionResolutionView (investigation) → ExceptionResolutionFlow (resolution)

---

### 5.4 Screen sequence

#### Screen 1 — Inventory Monitor (OverviewMonitor)

**Pattern:** OverviewMonitor (P3)  
**Entry state:** 3 active Critical alerts; 2 Warning alerts; TaskContextPanel has 3 pending reorder approvals.

| Slot | Component | Configuration |
|---|---|---|
| `summary-strip` | MetricTile (S3) × 3 | sum_total_skus (142, threshold not crossed), sum_below_reorder (7, alertThresholdCrossed=true → alert indicator active), sum_pending_reorders (3). Layout: `summary_strip_layout=card_grid` user variant → SummaryCardGrid IR node (tiles in 3-column card grid instead of horizontal strip). |
| `alert-list` | AlertList (S4) | Alert ordering: Critical (3 items) → Warning (2 items) → Informational (0). "STEEL-A42 — Below Minimum Threshold" is the first Critical item. EmptyState if no alerts: "No active alerts — all operational." (context=positive_confirmation — not blank). |
| `tasks-section` | TaskContextPanel (C14) | taskSpec=task_reorder_approval_watch. taskInstance: 3 pending instances (aggregate). State: active. ActionPanel: "View Reorder Queue" (action_view_reorder_queue). **GW-G2 aggregate mode:** only `active` and `no_task_instance` states are valid here — individual states (`blocked`, `overdue`) do not apply to aggregate counts. Amendment to 10.5 §C14. |
| `page-header / action-group` | ActionPanel (C5) | No primary action on OverviewMonitor. |

**Alert ordering invariant (10.5 §S4):** Critical → Warning → Informational ordering is invariant. The compiler must not expose this ordering as a personalization surface. No variant may change it.

**TaskContextPanel null case:** If no active TaskInstance exists for `task_reorder_approval_watch` (no pending reorders), component renders `no_task_instance` state: "No pending reorder approvals" with no action path. This is a passive EmptyState — no call to action.

---

#### Screen 2 — SKU Exception Detail (ExceptionResolutionView)

**Pattern:** ExceptionResolutionView (P5)  
**Navigated from:** Alert click on "STEEL-A42" in AlertList.

| Slot | Component | Configuration |
|---|---|---|
| `page-header` | RecordHeader (C1) | displayName: "STEEL-A42 — Cold Rolled Steel Sheet A-42". entityTypeBadgeLabel: "Inventory SKU". statusLabel: "Critical — Below Threshold". statusSemantic: critical. KeyFactsStrip: current stock (23 units), minimum threshold (100 units), reorder point (150 units), last restock date, primary supplier (5 facts — within role key_fact_count range). |
| `page-header / action-group` | ActionPanel (C5) | `context=page`. Primary: "Initiate Reorder" (action_initiate_reorder). Secondary: "Adjust Threshold" (action_adjust_threshold), "Dismiss Alert" (action_dismiss_alert). Budget: 1 primary + 2 secondary = 3 total. **GW-G5:** on "Dismiss Alert" confirm, navigate to parent OverviewMonitor (`view_inventory_monitor`). Amendment to 10.3 P5. |
| `content / status-banner` | StatusBanner (C4) | severity=error. message: "Current stock (23 units) is 77% below minimum threshold. Immediate reorder recommended." isPersistent=true. isDismissible=false. (System-generated; condition still active.) |
| `content / timeline` | ActivityTimeline (C8) | Events: [stock_alert_triggered_today, stock_updated_yesterday, previous_reorder_received (3 months ago), prior_alert_dismissed]. maxVisible=5. isAIInitiated=false. Placement: `timeline_placement=inline_below_sections` (default; user has not set a preference). |

**No EmptyState rendered:** ExceptionResolutionView is always navigated to from an active alert. Data is always present. EmptyState (C12) is not applicable at page level here.

---

#### Screen 3 — ExceptionResolutionFlow: Step 1 of 2 — Reorder Details

**Pattern:** ExceptionResolutionFlow (F3)  
**Triggered by:** "Initiate Reorder" primary action (`actionType=create`; `confirmationType=none` — no pre-confirmation).

| Slot | Component | Configuration |
|---|---|---|
| `flow-header` | FlowHeader (S5) | flowTitle: "Initiate Reorder — STEEL-A42". currentStep: 1. totalSteps: 2. cancelAction: exits to ExceptionResolutionView (no draft saved — `autosave=disabled` at role scope for Warehouse Manager in this flow). |
| `context-panel` | KeyFactsStrip (C2) | Read-only context: current stock (23), minimum threshold (100), reorder point (150). `orientation=horizontal`. Reminds the user what they are solving. |
| `form-body` | SmartFormSection (C3) | sectionKey=reorder_details. Fields: reorder_quantity (number, required, hint: "Recommended: 200 units based on reorder point"), preferred_supplier (select, pre-filled from SKU's `last_reorder_supplier` attribute), urgency (select: normal/urgent, required). isReadOnly=false. autosaveEnabled=false. |
| `action-group` | ActionPanel (C5) | Primary: "Next → Review" (disabled until reorder_details section valid). Secondary: "Cancel". |

**Pre-fill:** `preferred_supplier` is pre-filled from the SKU's last reorder supplier. This is a schema-level field default — not an AI suggestion. No `isAIGenerated` flag, no accept step.

---

#### Screen 4 — ExceptionResolutionFlow: Step 2 of 2 — Review & Submit

**Pattern:** ExceptionResolutionFlow (F3 — review step)

| Slot | Component | Configuration |
|---|---|---|
| `flow-header` | FlowHeader (S5) | currentStep: 2. totalSteps: 2. (Review.) |
| `review-body` | ReviewSummary (S7) | Section: Reorder Details — reorder_quantity, preferred_supplier, urgency. Edit link navigates back to Step 1. |
| `action-group` | ActionPanel (C5) | Primary: "Submit Reorder" (action_initiate_reorder; confirmationType=none — no second confirmation). Secondary: "Back", "Cancel". |

**On Submit:**  
- New PO created in "Pending Approval" state; Finance Manager receives approval task  
- Alert "Steel Sheet A-42" transitions to resolved state  
- User navigates back to ExceptionResolutionView  
- StatusBanner state update: the persistent `severity=error` banner is replaced by the system with `severity=warning`: "Reorder pending Finance approval. Stock remains below threshold until delivery." isPersistent=true  
- Additional StatusBanner: `severity=success`, "Reorder initiated. PO-1043 is pending Finance approval.", isDismissible=true  

**Two-banner coexistence (post-submit):** warning (system condition, persistent) + success (action feedback, dismissible).  
Stacking order (10.5 §C4): warning above success. 8px gap. Warning banner is nearest the top.

**System-replace rule (GW-G4 — amendment to 10.5 §C4):** The persistent error banner is not dismissed and re-added — it is replaced in-place by a system event (new severity: warning, new message). The replace operation preserves the banner's stacking position and does not require user action. This is distinct from user dismissal. Amendment required: add to 10.5 §C4 composition rules — a `isPersistent=true` banner may be updated in-place by a system event with a different severity or message.

---

### 5.5 Variant surfaces exercised

| Variant id | Type | Scope | Active in | Effect |
|---|---|---|---|---|
| `summary_strip_layout=card_grid` | Structural | User | OverviewMonitor SummaryStrip | IR transformation: SummaryStrip → SummaryCardGrid (10.4 §4.11); MetricTile instances in 3-column card grid |
| `timeline_placement=inline_below_sections` | Emphasis | Role (default) | ExceptionResolutionView | ActivityTimeline renders inline below content sections — default; user has not set a preference |

**Structural variant governance:** `summary_strip_layout=card_grid` is a user-scope structural variant — it requires tenant-scope pre-approval before a user can select it (10.4 §5.3). In this projection, Apex's tenant has pre-approved this value. The full governance path trace, including the failure path, is in §6.4.

---

### 5.6 Personalization walk

**Scenario:** Warehouse Manager sets user-scope `timeline_placement=tab` preference on ExceptionResolutionView.

**Resolution path (10.4 §5.3):**

1. `timeline_placement` is in `allowedSurfaces` for `pos_warehouse_manager` — eligible
2. User-scope value exists: `tab`
3. Role-scope default: `inline_below_sections`
4. conflictPolicy: `user_overrides_role`
5. **User value wins (`tab`)**

**Result:** ActivityTimeline on ExceptionResolutionView renders as a tab (not inline). The tab appears in the RecordPage tab bar alongside the main content section. The content area shows the main exception details; the user clicks the "Activity" tab to see the timeline.

**Constraint check (10.5 §C8):** `timeline_placement=tab` is valid only for RecordPage and ExceptionResolutionView patterns — not for OverviewMonitor. The compiler must validate this constraint at personalization application time.

---

### 5.7 Generator implications

| Task | Compiler action required |
|---|---|
| Landing pattern | Priority 1 landing rule: summarySpecs + alertSpecs critical → select OverviewMonitor (P3) |
| SummaryStrip | Compose 3 MetricTile instances from summarySpecs; apply `summary_strip_layout=card_grid` variant → SummaryCardGrid IR node |
| AlertList | Compose AlertList from alertSpecs; enforce Critical → Warning → Informational ordering as invariant |
| TaskContextPanel | Compose from taskSpecs[0]; if taskInstance is null → render `no_task_instance` state |
| ExceptionResolutionView | Navigate from alertSpec.linkedViewId; bind RecordHeader and KeyFactsStrip from InventorySKU entityTypeRef |
| Action binding | "Initiate Reorder" as primary (actionType=create); "Adjust Threshold" and "Dismiss Alert" as secondary |
| ExceptionResolutionFlow | `action_initiate_reorder` with multi-field required inputs → select F3; inject 1 SmartFormSection step + ReviewSummary |
| Post-submit StatusBanner | On flow success: system replaces persistent error banner with warning banner; adds dismissible success banner; apply stacking order rule |
| Structural variant governance | `summary_strip_layout=card_grid` user preference → check tenant-scope pre-approval before applying; fall back to `horizontal_strip` if not approved |

---

### 5.8 Issues surfaced

| ID | Issue | Severity | Resolution path |
|---|---|---|---|
| W3-I1 | After reorder submission, the persistent `severity=error` StatusBanner needs to transition to `severity=warning` (condition still active but mitigated). 10.5 §C4 StatusBanner defines only user-triggered dismissal — system-triggered replacement is not specified. | Minor | Add to 10.5 §C4 composition rules: a `isPersistent=true` banner may be replaced in-place by a system event (new severity, new message) without user action. This is a system state change, not dismissal. The replace operation must preserve the banner's position in the stacking order relative to other banners. This is grammar gap GW-G4. |
| W3-I2 | OverviewMonitor's TaskContextPanel shows an aggregate count of pending reorder approvals — not a single TaskInstance. The states `blocked` and `overdue` apply to individual TaskInstances, not to aggregate counts. The component state model has no "aggregate" mode. | Minor | Add to 10.5 §C14: when TaskContextPanel renders in OverviewMonitor with aggregate data (multiple task instances summarized as a count), only `active` and `no_task_instance` states are valid. Individual instance states (blocked, overdue) are not rendered for aggregate contexts. This is grammar gap GW-G2. |
| W3-I3 | `action_dismiss_alert` in ActionPanel could be triggered before the exception is resolved, leaving the ExceptionResolutionView without a triggering alert. Post-dismiss navigation is undefined in 10.3 P5. | Minor | This is grammar gap GW-G5. Add to 10.3 P5: "Dismiss Alert" action always carries `confirmationType=inline` minimum (already in this projection); on confirm, navigate to parent OverviewMonitor. |
| W3-I4 | `exception_list_mode=list_with_drill_in` structural variant (10.4 §4.4) is active at tenant scope in Apex's deployment but is not exercised by this single-alert trace. The split-pane layout where exception list and detail panel coexist is not traced. | Note | Deferred. A fourth mini-trace showing the split-view pattern would be needed to fully exercise this variant. Logged as a known gap in this v0 prototype set. |

---

## 6. Cross-workflow validation findings

### 6.1 Pattern coverage

| Pattern | Exercised in | Coverage notes |
|---|---|---|
| CollectionView (P1) | W1 (PO list), W2 (approvals queue) | Pre-filter exercised in W2; FilterRail pre-applied on mount |
| RecordPage (P2) | W1 (post-creation landing) | Used as a landing destination, not as an entry point |
| OverviewMonitor (P3) | W3 | SummaryStrip, AlertList, and TaskContextPanel all exercised |
| SearchResults (P4) | — | Not exercised. Gap. |
| ExceptionResolutionView (P5) | W3 | Fully exercised as main workflow entry |
| ApprovalReviewView (P6) | W2 | Both approve (inline) and reject (full flow) paths exercised |
| CreateEditFlow (F1) | W1 | Multi-step, autosave, review step all exercised |
| ApprovalFlow (F2) | W2 | Rejection path exercised; approve path uses inline confirmation (no flow launch) |
| ExceptionResolutionFlow (F3) | W3 | Single SmartFormSection step + auto-injected ReviewSummary exercised |
| AssistedSetupFlow (F4) | — | Not exercised. Gap. |

**Unexercised patterns:** SearchResults (P4) and AssistedSetupFlow (F4). Both are valid candidates for a W4 or W5 in a future 10.6 revision.

### 6.2 Component coverage

| Component | Exercised in |
|---|---|
| C1 RecordHeader | W1 (RecordPage), W2 (ApprovalReviewView), W3 (ExceptionResolutionView) |
| C2 KeyFactsStrip | W1 (row + header), W2 (header + rejection context), W3 (header + flow context) |
| C3 SmartFormSection | W1 (3 steps), W2 (rejection reason step), W3 (reorder details step) |
| C4 StatusBanner | W1 (success post-submit), W2 (approve/reject feedback + stacking), W3 (error + post-resolve stacking) |
| C5 ActionPanel | All three workflows; budget invariant held in all 8 ActionPanel instances traced |
| C6 FilterRail | W1 (CollectionView), W2 (pre-filtered CollectionView) |
| C7 BulkActionBar | W1 (declared on CollectionView; not triggered in this trace) |
| C8 ActivityTimeline | W2 (ApprovalReviewView), W3 (ExceptionResolutionView) |
| C9 ApprovalPanel | W2 (pending → approved/rejected transitions both traced) |
| C10 RelatedRecordsSection | — Not exercised |
| C11 AttachmentPanel | W1 (upload in step 3, at_limit state), W2 (read-only view) |
| C12 EmptyState | W1 (empty PO list), W2 (no pending approvals — positive), W3 (no-alerts OverviewMonitor — positive) |
| C13 ValidationSummary | W1 (create flow validation error path) |
| C14 TaskContextPanel | W3 (OverviewMonitor tasks section; aggregate mode + no_task_instance state) |
| C15 CommentConversationPanel | — Not exercised |
| S1 SearchBar | W1 and W2 FilterRail (always first in FilterRail) |
| S2 AIAssistPanel | — Not exercised |
| S3 MetricTile | W3 (OverviewMonitor SummaryStrip) |
| S4 AlertList | W3 (OverviewMonitor alert section) |
| S5 FlowHeader | W1 (all 4 CreateEditFlow steps), W2 (ApprovalFlow rejection step), W3 (ExceptionResolutionFlow both steps) |
| S6 StepNavigator | W1 (CreateEditFlow with form_layout=stepped_multi_section) |
| S7 ReviewSummary | W1 (PO create review), W3 (reorder review) |

**Unexercised:** C10 RelatedRecordsSection, C15 CommentConversationPanel, S2 AIAssistPanel. These would be exercised in workflows that involve related entity navigation, collaborative annotation, or AI-assisted form fill.

### 6.3 Action budget invariant check (UX Doctrine §6)

| Check | All instances verified? | Evidence |
|---|---|---|
| Max 1 primary action per ActionPanel | Yes | W1: "Create PO", "Next", "Submit for Approval". W2: "Approve". W3: "Initiate Reorder". Each has exactly 1 primary. |
| Max 3 secondary actions visible | Yes | Maximum visible secondary in any traced ActionPanel: 2 (W1 Review: "Back" + "Save Draft"; W2: "Reject" + "Request More Info"; W3: "Adjust Threshold" + "Dismiss Alert") |
| Destructive actions in destructive-zone | Checked | No `isDestructive=true` actions in these three workflows. "Dismiss Alert" and "Reject" are not data-destructive. |
| No two ActionPanels per context | Yes | Every context (screen, flow step, component) has exactly one ActionPanel. |
| Total ≤ 5 without overflow | Yes | Max 3 per context in all traces. No overflow menus needed. |

### 6.4 Personalization model validation

| Behavior tested | Validated in |
|---|---|
| Allowed surfaces restriction (surface not in allowedSurfaces → discarded) | W1 §3.6 — `filter_placement` not in allowed surfaces |
| `role_overrides_user` conflict policy | W2 §4.6 — `key_fact_count` user value overridden by role value |
| `user_overrides_role` conflict policy | W3 §5.6 — `timeline_placement` user value wins |
| Structural variant governance (tenant pre-approval required) | W3 §5.5 + full trace below — `summary_strip_layout=card_grid` exercises the pre-approval gate |
| Orphan preference logging | W1 §3.6 — orphaned `filter_placement` preference logged with 5-field record |
| Variant → IR transformation | W3 §5.5 — `summary_strip_layout=card_grid` → SummaryStrip node replaced with SummaryCardGrid |

#### Structural variant governance — full path trace

This is one of the most critical governance behaviors in 10.4: a user-scope structural variant selection must pass a tenant-scope pre-approval gate before it can be applied. This gate is independent of and precedes conflictPolicy evaluation.

**Success path (W3 scenario):**

1. Warehouse Manager sets user-scope preference: `summary_strip_layout=card_grid`
2. `summary_strip_layout` is in `allowedSurfaces` for `pos_warehouse_manager` — eligible surface
3. Variant type check: `card_grid` is a structural variant value (not token-driven, not emphasis)
4. Governance gate: is `card_grid` listed in the tenant-scope approved structural variants for `summary_strip_layout`?
5. Apex Components Ltd. tenant has pre-approved this value → gate passes
6. conflictPolicy=`user_overrides_role` → user preference applied
7. IR transformation fires: SummaryStrip node → SummaryCardGrid node (10.4 §4.11)
8. OverviewMonitor renders MetricTile instances in 3-column card grid layout

**Failure path (tenant has NOT pre-approved `card_grid`):**

1–4 same as above
5. Tenant has not approved `card_grid` → governance gate fails
6. Preference discarded — conflictPolicy is not evaluated (gate precedes policy)
7. Surface falls back to role-scope or global default: `horizontal_strip`
8. Orphan log created: same 5-field record format as §3.6 — `reason: "structural_variant_not_tenant_approved"`
9. User sees no error; silent fallback to default layout

**Why this matters:** This path validates that structural variant governance (described in 10.4) actually functions as a separate, ordered gate — not just as a preference override. Compiler must enforce the governance check before applying any user structural variant preference, and must produce the correct orphan log on failure. This behavior had been specified in 10.4 but had not been traced end-to-end until this workflow.

### 6.5 Grammar gaps requiring follow-up

Gaps are annotated inline in the relevant screen sequences with their Gap ID (e.g., **GW-G1:**). The table below summarizes each gap, its in-document location, and the required spec amendment.

| Gap ID | Description | In-document location | Required amendment |
|---|---|---|---|
| GW-G1 | `actionType=request_information` has no defined UI outcome — no seam to CommentConversationPanel or task creation | W2 Screen 2 ActionPanel note | 10.3 P6: `actionType=request_information` → mounts CommentConversationPanel in compose state. 10.5 §C15 open seams: add reference. |
| GW-G2 | TaskContextPanel aggregate mode (multiple instances as count) has no distinct state model from single-instance mode | W3 Screen 1 TaskContextPanel note | 10.5 §C14: when rendering in OverviewMonitor with aggregate data, only `active` and `no_task_instance` states are valid. |
| GW-G3 | AttachmentPanel inside `isReadOnly=true` SmartFormSection has no declared slot | W1 Screen 4 SmartFormSection note | 10.5 §C3: add `additional-content` slot, available when `isReadOnly=true`, accepting AttachmentPanel and ActivityTimeline. |
| GW-G4 | StatusBanner system-replace behavior (persistent banner updated in-place by system event) is unspecified | W3 Screen 4 post-submit note | 10.5 §C4 composition rules: `isPersistent=true` banner may be replaced in-place by a system event; replace preserves stacking order position. |
| GW-G5 | ExceptionResolutionView: post-dismiss-alert navigation target is undefined | W3 Screen 2 ActionPanel note | 10.3 P5: on `action_dismiss_alert` confirm, navigate to originating view (OverviewMonitor or parent). |

All five gaps are minor specification amendments — no new patterns or components are required.

### 6.6 Deferred concerns — out of scope for v0

These are real system concerns that surfaced during the trace but are not in scope for 10.6 or the current 10.1–10.5 grammar documents. They are tracked here for visibility.

| Concern | Why it surfaced | Scope |
|---|---|---|
| Interaction / State Transition Model | The traces describe what happens after each action (e.g., "PO transitions to Pending Approval") but do not define the event → handler → state update → re-render pipeline. The grammar specifies what the UI grammar should express, not the runtime execution model. | Separate future document — runtime/compiler execution model. Not a UI grammar concern. |
| Data binding and fetching layer | Component inputs are marked "runtime" throughout — but the mechanism for fetching entity data, caching, mutation, and sync is not defined. The grammar declares what data components receive; it does not specify how that data arrives. | Compiler integration / backend contract layer. Outside workstream scope. |
| SearchResults (P4) not validated | No workflow in this set exercises the SearchResults pattern or cross-entity search. Search is cross-cutting and exercises different grammar rules than record-centric workflows. | W4 candidate for a future 10.6 revision. |
| AI system not validated | AIAssistPanel (S2) is defined in 10.5 but not exercised. The AI suggestion lifecycle (suggestion shown → accepted / rejected → impact on business state) was not traced. | W5 candidate for a future 10.6 revision covering AI-assisted form fill or AI-generated summaries. |
| RelatedRecordsSection and CommentConversationPanel not exercised | C10 and C15 appear in the composition matrix but are absent from these three workflows. Both would be exercised in record-centric workflows with linked entities or collaborative annotation. | W4/W5 candidates. Not critical for the core grammar validation this document provides. |

---

## 7. Milestone 6 exit criteria — self-assessment

| Exit criterion | Status | Evidence |
|---|---|---|
| Representative workflows can be expressed with the patterns and components | Met | All three workflows traced end-to-end using only vocabulary from 10.1–10.5. No new patterns, components, or variant surfaces required. |
| Usability issues are surfaced early | Met | 10 workflow-specific issues (W1-I1 through W3-I4) + 5 grammar-level gaps (GW-G1 through GW-G5). All are specification-level issues — inexpensive to fix before implementation begins. |
| Personalization assumptions are exercised | Met | Six personalization behaviors traced: allowed surfaces restriction, role_overrides_user, user_overrides_role, structural variant governance (full path trace in §6.4 including failure path), orphan logging, and variant → IR transformation. |
| Generator implications are visible | Met | Each workflow includes a Generator Implications table. 24 distinct compiler tasks identified. Grammar is sufficient to drive each task deterministically. |

---

*This document is version 0. It validates the grammar defined in 10.1–10.5 against three illustrative SMB workflows for Apex Components Ltd. It is expected to be updated when 10.7 Primitive Mapping resolves implementation substrate decisions, when real pilot workflows replace the illustrative scenarios, and when a W4 trace exercises the unexercised patterns (SearchResults, AssistedSetupFlow) and components (RelatedRecordsSection, CommentConversationPanel, AIAssistPanel).*
