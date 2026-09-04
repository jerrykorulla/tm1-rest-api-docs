# Databases API

Endpoints for creating, listing, inspecting, and deleting TM1 databases. See [Databases](../concepts/databases.md) for a conceptual overview.

!!! note
    Every request below requires a valid `Authorization` header. See [Authentication](../getting-started/authentication.md).

## Create a database

Creates a new, empty database and starts provisioning it.

```http
POST /api/v1/Databases
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Unique name for the database. Letters, numbers, and underscores only. |
| `Type` | string | No | Database type, e.g. `Standard`. Defaults to `Standard` if omitted. |

=== "curl"

    ```bash
    curl -X POST "https://<host>/api/v1/Databases" \
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
        "https://<host>/api/v1/Databases",
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

`201 Created` on success, with the new database in the response body and a `Location` header pointing at it.

```json
{
  "Name": "SalesPlanning",
  "Type": "Standard",
  "State": "Provisioning"
}
```

Provisioning happens asynchronously — poll [Get a database](#get-a-database) until `State` is `Running` before sending it any other requests.

### Errors

| Status | Meaning |
|---|---|
| `400 Bad Request` | `Name` is missing, malformed, or uses reserved characters. |
| `401 Unauthorized` | The bearer token is missing, expired, or invalid. |
| `409 Conflict` | A database with that name already exists. |

## List databases

```http
GET /api/v1/Databases
```

```bash
curl "https://<host>/api/v1/Databases" \
  -H "Authorization: Bearer $TOKEN"
```

Returns an array of database objects in the same shape as [Create a database](#create-a-database).

## Get a database

```http
GET /api/v1/Databases('{name}')
```

```bash
curl "https://<host>/api/v1/Databases('SalesPlanning')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `404 Not Found` if no database with that name exists.

## Delete a database

```http
DELETE /api/v1/Databases('{name}')
```

```bash
curl -X DELETE "https://<host>/api/v1/Databases('SalesPlanning')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success. This permanently removes the database and all of its cubes, dimensions, and data — the operation cannot be undone.
