# Databases

A **database** is the top-level container in TM1 / IBM Planning Analytics. It's a single, independently managed TM1 instance that holds its own cubes, dimensions, processes, and data — comparable to what earlier TM1 documentation called a "server" or "model."

Everything else in this API — [dimensions](dimensions.md), [cubes](cubes.md), [views](views.md), [cellsets](cellsets.md), and [cells](cells.md) — lives *inside* a database and is addressed relative to it.

## Lifecycle

A database goes through a small number of states as it's provisioned and torn down:

| State | Meaning |
|---|---|
| `Provisioning` | The database has been requested and infrastructure is being allocated. |
| `Running` | The database is available and accepting requests. |
| `Stopped` | The database exists but is not currently running. |
| `Deleting` | The database has been marked for deletion and is being removed. |

You create a database once and then interact with it repeatedly through the [Databases API](../api/databases.md) and the object APIs scoped underneath it.

## Addressing a database

Once a database exists, requests to objects inside it are scoped by the database name in the URL path, e.g.:

```
/api/v1/Databases('SalesPlanning')/Cubes
```

See [Creating a database](../api/databases.md#create-a-database) for how to provision one, and [First Request](../getting-started/first-request.md) for a walkthrough of calling the API end to end.
