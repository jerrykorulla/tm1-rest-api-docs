# Cells API

There is no separate `Cells` endpoint on a cube. Confirmed against this API's `$metadata`: `Cube` has no `Cells` navigation property at all — cell access is always mediated by a [cellset](cellsets.md), produced by running MDX or executing a view.

!!! success "Verified 2026-09-07 against TM1 12.6.4"
    Confirmed against a local test instance (database `test`) that reading and writing a cell's value only works through a cellset — see [Cellsets API — Read or write a single cell](cellsets.md#read-or-write-a-single-cell) for the actual requests.

## Reading or writing a cell

1. Get a cellset that resolves to the cell(s) you want — either [`ExecuteMDX`](cellsets.md#execute-a-view-or-mdx) with an MDX query whose axes pin down the coordinates you need, or [execute a saved view](views.md#execute-a-view).
2. `GET` or `PATCH` the cell by its ordinal position in that cellset: `Cellsets('{id}')/Cells({ordinal})`. Writing several cells at once is also possible via `tm1.UpdateCells` on the cellset — see [Cellsets API — Write multiple cells at once](cellsets.md#write-multiple-cells-at-once).

See [Cells](../concepts/cells.md) for what makes up a cell beyond its raw value (status, whether it's consolidated or rule-derived), and [Cellsets API](cellsets.md) for the full request/response detail, including the confirmed snapshot-staleness gotcha after a write.
