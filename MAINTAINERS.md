# Maintainers

| Maintainer | Responsibilities |
| --- | --- |
| [@Mahlatse32](https://github.com/Mahlatse32) | Review, releases, the Skolo Saka deployment and its data operations |
| [@EmmanuelTshehla](https://github.com/EmmanuelTshehla) | Review and releases |

## What maintainers do

- **Triage:**
  - label new issues within about a week;
  - mark well-scoped tasks `good first issue` or `help wanted`;
  - remove personal information from reports. Edit it out, or close the issue and ask for a new one without it.
- **Review:**
  - every change needs one approval from a maintainer who isn't its author;
  - check tests, fictional data only, additive SQL, and that no credentials or real records are added;
  - changes to import rules or thresholds also need the agreement of whoever runs the Skolo Saka deployment.
- **School-data corrections:** check each report against the official source, and say whether the fix belongs with DBE or with a deployment ([OPERATIONS.md](OPERATIONS.md#school-corrections)).
- **Releases:** follow [RELEASING.md](RELEASING.md). Only repository administrators can create `v*` tags.
- **Security:** acknowledge private vulnerability reports within 5 working days ([SECURITY.md](SECURITY.md)).
- **Deployment:** the Skolo Saka website maintainers watch the daily refresh and handle review-required runs ([OPERATIONS.md](OPERATIONS.md)). Contributors never need deployment access.

Regular contributors who review well may be invited to become maintainers. Access is granted by the organisation's owners.
