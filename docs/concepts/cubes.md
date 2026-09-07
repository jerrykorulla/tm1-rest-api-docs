# Cubes

A **cube** is a multidimensional array of data, shaped by an ordered list of [dimensions](dimensions.md). Every cell in a cube is addressed by picking one element from each of the cube's dimensions — a 3-dimensional cube with `Region`, `Product`, and `Time` dimensions addresses a cell as `(Region element, Product element, Time element)`.

A cube's shape is fixed at creation by which dimensions it has and in what order; the dimensions themselves (their elements and hierarchies) can keep changing afterward. Before creating a cube, every dimension it needs must already exist — see [Creating a dimension](../api/dimensions.md#create-a-dimension).

## Rules

A cube can have **rules** — a script (TurboIntegrator's rule language, not TI processes) that derives some cells' values from others instead of storing them directly, e.g. computing a margin cube cell from revenue and cost cube cells. Rule-derived cells are read-only from the API's perspective; you can't overwrite a value that a rule computes.

## Views

You rarely query a cube's raw cell space directly. Instead you define a **[view](views.md)** — a saved (or ad hoc) slice of the cube along particular rows, columns, and titles — and execute that. Views live under their cube; see [Views](views.md).

## Addressing a cube

```
/{instance}/api/v1/Databases('{db}')/Cubes('{cube}')
```

See the [Cubes API](../api/cubes.md) for endpoint details.
