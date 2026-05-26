# Verification Status Template

> **Version:** 1.0

Use this template for:
`ai-output/<basename>/<basename>-validation-status.md`

Keep content concise and user-readable.

```markdown
# Validation Status: {basename}

## Result
- status: {PASS|FAIL}
- decision_code: {READY_FOR_LAUNCHER|BLOCKED_FAILED_CHECKS|BLOCKED_COVERAGE}
- rest_file: {absolute-or-workspace-path}
- swagger_file: {absolute-or-workspace-path}

## Summary
- checks_passed: {number}
- checks_failed: {number}
- endpoint_coverage_percent: {number}
- coverage_warning: {true|false}

## Applied Fixes (validator changes)
| file | location | issue | fix_applied |
| ---- | -------- | ----- | ----------- |
| {path-or-none} | {line-or-scope} | {short issue} | {short deterministic fix} |

## Needs Review (not auto-fixed)
| scope | issue | why_not_auto_fixed | suggested_action |
| ----- | ----- | ------------------ | ---------------- |
| {entity/path/field} | {short issue} | {ambiguous/non-obvious} | {concise action} |

## Preserved User-Edited or Non-Derivable Valid Content
| scope | preserved_item | reason_preserved | user_check |
| ----- | -------------- | ---------------- | ---------- |
| {entity/path/field} | {short description} | {valid but not swagger-derivable} | {confirm intent} |

## Focused Re-checks from Generation Status
| flagged_area | final_result | notes |
| ------------ | ------------ | ----- |
| {item-or-none} | {PASS|FAIL|N/A} | {short explanation} |

## Quality Gate
```yaml
quality_gate:
  gate_status: PASS|FAIL
  failed_checks: {number}
  max_failed_checks: {number}
  mapped_endpoints: {number}
  swagger_endpoints_total: {number}
  endpoint_coverage_percent: {number}
  min_endpoint_coverage_percent: {number}
  has_coverage_warning: {true|false}
  allow_coverage_warnings: {true|false}
  decision_code: {READY_FOR_LAUNCHER|BLOCKED_FAILED_CHECKS|BLOCKED_COVERAGE}
```
```
