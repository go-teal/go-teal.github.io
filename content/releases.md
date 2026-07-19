---
title: "Releases"
description: "What's new in Teal — highlights of recent releases, with links to the full notes and changelog on GitHub."
---

Teal ships often. This page highlights what's new; the full, auto-generated notes
for every version live on [GitHub Releases](https://github.com/go-teal/teal/releases),
and the complete history is in the
[CHANGELOG](https://github.com/go-teal/teal/blob/main/CHANGELOG.md).

Install or upgrade to the latest:

```bash
go install github.com/go-teal/teal/cmd/teal@latest
```

## Debug UI dashboard

The `teal ui` dashboard gained a set of asset actions and canvas improvements:

- **Data Preview** — read a table's contents (`SELECT * FROM`) straight into the
  Data panel, with pagination, independent of the model's transformation SQL
  ([v1.2.10](https://github.com/go-teal/teal/releases/tag/v1.2.10))
- **Drop & Truncate persisted data** — drop the materialized table/view or empty
  a table, right from the asset panel, with a confirmation dialog
  ([v1.2.8](https://github.com/go-teal/teal/releases/tag/v1.2.8))
- **Asset action toolbar & drag-to-pan** — Run Select, Run Mutation, Truncate and
  Delete moved into an icon toolbar under the asset name; pan the DAG canvas by
  holding the mouse button
  ([v1.2.9](https://github.com/go-teal/teal/releases/tag/v1.2.9))
- **Data panel follows the latest write** — select results are no longer hidden
  behind a prior mutation
  ([v1.2.8](https://github.com/go-teal/teal/releases/tag/v1.2.8))

## Correctness & driver fixes

- **SQL column order preserved** in select/test data responses via a `columns`
  field ([v1.2.2](https://github.com/go-teal/teal/releases/tag/v1.2.2))
- **Postgres `int4` / `int8`** columns no longer degrade to null in DataFrames
  ([v1.2.3](https://github.com/go-teal/teal/releases/tag/v1.2.3))
- **Preview for non-SELECT models** — select falls back to reading the
  materialized relation for SCD2 / custom scripts
  ([v1.2.4](https://github.com/go-teal/teal/releases/tag/v1.2.4))
- **Correct materialization icons** — the gopher marks raw Go assets only; custom
  materialization is a SQL model
  ([v1.2.6](https://github.com/go-teal/teal/releases/tag/v1.2.6))
- **`teal gen` keeps your Makefile** instead of overwriting it
  ([v1.2.7](https://github.com/go-teal/teal/releases/tag/v1.2.7))
- **Dashboard no longer served stale** — the UI bundle is sent with `no-cache`
  ([v1.2.9](https://github.com/go-teal/teal/releases/tag/v1.2.9))

## Build & maintenance

- **Dependencies updated** to their latest versions — gin, pgx/v5, zerolog,
  pongo2/v6 and more ([v1.2.0](https://github.com/go-teal/teal/releases/tag/v1.2.0))
- **Slim default build** — `pkg/ui` is gated behind the `teal_ui` build tag, and
  `teal ui` spawns the project UI process with it
  ([v1.1.1](https://github.com/go-teal/teal/releases/tag/v1.1.1),
  [v1.2.1](https://github.com/go-teal/teal/releases/tag/v1.2.1))
- **PostgreSQL pipelines work out of the box**
  ([v1.1.0](https://github.com/go-teal/teal/releases/tag/v1.1.0))

## All releases

| Version | Highlight |
| --- | --- |
| [v1.2.10](https://github.com/go-teal/teal/releases/tag/v1.2.10) | Data Preview |
| [v1.2.9](https://github.com/go-teal/teal/releases/tag/v1.2.9) | Action toolbar, drag-to-pan, cache fix |
| [v1.2.8](https://github.com/go-teal/teal/releases/tag/v1.2.8) | Drop / Truncate, Data panel fix |
| [v1.2.7](https://github.com/go-teal/teal/releases/tag/v1.2.7) | `teal gen` keeps existing Makefile |
| [v1.2.6](https://github.com/go-teal/teal/releases/tag/v1.2.6) | Materialization icons |
| [v1.2.5](https://github.com/go-teal/teal/releases/tag/v1.2.5) | Columns in SQL order (UI) |
| [v1.2.4](https://github.com/go-teal/teal/releases/tag/v1.2.4) | Preview for non-SELECT models |
| [v1.2.3](https://github.com/go-teal/teal/releases/tag/v1.2.3) | Postgres int4/int8 fix |
| [v1.2.2](https://github.com/go-teal/teal/releases/tag/v1.2.2) | SQL column order |
| [v1.2.1](https://github.com/go-teal/teal/releases/tag/v1.2.1) | `teal_ui` tag when spawning UI |
| [v1.2.0](https://github.com/go-teal/teal/releases/tag/v1.2.0) | Dependency updates |
| [v1.1.1](https://github.com/go-teal/teal/releases/tag/v1.1.1) | Slim default build (`teal_ui` tag) |
| [v1.1.0](https://github.com/go-teal/teal/releases/tag/v1.1.0) | PostgreSQL out of the box |

For older versions, see the full
[CHANGELOG](https://github.com/go-teal/teal/blob/main/CHANGELOG.md).
