# Contributing

Thanks for helping. We welcome:
- importer fixes and tests;
- SQL improvements;
- documentation;
- school-data correction reports.

By taking part you agree to follow the [code of conduct](CODE_OF_CONDUCT.md). Report security problems privately, as [SECURITY.md](SECURITY.md) describes, not in issues.

You don't need any access to Skolo Saka's systems. Everything here runs on your machine with **fictional** data. Nothing you open, push or merge writes to a real database.

## Before you start

- **Bugs:** [open an issue](https://github.com/skolo-saka/school-directory/issues/new/choose) with the bug form first, unless the fix is small and obvious.
- **Behaviour changes:** for anything that changes what an import does (thresholds, matching, field ownership, phase mapping), open a feature issue and agree the approach first. These rules protect real school records.
- **Wrong school data:** use the [school-data correction form](https://github.com/skolo-saka/school-directory/issues/new?template=school_data_correction.yml). Don't open a pull request with real records.
- **Where to start:** issues labelled [`good first issue`](https://github.com/skolo-saka/school-directory/labels/good%20first%20issue) or [`help wanted`](https://github.com/skolo-saka/school-directory/labels/help%20wanted). Comment on one to say you're taking it.

## Set up

You need Git and Python 3.11 or later. Node.js 22 or later is only needed for the search API test. PostgreSQL 15 or later (Docker is easiest) is only needed for the SQL tests.

```sh
# Fork skolo-saka/school-directory on GitHub first
git clone https://github.com/<your-username>/school-directory.git
cd school-directory
git remote add upstream https://github.com/skolo-saka/school-directory.git
python -m unittest discover -s tests -v
```

## Make a change

1. Branch from an up-to-date `main`:

   ```sh
   git fetch upstream
   git switch -c fix/short-description upstream/main
   ```

   Use a `fix/`, `feature/`, `docs/` or `test/` prefix.
2. Keep the change focused on one thing, and add or update a test for any behaviour you change:
   - **Importer and rollout:** `tests/test_importer.py` and `tests/test_rollout.py`. Build fictional rows like the existing `HEADERS`/`ROW` fixtures. Use EMIS numbers in the `9000xxxxx` range and `example.invalid` addresses.
   - **SQL:** `tests/integration.sql`. It runs only against an empty, throwaway database.
   - **Search API:** `tests/search-api.test.mjs`.
3. Run the same checks as CI. The database steps are in the [README](README.md#database-tests).

   ```sh
   python -m unittest discover -s tests -v
   python -m py_compile school_directory.py
   node --test tests/search-api.test.mjs
   ```

4. **SQL changes:**
   - Keep them additive, and safe to apply to a database that already has a directory: `create ... if not exists`, `create or replace function`, no dropped columns or data.
   - Say in the pull request whether existing deployments need a new migration.
5. **Real data:** never commit or paste DBE spreadsheets, production records, previews, staging SQL, database dumps, connection strings or anyone's personal information. Tests, issues and screenshots use fictional schools only.

## Pull requests

- Describe what changed and why, and link the issue (`Fixes #12`).
- Fill in the pull-request checklist.
- All CI checks must pass on the latest commit, and a maintainer other than the author must approve. A new commit dismisses an earlier approval, and all review conversations must be resolved. `main` accepts no direct pushes, including from administrators.
- Pull requests from forks run CI with read-only permissions and no secrets.
- Leave the version headings in `CHANGELOG.md` alone; maintainers set them when they release ([RELEASING.md](RELEASING.md)). Add a line under `Unreleased` for a user-visible change.

## How a merged change reaches Skolo Saka

**Merging to `main` changes no real database.**
1. Maintainers publish a versioned release from `main`.
2. Skolo Saka's private website project then proposes a merge request that vendors that exact release, with its tests and any required database migration.
3. That merge request is reviewed again and validated on UAT, and goes to production through the website's normal release.

The daily school-data refresh is separate. It uses whichever importer version is already deployed.

## Licence

Code contributions are accepted under this project's [MIT licence](LICENSE). Source-data contributions need their own documented reuse rights; see [DATA_SOURCES.md](DATA_SOURCES.md).
