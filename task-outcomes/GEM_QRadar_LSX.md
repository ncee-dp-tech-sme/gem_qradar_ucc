Prompt: Create a QRadar Log Source Extension so all objects in issue.json and issue_detail.json can be uploaded to QRadar as custom event properties for the Guardium Exposure Manager log source.

Date completed: 2026-08-01

---

## File created

| File | Purpose |
|---|---|
| `GEM_QRadar_UCC/GEM-LogSourceExtension.xml` | QRadar Log Source Extension — parses GEM JSON events and maps all fields to 20 Custom Event Properties |

## Files updated

| File | Change |
|---|---|
| `README.md` | Added LSX to repository structure, new "Custom Event Properties" section, updated Deployment steps (2→4) |

---

## Custom Event Properties registered (20 total)

| CEP Name | JSONPath | Type |
|---|---|---|
| gem_issue_id | $.issue_id | AlphaNumeric |
| gem_sequential_id | $.sequential_id | AlphaNumeric |
| gem_issue_name | $.issue_name | AlphaNumeric |
| gem_issue_type | $.issue_type | AlphaNumeric |
| gem_issue_executor | $.issue_executor | AlphaNumeric |
| gem_issue_status | $.issue_status | AlphaNumeric |
| gem_asset_name | $.asset_name | AlphaNumeric |
| gem_asset_id | $.asset_id | AlphaNumeric |
| gem_asset_type | $.asset_type | AlphaNumeric |
| gem_affected_resources_count | $.affected_resources_count | Numeric |
| gem_policy_id | $.policy_id | AlphaNumeric |
| gem_policy_name | $.policy_name | AlphaNumeric |
| gem_date_detected | $.date_detected | AlphaNumeric |
| gem_last_modified | $.last_modified | AlphaNumeric |
| gem_scope | $.scope | AlphaNumeric |
| gem_scope_id | $.scope_id | AlphaNumeric |
| gem_rule_id | $.rule_id | AlphaNumeric |
| gem_data_classifications | $.data_classifications | AlphaNumeric |
| gem_sensitivity_category | $.resource_classification_counts[0].sensitivity_tag_category | AlphaNumeric |
| gem_sensitivity_count | $.resource_classification_counts[0].count | Numeric |

---

## Deployment steps

1. Admin → Log Source Extensions → New Extension → upload GEM-LogSourceExtension.xml → Deploy
2. Admin → Log Sources → Add → set Log Source Extension to "Guardium Exposure Manager"
3. All 20 CEPs are available immediately in AQL and Log Activity after first events are received.
