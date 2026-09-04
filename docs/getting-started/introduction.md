# Introduction

The TM1 REST API lets you manage and query IBM Planning Analytics (TM1) programmatically over HTTP. Anything you can do in the TM1 client tools — create a database, define dimensions and cubes, read or write cell data, run MDX queries, execute processes — you can also do with a plain HTTP request.

## What you can do with it

- **Provision and manage databases** — create, inspect, and delete TM1 databases.
- **Model data** — define [dimensions](../concepts/dimensions.md), [cubes](../concepts/cubes.md), and their structure.
- **Read and write data** — query and update [cells](../concepts/cells.md) through [views](../concepts/views.md) and [cellsets](../concepts/cellsets.md).
- **Run logic** — execute MDX queries and TI processes against your data.

## Conventions

The API follows [OData v4](https://www.odata.org/) conventions:

- **Base URL** — every endpoint is rooted at `https://<host>/api/v1/`.
- **JSON** — requests and responses use `application/json`.
- **Entity addressing** — named entities are addressed with single quotes inside parentheses, e.g. `Databases('SalesPlanning')`, `Dimensions('Region')`.
- **Nesting** — child resources are addressed by chaining paths, e.g. `Databases('SalesPlanning')/Dimensions('Region')/Hierarchies('Region')`.
- **Standard HTTP verbs** — `GET` to read, `POST` to create, `PATCH` to update, `DELETE` to remove.

## Prerequisites

To follow along with this documentation you'll need:

- Access to a TM1 / Planning Analytics environment and its hostname.
- A username and password (or other credentials) authorized against it.

## How this documentation is organized

- **Getting Started** *(you are here)* — authentication and your first request.
- **Concepts** — what each object (database, dimension, cube, ...) is and how they relate.
- **API Reference** — endpoint-by-endpoint request/response detail.
- **Tutorials** — task-oriented walkthroughs, like reading or writing cube data.
- **OData** — query options (`$filter`, `$select`, `$expand`, ...) supported across endpoints.
- **Cookbook** — short, copy-pasteable recipes for common operations.

## Next steps

Head to [Authentication](authentication.md) to get a token, then [First Request](first-request.md) to make your first call.
