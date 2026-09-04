# Dimensions

A **dimension** defines one axis of a [cube](cubes.md) — for example `Time`, `Region`, or `Product`. Every cube is built from an ordered list of dimensions, so a dimension must exist before it can be referenced when creating a cube.

## Elements

A dimension is made up of **elements** — the individual members you can slice data by. Each element has a type:

| Type | Meaning |
|---|---|
| `Numeric` | A leaf element that stores numeric values, e.g. `North`, `South`. |
| `String` | A leaf element that stores text values. |
| `Consolidated` | A parent element that aggregates other elements (its children), e.g. `Total Region` rolling up `North` + `South`. |

Consolidated elements don't store data directly — their value is derived by summing (or otherwise aggregating) their children.

## Hierarchies

Elements live inside a **hierarchy**, and a dimension can contain more than one. Every dimension has a default hierarchy with the same name as the dimension itself; additional hierarchies let you expose alternate rollups of the same elements (e.g. a `Region` dimension with both a geographic hierarchy and a sales-org hierarchy) without duplicating them.

Consolidations — which elements roll up into which — are defined as **edges** between a parent element and a child element within a hierarchy, not as a property of the element itself.

## Relationship to cubes

A cube's shape is defined entirely by the dimensions assigned to it, in order. Before [creating a cube](cubes.md), make sure every dimension it needs already exists — see [Creating a dimension](../api/dimensions.md#create-a-dimension).

## Addressing a dimension

Dimensions are scoped under a database:

```
/api/v1/Databases('SalesPlanning')/Dimensions('Region')
```

And a hierarchy is scoped under its dimension:

```
/api/v1/Databases('SalesPlanning')/Dimensions('Region')/Hierarchies('Region')
```
