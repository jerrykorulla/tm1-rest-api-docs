# Databases

A **database** is a data container in TM1 / IBM Planning Analytics: it holds its own cubes, dimensions, processes, and data — comparable to what earlier TM1 documentation called a "server" or "model." A database is not the top-level container, though — it lives inside a TM1 **instance**, and in turn holds its own **[users](users.md)** and groups. See [Architecture](architecture.md) for the full instance/database/user hierarchy.

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

Once a database exists, requests to objects inside it are scoped by the instance and database name in the URL path, e.g.:

```
/{instance}/api/v1/Databases('SalesPlanning')/Cubes
```

See [Creating a database](../api/databases.md#create-a-database) for how to provision one, and [First Request](../getting-started/first-request.md) for a walkthrough of calling the API end to end.

## Who can access a new database

Whoever creates a database — any authenticated user via the Service API, or root credentials via the [Management API](../api/management.md) — automatically becomes a member of its `ADMIN` [group](groups.md). No one else can access it until that admin adds them via the [Users API](../api/users.md#create-a-user). See [Users](users.md) and [Architecture](architecture.md#who-can-create-a-database) for the full picture.
