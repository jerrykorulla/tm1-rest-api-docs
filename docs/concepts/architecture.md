# Architecture

TM1 v12 exposes two distinct API planes. Knowing which one you're calling — and which credentials it expects — determines what you can do and how you authenticate.

| API Plane | Base Path | Manages | Authentication |
|---|---|---|---|
| **Management API** | `/manage/v1/` | Instances, Applications, Databases | Root credentials |
| **Service API** | `/{instance}/api/v1/` | Databases, Users, Groups, Cubes, Sessions, etc. | User session |

Everything documented elsewhere in **API Reference** (Databases, Dimensions, Cubes, Views, Cellsets, Cells, Processes) is Service API — called against a specific instance and authenticated as a user. The Management API is a separate, infrastructure-level surface documented on its own page: [Management API](../api/management.md).

## Instances, databases, and users

```
TM1 Service
└── Instance  (logical isolation, managed via /manage/v1/)
    └── Database  (data container, has its own Users and Groups)
        └── User  (exists only within a Database)
```

- An **instance** is a logical boundary that separates customer environments from one another. Instances are created and managed through the Management API.
- A **[database](databases.md)** lives inside an instance and holds all TM1 data — cubes, dimensions, processes, and so on.
- A **[user](users.md)** exists only inside a specific database. There is no instance-level or service-level user concept — a login that works against one database has no standing in another, even within the same instance.

This nesting is why every Service API path is shaped `/{instance}/api/v1/Databases('{db}')/...`: you're always addressing a database through the instance that hosts it, and (for anything below the database itself, like Users, Cubes, or Dimensions) the database that contains it.

## Root credentials

Root credentials are a master `client_id`/`client_secret` pair that govern the Management API. They can:

- Create and delete instances
- Create and delete databases within an instance
- Register [Applications](#applications-and-the-s2s-flow) (trusted service-to-service clients) for an instance

**Root credentials cannot access TM1 database data.** They are purely administrative at the infrastructure level — they can bring a database into existence, but they can't read a cube or list a database's users. Once a database exists, working with its data requires a user session against the Service API.

## Who can create a database

| Credential type | Can create a database? | Notes |
|---|---|---|
| Root credentials | Yes — via the [Management API](../api/management.md) | Cannot access database data |
| Any authenticated user session | Yes — via the [Service API](../api/databases.md#create-a-database) | Becomes Admin of the new database |
| A database's own `ADMIN` group member | Not by virtue of that role | `ADMIN` is scoped to within an existing database — see below |

Whichever path is used, the user (or credential) that creates a database is automatically added to that database's Users list and assigned to its `ADMIN` group. No one else can access the database until the creator adds them — see [Users](users.md).

The database-level `ADMIN` group is a role *inside* a database — one of several built-in groups; see [Groups](groups.md). It grants control over that database's users, groups, and security — it does not grant the ability to create new databases. Being an `ADMIN` of one database gives you no special standing in any other database, even on the same instance.

## Applications and the S2S flow

Because users don't exist at the instance level, giving a third-party service (like Planning Analytics Workspace) the ability to act on behalf of arbitrary users is brokered through **Applications** — trusted service-to-service (S2S) clients registered against an instance. This is one of [three ways to obtain a session](../getting-started/authentication.md#how-a-session-is-created); the others are presenting a bearer token from an external Authorization Server, or HTTP pass-through to an external auth service.

1. **Register an Application** for the instance using root credentials:

    ```http
    POST /manage/v1/Instances('{instance}')/Applications
    Authorization: Basic {ROOT_CLIENT_ID}:{ROOT_CLIENT_SECRET}

    { "Name": "PAW" }
    ```

    The response includes a `ClientID` and `ClientSecret` for that application.

2. **Exchange those credentials for an access token:**

    ```http
    POST /{instance}/auth/v1/token
    Content-Type: application/json

    {
        "grant_type": "client_credentials",
        "client_id": "{App.ClientID}",
        "client_secret": "{App.ClientSecret}"
    }
    ```

3. **Create a session on behalf of a user** using that access token:

    ```http
    POST /{instance}/api/v1/Sessions
    Authorization: Bearer {access_token}

    { "UserName": "testuser@ibm.com" }
    ```

    The response sets a `TM1SessionId` cookie, used on subsequent requests in place of the `Authorization` header. See the [Sessions API](../api/sessions.md) for full detail, including optional session variables.

4. **The user must already exist in the target database** — an Application can create sessions for users, but it can't conjure a user that hasn't been added via [`POST .../Users`](../api/users.md#create-a-user).

!!! warning "Deprecated endpoint"
    `POST /{instance}/auth/v1/session` (singular, taking the Application's credentials directly as HTTP Basic) still exists but is deprecated in favor of the token-then-session flow above.

## Key takeaways

1. **Users are database-scoped.** There is no `/manage/v1/Users` endpoint. Always go through `/{instance}/api/v1/Databases('{db}')/Users`.
2. **Root credentials ≠ an ADMIN user.** Root credentials manage infrastructure (instances, databases). `ADMIN` is a security [group](groups.md) inside a database.
3. **Any authenticated user can create a database** via the Service API — not just a user already in some `ADMIN` group. Whoever creates it automatically becomes a member of the new database's `ADMIN` group.
4. **Root credentials can also create databases**, via the Management API, but can't read or write database data.
5. **Applications broker user access.** A third-party service that needs to create sessions for users registers as an Application (via root credentials) and then authenticates as that application to create sessions on users' behalf.
