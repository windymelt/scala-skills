---
name: publish-library
description: >-
  Set up automated publishing of a Scala library to Maven Central from
  GitHub Actions using sbt-ci-release: pre-release CI (scalafmt, scalafix,
  Java/platform test matrix), plugin setup, build.sbt metadata, Sonatype
  user token, PGP key bootstrap, the release workflow (snapshot on main
  merge, release on v-tag), repository hardening, binary releases for
  CLI tools (compressed executables, SHA-512 checksums, PGP signatures,
  provenance attestation), and README setup with a Maven Central version
  badge. Triggers on requests like
  "publish this library to Maven Central", "set up sbt-ci-release",
  "release this library", "ライブラリをパブリッシュしたい",
  "Maven Centralに公開して", "sbt-ci-releaseをセットアップして".
---

# Publishing a Scala Library to Maven Central with sbt-ci-release

Set up the sbt plugin [sbt-ci-release](https://github.com/sbt/sbt-ci-release)
so that GitHub Actions publishes the library to Maven Central. After setup,
the operational flow is:

- merging a PR into `main` automatically publishes a SNAPSHOT
- pushing a `v`-prefixed tag (e.g. `v0.1.0`) publishes a proper release

References:
- https://blog.3qe.us/entry/2024/03/24/012435
- https://blog.3qe.us/entry/2025/12/10/015333 (scala-pgp-bootstrap)

## Secret-handling policy

Some steps involve credentials (Sonatype token, PGP secret key). Follow
these rules throughout:

- Never ask the user to paste secret values into the conversation, and
  never echo them. Have the user export them as environment variables and
  register them with `gh secret set` themselves
- Delete any local file containing key material with `shred -uv` (or
  `srm`) as soon as it is no longer needed

## Prerequisites

Check these up front and guide the user through anything missing:

1. A [Maven Central](https://central.sonatype.com/) account and a
   namespace. Signing in with GitHub automatically grants
   `io.github.<username>`; owning a domain allows claiming a namespace via
   DNS (see https://central.sonatype.org/register/namespace/). If SNAPSHOT
   publishing is desired, it must be enabled for the namespace in the
   Maven Central settings beforehand
2. An sbt project with **sbt 1.11.x or later** — check
   `project/build.properties` first, before anything else. sbt-ci-release
   must be **1.11.0 or later**
3. CLI tools: `gh` (authenticated), `jq`, `gpg`, and `shred` or `srm`
4. For the PGP bootstrap tool: Java 21+ and Coursier (`cs`)

## Pre-release preparation: formatting, linting, and CI

Before wiring up the release itself, put code-style enforcement and a CI
workflow in place so that every PR is validated before it can trigger a
SNAPSHOT publish.

### Set up scalafmt

Append to `project/plugins.sbt` (check the latest version on
https://github.com/scalameta/sbt-scalafmt):

```scala
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "<version>")
```

Create `.scalafmt.conf` with the following defaults — they are a starting
point, and the user may adjust any of them to the project's taste. Fill in
the latest scalafmt version
(https://github.com/scalameta/scalafmt/releases) and the dialect matching
the project's Scala version:

```hocon
version = <latest scalafmt version>
runner.dialect = <e.g. scala3, scala213>

maxColumn = 120
trailingCommas = always
align.preset = more
danglingParentheses.preset = true
indent.defnSite = 2
runner.dialectOverride.allowSignificantIndentation = false
runner.dialectOverride.allowQuietSyntax = true
newlines.topLevelStatementBlankLines = [
  {
    blanks { after = 1 }
  }
]
```

The `runner.dialectOverride.*` keys assume a Scala 3 dialect (keep braces,
allow quiet syntax); drop them on Scala 2 projects.

Format the whole codebase once and confirm the check passes:

```sh
sbt 'scalafmtAll; scalafmtSbt'
sbt 'scalafmtCheckAll; scalafmtSbtCheck'
```

### Set up scalafix

Append to `project/plugins.sbt` (check the latest version on
https://github.com/scalacenter/sbt-scalafix):

```scala
addSbtPlugin("ch.epfl.scala" % "sbt-scalafix" % "<version>")
```

Create `.scalafix.conf` with the following rules. These are defaults, not
requirements — adjust them with the user to fit the project (e.g. a
library with a Java-facing API may need `noThrows = false`):

```hocon
rules = [
  DisableSyntax
]

DisableSyntax.noVars = true
DisableSyntax.noThrows = true
DisableSyntax.noNulls = true
DisableSyntax.noReturns = true
DisableSyntax.noWhileLoops = true
DisableSyntax.noAsInstanceOf = true
DisableSyntax.noIsInstanceOf = true
DisableSyntax.noFinalVal = true
DisableSyntax.noFinalize = true
DisableSyntax.noValPatterns = true
```

`DisableSyntax` is a syntactic rule and needs no compiler support. If
possible, also organize imports with the semantic `OrganizeImports` rule.
It requires SemanticDB, enabled build-wide in `build.sbt` (for sbt 1 use
the `ThisBuild` scope or the `inThisBuild` list from the metadata step
below; for sbt 2 write bare settings):

```scala
semanticdbEnabled := true
semanticdbVersion := scalafixSemanticdb.revision // Scala 2 only; drop on Scala 3
```

Then prepend the rule in `.scalafix.conf`:

```hocon
rules = [
  OrganizeImports,
  DisableSyntax
]

// set true when the compiler provides -Wunused:imports
// (Scala 2.12.13+ / 2.13, or Scala 3.3+)
OrganizeImports.removeUnused = false
```

Apply the rules once and confirm the check passes. Existing code may
violate `DisableSyntax`; fix the violations (or discuss relaxing a rule
with the user) before wiring CI:

```sh
sbt scalafixAll
sbt 'scalafixAll --check'
```

### Add the CI workflow

Create `.github/workflows/ci.yml` meeting these requirements:

- run on every pull request and on pushes to the default branch
- check formatting and scalafix compliance **before** running tests
- compile and test on a matrix of the Java LTS versions the library
  supports (e.g. 8/11/17/21)
- end with an aggregate `ci-passed` job so that branch protection can
  require a single stable check name regardless of matrix expansion

Example (single-platform build; use the latest action versions, and pin
them to full-length SHAs if the SHA-pinning policy from the hardening
step below is enabled):

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: 17
      - uses: sbt/setup-sbt@v1
      - name: Check formatting
        run: sbt 'scalafmtCheckAll; scalafmtSbtCheck'
      - name: Check scalafix rules
        run: sbt 'scalafixAll --check'

  test:
    needs: lint
    strategy:
      fail-fast: false
      matrix:
        java: [8, 11, 17, 21]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: ${{ matrix.java }}
      - uses: sbt/setup-sbt@v1
      - name: Compile and test
        run: sbt +test

  ci-passed:
    needs: [lint, test]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          test "${{ needs.lint.result }}" = "success" && \
          test "${{ needs.test.result }}" = "success"
```

`sbt +test` compiles before testing, so compilation is covered; for a
project without tests yet, use `sbt +Test/compile` instead.

When the build cross-compiles with sbt-crossproject (JVM/JS/Native), add
a platform axis to the `test` job instead of duplicating jobs, and run
the JS/Native jobs on a single Java version:

```yaml
    strategy:
      fail-fast: false
      matrix:
        java: [8, 11, 17, 21]
        platform: [JVM]
        include:
          - { java: 17, platform: JS }
          - { java: 17, platform: Native }
```

```yaml
      - name: Compile and test
        run: sbt '+<module>${{ matrix.platform }}/test'
```

`<module>JVM` / `<module>JS` / `<module>Native` are the subproject names
generated by sbt-crossproject; adjust them to the build, and check the
existing platform matrix (`crossProject(...)`) to decide which axes to
include.

## Setup steps (once per project)

### 1. Add the plugin

Append to `project/plugins.sbt` (check the latest version on
https://github.com/sbt/sbt-ci-release):

```scala
addSbtPlugin("com.github.sbt" % "sbt-ci-release" % "<version>")
```

### 2. Remove settings that sbt-ci-release manages

Confirm that `build.sbt` does **not** define any of the following — the
plugin manages them, so they must not be present:

- `version` — this one almost always exists; make sure it is deleted
- `publishTo`
- `publishMavenStyle`
- `credentials`

### 3. Add publication metadata

Check the sbt major version first (`project/build.properties`) and use the
matching notation below.

**For sbt 1.x**, at the **very top** of `build.sbt`, add:

```scala
inThisBuild(List(
  organization := "io.github.example", // your namespace
  homepage := Some(url("https://github.com/<owner>/<repo>")),
  licenses := List(
    "MIT" -> url("https://spdx.org/licenses/MIT.html")
  ),
  developers := List(
    Developer(
      "ghUserName",      // ID on GitHub etc.
      "Display Name",
      "email@example.com",
      url("https://example.com") // homepage / SNS / contact page
    )
  ),
  versionScheme := Some("early-semver")
))
```

**For sbt 2.x**, align with sbt 2 notation instead of copying the sbt 1
snippet above:

- Bare settings at the top of `build.sbt` apply to all subprojects in
  sbt 2, so write the settings directly — do **not** wrap them in
  `inThisBuild(List(...))` and do not scope them to `ThisBuild`
- `licenses` changed from `Seq[(String, URL)]` to `Seq[License]`; prefer
  constants such as `License.MIT` / `License.Apache2`
- Keys typed as `URL` in sbt 1 (e.g. `homepage`) changed to `URI`; if
  `url("...")` does not compile, use `uri("...")` instead

```scala
// top of build.sbt; bare settings are build-wide in sbt 2
organization := "io.github.example"
homepage := Some(url("https://github.com/<owner>/<repo>"))
licenses := List(License.MIT)
developers := List(
  Developer(
    "ghUserName",
    "Display Name",
    "email@example.com",
    url("https://example.com")
  )
)
versionScheme := Some("early-semver")
```

Notes for both versions:

- Ask the user for the license, developer info, and namespace if they are
  not evident from the repository
- Licenses are `List[(SPDX id, URL)]`; the `License.*` constants from sbt
  librarymanagement may also be used. SPDX ids and reference URLs:
  https://raw.githubusercontent.com/spdx/license-list-data/master/json/licenses.json
  (e.g. BSD 3-Clause has id `BSD-3-Clause` and URL
  `https://spdx.org/licenses/BSD-3-Clause.html`)

### 4. Create a Sonatype user token (manual, cannot be automated)

Ask the user to:

1. Visit https://central.sonatype.com/usertoken and run
   `Generate User Token` (any name works — the repository name is a good
   choice; `Does not expire` is fine)
2. In their own terminal, export the resulting values and register them as
   repository secrets:

   ```sh
   export SONATYPE_USERNAME=...
   export SONATYPE_PASSWORD=...
   gh secret set SONATYPE_USERNAME --body "$SONATYPE_USERNAME"
   gh secret set SONATYPE_PASSWORD --body "$SONATYPE_PASSWORD"
   ```

### 5. Set up the PGP key

Maven Central verifies artifact authenticity with PGP signatures. Create a
key **per repository**.

**Primary method** — use the bootstrap tool. First look up its latest
version on Maven Central (or Scaladex):

```sh
curl -s 'https://repo1.maven.org/maven2/dev/capslock/scala-pgp-bootstrap_3/maven-metadata.xml' \
  | grep -oP '(?<=<latest>)[^<]+'
```

Then run it in the repository directory with that version:

```sh
cs launch dev.capslock::scala-pgp-bootstrap:<latest-version>
```

It interactively confirms each step (mostly answering "y") and automates:
generating a signing-only key with a random passphrase in an isolated
environment (the local keyring is untouched), sending the public key to
keyservers, and registering `PGP_SECRET` / `PGP_PASSPHRASE` as GitHub
Actions secrets. Repository and email information is taken from GitHub. If
a confirmation email arrives from a keyserver (keys.openpgp.org), ask the
user to approve it.

**Fallback** — if the tool cannot be used, follow the manual procedure in
the appendix below.

### 6. Add the GitHub Actions workflow

Copy the official workflow and commit everything:

```sh
mkdir -p .github/workflows && \
  curl -L https://raw.githubusercontent.com/sbt/sbt-ci-release/main/.github/workflows/release.yml \
    > .github/workflows/release.yml
git add .github/workflows/release.yml build.sbt project/plugins.sbt
git commit -m 'Add sbt-ci-release workflow'
git push
```

### 7. Harden the repository (recommended)

Protect the branch and tags that drive releases, and lock down the supply
chain. All four settings below can be applied with `gh`. Note that
enforced rulesets on **private** repositories require a paid GitHub plan;
on public repositories they are free.

**Protect the default branch** — require a pull request before merging,
require the CI to pass, and block force pushes and branch deletion.
`required_approving_review_count` is set to `0` so that solo maintainers
are not blocked; raise it for team repositories. The required check
`ci-passed` is the aggregate job from the CI workflow section — requiring
it keeps the ruleset stable even when the test matrix changes:

```sh
gh api -X POST 'repos/{owner}/{repo}/rulesets' --input - <<'EOF'
{
  "name": "protect-default-branch",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": false,
        "required_status_checks": [ { "context": "ci-passed" } ]
      } },
    { "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 0,
        "dismiss_stale_reviews_on_push": false,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false
      } }
  ]
}
EOF
```

**Require signed release tags** — reject unsigned `v*` tags:

```sh
gh api -X POST 'repos/{owner}/{repo}/rulesets' --input - <<'EOF'
{
  "name": "signed-release-tags",
  "target": "tag",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["refs/tags/v*"], "exclude": [] } },
  "rules": [ { "type": "required_signatures" } ]
}
EOF
```

This requires the user to have a signing key (GPG or SSH) configured in
git **and** registered on their GitHub account so that signatures verify;
confirm this before activating the ruleset. Release tags must then be
created with `git tag -s` (see the release flow below).

**Enable release immutability** — once enabled, newly published GitHub
Releases lock their assets and tag permanently (they cannot be edited,
deleted, or re-tagged; existing releases stay mutable until republished):

```sh
gh api -X PUT 'repos/{owner}/{repo}/immutable-releases'
```

**Require actions to be pinned to a full-length commit SHA** — workflows
referencing actions by tag or branch are rejected at run time:

```sh
gh api -X PUT 'repos/{owner}/{repo}/actions/permissions' \
  -F enabled=true -f allowed_actions=all -F sha_pinning_required=true
```

`PUT` replaces the whole permissions object, so check the current
`allowed_actions` value first with
`gh api 'repos/{owner}/{repo}/actions/permissions'` and pass it through
unchanged.

Enabling this **breaks workflows that reference actions by tag** — both
`ci.yml` from the pre-release preparation section and `release.yml`
copied in step 6 do. Rewrite each `uses:` line to a full-length commit
SHA, keeping the tag as a comment. Resolve a tag to its SHA with:

```sh
gh api repos/actions/checkout/commits/v7 --jq .sha
```

```yaml
# before
uses: actions/checkout@v7
# after
uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7
```

## Release flow (day-to-day operation)

- Merging a PR into `main` publishes a SNAPSHOT automatically
  - If this fails, the first thing to check is that `build.sbt` does
    **not** set `version` — the plugin derives the version from git tags
- Tagging publishes a release. The tag **must** start with `v`, and must
  be signed (`-s`) when the tag-signature ruleset from step 7 is active
  (`-a` is enough otherwise):

  ```sh
  git tag -s v0.1.0 -m "v0.1.0"
  git push origin v0.1.0
  ```

  > **WARNING (for agents): never create or push a release tag without
  > the user's explicit, per-instance permission.** Creating a signed tag
  > uses the user's own signing key in their name, and pushing it
  > triggers an irreversible publication to Maven Central (releases
  > cannot be unpublished). Neither a general instruction to "set up
  > releasing" nor permission granted for a previous tag counts as
  > permission. Before each `git tag` / `git push` of a `v*` tag, state
  > the exact tag name and target commit, and wait for the user's
  > approval in that conversation.

## Prepare the README

Once the first release is out, propose updating the README so that users
can find and depend on the library:

- **Version badge** — use
  [maven-badges](https://github.com/softwaremill/maven-badges) to show the
  latest version on Maven Central. The current domain is
  `maven-badges.sml.io` (the old Heroku domain is retired). Note that the
  artifact id must include the Scala-version suffix (`_3`, `_2.13`, and
  for Scala.js e.g. `_sjs1_3`):

  ```markdown
  [![Maven Central](https://maven-badges.sml.io/maven-central/<groupId>/<artifactId>_3/badge.svg)](https://maven-badges.sml.io/maven-central/<groupId>/<artifactId>_3/)
  ```

  The first path segment selects the repository to query — `maven-central`
  is right for the sbt-ci-release flow (artifacts published via Sonatype
  Central sync to Maven Central). The badge resolves only after the first
  proper (non-SNAPSHOT) release has been published and synced

- **Installation snippet** — the `libraryDependencies` line matching the
  released coordinates:

  ```scala
  libraryDependencies += "<groupId>" %% "<artifactId>" % "<version>"
  ```

  (use `%%%` in Scala.js / Scala Native crossProjects)

- For executables, the artifact verification commands from the
  "Releasing executables" section below

## Releasing executables (CLI tools)

When the artifact is an executable (e.g. a CLI tool) rather than — or in
addition to — a library, pushing a `v` tag should also create a GitHub
Release carrying the binaries. The release must include:

- release notes generated by
  [action-gh-release](https://github.com/softprops/action-gh-release)
- the executables compressed with zstd (or gzip)
- a SHA-512 checksum file covering the compressed artifacts
- detached PGP signatures made with the release key from the
  "Set up the PGP key" step (the same `PGP_SECRET` / `PGP_PASSPHRASE`
  secrets are reused)
- build provenance attestation via
  [actions/attest-build-provenance](https://github.com/actions/attest-build-provenance)

Add `.github/workflows/release-binary.yml`. The build step and binary
path are project-specific (GraalVM native-image, Scala Native,
sbt-native-packager, sbt-assembly, …) — adjust them, and check the latest
action versions (pin them to full-length SHAs if the SHA-pinning policy
from the hardening step is enabled; the third-party
`softprops/action-gh-release` is especially worth pinning):

```yaml
name: Release binaries

on:
  push:
    tags: ["v*"]

permissions:
  contents: write      # create the release
  id-token: write      # provenance attestation
  attestations: write  # provenance attestation

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: 17
      - uses: sbt/setup-sbt@v1
      - name: Build executable
        run: sbt <build command>            # project-specific
      - name: Compress and checksum
        run: |
          NAME=<tool>-${GITHUB_REF_NAME}-x86_64-linux
          zstd -19 -o "$NAME.zst" target/<binary>   # or: gzip -9 -c target/<binary> > "$NAME.gz"
          sha512sum ./*.zst > SHA512SUMS
      - name: Sign with the release PGP key
        env:
          PGP_SECRET: ${{ secrets.PGP_SECRET }}
          PGP_PASSPHRASE: ${{ secrets.PGP_PASSPHRASE }}
        run: |
          echo "$PGP_SECRET" | base64 -d | gpg --batch --import
          for f in ./*.zst SHA512SUMS; do
            gpg --batch --pinentry-mode loopback --passphrase "$PGP_PASSPHRASE" \
              --armor --detach-sign "$f"
          done
      - name: Attest build provenance
        uses: actions/attest-build-provenance@v4
        with:
          subject-path: "*.zst"
      - name: Create the release
        uses: softprops/action-gh-release@v3
        with:
          generate_release_notes: true
          files: |
            *.zst
            *.zst.asc
            SHA512SUMS
            SHA512SUMS.asc
```

Notes:

- With release immutability enabled (hardening step), assets are locked
  the moment the release is published — attach everything in a single
  `action-gh-release` invocation as above, and never split asset uploads
  across steps that run after publication
- For multi-platform binaries, build in a per-OS matrix job, pass the
  compressed artifacts to a final job with
  `actions/upload-artifact` / `actions/download-artifact`, and run
  checksum, signing, attestation, and release creation once there, so
  that a single `SHA512SUMS` covers every platform
- Verification commands worth documenting in the project README:
  `sha512sum -c SHA512SUMS`, `gpg --verify <file>.asc <file>`, and
  `gh attestation verify <file> -R <owner>/<repo>`

## Appendix: manual PGP key setup (fallback)

Run in the repository directory. Requires `gh`, `jq`, `gpg`, `openssl`,
and `shred`/`srm`. Written for linux/zsh; should also work in bash.

Authenticate so the user's email can be fetched from GitHub:

```sh
gh auth refresh -h github.com -s user
```

Generate the key unattended:

```sh
openssl rand 1000 | tr -dc '[:alnum:][:punct:]' | tr -d '\\' | fold -w 32 | head -1 > pgp-passphrase
KEY_NAME=$(gh repo view --json owner,name | jq -r '.owner.login + "/" + .name + " CI bot"')
cat > gen-key-script <<EOF
Key-Type: EDDSA
Key-Curve: ed25519
Key-Usage: sign
Passphrase: $(< pgp-passphrase)
Name-Real: $KEY_NAME
Name-Email: $(gh api user/emails --jq '.[].email' | head -n1)
Expire-Date: 0
EOF
gpg --batch --gen-key gen-key-script
KEY_ID=$(gpg --list-secret-keys --with-colons | grep -B3 -m1 "$KEY_NAME" | grep -e '^fpr' | cut -d : -f 10)
echo Key ID is: $KEY_ID
gpg --armor --batch --pinentry-mode loopback --passphrase-file pgp-passphrase --export-secret-keys "$KEY_ID" | openssl base64 | tr -d '\n' > pgp-secret-key
```

Publish the public key (an approval email will arrive from
keys.openpgp.org — ask the user to confirm it):

```sh
gpg --keyserver hkp://keyserver.ubuntu.com --send-key $KEY_ID && \
  gpg --keyserver hkp://keys.openpgp.org --send-key $KEY_ID
```

Register the secrets on GitHub:

```sh
gh secret set PGP_PASSPHRASE < pgp-passphrase
gh secret set PGP_SECRET     < pgp-secret-key
```

Destroy the local key material:

```sh
shred -uv pgp-passphrase pgp-secret-key gen-key-script
# or: srm pgp-passphrase pgp-secret-key gen-key-script
```
