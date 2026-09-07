# Databases API

Endpoints for creating, listing, inspecting, and deleting TM1 databases. See [Databases](../concepts/databases.md) for a conceptual overview.

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

Returns a single object in the same shape as one entry of `value` above (no `@odata.context`/`value` envelope). Returns `404 Not Found` if no database with that name exists.

## Create a database

Creates a new, empty database and starts provisioning it.

```http
POST /{instance}/api/v1/Databases
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Unique name for the database. Letters, numbers, and underscores only. |

=== "curl"

    ```bash
    curl -X POST "https://<host>/test/api/v1/Databases" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d '{
            "Name": "SalesPlanning"
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
        json={"Name": "SalesPlanning"},
    )
    response.raise_for_status()
    database = response.json()
    ```

### Response

`201 Created` on success, with the new database in the response body (same shape as [the database object](#the-database-object)) and a `Location` header pointing at it.

!!! warning "Unverified"
    The exact contents of a freshly created, still-provisioning database have not been confirmed against a live response. Expect `ActiveReplicas` to be empty or partially populated until provisioning finishes — poll [Get a database](#get-a-database) and watch each entry's `State` reach `ready` before sending the database any other requests, rather than relying on a single top-level status field.

### Errors

| Status | Meaning |
|---|---|
| `400 Bad Request` | `Name` is missing, malformed, or uses reserved characters. |
| `401 Unauthorized` | The credentials are missing, expired, or invalid. |
| `409 Conflict` | A database with that name already exists. |

## Delete a database

```http
DELETE /{instance}/api/v1/Databases('{name}')
```

```bash
curl -X DELETE "https://<host>/test/api/v1/Databases('SalesPlanning')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success. This permanently removes the database and all of its cubes, dimensions, and data — the operation cannot be undone.
