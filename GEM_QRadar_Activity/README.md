# GEM QRadar Activity Log — Universal Cloud REST API Connector

## Overview

This connector retrieves **Activity Log** entries from the IBM Guardium Exposure Manager (GEM) SaaS API and forwards them as individual events to a QRadar **Universal Cloud REST API** log source.

Each run queries the GEM `POST /api/v3/reports/run` endpoint for the Activity Log report (`000000000000000000002001`), using a rolling time window driven by the recurrence schedule configured on the log source. The window is `[now - recurrence_minutes, now]`, so runs are contiguous with no gaps or overlaps.

---

## Files

| File | Purpose |
|------|---------|
| `GEM-ActivityLog-Workflow.xml` | UCC workflow — authentication, date-window calculation, pagination loop, event posting |
| `GEM-ActivityLog-WorkflowParameterValues.xml` | Default parameter values to fill in before importing |
| `GEM-ActivityLog-LogSourceExtension.xml` | Log Source Extension (LSX) — maps JSON numeric keys to QRadar fields and CEPs |
| `examples/example_curl.txt` | Reference curl command used to design the workflow |
| `examples/activity_result.json` | Sample API response showing the data structure |

---

## Prerequisites

- QRadar 7.4+ with **Universal Cloud REST API** protocol support
- A valid GEM API key and secret with access to the Activity Log report
- The Activity Log report ID (default: `000000000000000000002001`)

---

## Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `gem_host` | Full base URL of the GEM instance, e.g. `https://eu.guardium.security.ibm.com` | _(required)_ |
| `api_key` | GEM API key | _(required, secret)_ |
| `api_secret` | GEM API secret | _(required, secret)_ |
| `report_id` | Activity Log report ID | `000000000000000000002001` |
| `fetch_size` | Records returned per page | `500` |
| `recurrence_minutes` | **Must match** the log source recurrence schedule in QRadar (minutes) | `10` |

> ⚠️ `recurrence_minutes` must be kept in sync with the **Recurrence** field on the QRadar log source. The workflow uses this value to compute the `QUERY_FROM_DATE` / `QUERY_TO_DATE` window. If they diverge, records will be missed or duplicated.

---

## Installation

### 1 — Import the Workflow

1. In QRadar, navigate to **Admin → Universal Cloud REST API Sources**.
2. Click **Add** and select **Workflow**.
3. Upload `GEM-ActivityLog-Workflow.xml`.

### 2 — Configure Parameter Values

1. Edit `GEM-ActivityLog-WorkflowParameterValues.xml` and fill in:
   - `gem_host` — your GEM base URL
   - `api_key` — your GEM API key
   - `api_secret` — your GEM API secret
2. Upload the filled-in parameter values file as the **Workflow Parameter Values** for the log source.

### 3 — Create the Log Source

1. Navigate to **Admin → Log Sources → Add**.
2. Set **Log Source Type** to `Universal Cloud REST API`.
3. Set **Protocol** to `Universal Cloud REST API`.
4. Select the imported workflow and parameter values.
5. Set **Recurrence** to `10 minutes` (or your desired interval — update `recurrence_minutes` to match).
6. Save and deploy.

### 4 — Import the Log Source Extension

1. Navigate to **Admin → Log Source Extensions**.
2. Click **Add** and upload `GEM-ActivityLog-LogSourceExtension.xml`.
3. Enable it and associate it with the newly created log source.

### 5 — Create Custom Event Properties (CEPs)

In QRadar, navigate to **Admin → Custom Event Properties → Add** and create the following properties (or right-click an event in Log Activity → Extract Property):

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
GEM-ActivityLog-Workflow.xml
  ├─ Compute date window: [now - recurrence_minutes, now]
  ├─ POST /api/v3/reports/run  (offset=0, fetch_size=500)
  │      ├─ Process result.data[] → PostEvent per record
  │      └─ If count == fetch_size → advance offset, repeat
  └─ Log completion
       │
       ▼
QRadar Log Activity
  └─ GEM-ActivityLog-LogSourceExtension.xml maps fields
```

---

## API Response Field Mapping

The GEM API returns activity records with numeric string keys inside each `results` object:

| Key | Header Name | QRadar Standard Field | CEP Name |
|-----|-------------|----------------------|----------|
| `"1"` | CreationTimeUTC | StartTime | GEM Activity CreationTimeUTC |
| `"2"` | PerformedBy | UserName | GEM Activity PerformedBy |
| `"3"` | Context | — | GEM Activity Context |
| `"4"` | ActionTaken | EventName | GEM Activity ActionTaken |
| `"5"` | ContextDescription | — | GEM Activity ContextDescription |
| `"6"` | Activity | — | GEM Activity Activity |
| `"7"` | AuditTrailID | — | GEM Activity AuditTrailID |
