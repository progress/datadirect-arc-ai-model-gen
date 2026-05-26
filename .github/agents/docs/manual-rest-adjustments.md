---
title: "Manual REST Configuration Updates"
generated_on: "2026-03-30T00:00:00Z"
version: "1.0"
updated_on: "2026-04-14T00:00:00Z"
---

# Manual REST Configuration Updates

This guide lists `.rest` concepts that often need manual edits after generation, especially when AI output is ambiguous or non-deterministic or when your API needs capabilities that are not represented in the Swagger/OpenAPI file.

> **Official reference:** For complete syntax and authoritative `.rest` guidance, use the [Progress Autonomous REST Connector documentation](https://docs.progress.com/bundle/datadirect-autonomous-rest-connector-jdbc-60/page/Welcome-to-the-Progress-DataDirect-Autonomous-REST-Connector-for-JDBC.html)

## Table of Contents

- [How To Use This File](#how-to-use-this-file)
- [Verification Checklist Before Saving](#verification-checklist-before-saving)
- [Common Concepts (Review First)](#common-concepts-review-first)
  - [Pagination And Result-Shape](#pagination-and-result-shape)
  - [Query And Filtering](#query-and-filtering)
  - [Authentication](#authentication)
  - [Runtime/Protocol Behavior](#runtimeprotocol-behavior)
  - [Schema And Metadata](#schema-and-metadata)
- [Less Common Concepts](#less-common-concepts)
- [Uncommon Concepts (Specific APIs Only)](#uncommon-concepts-specific-apis-only)
- [Advanced/Manual-Only Patterns](#advancedmanual-only-patterns)
- [Reference: Pagination Parameters](#reference-pagination-parameters)

## How To Use This File

- Start with generated output.
- Apply only the concepts your API actually needs.
- If behavior is uncertain from OpenAPI alone, verify with a real API call or the API documentation.

## Verification Checklist Before Saving

⚠️ **Critical** - These are the minimum checks required for a working `.rest` model to ensure proper ability to connect to and query the API.

- Primary key choice is explicit and not guessed.
- Paging behavior matches observed runtime responses.
- Root extraction points to actual row arrays/objects.
- Authentication flow is executable end-to-end (including reauthentication if needed).
- Write rules (`#readonly`, `#insert`, `#update`, `#sendOnlyUpdated`) match API behavior.
- Filter operators and query parameter mappings are validated against actual endpoint behavior.

## Common Concepts (Review First)

### Pagination And Result-Shape

#### Pagination Directives

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness and may still need manual updates.
- **Commonality:** Very common
- ⚠️ **Critical** - Incorrect or incomplete paging values may result in errors when attempting to retrieve API data.
- **What it is:** Paging controls used by the connector to request and parse paginated API results.
- **When to use:** OpenAPI does not fully encode real paging behavior, or runtime testing shows additional knobs are required. Always review against real API responses (parameter names, index base, continuation behavior, totals, and max limits).
- **Note:** See [Reference: Pagination Parameters](#reference-pagination-parameters) for available paging parameters and their uses.
- **Example:**

```json
"Boards": {
  "#path": ["/rest/agile/1.0/board /values"],
  "#maximumPageSize": 1000,
  "#pageSizeParameter": "maxResults",
  "#rowOffsetParameter": "startAt",
  "#pageSizeElement": "maxResults",
  "#totalRowsElement": "total"
}
```

#### Response Root Extraction (`#path` root suffix)

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Common
- **What it is:** A space + alias token appended to a `#path` (or `#insert`) entry tells the parser where rows or a single object live in the response. Works for list endpoints, IDENTITY single-object wrappers, and insert responses.
- **When to use:** Response body wraps items or a single object under a named key.
- **Examples:**

```json
// List root extraction (GET returns { "records": {...} }):
"#path": ["/rest/api/2/auditing/record /records"]

// Insert response wrapped under a key:
"#insert": "POST /api/v1/items /result"

// IDENTITY single-object wrapper:
"Payments": {
  "#path": ["IDENTITY /v2/payments/{id} /payment"],
  "id": "VarChar(64),#key"
}
```

### Query And Filtering

#### Dual-Role Fields + Operator Mapping

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness and may still need manual updates.
- **Commonality:** Common
- **What it is:** Some fields appear in both the API response schema (as returned columns) and as filter query parameters. Operator mapping (`#eq`, `#ne`, `#ge`, `#le`, `#contains`, `#in`) translates SQL predicates to API query parameter names.
- **When to review:**
  - If a generated file has a field like `latitude_filter` as a `#virtual` field, but `latitude` also appears in the response schema, remove the virtual alias and add mapping directly to the data column.
  - Use mapping when API filter names are non-standard or explicit operator translation is required.
- **Note:** Operator mapping is only required on non-virtual fields when the field name differs from the name required by the query parameter, but adding it can add clarity where query parameters are present in the response.
- **Example (correct):**

```json
"latitude": {
  "#type": "Double",
  "#eq": "latitude"
}
```

- **Incorrect:**

```json
"latitude": "Double",
"latitude_filter": {
  "#virtual": true,
  "#type": "Double",
  "#eq": "latitude"
}
```

- **Example (range bounds on a timestamp field):**

```json
"created": {
  "#type": "Timestamp(3)",
  "#ge": "from",
  "#le": "to"
}
```

#### Virtual Query Parameter Defaults (`#default`)

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Common
- **What it is:** A static default value pre-filled into a virtual query parameter when the user does not supply one.
- **When to use:** The API has query params where a sensible default improves usability. The OpenAPI spec may have a default that is not appropriate for your environment.
- **Example:**

```json
"location": {
  "#type": "VarChar(64)",
  "#virtual": true,
  "#eq": "location",
  "#default": "New York, NY"
}
```

#### JSON/SQL Name Mapping (`jsonName<sqlName>`)

- **Update guidance:** The AI may have generated this, or it may need manual edits depending on the API.
- **Commonality:** Common
- **What it is:** Keeps the JSON field name while using a different SQL-safe column name.
- **When to use:** JSON property names conflict with SQL naming rules, readability goals, or existing table expectations.
- **Example:**

```json
"id<boardId>": "BigInt,#key",
"groups<UserGroups>[]": "VarChar"
```

### Authentication

#### Authentication Selection + Options (`#options.authenticationMethod`)

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness and may still need manual updates.
- **Commonality:** Common
- ⚠️**Critical** - Incorrect authentication values may result in being unable to connect to the API.
- **What it is:** Selects which authentication mode(s) the connector should expose/default and configures file-level authentication endpoints/values.
- **When to use:** Your API supports one or more authentication mechanisms (OAuth2, bearer token, basic, header key, URL parameter key, etc.), and generation cannot safely choose final UX defaults alone. Also use when auth values are missing, relative, environment-specific, or tenant-specific.
- **Note:** `authenticationMethod` is an optional add-on to the `.rest` file. The Generator Agent will attempt to add this section if it is present in the OpenAPI specification, but users may also choose to have their authentication options present only in the connection string, in which case they are not required in the `.rest` file.
- **Example:**

```json
"#options": {
  "authenticationMethod": {
    "choices": "OAuth2,BearerToken",
    "default": "OAuth2"
  }
}
```

- **Example (OAuth2 options):**

```json
"#options": {
  "authenticationMethod": "OAuth2",
  "tokenURI": "https://login.example.com/oauth2/token",
  "authURI": "https://login.example.com/oauth2/authorize",
  "redirectURI": "https://localhost/oauth2/callback",
  "logoffURI": "https://login.example.com/oauth2/logout",
  "scope": "read write"
}
```

### Runtime/Protocol Behavior

#### HTTP Handling Overrides (`#http`)

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Common
- **What it is:** Status-code action mapping (for example `OK`, `ZERO_ROWS`).
- **When to use:** Source API semantics differ from default assumptions, or operation-specific status handling is required.
- **Example:**

```json
"#http": [
  {"#code": 200, "#action": "OK"},
  {"#code": 400, "#action": "ZERO_ROWS"},
  {"#code": 401, "#action": "REAUTHENTICATE"},
  {"#code": 404, "#action": "ZERO_ROWS"},
  {"#code": 429, "#action": "RETRY_AFTER"},
  {"#code": 503, "#action": "RETRY_AFTER"}
]
```

#### HTTP Error Message Path Verification (`#message`)

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Common
- **What it is:** The `#message` value is a JSON path expression that extracts a human-readable error string from the API's error response body.
- **When to review:** The JSON path must be derived from the actual error response schema. Verify by reading the swagger's error schema. `#message` is only valid on `FAIL` and `ZERO_ROWS`.
- **Example (nested error object — `{ "error": { "description": "..." } }`):**

```json
{"#code": 400, "#action": "ZERO_ROWS", "#message": "{/error/description}"},
{"#code": 403, "#action": "FAIL",      "#message": "{/error/description}"} 
```

### Schema And Metadata

#### API Version (`#version`)

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Common
- **What it is:** A top-level metadata string surfaced in driver metadata to identify the API version the connector targets.
- **When to use:** The API has a well-defined version identifier (e.g., `v2`, `2023-10`) that should be visible in connector metadata. Derive from `info.version` in the swagger or from the version path segment. Omit when the API has no meaningful version.
- **Example:**

```json
"#version": "v2"
```

## Less Common Concepts

### Field Projection Parameter (`#fieldListParameter`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Less common
- **What it is:** Declares the parameter name used to send selected/projected field lists.
- **When to use:** API supports sparse field sets and requires a specific parameter name (for example `fields`) to request selected columns.
- **Example:**

```json
"Issues": {
  "#path": ["/api/v1/issues /items"],
  "#fieldListParameter": "fields"
}
```

### Sandbox / Test Hostname -> Production Substitution

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Less common
- ⚠️**Critical** - If `#hostname` points to the wrong environment (for example sandbox instead of production), connector requests will fail or target the wrong data.
- **What it is:** Many swagger files reference a sandbox or test base URL. Generation copies this into `#hostname` verbatim.
- **When to review:** Check `#hostname` in every generated file before production use — the generated file will continue calling the sandbox until updated. The production URL is typically in the API provider's developer portal.
- **Example:**

```json
// Generated — sandbox URL (replace before production use):
"#hostname": "https://api.sandbox.paypal.com"

// After substitution:
"#hostname": "https://api.paypal.com"
```

### Text-Search Parameters: `#contains` vs `#eq`

- **Update guidance:** The AI usually does not generate this correctly; add or fix it manually if your API requires it.
- **Commonality:** Less common
- **What it is:** The `#contains` operator signals substring/prefix matching semantics (autocomplete, free-text search), while `#eq` signals exact/equality matching. The generator cannot reliably distinguish between them from the YAML schema alone — both typically appear as plain `string` parameters with no special format hint.
- **When to review:** Any parameter whose purpose is autocomplete, prefix completion, or full-text search should use `#contains`. The generator will often default to `#eq` for string parameters. Verify and update manually after generation.
- **Example:**

```json
"text": {
  "#type": "VarChar(256)",
  "#virtual": true,
  "#mandatory": true,
  "#contains": "text"
} 
```

### HTTP Operation Scoping (`#operation`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Less common
- **What it is:** Scopes an `#http` rule to a specific SQL operation (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, or service operations like `LOGIN`) so the same HTTP status code triggers different actions depending on which operation is running.
- **When to use:** The API returns the same code with different meanings by operation. For example, a `404` on `SELECT` means "no rows" but on `DELETE` means "already gone — treat as success".
- **Example:**

```json
"#http": [
  {"#code": 200, "#action": "OK"},
  {"#code": 404, "#action": "ZERO_ROWS", "#operation": "SELECT", "#message": "{/description}"},
  {"#code": 404, "#action": "OK",        "#operation": "DELETE"},
  {"#code": 401, "#action": "REAUTHENTICATE"},
  {"#code": 429, "#action": "RETRY_AFTER"}
] 
```

### Global/Resource Headers (`#headers`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Less common
- ⚠️ **Critical** - (API specific) Some APIs require specific headers to be sent in requests. Not including these could prevent successful querying of the API.
- **What it is:** Static request headers read from config and attached to outgoing requests.
- **When to use:** API requires non-authentication headers for all or specific calls.
- **Example** (global, applies to all resources in the file):

```json
"#headers": {
  "X-Api-Version": "2023-01",
  "Accept": "application/vnd.example+json"
}
```

- Resource-level headers override the global ones for that resource only:

```json
"SpecialResource": {
  "#path": ["/api/v1/special"],
  "#headers": {
    "X-Feature-Flag": "beta"
  }
} 
```

### Key Strategy When No Natural Primary Key Exists

- **Update guidance:** The AI usually does not generate this reliably; add or complete it manually if your API requires it.
- **Commonality:** Less common
- ⚠️**Critical** - (when no key is otherwise available) Primary keys are required for every endpoint for proper processing by the AutoREST driver.
- **What it is:**
  - **Synthetic key (`#rowid` / `rowid()`)**: a generated surrogate key when no reliable key exists.
  - **Compound keys (`#key` on multiple fields)**: explicit multi-field identity when no single field identifies a row.
- **When to use:**
  - No stable key can be derived from path parameters or payload fields.
  - Resource identity requires a combination of fields (for example, `project_id` + `issue_number`).
- **Example (synthetic key):**

```json
"ROWID": "VarChar(32)=rowid(),#key"
```

- **Example (compound key):**

```json
"ProjectIssues": {
  "#path": ["IDENTITY /api/v1/projects/{project_id}/issues/{issue_number}"],
  "project_id": "BigInt,#key",
  "issue_number": "Integer,#key",
  "title": "VarChar(256)"
} 
```

### Timestamp Format Strings

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Less common
- **What it is:** An explicit format string appended to a `Timestamp` type declaration that tells the driver how to parse the field's string representation.
- **When to use:** The API returns a date-time string in a **non-default** format. Only add the format string if the format differs from the driver's built-in default (ISO 8601 / `yyyy-MM-dd'T'HH:mm:ssZ`). The YAML description or example values will indicate the format.
- **Example:**

```json
"time_created": "Timestamp(0),dd-MM-yyyy HH:mm:ss" 
```

### Read-Only (`#readonly`)

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Less common
- **What it is:** Marks a field as not createable/updateable by default CRUD logic.
- **When to use:** API returns a field that must not be sent on writes.
- **Example:**

```json
"createdAt": "Timestamp(3),#readonly" 
```

### Non-Nullable Fields (`#notnull` / `#nonnull`)

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Less common
- **What it is:** Marks a response field as guaranteed non-null by the API contract. Both spellings (`#notnull` and `#nonnull`) are accepted by the runtime.
- **When to use:** The YAML response schema marks a property as `required` (meaning the API always returns a value — it is never absent or null).
- **Example:**

```json
"name": "VarChar(128),#notnull" 
```

### Polymorphic Fields (`oneOf` / `anyOf`) as JSON

- **Update guidance:** The AI may have generated this, but it must be reviewed for correctness.
- **Commonality:** Less common
- **What it is:** Fields whose swagger schema uses `oneOf` or `anyOf` are generated as `JSON` type because their structure cannot be statically determined. Users may want to expand these once the real API response shape is confirmed.
- **When to use:** You see a `JSON`-typed field in the generated output that in practice always returns one concrete structure. Replace `JSON` with the actual sub-fields after confirming the shape from real API responses or authoritative documentation. If the API truly can return different structures for the same field, `JSON` is correct.
- **Example:**

```json
// Generated — polymorphic schema; left as JSON:
"metadata": "JSON"

// After confirming the shape is always { type, value }:
"metadata": {
  "type": "VarChar(32)",
  "value": "LongVarChar"
}
```

## Uncommon Concepts (Specific APIs Only)

> These concepts are **uncommon** and usually required only by specific APIs. Check your API documentation to determine whether any apply.

### Fixed Static POST Body Parameters (`#post`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- **What it is:** A resource-level object that injects fixed key-value pairs into every POST request body for that resource, regardless of user-supplied fields.
- **When to use:** The API requires constant body parameters on every request (e.g., a fixed query type, a constant `fields` list, or a static action name) that should not be exposed as user-configurable fields.
- **Example:**

```json
"Reports": {
  "#path": ["/api/v1/reports"],
  "#post": {
    "query": "all",
    "fields": ["id", "name", "status"]
  },
  "id": "BigInt,#key",
  "name": "VarChar(128)"
} 
```

### Custom Multi-Step Authentication (`#authentication`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- ⚠️**Critical** (if custom authentication flow is required)
- **What it is:** Explicit scripted authentication sequence for non-standard login flows.
- **When to use:** Standard auth modes are insufficient (multi-request token exchange, custom headers, staged credential exchange).
- **Example** (POST credentials, extract token from response, set as header):

```json
"#authentication": [
  "content-type=application/json",
  "accept=application/json",
  {"user": "{user}", "password": "{password}"},
  "POST https://api.example.com/auth",
  "HEADER Authorization=Bearer {/token}"
] 
```

### Reauthentication (`#reauthentication`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- **What it is:** Explicit re-login/re-token flow used after auth expiry or auth failures.
- **When to use:** API requires a different token-refresh or reauthentication sequence than initial authentication.
- **Example** (refresh using existing token, replace header):

```json
"#reauthentication": [
  "content-type=application/json",
  "accept=application/json",
  {"user": "{user}", "password": "{password}", "token": "{/credentials}"},
  "POST https://api.example.com/reauth",
  "HEADER Authorization=Bearer {/credentials}"
] 
```

### Header Columns (`#header`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- ⚠️**Critical** - Header-mapped fields must be correct for APIs that require request values to be supplied as headers.
- **What it is:** Marks fields as HTTP header values.
- **When to use:** The API contract requires specific request values as HTTP headers.
- **Example:**

```json
"NotionVersion": {
  "#type": "VarChar(16)",
  "#virtual": true,
  "#header": true,
  "#eq": "Notion-Version"
} 
```

### Unique (`#unique`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- **What it is:** Adds uniqueness semantics for non-primary-key fields when needed.
- **When to use:** Backend contract guarantees uniqueness but the field is not modeled as the sole primary key.
- **Example:**

```json
"email": {
  "#type": "VarChar(256)",
  "#unique": true
} 
```

### Omit Empty (`#omitWhenEmpty`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- ⚠️**Critical** - This setting controls whether empty payload structures are sent or omitted; incorrect behavior can cause write requests to fail for APIs that require explicit empty containers.
- **What it is:** Controls whether empty post bodies are sent.
- **When to use:** API requires explicit empty containers as a `POST` body instead of omitting them.
- **Example:**

```json
"#post": {
  "#omitWhenEmpty": false
} 
```

### Extract Pattern (`#extract`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- **What it is:** Regex extraction pattern for transforming/parsing field values.
- **When to use:** Raw response value needs regex extraction before column mapping.
- **Example** (extract a semantic version string from a longer value; backslashes are JSON-escaped):

```json
"version": {
  "#type": "VarChar(20)",
  "#extract": "\\d+\\.\\d+\\.\\d+"
} 
```

### Post Parameter Mode (`#postParameter`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- **What it is:** Field-level POST behavior mode (`replace`, `json`).
- **When to use:** API expects specific POST payload substitution/embedding behavior for that parameter.
- **Example (`json` mode sends the field value merged into the post body):**

```json
"filter": {
  "#type": "LongVarChar",
  "#virtual": true,
  "#postParameter": "json"
}
```

### Send Only Updated (`#sendOnlyUpdated`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- ⚠️**Critical** - This controls full vs partial update behavior; an incorrect value can overwrite unchanged fields or break partial-update semantics.
- **What it is:** Write behavior toggle controlling whether only changed fields are sent.
- **When to use:** PATCH-like semantics are required even when update endpoint/method does not imply it by default.
- **Example:**

```json
"Users": {
  "#path": ["IDENTITY /api/v1/users/{id}", "/api/v1/users"],
  "#update": "PUT /api/v1/users/{id}",
  "#sendOnlyUpdated": true,
  "id": "BigInt,#key",
  "name": "VarChar"
} 
```

### Fixed Retry Delay (`#fixedDelay`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- **What it is:** Fixed retry-delay configuration value read at config level.
- **When to use:** API throttling/retry policy requires a fixed delay override.
- **Example (value is milliseconds):**

```json
"#fixedDelay": 1000 
```

### Chunked JSON-Line Parsing (`#chunked`)

- **Update guidance:** The AI usually does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- **What it is:** Enables JSON-lines style parsing mode for responses.
- **When to use:** Endpoint returns chunked/line-delimited JSON rather than a normal JSON array/object payload.
- **Example:**

```json
"StreamResults": {
  "#path": ["/api/v1/stream"],
  "#chunked": true,
  "id": "BigInt,#key",
  "value": "VarChar"
}
```

### Typed map-key syntax - Advanced Pattern

- **Update guidance:** The AI does not generate this; add it manually if your API requires it.
- **Commonality:** Uncommon
- **What it is:** Declares a JSON object as a dynamic key/value map where keys are not known ahead of time, but key and value types are explicitly defined.
- **When to use:** The payload contains an object whose property names vary by row and you 
  still need deterministic typing.
- **Example:**

```json
"attributes<key:VarChar(64)>": "Integer" 
```

- **Example API response shape this fits:**

```json
{
  "attributes": {
    "height": 180,
    "weight": 75,
    "rating": 5
  }
}
```

## Reference: Pagination Parameters

| Parameter                 | Purpose                                                                   | Paging Type  |
| ------------------------- | ------------------------------------------------------------------------- | ------------ |
| `#maximumPageSize`        | Max rows requested per page.                                              | General      |
| `#pageSizeParameter`      | Request parameter name for page size.                                     | General      |
| `#pageSizeElement`        | Response element that echoes/returns page size.                           | General      |
| `#postBodyPaging`         | Sends paging controls in POST body instead of query string.               | General      |
| `#totalRowsElement`       | Response element containing total row count.                              | General      |
| `#rowOffsetParameter`     | Request parameter name for row-offset paging.                             | Offset       |
| `#firstRowNumber`         | Base row index used with row offsets (`0` or `1`).                        | Offset       |
| `#pageNumberParameter`    | Request parameter name for page-number paging.                            | Page number  |
| `#firstPageNumber`        | Base page index (`0` or `1`).                                             | Page number  |
| `#totalPagesElement`      | Response element containing total page count.                             | Page number  |
| `#nextPageParameter`      | Request parameter for next-page token/URL when token is query/body based. | Token        |
| `#nextPageElement`        | Response element containing next-page token/URL.                          | Token        |
| `#nextPageResponseHeader` | Response header containing next-page token/URL.                           | Header token |
| `#nextPageRequestHeader`  | Request header used to send next-page token/URL.                          | Header token |
