# Validation Status: yelp-api

## Result
- status: PASS
- decision_code: READY_FOR_LAUNCHER
- rest_file: ai-output/yelp-api/yelp-api.rest
- swagger_file: input/swagger/yelp-api.yaml

## Summary
- checks_passed: 28
- checks_failed: 0
- endpoint_coverage_percent: 100
- coverage_warning: false

## Applied Fixes (validator changes)

| file | location | issue | fix_applied |
| ---- | -------- | ----- | ----------- |
| yelp-api.rest | businesses.location.address1/2/3, city, state, country | Duplicate top-level `#virtual` fields existed for fields already present as sub-properties of the `location` response object — violation of R-FC-005/R-PM-001b (MUST). These sub-fields must use the dual-role data column pattern, not separate virtuals. | Added `#eq` inline to each `location` sub-field (address1, address2, address3, city, state, country); removed 6 duplicate top-level virtual blocks. |
| yelp-api.rest | service_offerings entity (missing) | Endpoint `GET /v3/businesses/{business_id_or_alias}/service_offerings` was excluded from generation because its swagger schema is `type: object` with no defined properties. R-RM-013 (MUST) requires that even opaque/empty-schema endpoints include a `data: JSON` fallback field — they must not be fully excluded. | Added `service_offerings` entity with `"data": "JSON"` fallback and `business_id_or_alias` mandatory virtual. Coverage now 16/16 = 100%. |

## Needs Review (not auto-fixed)

| scope | issue | why_not_auto_fixed | suggested_action |
| ----- | ----- | ------------------ | ---------------- |
| autocomplete.text (virtual) | Typed `VarChar(256)` but field is named `text`. R-TM-002 says fields named `text` → `LongVarChar`. This is a filter virtual, not a response content field; the rule intent may not apply here. | Ambiguous: R-TM-002 lists `text` → LongVarChar but the rule is oriented toward response content fields; applying it to a search-input virtual would make `WHERE text LIKE '%…%'` harder to use. | Accept `VarChar(256)` if the autocomplete search text will always be short (typical search input). Change to `LongVarChar` if the API accepts very long free-text inputs. |
| ai_chat — #path POST + #insert POST both target same endpoint | Both `#path: ["POST /ai/chat/v2"]` and `#insert: "POST /ai/chat/v2"` reference the same endpoint. SELECT and INSERT both POST to `/ai/chat/v2`. This is intentional for POST-read APIs but uncommon. | Non-obvious; this may or may not be supported depending on ARC version and connector configuration. | Verify that ARC correctly resolves dual-use of the same POST endpoint for both SELECT and INSERT. If ARC cannot distinguish the two, remove `#path` (reads) or `#insert` (writes) as appropriate. |
| businesses — `#maximumPageSize: 50` applies to `/matches` path | The `/v3/businesses/matches` endpoint caps at 10 results (`limit` max 10), but the merged entity uses `#maximumPageSize: 50` inherited from the search paths. | Cannot auto-fix without changing entity structure; would require path-specific paging or splitting the entity. | Change `#maximumPageSize` to 10 (the lowest cap across all merged paths), or split `/matches` into its own entity with a separate `#maximumPageSize: 10`. |
| service_offerings — no `#key` field | Added entity uses `data: JSON` fallback with no primary key. R-FC-001 says not to invent a key when none is derivable. The IDENTITY path returns one row per business. | R-RM-013 canonical example also shows no key for opaque entities; adding a synthetic ROWID would work but is not required by spec. | Add `"ROWID": "VarChar(32)=rowid(),#key"` if ARC requires a key for IDENTITY queries. Otherwise leave as-is. |

## Preserved User-Edited or Non-Derivable Valid Content

| scope | preserved_item | reason_preserved | user_check |
| ----- | -------------- | ---------------- | ---------- |
| businesses | `alias` is a plain data column (not `#key`); `id` carries `#key` | R-FC-001: "id always beats alias/slug". `alias` is mutable; `id` is the stable immutable identifier. `business_id_or_alias` path param handled by a separate mandatory virtual. | Confirm `id` is always returned in all path variants. |
| review_highlights | `business_id` used as `#key` | The inline ReviewHighlightsResponse schema has no `id` field. `business_id` is the only stable identifier present in the response. | Confirm `business_id` is always returned and unique per response object. |
| autocomplete | Synthetic `ROWID` key | `AutocompleteResponse` has no natural stable identifier; synthetic key is the correct fallback. | No action required. |
| businesses | `postal_code` remains a standalone virtual despite `zip_code` in location sub-object | R-QF-008: query param name (`postal_code`) differs from response field name (`zip_code`). These are distinct fields — cannot be merged into a dual-role column. | Confirm `postal_code` is the correct query parameter name and `zip_code` is the correct response field name. |
| ai_chat | `response.text` typed `LongVarChar` | Field is named `text` and is a response content field (AI natural language response). R-TM-002 `text` → `LongVarChar`. | No action required. |

## Focused Re-checks from Generation Status

| flagged_area | final_result | notes |
| ------------ | ------------ | ----- |
| JSONRoot — all entities | PASS (structural) | Root suffixes (`/businesses`, `/reviews`, `/events`, `/categories`) are syntactically correct and match known swagger response envelope field names. Live API verification is still required before production use. |
| Pagination — businesses/matches max 10 | NEEDS_REVIEW | `#maximumPageSize: 50` exceeds the matches endpoint cap of 10. Flagged in Needs Review above. |
| Auth — BearerToken | PASS | Correctly derived from `apiKey in header` with `Authorization` header name and "Bearer YOUR_API_KEY" description. |
| service_offerings — excluded in generation | FIXED | Entity was missing. Added with `data: JSON` fallback per R-RM-013. Coverage is now 16/16. |

## Quality Gate

```yaml
quality_gate:
  gate_status: PASS
  failed_checks: 0
  max_failed_checks: 0
  mapped_endpoints: 16
  swagger_endpoints_total: 16
  endpoint_coverage_percent: 100
  min_endpoint_coverage_percent: 100
  has_coverage_warning: false
  allow_coverage_warnings: false
  decision_code: READY_FOR_LAUNCHER
```
