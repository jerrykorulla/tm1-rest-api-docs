# Cellsets API

Endpoints for running MDX, and for reading and writing the cells of the resulting [cellset](../concepts/cellsets.md). Executing a [view](views.md) (`tm1.Execute`) produces a cellset the same way — see [Views API — Execute a view](views.md#execute-a-view).

!!! success "Verified 2026-09-07 against TM1 12.6.4"
    `ExecuteMDX`, reading a cellset with `$expand` on `Axes`/`Cells`, writing a cell's `Value` (both single-cell `PATCH` and bulk `tm1.UpdateCells`), the write-rejection errors for consolidated/rule-derived cells, all three `Status` values (including forcing a real `Error` cell), and deleting a cellset were all confirmed live against a local test instance (database `test`).

!!! note
    Every request below requires a valid `Authorization` header. See [Authentication](../getting-started/authentication.md).

## Execute a view or MDX

Running an MDX expression directly, at the database root:

```http
POST /{instance}/api/v1/Databases('{database}')/ExecuteMDX
```

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases('test')/ExecuteMDX" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"MDX\": \"SELECT {[Measure].[Revenue]} ON COLUMNS, {[Region].[North],[Region].[South]} ON ROWS FROM [SalesPlanning]\"}"
```

### Response

`201 Created`, with a `Location` header (e.g. `Cellsets('AAAAAEYAAIAlAAAg')`) and just the cellset's `ID` in the body:

```json
{ "ID": "AAAAAEYAAIAlAAAg" }
```

The ID is only valid within your current session — see [Cellsets](../concepts/cellsets.md).

## Get a cellset

The initial response has no data — fetch `Axes` and `Cells` with `$expand`:

```http
GET /{instance}/api/v1/Databases('{database}')/Cellsets('{id}')?$expand=Axes($expand=Tuples($expand=Members($select=Name))),Cells($select=Ordinal,Value,FormattedValue)
```

```bash
curl "https://<host>/{instance}/api/v1/Databases('test')/Cellsets('AAAAAEYAAIAlAAAg')?%24expand=Axes(%24expand=Tuples(%24expand=Members(%24select=Name))),Cells(%24select=Ordinal,Value,FormattedValue)" \
  -H "Authorization: Bearer $TOKEN"
```

### Response

```json
{
  "ID": "AAAAAEYAAIAlAAAg",
  "Axes": [
    { "Ordinal": 0, "Cardinality": 1, "Tuples": [{ "Ordinal": 0, "Members": [{ "Name": "Revenue" }] }] },
    { "Ordinal": 1, "Cardinality": 2, "Tuples": [
      { "Ordinal": 0, "Members": [{ "Name": "North" }] },
      { "Ordinal": 1, "Members": [{ "Name": "South" }] }
    ]}
  ],
  "Cells": [
    { "Ordinal": 0, "Value": null, "FormattedValue": "0.00" },
    { "Ordinal": 1, "Value": null, "FormattedValue": "0.00" }
  ]
}
```

Axis `0` is columns, axis `1` is rows (a third axis, if present, is the title/context axis). `Cells` is flat and ordinal-indexed in axis order — correlate a cell back to its row/column via the matching tuple ordinals on each axis. `Value` is `null` with `FormattedValue: "0.00"` for an empty (no data) numeric cell.

A cell has more fields than shown above — `$select` any of `Ordinal`, `Value`, `FormattedValue`, `FormatString`, `Status`, `Consolidated`, `RuleDerived`, `NullIntersected`, `Annotated`, `Updateable`, `HasPicklist`, `HasDrillthrough`. All three confirmed live, on real cells built for the purpose:

| `Status` | When | Confirmed example |
|---|---|---|
| `Null` | The cell has no data (nothing stored, no rule). | `{"Status": "Null", "Value": null, "FormattedValue": "0.00"}` |
| `Data` | The cell holds a value — stored directly, or computed by a rule. | `{"Status": "Data", "Value": 999, "RuleDerived": true}` for a rule; `{"Status": "Data", "Value": 77, "RuleDerived": false}` for a plain stored value. |
| `Error` | A rule failed to evaluate for this cell (e.g. division by zero). | `{"Status": "Error", "Value": "#N/A", "FormattedValue": "#N/A", "RuleDerived": true}`, from a rule containing `1/0`. |

`NullIntersected` was `false` on every cell checked (including the `Null`-status one) — confirmed present in a live response when explicitly `$select`-ed, but a case that makes it `true` wasn't found.

`Consolidated` and `RuleDerived` cells are read-only — see [Read or write a single cell](#read-or-write-a-single-cell) below for the exact rejection errors, confirmed live.

## Read or write a single cell

```http
GET /{instance}/api/v1/Databases('{database}')/Cellsets('{id}')/Cells({ordinal})
PATCH /{instance}/api/v1/Databases('{database}')/Cellsets('{id}')/Cells({ordinal})
```

```bash
curl -X PATCH "https://<host>/{instance}/api/v1/Databases('test')/Cellsets('AAAAAEYAAIAlAAAg')/Cells(0)" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"Value\": 1234.5}"
```

`PATCH` returns `204 No Content` and writes straight through to the underlying cube.

### Errors

Writing to a consolidated or rule-derived cell is rejected — both confirmed live, by building a cell of each kind and attempting to `PATCH` it:

| Status | `code` | `message` | Meaning |
|---|---|---|---|
| `400 Bad Request` | `147` | `CubeCellWriteStatusRuleApplies` | The cell's value is computed by a cube rule — write to the rule's inputs instead. |
| `400 Bad Request` | `148` | `CubeCellWriteStatusElementIsConsolidated` | The cell is a consolidation of other cells — write to its leaf children instead. |

!!! warning "Snapshot staleness — confirmed"
    Re-reading `Cells(0)` on the **same** cellset right after the `PATCH` above still returned the old value (`null`). The write had, in fact, taken effect — executing a **fresh** `ExecuteMDX` against the same coordinates showed `1234.5`. A cellset doesn't refresh itself just because you wrote through it; re-run the query (or the view) to see the new value. See [Cellsets — Snapshots don't refresh themselves](../concepts/cellsets.md#snapshots-dont-refresh-themselves).

## Write multiple cells at once

```http
POST /{instance}/api/v1/Databases('{database}')/Cellsets('{id}')/tm1.UpdateCells
```

A bound action for writing more than one cell of a cellset in a single call, rather than one `PATCH` per ordinal.

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Updates` | array of `{Ordinal, Value}` | Yes | The cells to write. `Ordinal` matches the cellset's `Cells` ordinal; `Value` is the new value. |
| `Order` | number | No | Per `$metadata`: `0` for no ordering (default), `1` to apply leaf cells first, then their parents, and so on — relevant when an update touches both a consolidated cell and its children. Not independently exercised. |

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases('test')/Cellsets('AAAAAEYAAIC3AAAg')/tm1.UpdateCells" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"Updates\": [{\"Ordinal\": 0, \"Value\": 77}]}"
```

### Response

`204 No Content` on success, confirmed live — the write was verified by re-querying with a fresh cellset (per the staleness warning above) and seeing `"Status": "Data", "Value": 77`. Only a single-entry `Updates` array was tested; multiple entries per call should work the same way since `Updates` is a plain collection, but that wasn't independently exercised. `Updates` must be present — an empty or missing body is rejected with `400 Bad Request` / `{"error": {"code": "278", "message": "Missing parameters. Please specify at least an Updates parameter."}}`, confirmed live.

!!! note
    The service's `$metadata` also documents a *cube*-level `tm1.UpdateCells` (`POST .../Cubes('{cube}')/tm1.UpdateCells`, addressing cells by an element tuple instead of a cellset ordinal) — not exercised here.

## Delete a cellset

```http
DELETE /{instance}/api/v1/Databases('{database}')/Cellsets('{id}')
```

```bash
curl -X DELETE "https://<host>/{instance}/api/v1/Databases('test')/Cellsets('AAAAAEYAAIAlAAAg')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success. Not required — a cellset is deleted automatically when your session ends — but frees resources sooner if you're done with it.
