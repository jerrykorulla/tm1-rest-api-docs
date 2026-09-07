# Users

A **user** is an identity that can authenticate against a specific TM1 database and hold permissions within it. Users are always scoped to a single database — see [Architecture](architecture.md) for how that fits into the instance/database hierarchy.

There is no instance-level or service-level user concept. A user created in one database has no standing in any other database, even on the same instance, and there is no `/manage/v1/Users` endpoint — users only ever exist under a database: `/{instance}/api/v1/Databases('{db}')/Users`.

## Groups and the ADMIN role

Users are assigned to **[groups](groups.md)**, and groups carry permissions. Every database has a built-in `ADMIN` group: membership in it grants full control over that database's users, groups, and security.

The `ADMIN` role is scoped *inside* a database — it does not extend to other databases, and it does not grant the ability to create new databases (creating databases is a [Service or Management API](architecture.md#who-can-create-a-database) capability available to any authenticated user or to root credentials, independent of any existing database's `ADMIN` group). See [Groups](groups.md) for the other built-in groups and how custom ones work.

## Who becomes Admin

Whoever creates a database — whether an ordinary authenticated user via the Service API, or root credentials via the Management API — is automatically:

- added to that database's Users list, and
- assigned to its `ADMIN` group.

No other users can access the new database until this creator adds them via [`POST .../Users`](../api/users.md#create-a-user).

!!! note "Built-in Admin user, as of v12.4.0"
    If a database is created by a user other than the built-in `Admin` user, TM1 does **not** automatically create the built-in `Admin` user — the creating user becomes the database's sole administrator instead. `Admin` is the user's `Name` as stored (confirmed via `GET .../Users`); login credentials are typically matched case-insensitively.

## Granting access without a direct user session

Some callers (for example Planning Analytics Workspace) need to act on behalf of many users without each of them authenticating directly against the database. This is handled through **Applications** — trusted service-to-service clients registered at the instance level — which can create sessions for a user that already exists in the target database. See [Applications and the S2S flow](architecture.md#applications-and-the-s2s-flow) for the full sequence.

See the [Users API](../api/users.md) for endpoint details.
