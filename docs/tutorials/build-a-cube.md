# Build a Cube from Scratch

This walks through the full lifecycle of a small planning cube using nothing but the REST API: create a database, define dimensions, create a cube, load some data, save an MDX view and a native view, then add a rule with feeders for a computed `Variance` measure.

The end result is a tiny `SalesPlanning` cube with `Region` × `Time` × `Measure`, where `Variance` is calculated as `Actual - Budget` instead of being loaded directly.

## Prerequisites

- A running TM1 instance and the database-creation rights to go with it — see [First Request](../getting-started/first-request.md).

Export the host and instance once and every command below can be copy-pasted verbatim:

```bash
export TM1_HOST="127.0.0.1:4444"
export TM1_INSTANCE="test"
```

The commands below assume the session is already authorized with no explicit credentials needed on each call (e.g. a browser-based or pre-authenticated session against a local instance). If your environment requires it, add `-u "$TM1_USER:$TM1_PASSWORD"` or `-H "Authorization: Bearer $TOKEN"` to each `curl` call — see [Authentication](../getting-started/authentication.md).

## Step 1 — Create the database

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases" \
  -H "Content-Type: application/json" \
  -d '{ "Name": "CubeTutorial" }'
```

Poll it until every entry in `ActiveReplicas` reports `"State": "ready"` before moving on — see [Create a database](../api/databases.md#create-a-database):

```bash
curl "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')"
```

The rest of this tutorial uses `CubeTutorial` as `{database}`.

## Step 2 — Create the dimensions

The cube needs three dimensions: `Region` and `Time` (each with a consolidation, to show how [feeders](../concepts/rules-and-feeders.md) affect roll-ups later) and `Measure` (flat — no consolidation).

### Region

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Dimensions" \
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

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Dimensions('Region')/Hierarchies('Region')/Edges" \
  -H "Content-Type: application/json" \
  -d '{ "ParentName": "Total Region", "ComponentName": "North", "Weight": 1 }'
```

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Dimensions('Region')/Hierarchies('Region')/Edges" \
  -H "Content-Type: application/json" \
  -d '{ "ParentName": "Total Region", "ComponentName": "South", "Weight": 1 }'
```

### Time

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Dimensions" \
  -H "Content-Type: application/json" \
  -d '{
        "Name": "Time",
        "Hierarchies": [
          {
            "Name": "Time",
            "Elements": [
              { "Name": "Jan", "Type": "Numeric" },
              { "Name": "Feb", "Type": "Numeric" },
              { "Name": "Q1", "Type": "Consolidated" }
            ]
          }
        ]
      }'
```

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Dimensions('Time')/Hierarchies('Time')/Edges" \
  -H "Content-Type: application/json" \
  -d '{ "ParentName": "Q1", "ComponentName": "Jan", "Weight": 1 }'
```

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Dimensions('Time')/Hierarchies('Time')/Edges" \
  -H "Content-Type: application/json" \
  -d '{ "ParentName": "Q1", "ComponentName": "Feb", "Weight": 1 }'
```

### Measure

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Dimensions" \
  -H "Content-Type: application/json" \
  -d '{
        "Name": "Measure",
        "Hierarchies": [
          {
            "Name": "Measure",
            "Elements": [
              { "Name": "Actual", "Type": "Numeric" },
              { "Name": "Budget", "Type": "Numeric" },
              { "Name": "Variance", "Type": "Numeric" }
            ]
          }
        ]
      }'
```

`Variance` is `Numeric`, not `Consolidated` — it's going to be *rule-derived*, not summed from children. See [Dimensions](../concepts/dimensions.md) and [Create a dimension](../api/dimensions.md#create-a-dimension) for more on element types and edges.

## Step 3 — Create the cube

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Cubes" \
  -H "Content-Type: application/json" \
  -d '{
        "Name": "SalesPlanning",
        "Dimensions@odata.bind": ["Dimensions('"'"'Region'"'"')", "Dimensions('"'"'Time'"'"')", "Dimensions('"'"'Measure'"'"')"]
      }'
```

Dimension order matters for addressing cells later — this cube is `(Region, Time, Measure)`. See [Create a cube](../api/cubes.md#create-a-cube).

## Step 4 — Load some data

`Actual` and `Budget` are plain input cells; `Variance` will be computed. Get a pinpoint cellset for each cell you want to write via `ExecuteMDX`, then `PATCH` its value — see [Cellsets API](../api/cellsets.md).

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/ExecuteMDX" \
  -H "Content-Type: application/json" \
  -d "{\"MDX\": \"SELECT {[Measure].[Actual],[Measure].[Budget]} ON COLUMNS, {([Region].[North],[Time].[Jan]),([Region].[South],[Time].[Jan]),([Region].[North],[Time].[Feb]),([Region].[South],[Time].[Feb])} ON ROWS FROM [SalesPlanning]\"}"
```

This returns `{"ID": "..."}` for a cellset with 4 rows × 2 columns = 8 cells, ordinals `0`–`7` in row-major order (`Actual` then `Budget` for North/Jan, then Actual/Budget for South/Jan, and so on). Write each with `tm1.UpdateCells` in one call:

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Cellsets('{id}')/tm1.UpdateCells" \
  -H "Content-Type: application/json" \
  -d '{
        "Updates": [
          { "Ordinal": 0, "Value": 1000 },
          { "Ordinal": 1, "Value": 900 },
          { "Ordinal": 2, "Value": 800 },
          { "Ordinal": 3, "Value": 850 },
          { "Ordinal": 4, "Value": 1100 },
          { "Ordinal": 5, "Value": 950 },
          { "Ordinal": 6, "Value": 750 },
          { "Ordinal": 7, "Value": 800 }
        ]
      }'
```

(`{id}` is the cellset `ID` from the `ExecuteMDX` response above.) See [Write multiple cells at once](../api/cellsets.md#write-multiple-cells-at-once) — remember a cellset's own snapshot won't reflect the write; re-run `ExecuteMDX` if you want to see updated values through the same cellset.

## Step 5 — Query it with ad hoc MDX

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/ExecuteMDX" \
  -H "Content-Type: application/json" \
  -d "{\"MDX\": \"SELECT {[Measure].[Actual]} ON COLUMNS, {[Region].[North],[Region].[South],[Region].[Total Region]} ON ROWS FROM [SalesPlanning] WHERE ([Time].[Jan])\"}"
```

Fetch the resulting cellset's cells with `$expand` per [Get a cellset](../api/cellsets.md#get-a-cellset). `Total Region` should already show `1800` (`1000 + 800`) — ordinary numeric consolidation doesn't need feeders, only *rule-derived* values do.

## Step 6 — Save it as an MDX view

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Cubes('SalesPlanning')/Views" \
  -H "Content-Type: application/json" \
  -d "{\"@odata.type\": \"#tm1.MDXView\", \"Name\": \"ActualByRegion\", \"MDX\": \"SELECT {[Measure].[Actual]} ON COLUMNS, {[Region].[North],[Region].[South],[Region].[Total Region]} ON ROWS FROM [SalesPlanning] WHERE ([Time].[Jan])\"}"
```

Run it any time with `POST .../Views('ActualByRegion')/tm1.Execute` — see [Execute a view](../api/views.md#execute-a-view).

## Step 7 — Add a native view

A `NativeView` pins each of the cube's three dimensions to an axis instead of writing MDX by hand. Here, `Region` on rows, `Time` on columns, `Measure` as a title fixed to `Actual`:

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Cubes('SalesPlanning')/Views" \
  -H "Content-Type: application/json" \
  -d '{
        "@odata.type": "tm1.NativeView",
        "Name": "RegionByTime",
        "Rows": [
          { "Subset": { "Hierarchy": { "@odata.id": "Dimensions('"'"'Region'"'"')/Hierarchies('"'"'Region'"'"')" }, "Expression": "{[Region].[Region].[Total Region].Children}" } }
        ],
        "Columns": [
          { "Subset": { "Hierarchy": { "@odata.id": "Dimensions('"'"'Time'"'"')/Hierarchies('"'"'Time'"'"')" }, "Expression": "{[Time].[Time].[Q1].Children}" } }
        ],
        "Titles": [
          {
            "Subset": { "Hierarchy": { "@odata.id": "Dimensions('"'"'Measure'"'"')/Hierarchies('"'"'Measure'"'"')" }, "Expression": "{[Measure].[Measure].[Actual]}" },
            "Selected@odata.bind": "Dimensions('"'"'Measure'"'"')/Hierarchies('"'"'Measure'"'"')/Elements('"'"'Actual'"'"')"
          }
        ]
      }'
```

The title's subset `Expression` resolves to a single member (`Actual`) that matches `Selected@odata.bind` — see the [Views API note](../api/views.md#create-a-view) on why `Selected` only reliably takes effect when it names an element the expression itself would return.

Run it with `POST .../Views('RegionByTime')/tm1.Execute`, same as the MDX view.

## Step 8 — Add the Variance rule and its feeders

`Variance` should be `Actual - Budget`, computed on demand rather than stored. Save it by `PATCH`ing the cube's `Rules`:

```bash
curl -X PATCH "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/Cubes('SalesPlanning')" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{"Rules": "['Variance']=N:DB('SalesPlanning', !Region, !Time, 'Actual') - DB('SalesPlanning', !Region, !Time, 'Budget');\n\nFEEDERS;\n['Actual']=>['Variance'];\n['Budget']=>['Variance'];"}
EOF
```

!!! warning "Why a heredoc here, not `-d '...'`"
    The rule text contains `!Region`/`!Time` (TM1's context operator). In an interactive bash session, `!` triggers history expansion — `bash: !Region,: event not found` — even inside double quotes; only single quotes suppress it, and this JSON body also needs its own literal single quotes (`['Variance']`), which would otherwise require escaping every one of them. A quoted heredoc (`<<'EOF'`) sidesteps both problems: no history expansion, no quote escaping, the body goes to `curl` exactly as written via `-d @-` (read the body from stdin).

`Cubes('SalesPlanning')/tm1.CheckRules` can validate a rule's syntax before saving it — see [the note under Update a cube](../api/cubes.md#update-a-cube) — but its exact request body isn't nailed down in this documentation yet, so this tutorial validates the straightforward way: send it via `PATCH` and see whether the cube accepts it.

Read [Rules and Feeders](../concepts/rules-and-feeders.md) for what each part of that string means — in short: the first statement computes `Variance` for any cell where `Measure` is `Variance`, pulling the matching `Actual`/`Budget` cell from the same cube via `!Region`/`!Time`; the `FEEDERS;` block is what makes `Variance` show up correctly once you *consolidate* it (`Total Region`, `Q1`) rather than just read a single leaf cell.

## Step 9 — Verify it

A pinpoint leaf cell reflects the rule immediately, feeders or not:

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/ExecuteMDX" \
  -H "Content-Type: application/json" \
  -d "{\"MDX\": \"SELECT {[Measure].[Variance]} ON COLUMNS, {[Region].[North]} ON ROWS FROM [SalesPlanning] WHERE ([Time].[Jan])\"}"
```

`$select=Value,Status,RuleDerived` on the resulting cell (see [Get a cellset](../api/cellsets.md#get-a-cellset)) should show `"Value": 100, "Status": "Data", "RuleDerived": true` (`1000 - 900`).

Now check the **consolidated** view — `Total Region` × `Q1` × `Variance` — which is exactly what the `FEEDERS;` block above exists for:

```bash
curl -X POST "http://$TM1_HOST/$TM1_INSTANCE/api/v1/Databases('CubeTutorial')/ExecuteMDX" \
  -H "Content-Type: application/json" \
  -d "{\"MDX\": \"SELECT {[Measure].[Variance]} ON COLUMNS, {[Region].[Total Region]} ON ROWS FROM [SalesPlanning] WHERE ([Time].[Q1])\"}"
```

This should come back `Consolidated: true` with a `Value` equal to the sum of all four `Variance` leaf cells (North/South × Jan/Feb). If you ever see a consolidated total that's missing a rule-derived contribution, the feeder covering that source measure is the first thing to check — see [Rules and Feeders — under-feeding vs. over-feeding](../concepts/rules-and-feeders.md#under-feeding-vs-over-feeding).

## Next steps

- [Rules and Feeders](../concepts/rules-and-feeders.md) — the concepts behind step 8, in depth.
- [Views](../concepts/views.md) and [Cellsets](../concepts/cellsets.md) — what `MDXView`/`NativeView` and their execution results actually are.
- [Cells — Cell status](../concepts/cells.md#cell-status) — `Status`, `Consolidated`, and `RuleDerived` on individual cells.
