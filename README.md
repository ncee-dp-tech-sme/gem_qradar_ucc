# GEM QRadar Universal Cloud Connectors

https://github.com/ncee-dp-tech-sme/gem_qradar_ucc

A collection of QRadar **Universal Cloud REST API** connectors for **IBM Guardium Exposure Manager (GEM)**.

| Connector | Folder | Description |
|---|---|---|
| Open Issues + Details | [`GEM_QRadar_UCC/`](https://github.com/ncee-dp-tech-sme/gem_qradar_ucc/tree/main/GEM_QRadar_UCC/) | Retrieves open issues and their full details on a daily schedule |
| Activity Log | [`GEM_QRadar_Activity/`](https://github.com/ncee-dp-tech-sme/gem_qradar_ucc/tree/main/GEM_QRadar_Activity/) | Retrieves activity log entries on a rolling 10-minute schedule |
| **Combined — Issues + Activity Log** | [`GEM_QRadar_issue_activity_logs/`](https://github.com/ncee-dp-tech-sme/gem_qradar_ucc/tree/main/GEM_QRadar_issue_activity_logs/) | Single log source combining both of the above |

---

## Repository structure

```
GEM_QRADAR_UCC/
├── GEM_QRadar_UCC/                                         # Connector 1 — Open Issues + Details
│   ├── GEM-Workflow.xml                                    # Workflow definition
│   ├── GEM-WorkflowParameterValues.xml                    # Parameter values template
│   └── GEM-LogSourceExtension.xml                         # QRadar Log Source Extension (LSX)
├── GEM_QRadar_Activity/                                    # Connector 2 — Activity Log
│   ├── GEM-ActivityLog-Workflow.xml                        # Workflow definition
│   ├── GEM-ActivityLog-WorkflowParameterValues.xml         # Parameter values template
│   ├── GEM-ActivityLog-LogSourceExtension.xml              # QRadar Log Source Extension (LSX)
│   ├── README.md                                           # Activity Log connector documentation
│   └── examples/
│       ├── example_curl.txt                                # Reference curl command
│       └── activity_result.json                            # Sample API response
├── GEM_QRadar_issue_activity_logs/                         # Connector 3 — Combined (Issues + Activity Log)
│   ├── GEM-Combined-Workflow.xml                           # Combined workflow definition
│   ├── GEM-Combined-WorkflowParameterValues.xml            # Parameter values template
│   ├── GEM-Combined-LogSourceExtension.xml                 # QRadar Log Source Extension (LSX)
│   ├── GEM-Combined-LogSourceExtension-FieldMapping.md     # CEP field mapping reference
│   └── README.md                                           # Combined connector documentation
├── examples/
│   ├── issue.json                                          # Sample GEM API issues-list response
│   └── issue_detail.json                                   # Sample GEM API issue-detail response
├── workflows/
│   ├── Workflow-v2.xsd                                     # QRadar workflow XML schema
│   └── WorkflowParameterValues-v2.xsd                      # QRadar parameter values XML schema
├── prerequisites.txt                                       # API endpoint reference notes
└── README.md                                               # This file
```

---

## Connector 1 — Open Issues + Details

> **Current version: 1.2**

Runs on a configurable schedule (minimum once per day) and performs the following for each execution:

1. Computes a 24-hour window covering the **previous UTC calendar day** at runtime.
2. Queries `GET /api/v1/issues` with pagination (20 issues per page) to retrieve all open issues detected during that window.
3. For each issue, calls `GET /api/v1/issues/{issue_id}/details` to fetch the enriched detail record.
4. Posts each detail record as an individual event to the QRadar log source.

**Files:** [`GEM_QRadar_UCC/`](GEM_QRadar_UCC/)

---

## Connector 2 — Activity Log

> **Current version: 1.1**

Runs on a configurable recurring schedule (default: every 10 minutes) and performs the following for each execution:

1. Computes a rolling time window `[now - recurrence_minutes, now]`.
2. Queries `POST /api/v3/reports/run` with pagination (default: 500 records per page) for the Activity Log report.
3. Posts each activity record as an individual event to the QRadar log source.

**Files:** [`GEM_QRadar_Activity/`](GEM_QRadar_Activity/) — see [`GEM_QRadar_Activity/README.md`](GEM_QRadar_Activity/README.md) for full documentation.

---

## Connector 3 — Combined Issues + Activity Log

> **Current version: 1.0**

Combines Connector 1 and Connector 2 into a **single log source** — one set of credentials, one recurrence schedule, one Log Source Extension. Each run executes both sections in sequence:

- **Section A — Open Issues:** same daily midnight-window logic as Connector 1.
- **Section B — Activity Log:** same rolling `recurrence_minutes` window logic as Connector 2.

The LSX uses two `match-group` elements (device-type-id `4015` by default but after creating the logsource, change it to your logsource id) to parse the two mutually exclusive event shapes. Events are disambiguated structurally: issue events contain `issue_id`; activity log events contain numeric string key `"1"`.

**Files:** [`GEM_QRadar_issue_activity_logs/`](GEM_QRadar_issue_activity_logs/) — see [`GEM_QRadar_issue_activity_logs/README.md`](GEM_QRadar_issue_activity_logs/README.md) for full documentation.

---

## Prerequisites

| Requirement | Details |
|---|---|
| QRadar version | 7.4.x or later with Universal Cloud REST API protocol support |
| GEM instance | Accessible over HTTPS from the QRadar console / managed host |
| GEM API credentials | An API key and API secret with read access to Issues and Activity Log |

---

## Authentication

All connectors use **HTTP Basic Authentication**. The credential string `api_key:api_secret` is assembled at runtime and base64-encoded inline:

```
authorization: Basic <base64(api_key:api_secret)>
```

No pre-encoded value is stored. Credential rotation only requires updating the two parameter values on the log source.

---

## Schema references

Workflow XML files conform to the QRadar Universal Cloud REST API V2 schemas in [`workflows/`](workflows/):

| Schema file | Applies to |
|---|---|
| `workflows/Workflow-v2.xsd` | All `*-Workflow.xml` files |
| `workflows/WorkflowParameterValues-v2.xsd` | All `*-WorkflowParameterValues.xml` files |

---

## Change log

### Connector 1 — Open Issues + Details

| Version | Date | Description |
|---|---|---|
| 1.1 | 2026-08-01 | Initial release — paginated issue list retrieval + per-issue detail fetch |
| 1.2 | 2026-08-01 | Bug fix: `date_filter` query parameter was manually percent-encoded causing double-encoding and HTTP 500. Fixed by using plain XML-escaped JSON. |

### Connector 2 — Activity Log

| Version | Date | Description |
|---|---|---|
| 1.0 | 2026-08-01 | Initial release — paginated activity log retrieval with rolling 10-minute date window |
| 1.1 | 2026-08-01 | Bug fix: `gem_host` changed to hostname-only (no scheme). Resolves Client Protocol Exception from pre-flight test elements. |

### Connector 3 — Combined Issues + Activity Log

| Version | Date | Description |
|---|---|---|
| 1.0 | 2026-08-02 | Initial release — combined workflow and LSX for Issues + Activity Log under a single QRadar log source. LSX uses device-type-id 4015 (Guardium Exposure Manager Issues and Activities). |
