# Tablio docs — agent instructions

Repo: [tablio-hr/docs](https://github.com/tablio-hr/docs).

- Land work on **`develop`** with a direct commit. Do not open a feature PR.
- The only PR is **Promote to production** (`develop` → `main`). That PR is
  the CI gate. Do not commit to `main`.
- After a promote merge, delete the promote branch (local + remote). Never
  delete `develop` or `main`.
- An ADR locks a long-lived boundary. Do not re-open an accepted ADR in an API change.
- Implementation plans authorize API work. API work starts only after the plan is merged.
- Runner and release contract: see [tablio-hr/api AGENTS.md](https://github.com/tablio-hr/api/blob/develop/AGENTS.md).
