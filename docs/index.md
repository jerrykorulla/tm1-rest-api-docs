# TM1 REST API Documentation

The TM1 REST API lets you manage and query IBM Planning Analytics (TM1) programmatically over HTTP — provision databases, model dimensions and cubes, read and write cell data, run MDX, and execute processes.

## Quickstart

```bash
TOKEN=$(curl -s -X POST "https://<host>/api/v1/Authenticate" \
  -u "$TM1_USER:$TM1_PASSWORD" \
  | jq -r .AccessToken)

curl "https://<host>/api/v1/Databases" \
  -H "Authorization: Bearer $TOKEN"
```

See [First Request](getting-started/first-request.md) for the full walkthrough, including creating your first database.

## Where to go next

| Section | What it's for |
|---|---|
| [Getting Started](getting-started/introduction.md) | API conventions, authentication, and your first call. |
| [Concepts](concepts/databases.md) | What each object — database, dimension, cube, view, cellset, cell — is and how they relate. |
| [Tutorials](tutorials/read-cube-data.md) | Task-oriented walkthroughs, like reading or writing cube data. |
| [API Reference](api/databases.md) | Endpoint-by-endpoint request and response detail. |
| [OData](odata/query-options.md) | Query options (`$filter`, `$select`, `$expand`, ...) supported across endpoints. |
| [Cookbook](cookbook/common-operations.md) | Short, copy-pasteable recipes for common operations. |
