# Groups

A **group** is a named permission set within a database. [Users](users.md) are assigned to one or more groups, and a user's effective permissions are the union of what their groups grant. Like users, groups are scoped to a single database — there is no instance-level or service-level group.

```http
GET /{instance}/api/v1/Databases('{db}')/Groups
```

```json
{
  "@odata.context": "$metadata#Groups",
  "value": [
    { "Name": "ADMIN" },
    { "Name": "DataAdmin" },
    { "Name": "OperationsAdmin" },
    { "Name": "SecurityAdmin" }
  ]
}
```

## Built-in groups

Every database ships with a set of built-in groups covering common administrative roles:

| Group | Grants |
|---|---|
| `ADMIN` | Full control over the database — users, groups, security, and data. See [Who becomes Admin](users.md#who-becomes-admin). |
| `DataAdmin` | Administrative access to data (cubes, dimensions, cells) without full security control. |
| `OperationsAdmin` | Administrative access to operational concerns (e.g. processes, sessions) without full security control. |
| `SecurityAdmin` | Administrative access to users and groups without full data control. |

!!! warning "Unverified"
    The exact permissions each non-`ADMIN` built-in group grants haven't been confirmed against live behavior — only their names, from a real `GET .../Groups` response. Treat the split above as directional (each covers a facet of what full `ADMIN` access grants) rather than an authoritative permission matrix.

Beyond these built-ins, a database can also define custom groups — there is no default "everyone"/catch-all group — for business-specific access, e.g. a `Planners` group for users who only need to write to planning cubes.

A user's [`Type` field](../api/users.md#the-user-object) mirrors whichever built-in group they're highest-ranked in (`User` if none) — confirmed by assigning a test user to `DataAdmin` and seeing `Type` change to match. Membership in a custom group like `Planners` doesn't affect `Type`.

## Assigning users to groups

Group membership is set on the user, via `Groups@odata.bind` — see [Create a user](../api/users.md#create-a-user) and [Update a user](../api/users.md#update-a-user). There is no separate "add member to group" call; you bind the group from the user side.

See the [Groups API](../api/groups.md) for endpoint details.
