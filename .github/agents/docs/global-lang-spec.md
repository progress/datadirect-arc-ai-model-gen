---
title: "Global REST Language Specification"
---

# Global REST Language Specification

> **For generators:** Do NOT load this file in full. Use the **Section Navigator** below to find the exact section covering your question, then read only that section.

## Section Navigator — Load only what you need

| Question / Situation                                            | Read |
| --------------------------------------------------------------- | ---- |
| Auth config, hostname, `#options`                               | §2   |
| Which paths merge into one entity vs. split                     | §3   |
| HTTP method mapping, write ops (#insert/#update/#delete)        | §4   |
| Path params, query params, `#virtual`, `#mandatory`, `#default` | §5   |
| JSON response structure, root extraction, nested objects        | §6   |
| `$ref`, `allOf`, `oneOf`, `anyOf`, component reuse              | §7   |
| SQL type for any field (string/int/bool/date/epoch)             | §8   |
| `#key`, `#notnull`, `#readonly`, `#eq`, field flag rules        | §9   |
| Pagination (offset / cursor / token / total-rows)               | §10  |
| Filter operators (`#eq`, `#in`, `#ge`, `#le`, `#contains`)      | §11  |
| Field naming, SQL name sanitization, plural preservation        | §12  |
| Arrays, child tables, normalization threshold                   | §13  |
| Timestamp format, epoch vs ISO, precision                       | §17  |
| `#http` error/retry rules                                       | §15  |
| Default varchar size, type defaults                             | §16  |
| What can be auto-derived vs. requires manual config             | §21  |

**Notation used in rules:** MUST = required; SHOULD = recommended; MAY = optional.
`jsonName<sqlName>` = JSON and SQL names differ. `<alias>[]` = normalized child table. `IDENTITY` = single-resource endpoint prefix.

---

## 1. REST Directives Reference

A complete, deduplicated reference of all official `#` directives and special syntax used in `.rest` configuration files. For rule-level usage guidance, see the corresponding sections in this specification.

### 1.1 Top-Level Directives

| Directive      | Description                                                                                                                                                                                                                       | Example                                                                                        |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `#hostname`    | Base URL for all API endpoints (scheme + host only). Version numbers and base paths MUST go in the endpoint `#path` entries, not in `#hostname`. Omit entirely if no valid hostname is found; do **not** use `"/"` as a fallback. | `"#hostname": "https://api.example.com"`                                                       |
| `#version`     | API version identifier shown in metadata, derived from API URL version segments (not OpenAPI `info.version`).                                                                                                                     | `"#version": "v1"`                                                                             |
| `#description` | Top-level description for the generated `.rest` file as a whole. Do not use this inside individual resources.                                                                                                                     | `"#description": "REST mapping for Example API"`                                               |
| `#http`        | Array of HTTP status-code handling rules applied globally at the root object only. Each entry uses `#code`, `#action`, and optionally `#operation`, `#match`, and `#message`.                                                     | `"#http": [{"#code": 200, "#action": "OK"}, {"#code": 429, "#action": "RETRY_AFTER"}]`         |
| `#options`     | Optional global configuration object. Include it only when the source Swagger/OpenAPI document contains authentication information that must be surfaced.                                                                         | `"#options": {"authenticationmethod": {"default": "OAuth2", "choices": "OAuth2,BearerToken"}}` |
| `#components`  | Named, reusable schema fragments (field definitions, paging configs, header definitions) referenced elsewhere via `$ref`. Component fields must not include `#key`.                                                               | `"#components": {"userRef": {"id": "BigInt", "name": "VarChar(128)"}}`                         |
| `#headers`     | Static non-authentication HTTP headers sent with every request for this resource or globally, but only when explicitly indicated by the source document. Do not add `Content-Type: application/json` because it is the default.   | `"#headers": {"X-Request-Id": "auto"}`                                                         |

### 1.2 Resource Definition Directives

| Directive          | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Example                                               |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `#path`            | One or more REST endpoint patterns for a resource. Can be a string or an array of strings. Ordered most specific to least specific. Full syntax: `[METHOD] <uri-path>`. Methods: `GET` (default), `POST`, `PUT`, `PATCH`, `DELETE`, `IDENTITY`. `IDENTITY` signals a guaranteed single-object endpoint. Optional response-root suffixes exist but are hand-edited because they are not always derivable from Swagger alone. Path may be a full absolute URL when no global `#hostname` is used. | `"#path": ["IDENTITY /users/{id}", "/users"]`         |
| `#insert`          | Endpoint for create operations. Value is a string: `"[METHOD] /endpoint"`. The HTTP method prefix defaults to `POST` when omitted. Any response-root suffix is a hand edit.                                                                                                                                                                                                                                                                                                                     | `"#insert": "POST /api/v1/items"`                     |
| `#update`          | Endpoint for update operations. Value is a string: `"[METHOD] /endpoint/{id}"`. The HTTP method prefix defaults to `PUT` when omitted but can be `POST`, `PUT`, or `PATCH`.                                                                                                                                                                                                                                                                                                                     | `"#update": "PATCH /api/v1/items/{id}"`               |
| `#delete`          | Endpoint for delete operations. Value is a string: `"[METHOD] /endpoint/{id}"`. The HTTP method prefix defaults to `DELETE` when omitted but may be `POST` (for APIs that use POST for deletion).                                                                                                                                                                                                                                                                                               | `"#delete": "/api/v1/items/{id}"`                     |
| `#post`            | Object form only. Specifies fixed static key-value pairs sent as the POST body on every request.                                                                                                                                                                                                                                                                                                                                                                                                | `"#post": {"query": "all", "fields": ["id", "name"]}` |
| `#sendonlyupdated` | When `true`, only modified fields are sent in PATCH/UPDATE requests rather than the full object.                                                                                                                                                                                                                                                                                                                                                                                                | `"#sendonlyupdated": true`                            |

### 1.3 Field Constraint Directives

| Directive               | Description                                                                                      | Example                                                   |
| ----------------------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| `#type`                 | Explicit SQL data type for a field. Must be one of the supported types (see §1.10).              | `"field": {"#type": "VarChar(128)"}`                      |
| `#key`                  | Marks the field as the primary key for the resource table.                                       | `"id": "BigInt,#key"`                                     |
| `#notnull` / `#nonnull` | Marks the field as non-nullable / required. Both forms are supported for backward compatibility. | `"name": "VarChar(64),#notnull"`                          |
| `#mandatory`            | Marks a virtual path field as required for API requests.                                         | `"accountId": {"#mandatory": true}`                       |
| `#virtual`              | Field is used purely as an API input parameter and is not present in the response schema.        | `"accountId": {"#virtual": true, "#type": "VarChar(64)"}` |
| `#default`              | Default value applied to the parameter when none is supplied.                                    | `"part": {"#default": "snippet"}`                         |

### 1.4 HTTP Response Handling Directives

> Used inside `#http` rule objects.

| Directive               | Description                                                                                                                                                                                                         | Example                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `#code`                 | HTTP status code that triggers the rule.                                                                                                                                                                            | `{"#code": 404, "#action": "ZERO_ROWS"}`                    |
| `#action`               | Action taken when the matching `#code` is received. Values: `OK`, `ZERO_ROWS`, `FAIL`, `RETRY_AFTER`, `RETRY_FIXED`, `REAUTHENTICATE`.                                                                              | `{"#code": 200, "#action": "OK"}`                           |
| `#operation`            | Scopes a rule to a specific operation (`SELECT`, `UPDATE`, `INSERT`, `DELETE`) so different actions can be taken per method for the same status code.                                                               | `{"#code": 404, "#operation": "SELECT", "#action": "FAIL"}` |
| `#operation` (extended) | Additional `#operation` values beyond standard DML: `LOGIN`, `GOODBYE`, `HEALTH`, `NOAUTH`, `UDF`, `API`. These are used in complex multi-step or service-style REST APIs where operations go beyond standard CRUD. | `{"#code": 401, "#operation": "LOGIN", "#action": "FAIL"}`  |
| `#match`                | Regex pattern matched against the response body to conditionally apply an action.                                                                                                                                   | `{"#match": "error.*", "#action": "FAIL"}`                  |
| `#message`              | JSONPath template used to extract a human-readable error message from the response body.                                                                                                                            | `{"#message": "{/error/description}"}`                      |

### 1.5 Pagination Directives

Pagination directives are hand-authored and maintained in [manual-rest-adjustments.md](manual-rest-adjustments.md). Generated output should only emit pagination settings when Swagger explicitly provides the required information and the rules in section 10 apply.

### 1.6 Parameter & Mapping Directives

| Directive         | Description                                                                                                                                                                                                 | Example                                                                                                  |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `#queryparameter` | When `true`, the field value is sent as a URL query-string parameter on GET requests.                                                                                                                       | `"limit": {"#type": "Integer", "#queryparameter": true}`                                                 |
| `#header`         | When `true`, the field value is sent as an HTTP request header. Combine with `#virtual` for business-logic headers. Do **not** use for authentication headers; use `#options.authenticationmethod` instead. | `"Notion-Version": {"#type": "VarChar(16)", "#header": true, "#virtual": true, "#eq": "Notion-Version"}` |

### 1.7 Query Filter Operators

Most query-filter operator syntax is hand edited and maintained in [manual-rest-adjustments.md](manual-rest-adjustments.md). Exception: `#eq` remains generated for virtual path or query fields and should match the API parameter name.

### 1.8 Authentication Options

> Authentication is configured under `#options`, **not** as virtual header fields.

| Directive                       | Description                                                                                                                                                                                                                                                                                                                                          | Example                                                                                        |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `#options.authenticationmethod` | Object declaring supported authentication methods and the default. `choices` is a comma-separated list; `default` must be one of the listed values. Canonical method names: `BearerToken`, `OAuth2`, `Basic`, `HTTPHeader`, `URLParameter`, `AWS`, `Digest`, `Custom`. Omit `#options` entirely when the source Swagger defines no security schemes. | `"#options": {"authenticationmethod": {"choices": "OAuth2,BearerToken", "default": "OAuth2"}}` |

**Authentication method quick-reference:**

| Method Name    | Use Case                                                |
| -------------- | ------------------------------------------------------- |
| `BearerToken`  | Bearer token in `Authorization` header                  |
| `Basic`        | HTTP Basic authentication                               |
| `OAuth2`       | OAuth 2.0 authorization-code or client-credentials flow |
| `HTTPHeader`   | API key delivered via a custom HTTP header              |
| `URLParameter` | API key delivered as a query-string parameter           |
| `AWS`          | AWS Signature V4                                        |
| `Digest`       | HTTP Digest authentication                              |
| `Custom`       | Custom/scripted authentication scheme                   |

### 1.8.1 Simple String Form for `authenticationmethod`

When only a single authentication method is supported, `authenticationmethod` accepts a plain string value instead of a `choices`/`default` object:

```json
"#options": { "authenticationmethod": "HTTPHeader" }
```

When the source Swagger defines no security schemes, omit the `#options` block entirely — do **not** emit `authenticationmethod: "None"` or any other sentinel value.

Use the object form only when multiple methods should be offered to the user.

### 1.8.2 Custom Multi-Step Authentication (`#authentication`)

Custom multi-step authentication is hand edited and maintained in [manual-rest-adjustments.md](manual-rest-adjustments.md).

### 1.8.3 OAuth2 File-Level Parameters

The following OAuth2 configuration keys may appear in `#options` alongside `authenticationmethod`:

| Key           | Description                                      |
| ------------- | ------------------------------------------------ |
| `tokenURI`    | URI to obtain the access token                   |
| `authURI`     | URI for the authorization code flow redirect     |
| `redirectURI` | Redirect URI registered in the OAuth2 client app |
| `logoffURI`   | URI to end the OAuth2 session                    |
| `scope`       | OAuth2 scope string passed in token requests     |

These may be relative URIs (resolved against `#hostname`) or absolute URIs.

### 1.9 Component & Reuse Syntax

| Directive / Syntax | Description                                                                                                               | Example                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `#components`      | Top-level map of named, reusable schema fragments. Any resource or field definition can reference a component via `$ref`. | `"#components": {"address": {"street": "VarChar(128)", "city": "VarChar(64)"}}` |
| `$ref`             | JSON Reference pointer to a component defined in `#components`. Follows the syntax `#/components/<name>`.                 | `"billingAddress": {"$ref": "#/components/address"}`                            |

### 1.10 Data Type & Naming Extensions

| Syntax                   | Description                                                                                                                                                                   | Example                                              |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `"fieldName[]"`          | Declares a JSON array that is normalized into a child SQL table. Explicit alias selection is a hand edit handled in [manual-rest-adjustments.md](manual-rest-adjustments.md). | `"items[]": {"id": "BigInt,#key", "qty": "Integer"}` |
| Inline constraint syntax | Type and one or more constraints combined in a single string value, comma-separated.                                                                                          | `"email": "VarChar(128),#key,#unique"`               |

**Supported SQL Data Types:**

| Type                                      | Notes                                                                                                                                                                                                            |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Bit`                                     | Binary numeric type (0 or 1). Distinct from `Boolean`.                                                                                                                                                           |
| `BigInt`                                  | 64-bit integer                                                                                                                                                                                                   |
| `Binary(n)`                               | Fixed-length binary                                                                                                                                                                                              |
| `Boolean`                                 | True/false                                                                                                                                                                                                       |
| `Char(n)`                                 | Fixed-length string of exactly `n` characters                                                                                                                                                                    |
| `Date`                                    | Calendar date                                                                                                                                                                                                    |
| `Decimal` / `Decimal(p)` / `Decimal(p,s)` | Fixed-precision decimal. Precision and scale separated by comma: `Decimal(18,2)`. Note: some wiki docs show period separator `Decimal(18.2)` — both are accepted at runtime; use comma form in generated output. |
| `Double`                                  | 64-bit floating point                                                                                                                                                                                            |
| `Float`                                   | 32-bit floating point (synonym for single-precision)                                                                                                                                                             |
| `GUID`                                    | UUID/GUID string. Accepts both 36-character UUID form (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) and 16-byte Base64-encoded strings from the REST source. Equivalent to `VarChar(36)` in SQL metadata.             |
| `Integer`                                 | 32-bit integer                                                                                                                                                                                                   |
| `JSON`                                    | Raw JSON value — contents are stored/returned as serialized JSON string                                                                                                                                          |
| `LongNVarChar`                            | Unbounded Unicode string                                                                                                                                                                                         |
| `LongVarBinary`                           | Unbounded binary / BLOB                                                                                                                                                                                          |
| `LongVarChar`                             | Unbounded string / large text                                                                                                                                                                                    |
| `LongVarChar(n)`                          | Variable-length string with explicit max length; for strings exceeding `VarChar` bounds                                                                                                                          |
| `NVarChar(n)`                             | Unicode variable-length string                                                                                                                                                                                   |
| `SmallInt`                                | 16-bit integer                                                                                                                                                                                                   |
| `Time` / `Time(s)`                        | Time of day; scale `s` = fractional seconds digits (0–9)                                                                                                                                                         |
| `TimeWithTimeZone(s)`                     | Time with timezone offset. ⚠️ Not recommended — most JDBC clients do not implement. Prefer `Time`.                                                                                                               |
| `Timestamp(n)`                            | Date + time with fractional seconds precision `n`                                                                                                                                                                |
| `TimestampWithTimeZone(s)`                | Timestamp with timezone offset. ⚠️ Not recommended — most JDBC clients do not implement. Prefer `Timestamp`.                                                                                                     |
| `TinyInt`                                 | 8-bit integer                                                                                                                                                                                                    |
| `VarBinary(n)`                            | Variable-length binary                                                                                                                                                                                           |
| `VarBinary(max)`                          | Variable-length binary with no fixed upper bound (special `max` keyword)                                                                                                                                         |
| `VarChar(n)`                              | Variable-length string, max length `n`                                                                                                                                                                           |
| `VarCharIgnoreCase(n)`                    | Case-insensitive variant of VarChar; comparisons are case-folded                                                                                                                                                 |

### 1.10.1 Bare (Unsized) Types

All types that normally accept a size/precision parameter also accept a bare (unsized) form. The runtime applies a default:

| Bare form                | Default behavior                                                          |
| ------------------------ | ------------------------------------------------------------------------- |
| `VarChar` (no size)      | Default size applied by driver (`64` unless a more specific rule applies) |
| `Decimal` (no precision) | Default precision/scale applied by driver                                 |
| `null` (JSON null)       | Treated as `VarChar(50)`                                                  |
| `Double` (no precision)  | 64-bit floating point                                                     |
| `Char` (no size)         | Fixed-length char, driver default                                         |

### 1.10.2 Date/Time Format Specifiers in Type Strings

For `Date`, `Time`, and `Timestamp` columns, a Java `SimpleDateFormat` pattern string can be appended after the type (and any constraint flags) separated by a comma. This instructs the parser to parse the date value using the specified format instead of the default ISO formats.

```
"expiry_date": "Date,M/d/yyyy"
"event_time": "Timestamp(0),yyyy-MM-dd HH:mm:ss"
```

---

## 2. Authentication & Server Options

### Rules

**R-AU-001** (MUST): This must appear at the top of the generated .rest document

**R-AU-002** (MUST): Map `servers[0].url` to `.rest` `#hostname` as scheme + host only. Strip trailing slashes from the hostname. Do NOT include version segments or base paths in `#hostname`; place them in endpoint `#path` entries.

**R-AU-003** (MUST): Map OpenAPI `securitySchemes` to `#options.authenticationmethod` choices. Use the `.rest` source's authentication method when present (highest authority). When inferring from YAML:

| OpenAPI scheme type                             | `.rest` authenticationmethod                                   |
| ----------------------------------------------- | -------------------------------------------------------------- |
| `oauth2` (authorizationCode, clientCredentials) | `OAuth2`                                                       |
| `http` + `scheme: bearer`                       | `BearerToken`                                                  |
| `http` + `scheme: basic`                        | `Basic`                                                        |
| `apiKey` + `in: header`                         | `HTTPHeader` (with configured API-key name option)             |
| `apiKey` + `in: query`                          | `URLParameter` (with configured API-key name option)           |
| None declared                                   | Omit auth (public API — e.x. spacex, open_brewery, foursquare) |

For Swagger 2.0 specs that represent bearer auth as `apiKey` in header `Authorization` (often with docs like `Bearer <token>`), normalize to `BearerToken` instead of `HTTPHeader`.

**R-AU-004** (MUST): Do NOT emit authentication credentials, tokens, or secrets as `#virtual` header fields. All auth configuration MUST appear in `#options` as placeholders. The bearer token injection is handled at runtime.

**R-AU-004b** (MUST): Do NOT emit Swagger/OpenAPI authentication schema syntax into `.rest` output. Forbidden keys in generated `.rest` auth config include `securityDefinitions`, `securitySchemes`, `type`, `scheme`, `flows`, `authorizationUrl`, `tokenUrl`, and raw `in`/`name` scheme objects copied from YAML. Translate source auth into `.rest` keys (`authenticationmethod`, `authURI`, `tokenURI`, `apiKeyName`, `authPlacement`) instead.

**R-AU-005** (MUST): For OAuth2 flows, emit `authURI` and `tokenURI` from the YAML `authorizationUrl` and `tokenUrl` into `#options` when they are present. Leave values as empty placeholders when absent.

```
# Source: zoom
"#options": {
  "authenticationmethod": "OAuth2",
  "tokenURI": "https://zoom.us/oauth/token"
}
```

**R-AU-005b** (MUST): For APIs requiring API key authentication via header or query parameter, preserve the source authentication configuration in `#options` and keep API-key parameter names aligned to the underlying security scheme.

```
# Source: sap_fieldglass
"#options": { "authenticationmethod": "HTTPHeader", "apiKeyName": "apikey", "authPlacement": "header" }

# Source: bigcommerce
"#options": { "authenticationmethod": "HTTPHeader", "apiKeyName": "X-Auth-Token" }
```

**R-AU-006** (MUST): For sandbox/test environments, emit the sandbox hostname in `#hostname` and document it clearly. Add a note that the production hostname requires manual substitution.

```
# Source: square — #hostname: https://connect.squareupsandbox.com (sandbox)
# Source: paypal — #hostname: https://api.sandbox.paypal.com
```

**R-AU-007** (SHOULD): When an API supports multiple authentication methods, list all supported methods as choices in `authenticationmethod` with a default value.

**R-AU-008** (SHOULD): For workspace-templated server URLs (openapi-aha with `{workspace}` server variable), emit the templated hostname with the placeholder preserved and expose the template variable as a required `#options` field.

```
# Source: aha
"#hostname": "https://{workspace}.aha.io"
"#options": { "workspace": "" }
```

**R-AU-009** (MUST): Hostname extraction precedence and omission rules.

- Priority for deriving a global `#hostname` (highest → lowest):
  1. OpenAPI `servers[0].url` (extract scheme + host only; strip trailing slash; exclude version segments and base paths).
  2. Absolute URLs present in the source `.rest` `#path` or in YAML `servers[]` entries — choose the most common host across candidates when multiple absolute URLs are found.
  3. Full absolute URLs present only in `IDENTITY` `#path` entries: preserve the full URL verbatim for that `#path`, but set a global `#hostname` only if all absolute identity URLs share the same host.
  4. If none of the above yield a valid hostname, DO NOT emit `#hostname` at all. Never default `#hostname` to `/` (a single forward slash) or any placeholder string.

**R-AU-010** (MUST): Hostname normalization and formatting rules.

- Ensure the extracted hostname includes scheme (`https://` or `http://`) and a valid host. Lowercase the host portion for comparison, strip default ports `:80` and `:443`, and remove any trailing slash.
- Preserve templated server variables (e.g., `https://{workspace}.aha.io`) and expose template variables as `#options` placeholders.
- When multiple candidate hostnames exist and no clear priority winner is identified (e.g., `servers[0]` absent and absolute URLs vary), emit no default global `#hostname` and document the ambiguity for manual review in the {fileName}-generation-status.md.

**R-AU-011** (SHOULD): Emit `#version` at the top level when the API has a well-defined API version identifier in URL structure. Derive from a version segment in API-facing path/base path (preferred), or from the version path segment in `servers[0].url` when needed. Do NOT derive `#version` from `info.version` (Swagger/OpenAPI document version). Omit when no API version identifier is derivable.

```json
"#version": "v2"
```

**R-AU-012** (MUST): Set `#description` verbatim from the `info.description` field in the OpenAPI document. Do NOT paraphrase, abbreviate, or summarize — copy the description exactly as written, including punctuation and formatting characters (em dash, en dash, etc.). If `info.description` is absent, omit `#description` entirely.

---

## 3. Endpoint Matching & Path Mapping

### Rules

**R-EP-001** (MUST): Build a path index from all YAML paths (path string + method) and all `.rest` `#path` entries. Matching MUST be performed on normalized path strings: strip the server base URL or host only when a global hostname is known, normalize leading slash, ignore trailing whitespace, and ignore the ` /alias` suffix appended after a space in `.rest` `#path` tokens.

```
# Example normalization
YAML: /api/v2/tickets
.rest #path: "/api/v2/tickets"
```

**R-EP-002** (MUST): For each YAML path+method, find the `.rest` resource whose `#path` array contains the best match. Priority order (highest first):

1. Exact path template match (same path segments and placeholder positions).
2. Path with different placeholder *name* but same position (e.g., `{id}` vs `{buildId}`).
3. Query-variant path (`.rest` `#path` includes `?param={value}` suffix).

When path variants differ only by final segment or query variants (for example `my/path`, `my/path/{id}`, or `my/path?{queryParameter}`), represent them as a single endpoint resource using a `#path` array rather than separate single-path resources. This includes sub-path extensions: `/businesses/search` and `/businesses/search/phone` all share the `/businesses` base and MUST be in the same entity.

**Do NOT split IDENTITY paths or sub-path endpoints into separate entities** unless the endpoint's top-level response schema is fundamentally and irreconcilably different from the collection schema (e.g., it returns an entirely different object type). A richer schema (more fields, added sub-objects) is NOT grounds for splitting — add all fields to the shared entity.

Within a `#path` array: `IDENTITY` paths MUST always appear first. All remaining paths MUST be ordered from most specific (most path segments or most query-parameter constraints) to least specific.

**R-EP-003** (MUST): Emit `IDENTITY` in generated `.rest` when the `.rest` source file contains `IDENTITY <path>` in its `#path` array for a given resource, OR when a GET operation exists on `/resource/{id}` and that id maps to the resource primary key. Do NOT invent `IDENTITY` for collection endpoints.

```
"BuildDetails": {
  "#path": ["IDENTITY app/rest/builds/id:{buildId}"]
}
```

```
# YAML: {id}  →  .rest: {buildId}
# Action: produce virtual field "buildId"
```

**R-EP-005** (MUST): For query-variant paths (APIs that encode filters inside URLs), preserve the query template form in `#path` entries exactly as the `.rest` shows. Do not attempt to decompose these into structured parameters.

```
"#path": ["app/rest/builds?locator=status:{status}"]
```

**R-EP-008** (MUST): When a `.rest` `#path` uses URL percent-encoding (e.g., `%20`), decode the path before matching against YAML paths which use readable placeholder forms.

```
# Source: sap_fieldglass
# YAML path uses: Active%20Worker%20Download
# Normalize by URL-decoding before canonicalization
```

**R-EP-009** (MUST): When multiple `.rest` resources could match the same YAML path, select the resource whose `#path` contains the most-specific pattern. Specificity priority: `IDENTITY` with id placeholder > exact collection path > query-variant > name-similarity.

**R-EP-010** (MUST): Every entity MUST have at least one non-empty entry in its `#path` array. Even POST-only endpoints (chat, AI actions, webhooks) require a path. For POST-only entities, emit the POST endpoint as the `#path` entry (for example, `"POST /ai/chat/v2"`). An empty `"#path": []` is always invalid.

```
# WRONG — empty path array
"ai_chat": { "#path": [], "message": "LongVarChar" }

# CORRECT — POST endpoint path included
"ai_chat": { "#path": ["POST /ai/chat/v2"], "message": "LongVarChar" }
```

**R-EP-011** (MUST): When grouping sub-paths into entities, **response schema compatibility is the primary test** — a shared URL prefix alone does NOT justify combining paths. Apply these rules in order:

1. **Same/compatible response schema + related path → same entity**: Paths whose 200 response is the same YAML `$ref` definition, or whose response fields are a subset of the parent entity's schema, MUST be grouped into the same entity. This applies even when the paths have slight structural differences (list vs. IDENTITY vs. phone-search) or even a different base prefix, as long as the schema matches.

2. **Same base prefix + fundamentally different response schema → SEPARATE entity**: When a path shares a prefix with an existing entity but its 200 response uses a different `$ref` with different top-level keys, a different key structure, or a completely unrelated object type, it MUST be its own entity. The URL prefix relationship is irrelevant when schemas diverge.
   
   - **Quick test**: Look at the **root-level structure** — do the paths share the same primary key field? Do their top-level response objects describe the same concept? If the paths' top-level response structures are incompatible (different key, different entity semantics, different `$ref`), they belong in separate entities. Note: a field that is legitimately `JSON` type (per R-RM-010, e.g., free-form `additionalProperties: true`) is NOT a split signal — only an incompatible root-level structure is.
   - **Example**: `/v3/businesses/engagement` shares the `/v3/businesses/` prefix but returns `{businesses: [{id, engagement: {...metrics}}]}` — a different structure from `BusinessSearchResponse` — so it is a separate entity, NOT part of `businesses`.

3. **Different base prefix + identical response schema → CAN group**: A path whose base prefix differs from an existing entity MAY be added to that entity's `#path` array when it returns the **identical** YAML response schema.
   
   - **Example**: `/v3/transactions/{transaction_type}/search` returns `BusinessSearchResponse` (same `$ref` as `/v3/businesses/search`) → belongs in the `businesses` entity despite the different `/v3/transactions/` prefix.

4. **Conflict with existing top-level entity name → rename the virtual**: If a virtual filter parameter name would conflict with a top-level entity name (e.g., a `categories` virtual inside the `events` entity where `categories` is also a top-level entity), append `_filter` to the virtual field name to disambiguate (e.g., `categories_filter`).

**R-EP-012** (MUST): Never hardcode a specific value for a path segment that is a variable parameter in the YAML. If the YAML defines a path as `/v3/transactions/{transaction_type}/search`, generated endpoint directives MUST preserve `{transaction_type}` as a placeholder — never substitute a hardcoded example value such as `delivery`. This applies to `#path`, `#insert`, `#update`, and `#delete`.

```json
// WRONG — hardcoding the transaction_type parameter value
"#path": ["/v3/transactions/delivery/search /businesses"]

// CORRECT — preserving the path parameter as a placeholder
"#path": ["/v3/transactions/{transaction_type}/search /businesses"]
```

---

## 4. HTTP Methods & CRUD Mapping

### Rules

**R-CM-001** (MUST): Map HTTP verbs to `.rest` CRUD directives using this priority order:

1. If `.rest` explicitly defines `#insert`, `#update`, or `#delete`, use those exact values verbatim.
2. Default mapping: `POST` → `#insert`, `PUT/PATCH` → `#update`, `DELETE` → `#delete`, `GET` → read (no write directive).

**R-CM-001b** (MUST): When an entity contains unambiguous resource CRUD write operations in YAML, emit all corresponding directives (`#insert`, `#update`, `#delete`) for that entity. Do not drop valid write directives because the entity also includes multiple read paths.

**R-CM-002** (MUST): Success-code preference when YAML defines multiple 2xx responses: prefer `201` for creates (`#insert`), `200` for reads and updates, `204` for deletes (no body). If only `200` exists for a POST, use `#insert` with `200` semantics.

**R-CM-003** (SHOULD): When an API uses POST for non-create actions (e.g., delete via `POST /resource/{id}/delete`), preserve the `.rest` `#delete` value exactly, even if the HTTP method is POST.

```
# Source: wordpress
"#delete": "POST /rest/v1.1/me/connected-applications/{ID}/delete"
```

**R-CM-004** (MUST): For PATCH operations that send only changed fields, emit `#sendonlyupdated: true` on the resource. Detect PATCH semantics by checking if the YAML operation uses `application/merge-patch+json` content type, or if the `.rest` source already includes `#sendonlyupdated`.

```
# Source: mailchimp, sage_cloud_accounting, salesforce_chatter, square, wordpress, zoom
```

**R-CM-005** (MUST): Do NOT invent `#insert`/`#update`/`#delete` directives for endpoints where write semantics are ambiguous (e.g., action endpoints like `/macros/{id}/apply`).

**R-CM-006** (SHOULD): When an API is read-only (no write operations in YAML), omit `#insert`/`#update`/`#delete` entirely from the resource definition. Examples: salesforce_einstein, open_brewery (public read-only resources).

**R-CM-007** (MUST): For endpoints that return `204 No Content` on success (typically DELETE or PUT), map the corresponding resource directive to have no response body mapping. Do not attempt to extract response fields for 204 responses.

---

## 5. Request Parameter Mapping

### Rules

**R-PM-001** (MUST): Map path parameters to `.rest` fields as `#virtual` with `#type` inferred from the OpenAPI schema **only when that parameter name does not already exist in the response schema**. The field name MUST match the `.rest` placeholder name (see R-EP-004 for mismatches). If the path parameter is required, additionally mark it `#mandatory`.

```
# Source: zendesk
"ticket_id": { "#type": "BigInt", "#virtual": true, "#mandatory": true, "#eq": "ticket_id" }
```

**R-PM-001b** (MUST): If a path parameter name matches a response field name anywhere in the entity schema (top-level or nested sub-field), treat them as the same field. Do NOT add a duplicate top-level `#virtual`; instead add `#eq` on the existing response field.

```json
// Path uses {address1}, {city}, {state}, {country}; response already has location.address1/city/state/country
// WRONG — duplicate top-level virtuals
"address1": { "#type": "VarChar(64)", "#virtual": true, "#eq": "address1" },
"city":     { "#type": "VarChar(64)", "#virtual": true, "#eq": "city" },
"location": {
  "address1": "VarChar(64)",
  "city": "VarChar(64)",
  "state": "VarChar(8)",
  "country": "VarChar(2)"
}

// CORRECT — reuse response fields
"location": {
  "address1": { "#type": "VarChar(64)", "#eq": "address1" },
  "city":     { "#type": "VarChar(64)", "#eq": "city" },
  "state":    { "#type": "VarChar(8)",  "#eq": "state" },
  "country":  { "#type": "VarChar(2)",  "#eq": "country" }
}
```

**R-PM-001c** (MUST): Placeholder coverage must be complete across all emitted endpoint directives. Every `{param}` appearing in `#path`, `#insert`, `#update`, or `#delete` must map to an input field source (existing response field with inline operator, or explicit virtual field when no matching response field exists). This includes paths with multiple placeholders such as `/resource/{id}/runs/{run_id}`.

**R-PM-002** (MUST): Map query parameters to `#virtual` fields with `#eq` set to the query parameter name only when the parameter is not present in the response schema. Keep the existing `#virtual` guidance for filter-only parameters. Required query parameters (OpenAPI `required: true`) MUST additionally be embedded in the `#path` URL template as query-string placeholders (e.g., `?name={name}&city={city}`) so the connector always transmits them. A `#mandatory` virtual alone is NOT sufficient — the path URL template controls what is transmitted.

```json
// WRONG — required phone param only in virtual; never reaches the request
"#path": ["/v3/businesses/search/phone /businesses"],
"phone_filter": { "#virtual": true, "#mandatory": true, "#eq": "phone" }

// CORRECT — required param in the path URL AND as a virtual/data column
"#path": ["/v3/businesses/search/phone?phone={phone} /businesses"],
"phone": { "#type": "VarChar(32)", "#mandatory": true, "#eq": "phone" }
```

When multiple required params apply to a path variant within a merged entity, list all required params in that variant's path URL:

```json
"#path": [
  "IDENTITY /v3/businesses/{id}",
  "/v3/businesses/search /businesses",
  "/v3/businesses/matches?name={name}&address1={address1}&city={city}&state={state}&country={country} /businesses"
]
```

When an endpoint has two mutually-exclusive sets of required parameters (e.g., "pass `location` OR pass `latitude`+`longitude`"), add BOTH variants as separate path URL entries:

```json
"#path": [
  "/v3/events/featured?location={location} /events",
  "/v3/events/featured?latitude={latitude}&longitude={longitude} /events"
]
```

**R-PM-003** (MUST): Do NOT emit authentication-related header parameters as `#virtual` header fields. Instead, expose them through `#options` configuration placeholders. (See §2 for auth rules.)

**R-PM-005** (MUST): For body parameters in POST/PUT/PATCH requests, map JSON object properties to `.rest` resource fields using type mapping rules (§8). When the body schema is free-form (`additionalProperties: true`) or unspecified, emit a single `data` field with `#type: JSON`.

**R-PM-008** (MUST): For date-range virtual parameters (`startDate`/`endDate`, `from`/`to`), map them to `#ge`/`#le` or `#gt`/`#lt` range operators respectively, with typed date/timestamp comparison semantics.

```
# Source: quora_ads, youtube_analytics
"req_startDate": { "#type": "Date", "#virtual": true, "#ge": "startDate" }
"req_endDate":   { "#type": "Date", "#virtual": true, "#le": "endDate" }
```

**R-PM-009** (MUST): When a YAML query parameter is marked `required: true` AND a field with the same name already exists as a sub-property of a structured response object within the same entity, add the **scalar** filter operator (`#eq`) directly to the **response sub-field** — do NOT create a separate top-level `#virtual` field for scalar filters. The sub-field serves as the dual-role data column and required filter. Only add `#mandatory` if the field is required across **all** paths in the entity's `#path` array (see R-FC-002).

> **`#in` exception:** If the YAML parameter is an array-style filter (`type: array` or comma-separated values), do NOT put `#in` on the response sub-field. Instead, create a separate top-level `#virtual` field with `#in` pointing at the original parameter name (e.g., `price_filter: { "#virtual": true, "#in": "price" }`). The `#in` operator must always live on a virtual field — never on a response data column (see R-QF-009).

```json
// Scenario: entity has paths [IDENTITY, /search, /matches, /search/phone]
// /businesses/matches requires address1, city, state, country — but the other paths do not.
// These also appear as sub-fields of the response 'location' object.

// WRONG — separate top-level virtuals when the fields already exist in a response sub-object
"address1": { "#virtual": true, "#mandatory": true, "#eq": "address1" },
"city":      { "#virtual": true, "#mandatory": true, "#eq": "city" },
"location": { "address1": "VarChar(64)", "city": "VarChar(64)" }

// ALSO WRONG — #mandatory on sub-fields when the entity has other paths that do NOT require these fields
"location": {
  "address1": { "#type": "VarChar(64)", "#mandatory": true, "#eq": "address1" },
  "city":     { "#type": "VarChar(64)", "#mandatory": true, "#eq": "city" }
}

// CORRECT — #eq on the sub-fields (not #mandatory); path URL template handles required transmission
// The ?address1={address1}&city=... in the matches path URL ensures they are sent for that variant.
"location": {
  "address1": { "#type": "VarChar(64)", "#eq": "address1" },
  "city":     { "#type": "VarChar(64)", "#eq": "city" },
  "state":    { "#type": "VarChar(8)",  "#eq": "state" },
  "country":  { "#type": "VarChar(2)",  "#eq": "country" }
}

// #mandatory IS correct when the entity has ONLY the matches path (field required on every path):
// "#path": ["/v3/businesses/matches?name={name}&address1={address1}&city={city}&state={state}&country={country} /businesses"]
// "location": {
//   "address1": { "#type": "VarChar(64)", "#mandatory": true, "#eq": "address1" },
//   "city":     { "#type": "VarChar(64)", "#mandatory": true, "#eq": "city" }
// }
```

**R-PM-010** (MUST): When a YAML query parameter declares a `default` value, emit `#default` on the corresponding virtual or data field with that value. Do not omit defaults present in the OpenAPI spec.

```json
// YAML: sort_by: description: "Sort order", default: "best_match"
// CORRECT
"sort_by": { "#type": "VarChar(32)", "#virtual": true, "#eq": "sort_by", "#default": "best_match" }
```

---

## 6. Response Mapping: Root, Objects, Arrays

### Rules

**R-RM-001** (MUST): Identify the response root by examining the YAML response schema property name that contains the array of items. Map that array property as the resource row source. If `.rest` appends a `/alias` token to `#path` (e.g., `/tickets /tickets`), the alias token is the response root extraction key.

```
# Source: zendesk
"#path": ["/api/v2/tickets /tickets"]  →  response root key: "tickets"
```

**R-RM-002** (MUST): For paged list responses that wrap items under a standard envelope (e.g., `{ data: [...] }`, `{ result: { elements: [...] } }`, `{ items: [...] }`), configure the appropriate root extraction path using the ` /envelope_key` suffix in `#path` or `#nextPageElement` pointing to the envelope field.

| Pattern                        | Source                                 | Root extraction            |
| ------------------------------ | -------------------------------------- | -------------------------- |
| `{ data: [] }`                 | quora_ads, stripe, salesforce_einstein | ` /data` suffix in `#path` |
| `{ result: { elements: [] } }` | sap_qualtrics                          | `/result/elements`         |
| `{ result: {} }`               | marketo, sap_qualtrics singletons      | `/result`                  |
| `{ items: [] }`                | zoho_crm, youtube_analytics            | ` /items` or `/$items`     |
| `{ businesses: [] }`           | yelp                                   | ` /businesses`             |

**R-RM-003** (MUST): Map nested object arrays inside response items to child tables using `<alias>[]` notation. The alias MUST match the `.rest` source's aliasing pattern when one exists. When no alias exists in `.rest`, generate the alias as `<ParentResourceName>_<fieldName>`.

```
# Source: yelp
"categories<businesses_categories>[]": { "alias": "VarChar(64)", "title": "VarChar(64)" }

# Source: stripe
"refunds<Charges_refunds>[]": { "id": "VarChar(64),#key", "amount": "Integer" }
```

**R-RM-004** (MUST): When a child array item does not carry the parent's identity field, add a `#virtual` parent FK field to the child resource. The FK field name should match the parent's `#key` field name.

**R-RM-005** (MUST): For single-resource identity endpoints (paths with `{id}` or equivalent), map the response directly to the resource fields. Do not create a wrapper array — the response represents a single row.

**R-RM-006** (MUST): For response schemas that use `allOf`, `oneOf`, or `anyOf`, apply the following resolution strategy:

- `allOf`: flatten all merged properties into the resource.
- `oneOf`/`anyOf`: emit `#type: JSON` for the field.

**R-RM-010** (MUST): When an OpenAPI schema defines a named object with explicit sub-properties (via inline `properties` or `$ref`), map those sub-properties as structured nested fields rather than collapsing the object to `JSON`. Only collapse to `#type: JSON` when the object schema is free-form (`additionalProperties: true`), uses `oneOf`/`anyOf`, or contains no named properties. Collapsing structured objects to JSON loses queryability and is not acceptable when the schema is defined.

**R-RM-011** (MUST): When a `#path` entry uses root extraction (the `/alias` suffix), each element of that array IS the entity row. Do NOT additionally wrap those fields in a child table of the same name. The root extraction already makes the array elements the rows; creating a child table with the same alias double-nests the data.

```
# WRONG — path uses /reviews root extraction; creating reviews_list[] child table double-nests
"#path": ["/v3/businesses/{id}/reviews /reviews"],
"reviews<reviews_list>[]": {
  "id": "VarChar(64),#key",
  "text": "LongVarChar",
  "user": { "id": "VarChar(64)" }
}

# CORRECT — review fields are top-level; root extraction makes each array element a row
"#path": ["/v3/businesses/{id}/reviews /reviews"],
"id": "VarChar(64),#key",
"text": "LongVarChar",
"user": { "id": "VarChar(64)" },
"total": "Integer",
"possible_languages[]": "VarChar(16)"
```

**R-RM-007** (SHOULD): For APIs that embed response data inside a resource-name key (e.g., `{ "payment": {...} }` in square, `{ "charge": {...} }` in stripe), configure inner object extraction using the appropriate suffix hinting in `#path` (e.g., `IDENTITY /v2/payments/{id} /payment`).

```
# Source: square
"#path": ["IDENTITY /v2/payments/{id} /payment"]
```

**R-RM-008** (MUST): For array-of-arrays response structures (e.g., `ReportResultTable.rows` in youtube_analytics where `rows` is `array of array`), emit the field as `#type: JSON` since column structure cannot be statically determined.

**R-RM-009** (SHOULD): For deeply nested child arrays (e.g., `sections[]` containing `fields[]`), create multi-level child table aliases following the pattern `<ParentAlias>_<ChildAlias>[]`.

```
# Source: zoho_crm
"sections<LayoutsMetadata_sections>[]": {
  "fields<LayoutsMetadata_sections_fields>[]": { ... }
}
```

**R-RM-012** (MUST): All properties present in the YAML response schema MUST be emitted in the generated `.rest`. Do NOT silently drop fields because they seem redundant, overlap with another field, or seem unimportant. Every response field that the API can return has value for users querying the data.

```json
// WRONG — dropping 'region' and 'possible_languages' because they seem optional
"businesses": { "id": "VarChar(64),#key", "name": "VarChar(128)" }

// CORRECT — all response schema fields present
"businesses": {
  "id": "VarChar(64),#key",
  "name": "VarChar(128)",
  "region": { "center": { "latitude": "Double", "longitude": "Double" } }
}
```

**R-RM-013** (MUST): Every entity must include at least one non-virtual response field. An entity containing only directives and `#virtual` fields is invalid. If the endpoint response schema is opaque, empty, or cannot be reliably expanded into concrete columns, emit a generic fallback response field such as `"data": "JSON"` to preserve a valid queryable row shape.

```json
// WRONG — virtual-only entity
"service_offerings": {
  "#path": ["IDENTITY /v3/businesses/{business_id_or_alias}/service_offerings"],
  "business_id_or_alias": { "#type": "VarChar(64)", "#virtual": true, "#mandatory": true, "#eq": "business_id_or_alias" }
}

// CORRECT — includes a non-virtual response field fallback
"service_offerings": {
  "#path": ["IDENTITY /v3/businesses/{business_id_or_alias}/service_offerings"],
  "data": "JSON",
  "business_id_or_alias": { "#type": "VarChar(64)", "#virtual": true, "#mandatory": true, "#eq": "business_id_or_alias" }
}
```

---

## 7. Schemas, Components & `$ref` Usage

### Rules

**R-SC-001** (MUST): For OpenAPI component schemas referenced by 3 or more endpoints, create a shared component mapping in the `.rest` generator state. Do not inline the same schema structure into multiple resources.

**R-SC-002** (SHOULD): Map a schema as a reusable component only when it is reused by at least 3 endpoints. Otherwise inline the properties directly into the resource.

**R-SC-004** (MUST): Preserve `$ref` component names as the basis for child resource names and nested object type references in the generated `.rest`. Do not rename components during code generation.

**R-SC-005** (MUST): For `additionalProperties: true` schemas (common in teamcity, spacex, quickbooks_online), treat all fields as dynamic unless the `.rest` source provides concrete field definitions. In that case, the `.rest` field definitions take precedence.

**R-SC-006** (MUST): For polymorphic schemas using `oneOf`/`anyOf`, always collapse to `#type: JSON`.

**R-SC-007** (MUST): For `allOf` composition, merge all referenced schema properties into a single flat mapping. If the merged result contains duplicate field names from different schemas, prefer the more specific (non-generic) schema's type definition.

---

## 8. Type Mapping & Type Inference

### Rules

**R-TM-001** (MUST): Apply the following base type mapping table (in priority order). When both YAML type and `.rest` type are present, the `.rest` type wins.

| OpenAPI type + format                                                                                                      | `.rest` SQL type                                                                                                        |
| -------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `string` + `format: date-time`                                                                                             | `Timestamp(0)` (default; see C-002 for precision variants)                                                              |
| `string` + `format: date`                                                                                                  | `Date`                                                                                                                  |
| `string` + `format: uuid`                                                                                                  | `VarChar(36)`                                                                                                           |
| `string` with description mentioning `ISO 8601` or `RFC 3339`                                                              | `Timestamp(0)` — treat as a timestamp even though the YAML type is `string`                                             |
| `string` with description mentioning a date-only format (e.g., `YYYY-MM-DD`, `"format: date"`, `"in the form YYYY-MM-DD"`) | `Date` — treat as a date even though the YAML type is `string` with no `format`                                         |
| `string` (no format, no hints)                                                                                             | `VarChar(DefaultVarcharSize)` — see C-001                                                                               |
| `integer`                                                                                                                  | `Integer`                                                                                                               |
| `integer` (large values, `*_id` names)                                                                                     | `BigInt`                                                                                                                |
| `number` / `number + format: float\|double`                                                                                | `Double`                                                                                                                |
| `number + format: decimal` or `multipleOf`                                                                                 | `Decimal`                                                                                                               |
| `boolean`                                                                                                                  | `Boolean`                                                                                                               |
| `object` (free-form / additionalProperties)                                                                                | `JSON`                                                                                                                  |
| `array` of objects                                                                                                         | `<alias>[]` child table (see §13)                                                                                       |
| `array` of primitives (strings, integers, etc.)                                                                            | `<fieldName>[]` typed column — NEVER collapse to `JSON`; use `JSON` only when the array is heterogeneous or schema-less |

**R-TM-002** (MUST): Apply VarChar sizing heuristics when OpenAPI does not provide `maxLength`. Priority order:

1. Use the `maxLength` constraint from the OpenAPI parameter or schema definition (highest authority — always use this value when present).
2. Use `.rest` explicitly declared size.
3. Apply field-name heuristics (table below).
4. Fall back to `DefaultVarcharSize` (see C-001).

| Field name pattern                                                                        | Recommended VarChar size                                                                           |
| ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `url`, `uri`, `link`, `href`, `base_link`                                                 | VarChar(512) — some sources use 1024 or 2000                                                       |
| `email`                                                                                   | VarChar(256)                                                                                       |
| `description`, `body`, `content`, `bio`, `excerpt`, `text`, `message`, `notes`, `summary` | `LongVarChar` (these fields routinely exceed safe VarChar limits; do NOT size-cap with VarChar(N)) |
| `name`, `title`, `label`                                                                  | VarChar(128)                                                                                       |
| `id`, `hashed_id`, `resource_key`, UUID-like                                              | VarChar(64) or VarChar(36) for strict UUID                                                         |
| `summary`, `agenda`                                                                       | VarChar(1024) to VarChar(8192) depending on API — use LongVarChar if no explicit maxLength         |
| Default (all others)                                                                      | `DefaultVarcharSize` (see C-001)                                                                   |

**Hard ceiling rule**: When VarChar sizing would exceed `2056` characters for any field, use `LongVarChar` instead. Never emit `VarChar(N)` for N > 2056.

**R-TM-003** (MUST): Detect GUID/UUID fields and map to `VarChar(36)` (strict UUID) or `VarChar(64)` (general hash/key). Use `VarChar(36)` only when `format: uuid` is explicit or pattern confirms UUID regex. Use `VarChar(64)` for opaque hash strings like Wistia `hashed_id` or Vimeo `resource_key`.

**R-TM-004** (MUST): For currency amount fields (e.g., `amount` in payment APIs like stripe, quickbooks_payments, square), resolve the type as follows:

- `Integer` when the API uses minor units / cents (stripe, square).
- `Double` or `Decimal` when the API uses decimal amounts directly.

**R-TM-005** (MUST): For Unix epoch timestamp fields (integer values representing seconds since epoch, e.g., `created` in stripe), map to `BigInt` (not `Timestamp`). Document the epoch nature with a field comment. Do NOT apply `Timestamp` type to these fields.

```
# Source: stripe
"created": "BigInt"   # Unix epoch; NOT Timestamp
```

**R-TM-005b** (MUST): This rule applies equally to **query PARAMETERS** typed as `integer` in the OpenAPI schema. When a query parameter's YAML description contains any of: "Unix time", "Unix timestamp", "epoch", "seconds since 1970", "seconds since epoch", or similar wording — map that parameter to `BigInt`, NOT `Integer`. Current Unix timestamps (e.g., 1744000000) already exceed the 32-bit Integer maximum (2,147,483,647) and will overflow after the year 2038. Common examples: `open_at`, `created_since`, `available_at`, `start_time`, `end_time` when described as Unix epoch integers.

```
# Source: yelp — "open_at" parameter description: "An integer representing the Unix time..."
# WRONG: "#type": "Integer"    ← overflows after 2038
# CORRECT: "#type": "BigInt"
"open_at": { "#type": "BigInt", "#virtual": true, "#eq": "open_at" }
```

**R-TM-007** (SHOULD): For `string` fields with an explicit limited `enum` in OpenAPI, map to `VarChar(N)` where N is the maximum length of any enum value (minimum 32). Do not use an enum SQL type.

**R-TM-008** (MUST): For fields typed `number` without explicit format, default to `Double`. Use `Decimal` only when the API documentation or `.rest` source specifies decimal precision. Never promote `number` to `Integer` without explicit evidence.

**R-TM-009** (MUST): When a `string` field has no explicit `format` in the OpenAPI schema but the field description, example value, or pattern references a date-only format (e.g., "YYYY-MM-DD", "in the format YYYY-MM-DD", "format: date"), map to `Date` — not `VarChar`. Similarly, if the description references a date-plus-time pattern (e.g., "YYYY-MM-DD HH:mm:ss"), map to `Timestamp(0)` with the appropriate format string rather than `VarChar`.

```
# Field description: "Date of the reservation in format YYYY-MM-DD"
# WRONG
"reservation_date": "VarChar(16)"

# CORRECT — description signals date-only; no explicit format needed
"reservation_date": "Date"
```

**R-TM-010** (SHOULD): For `boolean` fields in OpenAPI, map to `Boolean` in `.rest`. Do not convert to Integer (0/1) unless the specific `.rest` source demonstrates Integer semantics for that field.

---

## 9. Field Constraints & Special Flags

### Rules

**R-FC-001** (MUST): Apply primary-key (`#key`) derivation in this priority order, and do not invent a key when no reliable candidate exists:

1. Explicit `,#key` annotation in the `.rest` source file.
2. `IDENTITY` path parameter that uniquely identifies the resource.
3. OpenAPI `required` property named `id`, `uri`, or `<resource>Id` with unique semantics.
4. Path parameter used in update/delete endpoints

`#key` does NOT imply `#notnull` (MUST): Do not auto-add `#notnull` to key fields. Add `#notnull` only when required by schema/constraints independent of key designation.

`#notnull` derivation is strict (MUST): Add `#notnull` only when the OpenAPI property explicitly declares `nullable: false`. Do not infer `#notnull` from `required` alone.

**`id` always beats `alias`/`slug` (MUST):** When a response schema contains both an `id` field and a human-readable identifier (e.g., `alias`, `slug`, `handle`, `key`), ALWAYS select `id` as `#key`. Human-readable identifiers are mutable — they can be renamed by the resource owner — and are not guaranteed globally unique. An `alias` that looks stable today can change tomorrow, breaking cached lookups and JOIN relationships. The `id` field is immutable and purpose-built as a unique row identifier. This rule applies even when the IDENTITY path parameter is named after the alias (e.g., `{business_id_or_alias}`) — in that case, add a mandatory virtual for the path placeholder and keep `id` as `#key`.

**R-FC-002** (MUST): Mark fields `#mandatory` when:

- The OpenAPI parameter or property has `required: true` on **every** path in the entity's `#path` array, OR
- The `.rest` source explicitly marks the field `#mandatory`.

When a field is required for only a *subset* of the paths in a merged entity, do NOT add `#mandatory`. The path URL template embedding (e.g., `?name={name}`) already controls transmission for that specific path variant. `#mandatory` at the entity level signals that the field is required regardless of which path is active — applying it to a path-specific required field forces it to be mandatory for all paths, which is incorrect.

**Exception — `#virtual` path parameters (R-PM-001, R-FC-010b):** A `#virtual` field that maps a path placeholder (e.g., `{business_id_or_alias}` for an IDENTITY path) always takes `#mandatory`, even when that path is only one of several in `#path`. These virtuals exist solely to drive that specific path and would be meaningless without a value.

**R-FC-003** (MUST): Mark fields `#readonly` for server-generated timestamps (`created_at`, `updated_at`, `published_at`).

**R-FC-005** (MUST): Mark fields `#virtual` when:

- The field is a path parameter that does NOT already exist in the response schema, OR
- The field is a query parameter used **only** for filtering/selection and does NOT appear in the response schema, OR
- The field is a constant or computed value injected into requests (e.g., `moduleApiName=Leads`).

When a field appears in **both** the response schema and as a query/path filter parameter, treat it as a **data column** (not `#virtual`). Add the `#eq`/`#ge`/`#le` operator mapping inline on the data field definition. Do NOT rename the field or create a second virtual field with a different name. Do NOT add a disambiguating prefix (e.g., `match_name` for the `name` parameter) — always use the original parameter name.

**Exception — `#in` filters:** The dual-role pattern above ONLY applies to scalar operators (`#eq`, `#ge`, `#le`). When the query parameter accepts comma-separated or array values (requiring `#in`), you CANNOT use the dual-role approach even if the response has a same-named field — because `#in` is only valid on `#virtual: true` fields (R-QF-002, R-QF-009). In this case, keep the response field as a plain data column and create a separate `*_filter` virtual with `#in`.

```
# Correct: latitude appears in response AND as a filter parameter
"latitude": { "#type": "Double", "#eq": "latitude" }

# Incorrect: renaming to latitude_filter and making it virtual
"latitude_filter": { "#virtual": true, "#type": "Double", "#eq": "latitude" }

# Also incorrect: duplicate — data column exists AND a separate virtual is added
"latitude": "Double",
"latitude_filter": { "#virtual": true, "#type": "Double", "#eq": "latitude" }

# Also incorrect: prefixing to avoid a perceived conflict
"match_name": { "#virtual": true, "#mandatory": true, "#type": "VarChar(64)", "#eq": "name" }

# Correct: original name, no prefix
"name": { "#type": "VarChar(128)", "#mandatory": true, "#eq": "name" }
```

**R-FC-010** (MUST): When a `#path` entry uses a path parameter (e.g., `{alias}`) and that parameter name matches the `#key` field name exactly (i.e., the path parameter IS the resource identifier field), do NOT create a separate `#virtual` field for it. The IDENTITY path itself handles the lookup — the `#key` data column is sufficient.

```
# WRONG — alias is both the #key field and the path param; no virtual needed
"alias_filter": { "#virtual": true, "#type": "VarChar(64)", "#mandatory": true, "#eq": "alias" },
"alias": "VarChar(64),#key"

# CORRECT — the IDENTITY path /categories/{alias} uses the alias #key directly
"alias": "VarChar(64),#key"
```

**R-FC-010b** (MUST): When an IDENTITY path uses a path parameter whose name DIFFERS from the entity's `#key` field name, a `#mandatory` `#virtual` field MUST be added for that path parameter — **unless a same-named response field already exists in the entity schema**. In that case, use the existing response field with inline `#eq` and do not add a duplicate virtual. The goal is to ensure every path placeholder has an input source without creating duplicate fields.

```json
// IDENTITY path: IDENTITY /v3/businesses/{business_id_or_alias}
// #key is "id" — the immutable Yelp business ID (R-FC-001: id always beats alias)
// The path param "business_id_or_alias" differs from the #key "id" → virtual required.

// WRONG — no virtual for {business_id_or_alias}; path param is an orphan
"id": "VarChar(64),#key"

// ALSO WRONG — alias chosen as #key; id is the correct stable identifier
"alias": "VarChar(256),#key"

// CORRECT — id is #key; virtual added matching the path placeholder name
"business_id_or_alias": {
  "#type": "VarChar(64)",
  "#virtual": true,
  "#mandatory": true,
  "#eq": "business_id_or_alias"
},
"id": "VarChar(64),#key",
"alias": "VarChar(256)"
```

Contrast with R-FC-010: when the IDENTITY path parameter name IS the same as the `#key` field name, no virtual is needed. R-FC-010b applies only when the names differ.

**R-FC-011** (MUST): Do NOT add a top-level data column for a filter parameter that is NOT in the endpoint's top-level response schema. If the same-named field exists as a sub-property of a structured response object, add the `#eq` operator to the sub-field — do NOT duplicate it as a phantom top-level column. Phantom top-level columns produce extra SQL columns that don't correspond to any response data and will always be empty.

```
# WRONG — latitude/longitude are query filter params; NOT top-level response fields;
# they duplicate coordinates.latitude / coordinates.longitude which already exist in the response
"latitude":  { "#type": "Double", "#eq": "latitude" },
"longitude": { "#type": "Double", "#eq": "longitude" },
"coordinates": { "latitude": "Double", "longitude": "Double" }

# CORRECT — #eq on the nested response fields; no phantom top-level columns
"coordinates": {
  "latitude":  { "#type": "Double", "#eq": "latitude" },
  "longitude": { "#type": "Double", "#eq": "longitude" }
}
```

**R-FC-012** (MUST): Do NOT add a top-level `#virtual` field for a field that already exists as a sub-property of a structured response object within the same entity. The sub-object field is already addressable for filtering via the structured object. Duplicate top-level virtuals are redundant.

```
# WRONG — location.address2 and location.address3 already exist in the entity;
# adding top-level virtuals for the same fields is redundant
"address2": { "#virtual": true, "#type": "VarChar(128)", "#eq": "address2" },
"address3": { "#virtual": true, "#type": "VarChar(128)", "#eq": "address3" },
"location": { "address1": "VarChar(128)", "address2": "VarChar(128)", "address3": "VarChar(128)" }

# CORRECT — sub-object fields only; no duplicate virtuals
"location": { "address1": "VarChar(128)", "address2": "VarChar(128)", "address3": "VarChar(128)" }
```

**R-FC-008** (MUST): For compound primary keys (multiple fields required for identity), mark all key fields with `#key`. Document the compound key in for human review.

**R-FC-013** (MUST): The `#key` modifier MUST always use the inline comma form inside the type string — never as a standalone `"#key": true` JSON property. This applies even when the field also uses the object form for other properties such as `#eq`, `#mandatory`, or `#insert`/`#update`. Put `#key` inside the `#type` value string using comma syntax.

```json
// WRONG — #key as a standalone object property
"chat_id": { "#type": "VarChar(64)", "#key": true, "#eq": "chat_id" }

// ALSO WRONG — #key as a standalone object property without #eq
"id": { "#type": "BigInt", "#key": true }

// CORRECT — #key always inside the #type string using comma notation
"id": "BigInt,#key"

// CORRECT — #key inline even when other properties are needed
"chat_id": { "#type": "VarChar(64),#key", "#eq": "chat_id" }
```

**R-FC-009** (SHOULD): For `etag` fields returned by APIs (Google APIs), map to `VarChar(64)` and mark `#readonly`. Do not use `etag` as a primary key unless no other identity field exists.

---

## 10. Pagination & Paging Heuristics

### Rules

These rules apply only when the Swagger or OpenAPI source explicitly provides paging parameters or paging response fields. Do not invent paging behavior when the source material does not declare it.

**R-PG-001** (MUST): When the `.rest` source defines pagination directives (`#pageSizeParameter`, `#pageNumberParameter`, `#rowOffsetParameter`, `#nextPageParameter`, `#nextPageElement`, `#hasMoreElement`, `#totalRowsElement`, `#maximumPageSize`, `#firstPageNumber`), copy those values verbatim into the generated `.rest`. Do not override `.rest` pagination settings with YAML-inferred values.

**R-PG-002** (MUST): Detect the pagination style from YAML query parameters and map to the appropriate `.rest` directive:

| YAML parameters                        | Style             | `.rest` directives                                               |
| -------------------------------------- | ----------------- | ---------------------------------------------------------------- |
| `page` + `per_page` / `pageSize`       | Page-number       | `#pageNumberParameter`, `#pageSizeParameter`, `#firstPageNumber` |
| `offset` + `count` / `limit`           | Row-offset        | `#rowOffsetParameter`, `#pageSizeParameter`                      |
| `nextPageToken` / `pageToken`          | Token             | `#nextPageElement`, `#nextPageParameter`                         |
| `cursor` / `after` / `before`          | Cursor            | `#nextPageParameter`, `#nextPageElement`                         |
| `skipToken`                            | Skip-token        | `#nextPageParameter: skipToken`, `#nextPageElement: skipToken`   |
| `startPosition` + `maxResults` (SQL)   | SQL offset        | `#rowOffsetParameter`, custom SQL query                          |
| `locator=start:{offset},count:{limit}` | Locator template  | `#RowOffsetParameter` as compound template                       |
| `has_more` response flag               | Has-more sentinel | `#hasMoreElement`                                                |
| None detected                          | No paging         | Omit all paging directives                                       |

**R-PG-003** (MUST): Set `#firstPageNumber` to `0` when the API is 0-indexed (Salesforce Chatter uses `firstPage: 0`) or `1` when 1-indexed (most APIs). Default to `1` when the API does not document its first page.

**R-PG-004** (MUST): Set `#maximumPageSize` from the OpenAPI parameter `maximum` constraint if present. If absent, use the `.rest` source value. If both are absent, apply the following defaults by pagination style:

- page/per_page APIs: 100
- offset APIs: 50
- token APIs: 100
- cursor APIs: 100

**R-PG-005** (MUST): For APIs that return `next_page`, `next_page_token`, `next_page_url`, or similar as a JSON response field, map to `#nextPageElement` using the JSON path to that field (e.g., `/meta/next_page`, `/nextPageToken`, `/cursor`).

**R-PG-006** (MUST): For APIs that return `has_more: boolean` (stripe, square, dropbox), set `#hasMoreElement` to the JSON path of that flag (e.g., `has_more`). The generator uses this flag to determine whether to fetch another page.

**R-PG-007** (MUST): For token-based pagination where the next-page token is the ID of the last record (e.g., Stripe `starting_after`), configure `#nextPageElement` to point to the ID field of the last record in the `data` array (e.g., `/data/id`).

```
# Source: stripe
"#nextPageParameter": "starting_after"
"#nextPageElement": "/data/id"
"#hasMoreElement": "has_more"
```

**R-PG-008** (SHOULD): When no pagination parameters are documented, omit all paging directives.

**R-PG-013** (MUST): When pagination directives (`#pageSizeParameter`, `#rowOffsetParameter`, `#pageNumberParameter`, `#nextPageParameter`) are declared on an entity, do NOT additionally create `#virtual` fields with `#eq` for those same parameters. Pagination directives are the canonical mechanism for exposing limit/offset/page to callers. Duplicate virtual fields for the same pagination parameters create ambiguous bindings and may cause the parameter to be sent twice or conflict with the directive-driven value.

```json
// WRONG — pagination directives declared AND redundant explicit virtuals for the same params
"#pageSizeParameter": "limit",
"#rowOffsetParameter": "offset",
"limit":  { "#type": "Integer", "#virtual": true, "#eq": "limit",  "#default": "20" },
"offset": { "#type": "Integer", "#virtual": true, "#eq": "offset" }

// CORRECT — pagination directives only; no duplicate virtuals
"#pageSizeParameter": "limit",
"#rowOffsetParameter": "offset"
```

**R-PG-009** (MUST): For SQL-style pagination exposed as explicit request parameters, map the offset parameter to `#rowOffsetParameter` and the page-size parameter to `#pageSizeParameter`.

**R-PG-010** (MUST): When the response schema explicitly exposes a total-row-count field (e.g., a `total` integer property at the response root), configure `#totalRowsElement` so the driver can report total row counts without fetching all pages. Do NOT add `#totalRowsElement` when no such field is declared in the YAML — never guess at a total count field name.

**R-PG-011** (SHOULD): For APIs that support both offset and cursor pagination simultaneously (e.g., dropbox: `cursor` for continuation, `offset` for initial page), prefer cursor pagination when it enables stateless continuation across large datasets.

**R-PG-012** (MUST): When pagination parameters are query parameters inside a compound locator string (teamcity), preserve the entire locator template in `#RowOffsetParameter` rather than splitting into individual fields.

---

## 11. Query Operators & Filtering

### Rules

**R-QF-001** (MUST): For simple scalar equality query parameters, map to `#eq` semantics. The `#eq` value must equal the OpenAPI query parameter name.

```
"status": { "#virtual": true, "#type": "VarChar(64)", "#eq": "status" }
```

**R-QF-002** (MUST): For multi-value query parameters, use `#in` semantics — **never `#eq`**. Apply `#in` when ANY of the following is true:

- The OpenAPI parameter schema `type` is `array` — this is the primary signal; always use `#in` for array-typed parameters regardless of description wording
- The field name is a recognizable plural list form (`ids`, `categories`, `tag_ids`, `business_ids`, etc.)
- The OpenAPI description explicitly mentions "comma-separated", "list of", or "multiple values"

```
# Source: yelp
"bcategories": { "#in": "categories" }
"business_ids": { "#type": "VarChar(256)", "#virtual": true, "#mandatory": true, "#in": "business_ids" }
```

**Never use `#eq` for a parameter whose OpenAPI parameter schema type is `array`.**

**R-QF-003** (MUST): For date-range and numeric-range parameters, use typed comparison operators — NEVER `#eq`:

- Lower bound (`from_*`, `start_*`, `after_*`, `begin_*`, `date_from`, `since_*`) → `#ge` (inclusive lower bound). Use `#gt` only when the API semantics are strictly exclusive.
- Upper bound (`to_*`, `end_*`, `before_*`, `until_*`, `date_to`) → `#le` (inclusive upper bound). Use `#lt` only when the API semantics are strictly exclusive.

Using `#eq` on a date-range boundary parameter is always wrong — it would match only an exact timestamp value, not a range.

**R-QF-004** (MUST): Combine multiple filter parameters using logical AND by default. Emit explicit OR semantics only when the API documentation states that multiple parameters form an OR query.

**R-QF-005** (SHOULD): For free-text search parameters (`q`, `query`, `searchTerms`), map to `#contains` semantics rather than `#eq`.

**R-QF-006** (MUST): For `locator`-style compound query strings (teamcity), preserve the full locator template as a single virtual field rather than decomposing into individual filters. The locator handles all filtering internally.

**R-QF-007** (SHOULD): For APIs where filters exist, capture them in `.rest` as virtual fields (e.g., yelp's `blocation`, `elocation`, `bcategories`), preserve those virtual filter fields in the generated `.rest`.

**R-QF-008** (MUST): When a query parameter name DIFFERS from the response field name it conceptually represents, create a standalone `#virtual` field using the **parameter's actual name** — do NOT attach `#eq: "paramName"` to the differently-named response field. Attaching an `#eq` binding to a response field whose name does not match the query parameter name creates an invalid dual-role binding: the response field will be populated with the API's response value for the response field, not the query parameter value, and the query parameter will never be transmitted correctly.

Note: this is NOT a dual-role scenario (R-FC-005). The dual-role rule applies when a response field and a query parameter share the **same** name. When the names differ, the parameter cannot share a column with the response field — it must be a standalone virtual.

```json
// Scenario: response has "zip_code" field; query parameter is named "postal_code"
// WRONG — attaching postal_code #eq to the zip_code response field
"location": {
  "zip_code": { "#type": "VarChar(16)", "#eq": "postal_code" }
}

// CORRECT — zip_code is a plain response field; postal_code is a separate standalone virtual
"location": {
  "zip_code": "VarChar(16)"
},
"postal_code": {
  "#type": "VarChar(12)",
  "#virtual": true,
  "#eq": "postal_code"
}
```

**R-QF-009** (MUST): When a filter parameter accepts comma-separated/array values (`#in` semantics) AND a response field with the **same** JSON name exists, you CANNOT use the dual-role pattern. The `#in` operator is only valid on `#virtual: true` fields (see R-QF-002). Placing `#in` on a non-virtual response field is a schema error. Instead: keep the response field as a plain data column, and create a separate `*_filter` virtual field with `#in`.

```json
// Scenario: response has "price" field (VarChar); query parameter "price" is an array-type filter
// WRONG — #in placed on a non-virtual response field
"price": { "#type": "VarChar(8)", "#in": "price" }

// CORRECT — response field stays plain; separate virtual handles the array-filter
"price": "VarChar(8)",
"price_filter": {
  "#type": "VarChar(16)",
  "#virtual": true,
  "#in": "price"
}
```

This pattern applies whenever a response data column and an array-type query filter share the same API parameter name: `categories` + `categories_filter`, `attributes` + `attributes_filter`, etc.

## 12. Naming Conventions & Column Mapping

### Rules

**R-NC-001** (MUST): Preserve original JSON property names as SQL column names by default. Apply sanitization only when the name is not valid as a SQL identifier.

**R-NC-002** (MUST): Apply SQL name sanitization in this order:

1. Replace hyphens (`-`) with underscores (`_`).
2. Replace spaces with underscores.
3. Collapse consecutive underscores to single underscore.
4. If the resulting name is a SQL reserved keyword, append `_` suffix (e.g., `order` → `order_`, `type` → `type_`).
5. Record the mapping as `jsonName<sqlName>` notation in the `.rest` resource.

**R-NC-003** (MUST): When `.rest` source uses an explicit alias (e.g., `href<buildHref>`, `display_address<business_display_address>[]`), preserve that alias exactly. Do not regenerate aliases from JSON names.

**R-NC-007** (MUST): For child table alias names, follow the convention `<EntityName>_<fieldName>` (snake_case) where `<EntityName>` is the **exact name of the parent entity** as it appears in the `.rest` file, including any plural suffix. Never change plurality when constructing the prefix.

```
// Entity is named "businesses" (plural)
// CORRECT child table aliases:
"categories<businesses_categories>[]"
"hours<businesses_hours>[]"
"open<businesses_hours_open>[]"
"special_hours<businesses_special_hours>[]"

// WRONG — stripping the plural 's' from the entity name prefix:
"categories<business_categories>[]"
"hours<business_hours>[]"
```

**R-NC-008** (MUST): Preserve the plurality of all names exactly as they appear in the YAML. Do NOT change a plural parameter name or field name to singular or vice versa. This applies to virtual filter field names, child table aliases, and sub-entity names.

```
// YAML parameter name: "categories" (plural)
// CORRECT virtual filter name: "categories_filter"
// WRONG — changed to singular: "category_filter"

// YAML parameter name: "entities" (plural) in parent path "ai_chat_entities"
// CORRECT child table alias: "businesses<ai_chat_entities_businesses>[]"
// WRONG — changed to singular: "businesses<ai_chat_entity_businesses>[]"
```

**R-NC-009** (SHOULD): Field names starting with `@` denote XML/XSD attribute values (used in XML-response REST endpoints). Map using `jsonName<sqlName>` notation to provide a valid SQL column name:

```json
"@xsi:type<Type>": "VarChar(64)"
```

---

## 13. Arrays, Normalization & Flattening Policy

### Rules

**R-AN-001** (MUST): Apply the `SchemaFormat` setting to determine normalization policy:

- `NormalizeAll`: Every array of objects becomes a child table with `<alias>[]` notation.
- `Flatten`: All arrays are serialized inline as JSON or LongVarChar.
- `Mixed`: Normalize arrays of objects with stable schemas; flatten small/primitive arrays.

Default recommended `SchemaFormat` per source: see §19.

**R-AN-002** (MUST): When `SchemaFormat=NormalizeAll`, create child tables for all arrays of objects, regardless of nesting depth. Each child table must include:

- The parent identity field as a `#virtual` FK.
- A `#key` field if the child items have a unique identifier.

**R-AN-003** (SHOULD): Apply `SchemaFormat=Mixed` normalization threshold: if an array of objects has ≥ 3 distinct named properties and recurs across multiple rows, normalize to a child table.

**R-AN-004** (MUST): For arrays of primitive scalars (arrays of strings, integers, IDs), use the field's natural name with the `[]` suffix — do NOT invent or rename with a child-table alias. Only create named alias child tables for arrays of objects that carry actual sub-fields. Use `JSON` serialization only when primitives are heterogeneous or the array is unbounded.

```
# Source: zendesk — array of primitive strings, no alias renaming needed
"tags[]": "VarChar(64)"
```

**R-AN-005** (MUST): For deeply nested arrays (e.g., `sections[] → fields[]`), create a multi-level child table hierarchy following the aliasing pattern in §4 R-RM-009. Each level must carry FK virtual fields linking to its parent level.

**R-AN-006** (MUST): For polymorphic or schema-less arrays (`items: {}` or `additionalProperties: true`), emit `#type: JSON`. Do not attempt normalization of untyped arrays without reviewer approval.

**R-AN-007** (SHOULD): For arrays identified as small (average expected count ≤ 3 items) and containing only 1–2 primitive fields, consider `Flatten` treatment (inline as JSON string) rather than creating a separate child table, to reduce query joins.

**R-AN-008** (MUST): Ensure every normalized child table includes the full alias path in its `<alias>[]` declaration. The alias must be unique within the resource scope and must not collide with other resource names.

---

## 14. Advanced Features & Runtime Options

### Rules

**R-AF-001** (MUST): Preserve all `.rest` runtime directive flags verbatim when they exist in the source. Do not remove or normalize directives that are explicitly in scope for generation, such as `#sendonlyupdated` and `#headers`.

**R-AF-003** (MUST): Emit `#sendonlyupdated: true` on resources supporting partial update semantics (PATCH with sparse body). Detect from: YAML `application/merge-patch+json` content type, or `.rest` source containing `#sendonlyupdated`.

**R-AF-006** (MUST): Emit `#headers` only when the source explicitly requires custom non-authentication request headers. Map the header name to its value or placeholder.

```
# Source: quickbooks_payments
"#headers": { "Request-Id": "newRequest" }
```

---

## 15. HTTP Response Handling & Retry Rules

### Rules

**R-HR-001** (MUST): Use the `.rest` source's `#http` block as the authoritative HTTP status code to action mapping. When no `.rest` `#http` block exists, add HTTP handling only when the API documents non-default status behavior. The default fallback table is:

| HTTP Status | Default action    |
| ----------- | ----------------- |
| 200, 201    | `OK`              |
| 204         | `OK` (no content) |
| 400         | `ZERO_ROWS`       |
| 404         | `FAIL`            |
| 401         | `REAUTHENTICATE`  |
| 403         | `FAIL`            |
| 429         | `RETRY_AFTER`     |
| 500         | `RETRY_FIXED`     |
| 503         | `RETRY_AFTER`     |

When any non-default HTTP codes are present in the `#http` block, HTTP 200 MUST also be explicitly declared as `OK` in the same block. Do not rely on the driver default for 200 when other codes are explicitly mapped.

**R-HR-001b** (MUST): Emit `#http` at the global/root level only. Do NOT emit entity-level `#http` blocks.

**R-HR-007** (MUST): Only include HTTP status codes in the `#http` block that are explicitly documented in the API's YAML/OpenAPI specification. Do not add status codes that do not appear in the spec — if a code is not declared by the API, it is assumed the API does not return it and it does not need to be mapped.

**R-HR-002** (MUST): For 429 (rate limit) responses with a `Retry-After` header, configure `RETRY_AFTER` action and extract the `Retry-After` header value as the retry delay.

**R-HR-003** (SHOULD): For 503 responses, apply exponential backoff with a configurable maximum retry count (default 3). Do not apply infinite retry loops.

**R-HR-004** (MUST): For 401 (`REAUTHENTICATE`) responses, trigger the driver's token refresh flow before retrying the request. Emit `REAUTHENTICATE` in the `#http` mapping.

**R-HR-005** (MUST): When the YAML documents an error schema with a `message`, `description`, or `error` field for a status code, emit `#message` with the JSON path to that description field in the `#http` entry for that code. When the error applies only to a specific operation type, emit `#operation` (e.g., `"SELECT"`). Use `#match` only when a real response-body discriminator is explicitly required by the API behavior.

`#message` is only meaningful on `FAIL` and `ZERO_ROWS` actions. Do NOT add `#message` to `REAUTHENTICATE`, `RETRY_AFTER`, or `RETRY_FIXED` entries — those actions do not surface error text to the user.

**CRITICAL — error message JSON path:** The `#message` path (`{/error/description}`, `{/description}`, `{/message}`, etc.) MUST be derived by reading the API's actual error response schema. Do NOT default to either nested or root form without verification. If the schema defines a nested `error` object with `description`/`message`, use the nested path. If it defines `description`/`message` at the root, use the root path.

```
# Source: yelp — error response schema is flat: { "code": "...", "description": "..." }
# → description is at root
# WRONG: "#message": "{/error/description}"   ← no nested "error" object exists
# CORRECT: "#message": "{/description}"

# 404 handler MUST include #operation to scope it to read operations only:
{"#code": 404, "#action": "FAIL", "#operation": "SELECT", "#message": "{/description}"}
```

**R-HR-006** (MUST): For DELETE endpoints that return 204, mark the operation as successful without attempting to parse a response body. Emit the `#delete` directive without a response schema mapping.

---

## 16. DefaultVarcharSize Resolution

**C-001** — `DefaultVarcharSize` has source-specific variation, but the global default should be standardized.

**Resolution**:

- **Recommended global default**: `DefaultVarcharSize: 64`.
- Always override the default using field-name heuristics from R-TM-002 before falling back to the global default.

---

## 17. Timestamp Precision Resolution

**C-002** — Timestamp fractional precision varies across sources.

**Resolution**:

- **Recommended global default**: `Timestamp(0)`.
- Use higher precision only when the source explicitly shows sub-second precision.
- Use `BigInt` when the field is documented as a Unix epoch integer (not an ISO 8601 string).
- Never use `Timestamp` for epoch integers — this is a hard type error.
- **Format strings**: Only append an explicit format string to a `Timestamp` declaration when the API's date-time string format differs from the driver's built-in default (`yyyy-MM-dd'T'HH:mm:ssZ`) or its accepted space-separated equivalent (`yyyy-MM-dd HH:mm:ss`).
- **Space-separated equivalence assumption**: For this project, treat `yyyy-MM-dd HH:mm:ss` as equivalent to `yyyy-MM-dd'T'HH:mm:ssZ` (same timestamp shape with spaces instead of `T` and `Z`). Under this assumption, a format suffix is optional and SHOULD be omitted unless another source-specific parser requirement is known.
- **Detecting non-ISO formats**: MUST inspect the YAML `description` and `example` values for every `Timestamp` field. Add a format suffix only when values deviate beyond the accepted ISO forms above (for example, non-standard ordering, custom separators, localized month names, or fractional/offset patterns that are not handled by default).

```json
// CORRECT — accepted ISO-equivalent space-separated timestamp; format suffix may be omitted
"time_created": "Timestamp(0)"

// ALSO CORRECT — explicit format string is allowed when you want to pin parsing behavior
"time_created": "Timestamp(0),yyyy-MM-dd HH:mm:ss"

// CORRECT — non-ISO-equivalent format (day-first) requires explicit format suffix
"created_on": "Timestamp(0),dd-MM-yyyy HH:mm:ss"
```

---

## 18. Validation Checklist & Remediation

Run the following checks after generating any `.rest` resource mapping. Failed checks MUST be added to "Ambiguities & Manual Review" in the {fileName}-generation-status.md.

| #      | Check                            | Pass Condition                                                                                                                                                                                                                                           | Remediation if Fail                                                                    |
| ------ | -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| CHK-01 | All YAML paths mapped            | Every YAML path is in the endpoint mapping table (matched or UNMATCHED)                                                                                                                                                                                  | List unmatched paths in Ambiguities section; create stub resources                     |
| CHK-02 | IDENTITY resources have `#key`   | Every resource with `IDENTITY` in `#path` has at least one field marked `,#key`                                                                                                                                                                          | Derive `#key` from path placeholder; escalate for manual review if ambiguous           |
| CHK-03 | No auth in virtual headers       | No auth token, API key, or credentials appear as `#virtual` header fields                                                                                                                                                                                | Move to `#options` block; remove from field definitions                                |
| CHK-04 | Pagination directives complete   | List resources have `#pageSizeParameter` or `#nextPageElement` set (or confirmed no-paging)                                                                                                                                                              | Add appropriate pagination directive from detected style (R-PG-002)                    |
| CHK-05 | Type validity                    | All `.rest` SQL types are valid (see full type table in §1.10)                                                                                                                                                                                           | Correct invalid types; default to `VarChar(64)` for unresolvable cases                 |
| CHK-06 | Epoch timestamps typed correctly | Fields storing Unix epoch integers use `BigInt`, not `Timestamp`                                                                                                                                                                                         | Re-type to `BigInt`;                                                                   |
| CHK-07 | Required parameters flagged      | OpenAPI `required: true` parameters that are required across **all** paths in the entity have `#mandatory`. Parameters required on only a subset of paths do NOT get `#mandatory` — the path URL template controls per-path transmission (see R-FC-002). | Add `#mandatory` only to fields required across every path variant                     |
| CHK-08 | Child tables have FK             | Normalized child table resources include a `#virtual` FK field to parent identity                                                                                                                                                                        | Add parent ID as `#virtual` FK; name it consistently with parent's `#key` field        |
| CHK-09 | Ambiguities documented           | All Ambiguities written to {fileName}-generation-status.md                                                                                                                                                                                               | Write the ambiguity entry with recommended action                                      |
| CHK-10 | `#http` block present            | APIs with non-default rate-limit or auth failure codes have `#http` status mappings                                                                                                                                                                      | Copy `#http` from `.rest` source or add only the required non-default status handling  |
| CHK-11 | Multipart endpoints flagged      | POST endpoints using `multipart/form-data`                                                                                                                                                                                                               | Document field structure                                                               |
| CHK-12 | No duplicate aliases             | All `<alias>[]` names are unique within a resource scope                                                                                                                                                                                                 | Rename duplicates using `<ParentResource>_<fieldName>` convention                      |
| CHK-13 | No comments in output            | Generated `.rest` file contains no `//` comment lines                                                                                                                                                                                                    | Remove all comment lines from generated output (see R-SX-004)                          |
| CHK-14 | No trailing commas               | Generated `.rest` JSON has no trailing commas                                                                                                                                                                                                            | Fix JSON formatting                                                                    |
| CHK-15 | Swagger boundary respected       | Only auto-derivable elements are emitted; manual-only items flagged for review                                                                                                                                                                           | Add manual-only items to {fileName}-generation-status.md Ambiguities section (see §21) |
| CHK-16 | No YAML response fields dropped  | Every field present in the OpenAPI response schema for an endpoint exists in the `.rest` resource definition                                                                                                                                             | Re-add missing fields; document if intentionally excluded in Ambiguities section       |

---

## 18.1 Decimal Precision/Scale Separator Standard

**C-006** — Decimal precision/scale separator should be standardized.

**Resolution**: Generated output MUST use the comma form `Decimal(p,s)`.

---

## 19. Configuration & Generator Settings

The following configuration knobs and their recommended defaults apply globally. Per-source langspecs may override any of these.

| Knob                      | Recommended Default | Notes                                                                                                                        |
| ------------------------- | ------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `DefaultVarcharSize`      | `64`                | See C-001; override per-source as needed                                                                                     |
| `TimestampPrecision`      | `0`                 | See C-002; increase when sub-second precision confirmed                                                                      |
| `SchemaFormat`            | `Mixed`             | Use `NormalizeAll` explicitly when `.rest` demonstrates it                                                                   |
| `JSONType`                | `JSON`              | Use `LongVarChar` for downstream systems lacking JSON support                                                                |
| `NormalizeThreshold`      | `3`                 | Minimum property count for normalizing arrays to child tables                                                                |
| `MaxVarcharForURL`        | `512`               | VarChar size for URL/URI/Link field names                                                                                    |
| `MaxVarcharForLargeText`  | `LongVarChar`       | Fields named `description`, `body`, `content`, `bio`, `excerpt` always use LongVarChar — never VarChar(N) regardless of size |
| `ComponentReuseThreshold` | `3`                 | Minimum reference count before creating shared component                                                                     |
| `DefaultMaxPageSize`      | `100`               | Default `#maximumPageSize` when not specified in source or YAML                                                              |
| `FirstPageNumber`         | `1`                 | Default `#firstPageNumber`; override to `0` for 0-indexed APIs                                                               |
| `retry.maxAttempts`       | `3`                 | Maximum retry attempts for transient (5xx) errors                                                                            |

---

## 20. Syntax Constraints: No Inline Objects Except `#http` Child Rules (R-SX-008)

Nested objects and arrays MUST be fully expanded with each entry on its own indented line. This applies at every nesting depth, with one exception: `#http` child rule objects may remain one-line objects for readability when the `#http` array itself is expanded.

```json
// WRONG — nested objects and arrays written inline
"coordinates": { "latitude": "Double", "longitude": "Double" },
"#http": [{"#code": 200, "#action": "OK"}, {"#code": 401, "#action": "REAUTHENTICATE"}],
"display_address[]": "VarChar(128)"

// CORRECT — every nested object fully expanded; each entry on its own line
"coordinates": {
  "latitude": "Double",
  "longitude": "Double"
},
"#http": [
  {"#code": 200, "#action": "OK"},
  {"#code": 401, "#action": "REAUTHENTICATE"}
],
"display_address[]": "VarChar(128)"
```

**Exception**: `#http` child entries MAY remain on one line (for example, `{"#code": 200, "#action": "OK"}`), but the `#http` array itself MUST be expanded (one entry per line). All non-`#http` nested objects/arrays must remain fully expanded with closing `}` / `]` on their own lines.

---

## 21. Swagger / OpenAPI Translation Boundary

This section defines what `.rest` configuration CAN be automatically derived from an OpenAPI/Swagger document versus what REQUIRES manual configuration. The ARC Generator Agent MUST only emit directives it can confidently derive; it MUST NOT invent or guess values for manual-only items.

### 21.1 Derivable from Swagger (MAY be auto-generated)

| `.rest` Element                 | Swagger Source                                                  | Notes                                                                              |
| ------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `#hostname`                     | `servers[0].url`                                                | Extract scheme + host only; strip trailing slash; do not include version/base path |
| `#version`                      | API URL version segment in base path/paths or `servers[0].url`  | Do not use `info.version`; emit only when unambiguous                              |
| `#options.authenticationmethod` | `securitySchemes` type mapping                                  | Use mapping table in R-AU-003                                                      |
| `#options.apiKeyName`           | `securitySchemes` with `in: header` or `in: query`              | ApiKey parameter name                                                              |
| `#options.tokenURI`             | `flows.clientCredentials.tokenUrl`                              | OAuth2 token URL                                                                   |
| `#options.authURI`              | `flows.authorizationCode.authorizationUrl`                      | OAuth2 auth URL                                                                    |
| `#http`                         | Inferred from documented 4xx/5xx responses                      |                                                                                    |
| Resource name                   | OpenAPI path last meaningful segment + pluralization heuristics |                                                                                    |
| `#path` (read)                  | All `GET` paths from OpenAPI                                    | Preserve path variables as `{name}`                                                |
| `#insert`                       | `POST` path on resource collection                              | When POST semantics are unambiguous                                                |
| `#update`                       | `PUT`/`PATCH` path on `{id}` endpoint                           |                                                                                    |
| `#delete`                       | `DELETE` path on `{id}` endpoint                                |                                                                                    |
| Field names                     | Response schema property names                                  |                                                                                    |
| Field types                     | OpenAPI schema type/format + mapping table in §8                |                                                                                    |
| `#key`                          | Path parameter used in identity/update/delete endpoints         |                                                                                    |
| `#virtual`                      | Path and query parameters not present in the response schema    |                                                                                    |
| `#mandatory`                    | Required path parameters                                        |                                                                                    |
| Pagination directives           | Query parameters matching known paging patterns (R-PG-002)      |                                                                                    |
| `#maximumPageSize`              | OpenAPI parameter `maximum` constraint                          |                                                                                    |
| Child table arrays              | `array` type properties in response schema                      |                                                                                    |
| `<alias>[]` naming              | Schema component names or field names                           |                                                                                    |

### 21.2 Manual Configuration Only

Manual-only `.rest` elements and review guidance have been moved to [manual-rest-adjustments.md](manual-rest-adjustments.md). Use that file for hand-edited items that are not derivable from Swagger.

### 21.3 Ambiguity Cases (Emit with Manual Review Note)

When the following are encountered, the Generator Agent MUST emit a best-guess value AND add an entry to the `{fileName}-generation-status.md` for human review:

- `#maximumPageSize` when no YAML `maximum` constraint exists
- `#firstPageNumber` when API docs are ambiguous about 0 vs 1 indexing
- `#http` rules beyond the standard defaults (novel status codes)
- CRUD method when multiple HTTP verbs are present on the same path
- Child table normalization strategy when response schema is deeply nested or polymorphic

---

*End of Global REST Language Specification*
*Updated: 2026-04-09| Sources: 40+ API models, REST Syntax Wiki Documentation, Source Code Analysis*
