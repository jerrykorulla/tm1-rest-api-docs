# Management API

Infrastructure-level endpoints for managing TM1 instances, the databases within them, and the Applications registered against them. See [Architecture](../concepts/architecture.md) for how this plane relates to the per-instance Service API used by the rest of this reference.

!!! warning "Partially verified 2026-09-07 against TM1 12.6.4"
    Confirmed live against a local test instance that the `/manage/v1/` base path exists and enforces its own credential check (see the note below). Neither endpoint on this page (Create a database, Register an Application) was exercised — no root credentials were available in the test environment.

!!! warning "Root credentials required"
    Every request below is authenticated with root `client_id`/`client_secret` credentials, sent as HTTP Basic:
    ```
    Authorization: Basic {ROOT_CLIENT_ID}:{ROOT_CLIENT_SECRET}
    ```
    Root credentials manage infrastructure only — they cannot read or write data inside a database. Once a database exists, use a user session against the [Service API](databases.md) for that.

!!! note
    Confirmed live: the `/manage/v1/` base path exists and is authenticated separately from the Service API — a Service API user session (even the built-in `Admin` user's own credentials) is rejected against it with `401 Unauthorized` and a plain OAuth-style error body (`{"error": "invalid_client", "error_details": "Invalid credentials provided"}`), distinct from the `{"error": {"code": ..., "message": ...}}` shape used elsewhere in this documentation. The endpoints below themselves (Create a database, Register an Application) were not exercised, since no root credentials were available in the test environment.

## Create a database

Creates a database within an instance.

```http
POST /manage/v1/Instances('{instance}')/Databases
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Unique name for the database within the instance. |
| `Replicas` | number | No | Number of replicas to provision for the database. |

```bash
curl -X POST "https://<host>/manage/v1/Instances('test')/Databases" \
  -H "Authorization: Basic $ROOT_CLIENT_ID:$ROOT_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
        "Name": "db",
        "Replicas": 1
      }'
```

### Response

`201 Created` on success.

The credential that made the request is automatically added to the new database's Users list and assigned to its `ADMIN` group — see [Users](../concepts/users.md#who-becomes-admin). This is the same post-creation behavior as creating a database through the Service API; see [Databases API — Create a database](databases.md#create-a-database) for the alternative, user-session-based path and the full response shape.

## Register an Application

Registers a trusted service-to-service (S2S) client against an instance, so it can create sessions on behalf of users without each user authenticating directly. See [Applications and the S2S flow](../concepts/architecture.md#applications-and-the-s2s-flow) for the full sequence, including how the resulting credentials are then used against the Service API.

```http
POST /manage/v1/Instances('{instance}')/Applications
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Name for the application, e.g. `PAW`. |

```bash
curl -X POST "https://<host>/manage/v1/Instances('test')/Applications" \
  -H "Authorization: Basic $ROOT_CLIENT_ID:$ROOT_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{ "Name": "PAW" }'
```

### Response

`201 Created`, with the new application's `ClientID` and `ClientSecret` in the response body. Store `ClientSecret` securely — it is the credential the application exchanges for an access token (`POST /{instance}/auth/v1/token`) and then uses to create user sessions (`POST /{instance}/api/v1/Sessions`). See the [Sessions API](sessions.md) for both endpoints.

!!! warning "Unverified"
    The exact response body shape has not been confirmed against a live response; only the presence of `ClientID` and `ClientSecret` is documented here.
