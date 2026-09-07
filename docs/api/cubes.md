# Cubes API

Endpoints for creating, listing, inspecting, and deleting cubes within a database. See [Cubes](../concepts/cubes.md) for a conceptual overview.

!!! success "Verified 2026-09-07 against TM1 12.6.4"
    Create, List, Get, Update, and Delete were all confirmed live against a local test instance (database `test`), including the error cases below.

!!! note
    Every request below requires a valid `Authorization` header. See [Authentication](../getting-started/authentication.md).

## The cube object

`List cubes`, `Get a cube`, and `Create a cube` return objects in this shape:

```json
{
  "Name": "SalesPlanning",
  "Rules": null,
  "DrillthroughRules": null,
  "LastSchemaUpdate": "2026-09-07T06:54:21.862Z",
  "LastDataUpdate": "2026-09-07T06:54:21.862Z",
  "ViewStorageMaxMemory": null,
  "ViewStorageMinTime": null,
  "CalculationThresholdForStorage": null,
  "CellSecurityDefaultValue": null,
  "CellSecurityMostRestrictive": false,
  "AllowPersistentHolds": false,
  "Locked": false,
  "Attributes": { "Caption": "SalesPlanning" }
}
```

`Rules` is the cube's rule script (`null` if it has none — see [Rules](../concepts/cubes.md#rules)). The dimensions assigned to a cube aren't included by default; fetch them with `?$expand=Dimensions` (or `.../Cubes('{cube}')/Dimensions` directly).

## Create a cube

```http
POST /{instance}/api/v1/Databases('{database}')/Cubes
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Unique name for the cube within the database. |
| `Dimensions@odata.bind` | array of string | Yes | The cube's dimensions, in order, as OData bind syntax, e.g. `["Dimensions('Region')", "Dimensions('Measure')"]`. Every dimension must already exist. |

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases('test')/Cubes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"Name\": \"SalesPlanning\", \"Dimensions@odata.bind\": [\"Dimensions('Region')\", \"Dimensions('Measure')\"]}"
```

### Response

`201 Created`, with a `Location` header (e.g. `Cubes('SalesPlanning')`) and the new [cube object](#the-cube-object) in the body.

### Errors

All errors share the same envelope, `{"error": {"code": "...", "message": "..."}}`:

| Status | `message` | Meaning |
|---|---|---|
| `400 Bad Request` | `A cube with Name "{name}" already exists.` | A cube with that name already exists — `400`, not `409`. |
| `401 Unauthorized` | `Authentication required` | The credentials are missing, expired, or invalid. |
| `404 Not Found` | — | The database, or one of the dimensions in `Dimensions@odata.bind`, doesn't exist. |

## List cubes

```http
GET /{instance}/api/v1/Databases('{database}')/Cubes
```

```bash
curl "https://<host>/{instance}/api/v1/Databases('test')/Cubes" \
  -H "Authorization: Bearer $TOKEN"
```

Returns [cube objects](#the-cube-object) — including TM1's own control cubes (names starting with `}`) alongside the ones you created.

## Get a cube

```http
GET /{instance}/api/v1/Databases('{database}')/Cubes('{name}')
```

```bash
curl "https://<host>/{instance}/api/v1/Databases('test')/Cubes('SalesPlanning')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `404 Not Found` if no cube with that name exists in the database.

## Delete a cube

```http
DELETE /{instance}/api/v1/Databases('{database}')/Cubes('{name}')
```

```bash
curl -X DELETE "https://<host>/{instance}/api/v1/Databases('test')/Cubes('SalesPlanning')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success. Its [views](views.md) are deleted along with it; its dimensions are not.

## Update a cube

```http
PATCH /{instance}/api/v1/Databases('{database}')/Cubes('{name}')
```

Set any mutable field from [the cube object](#the-cube-object) — most commonly `Rules`, to attach or change a rule script:

```bash
curl -X PATCH "https://<host>/{instance}/api/v1/Databases('test')/Cubes('SalesPlanning')" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"Rules\": \"['Total']=N:1;\"}"
```

### Response

`200 OK`, with the updated [cube object](#the-cube-object) in the body (`Rules` now reflecting what you sent). Confirmed live — used to attach a real rule (later exercised to produce `RuleDerived`/`Error` cells; see [Cellsets API](cellsets.md#get-a-cellset)) and to clear it again by sending `""`.

!!! note
    Check a rule's syntax without saving it via `POST .../Cubes('{name}')/tm1.CheckRules` — returns `201 Created` with `{"value": []}` when the rule compiles cleanly, confirmed live. It only checks syntax, not runtime correctness: a rule that compiles fine can still produce `Status: "Error"` cells at evaluation time (e.g. a division by zero), also confirmed live.
