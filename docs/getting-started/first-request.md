# First Request

This walks through calling the TM1 REST API end to end: authenticate, make a read-only request, then create your first database.

## Prerequisites

- The hostname of your TM1 / Planning Analytics environment (`<host>` below).
- A username and password with access to it.

## Step 1: Authenticate

Exchange your credentials for a bearer token, as described in [Authentication](authentication.md):

```bash
TOKEN=$(curl -s -X POST "https://<host>/api/v1/Authenticate" \
  -u "$TM1_USER:$TM1_PASSWORD" \
  | jq -r .AccessToken)
```

Every request from here on sends this token in the `Authorization` header.

## Step 2: Make a read-only request

Confirm everything is working by listing the databases you already have access to:

=== "curl"

    ```bash
    curl "https://<host>/api/v1/Databases" \
      -H "Authorization: Bearer $TOKEN"
    ```

=== "Python"

    ```python
    import requests

    response = requests.post(
        "https://<host>/api/v1/Authenticate",
        auth=(username, password),
    )
    response.raise_for_status()
    token = response.json()["AccessToken"]

    response = requests.get(
        "https://<host>/api/v1/Databases",
        headers={"Authorization": f"Bearer {token}"},
    )
    response.raise_for_status()
    print(response.json())
    ```

A successful response is `200 OK` with a JSON array — empty if you haven't created any databases yet.

## Step 3: Create your first database

Now create one, per [Create a database](../api/databases.md#create-a-database):

```bash
curl -X POST "https://<host>/api/v1/Databases" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "Name": "GettingStarted" }'
```

The response comes back `201 Created` with `"State": "Provisioning"`. Poll it until it's ready:

```bash
curl "https://<host>/api/v1/Databases('GettingStarted')" \
  -H "Authorization: Bearer $TOKEN"
```

Once `State` is `"Running"`, the database is ready to hold [dimensions](../api/dimensions.md) and cubes.

## Next steps

- [Databases](../concepts/databases.md) — what you just created, conceptually.
- [Dimensions](../concepts/dimensions.md) — build the axes your cubes will use.
- [Authentication](authentication.md) — token expiry and error handling.
