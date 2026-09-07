# Sessions API

Endpoints for the service-to-service (S2S) authentication flow — exchanging application credentials for an access token, creating a session on behalf of a user, and closing sessions. See [Authentication](../getting-started/authentication.md) for how this fits alongside the OAuth bearer-token and HTTP pass-through options, and [Architecture](../concepts/architecture.md#applications-and-the-s2s-flow) for how an Application is registered in the first place.

!!! warning "Partially verified 2026-09-07 against TM1 12.6.4"
    Confirmed live against a local test instance: both endpoints exist, and their `401` error responses (below) are real. The success paths (a valid access token, and an actual `TM1SessionId` cookie from `Create a session`) were **not** exercised — that requires a registered Application/root credentials, which weren't available in the test environment.

!!! warning "Deprecated endpoint"
    `POST /{instance}/auth/v1/session` still exists but is deprecated. Use the two endpoints below instead.

## Get an access token

Exchanges an [Application's](management.md#register-an-application) `ClientID`/`ClientSecret` for an access token, using the OAuth2 client-credentials grant.

```http
POST /{instance}/auth/v1/token
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `grant_type` | string | Yes | Always `client_credentials`. |
| `client_id` | string | Yes | The Application's `ClientID`. |
| `client_secret` | string | Yes | The Application's `ClientSecret`. |

```bash
curl -X POST "https://<host>/test/auth/v1/token" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\": \"client_credentials\", \"client_id\": \"$APP_CLIENT_ID\", \"client_secret\": \"$APP_CLIENT_SECRET\"}"
```

### Response

`200 OK` with an access token in the response body. Use it as the bearer token in [Create a session](#create-a-session) below.

!!! warning "Unverified"
    The exact response field name for the token has not been confirmed against a live response — only that the endpoint exists and rejects bad credentials (below).

### Errors

Confirmed against a live instance with an invalid `client_id`/`client_secret`:

```
401 Unauthorized
{ "error": "invalid_client", "error_details": "Invalid credentials provided" }
```

This is a plain OAuth-style error envelope, distinct from the `{"error": {"code": ..., "message": ...}}` shape used by the rest of the Service API (including [Create a session](#create-a-session) below).

## Create a session

Creates a session for a specific user, using the access token obtained above. This is how a trusted Application (e.g. Planning Analytics Workspace) acts on behalf of a user without that user authenticating directly.

```http
POST /{instance}/api/v1/Sessions
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `UserName` | string | Yes | The TM1 user to create the session for. Must already exist in the target database — see [Create a user](users.md#create-a-user). |
| `Variables` | object | No | Session variables as key/value pairs, consumable in TI processes via `SessionVariableGet()`. Combined key+value size cannot exceed 8192 bytes. Available as of v12.4.0. |

```bash
curl -X POST "https://<host>/test/api/v1/Sessions" \
  -H "Authorization: Bearer $S2S_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"UserName\": \"user@email.com\"}"
```

With session variables:

```bash
curl -X POST "https://<host>/test/api/v1/Sessions" \
  -H "Authorization: Bearer $S2S_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"UserName\": \"docs@ibm.com\", \"Variables\": {\"url\": \"https://test.com\", \"bearer_token\": \"a65ithj5f413aru094561kdme=\"}}"
```

### Response

Sets a `TM1SessionId` cookie in the response. Attach it as `Cookie: TM1SessionId=<value>` on all subsequent requests instead of an `Authorization` header — see [Using a session](../getting-started/authentication.md#using-a-session).

Session variables are stored encrypted and are cleared when the session expires.

!!! note
    Creating a session at the service level doesn't log the user into any specific database — the database-level session is created lazily on the first request to that database.

### Errors

Confirmed against a live instance with no `Authorization` header:

```
401 Unauthorized
Www-Authenticate: Bearer realm="TM1"
{ "error": { "code": "278", "message": "Authentication required" } }
```

| Status | Meaning |
|---|---|
| `400 Bad Request` | `UserName` is missing, or `Variables` exceeds the 8192-byte combined limit. |
| `401 Unauthorized` | The access token is missing, expired, or invalid — confirmed above. |
| `404 Not Found` | `UserName` doesn't exist in any database on this instance. |

!!! warning "Unverified"
    Only the `401` case (missing token) has been confirmed live. `400` and `404` are inferred from conventions used elsewhere in this API — testing them requires a real registered Application, which wasn't available.

## Close a session

| Scope | Endpoint |
|---|---|
| Close session on a specific database | `POST /{instance}/api/v1/Databases('{db}')/ActiveSession/tm1.Close` |
| Close session on a specific database, by ID | `POST /{instance}/api/v1/Databases('{db}')/Sessions({id})/tm1.Close` |
| Close session at service level (all databases) | `POST /{instance}/api/v1/ActiveSession/tm1s.Close` |
| Close session at service level, by ID | `POST /{instance}/api/v1/Sessions({id})/tm1s.Close` |

```bash
curl -X POST "https://<host>/test/api/v1/ActiveSession/tm1s.Close" \
  -H "Cookie: TM1SessionId=$SESSION_ID"
```

Closing a session on a specific database does **not** close it at the service level — the session remains valid for other databases. Returns `204 No Content` on success.
