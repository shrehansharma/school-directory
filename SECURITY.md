# Security policy

## Reporting a vulnerability

Please report security problems privately. Don't open a public issue, pull request or discussion for them.

1. Go to the repository's [Security tab](https://github.com/skolo-saka/school-directory/security) and choose **Report a vulnerability** ([direct link](https://github.com/skolo-saka/school-directory/security/advisories/new)). Only you and the maintainers can see the report.
2. Include:
   - what an attacker could do;
   - the affected release or commit;
   - steps to reproduce it, using fictional records.

If you can't use GitHub, email [info@skolosaka.co.za](mailto:info@skolosaka.co.za) with "Security: school-directory" in the subject and no exploit details. A maintainer will reply with a private way to share them.

We aim to acknowledge reports within 5 working days and to agree a fix and disclosure date with you. We credit reporters in the advisory unless you ask us not to. Don't post live credentials, private database contents or exploitable production details anywhere public.

## Supported versions

| Version | Supported |
| --- | --- |
| 0.1.x (latest release) | Yes |
| Older releases | No. Upgrade to the latest release. |

## Scope

**In scope:**
- the importer (`school_directory.py`), including its source-host pinning, file validation and database writes;
- the SQL schema and its permissions;
- the public search handlers (`school_search_api.mjs` and the Supabase adapter).

**Design rules a report may show being broken:**
- The importer runs only in an operator-controlled job, with credentials in protected environment secrets.
- Import metadata is private.
- Every public search response uses an explicit public-field allowlist.

**Out of scope:** someone else's deployment of this code, its credentials or its data. Report those to that operator. Problems on https://www.skolosaka.co.za itself can be reported the same way as above.
