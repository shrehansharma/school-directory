# Releasing

Releases are versioned Git tags on `main` with a GitHub release. They aren't published to a package registry: deployments vendor a specific release.

## Versions

This project follows [semantic versioning](https://semver.org/). While it's 0.x:

| Type | Example | Use it for |
| --- | --- | --- |
| **Patch** | 0.1.0 → 0.1.1 | Fixes, tests and documentation. Import results and the SQL interface don't change. |
| **Minor** | 0.1.x → 0.2.0 | Anything a deployment must act on: new or changed SQL (it needs a migration), changed import rules or thresholds, a changed API response or CLI option. |
| **Major** | 1.0.0 | Reserved for when the schema and import behaviour are declared stable. |

A version is never reused, and a tag is never moved.

## Release a version

1. **Open a release pull request.** It moves the `Unreleased` entries in `CHANGELOG.md` under the new version and date, and says whether deployments need a database migration. Merge it after review.
2. **Tag the merge commit** on `main` and push the tag. Only repository administrators can create `v*` tags.

   ```sh
   git switch main && git pull --ff-only
   git tag -a v0.2.0 -m "v0.2.0"
   git push origin v0.2.0
   ```

3. **The [Release workflow](.github/workflows/release.yml) runs.** It:
   - checks that the tag is on `main` and has a matching `CHANGELOG.md` section;
   - runs the same tests as CI, including PostgreSQL;
   - creates the GitHub release with that section as its notes.

If the workflow fails before it creates the release, fix the problem through a pull request, delete the tag and tag the new merge commit.

## How a release reaches the Skolo Saka website

The website's private GitLab project has an updater job, `school-directory-upstream`, that polls this repository's releases. When it finds a newer release, it:

1. **Downloads that release's files.** Only these, at the tagged commit:
   - `school_directory.py`;
   - `LICENSE`;
   - `examples/sync_environments.py`;
   - `tests/*.py`;
   - `sql/*.sql`.

   It runs none of them.
2. **Records the tag, commit and file checksums** in `infra/school-directory/UPSTREAM.json`.
3. **Opens a merge request into the website's `staging` branch.** It never merges one itself.

That merge request runs the importer tests with no database credentials, and a website maintainer reviews it:
- **If `sql/` changed:** a check stays red until a maintainer adds a reviewed, additive Supabase migration that records the new SQL checksum.
- **Once merged:** it's validated on UAT and reaches production through the website's normal `staging` → `main` release.

The daily data refresh keeps using whichever importer is on the website's `main`. Publishing a release never changes a database by itself.
