# Operating the school directory

This guide is for whoever runs the importer against a real database: Skolo Saka's website maintainers, or you in your own deployment. Contributors don't need any of it; they work with fictional data only (see [CONTRIBUTING.md](CONTRIBUTING.md)).

## How Skolo Saka runs it

| | |
| --- | --- |
| **Where** | The private GitLab project behind https://www.skolosaka.co.za. The website vendors a reviewed release of this repository under `infra/school-directory`, with its MIT notice. |
| **When** | Daily at 05:17 (Africa/Johannesburg), from a pipeline schedule on the protected `main` branch, with the input `pipeline-task=school-directory`. The job runs alone: it doesn't build or deploy the website. |
| **Order** | `examples/sync_environments.py` downloads and validates one official file, then applies it to **UAT**. Production receives the same file and checksum only if UAT ends `applied` or `unchanged`. |
| **Credentials** | `UAT_DATABASE_URL` and `PROD_DATABASE_URL` are protected, masked CI/CD variables in the private project. Both are required and must differ. They never appear in this repository, its workflows or pull requests. |
| **Artifacts** | None. The downloaded source, the staging SQL and the previews aren't uploaded. |

**Last verified on 21 November 2026:** a scheduled run left UAT `unchanged` and production `applied`:
- **Release:** 2025-Q3, checksum `bd6f9588c4b1…`.
- **Supported records:** 25,511.
- **Changes:** 0 inserts and 0 updates.
- **Missing:** 16 schools that are in the database but not in the source; they were kept.
- **Excluded:** 16 stand-alone ECD centres.

## Freshness: four separate facts

`school_directory.sources` keeps four facts in separate columns. Don't merge them in reports or on the website.

| Column | Meaning |
| --- | --- |
| `dataset_release` | The period DBE printed for the newest national file it **lists**, e.g. `2025-Q3`. It isn't the date the data was checked. |
| `source_sha256` | Checksum of the exact file that was validated. |
| `last_checked_at` | When the job last downloaded and validated a file. A daily check moves this forward even when DBE has published nothing new. |
| `last_successful_sync_at` | When the database last ended a run `applied`. It doesn't move on `unchanged`, failed or review-required runs. |

A successful daily check doesn't mean a newer dataset exists. On 21 November 2026, DBE's newest listed national release was 2025-Q3.

## Outcomes and who acts on them

**Who monitors:** at Skolo Saka, the website maintainers. Failed scheduled pipelines are reported through GitLab's normal pipeline-failure notifications, which go to the schedule's owner and to anyone watching the project. No other alerting is configured, and nothing emails schools or the public.

| Exit | Status | What happened | What to do |
| --- | --- | --- | --- |
| 0 | `unchanged` | The same file and period were already applied. | Nothing. |
| 0 | `applied` | Changes passed every check and were written. | Read the change counts in the job log. |
| 2 | `review_required` | Something needs a person: too many changes, missing schools, a closure or an EMIS successor conflict. Production wasn't touched. | Investigate it (below), the same working day where possible. |
| 1 | failed | The download, layout, coverage or validation failed, or the database was unreachable. Nothing was written. | Read the error line. Retry once if it was a network failure, otherwise investigate. |

### Investigate a review or failure

1. **Open the failed job log.** It prints one JSON report per environment, holding:
   - `run_id`;
   - `status`;
   - the counts (`insert`, `update`, `missing`, `review`, `protected`);
   - `reasons`.

   It never prints connection strings.
2. **Inspect the run as an authorised database operator** (read-only SQL):

   ```sql
   select status, report from school_directory.import_runs where id = '<run_id>';
   select emis_number, fields from school_directory.staged_schools where run_id = '<run_id>' limit 50;
   ```

3. **Decide which kind of problem it is:**
   - **A real source change** (a new quarterly release with many updates): confirm it against DBE's published file, then apply it deliberately. Run `school_directory.py --database` from an operator machine with the intended `DATABASE_URL`, check the preview, then `--apply`. Do this on UAT first, then production.
   - **A source problem** (partial file, wrong layout, unexpected closures): don't apply. Wait for DBE's correction, or ask them, and leave the database as it is.
   - **An identity conflict** (a reused or successor EMIS number): resolve it through a reviewed database change, then run a fresh preview.
4. **Never relax validation to clear a review.** Don't raise thresholds or label the directory "up to date" without a reviewed change to this repository.

## Staging records

- **Applied runs:** their staged rows are deleted in the same transaction.
- **Review-required and failed runs:** their rows stay in `school_directory.staged_schools` for diagnosis.
- **Retention:** keep them only as long as you need them. Clear a resolved run with:

  ```sql
  delete from school_directory.staged_schools where run_id = '<run_id>';
  ```

  `import_runs` and `changes` are the audit trail; keep them.
- **Retention period:** choose one for your deployment and write it down. On 21 November 2026, Skolo Saka's UAT and production databases held no retained staging rows.

## School corrections

**Reporting:**
- Anyone can report a wrong name, place, level or a missing school with the [school-data correction form](https://github.com/skolo-saka/school-directory/issues/new?template=school_data_correction.yml).
- Include the EMIS number and a public official reference.
- Never include learners', parents', donors' or staff members' personal information.

**Review:**
- A maintainer checks the report against the official source.
- If DBE's data is wrong, the fix belongs with DBE. The next DBE release then reaches the directory automatically.
- If the deployment's local record is wrong, a website maintainer corrects it in that deployment. The importer then protects the local edit.
- Verification (`verified=true`) is a Skolo Saka decision, separate from the official source. Importing never verifies a school.

## Source coverage and exclusions

**Included:** ordinary and special schools from DBE's national masterlist, in all nine provinces.

**Excluded or not imported:**
- **Stand-alone ECD centres:** excluded. They're counted and existing records are kept.
- **Pending openings:** observed, but not inserted as active schools.
- **Closed and missing schools:** kept and recorded for review. They're never deleted.
- **Other masterlist columns:** ignored, including principals, contacts, enrolment and coordinates.

**Not covered:** LURITS and provincial lists aren't used.

**Phase mapping:** see [DATA_SOURCES.md](DATA_SOURCES.md).

## API limits and availability

**Skolo Saka's public search endpoint:**
- `https://www.skolosaka.co.za/api/public/schools/search?q=`
- 60 searches per minute per client address, at most 10 results.
- Public fields only. CORS allows only the GitHub Pages demo.

**This repository's portable and Supabase handlers:**
- 120 searches per minute per runtime instance, at most 6 results, and a 3-second timeout.

All limits are per instance. They aren't global quotas and could change without notice. **There is no service-level agreement:** availability is best effort. Don't build anything that needs guaranteed uptime on Skolo Saka's endpoint; deploy your own.

## Redistributing the data

The MIT licence covers this software only. DBE's general terms reserve copyright, and this project hasn't established a licence that allows unrestricted redistribution of the dataset. Before you do any of the following, establish the rights you need with the data owner:

- publish a bulk export;
- mirror the data;
- run a public API over the data for others.

Neither this repository nor its CI publishes real school records.
