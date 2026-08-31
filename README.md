# scala-skills

A collection of Claude Code skills for Scala development, distributed as a
plugin marketplace.

## Installation

Run the following in Claude Code:

```
/plugin marketplace add windymelt/scala-skills
/plugin install scala-skills@scala-skills
```

## Skills

| Skill | Invocation | Description |
|-------|------------|-------------|
| [containerize](./skills/containerize/SKILL.md) | `/scala-skills:containerize` | Containerize an sbt project into an OCI image with Cloud Native Buildpacks, without writing a Dockerfile |
| [test-tagging](./skills/test-tagging/SKILL.md) | `/scala-skills:test-tagging` | Tag ScalaTest tests and run or exclude specific subsets (slow, heavy, etc.) |
| [publish-library](./skills/publish-library/SKILL.md) | `/scala-skills:publish-library` | Set up automated publishing to Maven Central from GitHub Actions with sbt-ci-release |
| [sbt-2](./skills/sbt-2/SKILL.md) | `/scala-skills:sbt-2` | Work with sbt 2.x or migrate from 1.x, consulting the official sbt 2.x docs directly |

Skills also trigger automatically on relevant keywords (e.g. "containerize
this", "build a Docker image").

## Layout

```
.claude-plugin/
├── marketplace.json   # marketplace definition
└── plugin.json        # plugin definition
skills/
└── containerize/
    └── SKILL.md       # skill body
```

To add a skill, create a directory under `skills/` and place a `SKILL.md`
in it.
