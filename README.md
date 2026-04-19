# GitHub Copilot Agent Skills — Spec Tutorial

A practical guide to the [GitHub Copilot Agent Skills specification](https://agentskills.io) for experienced developers.

This repository teaches you how Agent Skills work, how to author them correctly, and how to wire them into a real project — either from scratch or into an existing codebase. No theory-only content. Every section produces something you can run or commit.

## What is in this repository

### [TUTORIAL.md](./TUTORIAL.md)

The full Agent Skills specification walkthrough. Written for developers who are comfortable with VS Code, git, and shell scripting.

Covers:

1. What Agent Skills are and are not
2. Prerequisites
3. Skill storage locations
4. SKILL.md specification
5. Writing a skill that actually works
6. Testing skills locally
7. Packaging with skillpack.sh
8. CI with GitHub Actions
9. Common mistakes
10. Spec field reference

### [Project1-Greenfield.md](./Project1-Greenfield.md)

A GitHub Copilot Agent mode prompt. Open an empty folder in VS Code, paste this file into Copilot Chat in Agent mode, and it builds a complete skills-enabled repository from the first commit.

What it creates:

- `.github/skills/feature-scaffold/`
- `.github/skills/code-review/`
- `.github/skills/git-commit/`
- `.github/skills/skill-scaffold/`
- `scripts/skillpack.sh`
- `.github/workflows/validate-skills.yml`
- `instructions.md`
- `README.md`
- `.gitignore`

Estimated time: no explicit time estimate is provided in [Project1-Greenfield.md](./Project1-Greenfield.md).

### [Project2-Retrofit.md](./Project2-Retrofit.md)

A GitHub Copilot Agent mode prompt. Open an existing repository in VS Code, paste this file into Copilot Chat in Agent mode, and it audits the codebase, identifies skill candidates, and adds skills without touching application code.

What it does:

- Runs a baseline repository audit across structure, documentation, existing CI, commit history patterns, code structure, and any existing skill infrastructure.
- Ranks candidate skills using explicit criteria for frequency, step complexity, and cost of mistakes, then asks the user to select which skills to build.
- Gathers repo-specific implementation context for each selected skill, then writes concrete `SKILL.md` files based on real commands and file paths from the target repository.
- Adds and validates skill infrastructure (`scripts/skillpack.sh`, skill validation workflow, `instructions.md`) and keeps the change set scoped away from application source code.

Estimated time: no explicit time estimate is provided in [Project2-Retrofit.md](./Project2-Retrofit.md).

## Prerequisites

- VS Code 1.108 or later
- GitHub Copilot subscription — Free, Pro, Pro+, Business, or Enterprise
- Git configured locally
- Bash available — macOS and Linux native; Windows users: Git Bash or WSL

For the project prompts specifically: Copilot must be running in **Agent mode**, not Ask mode. Open Chat with `Ctrl+Alt+I` (Windows/Linux) or `Cmd+Ctrl+I` (macOS), then switch the mode selector to Agent before pasting the prompt.

## How to use this repository

**If you are new to Agent Skills:**

Start with [TUTORIAL.md](./TUTORIAL.md). Read sections 1 through 6 before touching either project prompt. The tutorial covers the SKILL.md specification, the name-folder match rule, and how Copilot discovers skills — all of which affect whether the prompt run produces a working result.

**If you are starting a new repository:**

1. Read [TUTORIAL.md](./TUTORIAL.md) sections 1 through 6
2. Open an empty folder in VS Code
3. Switch Copilot Chat to Agent mode
4. Paste [Project1-Greenfield.md](./Project1-Greenfield.md) into the chat
5. Answer the four setup questions the prompt asks (project name, description, stack, GitHub username)
6. Review the generated files, then push to GitHub

**If you have an existing repository to retrofit:**

1. Read [TUTORIAL.md](./TUTORIAL.md) sections 1 through 6
2. Open your existing repository in VS Code
3. Switch Copilot Chat to Agent mode
4. Paste [Project2-Retrofit.md](./Project2-Retrofit.md) into the chat
5. The prompt audits your repo and presents skill candidates — select which ones to build
6. Review the generated files, then open a PR

**If you already know the spec and just need a reference:**

Go directly to [TUTORIAL.md](./TUTORIAL.md) sections 9 and 10 — Common mistakes and Spec field reference.

## What you will be able to do

After working through the tutorial and one project:

- Understand the Agent Skills open standard and how it differs from custom instructions, prompt files, and MCP servers
- Write `SKILL.md` files that pass VS Code's name-folder match requirement and spec field constraints
- Write descriptions that trigger Copilot auto-invocation reliably
- Package skills into a distributable archive with a JSON manifest using `skillpack.sh`
- Validate skills in CI so name mismatches and malformed frontmatter fail the build before merging
- Add skills to a new or existing project without disrupting application code or existing workflows

## Agent Skills quick reference

### What a skill is

A folder containing a `SKILL.md` file. Copilot reads the `name` and `description` from the frontmatter to decide when to load the skill. When a match is detected, it loads the body into context and follows the instructions.

### Required frontmatter

```yaml
---
name: skill-name          # Must match the folder name exactly. Lowercase + hyphens. Max 64 chars.
description: >            # Max 1024 chars. Include every phrase a developer would type.
  What this skill does.
  Use this when asked to X, Y, or Z.
---
```

The `name` and folder name must be identical. VS Code silently skips skills where they differ — no error, no warning.

### Storage locations

VS Code discovers skills from these directories:

```text
# Project skills (anyone who clones the repo gets them)
.github/skills/<skill-name>/SKILL.md
.claude/skills/<skill-name>/SKILL.md
.agents/skills/<skill-name>/SKILL.md

# Personal skills (follow you across workspaces)
~/.copilot/skills/<skill-name>/SKILL.md
```

### Invoking skills

- Type `/` in Copilot Chat to see all discovered skills
- Type `/skill-name` to invoke directly
- Describe what you want — Copilot matches the description and loads the skill automatically

### The one rule that breaks most setups

The `name` field in `SKILL.md` must exactly match the folder name. If they differ, VS Code skips the skill without reporting an error. Every skill in the project prompts is validated against this rule before the session completes.

## Further reading

- [VS Code Agent Skills documentation](https://code.visualstudio.com/docs/copilot/customization/agent-skills) — official Microsoft docs, updated with each VS Code release
- [Agent Skills open standard](https://agentskills.io) — the specification this tutorial is based on
- [GitHub Docs: Adding agent skills for Copilot](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills)
- [github/awesome-copilot](https://github.com/github/awesome-copilot) — community-contributed skills, instructions, and plugins
- [anthropics/skills](https://github.com/anthropics/skills) — reference skills from Anthropic
- [mickpletcher/VSCode](https://github.com/mickpletcher/VSCode) — production Agent Skills catalog with CI packaging
