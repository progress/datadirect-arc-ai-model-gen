# Generation Status Template

> **Version:** 1.0

Use this template for `ai-output/{fileName}/{fileName}-generation-status.md` on every generator run.

```yaml
generation_result: {COMPLETE|PARTIAL|FAILED}
output_file: ai-output/{fileName}/{fileName}.rest
swagger_endpoints_found: {swagger_endpoint_count}
rest_endpoints_modeled: {rest_modeled_endpoint_count}
unmapped_or_excluded_endpoints: {unmapped_or_excluded_count}
paths_processed: {paths_count}
entities_generated: {entity_count}
entities_skipped: {none|comma-separated-entity-names}
validation_status: NOT RUN
```

Output is LLM-generated and may vary between runs. Validator pass/fail is the authoritative structural check.

## Table of Contents

- [Mandatory User Review Items](#mandatory-user-review-items)
- [Unmapped or Excluded Endpoints](#unmapped-or-excluded-endpoints)
- [Assumptions Detected](#assumptions-detected)
- [Next Steps](#next-steps)
- [Notes](#notes)
- [Generation Fingerprint](#generation-fingerprint)

## Mandatory User Review Items

Use this section to review the important decisions made during generation. For each entity, confirm the decision and make any `.rest` changes as needed.

- Formatting rules:
  
  - Use a compact table for each review item with columns: `entity`, `decision`, `action`.
  - Keep each entity row to a single line with concise wording.

- JSONRoot:
  
  - status: REQUIRED
  
  - documentation: [JSONRoot](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/JSONRoot.html)
  
  - notes:
    
    | entity          | decision              | action                                           |
    | --------------- | --------------------- | ------------------------------------------------ |
    | {entity_name_1} | {jsonroot_decision_1} | {jsonroot_action_1_using_/path_/jsonRoot_syntax} |
    | {entity_name_2} | {jsonroot_decision_2} | {jsonroot_action_2_using_/path_/jsonRoot_syntax} |

- Keys (`#key` / `#rowid`):
  
  - status: REQUIRED
  
  - documentation: [Determining the primary key](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/Determining-the-primary-key.html)
  
  - notes:
    
    | entity          | decision                 | action                 |
    | --------------- | ------------------------ | ---------------------- |
    | {entity_name_1} | {primary_key_decision_1} | {primary_key_action_1} |
    | {entity_name_2} | {primary_key_decision_2} | {primary_key_action_2} |

- Pagination:
  
  - status: REQUIRED
  
  - documentation: [Paging](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/Paging.html)
  
  - notes:
    
    | entity          | decision                | action                |
    | --------------- | ----------------------- | --------------------- |
    | {entity_name_1} | {pagination_decision_1} | {pagination_action_1} |
    | {entity_name_2} | {pagination_decision_2} | {pagination_action_2} |

- Auth (`#options.authenticationMethod`):
  
  - status: {REQUIRED|REQUIRED_IF_APPLICABLE}
  
  - documentation: [Authentication](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/Authentication.html)
  
  - notes:
    
    | entity          | decision                            | action                            |
    | --------------- | ----------------------------------- | --------------------------------- |
    | {entity_name_1} | {authentication_options_decision_1} | {authentication_options_action_1} |
    | {entity_name_2} | {authentication_options_decision_2} | {authentication_options_action_2} |

- Hostname (`#hostname`):
  
  - status: {REQUIRED|REQUIRED_IF_APPLICABLE}
  
  - documentation: [Server Name (alias)](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/ServerName.html)
  
  - notes:
    
    | entity          | decision                           | action                           |
    | --------------- | ---------------------------------- | -------------------------------- |
    | {entity_name_1} | {hostname_substitution_decision_1} | {hostname_substitution_action_1} |
    | {entity_name_2} | {hostname_substitution_decision_2} | {hostname_substitution_action_2} |

For manual modifications that may be required based on your API, see `.github/agents/docs/manual-rest-adjustments.md`.

## Unmapped or Excluded Endpoints

Use this section to see API coverage gaps at a glance. It should list every endpoint that is missing from the generated `.rest` file, including both intentionally excluded paths and paths that failed to map.

Status values:

- `EXCLUDED_VALID`: legitimate non-relational/utility exclusion (scalar field-accessor endpoint, batch utility, upload-only endpoint, or mixed heterogeneous payload that cannot be modeled as a stable relational entity)
- `EXCLUDED_REVIEW`: endpoint returns a resource object/array and likely should be modeled (existing entity path or new entity)
- `FAILED_TO_MAP`: mapping attempted but generation could not resolve deterministically

| Endpoint       | Method          | Status                                        | Reason             | Action             |
| -------------- | --------------- | --------------------------------------------- | ------------------ | ------------------ |
| {path-or-none} | {method-or-n/a} | {EXCLUDED_VALID_OR_EXCLUDED_REVIEW_OR_FAILED_TO_MAP_OR_NOT_APPLICABLE} | {reason-or-"none"} | {action-or-"none"} |

## Assumptions Detected

Use this section to verify assumptions the generator made (for auth, paging, JSON roots, and hostname). Confirm each one or update your `.rest` file if an assumption is incorrect.

```yaml
assumptions:
  authentication: {assumption_authentication}
  pagination: {assumption_pagination}
  jsonroot: {assumption_jsonroot}
  hostname: {assumption_hostname}
```

## Next Steps

Use this checklist to move from generated output to a validated `.rest` file.

1. Review and edit `ai-output/{fileName}/{fileName}.rest` based on the Mandatory User Review Items.
2. Run the ARCGenAI-StaticValidator with the generated `.rest` file and source OpenAPI file.
3. If issues are found, update the `.rest` file and re-run validation.
4. If temporary assembly files remain, delete `ai-output/{fileName}/entities/*.entity.tmp` and `ai-output/{fileName}/entities/_header.assembly.tmp` (safe cleanup when automatic deletion is unavailable).

## Notes

- `generation_result` meanings:
  - `COMPLETE`: all planned entities generated and assembled
  - `PARTIAL`: one or more entities skipped; output assembled with available entities
  - `FAILED`: assembly or generation failed before usable output
- `validation_status` is `NOT RUN` when generated initially.
- After ARCGenAI-StaticValidator runs, `validation_status` should be updated to `RUN`.

## Generation Fingerprint

```yaml
swagger_source_filename: {swagger_source_filename}
generated_timestamp: {timestamp_iso8601}
generator_agent_version: {generator_agent_version}
total_endpoint_count: {swagger_endpoint_count}
```
