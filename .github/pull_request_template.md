## What and why

<!-- What does this change, and which issue does it address? e.g. Fixes #12 -->

## Effect on deployments

<!-- Does this change what an import inserts, updates, protects or rejects? Does an existing database need a new migration? Write "None" if neither. -->

## How I tested it

- [ ] `python -m unittest discover -s tests -v` passes
- [ ] `node --test tests/search-api.test.mjs` passes (if the API changed)
- [ ] SQL applied to an empty PostgreSQL database and `tests/integration.sql` passes (if SQL changed)
- [ ] Tests added or updated for changed behaviour
- [ ] SQL changes are additive and safe for a database that already has a directory
- [ ] Fictional data only: no DBE spreadsheets, production records, credentials or personal information
- [ ] `CHANGELOG.md` has a line under `Unreleased` (for user-visible changes)
