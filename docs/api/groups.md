# Groups API

Endpoints for listing and managing groups within a database. See [Groups](../concepts/groups.md) for a conceptual overview — in particular, groups (like users) are always scoped to a specific database.

!!! success "Verified 2026-09-07 against TM1 12.6.4"
    List, Get, Create, and Delete (of a custom group) were all confirmed live against a local test instance (database `test`), including the duplicate-name error. Deleting a built-in group was **not** attempted, to avoid disabling admin access on the test instance.

!!! note
    Every request below requires a valid `Authorization` header for a user session — either a bearer token or HTTP Basic credentials both work. See [Authentication](../getting-started/authentication.md).

## List groups

```http
GET /{instance}/api/v1/Databases('{db}')/Groups
```

```bash
curl "https://<host>/test/api/v1/Databases('test')/Groups" \
  -H "Authorization: Bearer $TOKEN"
```

### Response

```json
{
  "@odata.context": "$metadata#Groups",
  "value": [
    { "Name": "ADMIN" },
    { "Name": "DataAdmin" },
    { "Name": "OperationsAdmin" },
    { "Name": "SecurityAdmin" }
  ]
}
```

Every database has these four built-in groups at minimum — see [Groups — Built-in groups](../concepts/groups.md#built-in-groups). Custom groups, if any have been created, appear alongside them in the same list.

## Get a group

```http
GET /{instance}/api/v1/Databases('{db}')/Groups('{name}')
```

```bash
curl "https://<host>/test/api/v1/Databases('test')/Groups('ADMIN')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns a single object in the same shape as one entry of `value` above (no `value` envelope). Returns `404 Not Found` if no group with that name exists in the database.

## Create a group

Creates a custom group. The built-in groups (`ADMIN`, `DataAdmin`, `OperationsAdmin`, `SecurityAdmin`) already exist on every database and don't need to be created.

```http
POST /{instance}/api/v1/Databases('{db}')/Groups
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Unique name for the group within the database. |

```bash
curl -X POST "https://<host>/test/api/v1/Databases('test')/Groups" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"Name\": \"Planners\"}"
```

### Response

`201 Created`, with a `Location` header (e.g. `Groups('Planners')`) and `{"Name": "Planners"}` in the body.

## Delete a group

```http
DELETE /{instance}/api/v1/Databases('{db}')/Groups('{name}')
```

```bash
curl -X DELETE "https://<host>/test/api/v1/Databases('test')/Groups('Planners')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success.

!!! warning "Unverified"
    Deleting one of the four built-in groups (`ADMIN`, `DataAdmin`, `OperationsAdmin`, `SecurityAdmin`) was not tested against the live environment used to verify this page, to avoid disabling admin access on it. Expect it to be rejected, but the exact status code is unconfirmed.

## Errors

All errors share the same envelope:

```json
{ "error": { "code": "278", "message": "..." } }
```

| Status | `message` | Meaning |
|---|---|---|
| `400 Bad Request` | `A group with Name "{name}" already exists.` | A group with that name already exists — note this is `400`, not `409`. |
| `401 Unauthorized` | `Authentication required` | The credentials are missing, expired, or invalid. |
| `404 Not Found` | `'{name}' can not be found in collection of type 'Group'.` | No group with that name exists in the database. |

!!! note
    `403 Forbidden` for a non-admin caller is expected but hasn't been observed directly — the test environment used to verify this page authenticates as the built-in `Admin` user.
