# Agent skills

Two installable skills for coding agents. They follow the [Agent Skills](https://agentskills.io/specification) format and are discovered by [`npx skills`](https://skills.sh).

The skills are original procedures. They are informed by John Ousterhout's *A Philosophy of Software Design* and Robert Nystrom's *Game Programming Patterns*. They are not reproductions of either book. Study notes in `notes/` are the working synthesis behind the skills. The installer does not copy `notes/`.

| Skill | Use it when |
| --- | --- |
| `philosophy-of-software-design` | Designing or reviewing modules, APIs, error contracts, comments, names, and refactors |
| `game-programming-patterns` | Choosing language-neutral game architecture for time, entities, decoupling, and hot-path layout |

## Install

From this checkout, list what the CLI will see:

```bash
npx skills add ~/Projects/agent-skills --list
```

Install both into the current project:

```bash
npx skills add ~/Projects/agent-skills --skill philosophy-of-software-design --skill game-programming-patterns
```

Add `-g` to install for every project on this machine. After this repo is published, the same commands work with `<owner>/agent-skills` in place of the local path.

## Layout

```text
skills/<skill-name>/SKILL.md
skills/<skill-name>/references/
```

There is no `SKILL.md` at the repository root. A root skill file would hide the skills in `skills/`.
