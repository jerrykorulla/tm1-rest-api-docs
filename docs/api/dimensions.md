# Dimensions API

Endpoints for creating, listing, inspecting, and deleting dimensions within a database. See [Dimensions](../concepts/dimensions.md) for a conceptual overview.

!!! note
    Every request below requires a valid `Authorization` header. See [Authentication](../getting-started/authentication.md).

## Create a dimension

Creates a dimension, its default hierarchy, and any leaf elements you supply, in one call.

```http
POST /api/v1/Databases('{database}')/Dimensions
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Unique name for the dimension within the database. |
| `Hierarchies` | array | Yes | At least one hierarchy. The first is treated as the default hierarchy. |
| `Hierarchies[].Name` | string | Yes | Hierarchy name. Use the same value as `Name` for the default hierarchy. |
| `Hierarchies[].Elements` | array | No | Leaf elements to create up front. |
| `Hierarchies[].Elements[].Name` | string | Yes | Element name. |
| `Hierarchies[].Elements[].Type` | string | Yes | `Numeric`, `String`, or `Consolidated`. |

=== "curl"

    ```bash
    curl -X POST "https://<host>/api/v1/Databases('SalesPlanning')/Dimensions" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d '{
            "Name": "Region",
            "Hierarchies": [
              {
                "Name": "Region",
                "Elements": [
                  { "Name": "North", "Type": "Numeric" },
                  { "Name": "South", "Type": "Numeric" }
                ]
              }
            ]
          }'
    ```

=== "Python"

    ```python
    import requests

    response = requests.post(
        "https://<host>/api/v1/Databases('SalesPlanning')/Dimensions",
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
                    ],
                }
            ],
        },
    )
    response.raise_for_status()
    dimension = response.json()
    ```

### Response

`201 Created` on success, with the new dimension in the response body.

```json
{
  "Name": "Region",
  "Hierarchies": [
    {
      "Name": "Region",
      "Elements": [
        { "Name": "North", "Type": "Numeric" },
        { "Name": "South", "Type": "Numeric" }
      ]
    }
  ]
}
```

### Errors

| Status | Meaning |
|---|---|
| `400 Bad Request` | `Name` is missing, or an element has an invalid `Type`. |
| `401 Unauthorized` | The bearer token is missing, expired, or invalid. |
| `404 Not Found` | The database in the URL doesn't exist. |
| `409 Conflict` | A dimension with that name already exists. |

## Adding a consolidated element

Consolidations are added as edges between a parent and child element in a hierarchy, after both elements exist:

```http
POST /api/v1/Databases('{database}')/Dimensions('{dimension}')/Hierarchies('{hierarchy}')/Edges
```

```bash
curl -X POST "https://<host>/api/v1/Databases('SalesPlanning')/Dimensions('Region')/Hierarchies('Region')/Edges" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "ParentName": "Total Region",
        "ComponentName": "North",
        "Weight": 1
      }'
```

`Total Region` must already exist as an element with `Type: "Consolidated"` before you can add children to it.

## List dimensions

```http
GET /api/v1/Databases('{database}')/Dimensions
```

```bash
curl "https://<host>/api/v1/Databases('SalesPlanning')/Dimensions" \
  -H "Authorization: Bearer $TOKEN"
```

Returns an array of dimension objects in the same shape as [Create a dimension](#create-a-dimension).

## Get a dimension

```http
GET /api/v1/Databases('{database}')/Dimensions('{name}')
```

```bash
curl "https://<host>/api/v1/Databases('SalesPlanning')/Dimensions('Region')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `404 Not Found` if no dimension with that name exists in the database.

## Delete a dimension

```http
DELETE /api/v1/Databases('{database}')/Dimensions('{name}')
```

```bash
curl -X DELETE "https://<host>/api/v1/Databases('SalesPlanning')/Dimensions('Region')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success.

!!! warning
    A dimension can't be deleted while it's still in use by a cube. Remove it from any cubes first.
