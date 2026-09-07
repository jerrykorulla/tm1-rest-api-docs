# Processes API

Endpoints for creating, listing, executing, and deleting TurboIntegrator (TI) processes within a database.

!!! success "Verified 2026-09-07 against TM1 12.6.4"
    Create, List, Get, Execute, and Delete were all confirmed live against a local test instance (database `test`), including the duplicate-name error. Execution used a trivial process (a single Prolog line) — a process with a real `DataSource` (file/SQL/view-driven) was not exercised.

!!! note
    Every request below requires a valid `Authorization` header. See [Authentication](../getting-started/authentication.md).

## The process object

```json
{
  "Name": "LoadActuals",
  "HasSecurityAccess": false,
  "PrologProcedure": "sTest = 1;",
  "MetadataProcedure": "",
  "DataProcedure": "",
  "EpilogProcedure": "",
  "DataSource": { "Type": "None" },
  "Parameters": [],
  "Variables": [],
  "Attributes": { "Caption": "LoadActuals" }
}
```

A process is built from up to four procedures (each a TI script, run in this order): `PrologProcedure`, `MetadataProcedure`, `DataProcedure`, `EpilogProcedure`. `DataSource` describes where `MetadataProcedure`/`DataProcedure` iterate over (a file, SQL query, another cube view, or `{"Type": "None"}` for a process with no data source, as above).

## Create a process

```http
POST /{instance}/api/v1/Databases('{database}')/Processes
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Name` | string | Yes | Unique name for the process within the database. |
| `PrologProcedure` / `MetadataProcedure` / `DataProcedure` / `EpilogProcedure` | string | No | TI script for that stage. Omitted stages default to an empty string. |
| `DataSource` | object | No | Defaults to `{"Type": "None"}`. |
| `Parameters` | array | No | Runtime parameters the process accepts — defaults to `[]`. |

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases('test')/Processes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"Name\": \"LoadActuals\", \"PrologProcedure\": \"sTest = 1;\"}"
```

### Response

`201 Created`, with a `Location` header (e.g. `Processes('LoadActuals')`) and the new [process object](#the-process-object) in the body.

### Errors

All errors share the same envelope, `{"error": {"code": "...", "message": "..."}}`:

| Status | `message` | Meaning |
|---|---|---|
| `400 Bad Request` | `A process with name "{name}" already exists.` | A process with that name already exists — `400`, not `409`. |
| `401 Unauthorized` | `Authentication required` | The credentials are missing, expired, or invalid. |
| `404 Not Found` | — | The database in the URL doesn't exist. |

## List processes

```http
GET /{instance}/api/v1/Databases('{database}')/Processes
```

```bash
curl "https://<host>/{instance}/api/v1/Databases('test')/Processes" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `{"value": []}` on a database with no processes defined — confirmed; TM1 doesn't ship any built-in ones.

## Get a process

```http
GET /{instance}/api/v1/Databases('{database}')/Processes('{name}')
```

```bash
curl "https://<host>/{instance}/api/v1/Databases('test')/Processes('LoadActuals')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `404 Not Found` if no process with that name exists.

## Execute a process

```http
POST /{instance}/api/v1/Databases('{database}')/Processes('{name}')/tm1.Execute
```

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `Parameters` | array of `{Name, Value}` | No | Overrides for the process's declared parameters. |

```bash
curl -X POST "https://<host>/{instance}/api/v1/Databases('test')/Processes('LoadActuals')/tm1.Execute" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{}"
```

!!! warning
    Send a JSON body — even `{}` — with this action. A `POST` with no body at all fails with `400 Bad Request` / `{"error": {"code": "173", "message": "no content-length or transfer-encoding header provided"}}`, confirmed live (on the equivalent view-execute action; the same requirement applies here).

### Response

`204 No Content` on a successful run with no return value, confirmed for a trivial process. The service's `$metadata` also documents an `ExecuteWithReturn` action (returning execution status/output) and per-process `ErrorLogs`, neither of which was exercised live.

## Delete a process

```http
DELETE /{instance}/api/v1/Databases('{database}')/Processes('{name}')
```

```bash
curl -X DELETE "https://<host>/{instance}/api/v1/Databases('test')/Processes('LoadActuals')" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success.
