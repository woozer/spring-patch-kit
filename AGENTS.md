# Agent Instructions

These instructions apply to the entire Spring patch-kit repository.

Before changing a project definition or patch series, read completely:

- `README.md`
- `docs/new-cve-runbook.md`
- the affected upstream repository's `AGENTS.md` and `CONTRIBUTING.md`

Mandatory rules:

1. Pin every project to an exact public upstream base tag. Do not track a
   moving branch as a release input.
2. Confirm the affected Spring project before editing; Spring Framework,
   Integration, Security, Boot, Data, and Cloud are separate projects.
3. Prefer an authoritative public upstream fix. Record its full commit hash in
   the private backport commit message.
4. Keep one minimal production fix plus its regression tests in one atomic
   commit per CVE. Include the CVE identifier in the subject. Never combine
   multiple CVEs in one commit.
5. Keep private-version and documentation changes in separate commits.
6. Private releases are immutable and cumulative. Increment `cve.N`; never
   modify or republish an existing private version and never use an official
   Spring version number for a private build.
7. Follow the upstream project's code style, DCO/sign-off, author, and test
   requirements. Preserve upstream authorship where appropriate.
8. Add focused regression tests, then run the registered focused tests and
   build. Prove the exported series in a fresh workspace with `patchctl apply`,
   `verify`, `test`, and `build`.
9. Update `project.conf`, including the cumulative version, CVE list, and test
   tasks. Capture the complete linear series from the pinned base.
10. Do not commit credentials, tokens, private keys, internal repository URLs,
    local absolute paths, Gradle caches, compiled artifacts, or dependency
    caches. Scan every exported patch series before committing it.
11. Preserve Windows Git Bash compatibility. Keep scripts, configurations,
    Markdown, and mail patches on LF line endings and avoid dependencies on
    Python, Maven, `jq`, or a system patch utility.
12. Do not push, publish artifacts, create releases, or change remote state
    without explicit user authorization. Publishing credentials must remain
    outside this repository and agent context.
13. This environment requires manual Artifactory upload. `patchctl` may prepare
    Maven-local publications but must not connect to Artifactory or handle its
    URL or credentials. Document and verify the human upload from a clean
    consumer environment.

When a security fix cannot be backported confidently from authoritative source,
stop and report the missing provenance or technical blocker instead of
inventing a fix.
