# CVE Backport Release Candidate Evidence — 2026-09-03

This record covers the source and selective-artifact release candidate in this
repository. It is not proof of deployment: the manual Artifactory upload and
clean application resolution remain approval steps in the target environment.

## Source identity

| Project | Upstream base | Private version | CVE commits | Backport branch head |
| --- | --- | --- | ---: | --- |
| Spring Framework | `v6.2.19` (`6214eae8bd02c2ed7ab382bb8d16a9cc6de49522`) | `6.2.19-cve.2` | 10 | `35bbec1fecbe4d84be10bacfba825c92405676b5` |
| Spring Integration | `v6.5.10` (`5624e08f74a436c5709db81888f552a19ed779d1`) | `6.5.10-cve.2` | 11 | `e636308a04d715662ed22d8185b8ce446e793722` |
| Spring Security | `6.5.11` (`73b077790fcb04ac3712033d3e939daf42264545`) | `6.5.11-cve1` | 2 | `bcb6bdc09b723f13f4c6f16c11d449ea29d2623d` |

All 23 CVEs have one distinct `Backport fix for CVE-...` commit containing the
production change and its focused regression tests. Every CVE mail patch has a
`Signed-off-by` trailer. Public-fix hashes and artifact coverage are recorded
in [the coverage registry](cve-2026-coverage.md).

## Verification performed

The following checks completed on macOS with Temurin JDK 21.0.6. Clean replay
used local `file://` mirrors of the exact upstream repositories, which exercises
the same clone-from-tag and `git am --3way` path as a network replay.

- `patchctl apply all`: all mail patches applied from the three pinned tags and
  all versions, branches, clean worktrees, and expected CVE subjects verified.
- `patchctl test all`: all registered Framework, Integration, and Security test
  gates passed.
- `patchctl build all`: the complete Spring Framework build and the seven
  affected Spring Integration module builds passed. The final clean Spring
  Security build passed 168 tasks, including unit, ApacheDS configuration,
  UnboundID LDAP, style, Javadoc, and assembly tasks.
- `patchctl stage-selective all`: 18 registered coordinates produced 72 files
  (main, sources, Javadoc, and POM for each coordinate). All SHA-256 checksums
  verified, all selective POM checks passed, and no Gradle `.module` metadata
  was staged.
- The selective Maven consumer example resolved successfully from an empty
  local Maven repository using the staged artifacts plus Maven Central. Its
  dependency tree selected the intended private Framework, Integration, and
  Security coordinates. The temporary file repository did not provide Maven
  sidecar checksum files; the aggregate `SHA256SUMS` verification passed, and
  Artifactory remains responsible for repository-side checksum metadata.
- `patchctl doctor`: accepted JDK 21, rejected JDK 25, rejected a non-HTTPS
  Artifactory URL, and rejected incomplete username/password configuration.
- Bash syntax, XML syntax, and Git whitespace checks passed. `shellcheck` was
  not installed in the verification environment.

The Spring Integration root-wide build also exercised unrelated mail and
ZeroMQ modules and encountered failures outside the patched/staged modules.
The deterministic release gate intentionally builds and tests all seven
affected modules instead; those gates passed.

## Target-environment approval still required

- Re-run `doctor`, `apply`, `verify`, `test`, `build`, and `stage-selective` on
  the approved Windows build host using its corporate Artifactory mirror.
- Retain that host's generated `SHA256SUMS`; do not assume independently built
  JARs have byte-identical ZIP timestamps.
- Manually upload the staged `org/` tree without `SHA256SUMS`, preserving every
  Maven path. Record uploader, time, approved target repository, and stored
  checksums.
- From an empty Maven local repository, resolve through Artifactory with the
  checked-in settings template, inspect the dependency tree, and run the
  consuming application's smoke and integration tests.
- Attach the scanner result and, when a version-only scanner still identifies
  the public base version, provide the source commits, coverage registry,
  checksums, resolved graph, and application test evidence for VEX/review.
