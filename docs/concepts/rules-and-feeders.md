# Rules and Feeders

A [cube](cubes.md)'s `Rules` field holds a script, written in TM1's **rule language** (distinct from TurboIntegrator's *process* language, despite both being commonly called "TI"), that derives some cells' values from others instead of storing them directly. This page covers the rule and feeder syntax well enough to read, write, and debug a cube's `Rules` through the REST API — it isn't a full language reference.

## Rule statements

A rules script is a sequence of statements, each ending in `;`. The core statement shape is:

```
[area] = TYPE: formula;
```

- **`area`** selects which cells the statement applies to: a comma-separated list of element names, positional against the cube's dimensions in order, e.g. `['Jan','Sales']` for a cube whose first two dimensions are `Time` and `Measure`. A trailing dimension can be omitted to mean "every element on it."
- **`TYPE`** is `N` for a numeric formula or `S` for a string formula.
- **`formula`** is the expression to evaluate, which can reference other cells — including cells in *other* cubes — via `DB('CubeName', elem1, elem2, ...)`.

```
['Total']=N:1;
```

is the simplest possible rule: every cell where the relevant dimension's element is `Total` gets the constant value `1`.

Inside a formula, `!DimensionName` refers to the *current* cell's element on that dimension — useful for pulling the corresponding cell from another cube:

```
['Variance']=N:DB('SalesPlanning', !Region, !Time, 'Actual') - DB('SalesPlanning', !Region, !Time, 'Budget');
```

This computes `Variance` as `Actual - Budget` for whatever `Region`/`Time` element the consuming query is looking at, by pulling both source values from the (in this example, same) cube.

### Overlapping statements

If more than one statement's area matches a given cell, **the statement that appears later in the script wins** — write general rules first and more specific overrides after them, not the other way around.

### `STET`

`STET` (in place of a formula) tells TM1 to leave the cell's stored/fed value alone instead of computing it — useful for carving an exception out of a broader rule:

```
['Actual']=N:STET;
```

This is typically paired with a preceding, broader statement that would otherwise force a computed value onto cells that should stay user-editable input.

### Other rule-file keywords

- **`SKIPCHECK;`** — skip TM1's check for whether a cell is already fed before evaluating a rule against it. A performance optimization for rules with heavy consolidations; used incorrectly, it can cause cells to be silently skipped. Treat it as an advanced, measure-first optimization, not a default.
- **`FEEDSTRINGS;`** — allow string cells to be fed (by default, feeding is a numeric-cell concept). Needed if your rules compute `S:` values that participate in feeding.

## Feeders

Rule-derived cells are calculated **lazily** — TM1 does not evaluate every possible rule-derived cell in a cube up front (for most cubes that would be an astronomically large space). Instead, it only evaluates a rule-derived cell when something asks for it, and — critically — it only includes a rule-derived cell in a **consolidation** if that cell has been explicitly marked as "reachable" via a **feeder** statement.

A feeder statement lives after a `FEEDERS;` marker at the end of the rules script and has the shape:

```
FEEDERS;
[source area] => [target area];
```

It reads as: "whenever a cell in `source area` has a value, make sure `target area`'s rule-derived cells are included when consolidating." Continuing the variance example above:

```
['Total']=N:1;

['Variance']=N:DB('SalesPlanning', !Region, !Time, 'Actual') - DB('SalesPlanning', !Region, !Time, 'Budget');

FEEDERS;
['Actual']=>['Variance'];
['Budget']=>['Variance'];
```

Without those two feeder statements, a query for a *specific, precisely-addressed* `Variance` cell would still return the correct computed value (see [Why direct reads still work](#why-direct-reads-still-work) below) — but a consolidated total that rolls up several `Variance` cells could silently omit them, understating the total with no error raised anywhere.

### Feed leaves, not consolidations

Feed statements should target **N-level (leaf) elements**. TM1 automatically propagates a fed leaf's value up through every consolidated ancestor above it in whatever hierarchy it belongs to — you don't feed each consolidated parent separately, and feeding a consolidated element directly does not cause its children to be included.

### Feeding across cubes

A rule's formula can pull from another cube via `DB(...)`; when it does, the *source* cube also needs a feeder pointing at the *rule* cube, qualified with the target cube's name:

```
FEEDERS;
['Actual']=>SalesPlanning:['Variance'];
```

Feeders live in whichever cube contains the cells being read by `DB()` — not necessarily the cube whose rule you're writing.

### Under-feeding vs. over-feeding

- **Under-feeding** (missing or too-narrow feeders) is the dangerous direction: consolidated totals quietly exclude rule-derived contributions. There's no error — the numbers are just wrong.
- **Over-feeding** (feeders broader than necessary, e.g. feeding a whole dimension with `*` when only a few elements need it) is a performance problem, not a correctness one: TM1 keeps feeder lists in memory per cube, and an overly broad feeder inflates that memory and slows recalculation after data loads.

### Why direct reads still work

A cellset query that pins down one specific, non-consolidated cell is a direct lookup: TM1 evaluates that cell's rule on demand regardless of feeding. Feeding only matters once a **consolidation** is involved, because consolidating is a sum over a cell's children, and TM1 needs the feeder information to know which rule-derived children to include in that sum. So through the REST API:

- Reading a single `RuleDerived` leaf cell (see [Cells — Cell status](cells.md#cell-status)) always reflects the rule, feeders or not.
- Reading a `Consolidated` cell whose children include rule-derived values requires those children to be correctly fed, or the total can be understated.

If a consolidated total looks stale or too low after writing to a cell that a rule depends on, suspect a missing or too-narrow feeder before suspecting the API.

## Rules and feeders through the REST API

- Both live in the same place: the cube's `Rules` string field (`null` if the cube has no rule). See [the cube object](../api/cubes.md#the-cube-object) and [Update a cube](../api/cubes.md#update-a-cube) for setting it via `PATCH`.
- `POST .../Cubes('{name}')/tm1.CheckRules` validates the syntax of the whole `Rules` string, including any `FEEDERS;` block, and returns `201 Created` with `{"value": []}` when it compiles. It checks **syntax only** — it can't tell you a feeder is missing or too narrow, since that's a semantic property of your model, not a parse error. There is no separate endpoint for validating feeder completeness; that's a design/testing concern (build the model, load representative data, and confirm consolidated totals match expectations) rather than something the API surfaces directly.
- There's no way to inspect "what does this feeder feed" as structured data via the API — the `Rules` string is the only representation. Parsing out the `FEEDERS;` block yourself is the only option if you need to reason about feeders programmatically.

See [Cells — Cell status](cells.md#cell-status) for how `RuleDerived` and `Consolidated` show up on individual cells, and [Cellsets API](../api/cellsets.md) for reading/writing the cells a rule and its feeders affect.
