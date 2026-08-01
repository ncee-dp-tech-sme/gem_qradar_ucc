# GEM-Combined-LogSourceExtension — Field Mapping Reference

## Overview

The LSX parses two mutually exclusive JSON event shapes emitted by `GEM-Combined-Workflow.xml`:

| Match Group | Triggers when | Event source |
|---|---|---|
| 1 | JSON contains `issue_id` | Section A — Open Issues |
| 2 | JSON contains numeric key `"1"` | Section B — Activity Log |

`device-type-id-override` = **4026** (Universal Cloud REST API)

---

## Group 1 — Issue Events

Posted object: `issue_details/single`

### Standard QRadar fields

| QRadar field | JSON key |
|---|---|
| EventName | `issue_name` |
| EventCategory | `issue_type` |
| UserName | `issue_executor` |
| DeviceAddress | `asset_id` |

### Custom Event Properties (CEPs)

| CEP name | JSON key |
|---|---|
| GEM Issue ID | `issue_id` |
| GEM Sequential ID | `sequential_id` |
| GEM Issue Name | `issue_name` |
| GEM Issue Type | `issue_type` |
| GEM Issue Executor | `issue_executor` |
| GEM Issue Status | `issue_status` |
| GEM Asset Name | `asset_name` |
| GEM Asset ID | `asset_id` |
| GEM Asset Type | `asset_type` |
| GEM Affected Resources Count | `affected_resources_count` |
| GEM Policy ID | `policy_id` |
| GEM Policy Name | `policy_name` |
| GEM Date Detected | `date_detected` |
| GEM Last Modified | `last_modified` |
| GEM Scope | `scope` |
| GEM Scope ID | `scope_id` |
| GEM Rule ID | `rule_id` |
| GEM Data Classifications | `data_classifications` |
| GEM Sensitivity Category | `resource_classification_counts[0].sensitivity_tag_category` |
| GEM Sensitivity Count | `resource_classification_counts[0].count` |

---

## Group 2 — Activity Log Events

Posted object: `result.data[].results` (numeric string keys)

### Standard QRadar fields

| QRadar field | JSON key | Field name |
|---|---|---|
| EventName | `"4"` | ActionTaken |
| UserName | `"2"` | PerformedBy |
| StartTime | `"1"` | CreationTimeUTC |

### Custom Event Properties (CEPs)

| CEP name | JSON key | Field name |
|---|---|---|
| GEM Activity CreationTimeUTC | `"1"` | CreationTimeUTC |
| GEM Activity PerformedBy | `"2"` | PerformedBy |
| GEM Activity Context | `"3"` | Context |
| GEM Activity ActionTaken | `"4"` | ActionTaken |
| GEM Activity ContextDescription | `"5"` | ContextDescription |
| GEM Activity Activity | `"6"` | Activity |
| GEM Activity AuditTrailID | `"7"` | AuditTrailID |

---

## Creating CEPs in QRadar

Navigate to **Admin -> Custom Event Properties -> Add**, or right-click an event in
Log Activity and choose **Extract Property**. CEP names must match the `field`
attributes in the LSX exactly.
