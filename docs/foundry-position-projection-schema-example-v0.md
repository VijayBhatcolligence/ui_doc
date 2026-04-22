# Foundry Position Projection Schema — Worked Example

**Document type:** Companion example to Work Product 10.2  
**References:** Foundry Position Projection Schema v0  
**Purpose:** Walk through every field in the schema using a single concrete position — Procurement Coordinator at Apex Components Ltd. — so each field has a real value, a reason for that value, and the rule it satisfies.

---

## The position being modelled

**Company:** Apex Components Ltd. — mid-sized manufacturing SMB  
**Position:** Procurement Coordinator  
**What they do:** Create and manage purchase orders, submit them for Finance Manager approval, track delivery status, and chase overdue supplier quotes.  
**What they do not do:** Approve anything (that is Finance Manager), receive goods (that is Warehouse Manager).

This projection is **partial** — approvals and tasks are modelled; alerts and decisions are deferred to the next semantic slice iteration.

---

## How to read this document

Each section shows the actual JSON snippet followed by a field-by-field explanation table:

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|

---

## Section 1 — `identity`

```json
"identity": {
  "positionId": "proc-coord-apex-001",
  "positionTitle": "Procurement Coordinator",
  "positionSlug": "procurement-coordinator",
  "tenantId": "apex-components",
  "boundedContextId": "bc-procurement",
  "orgUnit": "Supply Chain",
  "positionVersion": "1.0.0"
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `positionId` | `proc-coord-apex-001` | Unique stable identifier for this position at this tenant. Used as the primary key everywhere the system references this position. | Invariant — must never change across regenerations. Changing it is a breaking change. |
| `positionTitle` | `Procurement Coordinator` | The human-readable name shown in the app header and navigation. Matches the role name the business uses. | Shown in UI; must match business vocabulary. |
| `positionSlug` | `procurement-coordinator` | URL-safe version of the title used in routing. Lowercase, hyphenated, no special characters. | Invariant — must never change. Used in URLs and deep links. |
| `tenantId` | `apex-components` | Identifies which company this position belongs to. Scopes all data, policies, and personalization. | Required. All projection data is tenant-scoped. |
| `boundedContextId` | `bc-procurement` | Points to the bounded context in the semantic layer that owns this position's primary data (PurchaseOrder, Supplier, etc.). The UI layer uses this as a reference — it does not read the bounded context definition itself. | Seam — the bounded context definition lives in the semantic layer, not here. |
| `orgUnit` | `Supply Chain` | Display-only department label. Used for grouping in admin views. Not a routing key. | Optional. Not used in pattern selection or permissions logic. |
| `positionVersion` | `1.0.0` | Semver version of this specific position's compiled projection. Incremented when any invariant field changes. | Major bump = breaking change. Triggers migration plan. |

---

## Section 2 — `workSurface`

WorkSurface is split into two parts: **NavigationSpec** (how the user moves between views) and **ViewRegistry** (what each view is).

### 2a — NavigationSpec

```json
"workSurface": {
  "navigation": {
    "defaultLandingViewId": "view-po-list",
    "navItems": [
      {
        "viewId": "view-po-list",
        "navLabel": "Purchase Orders",
        "navOrder": 1,
        "navIcon": "document-list",
        "isHiddenFromNav": false
      },
      {
        "viewId": "view-po-overview",
        "navLabel": "Overview",
        "navOrder": 2,
        "navIcon": "dashboard",
        "isHiddenFromNav": false
      },
      {
        "viewId": "view-create-po",
        "navLabel": null,
        "navOrder": null,
        "navIcon": null,
        "isHiddenFromNav": true
      }
    ],
    "navStyle": "sidebar"
  }
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `defaultLandingViewId` | `view-po-list` | The PO list is the Procurement Coordinator's daily starting point — they arrive and immediately see their open orders. | Must reference a valid `viewId` in the view registry. |
| `navItems[0].viewId` | `view-po-list` | The Purchase Orders list is the primary work surface — first in nav. | Min 1 nav item required. |
| `navItems[0].navLabel` | `Purchase Orders` | Plain business language. Not "PO Management" or "Orders Module". | Must match business vocabulary used elsewhere in the position app. |
| `navItems[0].navOrder` | `1` | First in sidebar because it is the most-used view. | 1-indexed; unique within projection. |
| `navItems[2].isHiddenFromNav` | `true` | The CreateEditFlow for creating a PO is launched from a button, not navigated to directly. It is reachable but not a primary nav destination. | `isHiddenFromNav: true` means the view exists but does not appear in the main nav. The view is still reachable via action buttons. |
| `navStyle` | `sidebar` | Default navigation presentation for desktop position apps. | Must be an approved variant. `sidebar` is the default; `topbar` and `tabs` are alternates. |

---

### 2b — ViewRegistry (ViewSpec per view)

#### View 1: Purchase Orders List

```json
{
  "viewId": "view-po-list",
  "pattern": "CollectionView",
  "patternVariant": null,
  "primaryEntityType": "PurchaseOrder",
  "patternInputBinding": {
    "patternType": "CollectionView",
    "entityTypeRef": {
      "entityType": "PurchaseOrder",
      "displayNameTemplate": "{{po_number}} — {{supplier_name}}",
      "viewLink": "view-po-detail"
    },
    "listSourceDescription": "All purchase orders created by or assigned to this position, filtered by data policy",
    "filterSources": [
      { "filterKey": "status", "filterLabel": "Status", "filterType": "status", "isDefaultVisible": true },
      { "filterKey": "supplier", "filterLabel": "Supplier", "filterType": "select", "isDefaultVisible": true },
      { "filterKey": "created_date", "filterLabel": "Created Date", "filterType": "date_range", "isDefaultVisible": false }
    ],
    "actionSlots": {
      "primaryActionId": "action-create-po",
      "secondaryActionIds": ["action-bulk-export"],
      "overflowActionIds": [],
      "bulkActionIds": ["action-bulk-cancel"]
    }
  },
  "layoutHints": null,
  "accessConditionHookId": null
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `viewId` | `view-po-list` | Stable identifier for this view. Used everywhere this view is referenced (nav, viewLink, default landing, etc.). | Invariant — cannot change across regenerations. Changing it breaks all references. |
| `pattern` | `CollectionView` | The Procurement Coordinator browses a list of POs daily — this is exactly what CollectionView is for. | Must match a pattern defined in UI Pattern Library v0 (10.3). Changing this across a regeneration is a breaking change (schema C, UX Doctrine I4). |
| `patternVariant` | `null` | Using the default pattern variant — no approved structural or emphasis variant needed here. | When null, the pattern's default layout applies. |
| `entityTypeRef.displayNameTemplate` | `{{po_number}} — {{supplier_name}}` | Each PO row should show the PO number first (identifier) then the supplier name (context). Templates are resolved at runtime — the schema only declares the template string, never actual values. | Must be a template string, not a resolved value. `{{po_number}}` is resolved at runtime by the UI from live backend data. |
| `entityTypeRef.viewLink` | `view-po-detail` | Clicking a PO row in the list navigates to its detail page. The `viewLink` tells the generator which view to route to. | Optional. When present and accessible, rendered as a navigable link. When the linked view is inaccessible, rendered as a plain non-navigable label — never a broken link. |
| `listSourceDescription` | `"All purchase orders created by or assigned to this position..."` | Plain-language description of the data source. The UI layer does not implement data fetching — this tells the compiler and reviewer what data this view is sourcing. | Required for list-based patterns. Must be plain language. Not a query — that lives in the backend. |
| `filterSources[status].isDefaultVisible` | `true` | Status is the most common filter a Procurement Coordinator uses — show it without needing to expand. | When `true`, filter is visible by default. When `false`, it is behind the "More filters" expansion. |
| `filterSources[created_date].isDefaultVisible` | `false` | Date filtering is less frequent — advanced usage only. Hidden behind expansion to reduce noise. | Doctrine §7.2: common filters visible by default; advanced filters behind expansion. |
| `actionSlots.primaryActionId` | `action-create-po` | Creating a PO is the single most important action on this view. One primary action only. | Max 1 primary action (UX Doctrine P1, A1, schema C10). |
| `actionSlots.secondaryActionIds` | `["action-bulk-export"]` | Export is a useful secondary action — visible but not the focus. | Max 3 secondary actions (A2). |
| `actionSlots.bulkActionIds` | `["action-bulk-cancel"]` | Bulk cancel is only relevant when items are selected. It lives in the bulk action bar, not the page header. | Bulk actions replace page header actions during selection state (A4). |

---

#### View 2: PO Detail (RecordPage)

```json
{
  "viewId": "view-po-detail",
  "pattern": "RecordPage",
  "patternVariant": null,
  "primaryEntityType": "PurchaseOrder",
  "patternInputBinding": {
    "patternType": "RecordPage",
    "entityTypeRef": {
      "entityType": "PurchaseOrder",
      "displayNameTemplate": "PO {{po_number}}",
      "viewLink": "view-po-detail"
    },
    "actionSlots": {
      "primaryActionId": "action-submit-po",
      "secondaryActionIds": ["action-edit-po", "action-cancel-po"],
      "overflowActionIds": [],
      "bulkActionIds": []
    }
  },
  "layoutHints": {
    "primaryRegion": "PO Details",
    "sidebarRegion": "Context and Tasks",
    "headerRegion": "PO Summary",
    "groupedSections": [
      {
        "groupKey": "po-basics",
        "groupLabel": "Order Details",
        "sectionKeys": ["po_number", "supplier", "required_date"],
        "isCollapsedByDefault": false
      },
      {
        "groupKey": "line-items",
        "groupLabel": "Line Items",
        "sectionKeys": ["line_items"],
        "isCollapsedByDefault": false
      },
      {
        "groupKey": "shipping",
        "groupLabel": "Delivery and Shipping",
        "sectionKeys": ["delivery_address", "shipping_method"],
        "isCollapsedByDefault": true
      }
    ]
  },
  "accessConditionHookId": null
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `pattern` | `RecordPage` | The user clicked a specific PO and needs to see its full detail, act on it, and see its history. RecordPage is the canonical single-entity surface. | Cannot change silently across regenerations (I4). |
| `actionSlots.primaryActionId` | `action-submit-po` | On a draft PO, the most important thing the coordinator can do is submit it for approval. This is the single primary action. | Changes per record state — the schema declares the primary action for the default state. State-dependent primary action mapping is defined in the ActionSpec. |
| `actionSlots.secondaryActionIds` | `["action-edit-po", "action-cancel-po"]` | Edit and Cancel are relevant secondary actions visible alongside Submit. Two secondary actions — within the budget. | Max 3 secondary (A2). Cancel is destructive, so it will be visually isolated from Submit (A5). |
| `layoutHints.groupedSections[shipping].isCollapsedByDefault` | `true` | Shipping details are secondary information — the coordinator sees them when needed but they should not dominate the view on load. | Doctrine §7.3: progressive disclosure — relevant section expanded, rest collapsed. |
| `layoutHints.sidebarRegion` | `"Context and Tasks"` | The sidebar shows task context (follow-up tasks linked to this PO) and AI assist. Labelling it here tells the generator what to put there. | Optional. When absent, the pattern's default layout applies. |

---

#### View 3: Overview Monitor

```json
{
  "viewId": "view-po-overview",
  "pattern": "OverviewMonitor",
  "patternVariant": null,
  "primaryEntityType": null,
  "patternInputBinding": {
    "patternType": "OverviewMonitor",
    "actionSlots": {
      "primaryActionId": "action-create-po"
    }
  },
  "layoutHints": null,
  "accessConditionHookId": null
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `pattern` | `OverviewMonitor` | The coordinator needs an at-a-glance dashboard of their PO health — open count, pending approval, committed spend. | Requires at least one `StateSummarySpec` in the projection (schema C7). |
| `primaryEntityType` | `null` | OverviewMonitor is cross-entity — it shows summaries across PurchaseOrders and Suppliers, not one entity type. No single entity type applies. | Optional for OverviewMonitor — unlike CollectionView, a primary entity type is not required. |
| `actionSlots.primaryActionId` | `action-create-po` | Even from the dashboard, the coordinator's most common next action is creating a new PO. | Optional for OverviewMonitor — the view is read-first, action is secondary. |

---

#### View 4: Create PO Flow

```json
{
  "viewId": "view-create-po",
  "pattern": "CreateEditFlow",
  "patternVariant": null,
  "primaryEntityType": "PurchaseOrder",
  "patternInputBinding": {
    "patternType": "CreateEditFlow",
    "entityTypeRef": {
      "entityType": "PurchaseOrder",
      "displayNameTemplate": "New Purchase Order",
      "viewLink": null
    },
    "actionSlots": {
      "primaryActionId": "action-create-po"
    }
  },
  "layoutHints": {
    "groupedSections": [
      { "groupKey": "supplier-step", "groupLabel": "Select Supplier", "sectionKeys": ["supplier_id", "supplier_contact"], "isCollapsedByDefault": false },
      { "groupKey": "items-step", "groupLabel": "Add Items", "sectionKeys": ["line_items", "total_amount"], "isCollapsedByDefault": false },
      { "groupKey": "delivery-step", "groupLabel": "Delivery Details", "sectionKeys": ["required_date", "delivery_address", "shipping_notes"], "isCollapsedByDefault": false }
    ]
  },
  "accessConditionHookId": null
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `pattern` | `CreateEditFlow` | Creating a PO requires three sections of input — supplier, items, delivery. This spans multiple sections, making CreateEditFlow the right choice over a simple inline form. | Required when `ActionSpec.requiredInputs` spans more than one section (pattern selection rule). |
| `entityTypeRef.viewLink` | `null` | During PO creation, there is no existing PO to link to. The `viewLink` would only apply to an existing record's entity reference. | Optional. Null is valid when the entity does not yet exist. |
| `layoutHints.groupedSections` | 3 steps | Three logical groups — supplier, items, delivery — become three steps in the flow. `groupedSections` drives the stepper. | `groupedSections` count > 2 triggers the `CreateEditFlow.form_layout` structural variant to use stepped multi-section layout (from structural variant registry). |

---

## Section 3 — `actions`

### Action: Create Purchase Order

```json
{
  "actionId": "action-create-po",
  "actionType": "CreatePurchaseOrder",
  "label": "Create Purchase Order",
  "targetEntityType": "PurchaseOrder",
  "preconditions": [],
  "requiredInputs": [
    { "fieldKey": "supplier_id", "fieldLabel": "Supplier", "inputType": "select", "isRequired": true, "validationRules": ["Must be an approved supplier"] },
    { "fieldKey": "line_items", "fieldLabel": "Line Items", "inputType": "multiselect", "isRequired": true, "validationRules": ["At least one line item required"] },
    { "fieldKey": "required_date", "fieldLabel": "Required By", "inputType": "date", "isRequired": true, "validationRules": ["Must be at least 3 business days from today"] }
  ],
  "confirmationType": "none",
  "isDestructive": false,
  "requiresApproval": false,
  "postActionBehavior": "navigateTo",
  "postActionNavigateToViewId": "view-po-detail",
  "aiAssistHookIds": [],
  "telemetryKey": "procurement_coordinator.create_purchase_order"
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `actionId` | `action-create-po` | Stable identifier. Referenced from `actionSlots`, `tasks.availableActionIds`, `approvals.linkedActionId`. | Invariant. Must not change across regenerations. |
| `actionType` | `CreatePurchaseOrder` | Business classification used by the cross-projection action label registry. If another position also creates POs, its `CreatePurchaseOrder` action must use the same label. | Cross-projection label consistency enforced at compiler level (P8, I8). |
| `label` | `Create Purchase Order` | The exact label shown on the button. Must match what Finance Manager sees if they also see this action label anywhere. | Must be identical for the same `actionType` across all positions in the same tenant. |
| `preconditions` | `[]` | No preconditions — a Procurement Coordinator can create a PO at any time. | Empty list is valid. When conditions exist, each declares whether to `hide` or `disable_with_tooltip` (A6). |
| `confirmationType` | `none` | Creating a PO is reversible (it starts as Draft). No confirmation step needed before the form is launched. | `isDestructive: true` would require `confirmationType: full`. Non-destructive actions can use `none` or `inline`. |
| `requiresApproval` | `false` | The creation itself does not require approval — submission to approval is a separate action (`action-submit-po`). | When `true`, a corresponding `ApprovalSpec` with `linkedActionId` pointing here must exist (schema C9). |
| `postActionBehavior` | `navigateTo` | After creating the PO, navigate the user to the new PO's detail page so they can review it and submit. | When `navigateTo`, `postActionNavigateToViewId` must be non-null. |
| `telemetryKey` | `procurement_coordinator.create_purchase_order` | Stable key for the telemetry hook. Uses dot notation: `{position}.{action}`. Never changes — changing it breaks analytics. | Invariant. Tracked in the instrumentation/evaluation layer. Changing it is a breaking change. |

---

### Action: Submit for Approval

```json
{
  "actionId": "action-submit-po",
  "actionType": "SubmitPurchaseOrderForApproval",
  "label": "Submit for Approval",
  "targetEntityType": "PurchaseOrder",
  "preconditions": [
    {
      "conditionDescription": "PO must be in Draft status",
      "displayWhenNotMet": "disable_with_tooltip",
      "tooltipText": "This PO has already been submitted or is not in draft state"
    }
  ],
  "confirmationType": "inline",
  "isDestructive": false,
  "requiresApproval": true,
  "postActionBehavior": "stay",
  "postActionNavigateToViewId": null,
  "aiAssistHookIds": [],
  "telemetryKey": "procurement_coordinator.submit_po_for_approval"
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `preconditions[0].displayWhenNotMet` | `disable_with_tooltip` | The user needs to know this action exists — they may wonder why they cannot submit. Hiding it would leave them confused. The tooltip explains why. | `hide` = user does not need to know the action exists. `disable_with_tooltip` = user needs to know it exists but cannot use it yet (A6). |
| `confirmationType` | `inline` | Submitting for approval is a significant step but not irreversible — the coordinator can withdraw. Inline confirmation (a prompt in place) is proportional to this risk level. | Doctrine P5: low risk = none; medium risk = inline; high/irreversible = full. |
| `requiresApproval` | `true` | This action initiates an approval workflow. The Finance Manager must review and approve/reject. | When `true`, a corresponding `ApprovalSpec` must exist with `linkedActionId: "action-submit-po"` (C9). |
| `postActionBehavior` | `stay` | After submitting, the user stays on the PO detail page to see the updated status (now showing "Pending Approval"). | `stay` = remain on current view. The status badge update gives immediate feedback. |

---

### Action: Cancel PO

```json
{
  "actionId": "action-cancel-po",
  "actionType": "CancelPurchaseOrder",
  "label": "Cancel PO",
  "targetEntityType": "PurchaseOrder",
  "preconditions": [
    {
      "conditionDescription": "PO must not be in Completed or already Cancelled status",
      "displayWhenNotMet": "hide",
      "tooltipText": null
    }
  ],
  "requiredInputs": [
    {
      "fieldKey": "cancellation_reason",
      "fieldLabel": "Reason for Cancellation",
      "inputType": "textarea",
      "isRequired": true,
      "validationRules": ["Minimum 10 characters"]
    }
  ],
  "confirmationType": "full",
  "isDestructive": true,
  "requiresApproval": false,
  "postActionBehavior": "navigateTo",
  "postActionNavigateToViewId": "view-po-list",
  "aiAssistHookIds": [],
  "telemetryKey": "procurement_coordinator.cancel_purchase_order"
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `preconditions[0].displayWhenNotMet` | `hide` | There is no reason for the user to see a "Cancel PO" button on an already-cancelled PO. The action is irrelevant in that state — hide it entirely. | A6: context-sensitive unavailable actions are hidden, not disabled, unless the user needs to know they exist. Here they do not. |
| `isDestructive` | `true` | Cancelling a PO cannot be undone — the order is gone from the supplier's pipeline. | Requires `confirmationType: full` — hard constraint (schema C8). |
| `confirmationType` | `full` | Full confirmation step required because this is irreversible. The user sees a modal with the consequence before confirming. | Forced by `isDestructive: true` (C8). Not a design choice — a schema rule. |
| `requiredInputs` | `cancellation_reason` (textarea, required) | The business needs to know why a PO was cancelled for audit and reporting. This input is collected as part of the cancellation flow, not optionally. | Doctrine §7.4: decisions that materially change business state must capture rationale. |
| `postActionBehavior` | `navigateTo` → `view-po-list` | After cancellation the PO is gone — there is nothing to show on its detail page. Navigate back to the list. | When `navigateTo`, `postActionNavigateToViewId` must be non-null. |

---

## Section 4 — `permissionsHooks`

```json
"permissionsHooks": {
  "permissionsVersion": "opa-policy-v2.1",
  "allowedActionIds": [
    "action-create-po",
    "action-submit-po",
    "action-edit-po",
    "action-cancel-po",
    "action-bulk-cancel",
    "action-bulk-export"
  ],
  "visibleEntityTypes": ["PurchaseOrder", "Supplier", "Product"],
  "dataFilterPolicies": [
    {
      "policyId": "policy-own-pos-only",
      "description": "Show only POs created by or assigned to this user or their team",
      "appliesTo": "PurchaseOrder"
    }
  ],
  "conditionalAccessHooks": [
    {
      "hookId": "cond-po-edit-access",
      "description": "Edit is available only when PO is in Draft or Rejected status",
      "evaluationTiming": "runtime"
    }
  ]
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `permissionsVersion` | `opa-policy-v2.1` | The version of the OPA policy schema this projection was compiled against. At runtime, OPA checks this version matches. Mismatch = policy enforcement error. | Seam — the UI layer passes this to the backend; OPA does the matching. The UI layer does not read policy internals. |
| `allowedActionIds` | 6 action ids | The Procurement Coordinator can do these 6 things. The generator will never surface any action not in this list regardless of what `actionSlots` declares. | The generator must not surface any action absent from this list (schema rule, C4). This is the security gate at the schema level. |
| `visibleEntityTypes` | `PurchaseOrder, Supplier, Product` | The coordinator can see POs, the suppliers they are from, and the products being ordered. They cannot see Finance records, Payroll, or other bounded contexts. | The UI does not render data for entity types not listed here. |
| `dataFilterPolicies[0]` | `policy-own-pos-only` | The coordinator sees their team's POs, not every PO company-wide. The `policyId` is an opaque reference — the UI passes it to the backend. The filter logic lives in OPA, not here. | The UI layer is responsible for routing the policy id — not for implementing the filter. Enforcement belongs to the policy layer. |
| `conditionalAccessHooks[0].evaluationTiming` | `runtime` | Whether a PO is in Draft or Rejected status is only known at runtime (it depends on the live record). The generator cannot evaluate this at compile time. | `compile_time` = can be resolved statically from the schema. `runtime` = depends on live data, evaluated by the permissions layer when the view loads. |

---

## Section 5 — `defaultViewHints`

```json
"defaultViewHints": {
  "defaultLandingViewId": "view-po-list",
  "defaultDensityMode": "default",
  "defaultSortSpecs": [
    { "viewId": "view-po-list", "fieldKey": "created_date", "direction": "desc" }
  ],
  "defaultFilterSpecs": [
    { "viewId": "view-po-list", "fieldKey": "status", "filterDescription": "Open — excludes Completed and Cancelled" }
  ],
  "defaultExpandedSections": [
    { "viewId": "view-po-detail", "sectionKey": "po-basics" }
  ],
  "pinnedSummaryIds": ["summary-open-pos", "summary-pending-approval"]
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `defaultLandingViewId` | `view-po-list` | The PO list is the most-used daily starting point. The coordinator opens the app and immediately sees their active POs. | Must reference a valid `viewId` in `workSurface.views`. |
| `defaultDensityMode` | `default` | Comfortable density for a desktop user who reads PO details carefully. Power-user compact mode is not the default. | Doctrine D1: generous spacing is the baseline. Compact is an opt-in, not the default. |
| `defaultSortSpecs` | `created_date desc` | Most recent POs first — the coordinator is usually working on recently created orders. | This is a generator-time default. The user can change it and save it via personalization. |
| `defaultFilterSpecs` | `status = Open` | The coordinator cares about active POs, not completed or cancelled ones. Filtering out closed orders reduces noise on every load. | Plain-language description only — the actual filter value interpretation lives in the backend. |
| `defaultExpandedSections` | `po-basics` on `view-po-detail` | "Order Details" contains the most critical fields (PO number, supplier, required date). It should be expanded immediately on arrival. | Other sections default to collapsed per `layoutHints.groupedSections.isCollapsedByDefault`. |
| `pinnedSummaryIds` | `summary-open-pos`, `summary-pending-approval` | Open POs and pending approvals are the two numbers the coordinator checks first every morning. Pin them at the top of the OverviewMonitor. | References entries in `stateSummaries` section. |

---

## Section 6 — `personalizationHooks`

```json
"personalizationHooks": {
  "allowedSurfaces": [
    {
      "surfaceId": "surface-density",
      "surfaceType": "density_mode",
      "scope": "user",
      "description": "Switch between comfortable and compact density",
      "isStructuralVariant": false,
      "targetViewIds": [],
      "targetComponentTypes": [],
      "configurationSchema": [
        {
          "fieldKey": "density",
          "fieldLabel": "Display Density",
          "valueType": "enum",
          "allowedValues": ["default", "compact"],
          "defaultValue": "default"
        }
      ]
    },
    {
      "surfaceId": "surface-saved-filter",
      "surfaceType": "saved_filter",
      "scope": "user",
      "description": "Save and recall custom filter combinations on the Purchase Orders list",
      "isStructuralVariant": false,
      "targetViewIds": ["view-po-list"],
      "targetComponentTypes": ["FilterRail"],
      "configurationSchema": []
    },
    {
      "surfaceId": "surface-po-list-style",
      "surfaceType": "structural_variant",
      "scope": "tenant",
      "description": "Switch the PO list between semantic record list and compact grid",
      "isStructuralVariant": true,
      "targetViewIds": ["view-po-list"],
      "targetComponentTypes": ["RecordList"],
      "configurationSchema": [
        {
          "fieldKey": "list_style",
          "fieldLabel": "List Style",
          "valueType": "enum",
          "allowedValues": ["semantic_list", "compact_grid"],
          "defaultValue": "semantic_list"
        }
      ]
    }
  ],
  "conflictPolicy": "notify_and_preserve"
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `surface-density.scope` | `user` | Each individual coordinator can choose their own density — compact for experienced users, comfortable for new ones. This does not affect other users. | Scope hierarchy: global → tenant → role → user. User scope is the most granular. |
| `surface-density.isStructuralVariant` | `false` | Density mode only changes spacing and font size. It does not change layout structure, regions, or interaction grammar. | `isStructuralVariant: false` = token-driven or view-state change, not a layout change. |
| `surface-saved-filter.configurationSchema` | `[]` (empty) | Saved filters are free-form — the coordinator can save any filter combination they want. There is no fixed schema to configure; the runtime filter state is what gets saved. | Empty `configurationSchema` is valid when the surface captures free-form user state rather than a fixed set of options. |
| `surface-po-list-style.isStructuralVariant` | `true` | Switching from semantic record list to compact grid changes the layout structure of the list region. This is an approved pattern variant, not cosmetic. | **Critical rule:** `isStructuralVariant: true` requires `scope: tenant` or `scope: role`. `scope: user` is not permitted. Structural variants are product artifacts — they are set by the company (tenant), not individual users. Enforced at compile time (C12). |
| `surface-po-list-style.scope` | `tenant` | The compact grid layout is a company-level decision at Apex Components. All coordinators at Apex either get the grid or they do not — it is not individual choice. | Schema C12: `isStructuralVariant: true` + `scope: user` = compile error. |
| `conflictPolicy` | `notify_and_preserve` | If a regeneration invalidates a saved preference (e.g., a filter field no longer exists), notify the coordinator and preserve what remains valid. Do not silently reset everything. | Doctrine §3.5: silent loss of personalization is not acceptable. |

---

## Section 7 — `completeness`

```json
"completeness": {
  "sliceStatus": "partial",
  "resolvedCapabilities": ["tasks", "approvals", "stateSummaries"],
  "deferredCapabilities": ["alerts", "decisions"],
  "outOfScopeCapabilities": ["collaborators", "monitoredEntities"]
}
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `sliceStatus` | `partial` | The semantic slice for this position is not fully modelled yet. Tasks and approvals are done. Alerts and decisions are known to be needed but not yet modelled in the semantic layer. | `full` = everything modelled. `partial` = some in-scope capabilities deferred. `provisional` = the whole model is subject to revision. |
| `resolvedCapabilities` | `tasks, approvals, stateSummaries` | These three extension sections are fully modelled and present in this projection. The generator renders them normally. | The generator uses this list to confirm which sections to render fully. |
| `deferredCapabilities` | `alerts, decisions` | These are known to be in scope for the Procurement Coordinator — they will get low-stock alerts and supplier selection decisions in a future slice. But they are not modelled yet. The UI generator must NOT treat them as absent. It must render their slots as DeferredCapabilityPlaceholder: "not yet available." | **Key rule:** deferred ≠ absent. A deferred slot must never render as empty, zero, or blank. The generator renders a placeholder. This satisfies UX Doctrine §2.8: unknown must not masquerade as zero or empty. (C11) |
| `outOfScopeCapabilities` | `collaborators, monitoredEntities` | The Procurement Coordinator genuinely does not need cross-position hand-off flows (collaborators) or continuous entity monitoring (monitoredEntities) in the current scope. Their absence is intentional, not a gap. | Out-of-scope = definitively absent. The generator hides these slots entirely. No placeholder needed. |

**Why this matters:** Without this section, the generator seeing no `alerts` section would not know whether to show an empty state, a placeholder, or nothing. Now it knows exactly: `alerts` → deferred → show placeholder. `collaborators` → out of scope → show nothing.

---

## Section 8 — `tasks`

```json
"tasks": [
  {
    "taskSpecId": "task-follow-up-delivery",
    "taskType": "FollowUpDelivery",
    "titleTemplate": "Follow up on delivery for {{po_number}}",
    "entityTypeRef": {
      "entityType": "PurchaseOrder",
      "displayNameTemplate": "PO {{po_number}} — {{supplier_name}}",
      "viewLink": "view-po-detail"
    },
    "stateModel": ["open", "in_progress", "resolved", "cancelled"],
    "priorityLevels": ["low", "normal", "high", "urgent"],
    "hasDueDate": true,
    "availableActionIds": ["action-cancel-po"],
    "aiAssistHookIds": []
  },
  {
    "taskSpecId": "task-review-quote",
    "taskType": "ReviewSupplierQuote",
    "titleTemplate": "Review quote from {{supplier_name}}",
    "entityTypeRef": null,
    "stateModel": ["open", "accepted", "rejected"],
    "priorityLevels": ["low", "normal", "high"],
    "hasDueDate": true,
    "availableActionIds": ["action-create-po"],
    "aiAssistHookIds": []
  }
]
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `titleTemplate` | `"Follow up on delivery for {{po_number}}"` | The task title resolves at runtime using the actual PO number. Schema-level only stores the template — never the resolved string. | Template string only. `{{po_number}}` is resolved at runtime. No static runtime values in the schema. |
| `entityTypeRef` on task-follow-up-delivery | PurchaseOrder with `viewLink: view-po-detail` | This task is about a specific PO. Clicking the task navigates to the PO's detail page via `viewLink`. | `viewLink` is optional. When accessible, renders as a navigable link. When inaccessible, renders as plain label. |
| `entityTypeRef` on task-review-quote | `null` | A quote review task is not about a specific existing PO — it is a pre-PO step. No entity to link to yet. | Null is valid when the task is not tied to a specific existing entity. |
| `stateModel` | `["open", "in_progress", "resolved", "cancelled"]` | These are the valid lifecycle states for a delivery follow-up task. The schema declares them — the UI renders them as status badges using these exact labels. | State labels declared here are invariant — the UI Doctrine I3 requires business state labels to remain stable across regenerations. |
| `availableActionIds` | `["action-cancel-po"]` on FollowUpDelivery | From a delivery follow-up task, the coordinator may need to cancel the PO if the supplier cannot deliver. The schema wires that action to this task type. | Must only reference `actionId` values defined in the `actions` section (C1). |

---

## Section 9 — `approvals`

```json
"approvals": [
  {
    "approvalSpecId": "approval-po-standard",
    "approvalType": "PurchaseOrderApproval",
    "subjectEntityType": "PurchaseOrder",
    "thisPositionRole": "initiator",
    "requiredApproverPositionIds": ["finance-manager"],
    "approvalDeadlineHours": 48,
    "showConsequenceOnReview": true,
    "rejectionRequiresReason": true,
    "approvalHistoryVisible": true,
    "linkedActionId": "action-submit-po",
    "linkedViewId": null
  }
]
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `thisPositionRole` | `initiator` | The Procurement Coordinator starts the approval — they do not approve. The Finance Manager's projection has a separate `ApprovalSpec` with `thisPositionRole: approver` for the same `approvalType`. | Each position's projection declares only its own role. The two sides of the approval workflow are described in separate projections. |
| `requiredApproverPositionIds` | `["finance-manager"]` | Only the Finance Manager approves POs at Apex Components. This wires the approval routing — the system knows who to notify. | Required when `thisPositionRole: initiator`. Tells the system who must act on the other side. |
| `approvalDeadlineHours` | `48` | PO approvals must be acted on within 48 hours at Apex. If the Finance Manager does not act, the system escalates. | Optional. When absent, no automated escalation. When present, connects to the Temporal orchestration layer for escalation execution. |
| `showConsequenceOnReview` | `true` | The Finance Manager reviewing this PO must see what they are approving — the supplier, amount, and business consequence. This ensures the approval review view surfaces that context. | Doctrine §7.4: approval must always show what is being approved, the consequence, and who initiated it. |
| `rejectionRequiresReason` | `true` | When the Finance Manager rejects a PO, the Procurement Coordinator needs to know why so they can revise and resubmit. Rejection without a reason leaves the coordinator unable to act. | **Must always be `true` — non-negotiable** (doctrine §7.4). This is one of the few fields in the schema where a specific boolean value is required. |
| `approvalHistoryVisible` | `true` | The coordinator must be able to see the approval history on the PO detail page — who approved, who rejected, when, and why. | **Must always be `true`** (doctrine §7.4). Approval history is always surfaced on the record. |
| `linkedActionId` | `action-submit-po` | This approval workflow is triggered by the "Submit for Approval" action. The schema wires the action to the spec. | C9: any `ActionSpec` with `requiresApproval: true` must have a corresponding `ApprovalSpec` with `linkedActionId` pointing to it. |
| `linkedViewId` | `null` | The coordinator does not have an ApprovalReviewView — they are the initiator, not the approver. The Finance Manager's projection has `linkedViewId` pointing to their approval review view. | Optional. Only needed on the approver side. |

---

## Section 10 — `stateSummaries`

```json
"stateSummaries": [
  {
    "summaryId": "summary-open-pos",
    "summaryType": "count",
    "label": "Open Purchase Orders",
    "entityType": "PurchaseOrder",
    "filterDescription": "Status is Open or Draft",
    "alertThreshold": null,
    "linkedViewId": "view-po-list",
    "displayOrder": 1
  },
  {
    "summaryId": "summary-pending-approval",
    "summaryType": "count",
    "label": "Pending Approval",
    "entityType": "PurchaseOrder",
    "filterDescription": "Status is Pending Approval",
    "alertThreshold": {
      "thresholdDescription": "More than 5 POs waiting over 24 hours",
      "severity": "warning"
    },
    "linkedViewId": "view-po-list",
    "displayOrder": 2
  },
  {
    "summaryId": "summary-committed-spend",
    "summaryType": "amount",
    "label": "Committed This Month",
    "entityType": "PurchaseOrder",
    "filterDescription": "Approved POs created in the current calendar month",
    "alertThreshold": null,
    "linkedViewId": null,
    "displayOrder": 3
  }
]
```

| Field | Value used | Why this value | Rule / constraint |
|---|---|---|---|
| `summaryType` | `count` on first two, `amount` on third | Open POs and pending approvals are counts (how many). Committed spend is a currency amount (how much). The generator uses `summaryType` to decide how to format the tile value. | Supported types: `count`, `amount`, `ratio`, `status`. |
| `filterDescription` | `"Status is Open or Draft"` | Plain-language description of what filter produces this metric. The backend resolves the actual value — the schema only describes the intent in human-readable terms. | Must be plain language. Not a query expression. The actual filter logic lives in the backend. |
| `alertThreshold` on summary-pending-approval | severity: `warning` | If more than 5 POs are waiting over 24 hours, the tile turns into a warning indicator — the coordinator knows action is needed. | Optional. When the threshold is crossed at runtime, the tile renders with the declared severity styling. |
| `alertThreshold` on summary-open-pos | `null` | Having open POs is normal — it is not a warning condition. No threshold needed. | Null is valid and common. Not every metric needs an alert threshold. |
| `linkedViewId` on summary-committed-spend | `null` | Committed spend is informational — there is no list to navigate to that makes sense for "all approved POs this month." | Optional. When null, the tile is non-clickable. When present, clicking the tile navigates to the linked view. |
| `displayOrder` | `1, 2, 3` | Open POs first (most operationally relevant), pending approval second (requires action), committed spend third (informational). | 1-indexed; unique within projection. Generator renders tiles in this order on the OverviewMonitor strip. |

---

## What is absent from this projection and why

| Section | Status | Reason |
|---|---|---|
| `alerts` | Deferred | In scope — the coordinator will receive low-stock and overdue-delivery alerts in a future semantic slice. Not modelled yet. Generator renders DeferredCapabilityPlaceholder. |
| `decisions` | Deferred | In scope — supplier selection decisions are planned for the next slice. Not modelled yet. Generator renders DeferredCapabilityPlaceholder. |
| `collaborators` | Out of scope | The Procurement Coordinator's cross-position hand-offs are handled via approval workflow, not explicit collaborator specs. Genuinely not needed here. Generator hides entirely. |
| `monitoredEntities` | Out of scope | Continuous monitoring is covered by `stateSummaries`. Separate `monitoredEntities` section is not needed for this position. Generator hides entirely. |
| `aiAssistHooks` | Out of scope (v0) | AI assistance for PO creation (vendor suggestion) is noted but not modelled in v0. Deferred to a future iteration. |

---

## Complete assembled PositionProjection JSON

This is the single JSON object the compiler produces for the Procurement Coordinator at Apex Components Ltd. This is exactly what the UI generator receives and reads to build the position app — nothing more, nothing less.

```json
{
  "identity": {
    "positionId": "proc-coord-apex-001",
    "positionTitle": "Procurement Coordinator",
    "positionSlug": "procurement-coordinator",
    "tenantId": "apex-components",
    "boundedContextId": "bc-procurement",
    "orgUnit": "Supply Chain",
    "positionVersion": "1.0.0"
  },

  "workSurface": {
    "navigation": {
      "defaultLandingViewId": "view-po-list",
      "navStyle": "sidebar",
      "navItems": [
        {
          "viewId": "view-po-list",
          "navLabel": "Purchase Orders",
          "navOrder": 1,
          "navIcon": "document-list",
          "isHiddenFromNav": false
        },
        {
          "viewId": "view-po-overview",
          "navLabel": "Overview",
          "navOrder": 2,
          "navIcon": "dashboard",
          "isHiddenFromNav": false
        },
        {
          "viewId": "view-create-po",
          "navLabel": null,
          "navOrder": null,
          "navIcon": null,
          "isHiddenFromNav": true
        }
      ]
    },
    "views": [
      {
        "viewId": "view-po-list",
        "pattern": "CollectionView",
        "patternVariant": null,
        "primaryEntityType": "PurchaseOrder",
        "patternInputBinding": {
          "patternType": "CollectionView",
          "entityTypeRef": {
            "entityType": "PurchaseOrder",
            "displayNameTemplate": "{{po_number}} — {{supplier_name}}",
            "viewLink": "view-po-detail"
          },
          "listSourceDescription": "All purchase orders created by or assigned to this position, filtered by data policy",
          "filterSources": [
            { "filterKey": "status",       "filterLabel": "Status",       "filterType": "status",     "isDefaultVisible": true  },
            { "filterKey": "supplier",     "filterLabel": "Supplier",     "filterType": "select",     "isDefaultVisible": true  },
            { "filterKey": "created_date", "filterLabel": "Created Date", "filterType": "date_range", "isDefaultVisible": false }
          ],
          "actionSlots": {
            "primaryActionId":   "action-create-po",
            "secondaryActionIds": ["action-bulk-export"],
            "overflowActionIds":  [],
            "bulkActionIds":      ["action-bulk-cancel"]
          }
        },
        "layoutHints": null,
        "accessConditionHookId": null
      },
      {
        "viewId": "view-po-detail",
        "pattern": "RecordPage",
        "patternVariant": null,
        "primaryEntityType": "PurchaseOrder",
        "patternInputBinding": {
          "patternType": "RecordPage",
          "entityTypeRef": {
            "entityType": "PurchaseOrder",
            "displayNameTemplate": "PO {{po_number}}",
            "viewLink": "view-po-detail"
          },
          "actionSlots": {
            "primaryActionId":    "action-submit-po",
            "secondaryActionIds": ["action-edit-po", "action-cancel-po"],
            "overflowActionIds":  [],
            "bulkActionIds":      []
          }
        },
        "layoutHints": {
          "primaryRegion": "PO Details",
          "sidebarRegion": "Context and Tasks",
          "headerRegion":  "PO Summary",
          "groupedSections": [
            {
              "groupKey": "po-basics",
              "groupLabel": "Order Details",
              "sectionKeys": ["po_number", "supplier", "required_date"],
              "isCollapsedByDefault": false
            },
            {
              "groupKey": "line-items",
              "groupLabel": "Line Items",
              "sectionKeys": ["line_items"],
              "isCollapsedByDefault": false
            },
            {
              "groupKey": "shipping",
              "groupLabel": "Delivery and Shipping",
              "sectionKeys": ["delivery_address", "shipping_method"],
              "isCollapsedByDefault": true
            }
          ]
        },
        "accessConditionHookId": null
      },
      {
        "viewId": "view-po-overview",
        "pattern": "OverviewMonitor",
        "patternVariant": null,
        "primaryEntityType": null,
        "patternInputBinding": {
          "patternType": "OverviewMonitor",
          "actionSlots": {
            "primaryActionId":    "action-create-po",
            "secondaryActionIds": [],
            "overflowActionIds":  [],
            "bulkActionIds":      []
          }
        },
        "layoutHints": null,
        "accessConditionHookId": null
      },
      {
        "viewId": "view-create-po",
        "pattern": "CreateEditFlow",
        "patternVariant": null,
        "primaryEntityType": "PurchaseOrder",
        "patternInputBinding": {
          "patternType": "CreateEditFlow",
          "entityTypeRef": {
            "entityType": "PurchaseOrder",
            "displayNameTemplate": "New Purchase Order",
            "viewLink": null
          },
          "actionSlots": {
            "primaryActionId":    "action-create-po",
            "secondaryActionIds": [],
            "overflowActionIds":  [],
            "bulkActionIds":      []
          }
        },
        "layoutHints": {
          "primaryRegion": null,
          "sidebarRegion": null,
          "headerRegion":  null,
          "groupedSections": [
            {
              "groupKey": "supplier-step",
              "groupLabel": "Select Supplier",
              "sectionKeys": ["supplier_id", "supplier_contact"],
              "isCollapsedByDefault": false
            },
            {
              "groupKey": "items-step",
              "groupLabel": "Add Items",
              "sectionKeys": ["line_items", "total_amount"],
              "isCollapsedByDefault": false
            },
            {
              "groupKey": "delivery-step",
              "groupLabel": "Delivery Details",
              "sectionKeys": ["required_date", "delivery_address", "shipping_notes"],
              "isCollapsedByDefault": false
            }
          ]
        },
        "accessConditionHookId": null
      }
    ]
  },

  "actions": [
    {
      "actionId": "action-create-po",
      "actionType": "CreatePurchaseOrder",
      "label": "Create Purchase Order",
      "targetEntityType": "PurchaseOrder",
      "preconditions": [],
      "requiredInputs": [
        { "fieldKey": "supplier_id",  "fieldLabel": "Supplier",     "inputType": "select",      "isRequired": true, "validationRules": ["Must be an approved supplier"] },
        { "fieldKey": "line_items",   "fieldLabel": "Line Items",   "inputType": "multiselect", "isRequired": true, "validationRules": ["At least one line item required"] },
        { "fieldKey": "required_date","fieldLabel": "Required By",  "inputType": "date",        "isRequired": true, "validationRules": ["Must be at least 3 business days from today"] }
      ],
      "confirmationType": "none",
      "isDestructive": false,
      "requiresApproval": false,
      "postActionBehavior": "navigateTo",
      "postActionNavigateToViewId": "view-po-detail",
      "aiAssistHookIds": [],
      "telemetryKey": "procurement_coordinator.create_purchase_order"
    },
    {
      "actionId": "action-submit-po",
      "actionType": "SubmitPurchaseOrderForApproval",
      "label": "Submit for Approval",
      "targetEntityType": "PurchaseOrder",
      "preconditions": [
        {
          "conditionDescription": "PO must be in Draft status",
          "displayWhenNotMet": "disable_with_tooltip",
          "tooltipText": "This PO has already been submitted or is not in draft state"
        }
      ],
      "requiredInputs": [],
      "confirmationType": "inline",
      "isDestructive": false,
      "requiresApproval": true,
      "postActionBehavior": "stay",
      "postActionNavigateToViewId": null,
      "aiAssistHookIds": [],
      "telemetryKey": "procurement_coordinator.submit_po_for_approval"
    },
    {
      "actionId": "action-edit-po",
      "actionType": "EditPurchaseOrder",
      "label": "Edit",
      "targetEntityType": "PurchaseOrder",
      "preconditions": [
        {
          "conditionDescription": "PO must be in Draft or Rejected status",
          "displayWhenNotMet": "hide",
          "tooltipText": null
        }
      ],
      "requiredInputs": [],
      "confirmationType": "none",
      "isDestructive": false,
      "requiresApproval": false,
      "postActionBehavior": "navigateTo",
      "postActionNavigateToViewId": "view-create-po",
      "aiAssistHookIds": [],
      "telemetryKey": "procurement_coordinator.edit_purchase_order"
    },
    {
      "actionId": "action-cancel-po",
      "actionType": "CancelPurchaseOrder",
      "label": "Cancel PO",
      "targetEntityType": "PurchaseOrder",
      "preconditions": [
        {
          "conditionDescription": "PO must not be in Completed or already Cancelled status",
          "displayWhenNotMet": "hide",
          "tooltipText": null
        }
      ],
      "requiredInputs": [
        {
          "fieldKey": "cancellation_reason",
          "fieldLabel": "Reason for Cancellation",
          "inputType": "textarea",
          "isRequired": true,
          "validationRules": ["Minimum 10 characters"]
        }
      ],
      "confirmationType": "full",
      "isDestructive": true,
      "requiresApproval": false,
      "postActionBehavior": "navigateTo",
      "postActionNavigateToViewId": "view-po-list",
      "aiAssistHookIds": [],
      "telemetryKey": "procurement_coordinator.cancel_purchase_order"
    },
    {
      "actionId": "action-bulk-cancel",
      "actionType": "BulkCancelPurchaseOrders",
      "label": "Cancel Selected",
      "targetEntityType": "PurchaseOrder",
      "preconditions": [],
      "requiredInputs": [],
      "confirmationType": "full",
      "isDestructive": true,
      "requiresApproval": false,
      "postActionBehavior": "stay",
      "postActionNavigateToViewId": null,
      "aiAssistHookIds": [],
      "telemetryKey": "procurement_coordinator.bulk_cancel_purchase_orders"
    },
    {
      "actionId": "action-bulk-export",
      "actionType": "ExportPurchaseOrders",
      "label": "Export",
      "targetEntityType": "PurchaseOrder",
      "preconditions": [],
      "requiredInputs": [],
      "confirmationType": "none",
      "isDestructive": false,
      "requiresApproval": false,
      "postActionBehavior": "stay",
      "postActionNavigateToViewId": null,
      "aiAssistHookIds": [],
      "telemetryKey": "procurement_coordinator.export_purchase_orders"
    }
  ],

  "permissionsHooks": {
    "permissionsVersion": "opa-policy-v2.1",
    "allowedActionIds": [
      "action-create-po",
      "action-submit-po",
      "action-edit-po",
      "action-cancel-po",
      "action-bulk-cancel",
      "action-bulk-export"
    ],
    "visibleEntityTypes": ["PurchaseOrder", "Supplier", "Product"],
    "dataFilterPolicies": [
      {
        "policyId": "policy-own-pos-only",
        "description": "Show only POs created by or assigned to this user or their team",
        "appliesTo": "PurchaseOrder"
      }
    ],
    "conditionalAccessHooks": [
      {
        "hookId": "cond-po-edit-access",
        "description": "Edit is available only when PO is in Draft or Rejected status",
        "evaluationTiming": "runtime"
      }
    ]
  },

  "defaultViewHints": {
    "defaultLandingViewId": "view-po-list",
    "defaultDensityMode": "default",
    "defaultSortSpecs": [
      { "viewId": "view-po-list", "fieldKey": "created_date", "direction": "desc" }
    ],
    "defaultFilterSpecs": [
      { "viewId": "view-po-list", "fieldKey": "status", "filterDescription": "Open — excludes Completed and Cancelled" }
    ],
    "defaultExpandedSections": [
      { "viewId": "view-po-detail", "sectionKey": "po-basics" }
    ],
    "pinnedSummaryIds": ["summary-open-pos", "summary-pending-approval"]
  },

  "personalizationHooks": {
    "allowedSurfaces": [
      {
        "surfaceId": "surface-density",
        "surfaceType": "density_mode",
        "scope": "user",
        "description": "Switch between comfortable and compact density",
        "isStructuralVariant": false,
        "targetViewIds": [],
        "targetComponentTypes": [],
        "configurationSchema": [
          {
            "fieldKey": "density",
            "fieldLabel": "Display Density",
            "valueType": "enum",
            "allowedValues": ["default", "compact"],
            "defaultValue": "default"
          }
        ]
      },
      {
        "surfaceId": "surface-saved-filter",
        "surfaceType": "saved_filter",
        "scope": "user",
        "description": "Save and recall custom filter combinations on the Purchase Orders list",
        "isStructuralVariant": false,
        "targetViewIds": ["view-po-list"],
        "targetComponentTypes": ["FilterRail"],
        "configurationSchema": []
      },
      {
        "surfaceId": "surface-po-list-style",
        "surfaceType": "structural_variant",
        "scope": "tenant",
        "description": "Switch the PO list between semantic record list and compact grid",
        "isStructuralVariant": true,
        "targetViewIds": ["view-po-list"],
        "targetComponentTypes": ["RecordList"],
        "configurationSchema": [
          {
            "fieldKey": "list_style",
            "fieldLabel": "List Style",
            "valueType": "enum",
            "allowedValues": ["semantic_list", "compact_grid"],
            "defaultValue": "semantic_list"
          }
        ]
      }
    ],
    "conflictPolicy": "notify_and_preserve"
  },

  "completeness": {
    "sliceStatus": "partial",
    "resolvedCapabilities": ["tasks", "approvals", "stateSummaries"],
    "deferredCapabilities": ["alerts", "decisions"],
    "outOfScopeCapabilities": ["collaborators", "monitoredEntities"]
  },

  "tasks": [
    {
      "taskSpecId": "task-follow-up-delivery",
      "taskType": "FollowUpDelivery",
      "titleTemplate": "Follow up on delivery for {{po_number}}",
      "entityTypeRef": {
        "entityType": "PurchaseOrder",
        "displayNameTemplate": "PO {{po_number}} — {{supplier_name}}",
        "viewLink": "view-po-detail"
      },
      "stateModel": ["open", "in_progress", "resolved", "cancelled"],
      "priorityLevels": ["low", "normal", "high", "urgent"],
      "hasDueDate": true,
      "availableActionIds": ["action-cancel-po"],
      "aiAssistHookIds": []
    },
    {
      "taskSpecId": "task-review-quote",
      "taskType": "ReviewSupplierQuote",
      "titleTemplate": "Review quote from {{supplier_name}}",
      "entityTypeRef": null,
      "stateModel": ["open", "accepted", "rejected"],
      "priorityLevels": ["low", "normal", "high"],
      "hasDueDate": true,
      "availableActionIds": ["action-create-po"],
      "aiAssistHookIds": []
    }
  ],

  "approvals": [
    {
      "approvalSpecId": "approval-po-standard",
      "approvalType": "PurchaseOrderApproval",
      "subjectEntityType": "PurchaseOrder",
      "thisPositionRole": "initiator",
      "requiredApproverPositionIds": ["finance-manager"],
      "approvalDeadlineHours": 48,
      "showConsequenceOnReview": true,
      "rejectionRequiresReason": true,
      "approvalHistoryVisible": true,
      "linkedActionId": "action-submit-po",
      "linkedViewId": null
    }
  ],

  "stateSummaries": [
    {
      "summaryId": "summary-open-pos",
      "summaryType": "count",
      "label": "Open Purchase Orders",
      "entityType": "PurchaseOrder",
      "filterDescription": "Status is Open or Draft",
      "alertThreshold": null,
      "linkedViewId": "view-po-list",
      "displayOrder": 1
    },
    {
      "summaryId": "summary-pending-approval",
      "summaryType": "count",
      "label": "Pending Approval",
      "entityType": "PurchaseOrder",
      "filterDescription": "Status is Pending Approval",
      "alertThreshold": {
        "thresholdDescription": "More than 5 POs waiting over 24 hours",
        "severity": "warning"
      },
      "linkedViewId": "view-po-list",
      "displayOrder": 2
    },
    {
      "summaryId": "summary-committed-spend",
      "summaryType": "amount",
      "label": "Committed This Month",
      "entityType": "PurchaseOrder",
      "filterDescription": "Approved POs created in the current calendar month",
      "alertThreshold": null,
      "linkedViewId": null,
      "displayOrder": 3
    }
  ]
}
```

---

*This document is a companion example to Foundry Position Projection Schema v0 (Work Product 10.2). It is not normative — the schema document is authoritative. This example exists to make every field concrete and to show the reasoning behind each value choice.*
