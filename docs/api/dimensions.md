# Dimensions API

Endpoints for creating, listing, inspecting, and deleting dimensions within a database. See [Dimensions](../concepts/dimensions.md) for a conceptual overview.

!!! success "Verified 2026-09-07 against TM1 12.6.4"
    Create, List, Get, Delete, listing Elements, and adding an Edge were all confirmed live against a local test instance (database `test`), including the error cases below, except where a note says otherwise.

!!! note
    Every request below requires a valid `Authorization` header. See [Authentication](../getting-started/authentication.md).

## The dimension object

`List dimensions` and `Get a dimension` return metadata only — **not** the dimension's hierarchies or elements:

```json
{
  "Name": "Region",
  "UniqueName": "[Region]",
  "AllLeavesHierarchyName": "Leaves",
  "Locked": false,
  "Attributes": { "Caption": "Region" }
}
```

To get a dimension's elements, query its default hierarchy's `Elements` collection — see [List elements](#list-elements) below.

## Create a dimension

Creates a dimension, its default hierarchy, and any leaf elements you supply, in one call.

```http
POST /{instance}/api/v1/Databases('{database}')/Dimensions
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Unique name for the dimension within the database. |
| `Hierarchies` | array | Yes | At least one hierarchy. The first is treated as the default hierarchy. |
| `Hierarchies[].Name` | string | Yes | Hierarchy name. Use the same value as `Name` for the default hierarchy. |
| `Hierarchies[].Elements` | array | No | Leaf and consolidated elements to create up front. |
| `Hierarchies[].Elements[].Name` | string | Yes | Element name. |
| `Hierarchies[].Elements[].Type` | string | Yes | `Numeric`, `String`, or `Consolidated`. |

=== "curl"

    ```bash
    curl -X POST "https://<host>/{instance}/api/v1/Databases('SalesPlanning')/Dimensions" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d '{
            "Name": "Region",
            "Hierarchies": [
              {
                "Name": "Region",
                "Elements": [
                  { "Name": "North", "Type": "Numeric" },
                  { "Name": "South", "Type": "Numeric" },
                  { "Name": "Total Region", "Type": "Consolidated" }
                ]
              }
            ]
          }'
    ```

=== "Python"

    ```python
    import requests

    response = requests.post(
        "https://<host>/{instance}/api/v1/Databases('SalesPlanning')/Dimensions",
        headers={
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json",
        },
        json={
            "Name": "Region",
            "Hierarchies": [
                {
                    "Name": "Region",
                    "Elements": [
                        {"Name": "North", "Type": "Numeric"},
                        {"Name": "South", "Type": "Numeric"},
                        {"Name": "Total Region", "Type": "Consolidated"},
                    ],
                }
            ],
        },
    )
    response.raise_for_status()
    dimension = response.json()
    ```

Creating a `Consolidated` element this way only creates the element itself — it has no children yet. Add its children as [edges](#adding-a-consolidated-element) afterward.

### Response

`201 Created`, with a `Location` header (e.g. `Dimensions('Region')`) and the new [dimension object](#the-dimension-object) in the body — metadata only, **not** an echo of the `Hierarchies`/`Elements` you posted:

```json
{
  "Name": "Region",
  "UniqueName": "[Region]",
  "AllLeavesHierarchyName": "Leaves",
  "Locked": false,
  "Attributes": { "Caption": "Region" }
}
```

### Errors

All errors share the same envelope, `{"error": {"code": "...", "message": "..."}}`:

| Status | `message` | Meaning |
|---|---|---|
| `400 Bad Request` | `Empty dimension definition. Failed to create dimension.` | `Name` (or the whole body) is missing. |
| `400 Bad Request` | `A dimension named "{name}" already exists.` | A dimension with that name already exists — `400`, not `409`. |
| `400 Bad Request` | `Dimension name is a reserved system Dimension name.` | The name collides with a control dimension (e.g. one starting with `}`). |
| `401 Unauthorized` | `Authentication required` | The credentials are missing, expired, or invalid. |
| `404 Not Found` | — | The database in the URL doesn't exist. |

## List elements

```http
GET /{instance}/api/v1/Databases('{database}')/Dimensions('{dimension}')/Hierarchies('{hierarchy}')/Elements
```

```bash
curl "https://<host>/{instance}/api/v1/Databases('SalesPlanning')/Dimensions('Region')/Hierarchies('Region')/Elements" \
  -H "Authorization: Bearer $TOKEN"
```

### Response

```json
{
  "value": [
    { "Name": "North", "UniqueName": "[Region].[Region].[North]", "Type": "Numeric", "Level": 0, "Index": 1, "Locked": false, "Attributes": { "Caption": "North" } },
    { "Name": "South", "UniqueName": "[Region].[Region].[South]", "Type": "Numeric", "Level": 0, "Index": 2, "Locked": false, "Attributes": { "Caption": "South" } },
    { "Name": "Total Region", "UniqueName": "[Region].[Region].[Total Region]", "Type": "Consolidated", "Level": 1, "Index": 3, "Locked": false, "Attributes": { "Caption": "Total Region" } }
  ]
}
```

`Level` is `0` for leaves and increases going up a consolidation; `Index` is the element's position within the hierarchy.

## Adding a consolidated element

Consolidations are added as edges between a parent and child element in a hierarchy, after both elements exist:

```http
POST /{instance}/api/v1/Databases('{database}')/Dimensions('{dimension}')/Hierarchies('{hierarchy}')/Edges
```

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases('SalesPlanning')/Dimensions('Region')/Hierarchies('Region')/Edges" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "ParentName": "Total Region",
        "ComponentName": "North",
        "Weight": 1
      }'
```

`Total Region` must already exist as an element with `Type: "Consolidated"` before you can add children to it.

### Response

`201 Created`, with a `Location` header and the edge echoed back in the body:

```json
{ "ParentName": "Total Region", "ComponentName": "North", "Weight": 1 }
```

## List dimensions

```http
GET /{instance}/api/v1/Databases('{database}')/Dimensions
```

```bash
curl "https://<host>/{instance}/api/v1/Databases('SalesPlanning')/Dimensions" \
  -H "Authorization: Bearer $TOKEN"
```

Returns dimension [metadata objects](#the-dimension-object) — including TM1's own control dimensions (names starting with `}`, e.g. `}Clients`, `}Groups`) alongside the ones you created.

## Get a dimension

```http
GET /{instance}/api/v1/Databases('{database}')/Dimensions('{name}')
```

```bash
curl "https://<host>/{instance}/api/v1/Databases('SalesPlanning')/Dimensions('Region')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `404 Not Found` if no dimension with that name exists in the database.

## Delete a dimension

```http
DELETE /{instance}/api/v1/Databases('{database}')/Dimensions('{name}')
```

```bash
curl -X DELETE "https://<host>/{instance}/api/v1/Databases('SalesPlanning')/Dimensions('Region')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success.

!!! warning
    A dimension can't be deleted while it's still in use by a cube:
    ```json
    { "error": { "code": "31", "message": "DimensionIsBeingUsedByCube" } }
    ```
    (`400 Bad Request`.) Remove it from any cubes first — see [Delete a cube](cubes.md#delete-a-cube).
