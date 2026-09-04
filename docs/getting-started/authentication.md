# Authentication

Every request to the TM1 REST API must include a bearer token in the `Authorization` header:

```
Authorization: Bearer <token>
```

Requests without a valid token — or with an expired one — receive `401 Unauthorized`.

## Obtaining a token

Exchange your username and password for a token by calling the authentication endpoint with HTTP Basic credentials:

```http
POST /api/v1/Authenticate
```

=== "curl"

    ```bash
    curl -X POST "https://<host>/api/v1/Authenticate" \
      -u "$TM1_USER:$TM1_PASSWORD"
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
    ```

### Response

```json
{
  "AccessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "ExpiresIn": 3600
}
```

`AccessToken` is the value you send as the bearer token; `ExpiresIn` is its lifetime in seconds.

## Using the token

Attach the token to every subsequent request:

```bash
curl "https://<host>/api/v1/Databases" \
  -H "Authorization: Bearer $TOKEN"
```

The examples throughout this documentation assume `$TOKEN` (or, in Python, a `token` variable) holds a valid access token obtained this way.

## Token expiry

Tokens expire after `ExpiresIn` seconds. Once a token expires, requests fail with `401 Unauthorized` — re-authenticate to obtain a new one rather than trying to refresh it in place.

!!! tip
    Don't re-authenticate on every request. Cache the token and its expiry, and only fetch a new one when it's close to expiring or a request comes back `401`.

## Errors

| Status | Meaning |
|---|---|
| `400 Bad Request` | The request is missing credentials or is malformed. |
| `401 Unauthorized` | The username or password is incorrect. |
| `429 Too Many Requests` | Too many authentication attempts in a short period. Back off before retrying. |

!!! warning
    Never hard-code credentials or tokens in source control. Load them from environment variables or a secrets manager, as shown in the examples above.
