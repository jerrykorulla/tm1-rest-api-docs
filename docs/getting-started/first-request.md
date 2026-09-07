# First Request

This walks through calling the TM1 REST API end to end: authenticate, make a read-only request, then create your first database.

## Prerequisites

- The hostname of your TM1 / Planning Analytics environment (`<host>` below).
- The name of the TM1 instance you're connecting to (`{instance}` below).
- A username and password with access to it.

## Step 1: Authenticate

The simplest way to start is HTTP Basic, using your TM1 username and password directly on each request — see [Authentication](authentication.md) for the other options (bearer tokens and service-to-service sessions), which are better suited to production integrations.

## Step 2: Make a read-only request

Confirm everything is working by listing the databases you already have access to:

=== "curl"

    ```bash
    curl "https://<host>/{instance}/api/v1/Databases" \
      -u "$TM1_USER:$TM1_PASSWORD"
    ```

=== "Python"

    ```python
    import requests

    response = requests.get(
        "https://<host>/{instance}/api/v1/Databases",
        auth=(username, password),
    )
    response.raise_for_status()
    print(response.json())
    ```

A successful response is `200 OK` with a JSON array — empty if you haven't created any databases yet.

## Step 3: Create your first database

Now create one, per [Create a database](../api/databases.md#create-a-database):

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases" \
  -u "$TM1_USER:$TM1_PASSWORD" \
  -H "Content-Type: application/json" \
  -d '{ "Name": "GettingStarted" }'
```

The response comes back `201 Created`. Poll it until every entry in `ActiveReplicas` reports `"State": "ready"`:

```bash
curl "https://<host>/{instance}/api/v1/Databases('GettingStarted')" \
  -u "$TM1_USER:$TM1_PASSWORD"
```

Once it's ready, the database can hold [dimensions](../api/dimensions.md) and cubes.

## Next steps

- [Databases](../concepts/databases.md) — what you just created, conceptually.
- [Dimensions](../concepts/dimensions.md) — build the axes your cubes will use.
- [Authentication](authentication.md) — bearer tokens, service-to-service sessions, and closing sessions.
