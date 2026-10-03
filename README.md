# kr-skills

Personal [Agent Skills](https://code.claude.com/docs/en/skills) for Claude Code (also
compatible with Codex CLI and OpenCode, which share the `SKILL.md` convention).

## Skills

- **sync-repos** — recursively find every git repo reachable from the current
  directory and bring each one onto its main branch, fast-forwarded to its remote.
- **code-review**, **codebase-design**, **domain-modeling**, **grill-me**,
  **grill-with-docs**, **grilling**, **to-spec** — imported from
  [mattpocock/skills](https://github.com/mattpocock/skills), see `SOURCES.md`.

## Installation

```
/plugin marketplace add kartikrustagi/kr-skills
/plugin install kr-skills@kartikrustagi
```
