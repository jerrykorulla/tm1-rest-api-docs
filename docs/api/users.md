# Users API

Endpoints for creating, listing, updating, and deleting users within a database. See [Users](../concepts/users.md) for a conceptual overview — in particular, users always live inside a specific database; there is no instance- or service-level user endpoint. See the [Groups API](groups.md) for managing the groups referenced below.

!!! success "Verified 2026-09-07 against TM1 12.6.4"
    List, Get (including `$expand=Groups`), Create, Update, and Delete were all confirmed live against a local test instance (database `test`), including the error cases below. Only `403 Forbidden` wasn't observed directly (see the note under [Errors](#errors)).

!!! note
    Every request below requires a valid `Authorization` header for a user session — either a bearer token or HTTP Basic credentials both work. See [Authentication](../getting-started/authentication.md).

## The user object

`List users`, `Get a user`, and `Create a user` return objects in this shape:

| Field | Type | Description |
|---|---|---|
| `Name` | string | Username. |
| `FriendlyName` | string | Display name. Defaults to `Name` if not set otherwise. |
| `Type` | string | Reflects the user's highest-ranked built-in group: `User` if they belong to none of the built-in groups, otherwise the name of the built-in group (`Admin`, `DataAdmin`, `OperationsAdmin`, `SecurityAdmin`) they're in. Membership in a custom group doesn't change `Type`. |
| `IsActive` | boolean | Whether the user is currently logged in. `false` for a user who has never logged in or has no active session. |
| `Groups` | array | The user's groups, as `{"Name": "..."}` objects. Only present when requested with `$expand=Groups` — not returned by default. |

```json
{
  "Name": "jane.smith",
  "FriendlyName": "jane.smith",
  "Type": "User",
  "IsActive": false
}
```

The built-in `Admin` user itself appears with `"Type": "Admin"` — TM1's own admin account is a user like any other, distinguished only by its group membership, not by a special flag.

## List users

```http
GET /{instance}/api/v1/Databases('{db}')/Users
```

```bash
curl "https://<host>/test/api/v1/Databases('test')/Users" \
  -H "Authorization: Bearer $TOKEN"
```

### Response

```json
{
  "@odata.context": "$metadata#Users",
  "value": [
    { "Name": "Admin", "FriendlyName": "Admin", "Type": "Admin", "IsActive": true },
    { "Name": "jane.smith", "FriendlyName": "jane.smith", "Type": "User", "IsActive": false }
  ]
}
```

## Get a user

```http
GET /{instance}/api/v1/Databases('{db}')/Users('{name}')
```

```bash
curl "https://<host>/test/api/v1/Databases('test')/Users('Admin')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns a single [user object](#the-user-object) (no `value` envelope). Add `?$expand=Groups` to include the user's group memberships:

```bash
curl "https://<host>/test/api/v1/Databases('test')/Users('Admin')?\$expand=Groups" \
  -H "Authorization: Bearer $TOKEN"
```

```json
{
  "Name": "Admin",
  "FriendlyName": "Admin",
  "Type": "Admin",
  "IsActive": true,
  "Groups": [{ "Name": "ADMIN" }]
}
```

Returns `404 Not Found` if no user with that name exists — see [Errors](#errors).

## Create a user

```http
POST /{instance}/api/v1/Databases('{db}')/Users
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Username. |
| `Groups@odata.bind` | array of string | No | OData bind syntax assigning the new user to one or more existing groups at creation time, e.g. `["Groups('Planners')"]`. The group must already exist — see [Groups](groups.md#built-in-groups) for the built-in groups every database has (there is no default "everyone" group), or [Create a group](groups.md#create-a-group) first for a custom one. |

```bash
curl -X POST "https://<host>/test/api/v1/Databases('test')/Users" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"Name\": \"jane.smith\"}"
```

### Response

`201 Created`, with a `Location` header (e.g. `Users('jane.smith')`) and the new [user object](#the-user-object) in the body. A user created without a group has `"Type": "User"` and `"IsActive": false` until they first log in.

## Update a user

Use to change a user's group membership (for example, adding them to `ADMIN`) or other mutable properties.

```http
PATCH /{instance}/api/v1/Databases('{db}')/Users('{name}')
```

```bash
curl -X PATCH "https://<host>/test/api/v1/Databases('test')/Users('jane.smith')" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"Groups@odata.bind\": [\"Groups('DataAdmin')\"]}"
```

### Response

`200 OK` with the updated [user object](#the-user-object) — assigning a built-in group updates `Type` to match (e.g. `"Type": "DataAdmin"` above).

## Delete a user

```http
DELETE /{instance}/api/v1/Databases('{db}')/Users('{name}')
```

```bash
curl -X DELETE "https://<host>/test/api/v1/Databases('test')/Users('jane.smith')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success.

## Errors

All errors share the same envelope:

```json
{ "error": { "code": "278", "message": "..." } }
```

| Status | `message` | Meaning |
|---|---|---|
| `400 Bad Request` | `Empty user definition. Failed to create user.` | `Name` is missing from the request body. |
| `400 Bad Request` | `A user with Name "{name}" already exists.` | A user with that name already exists — note this is `400`, not `409`. |
| `401 Unauthorized` | `Authentication required` | The credentials are missing, expired, or invalid. |
| `404 Not Found` | `'{name}' can not be found in collection of type 'User'.` | No user with that name exists in the database. |

!!! note
    `403 Forbidden` for a non-admin caller is expected but hasn't been observed directly — the test environment used to verify this page authenticates as the built-in `Admin` user.
