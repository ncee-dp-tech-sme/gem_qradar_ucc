# Task: Combine GEM Activity Logs + GEM UCC Issues into one log source

Prompt: "Can you investigate if it is possible to combine the GEM activity logs and the GEM UCC Issues log source into one single logsource workflow. If yes, create a new folder GEM_QRadar_issue_activity_logs and create the workflows."

## Feasibility Analysis

**Yes — fully feasible.** Key findings:

1. Both connectors share the same three credentials (gem_host, api_key, api_secret).
2. Both connectors target the same QRadar log source type (4026 — Universal Cloud REST API).
3. The two event shapes (issue detail JSON vs. activity log results object) are structurally mutually exclusive:
   - Issue events always contain the field `issue_id`.
   - Activity log events always contain the numeric string key `"1"` (CreationTimeUTC).
   This enables reliable event discrimination inside a single LSX via two ordered match groups.
4. The QRadar LSX spec supports multiple `<match-group>` elements evaluated in order — this is the standard mechanism for handling multiple event shapes from one log source.
5. A single `<DoWhile>` loop per section can independently paginate each API without interference.

## What Was Created

Folder: GEM_QRadar_issue_activity_logs/

| File | Description |
|------|-------------|
| GEM-Combined-Workflow.xml | UCC workflow v1.0 — Section A (Open Issues, daily midnight window) followed by Section B (Activity Log, rolling recurrence_minutes window). Shared credentials setup. All API calls use https://${/gem_host}/... |
| GEM-Combined-WorkflowParameterValues.xml | Parameter defaults: report_id=000000000000000000002001, fetch_size=500, recurrence_minutes=10 |
| GEM-Combined-LogSourceExtension.xml | LSX with two match-groups: Group 1 (order=1) gates on issue_id for issue events; Group 2 (order=2) gates on key "1" for activity log events. All CEPs from both original extensions are preserved. |
| README.md | Full installation guide, parameter table, data flow diagram, event discrimination explanation, comparison table vs. two separate connectors |

## Key Design Decisions

- gem_host is hostname-only (no scheme) in both sections — consistent with the pre-flight Tests block (DNSResolutionTest, TCPConnectionTest, SSLHandshakeTest require bare hostname).
- Activity Log date window uses millisecond arithmetic on time() to produce a precise rolling window.
- Issues date window uses UTC midnight boundaries (yesterday→today) matching original behaviour.
- LSX match-group discrimination is structural (field presence), not content-based, making it robust and maintenance-free.
- All original CEPs from both connectors are retained unchanged — no breaking changes to existing QRadar dashboards/searches.
