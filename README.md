# GEM QRadar Universal Cloud Connectors

A collection of QRadar **Universal Cloud REST API** connectors for **IBM Guardium Exposure Manager (GEM)**.

| Connector | Folder | Description |
|---|---|---|
| Open Issues + Details | [`GEM_QRadar_UCC/`](GEM_QRadar_UCC/) | Retrieves open issues and their full details on a daily schedule |
| Activity Log | [`GEM_QRadar_Activity/`](GEM_QRadar_Activity/) | Retrieves activity log entries on a rolling 10-minute schedule |

---

---

## Connector 1 — Open Issues + Details

> **Current version: 1.2** — see [Change log](#change-log) for details.

### Overview

Runs on a configurable schedule (minimum once per day) and performs the following actions for each execution:

1. Computes a 24-hour window covering the **previous UTC calendar day** at runtime.
2. Queries `GET /api/v1/issues` with pagination (20 issues per page) to retrieve all open issues detected during that window.
3. For each issue, calls `GET /api/v1/issues/{issue_id}/details` to fetch the enriched detail record.
4. Posts each detail record as an individual event to the QRadar log source.

---

## Repository structure

```
GEM_QRADAR_UCC/
├── GEM_QRadar_UCC/                               # Connector 1 — Open Issues + Details
│   ├── GEM-Workflow.xml                          # Workflow definition
│   ├── GEM-WorkflowParameterValues.xml           # Parameter values template
│   └── GEM-LogSourceExtension.xml                # QRadar Log Source Extension (LSX)
├── GEM_QRadar_Activity/                          # Connector 2 — Activity Log
│   ├── GEM-ActivityLog-Workflow.xml              # Workflow definition
│   ├── GEM-ActivityLog-WorkflowParameterValues.xml  # Parameter values template
│   ├── GEM-ActivityLog-LogSourceExtension.xml    # QRadar Log Source Extension (LSX)
│   ├── README.md                                 # Activity Log connector documentation
│   └── examples/
│       ├── example_curl.txt                      # Reference curl command
│       └── activity_result.json                  # Sample API response
├── examples/
│   ├── issue.json                                # Sample GEM API issues-list response
│   └── issue_detail.json                         # Sample GEM API issue-detail response
├── workflows/
│   ├── Workflow-v2.xsd                           # QRadar workflow XML schema
│   └── WorkflowParameterValues-v2.xsd            # QRadar parameter values XML schema
├── prerequisites.txt                             # API endpoint reference notes
└── README.md                                     # This file
```

---

---

## Prerequisites

| Requirement | Details |
|---|---|
| QRadar version | 7.4.x or later with Universal Cloud REST API protocol support |
| GEM instance | Accessible over HTTPS from the QRadar console / managed host |
| GEM API credentials | An API key and API secret with at least read access to Issues |

---

## Parameters

These values are configured in [`GEM-WorkflowParameterValues.xml`](GEM_QRadar_UCC/GEM-WorkflowParameterValues.xml) before deployment, or entered directly in the QRadar log source wizard. For the Activity Log connector parameters, see [`GEM_QRadar_Activity/README.md`](GEM_QRadar_Activity/README.md).

| Parameter | Label | Required | Secret | Description |
|---|---|---|---|---|
| `gem_host` | GEM Host (base URL) | Yes | No | Hostname of the GEM instance — **no** `https://` prefix and **no** trailing slash. Example: `gem.example.com` |
| `api_key` | API Key | Yes | Yes | GEM API key |
| `api_secret` | API Secret | Yes | Yes | GEM API secret paired with the key above |

---

## Authentication

The connector uses **HTTP Basic Authentication**. The credential string `apikey:apisecret` is assembled at runtime and base64-encoded inline using the built-in `base64_encode()` function:

```
authorization: Basic <base64(api_key:api_secret)>
```

No pre-encoded value is stored. The encoding happens at each request, so credential rotation only requires updating the two parameter values.

---

## API endpoints used

### List open issues

```
GET https://{gem_host}/api/v1/issues
```

Query parameters sent on every request:

| Parameter | Value |
|---|---|
| `offset` | Current pagination offset (starts at `0`, increments by `20`) |
| `limit` | `20` |
| `filters.status[]` | `ISSUE_STATUS_OPEN` |
| `filters.date_detected` | JSON: `{"preset":"custom","from":"<yesterday_midnight_UTC>","to":"<today_midnight_UTC>"}` — URL-encoded automatically by the UCC framework |

Example decoded URL for reference:
```
/api/v1/issues?offset=0&limit=20
  &filters.status[]=ISSUE_STATUS_OPEN
  &filters.date_detected={"preset":"custom","from":"2026-07-31T00:00:00Z","to":"2026-08-01T00:00:00Z"}
```

### Get issue details

```
GET https://{gem_host}/api/v1/issues/{issue_id}/details
```

Called once per issue retrieved from the list endpoint. The `issue_id` is read from the `single.issue_id` field of each issue object in the list response.

---

## Workflow logic

```
Set limit=20, offset=0
Compute yesterday_midnight and today_midnight (UTC, millisecond epoch)
Format both as ISO-8601 strings
Build date filter JSON string (UCC framework handles URL encoding)
Assemble credentials string (api_key:api_secret)

DoWhile (count of issues returned == 20):
  GET /api/v1/issues  [offset, limit, status filter, date filter]
  If status != 200 → Log error + Abort

  If issues returned > 0:
    ForEach issue:
      GET /api/v1/issues/{issue.single.issue_id}/details
      If status != 200 → Log error + skip this issue
      Else → PostEvent (detail body) to QRadar

  offset = offset + 20

Log completion
```

### Pagination termination

The `DoWhile` loop exits when the API returns fewer than 20 issues (a partial or empty page), indicating that all matching issues have been retrieved.

---

## Event structure

Each QRadar event contains the `issue_details.single` object from `GET /api/v1/issues/{issue_id}/details`. The API response envelope (`status`, `issue_details` wrapper) is stripped — only the inner record is posted.

Example event payload:

```json
{
  "issue_id": "16",
  "sequential_id": "ISS-0016",
  "issue_name": "SECRETS sensitivities are violating project Dev Knowledge Manager policy",
  "issue_type": "ISSUE_TYPE_VIOLATION",
  "issue_executor": "ISSUE_EXECUTOR_WORKLOAD",
  "asset_name": "medtech-dev-docs",
  "asset_id": "arn:aws:s3:::medtech-dev-docs",
  "asset_type": "ASSET_TYPE_DATASTORE",
  "affected_resources_count": 4,
  "resource_classification_counts": [
    { "sensitivity_tag_category": "SECRETS", "count": 2 }
  ],
  "issue_status": "ISSUE_STATUS_OPEN",
  "policy_id": "bc7be97c-ffa7-45f0-b8e1-dcbf19f14fed",
  "policy_name": "Dev Workload Policy",
  "date_detected": "2026-07-31T01:00:34.809693Z",
  "last_modified": "2026-08-01T01:00:29.269356Z",
  "scope": "Dev Knowledge Manager",
  "scope_id": "proj-dev-kb-001",
  "timeline": [...],
  "rule_id": "aec078c2-ea90-49cb-a915-bf8b7de94b60",
  "data_classifications": ["SECRETS"]
}
```

The log source identifier (QRadar `source`) is set to `gem_host`.

---

## Custom Event Properties

The [`GEM-LogSourceExtension.xml`](GEM_QRadar_UCC/GEM-LogSourceExtension.xml) registers 20 Custom Event Properties (CEPs) mapped via JSONPath from the event payload:

| CEP Name | JSONPath | Label | Type |
|---|---|---|---|
| `gem_issue_id` | `$.issue_id` | GEM Issue ID | AlphaNumeric |
| `gem_sequential_id` | `$.sequential_id` | GEM Sequential ID | AlphaNumeric |
| `gem_issue_name` | `$.issue_name` | GEM Issue Name | AlphaNumeric |
| `gem_issue_type` | `$.issue_type` | GEM Issue Type | AlphaNumeric |
| `gem_issue_executor` | `$.issue_executor` | GEM Issue Executor | AlphaNumeric |
| `gem_issue_status` | `$.issue_status` | GEM Issue Status | AlphaNumeric |
| `gem_asset_name` | `$.asset_name` | GEM Asset Name | AlphaNumeric |
| `gem_asset_id` | `$.asset_id` | GEM Asset ID | AlphaNumeric |
| `gem_asset_type` | `$.asset_type` | GEM Asset Type | AlphaNumeric |
| `gem_affected_resources_count` | `$.affected_resources_count` | GEM Affected Resources Count | Numeric |
| `gem_policy_id` | `$.policy_id` | GEM Policy ID | AlphaNumeric |
| `gem_policy_name` | `$.policy_name` | GEM Policy Name | AlphaNumeric |
| `gem_date_detected` | `$.date_detected` | GEM Date Detected | AlphaNumeric |
| `gem_last_modified` | `$.last_modified` | GEM Last Modified | AlphaNumeric |
| `gem_scope` | `$.scope` | GEM Scope | AlphaNumeric |
| `gem_scope_id` | `$.scope_id` | GEM Scope ID | AlphaNumeric |
| `gem_rule_id` | `$.rule_id` | GEM Rule ID | AlphaNumeric |
| `gem_data_classifications` | `$.data_classifications` | GEM Data Classifications | AlphaNumeric |
| `gem_sensitivity_category` | `$.resource_classification_counts[0].sensitivity_tag_category` | GEM Sensitivity Category | AlphaNumeric |
| `gem_sensitivity_count` | `$.resource_classification_counts[0].count` | GEM Sensitivity Count | Numeric |

CEPs must be created in QRadar first (**Admin → Custom Event Properties → Add**), using the exact property names from the table above. Alternatively, once events are flowing, you can right-click an event in **Log Activity → event details** and use **Extract Property** to create them interactively. The LSX `json-matcher` field values reference those names to populate the properties automatically.

---

## Deployment

### 1. Fill in parameter values

Open [`GEM_QRadar_UCC/GEM-WorkflowParameterValues.xml`](GEM_QRadar_UCC/GEM-WorkflowParameterValues.xml) and set the three values:

```xml
<Value name="gem_host"   value="gem.example.com" />
<Value name="api_key"    value="your-api-key" />
<Value name="api_secret" value="your-api-secret" />
```

### 2. Install the Log Source Extension

1. Go to **Admin → Log Source Extensions → New Extension**.
2. Upload [`GEM-LogSourceExtension.xml`](GEM_QRadar_UCC/GEM-LogSourceExtension.xml) and click **Deploy**.
3. QRadar loads the extension. CEPs still need to be created separately — see the **Custom Event Properties** section above.

### 3. Create a log source in QRadar

1. Go to **Admin → Log Sources → Add**.
2. Set **Log Source Type** to `Universal Cloud REST API`.
3. Set **Protocol** to `Universal Cloud REST API`.
4. Upload [`GEM-Workflow.xml`](GEM_QRadar_UCC/GEM-Workflow.xml) as the workflow.
5. Upload [`GEM-WorkflowParameterValues.xml`](GEM_QRadar_UCC/GEM-WorkflowParameterValues.xml) as the parameter values.
6. Set **Log Source Extension** to `Guardium Exposure Manager`.
7. Set the **Recurrence** to `1440` minutes (once per day) or adjust to your needs.
8. Save and deploy the log source.

### 4. Verify connectivity

QRadar will run the built-in pre-flight tests (DNS resolution, TCP connection, SSL handshake, proxy connectivity) against `gem_host` before the first collection cycle.

---

## Error handling

| Scenario | Behaviour |
|---|---|
| Non-200 response from the issues list endpoint | Log `ERROR` with status code and message, then `Abort` the workflow run |
| Non-200 response from a single issue detail endpoint | Log `ERROR` with the issue ID and status, then **skip** that issue and continue |
| Empty page (0 issues returned) | `DoWhile` exits cleanly; nothing is posted; run completes successfully |

---

## Schema references

Both XML files conform to the QRadar Universal Cloud REST API V2 schemas located in [`workflows/`](workflows/):

| File | Schema |
|---|---|
| `GEM-Workflow.xml` | `workflows/Workflow-v2.xsd` — namespace `http://qradar.ibm.com/UniversalCloudRESTAPI/Workflow/V2` |
| `GEM-WorkflowParameterValues.xml` | `workflows/WorkflowParameterValues-v2.xsd` — namespace `http://qradar.ibm.com/UniversalCloudRESTAPI/WorkflowParameterValues/V2` |

---

## Change log

### Open Issues + Details connector

| Version | Date | Description |
|---|---|---|
| 1.1 | 2026-08-01 | Initial release — paginated issue list retrieval + per-issue detail fetch |
| 1.2 | 2026-08-01 | **Bug fix:** `date_filter` query parameter was manually percent-encoded, causing the UCC framework to double-encode it (`%7B` → `%257B`), which produced a malformed filter and an HTTP 500 from the GEM API. Fixed by using plain XML-escaped JSON — the UCC framework now handles the single-pass URL encoding correctly. |

### Activity Log connector

| Version | Date | Description |
|---|---|---|
| 1.0 | 2026-08-01 | Initial release — paginated activity log retrieval with rolling 10-minute date window |
| 1.1 | 2026-08-01 | **Bug fix:** `gem_host` changed to hostname-only (no `https://` scheme). Resolves `Client Protocol Exception: null` caused by passing a full URL to DNS/TCP/SSL pre-flight test elements. |
