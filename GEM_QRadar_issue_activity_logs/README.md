# GEM QRadar Combined — Open Issues + Activity Log

## Overview

This connector combines the two GEM QRadar connectors into a **single Universal Cloud REST API log source**:

| Section | Source | Window |
|---------|--------|--------|
| **A — Open Issues** | `GET /api/v1/issues` → `GET /api/v1/issues/{id}/details` | Previous calendar day (yesterday midnight → today midnight UTC) |
| **B — Activity Log** | `POST /api/v3/reports/run` | Rolling window `[now − recurrence_minutes, now]` |

Both sections run in sequence every time the log source fires, sharing the same `gem_host`, `api_key`, and `api_secret` credentials.

A single **Log Source Extension (LSX)** uses two `<match-group>` elements to parse both event shapes:
- **Group 1** — triggered by the presence of `issue_id` (issue events)
- **Group 2** — triggered by the presence of numeric string key `"1"` (activity log events)

Because the two JSON shapes are mutually exclusive, the groups never conflict.

---

## Files

| File | Purpose |
|------|---------|
| `GEM-Combined-Workflow.xml` | UCC workflow — runs Issues (Section A) then Activity Log (Section B) each cycle |
| `GEM-Combined-WorkflowParameterValues.xml` | Default parameter values to fill in before importing |
| `GEM-Combined-LogSourceExtension.xml` | LSX — two match groups, one per event shape |

---

## Prerequisites

- QRadar 7.4+ with **Universal Cloud REST API** protocol support
- A GEM API key and secret with access to both the Issues API and the Activity Log report
- The Activity Log report ID (default: `000000000000000000002001`)

---

## Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `gem_host` | Hostname of the GEM instance — **no scheme, no trailing slash** e.g. `eu.guardium.security.ibm.com` | _(required)_ |
| `api_key` | GEM API key | _(required, secret)_ |
| `api_secret` | GEM API secret | _(required, secret)_ |
| `report_id` | Activity Log report ID | `000000000000000000002001` |
| `fetch_size` | Records per page for the Activity Log | `500` |
| `recurrence_minutes` | **Must match** the QRadar log source recurrence schedule | `10` |

> ⚠️ `recurrence_minutes` must stay in sync with the **Recurrence** field on the QRadar log source. The workflow uses this value to calculate the Activity Log time window. If they diverge, records will be missed or duplicated.

---

## Installation

### 1 — Import the Workflow

1. In QRadar, navigate to **Admin → Universal Cloud REST API Sources**.
2. Click **Add** and select **Workflow**.
3. Upload `GEM-Combined-Workflow.xml`.

### 2 — Configure Parameter Values

1. Edit `GEM-Combined-WorkflowParameterValues.xml` and fill in:
   - `gem_host` — your GEM hostname (e.g. `eu.guardium.security.ibm.com`)
   - `api_key` — your GEM API key
   - `api_secret` — your GEM API secret
2. Upload the filled-in file as the **Workflow Parameter Values** for the log source.

### 3 — Create the Log Source

1. Navigate to **Admin → Log Sources → Add**.
2. Set **Log Source Type** to `Universal Cloud REST API`.
3. Set **Protocol** to `Universal Cloud REST API`.
4. Select the imported workflow and parameter values.
5. Set **Recurrence** to `10 minutes` (or your desired interval — update `recurrence_minutes` to match).
6. Save and deploy.

### 4 — Import the Log Source Extension

1. Navigate to **Admin → Log Source Extensions**.
2. Click **Add** and upload `GEM-Combined-LogSourceExtension.xml`.
3. Enable it and associate it with the newly created log source.

### 5 — Create Custom Event Properties (CEPs)

In QRadar navigate to **Admin → Custom Event Properties → Add** and create all properties listed below, or right-click an event in Log Activity → Extract Property.

#### Issue CEPs

| CEP Name | Type | Description |
|----------|------|-------------|
| GEM Issue ID | Text | Internal numeric issue ID |
| GEM Sequential ID | Text | Human-readable ID e.g. ISS-0016 |
| GEM Issue Name | Text | Short issue title |
| GEM Issue Type | Text | Issue category / type |
| GEM Issue Executor | Text | User assigned to the issue |
| GEM Issue Status | Text | Issue status e.g. ISSUE_STATUS_OPEN |
| GEM Asset Name | Text | Name of the affected asset |
| GEM Asset ID | Text | Asset identifier |
| GEM Asset Type | Text | Asset type e.g. database |
| GEM Affected Resources Count | Text | Number of affected resources |
| GEM Policy ID | Text | ID of the triggering policy |
| GEM Policy Name | Text | Name of the triggering policy |
| GEM Date Detected | Text | ISO-8601 timestamp of detection |
| GEM Last Modified | Text | ISO-8601 timestamp of last modification |
| GEM Scope | Text | Scope name |
| GEM Scope ID | Text | Scope identifier |
| GEM Rule ID | Text | Rule that triggered the issue |
| GEM Data Classifications | Text | Data classification tags |
| GEM Sensitivity Category | Text | Sensitivity tag category |
| GEM Sensitivity Count | Text | Count for the sensitivity category |

#### Activity Log CEPs

| CEP Name | Type | Description |
|----------|------|-------------|
| GEM Activity CreationTimeUTC | Text | Timestamp the activity was created |
| GEM Activity PerformedBy | Text | User who performed the activity |
| GEM Activity Context | Text | Service context of the activity |
| GEM Activity ActionTaken | Text | Action taken |
| GEM Activity ContextDescription | Text | Detailed context description |
| GEM Activity Activity | Text | Detailed activity context |
| GEM Activity AuditTrailID | Text | Audit trail ID |

---

## Data Flow

```
QRadar UCC Scheduler
       │
       ▼
GEM-Combined-Workflow.xml
  │
  ├─ SECTION A — Open Issues
  │    ├─ Compute date window: [yesterday midnight, today midnight UTC]
  │    ├─ GET /api/v1/issues  (paginated, offset steps by 20)
  │    │      └─ For each issue → GET /api/v1/issues/{id}/details
  │    │              └─ PostEvent  issue_details/single  ──► Group 1 (LSX)
  │    └─ Loop until partial page
  │
  └─ SECTION B — Activity Log
       ├─ Compute rolling window: [now − recurrence_minutes, now]
       ├─ POST /api/v3/reports/run  (paginated, offset steps by fetch_size)
       │      └─ For each record → PostEvent  results object  ──► Group 2 (LSX)
       └─ Loop until partial page
              │
              ▼
       QRadar Log Activity
         ├─ Group 1 LSX → Issue event fields + CEPs
         └─ Group 2 LSX → Activity log fields + CEPs
```

---

## Event Discrimination

The LSX uses two `<match-group>` elements evaluated in order:

| Group | Order | Discriminator | Fires for |
|-------|-------|---------------|-----------|
| 1 | 1 | `json-matcher` on `"issue_id"` present | Issue events |
| 2 | 2 | `json-matcher` on `"1"` (CreationTimeUTC) present | Activity log events |

The two event shapes are structurally incompatible — an issue detail object never contains a key `"1"`, and an activity results object never contains `"issue_id"` — so the groups are guaranteed to be mutually exclusive.

---

## Comparison with Separate Connectors

| | Separate (2 log sources) | Combined (this connector) |
|--|--------------------------|---------------------------|
| Log sources in QRadar | 2 | **1** |
| API credential config | ×2 | **×1** |
| Recurrence schedules | 2 to maintain | **1** |
| LSX | 2 separate | **1 with 2 match groups** |
| CEPs | Identical set | Identical set |
| Event differentiation | By log source | By LSX match group |
