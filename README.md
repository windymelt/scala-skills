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
