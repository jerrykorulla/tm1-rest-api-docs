# Authentication

Every request to the TM1 REST API must be authenticated — either with a bearer token presented on every request, or with a `TM1SessionId` session cookie obtained once and reused. See [Sessions](../api/sessions.md) for full endpoint detail; this page covers the concepts and the common cases.

!!! note "Native HTTP Basic"
    Many deployments — including the examples elsewhere in this documentation — also accept plain HTTP Basic credentials (a TM1 username and password) directly on any request:
    ```
    Authorization: Basic <base64 of user:password>
    ```
    TM1 validates the credentials and creates a session lazily on first use, with no separate token-exchange call required. This is the simplest option for local testing and is what [First Request](first-request.md) uses.

## How a session is created

There are three ways to obtain a session, depending on your authentication setup.

### 1. OAuth 2.0 bearer token (recommended)

If an external Authorization Server (Okta, Entra ID, Keycloak, etc.) is configured for your instance, obtain a token from it however it issues one, then present it on any request. TM1 validates the token, maps its `sub` claim to a TM1 user, and creates the session automatically — no explicit session-creation call needed.

```http
GET /{instance}/api/v1/Databases('PlanSamp')/Dimensions
Authorization: Bearer eyJhbGci...
```

This is the pattern used by `Authorization: Bearer $TOKEN` throughout the rest of this documentation.

### 2. Service-to-service (S2S)

A trusted application (registered as an [Application](../concepts/architecture.md#applications-and-the-s2s-flow) via the Management API) exchanges its own credentials for an access token, then uses that token to create a session on behalf of a specific user.

**Step 1 — get an access token:**

```http
POST /{instance}/auth/v1/token
Content-Type: application/json

{
    "grant_type": "client_credentials",
    "client_id": "{App.ClientID}",
    "client_secret": "{App.ClientSecret}"
}
```

**Step 2 — create a session for a user:**

```http
POST /{instance}/api/v1/Sessions
Authorization: Bearer {s2s_access_token}
Content-Type: application/json

{
    "UserName": "user@email.com"
}
```

The response sets a `TM1SessionId` cookie — see [Using a session](#using-a-session) below. Full request/response detail is in the [Sessions API](../api/sessions.md#create-a-session), including optional Session Variables.

### 3. HTTP pass-through

TM1 delegates authentication to a configured external HTTP service (for example, a TM1 v11 server). The `Authorization` header is forwarded to that service; if it responds `2xx`, TM1 creates a session for the returned user automatically. No explicit session-creation call is needed from the client.

## Using a session

Once you have a session — a bearer token (Method 1), or a `TM1SessionId` cookie (Method 2 or 3) — attach it to every subsequent request:

=== "Bearer token"

    ```bash
    curl "https://<host>/{instance}/api/v1/Databases" \
      -H "Authorization: Bearer $TOKEN"
    ```

=== "Session cookie"

    ```bash
    curl "https://<host>/{instance}/api/v1/Databases" \
      -H "Cookie: TM1SessionId=$SESSION_ID"
    ```

The examples throughout this documentation show `Authorization: Bearer $TOKEN`; if your integration uses the S2S flow instead, substitute `Cookie: TM1SessionId=$SESSION_ID` wherever you see that header.

!!! note
    A service-level session doesn't automatically mean you're logged into a specific database — the database-level session is created lazily on your first request to that database.

## Closing a session

| Scope | Endpoint |
|---|---|
| Close session on a specific database | `POST /{instance}/api/v1/Databases('{db}')/ActiveSession/tm1.Close` |
| Close session on a specific database, by ID | `POST /{instance}/api/v1/Databases('{db}')/Sessions({id})/tm1.Close` |
| Close session at service level (all databases) | `POST /{instance}/api/v1/ActiveSession/tm1s.Close` |
| Close session at service level, by ID | `POST /{instance}/api/v1/Sessions({id})/tm1s.Close` |

Closing a session on a specific database does **not** close it at the service level — the session remains valid for other databases.

## Important notes

- `POST /{instance}/auth/v1/session` exists but is **deprecated** — use the S2S flow above (`POST /{instance}/auth/v1/token` then `POST /{instance}/api/v1/Sessions`) instead.
- Session variable keys and values combined cannot exceed **8192 bytes**.
- Session variables are stored encrypted and are cleared when the session expires.

!!! warning
    Never hard-code credentials, tokens, or client secrets in source control. Load them from environment variables or a secrets manager, as shown in the examples above.
