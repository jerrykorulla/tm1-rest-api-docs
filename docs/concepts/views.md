# Views

A **view** is a saved slice of a [cube](cubes.md): which dimension elements go on rows, columns, and titles (the context you're filtering on). Views live under their cube and come in two kinds:

| Kind | Defined by | Use when |
|---|---|---|
| `MDXView` | A single MDX expression (`MDX` property) | You want full control over the query, or need something MDX can express that a simple grid can't. |
| `NativeView` | Explicit `Rows`/`Columns`/`Titles` selections | You're modeling a straightforward grid and want TM1 to manage the axis selections (typically backed by [subsets](dimensions.md)) for you. |

Both kinds are stored under `Cubes('{cube}')/Views` and share the same base fields (`Name`, `Attributes`); the API tells them apart with an `@odata.type` of `#tm1.MDXView` or `#tm1.NativeView`.

## Executing a view

A view is a saved *definition* — it doesn't hold data until you run it. Executing a view (`POST .../Views('{name}')/tm1.Execute`) evaluates it against the cube's current data and returns a **[cellset](cellsets.md)**: a live, point-in-time snapshot of the resulting cells. This is exactly what running the equivalent MDX directly via `ExecuteMDX` would give you — a saved `MDXView` is really just a name for a query you'd otherwise have to repeat.

See the [Views API](../api/views.md) for endpoint details.
