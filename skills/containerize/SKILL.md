---
name: containerize
description: >-
  Containerize a Scala project into an OCI container image without writing a
  Dockerfile, using Cloud Native Buildpacks (pack CLI + Paketo builder) and
  sbt-native-packager. The resulting image auto-computes JVM memory settings
  at startup. Triggers on requests like "containerize this", "build a Docker
  image", "create an OCI image", "pack build", "コンテナ化して",
  "Dockerイメージにして", "OCIイメージを作って".
---

# Containerizing a Scala Project as an OCI Image (Cloud Native Buildpacks)

Convert an sbt project into an OCI container image with Cloud Native
Buildpacks (CNB), without writing a Dockerfile. JVM memory parameters
(`-Xmx` etc.) are computed automatically at startup from the container's
memory limit, so no manual tuning is required.

Reference: https://blog.3qe.us/entry/2026/07/24/154004

## Prerequisites

Check the following before starting. If anything is missing, guide the user
through installing it.

1. The Docker daemon is running (`docker info`)
2. The `pack` CLI is installed (`pack version`)
   - If not installed, point the user to
     https://buildpacks.io/docs/for-platform-operators/how-to/integrate-ci/pack/
3. The target is an sbt project (the Paketo sbt buildpack detects on the
   presence of `build.sbt`)
   - scala-cli and Mill projects are out of scope for this skill; say so
     explicitly

## Steps

### 1. Determine the sbt version

Read `project/build.properties` to determine the sbt major version. sbt 1.x
and 2.x differ in artifact paths and required configuration (see step 3).

### 2. Set up sbt-native-packager

Add to `project/plugins.sbt` (skip if already present):

```scala
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.11.7")
```

Enable `JavaAppPackaging` on the target project in `build.sbt`:

```scala
lazy val root = project
  .enablePlugins(JavaAppPackaging)
  .settings(/* ... */)
```

If the entry point (`Compile / mainClass`) is ambiguous, set it explicitly.

### 3. Place project.toml

Create `project.toml` at the project root.

**For sbt 2.x** (required, because the task name and artifact path differ
from the buildpack defaults):

```toml
[_]
schema-version = "0.2"

[io.buildpacks]
exclude = [
  "target/",
  ".bloop/",
  ".bsp/",
  ".metals/",
]

[[io.buildpacks.build.env]]
name = "BP_SBT_BUILD_ARGUMENTS"
value = "Universal/packageBin"

[[io.buildpacks.build.env]]
name = "BP_SBT_BUILT_ARTIFACT"
value = "target/out/jvm/scala-*/*/universal/*.zip"
```

**For sbt 1.x**: the buildpack defaults
(`BP_SBT_BUILD_ARGUMENTS=universal:packageBin`,
`BP_SBT_BUILT_ARTIFACT=target/universal/*.zip`) work as-is, so a
`project.toml` with only the `exclude` section is sufficient:

```toml
[_]
schema-version = "0.2"

[io.buildpacks]
exclude = [
  "target/",
  ".bloop/",
  ".bsp/",
  ".metals/",
]
```

For multi-module builds where the target is not the root module, set the
module name via `BP_SBT_BUILT_MODULE`.

### 4. Build

```bash
pack build <image-name> --builder paketobuildpacks/ubuntu-noble-builder
```

- The first run fetches the builder image and resolves dependencies, so it
  takes a while (several minutes or more). Use a generous timeout (around
  10 minutes)
- On build failure, the buildpack log contains the raw sbt output. First
  verify that `sbt Universal/packageBin` (or `sbt universal:packageBin` for
  sbt 1.x) succeeds locally

### 5. Verify

```bash
docker run --rm -p 8080:8080 <image-name>
```

If the startup log contains a line like
`Calculated JVM Memory Configuration: -Xmx...`, automatic memory
configuration is working. Adjust the port number to match the application.

## Optional Settings

Pass via `--env` as needed (or via `[[io.buildpacks.build.env]]` in
`project.toml`).

| Purpose | Setting |
|---------|---------|
| Pin the JRE version | `--env BP_JVM_VERSION=21` |
| Bundle a JRE instead of a JDK | `--env BP_JVM_TYPE=JRE` |
| Minimize the runtime with jlink | `--env BP_JVM_JLINK_ENABLED=true` |
| Push directly to a registry | `pack build ghcr.io/<owner>/<name> --publish` |

To use Amazon Corretto (image size tends to be larger):

```bash
pack build <image-name> \
    --builder paketobuildpacks/ubuntu-noble-builder \
    --buildpack docker.io/paketobuildpacks/amazon-corretto \
    --buildpack paketo-buildpacks/java \
    --env BP_JVM_VERSION=21
```

## Caveats

- **Avoid UBI-based builders**: UBI 8 can break sbt due to glibc
  compatibility issues. Default to `paketobuildpacks/ubuntu-noble-builder`
- **Bundling static files**: files placed under `src/universal/` end up
  under `./<project-name>/` at runtime
- **Runtime JVM options**: pass `docker run -e JAVA_OPTS=...` as an
  environment variable and it is picked up as-is; no image rebuild needed
- For detailed buildpack configuration, see
  https://github.com/paketo-buildpacks/sbt
