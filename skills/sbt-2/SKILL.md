---
name: sbt-2
description: >-
  Work with sbt 2.x projects or migrate a project from sbt 1.x, by consulting
  the official sbt 2.x documentation (the Book of sbt and "Migrating from
  sbt 1.x") directly instead of relying on sbt 1.x-era knowledge. Triggers on
  requests like "use sbt 2", "upgrade to sbt 2", "migrate this build to
  sbt 2.x", "why does this fail on sbt 2", "sbt 2に移行して",
  "sbt 2.xを使いたい", "sbtをアップグレードして", "sbt 2で動かして".
---

# Working with sbt 2.x

Use this skill when the target project uses sbt 2.x, or when migrating a
project from sbt 1.x to 2.x. sbt 2.x changes the build DSL, task semantics,
and command-line parsing in ways that invalidate sbt 1.x-era knowledge.

**Do not answer sbt 2.x questions from memory.** Training data is dominated
by sbt 1.x material, and sbt 2.x is still evolving. Fetch the official
documentation below with WebFetch and base every answer and edit on what the
docs actually say. When this file's summary and a fetched page disagree, the
fetched page wins.

## Primary sources

Fetch these directly:

| Purpose | URL |
|---------|-----|
| The Book of sbt (2.x reference, table of contents) | https://www.scala-sbt.org/2.x/docs/en/ |
| Migrating from sbt 1.x | https://www.scala-sbt.org/2.x/docs/en/changes/migrating-from-sbt-1.x.html |
| sbt 2.0 change summary | https://www.scala-sbt.org/2.x/docs/en/changes/sbt-2.0-change-summary.html |
| Cached tasks reference | https://www.scala-sbt.org/2.x/docs/en/reference/cached-task.html |
| Releases (current version) | https://github.com/sbt/sbt/releases |

Which to fetch first:

- **Migration task** ("upgrade this build to sbt 2") → the migration guide.
  It is organized as one section per breaking change; walk the sections and
  apply the ones that match the build.
- **Usage or reference question** (DSL, tasks, settings, plugins) → start at
  the Book of sbt table of contents and follow links to the relevant
  chapter. Note that page paths differ from the 1.x manual; do not guess
  1.x-style URLs.
- **"Why does X behave differently from sbt 1.x?"** → the change summary,
  then the matching migration-guide section.
- **Version choice** → check the GitHub releases page for the latest 2.0.x
  (sbt 2.0.0 is GA; 2.0.8 was current as of September 2026 — verify, do not
  reuse this number).

## Detecting the sbt version

Read `project/build.properties`. `sbt.version=2.*` means this skill's rules
apply; `sbt.version=1.*` means the project is on 1.x and "migration" means
also bumping this value.

## High-impact differences from sbt 1.x

This is an orientation list, not a substitute for the docs — confirm each
item's details in the migration guide before acting on it:

- **`build.sbt` DSL is Scala 3**: custom tasks and plugin code in the build
  must compile under Scala 3.x syntax.
- **Batch mode takes one command string**: a sequence of commands must be a
  single quoted, semicolon-separated argument — `sbt "clean; compile; test"`.
  The sbt 1.x space-separated form (`sbt clean compile test`) is an error
  ("Expected whitespace character"). The quoted form also works on 1.x, so
  prefer it everywhere, including CI `run:` lines.
- **Bare settings apply to all subprojects** (not just the root), replacing
  most `ThisBuild` usage; scope with `LocalRootProject` for root-only.
- **Tasks are cached by default**; non-serializable results need
  `Def.uncached(...)` (see the cached-task reference).
- **`%%` is platform-aware**: it subsumes `%%%` for Scala.js / Scala Native.
- **Slash syntax only**: `test:compile`-style 0.13 syntax is gone; use
  `Test/compile`.
- **`IntegrationTest` configuration is removed**: use a separate subproject.
- **Output layout changed**: artifacts live under a unified `target/out`
  tree, which affects CI paths that reach into `target/`.
- **`exportJars` defaults to `true`**, and forked runs use the build root as
  the working directory.

## Steps for a migration task

1. Read `project/build.properties`, `build.sbt`, and `project/*.sbt` /
   `project/*.scala` to inventory what the build uses.
2. Fetch the migration guide and list which of its sections apply to this
   build.
3. Check every sbt plugin in `project/plugins.sbt` for an sbt 2.x-compatible
   release (plugin READMEs / Maven Central). A plugin without one is a
   blocker to surface to the user, not to work around silently.
4. Apply the applicable migration sections, bump
   `project/build.properties`, and update CI command lines to the quoted
   semicolon form.
5. Verify with `sbt "clean; Test/compile; test"` (or the project's
   equivalent) and report any remaining deprecation output.

## Caveats

- **The 2.x docs move**: pages are added and reorganized; if a URL above
  404s, start from the table of contents rather than guessing paths.
- **Plugin ecosystem lag**: sbt 1.x plugins do not load on 2.x unless
  cross-built; check compatibility before promising a migration is
  mechanical.
- **Mixed knowledge is the main failure mode**: an answer that blends 1.x
  defaults with 2.x syntax often type-checks but misbehaves. When in doubt,
  re-fetch the relevant chapter.
