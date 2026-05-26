```yaml
generation_result: COMPLETE
output_file: ai-output/yelp-api/yelp-api.rest
swagger_endpoints_found: 16
rest_endpoints_modeled: 15
unmapped_or_excluded_endpoints: 1
paths_processed: 16
entities_generated: 8
entities_skipped: none
validation_status: RUN
```

Output is LLM-generated and may vary between runs. Validator pass/fail is the authoritative structural check.

## Table of Contents

- [Mandatory User Review Items](#mandatory-user-review-items)
- [Unmapped or Excluded Endpoints](#unmapped-or-excluded-endpoints)
- [Assumptions Detected](#assumptions-detected)
- [Next Steps](#next-steps)
- [Notes](#notes)
- [Generation Fingerprint](#generation-fingerprint)

---

## Mandatory User Review Items

### JSONRoot

- status: REQUIRED
- documentation: [JSONRoot](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/JSONRoot.html)
- notes:

| entity | decision | action |
| --- | --- | --- |
| businesses | Search/phone/matches paths use `/businesses` root suffix in `#path`; IDENTITY returns `BusinessDetails` at root | Verify root suffix against live API response; IDENTITY needs no root suffix |
| reviews | Path suffix `/reviews` extracts the `reviews` array from `ReviewsResponse` | Verify `/reviews` root suffix against live response |
| review_highlights | IDENTITY path returns single object at root; no root suffix needed | Verify no root suffix needed |
| events | `/v3/events` uses `/events` root suffix; featured paths return single `Event` at root | Verify `/events` suffix; confirm featured returns single object (no suffix) |
| categories | `/v3/categories` uses `/categories` root suffix; IDENTITY returns single `Category` at root | Verify suffix against live response |
| autocomplete | `AutocompleteResponse` is a single wrapper object; no root suffix | Confirm no root suffix needed; response not an array |
| ai_chat | `AIChatResponse` returned at root of POST response | Verify no root suffix needed on `#insert` path |
| business_engagement | Uses `/businesses` root suffix to extract items array | Verify suffix against live API response |

### Keys (`#key` / `#rowid`)

- status: REQUIRED
- documentation: [Determining the primary key](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/Determining-the-primary-key.html)
- notes:

| entity | decision | action |
| --- | --- | --- |
| businesses | `id` is the Yelp business ID; `VarChar(64),#key` | Confirm `id` is always present in all path variants |
| reviews | `id` is the Yelp review ID; `VarChar(64),#key` | Confirm `id` is always populated |
| review_highlights | `business_id` used as key (no `id` field); highlights child table uses synthetic `ROWID` | Confirm `business_id` uniqueness per response; synthetic ROWID for child rows |
| events | `id` is the Yelp event ID; `VarChar(64),#key` | Confirm `id` always present |
| categories | `alias` used as key (no `id` field); `VarChar(64),#key` | Confirm `alias` is unique per category |
| autocomplete | No natural key; synthetic `ROWID` on parent | Confirm synthetic key acceptable for this utility entity |
| ai_chat | `chat_id` is the conversation identifier; `VarChar(64),#key` with dual-role `#eq` | Confirm `chat_id` always returned in response |
| business_engagement | `id` is the Yelp business ID; `VarChar(64),#key` | Confirm `id` present in all engagement response items |

### Pagination

- status: REQUIRED
- documentation: [Paging](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/Paging.html)
- notes:

| entity | decision | action |
| --- | --- | --- |
| businesses | `limit`/`offset` paging; `total` from `BusinessSearchResponse`; max 50 (search) / 10 (matches) | Verify `offset`-based paging works; matches endpoint caps at 10 — may need separate entity or `#maximumPageSize` override |
| reviews | No pagination directives; API returns up to 3 reviews; `total` is plain column | Confirm API returns all reviews in one page; remove pagination directives if none needed |
| review_highlights | No pagination; IDENTITY path returns complete object | No action needed |
| events | `limit`/`offset` paging; `total` from `EventSearchResponse`; max 50 | Verify offset paging; confirm `total` element name matches live response |
| categories | No pagination; all categories returned in single response | No action needed |
| autocomplete | No pagination; single response per query | No action needed |
| ai_chat | No pagination; single response per POST | No action needed |
| business_engagement | No pagination; all requested businesses returned in single response | No action needed |

### Auth (`#options.authenticationMethod`)

- status: REQUIRED
- documentation: [Authentication](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/Authentication.html)
- notes:

| entity | decision | action |
| --- | --- | --- |
| all | `BearerToken` derived from apiKey-in-header with `Authorization` name and "Bearer YOUR_API_KEY" description | Confirm `BearerToken` works end-to-end; update `#options.authenticationmethod` if a different method is needed |

### Hostname (`#hostname`)

- status: REQUIRED
- documentation: [Server Name (alias)](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/ServerName.html)
- notes:

| entity | decision | action |
| --- | --- | --- |
| all | `https://api.yelp.com` derived from `schemes: [https]` + `host: api.yelp.com` | Confirm hostname is correct for your environment; no substitution variables needed for this API |

For manual modifications that may be required based on your API, see `.github/agents/docs/manual-rest-adjustments.md`.

---

## Unmapped or Excluded Endpoints

| Endpoint | Method | Status | Reason | Action |
| --- | --- | --- | --- | --- |
| /v3/businesses/{business_id_or_alias}/service_offerings | GET | EXCLUDED_REVIEW | Response schema is `type: object` with no defined properties — cannot form a stable relational model | Add a dedicated entity if service offering fields become stable; verify against live API response |

---

## Assumptions Detected

```yaml
assumptions:
  authentication: >
    apiKey in header named "Authorization" with description "Bearer YOUR_API_KEY" normalized to
    authenticationmethod: BearerToken. If the API uses a raw API key header (not Bearer format),
    change to HTTPHeader and set the header name accordingly.
  pagination: >
    businesses and events use offset-based paging (limit/offset). The businesses/matches endpoint
    caps at 10 results — its #maximumPageSize may need to be overridden to 10. reviews returns
    up to 3 items with no paging. All other entities return complete results in a single call.
  jsonroot: >
    Search path suffixes (/businesses, /reviews, /events, /categories) derived from known
    response wrapper field names in the swagger definitions. IDENTITY and single-object paths
    have no suffix. Verify all root suffixes against live API responses before production use.
  hostname: >
    https://api.yelp.com derived from swagger schemes[0]=https and host=api.yelp.com. No
    environment-specific substitution variables are generated — update hostname directly
    if a staging or sandbox endpoint is needed.
```

---

## Next Steps

1. Review and edit `ai-output/yelp-api/yelp-api.rest` based on the Mandatory User Review Items above.
2. Pay particular attention to:
   - Pagination for `businesses` (matches endpoint caps at 10, not 50)
   - Root suffix correctness for all entities — verify against live API responses
   - `ai_chat` entity: POST-only path using `#insert`; confirm this pattern works in your ARC version
3. Run the ARCGenAI-StaticValidator with the generated `.rest` file and source OpenAPI file.
4. If issues are found, update the `.rest` file and re-run validation.
5. If temporary assembly files remain in `ai-output/yelp-api/entities/`, delete `*.entity.tmp` and `_header.assembly.tmp` (safe manual cleanup).

---

## Notes

- `generation_result` meanings:
  - `COMPLETE`: all planned entities generated and assembled
  - `PARTIAL`: one or more entities skipped; output assembled with available entities
  - `FAILED`: assembly or generation failed before usable output
- `validation_status` is `NOT RUN` when generated initially.
- After ARCGenAI-StaticValidator runs, `validation_status` should be updated to `RUN`.

---

## Generation Fingerprint

| Field | Value |
| --- | --- |
| Swagger source | yelp-api.yaml |
| Generation timestamp | 2026-05-26 |
| Generator agent version | ARCGenAI-Generator 1.0 |
| Total swagger endpoints | 16 |
| Total endpoints modeled | 15 |
| Total entities generated | 8 |
| Excluded endpoints | 1 (service_offerings — undefined schema) |
