# Entity Plan: yelp-api

## Global Configuration
- swagger_file: d:\github_clones\autorest_recipes\arc-genai-agents\input\swagger\yelp-api.yaml
- output_file: d:\github_clones\autorest_recipes\arc-genai-agents\ai-output\yelp-api\yelp-api.rest
- hostname: https://api.yelp.com
- base_path: /
- api_version: v3 (derived from /v3/ path prefix on all versioned endpoints)
- auth_type: BearerToken
- auth_param_name: Authorization
- description: The Yelp Fusion API lets you connect with local businesses using Yelp's extensive data. Includes AI-powered conversational search, business search, reviews, events, categories, and more. Requires a Yelp API key passed as a Bearer token.
- definitions_section: lines 1432–1903
- error_message_path: {/description} — flat Error schema — description field is at root level (Error: { code: string, description: string })

## Entity Order
1. businesses
2. reviews
3. review_highlights
4. events
5. categories
6. autocomplete
7. ai_chat
8. business_engagement

## Shared Schemas
- Location: owned by businesses, referenced by events
- Coordinates: owned by businesses, referenced by events
- Category: owned by businesses, referenced by categories (IDENTITY response), autocomplete (inline sub-object)
- BusinessSummary: owned by businesses, referenced by ai_chat (AIChatResponse.entities[].businesses[])

---

## Entity: businesses
- order: 1 of 8
- paths:
  - IDENTITY GET /v3/businesses/{business_id_or_alias}
    swagger lines: 546–613
    path_params: [business_id_or_alias]
    response_ref: #/definitions/BusinessDetails (defs line 1501)
    optional_params: [locale, device_platform]
  - GET /v3/businesses/search
    swagger lines: 142–329
    path_params: []
    response_ref: #/definitions/BusinessSearchResponse (defs line 1847)
    response_root_element: businesses
    required_param_variants:
      - [location]
      - [latitude, longitude]
    optional_params: [term, radius, categories, locale, price, open_now, open_at, attributes, sort_by, device_platform, reservation_date, reservation_time, reservation_covers, matches_party_size_param, job_alias, limit, offset]
  - GET /v3/businesses/search/phone
    swagger lines: 330–389
    path_params: []
    response_ref: #/definitions/BusinessSearchResponse (defs line 1847)
    response_root_element: businesses
    required_param_variants:
      - [phone]
    optional_params: [locale]
  - GET /v3/businesses/matches
    swagger lines: 390–545
    path_params: []
    response_ref: inline { businesses: [BusinessSummary], total: integer } (defs line 1442 for item type)
    response_root_element: businesses
    required_param_variants:
      - [name, address1, city, state, country]
    optional_params: [address2, address3, postal_code, latitude, longitude, phone, yelp_business_id, limit, match_threshold]
  - GET /v3/transactions/{transaction_type}/search
    swagger lines: 614–710
    path_params: [transaction_type]
    response_ref: #/definitions/BusinessSearchResponse (defs line 1847)
    response_root_element: businesses
    required_param_variants:
      - [location]
      - [latitude, longitude]
    optional_params: [term, categories, price]
- write_paths: none
- schemas:
  - BusinessDetails: defs lines 1501–1538, owned: true
  - BusinessSummary: defs lines 1442–1500, owned: true
  - BusinessSearchResponse: defs lines 1847–1864, owned: true
  - Hours: defs lines 1539–1551, owned: true
  - OpenPeriod: defs lines 1552–1568, owned: true
  - SpecialHours: defs lines 1569–1586, owned: true
  - Location: defs lines 1587–1618, owned: true
  - Coordinates: defs lines 1619–1629, owned: true
  - Category: defs lines 1630–1654, owned: true
- http_codes: [200, 400, 401, 403, 404, 413, 429, 500, 503]

---

## Entity: reviews
- order: 2 of 8
- paths:
  - IDENTITY GET /v3/businesses/{business_id_or_alias}/reviews
    swagger lines: 908–976
    path_params: [business_id_or_alias]
    response_ref: #/definitions/ReviewsResponse (defs line 1865)
    response_root_element: reviews
    optional_params: [locale, sort_by]
- write_paths: none
- schemas:
  - ReviewsResponse: defs lines 1865–1881, owned: true
  - Review: defs lines 1655–1676, owned: true
  - User: defs lines 1677–1691, owned: true
- http_codes: [200, 400, 401, 403, 404, 413, 429, 500, 503]

---

## Entity: review_highlights
- order: 3 of 8
- paths:
  - IDENTITY GET /v3/businesses/{business_id_or_alias}/review_highlights
    swagger lines: 830–907
    path_params: [business_id_or_alias]
    response_ref: inline ReviewHighlightsResponse (no $ref; title: ReviewHighlightsResponse; properties: business_id, highlights[])
    response_root_element: (none — response IS the single object)
    optional_params: [locale]
- write_paths: none
- schemas:
  - ReviewHighlightsResponse: inline schema at swagger lines 856–905 (no definition entry)
- http_codes: [200, 400, 401, 403, 404, 429, 500, 503]

---

## Entity: events
- order: 4 of 8
- paths:
  - IDENTITY GET /v3/events/{event_id}
    swagger lines: 1187–1243
    path_params: [event_id]
    response_ref: #/definitions/Event (defs line 1692)
    optional_params: [locale]
  - GET /v3/events
    swagger lines: 977–1113
    path_params: []
    response_ref: #/definitions/EventSearchResponse (defs line 1882)
    response_root_element: events
    required_param_variants: [] (all params optional)
    optional_params: [locale, offset, limit, sort_by, sort_on, start_date, end_date, categories, is_free, excluded_events, location, latitude, longitude, radius]
  - GET /v3/events/featured
    swagger lines: 1114–1186
    path_params: []
    response_ref: #/definitions/Event (defs line 1692)
    response_root_element: (none — returns single Event object)
    required_param_variants:
      - [location]
      - [latitude, longitude]
    optional_params: [locale]
- write_paths: none
- schemas:
  - Event: defs lines 1692–1759, owned: true
  - EventSearchResponse: defs lines 1882–1893, owned: true
  - Location: defs lines 1587–1618, shared: true, owned_by: businesses
  - Coordinates: defs lines 1619–1629, shared: true, owned_by: businesses
- http_codes: [200, 400, 401, 403, 404, 413, 429, 500, 503]

---

## Entity: categories
- order: 5 of 8
- paths:
  - IDENTITY GET /v3/categories/{alias}
    swagger lines: 1299–1357
    path_params: [alias]
    response_ref: #/definitions/Category (defs line 1630)
    optional_params: [locale]
  - GET /v3/categories
    swagger lines: 1244–1298
    path_params: []
    response_ref: #/definitions/CategoriesResponse (defs line 1894)
    response_root_element: categories
    optional_params: [locale]
- write_paths: none
- schemas:
  - CategoriesResponse: defs lines 1894–1903, owned: true
  - Category: defs lines 1630–1654, shared: true, owned_by: businesses
- http_codes: [200, 400, 401, 403, 404, 413, 429, 500, 503]

---

## Entity: autocomplete
- order: 6 of 8
- paths:
  - GET /v3/autocomplete
    swagger lines: 1358–1431
    path_params: []
    response_ref: #/definitions/AutocompleteResponse (defs line 1760)
    required_param_variants:
      - [text]
    optional_params: [locale, latitude, longitude]
- write_paths: none
- schemas:
  - AutocompleteResponse: defs lines 1760–1796, owned: true
- http_codes: [200, 400, 401, 403, 404, 413, 429, 500, 503]

---

## Entity: ai_chat
- order: 7 of 8
- paths:
  - POST /ai/chat/v2
    swagger lines: 43–141
    path_params: []
    request_body: inline (required: query; optional: chat_id, user_context, request_context)
    response_ref: #/definitions/AIChatResponse (defs line 1797)
    note: POST-only endpoint; no GET path; chat_id in response is the stable conversation identifier
- write_paths:
  - #insert: POST /ai/chat/v2
    swagger lines: 43–141
    path_params: []
- schemas:
  - AIChatResponse: defs lines 1797–1846, owned: true
  - BusinessSummary: defs lines 1442–1500, shared: true, owned_by: businesses
- http_codes: [200, 400, 401, 403, 404, 413, 429, 500, 503]

---

## Entity: business_engagement
- order: 8 of 8
- paths:
  - GET /v3/businesses/engagement
    swagger lines: 711–776
    path_params: []
    response_ref: inline { businesses: [{ id: string, engagement: object (additionalProperties: true) }] }
    response_root_element: businesses
    required_param_variants:
      - [business_ids]
- write_paths: none
- schemas:
  - BusinessEngagementResponse: inline schema at swagger lines 718–766 (no definition entry)
- http_codes: [200, 400, 401, 403, 404, 429, 500, 503]
