# Databases API

Endpoints for creating, listing, inspecting, and deleting TM1 databases. See [Databases](../concepts/databases.md) for a conceptual overview, and [Architecture](../concepts/architecture.md) for how instances, databases, and users relate.

!!! success "Verified 2026-09-07 against TM1 12.6.4"
    List, Get, Create, and Delete were all confirmed live against a local test instance (database `test`), including the error cases below.

!!! note
    Every request below requires a valid `Authorization` header — either a bearer token or HTTP Basic credentials both work against these endpoints. See [Authentication](../getting-started/authentication.md).

!!! note "Path is scoped to a TM1 instance"
    The API is called over a specific TM1 instance, so the path is `/{instance}/api/v1/Databases`, not a global, instance-independent route. An instance's `/Databases` collection describes itself (as a replica cluster) rather than every database in the environment — see [List databases](#list-databases) below. The examples on this page use `test` as the instance name.

## The database object

`List databases` and `Get a database` both return objects in this shape:

| Field | Type | Description |
|---|---|---|
| `ID` | string | Internal identifier for the underlying deployment/cluster. |
| `Name` | string | The database name, as used in the URL path. |
| `ProductVersion.SemVer` | string | TM1 version running on the database. |
| `ServiceRootURL` | string | Relative URL of this database's own REST API root. |
| `Replicas` | number | Number of replicas configured for the database. |
| `ActiveReplicas` | array | One entry per running replica — see below. |
| `Resources.Replica.CPU.Requests` / `.Limits` | string | CPU request/limit per replica. |
| `Resources.Replica.Memory.Requests` / `.Limits` | string | Memory request/limit per replica. |
| `Resources.Storage.Size` | string | Storage allocated to the database. |
| `CollectDiagnostics` | boolean | Whether diagnostic collection is enabled for the database. |

Each entry in `ActiveReplicas`:

| Field | Type | Description |
|---|---|---|
| `ID` | string | Replica identifier, e.g. `"0"`. |
| `Fingerprint` | string | Fingerprint of the replica's current state. |
| `State` | string | Replica health, e.g. `ready`. There is no separate top-level database `State` — health is reported per replica. |
| `Commit` | string | Commit/version marker for the replica's data. |
| `Role` | string | `Leader` or `Follower`. |
| `ProductVersion.SemVer` | string | TM1 version running on this replica. |

## List databases

Returns the database(s) visible through this TM1 instance, wrapped in an OData envelope.

```http
GET /{instance}/api/v1/Databases
```

```bash
curl "https://<host>/test/api/v1/Databases" \
  -H "Authorization: Bearer $TOKEN"
```

### Response

```json
{
  "@odata.context": "$metadata#Databases",
  "value": [
    {
      "ID": "tm1-i-d916m0nkt9h1ei299frg-d-daf4nknkt9h0kl37mf40",
      "Name": "test",
      "ProductVersion": {
        "SemVer": "12.6.4"
      },
      "ServiceRootURL": "./Databases('test')/",
      "Replicas": 1,
      "ActiveReplicas": [
        {
          "ID": "0",
          "Fingerprint": "6cb72619281126057b61f71f62b5ff84",
          "State": "ready",
          "Commit": "2026090705300500002",
          "Role": "Leader",
          "ProductVersion": {
            "SemVer": "12.6.4"
          }
        }
      ],
      "Resources": {
        "Replica": {
          "CPU": { "Requests": "", "Limits": "" },
          "Memory": { "Requests": "", "Limits": "" }
        },
        "Storage": { "Size": "" }
      },
      "CollectDiagnostics": true
    }
  ]
}
```

## Get a database

```http
GET /{instance}/api/v1/Databases('{name}')
```

```bash
curl "https://<host>/test/api/v1/Databases('test')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns a single object in the same shape as one entry of `value` above, minus the `value` array wrapper — `@odata.context` is still present, but reads `$metadata#Databases/$entity` instead. `ServiceRootURL` is also relative to the entity itself here (`"./"`) rather than to the collection (`"./Databases('test')/"` in [List databases](#list-databases)).

Returns `404 Not Found` if no database with that name exists:

```json
{ "error": { "code": "278", "message": "'{name}' can not be found in collection of type 'Database'." } }
```

## Create a database

Creates a new, empty database and starts provisioning it.

This is the Service API path, callable with any authenticated user session — the caller automatically becomes a member of the new database's `ADMIN` group (see [Users](../concepts/users.md#who-becomes-admin)). Root credentials can create a database instead through the [Management API](management.md#create-a-database), which has the same post-creation behavior but cannot access the database's data afterward.

```http
POST /{instance}/api/v1/Databases
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Unique name for the database. Letters, numbers, and underscores only. |
| `Replicas` | number | No | Number of replicas to provision for the database. |

=== "curl"

    ```bash
    curl -X POST "https://<host>/test/api/v1/Databases" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d '{
            "Name": "SalesPlanning",
            "Replicas": 3
          }'
    ```

=== "Python"

    ```python
    import requests

    response = requests.post(
        "https://<host>/test/api/v1/Databases",
        headers={
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json",
        },
        json={"Name": "SalesPlanning", "Replicas": 3},
    )
    response.raise_for_status()
    database = response.json()
    ```

### Response

`201 Created`, with a `Location` header (e.g. `Databases('doctest')`) and the new database in the response body — the same shape as [the database object](#the-database-object):

```json
{
  "ID": "tm1-i-d916m0nkt9h1ei299frg-d-daf64snkt9h0kl37mf4g",
  "Name": "doctest",
  "ProductVersion": { "SemVer": "12.6.4" },
  "ServiceRootURL": "./Databases('doctest')/",
  "Replicas": 1,
  "ActiveReplicas": [
    { "ID": "0", "Fingerprint": "62fdd386ec9c277de9dc2f99c5f4a57f", "State": "ready", "Commit": "2026090707063700002", "Role": "Leader", "ProductVersion": { "SemVer": "12.6.4" } }
  ],
  "Resources": { "Replica": { "CPU": { "Requests": "", "Limits": "" }, "Memory": { "Requests": "", "Limits": "" } }, "Storage": { "Size": "" } },
  "CollectDiagnostics": true
}
```

Confirmed live: on this test instance, the database came back with `ActiveReplicas` already populated and `"State": "ready"` in the very same `201` response — no provisioning delay or polling was needed here. A larger or cloud-hosted deployment may still take time to bring replicas up; if a response ever comes back with `ActiveReplicas` empty or a replica not yet `ready`, poll [Get a database](#get-a-database) until every entry reports `"State": "ready"` before using it.

No other users can access the new database until the creator adds them — see the [Users API](users.md#create-a-user).

### Errors

All errors share the same envelope, `{"error": {"code": "...", "message": "..."}}`:

| Status | `message` | Meaning |
|---|---|---|
| `400 Bad Request` | `Required 'Name' field is missing.` | `Name` is missing from the request body. |
| `400 Bad Request` | `A database named {name} already exists.` | A database with that name already exists — `400`, not `409`. |
| `401 Unauthorized` | `Authentication required` | The credentials are missing, expired, or invalid. |

## Delete a database

```http
DELETE /{instance}/api/v1/Databases('{name}')
```

```bash
curl -X DELETE "https://<host>/test/api/v1/Databases('SalesPlanning')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success, confirmed live. This permanently removes the database and all of its cubes, dimensions, and data — the operation cannot be undone. Deleting an already-deleted (or never-existent) database returns `404 Not Found`, same error shape as [Get a database](#get-a-database):

```json
{ "error": { "code": "278", "message": "'{name}' can not be found in collection of type 'Database'." } }
```
