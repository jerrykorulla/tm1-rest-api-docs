# Views API

Endpoints for creating, listing, executing, and deleting views on a cube. See [Views](../concepts/views.md) for the difference between an `MDXView` and a `NativeView`, and how running one relates to [Cellsets](cellsets.md).

!!! success "Verified 2026-09-07 against TM1 12.6.4"
    Create (both `MDXView` and `NativeView`), List, Execute, and Delete were all confirmed live against a local test instance (database `test`).

!!! note
    Every request below requires a valid `Authorization` header. See [Authentication](../getting-started/authentication.md).

## Create a view

```http
POST /{instance}/api/v1/Databases('{database}')/Cubes('{cube}')/Views
```

### Request body — MDXView

| Field | Type | Required | Description |
|---|---|---|---|
| `@odata.type` | string | Yes | `#tm1.MDXView`. |
| `Name` | string | Yes | Unique name for the view within the cube. |
| `MDX` | string | Yes | The MDX expression defining the view. |

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases('test')/Cubes('SalesPlanning')/Views" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"@odata.type\": \"#tm1.MDXView\", \"Name\": \"RegionByMeasure\", \"MDX\": \"SELECT {[Measure].[Revenue]} ON COLUMNS, {[Region].[North],[Region].[South]} ON ROWS FROM [SalesPlanning]\"}"
```

### Response

`201 Created`, with a `Location` header and the view echoed back:

```json
{
  "@odata.type": "#ibm.tm1.api.v1.MDXView",
  "Name": "RegionByMeasure",
  "Attributes": { "Caption": "RegionByMeasure" },
  "MDX": "SELECT {[Measure].[Revenue]} ON COLUMNS, {[Region].[North],[Region].[South]} ON ROWS FROM [SalesPlanning]"
}
```

The server echoes `@odata.type` back fully qualified (`#ibm.tm1.api.v1.MDXView`) even though `#tm1.MDXView` is what you send.

### Request body — NativeView

A `NativeView` pins each of the cube's dimensions to one axis (`Rows`, `Columns`, or `Titles`) — every dimension in the cube must be covered exactly once across the three. Each axis entry is a `ViewAxisSelection`:

| Field | Type | Required | Description |
|---|---|---|---|
| `Subset.Hierarchy.@odata.id` | string | Yes | The dimension/hierarchy this axis entry is for, as an OData id, e.g. `"Dimensions('Region')/Hierarchies('Region')"`. |
| `Subset.Expression` | string | Yes | An MDX set expression, evaluated against that hierarchy, defining which elements appear on this axis — e.g. `"{[Region].[Region].[Total Region].Children}"`. |
| `Selected@odata.bind` | string | Titles only | Which single element of the title's subset is "in effect" — an OData bind path, e.g. `"Dimensions('Region')/Hierarchies('Region')/Elements('North')"`. |

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases('test')/Cubes('Car Sales')/Views" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "@odata.type": "tm1.NativeView",
        "Name": "ModelsByTime",
        "Rows": [
          { "Subset": { "Hierarchy": { "@odata.id": "Dimensions('"'"'Models'"'"')/Hierarchies('"'"'Models'"'"')" }, "Expression": "{[Models].[Models].[All Body Types].Children}" } }
        ],
        "Columns": [
          { "Subset": { "Hierarchy": { "@odata.id": "Dimensions('"'"'Time'"'"')/Hierarchies('"'"'Time'"'"')" }, "Expression": "{[Time].[Time].[2025].Children}" } }
        ],
        "Titles": [
          {
            "Subset": { "Hierarchy": { "@odata.id": "Dimensions('"'"'Zones'"'"')/Hierarchies('"'"'Zones'"'"')" }, "Expression": "{[Zones].[Zones].[All Zones]}" },
            "Selected@odata.bind": "Dimensions('"'"'Zones'"'"')/Hierarchies('"'"'Zones'"'"')/Elements('"'"'North'"'"')"
          },
          {
            "Subset": { "Hierarchy": { "@odata.id": "Dimensions('"'"'Measures'"'"')/Hierarchies('"'"'Measures'"'"')" }, "Expression": "{[Measures].[Measures].[Units Sold]}" },
            "Selected@odata.bind": "Dimensions('"'"'Measures'"'"')/Hierarchies('"'"'Measures'"'"')/Elements('"'"'Units Sold'"'"')"
          }
        ]
      }'
```

Note `@odata.type` here is `tm1.NativeView` with no leading `#` — confirmed working as given; `MDXView`'s example above uses `#tm1.MDXView`, and both forms were accepted live.

### Response

`201 Created`, but the body does **not** echo `Rows`/`Columns`/`Titles` back with content — each comes back as an empty placeholder (e.g. `"Rows": [{}]`):

```json
{
  "@odata.type": "#ibm.tm1.api.v1.NativeView",
  "Name": "ModelsByTime",
  "Attributes": { "Caption": "ModelsByTime" },
  "Columns": [{}],
  "Rows": [{}],
  "Titles": [{}, {}],
  "SuppressEmptyColumns": false,
  "SuppressEmptyRows": false,
  "FormatString": ""
}
```

To see what was actually saved, `GET` the view back with `$expand` — note the syntax for these fields is `$expand=Rows/Subset,Columns/Subset,Titles/Subset,Titles/Selected` (a `/`, not `Rows($expand=Subset)`, since `Rows`/`Columns`/`Titles` are complex-type collections, not navigation properties):

```bash
curl "https://<host>/{instance}/api/v1/Databases('test')/Cubes('Car Sales')/Views('ModelsByTime')?%24expand=Rows/Subset,Columns/Subset,Titles/Subset,Titles/Selected" \
  -H "Authorization: Bearer $TOKEN"
```

```json
{
  "Columns": [{ "Subset": { "Expression": "{[Time].[Time].[2025].Children}", "Name": "", "UniqueName": "" } }],
  "Rows": [{ "Subset": { "Expression": "{[Models].[Models].[All Body Types].Children}", "Name": "", "UniqueName": "" } }],
  "Titles": [
    { "Subset": { "Expression": "{[Zones].[Zones].[All Zones]}" }, "Selected": { "Name": "All Zones", "Type": "Consolidated" } },
    { "Subset": { "Expression": "{[Measures].[Measures].[Units Sold]}" }, "Selected": { "Name": "Units Sold", "Type": "Numeric" } }
  ]
}
```

!!! warning "Selected@odata.bind didn't take effect here — confirmed"
    The request above asked for the Zones title to select `North`, but the saved view's `Selected` came back as `All Zones` instead — matching what the title's `Subset.Expression` itself resolves to (a single-element set, `{[Zones].[Zones].[All Zones]}`), not the element named in `Selected@odata.bind`. `North` isn't a member of that expression's result set. The Measures title, where `Selected@odata.bind` and the subset expression already agreed (`Units Sold`), can't distinguish whether the bind was honored or coincidental. Best guess: `Selected` must name an element that's actually within the title's `Subset.Expression` result — if you want a specific title selection, make sure the expression includes it (e.g. `{[Zones].[Zones].[All Zones].Children}` if you want to select `North`).

## List views

```http
GET /{instance}/api/v1/Databases('{database}')/Cubes('{cube}')/Views
```

```bash
curl "https://<host>/{instance}/api/v1/Databases('test')/Cubes('SalesPlanning')/Views" \
  -H "Authorization: Bearer $TOKEN"
```

Returns the cube's saved views (both kinds), each with its `@odata.type`.

## Execute a view

Evaluates the view against the cube's current data and returns a **[cellset](cellsets.md)**.

```http
POST /{instance}/api/v1/Databases('{database}')/Cubes('{cube}')/Views('{name}')/tm1.Execute
```

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases('test')/Cubes('SalesPlanning')/Views('RegionByMeasure')/tm1.Execute" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{}"
```

!!! warning
    Send a JSON body — even `{}` — with this and other bound `tm1.*` actions. A `POST` with no body at all fails with `400 Bad Request` / `{"error": {"code": "173", "message": "no content-length or transfer-encoding header provided"}}`, confirmed live.

### Response

`201 Created`, with a `Location` header pointing at the new cellset and `{"ID": "..."}` in the body — see [Cellsets — Execute a view or MDX](cellsets.md#execute-a-view-or-mdx) for reading the actual cell data out of it.

Confirmed for both view kinds. Executing a `NativeView` produces a cellset with one axis per axis you defined that has more than a single fixed member — `Rows` and `Columns` each became their own cellset axis, while the two `Titles` entries (Zones + Measures) were combined into a single third axis with one tuple carrying both selected members.

## Delete a view

```http
DELETE /{instance}/api/v1/Databases('{database}')/Cubes('{cube}')/Views('{name}')
```

```bash
curl -X DELETE "https://<host>/{instance}/api/v1/Databases('test')/Cubes('SalesPlanning')/Views('RegionByMeasure')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success.
