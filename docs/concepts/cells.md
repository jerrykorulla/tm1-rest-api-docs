# Cells

A **cell** is a single value in a [cube](cubes.md), addressed by one element from each of the cube's dimensions. There is no standalone "give me cell (A, B, C)" endpoint, and no flat `Cells` collection hanging directly off a cube — every cell read or write happens through a **[cellset](cellsets.md)**: run an MDX query (or execute a [view](views.md)) that resolves to the cell(s) you want, then read or write `Value` on the resulting cellset's `Cells` entries.

For a single, precisely-addressed cell, the simplest MDX is a query whose rows and columns axes each resolve to exactly one member — see [Cellsets API — Read or write a single cell](../api/cellsets.md#read-or-write-a-single-cell).

## Cell status

A cell isn't just a bare number. Confirmed live, by building cells of each kind (see [Cellsets API — Get a cellset](../api/cellsets.md#get-a-cellset) for the full field list):

- **`Status`** is `Null` (no data), `Data` (a stored or rule-computed value), or `Error` (a rule failed to evaluate — e.g. division by zero produced `Value: "#N/A"`).
- **`Consolidated`** marks a rolled-up value (the sum, typically, of its children). **`RuleDerived`** marks a value computed by a cube [rule](rules-and-feeders.md). Both are read-only: writing to either is rejected outright, with a distinct error for each — see [Cellsets API — Errors](../api/cellsets.md#errors). A cell can be both: a consolidated total whose rule-derived children weren't properly [fed](rules-and-feeders.md#feeders) can understate the total with no error at all.
- **`NullIntersected`** exists and is returned when explicitly requested (confirmed `false` on every cell checked, including empty ones) — no case producing `true` was found.

See [Cells — via the Cellsets API](../api/cells.md) for how this maps to actual requests.
