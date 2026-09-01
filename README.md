# Spring Patch Kit

This kit turns privately maintained Spring security backports into repeatable,
reviewable inputs. A project definition pins the upstream repository and base
tag, declares the private version and tests, and stores one ordered Git patch
per atomic change.

The initial registry contains:

| Project definition | Base | Private version | CVEs |
| --- | --- | --- | --- |
| `spring-framework-6.2.19` | `v6.2.19` | `6.2.19-cve.1` | CVE-2026-59313, CVE-2026-59314 |
| `spring-integration-6.5.10` | `v6.5.10` | `6.5.10-cve.1` | CVE-2026-59307, CVE-2026-59311, CVE-2026-59321 |

These are private, source-maintained builds and are not vendor-supported
Spring releases.

## Move to a clean machine

The kit is self-contained and requires no Python, Maven, `jq`, or patch utility.
It uses Bash and Git to clone and apply patches, and each cloned Spring project
provides its own Gradle wrapper. IntelliJ can import either generated repository
as a Gradle project.

Transfer either this Git repository or the generated
`spring-patch-kit.bundle` file to the empty directory. With the bundle:

```bash
git clone spring-patch-kit.bundle spring-patch-kit
cd spring-patch-kit
./patchctl doctor
./patchctl apply all ../patched-spring
./patchctl verify all ../patched-spring
./patchctl test all ../patched-spring
```

The resulting layout is:

```text
empty-directory/
├── spring-patch-kit/
└── patched-spring/
    ├── spring-framework-upstream/
    └── spring-integration-upstream/
```

Open either directory under `patched-spring` directly in IntelliJ. Use the
project's Gradle wrapper when IntelliJ asks which Gradle distribution to use.

The target machine needs network access to GitHub and Gradle/Maven artifact
repositories for the initial clone and dependency download. An actually
offline installation would also need pinned upstream source bundles and a
pre-populated Gradle dependency cache; those large binary inputs are not stored
in this source-only kit.

### Windows Git Bash

Git for Windows supplies Bash, Git, and the standard command-line utilities
used by `patchctl`. The script is part of this repository rather than a global
command, so invoke it explicitly with Bash:

```bash
cd spring-patch-kit

export PATCH_JAVA_HOME="/c/Program Files/Eclipse Adoptium/jdk-21"
bash ./patchctl doctor
bash ./patchctl apply all ../patched-spring
bash ./patchctl verify all ../patched-spring
bash ./patchctl test all ../patched-spring
```

Adjust the Java directory to the installed JDK 21 location. Forward-slash Git
Bash paths such as `/c/Program Files/...` avoid ambiguity with Windows
backslashes. The repository pins scripts, configurations, documentation, and
mail patches to LF line endings so `core.autocrlf` does not make them invalid.

### IntelliJ IDEA workflow on Windows

Create and verify the patched repositories from Git Bash before opening
IntelliJ:

```bash
cd spring-patch-kit
export PATCH_JAVA_HOME="/c/Program Files/Eclipse Adoptium/jdk-21"
bash ./patchctl apply all ../patched-spring
bash ./patchctl verify all ../patched-spring
```

Spring Framework and Spring Integration are independent Gradle builds. Open
each one separately with **File > Open** (using separate IntelliJ windows):

```text
patched-spring/spring-framework-upstream
patched-spring/spring-integration-upstream
```

For each project, configure **Settings > Build, Execution, Deployment > Build
Tools > Gradle** as follows:

- Build and run using: `Gradle`
- Run tests using: `Gradle`
- Gradle distribution: `gradle-wrapper.properties` / Wrapper
- Gradle JVM: the installed JDK 21

Reload the Gradle project after changing these settings. IntelliJ obtains the
Gradle version from the repository's wrapper; no system Gradle installation or
Gradle `PATH` entry is needed.

You may navigate, edit, and debug focused tests from IntelliJ. The authoritative
security and release build remains the clean Git Bash workflow:

```bash
bash ./patchctl test all ../patched-spring
bash ./patchctl build all ../patched-spring
bash ./patchctl publish-local all ../patched-spring
```

Do not use IntelliJ's generic **Build Project** output as the release artifact
source and do not upload to Artifactory from IntelliJ. Use the Gradle-wrapper
artifacts produced by `patchctl`, followed by the documented manual Artifactory
upload and clean-consumer validation.

## Recreate everything

Choose a new workspace. Existing target repositories are never overwritten:

```bash
./patchctl apply all /path/to/replayed-spring
./patchctl verify all /path/to/replayed-spring
```

Run all focused regression tests. On macOS, Java 21 is selected for these
commands only; the system Java default is unchanged:

```bash
./patchctl test all /path/to/replayed-spring
```

On other systems, or to select a specific JDK explicitly:

```bash
PATCH_JAVA_HOME=/path/to/jdk-21 \
  ./patchctl test all /path/to/replayed-spring
```

Build or publish both complete custom artifact sets to Maven Local:

```bash
./patchctl build all /path/to/replayed-spring
./patchctl publish-local all /path/to/replayed-spring
```

## Manual Artifactory upload

This environment requires a manual Artifactory upload. `patchctl` deliberately
does not connect or authenticate to Artifactory. The `publish-local` command
prepares the complete Maven publications under the current user's local Maven
repository:

```text
~/.m2/repository/org/springframework/
```

Upload both complete private version sets to the approved Artifactory Maven
repository while preserving their Maven coordinates and directory layout:

```text
org.springframework:*:6.2.19-cve.1
org.springframework.integration:*:6.5.10-cve.1
```

Upload the generated POMs and required JARs (including BOM POMs and any source
or Javadoc artifacts required by local policy). Do not upload Maven-local
bookkeeping files such as `_remote.repositories` or `maven-metadata-local.xml`;
Artifactory manages its own repository metadata.

Enter Artifactory credentials only in the approved Artifactory interface. Do
not place its URL, username, password, access token, or API key in this
repository, shell history, patch files, or build logs.

After the manual upload, validate from a clean application environment that has
no matching artifacts in Maven Local. Resolve both private versions from
Artifactory, inspect the dependency graph, and run the application smoke and
integration tests. Record the uploader, date, target repository, artifact
checksums, and validation result in the CVE release evidence.

Commands can target one project instead of `all`:

```bash
./patchctl list
./patchctl test spring-framework-6.2.19 /path/to/replayed-spring
```

## Add another CVE

Follow the complete [new CVE backport runbook](docs/new-cve-runbook.md). It
includes the release checklist, immutable versioning rules, Windows Git Bash
commands, clean-room replay, and evidence requirements. Start an audit record
from [the CVE record template](docs/cve-record-template.md).

1. Start from the pinned base tag in a clean clone.
2. Create or reuse the project's backport branch.
3. Add one production change and its tests in one commit per CVE. Include the
   CVE identifier in each commit subject.
4. Keep private version and documentation changes in separate commits.
5. Run the narrow module tests and then the project build.
6. Capture the linear commit series into a new directory:

```bash
./patchctl capture spring-framework-6.2.19 \
  /path/to/clean/source-repository \
  /path/to/new-patch-export
```

The capture command refuses to replace an existing directory. Review the
exported patches, then replace the corresponding `projects/<id>/patches`
directory through your normal code-review process.

For a new Spring line, copy an existing `project.conf`, change its pinned base,
private version, branch, CVE list, and Gradle test arguments, and add its patch
series.

## CI and artifact repository workflow

A production pipeline should run these stages:

```text
apply -> verify -> test -> build -> publish-local -> manual Artifactory upload
      -> clean dependency resolution -> application smoke test
```

Use immutable versions such as `6.2.19-cve.1`; never publish private artifacts
under Spring's official version numbers. Publish to an internal Nexus or
Artifactory repository, generate checksums and an SBOM, and retain:

- upstream advisory and fix references;
- the atomic patch commits;
- test and build logs;
- artifact checksums and provenance;
- VEX statements or scanner exceptions for version-only findings.

Credentials and internal repository URLs intentionally do not live in this
kit. In this environment, a human supplies them only through the approved
manual Artifactory upload process.
