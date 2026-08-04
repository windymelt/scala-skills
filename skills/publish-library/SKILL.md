---
name: publish-library
description: >-
  Set up automated publishing of a Scala library to Maven Central from
  GitHub Actions using sbt-ci-release: plugin setup, build.sbt metadata,
  Sonatype user token, PGP key bootstrap, and the release workflow
  (snapshot on main merge, release on v-tag). Triggers on requests like
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

At the **very top** of `build.sbt`, add:

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

**Primary method** — run the bootstrap tool in the repository directory:

```sh
cs launch dev.capslock::scala-pgp-bootstrap:0.0.2
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

## Release flow (day-to-day operation)

- Merging a PR into `main` publishes a SNAPSHOT automatically
  - If this fails, the first thing to check is that `build.sbt` does
    **not** set `version` — the plugin derives the version from git tags
- Tagging publishes a release. The tag **must** start with `v`:

  ```sh
  git tag -a v0.1.0 -m "v0.1.0"
  git push origin v0.1.0
  ```

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
