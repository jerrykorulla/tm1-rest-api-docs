# Cellsets

A **cellset** is the result of running an MDX query or executing a [view](views.md): a point-in-time snapshot of a set of [cells](cells.md), organized into axes (typically rows and columns, sometimes also a title/context axis). It's what you get back from `ExecuteMDX` or a view's `tm1.Execute`, and it's also how you read or write individual cell values — there's no separate flat "give me this one cell" endpoint; see [Cells](cells.md).

A cellset is **transient and session-scoped**: it exists only for your current session, isn't guaranteed to still be there if you reconnect, and is deleted automatically when the session ends (or you can delete it explicitly once you're done with it, to free resources sooner).

## Structure

| Part | What it is |
|---|---|
| `Axes` | The query's axes (e.g. columns = axis 0, rows = axis 1). Each axis has `Tuples`, and each tuple has `Members` — the dimension elements that identify that row/column. |
| `Cells` | A flat, ordinally-indexed list of the actual cell values, in the order implied by the axes. |

## Snapshots don't refresh themselves

Because a cellset is a snapshot, **writing a new value into one of its cells does not change what that same cellset reports back if you re-read it** — confirmed directly: `PATCH`ing a cell's `Value`, then immediately `GET`ing that same cell from the same cellset, still returned the pre-write value. The write did land on the underlying cube (a *freshly executed* cellset against the same cube showed the new value). If you need to see the effect of a write, re-run the query (or execute the view again) rather than re-reading the cellset you wrote through.

See the [Cellsets API](../api/cellsets.md) for endpoint details.
