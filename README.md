# school-directory

[![CI](https://github.com/skolo-saka/school-directory/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/skolo-saka/school-directory/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/skolo-saka/school-directory?sort=semver)](https://github.com/skolo-saka/school-directory/releases)
[![Licence: MIT (software only)](https://img.shields.io/badge/licence-MIT%20(software%20only)-blue)](LICENSE)

A South African school directory for PostgreSQL and Supabase. It provides:

- the database schema;
- a validated importer for the Department of Basic Education (DBE) national school masterlist;
- an audit trail of every import;
- a small public, read-only school search API.

It was built by [Skolo Saka](https://www.skolosaka.co.za) and runs in its private deployment, which refreshes the Skolo Saka directory every day. The refresh runs on UAT first, then production.

| Project | What it is |
| --- | --- |
| **school-directory** (this repository) | Database schema, importer and search API. Python and SQL. |
| [school-search](https://github.com/skolo-saka/school-search) | The React search box that finds a school as you type: [`@skolo-saka/school-search`](https://www.npmjs.com/package/@skolo-saka/school-search) on npm. It gets its results from a `searchSchools` callback, which can call this directory's search API. |
| [Live demo](https://skolo-saka.github.io/school-search/) | The search box searching the real Skolo Saka directory through the website's public API. |

**Software: MIT. Source data: separate terms.** The MIT licence covers this repository's code, SQL and documentation. It does not cover DBE spreadsheets or school records; see [DATA_SOURCES.md](DATA_SOURCES.md). This repository contains no production records, official spreadsheets or credentials, and contributions must not add any.

## Quick start (no database, no real data)

You need Python 3.11 or later. The importer uses only the standard library, so there is nothing to install.

```sh
git clone https://github.com/skolo-saka/school-directory.git
cd school-directory
python -m unittest discover -s tests -v      # importer and rollout tests, fictional data only
python school_directory.py --help
```

The search API test needs Node.js 22 or later:

```sh
node --test tests/search-api.test.mjs
```

### Database tests

The SQL tests need an empty, throwaway PostgreSQL database. With Docker, run the same steps as CI:

```sh
docker run --rm -d --name school-directory-test -e POSTGRES_PASSWORD=local-test-only -e POSTGRES_DB=school_directory_test -p 55432:5432 postgres:17
export PGHOST=localhost PGPORT=55432 PGUSER=postgres PGPASSWORD=local-test-only PGDATABASE=school_directory_test
psql -v ON_ERROR_STOP=1 -c 'create role anon; create role authenticated; create role service_role bypassrls;'
psql -v ON_ERROR_STOP=1 -f sql/001_school_directory.sql
psql -v ON_ERROR_STOP=1 -f sql/002_school_search.sql
psql -v ON_ERROR_STOP=1 -f tests/integration.sql
docker stop school-directory-test
```

`tests/integration.sql` refuses to run against a database that already has schools. Never point it at a real database.

### Supported versions

| Component | Supported |
| --- | --- |
| Python | 3.11, 3.12, 3.13 (CI tests 3.11 and 3.13) |
| PostgreSQL | 15 or later, with the `pg_trgm` extension (CI tests 17) |
| Node.js (search API handler) | 22 or later (CI tests 24) |
| Releases | The latest `0.x` release; see [SECURITY.md](SECURITY.md) |

## Try the importer against the real source

```sh
python school_directory.py
```

This discovers the newest national file listed on DBE's [EMIS Downloads](https://www.education.gov.za/Programmes/EMIS/EMISDownloads.aspx) page, downloads it and validates it. It prints **counts and provenance only** and doesn't contact any database.

It fails closed. Any of the following stops it:
- an unexpected layout;
- duplicate EMIS numbers or invalid fields;
- partial national coverage;
- an unknown phase;
- macros or formulas in the file.

Don't commit the downloaded file or paste its rows into issues.

## Database setup

Apply `sql/001_school_directory.sql`, then `sql/002_school_search.sql`, as your database owner through your normal migration process.

- **Empty database:** they create a minimal `public.schools` table.
- **Existing directory (such as Skolo Saka's):** they reuse the existing table and enum without replacing them.

An empty database needs an explicit, reviewed bootstrap. Scheduled jobs never fill one blindly.

**Private tables (schema `school_directory`):** these hold source versions, import runs, staging records, last-seen source fields and before/after changes.
- They aren't exposed through the public Data API.
- The import functions use invoker security, and only the owner and Supabase's `service_role` can call them.

**Public table (`public.schools`):** read-only through row-level security. Nothing in this module grants public write access.

Set `DATABASE_URL` privately in the environment, then:

```sh
python school_directory.py --database       # preview: school rows stay unchanged
python school_directory.py --apply          # preview, check the thresholds, then apply atomically
```

**Your own empty database:** inspect `--database --bootstrap` first, then use `--apply --bootstrap` for that reviewed first import. Bootstrap can't update a directory that already has schools, and the scheduled job never uses it.

**Credentials:**
- The connection string reaches `psql` through environment fields. It is never a process argument or a log line.
- Remote connections require TLS.
- Don't commit environment files, staging SQL, spreadsheets or database dumps.

To use an official file you've already downloaded:

```sh
python school_directory.py --file private/national.xlsx --release 2025-Q3 --source-url 'https://www.education.gov.za/LinkClick.aspx?...' --database
```

## What an update can do

- **Existing schools:** matched by their unique EMIS number, so their UUID and relationships stay intact.
- **New open schools:** inserted with new IDs and `verified=false`.
- **Source-owned fields:** the name, phase and location fields are updated. Verification flags are preserved.
- **Missing source values:** a non-empty local location is kept when the source has a missing or unknown value.
- **Local edits:** verification flags and local edits detected against the last applied snapshot are preserved. On the first import, existing differences are protected until ownership is established.
- **Missing schools, closures and EMIS successor conflicts:** recorded for review. Schools are never deleted, and an existing canonical EMIS is never remapped because of a historical alias.
- **Rejected sources:**
  - an older source release;
  - incomplete national coverage;
  - more than 10% new or changed records (with a floor of 100);
  - more than 5% missing records (with a floor of 25).
- **Applying:** the preview fingerprint is rechecked under a database lock first. All live writes, observations and audit records commit or roll back together.
- **Unchanged source:** a previously applied checksum and period isn't applied again.

**Exit codes:**

| Exit code | Meaning |
| --- | --- |
| `0` | A successful preview, application or unchanged check. |
| `1` | Validation failed. Nothing was changed. |
| `2` | Review is required. The run's private staging data is kept. |

Treat a review as a real operational outcome. Resolve it; don't disable validation or label the database "up to date". [OPERATIONS.md](OPERATIONS.md) explains how.

## Scheduling and operations

Use the [GitLab example](examples/gitlab-school-directory.yml) and the [UAT-then-production runner](examples/sync_environments.py) in a **private** deployment repository. Never expose deployment credentials to public pull requests.

[OPERATIONS.md](OPERATIONS.md) covers the rest:
- the schedule and monitoring;
- failure and review handling;
- staging retention;
- school corrections;
- source coverage;
- API limits;
- the permissions needed to redistribute data.

## Public search API

[API.md](API.md) has the public response contract and its permissions.

- **`school_search_api.mjs`:** a dependency-free Request/Response handler.
- **`supabase/functions/school-directory-search/index.ts`:** a Supabase Edge Function adapter. It uses the anonymous database role and the public search RPC.

The browser never receives database administrator credentials. The in-memory rate limit is per runtime instance; it isn't a global quota or an availability guarantee.

To use the [school-search](https://github.com/skolo-saka/school-search) component with this API, pass a `searchSchools` function that calls your deployed endpoint:

```ts
const searchSchools = async (q: string, { signal }: { signal: AbortSignal }) => {
  const res = await fetch(`https://<your-project>.supabase.co/functions/v1/school-directory-search?q=${encodeURIComponent(q)}`, { signal });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return (await res.json()).items;
};
```

## Contributing

Contributions are welcome from anyone with a GitHub account. You don't need production access, and nothing you contribute writes to a real database.

- [CONTRIBUTING.md](CONTRIBUTING.md): how to fork, test with fictional data and open a pull request.
- [Good first issues](https://github.com/skolo-saka/school-directory/labels/good%20first%20issue): good places to start.
- [Report a school-data correction](https://github.com/skolo-saka/school-directory/issues/new?template=school_data_correction.yml).
- [MAINTAINERS.md](MAINTAINERS.md): who reviews and releases.
- [RELEASING.md](RELEASING.md): how a release reaches the Skolo Saka website.
- [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) and [SECURITY.md](SECURITY.md).
