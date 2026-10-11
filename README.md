# Agent skills

Reusable skills for coding agents, from [aecorredor](https://github.com/aecorredor). Each skill is a folder an agent loads when the task matches its description. The collection starts with the two skills below and will grow.

Skills follow the [Agent Skills](https://agentskills.io/specification) format. Install them with the [skills CLI](https://github.com/vercel-labs/skills) (`npx skills`). The CLI targets Cursor, Claude Code, Codex, and other agents. Browse the catalog at [skills.sh/aecorredor/agent-skills](https://skills.sh/aecorredor/agent-skills).

## Install

List the skills in this repo:

```bash
npx skills add aecorredor/agent-skills --list
```

Install interactively. The CLI asks which skills and which agents to set up in the current project:

```bash
npx skills add aecorredor/agent-skills
```

Install one skill:

```bash
npx skills add aecorredor/agent-skills --skill philosophy-of-software-design
```

Install both:

```bash
npx skills add aecorredor/agent-skills --skill philosophy-of-software-design --skill game-programming-patterns
```

Add `-g` to install for every project on your machine instead of only the current one:

```bash
npx skills add aecorredor/agent-skills -g
```

A project install can be committed so teammates share the same skills. A global install stays on your machine. Update later with `npx skills update`.

## Skills

| Skill | What it does |
| --- | --- |
| [`philosophy-of-software-design`](skills/philosophy-of-software-design/SKILL.md) | Reviews and shapes modules, APIs, and refactors so complexity stays low: deep modules, information hiding, error contracts, comments, and names. |
| [`game-programming-patterns`](skills/game-programming-patterns/SKILL.md) | Chooses language-neutral game architecture for frame timing, entities, decoupling, behavior, and hot-path layout. |

### `philosophy-of-software-design`

Use this when designing or reviewing modules, interfaces, error handling, comments, names, or refactors. The agent checks whether a module is deep, whether a design decision lives in one place, and whether an edit leaves the design easier to change.

Informed by John Ousterhout's *A Philosophy of Software Design*. The skill is an original procedure for agents.

```bash
npx skills add aecorredor/agent-skills --skill philosophy-of-software-design
```

### `game-programming-patterns`

Use this when a game needs a frame loop, an entity model, input and undo, events, services, state machines, content-driven behavior, object lifetime, spatial queries, or a cache-friendly layout. The guidance applies in any language or engine.

Informed by Robert Nystrom's *Game Programming Patterns*. The book's samples are C++. This skill keeps the decisions and leaves that code behind.

```bash
npx skills add aecorredor/agent-skills --skill game-programming-patterns
```

## Provenance

These skills are original procedures you can reuse under the [MIT license](LICENSE). They are informed by the books named above. They are independent works: they do not reproduce either book's text or code.

`notes/` holds the study notes behind the current skills. The installer copies only each skill folder (`SKILL.md` and its `references/`).

## Adding a skill

New skills land beside the ones already here:

```text
skills/<skill-name>/SKILL.md
skills/<skill-name>/references/   # optional, linked from SKILL.md
```

`name` in the frontmatter matches the folder name. The `description` says what the skill does and when to use it. That description is what agents match against, and what `npx skills add --list` prints. Keep `SKILL.md` at `skills/<skill-name>/`. A `SKILL.md` at the repository root would hide the rest of the collection from the CLI.
