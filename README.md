# Spring Patch Kit

This kit turns privately maintained Spring security backports into repeatable,
reviewable inputs. A project definition pins the upstream repository and base
tag, declares the private version and tests, and stores one ordered Git patch
per atomic change.

The registry contains:

| Project definition | Base | Private version | CVEs |
| --- | --- | --- | --- |
| `spring-framework-6.2.19` | `v6.2.19` | `6.2.19-cve.2` | 10 cumulative fixes |
| `spring-integration-6.5.10` | `v6.5.10` | `6.5.10-cve.2` | 11 cumulative fixes |
| `spring-security-6.5.11` | `6.5.11` | `6.5.11-cve1` | CVE-2026-59270, CVE-2026-59276 |

These are private, source-maintained builds and are not vendor-supported
Spring releases.

See [CVE coverage and artifact mapping](docs/cve-2026-coverage.md) for the
complete list, upstream fix provenance, and scanner-to-artifact caveats.

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
    ├── spring-integration-upstream/
    └── spring-security-upstream/
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

### Corporate Artifactory dependency mirror

If the build machine cannot access public artifact repositories, obtain the
URL of an Artifactory **virtual Maven repository** from the repository
administrator. The virtual repository must aggregate or proxy all of:

- Maven Central;
- Spring milestone and snapshot repositories (`repo.spring.io`);
- the Gradle Plugin Portal Maven repository (`plugins.gradle.org/m2`).

A browser URL ending only in `/artifactory/` is not sufficient; use the full
repository URL ending in its virtual repository key. A Central-only virtual
repository cannot supply milestone dependencies such as Reactor Netty 5 M3.

Configure the URL in the current Git Bash session, without adding it to this
repository:

```bash
export PATCH_ARTIFACTORY_MAVEN_URL="https://artifactory.example/artifactory/virtual-repository-key"
export PATCH_JAVA_HOME="/c/JAVA/JDK/OpenJDK21"
bash ./patchctl doctor
bash ./patchctl test all ../patched-spring
```

If authentication is required, prompt for the values so the password or token
is not written into shell history:

```bash
read -r -p "Artifactory user: " PATCH_ARTIFACTORY_USERNAME
read -r -s -p "Artifactory password/token: " PATCH_ARTIFACTORY_PASSWORD
echo
export PATCH_ARTIFACTORY_USERNAME PATCH_ARTIFACTORY_PASSWORD
```

The same environment variables can configure Maven on the other host. Use the
credential-free template without copying it over an existing user settings
file:

```bash
export PATCH_ARTIFACTORY_MAVEN_URL="https://artifactory.example/artifactory/virtual-repository-key"
read -r -p "Artifactory user: " PATCH_ARTIFACTORY_USERNAME
read -r -s -p "Artifactory password/token: " PATCH_ARTIFACTORY_PASSWORD
echo
export PATCH_ARTIFACTORY_USERNAME PATCH_ARTIFACTORY_PASSWORD

mvn --settings ./examples/maven-settings-artifactory.xml \
  -Dmaven.repo.local=/c/TEMP/empty-maven-repository \
  dependency:tree
```

`examples/maven-settings-artifactory.xml` mirrors every Maven repository to the
one approved virtual repository. Its `<server>` id matches the mirror id so
Maven supplies the credentials only to that mirror. If authentication is not
required, leave both credential variables unset and remove the `<servers>`
block from a local copy. The template configures dependency resolution only;
the Spring source builds are Gradle builds and use the `PATCH_ARTIFACTORY_*`
variables through `patchctl` as described below. Manual release upload remains
manual.

`patchctl` injects `gradle/artifactory-mirror.init.gradle` only when the URL is
set. That init script removes public plugin, buildscript, and dependency
repositories for the build invocation and uses only the configured mirror.
Credentials are inherited from the process environment and are never written
to a source repository, patch, generated POM, or log by the kit. Clear them
afterward with:

```bash
unset PATCH_ARTIFACTORY_PASSWORD PATCH_ARTIFACTORY_USERNAME
```

The Gradle wrapper ZIP is downloaded before an init script can run. If
`services.gradle.org` is also blocked and the wrapper is not already cached,
ask the Artifactory administrator for mirrored Gradle 8.14.3 and 8.14.5
distribution ZIPs or have those wrapper distributions pre-populated in the
build account's Gradle cache. Do not put wrapper credentials in this repository.

If Artifactory uses an internal certificate authority, import that CA into the
JDK 21 trust store approved for the build machine. Do not disable TLS certificate
or revocation checks. An HTML file downloaded with a `.jar` name usually means
the requested repository path returned a login, proxy, or error page rather
than the artifact.

### IntelliJ IDEA workflow on Windows

Create and verify the patched repositories from Git Bash before opening
IntelliJ:

```bash
cd spring-patch-kit
export PATCH_JAVA_HOME="/c/Program Files/Eclipse Adoptium/jdk-21"
bash ./patchctl apply all ../patched-spring
bash ./patchctl verify all ../patched-spring
```

Spring Framework, Spring Integration, and Spring Security are independent Gradle builds. Open
each one separately with **File > Open** (using separate IntelliJ windows):

```text
patched-spring/spring-framework-upstream
patched-spring/spring-integration-upstream
patched-spring/spring-security-upstream
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

Build or publish all complete custom artifact sets to Maven Local:

```bash
./patchctl build all /path/to/replayed-spring
./patchctl publish-local all /path/to/replayed-spring
```

### Selective Maven publication

The complete-family publication above remains the lowest-risk option. When the
environment requires uploading only affected modules, prepare a separate
Maven-only repository containing exactly the registered patched artifacts:

```bash
bash ./patchctl stage-selective all /path/to/replayed-spring \
  /path/to/new/selective-maven-upload
```

The output directory must not exist. The command generates and validates:

```text
org.springframework:{spring-beans,spring-context-support,spring-expression,spring-web,spring-webflux,spring-webmvc}:6.2.19-cve.2
org.springframework.integration:{spring-integration-core,spring-integration-http,spring-integration-ip,spring-integration-jdbc,spring-integration-scripting,spring-integration-smb,spring-integration-zip}:6.5.10-cve.2
org.springframework.security:{spring-security-config,spring-security-core,spring-security-crypto,spring-security-ldap,spring-security-web}:6.5.11-cve1
```

Each coordinate includes its main, source, and Javadoc JAR plus a POM. The
selective POM transformer preserves dependencies between patched artifacts
(`spring-webmvc` still requires patched `spring-web`) and rewrites unaffected
Spring Framework dependencies to `6.2.19` and unaffected Spring Integration
dependencies to `6.5.10`. It then rejects any remaining custom-version
dependency on an artifact that is not staged.

The staging repository deliberately omits Gradle module metadata so Maven and
Gradle consumers use the validated POM dependency graph. `SHA256SUMS` covers
every staged artifact. Retain that manifest as release evidence rather than
uploading it as a Maven coordinate.

Selective consumers must manage every staged module directly; do not import a
custom Framework, Integration, or Security BOM because the remaining custom-version
modules are intentionally absent. Start from the checked-in
[selective Maven consumer example](examples/selective-maven-consumer/pom.xml)
and confirm the resolved dependency tree from an empty local repository.

This mode creates a deliberately mixed release: patched artifacts plus
official base-version dependencies. Run application smoke and integration
tests before approval. Do not combine the selective and complete publication
modes under the same immutable private version after either one has been
released.

## Publish the patched source forks

Keep the patch kit and the three patched Spring source repositories separate.
Create these public forks once through GitHub's web interface:

- `spring-projects/spring-framework` to `woozer/spring-framework`
- `spring-projects/spring-integration` to `woozer/spring-integration`
- `spring-projects/spring-security` to `woozer/spring-security`

Do not enter a GitHub token in this repository or in a command URL. Preview the
publication from a verified workspace first:

```bash
bash ./patchctl publish-source all /path/to/replayed-spring
```

The preview shows the exact commit, branch, immutable private-version tag, and
public fork URL for each project. After reviewing it, run the push yourself in
Git Bash so GitHub authentication remains in Git Credential Manager or the
browser:

```bash
bash ./patchctl publish-source all /path/to/replayed-spring --push
```

For each repository, `--push` adds a dedicated `patch-publish` remote if needed
and atomically pushes only the registered backport branch and private-version
tag. It does not push the default branch, other branches, build output, Gradle
caches, or the patch-kit repository. It never force-pushes. The existing
official Spring `origin` remote is left unchanged.

Fork creation remains manual because automating it would require GitHub account
credentials. The public HTTPS fork URLs in `project.conf` contain no secrets.
If one project in an `all` operation fails, investigate and rerun it; Git cannot
make a single atomic transaction across independent repositories.

The patch-kit repository shows the portable mail patches; each source fork
shows the same changes as normal commits on its cumulative backport branch.
Inspect a CVE locally with `git log BASE_REF..HEAD` and `git show COMMIT`, or use
the corresponding GitHub branch after publication. `patchctl` does not create
GitHub pull requests or merge requests. Those are optional review objects, not
part of the reproducible release, and creating them would require authenticated
GitHub tooling. The immutable source tag, signed CVE commits, checked-in mail
patches, and release evidence remain available even without a pull request.

## Manual Artifactory upload

This environment requires a manual Artifactory upload. `patchctl` deliberately
does not connect or authenticate to Artifactory. Choose one immutable release
mode before uploading:

- `publish-local` prepares the complete Maven publication families under the
  current user's local Maven repository;
- `stage-selective` prepares only the registered affected Maven modules in a new,
  checksummed upload directory.

The complete publications are stored under:

```text
~/.m2/repository/org/springframework/
```

For complete mode, upload all private version sets to the approved Artifactory
Maven repository while preserving their Maven coordinates and directory layout:

```text
org.springframework:*:6.2.19-cve.2
org.springframework.integration:*:6.5.10-cve.2
org.springframework.security:*:6.5.11-cve1
```

Upload the generated POMs and required JARs (including BOM POMs and any source
or Javadoc artifacts required by local policy). Do not upload Maven-local
bookkeeping files such as `_remote.repositories` or `maven-metadata-local.xml`;
Artifactory manages its own repository metadata.

For selective mode, upload only the `org/` repository paths inside the new
staging directory. Preserve every POM and JAR path exactly. Retain
`SHA256SUMS` with the release record and verify Artifactory's stored checksums
against it. Do not upload a custom BOM: consumers manage the staged affected
coordinates directly using the example POM.

Enter Artifactory credentials only in the approved Artifactory interface. Do
not place its URL, username, password, access token, or API key in this
repository, shell history, patch files, or build logs.

After the manual upload, validate from a clean application environment that has
no matching artifacts in Maven Local. Resolve all private versions from
Artifactory, inspect the dependency graph, and run the application smoke and
integration tests. Record the uploader, date, target repository, artifact
checksums, and validation result in the CVE release evidence.

Commands can target one project instead of `all`:

```bash
./patchctl list
./patchctl test spring-framework-6.2.19 /path/to/replayed-spring
./patchctl stage-selective spring-framework-6.2.19 \
  /path/to/replayed-spring /path/to/new/framework-selective-upload
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
series. If its Gradle project names differ from their source directory names,
also define a positional `SELECTIVE_DIRECTORIES` array alongside
`SELECTIVE_MODULES`.

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
