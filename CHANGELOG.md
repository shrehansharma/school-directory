# Changelog

All notable changes are listed here. Versions follow [semantic versioning](https://semver.org/); see [RELEASING.md](RELEASING.md).

## Unreleased

- Contributor documentation: quick start, supported versions, operations guide, maintainers, release process, issue forms and pull-request template.
- A release workflow that tests a version tag and creates its GitHub release.

## 0.1.0 (2026-11-21)

First versioned release. This is the code the Skolo Saka website vendors and runs in its daily refresh.

- `school_directory.py`: a validated DBE national masterlist importer with previews, thresholds, atomic apply and an audit trail.
- `sql/001_school_directory.sql` and `sql/002_school_search.sql`: the directory schema, private import tables and the public search RPC.
- `school_search_api.mjs` and a Supabase Edge Function adapter: a public, read-only search with a field allowlist.
- `examples/`: the private GitLab schedule and the runner that applies changes to UAT before production.
