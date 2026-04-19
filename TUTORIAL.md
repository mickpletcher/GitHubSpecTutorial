# GitHub Copilot Agent Skills Tutorial

<!-- markdownlint-disable MD029 MD032 -->

Implementation focused guide for authoring, testing, and rolling out Agent Skills.

## Contents

1. [What Agent Skills are and are not](#what-agent-skills-are-and-are-not)
2. [Quick Start: Build a minimal working skill in 10 minutes](#quick-start-build-a-minimal-working-skill-in-10-minutes)
3. [Skill locations and discovery behavior](#skill-locations-and-discovery-behavior)
4. [SKILL.md frontmatter: required vs optional](#skillmd-frontmatter-required-vs-optional)
5. [allowed-tools: when to use it and when not to](#allowed-tools-when-to-use-it-and-when-not-to)
6. [How Copilot loads a skill](#how-copilot-loads-a-skill)
7. [Authoring patterns that work](#authoring-patterns-that-work)
8. [What is part of the spec vs what is this repo's convention](#what-is-part-of-the-spec-vs-what-is-this-repos-convention)
9. [Why my skill is not loading: troubleshooting checklist](#why-my-skill-is-not-loading-troubleshooting-checklist)
10. [Validation and test checklist](#validation-and-test-checklist)
11. [Project paths in this repository](#project-paths-in-this-repository)

---

## What Agent Skills are and are not

Agent Skills are task-specific instruction folders that contain a `SKILL.md` file and optional supporting resources.

Agent Skills are good for:
- Repeatable multi-step workflows
- Structured output requirements
- Workflows that benefit from explicit instructions and checks

Agent Skills are not the same as:
- Custom instructions: persistent coding rules that always apply
- Prompt files: one-shot reusable prompts
- Custom agents: broader persistent agent behavior and orchestration
- MCP servers: tool and API integration layers

Use Agent Skills when you want reusable workflow behavior, not global coding policy.

---

## Quick Start: Build a minimal working skill in 10 minutes

This is the smallest useful skill you can create and test.

### Step 1: Create the directory

```bash
mkdir -p .github/skills/commit-summary
```

### Step 2: Create SKILL.md

```markdown
---
name: commit-summary
description: Summarizes staged git changes in bullet points. Use this when asked to summarize staged changes, explain a git diff, or list what changed before committing.
---

# Commit Summary

## Inputs

Use staged git changes.

## Workflow

1. Read `git diff --staged`.
2. Group changes by file.
3. Produce a concise bullet summary.

## Output format

- One bullet per changed file
- One sentence per bullet
- Mention behavior impact if present
```

### Step 3: Validate basic correctness

Checklist:
- Folder name is `commit-summary`
- `name` value is `commit-summary`
- `description` includes phrases you would actually type

### Step 4: Verify discovery and invocation

1. Open Copilot Chat.
2. Type `/` and confirm `commit-summary` appears.
3. Stage any file and type `/commit-summary`.
4. Test natural language invocation:
   - "summarize my staged changes"

If step 2 fails, jump to [Why my skill is not loading: troubleshooting checklist](#why-my-skill-is-not-loading-troubleshooting-checklist).

---

## Skill locations and discovery behavior

Repository-level locations:
- `.github/skills/<skill-name>/SKILL.md`
- `.claude/skills/<skill-name>/SKILL.md`
- `.agents/skills/<skill-name>/SKILL.md`

Personal locations use the same pattern under your home directory, for example:
- `~/.copilot/skills/<skill-name>/SKILL.md`
- `~/.claude/skills/<skill-name>/SKILL.md`
- `~/.agents/skills/<skill-name>/SKILL.md`

Discovery behavior summary:
- Copilot reads frontmatter to index skills.
- It uses `description` matching to decide relevance.
- A skill can be invoked directly by `/skill-name`.

---

## SKILL.md frontmatter: required vs optional

Required fields:
- `name`
- `description`

Optional commonly used fields:
- `argument-hint`
- `user-invocable`
- `disable-model-invocation`
- `allowed-tools`

### Required fields

`name`:
- Must match the parent directory name exactly.
- Use lowercase and hyphens.

`description`:
- This is matching input, not just documentation.
- Include real trigger phrasing users type.

### Optional fields

`argument-hint`:
- UI hint for slash invocation inputs.

`user-invocable`:
- If false, hide from slash menu while still allowing model relevance behavior.

`disable-model-invocation`:
- If true, disables auto-invocation and requires explicit slash invocation.

`allowed-tools`:
- Optional tool allowlist when skill behavior needs tool execution.

Example with required plus optional fields:

```yaml
---
name: schema-review
description: Reviews database schema changes. Use this when asked to review migrations, check schema diffs, or assess DB change risk.
argument-hint: "[scope] optional path or migration name"
user-invocable: true
disable-model-invocation: false
allowed-tools:
  - read_file
  - grep_search
---
```

---

## allowed-tools: when to use it and when not to

Use `allowed-tools` when the skill needs deterministic tool boundaries.

Good use cases:
- Restricting a skill to read-only analysis tools
- Preventing accidental execution of unrelated tools
- Enforcing safer behavior for review-focused skills

Use caution with command execution tools:
- `shell` or `bash` pre-approval should be treated as high trust only.
- Never pre-approve shell tools in copied or unreviewed skills.
- If a skill can modify files or runtime state, require explicit review before use.

Recommended default for most teams:
- Start with read-only tools for new skills.
- Add execution tools only when necessary and documented.

---

## How Copilot loads a skill

Three-phase model:

1. Discovery
- Copilot indexes skill metadata from frontmatter.

2. Instruction loading
- When relevant, it loads skill instructions from `SKILL.md`.

3. Resource loading
- Referenced supporting files are loaded when needed.

Practical implication:
- A great body is useless if `description` never matches.
- A broad `description` with weak workflow steps causes noisy behavior.

---

## Authoring patterns that work

### Use specific workflow steps

Weak:
- "Review this change carefully"

Better:
- "Read `git diff --staged`, list risk areas by file, then output critical and advisory findings separately"

### Show expected output shape

Define headings and format explicitly. Output quality improves when structure is explicit.

### Keep supporting resources separate

Put long templates and checklists in separate files near `SKILL.md` and reference them from instructions.

### Include edge cases

At minimum include behavior for:
- Missing inputs
- Missing files
- Unsafe or ambiguous actions

---

## What is part of the spec vs what is this repo's convention

Spec-level behavior:
- Skills are folder-based with `SKILL.md`
- `name` and `description` are core frontmatter fields
- Description relevance drives auto matching behavior
- Skills can include optional resources and fields such as invocation controls and tool constraints

Repository conventions in this tutorial:
- Using a separate `README.md` per skill directory
- Using `version` in frontmatter for packaging metadata
- Using specific validation scripts and CI workflow names in project prompts

Important:
- Do not assume repo conventions are mandatory in every environment.

---

## Why my skill is not loading: troubleshooting checklist

Run this checklist top to bottom.

1. Path check
- Confirm skill is in a supported skills directory.

2. Name check
- Confirm directory name equals `name` exactly.

3. Frontmatter check
- Confirm valid YAML frontmatter with required fields.

4. Description check
- Confirm description includes actual user phrasing.

5. Invocation control check
- If `disable-model-invocation: true`, auto-invocation is off by design.

6. Tool policy check
- If `allowed-tools` is too restrictive, behavior may appear incomplete.

7. Explicit invoke test
- Run `/skill-name`. If this works but natural language does not, tune `description`.

---

## Validation and test checklist

Use this in any project using skills.

Pre-commit checks:
- Name and folder match
- Frontmatter parses
- Description includes trigger phrases
- Invocation controls are intentional
- Tool access policy is reviewed

Runtime checks:
- Slash discovery works
- Explicit invocation works
- Natural language invocation works
- Output format matches skill instructions
- No unintended command execution

Security checks:
- Review every shell-capable skill
- Avoid broad pre-approval for command execution tools
- Keep high-trust skills limited and documented

---

## Project paths in this repository

This repository contains tutorial documentation and project prompts.

- [Project1-Greenfield.md](./Project1-Greenfield.md): generate a skills-first repository from scratch.
- [Project2-Retrofit.md](./Project2-Retrofit.md): audit an existing codebase and add skills safely.

Note:
- This repository itself does not ship a live `.github/workflows/` or `scripts/skillpack.sh` implementation.
- Those are generated in target repositories by the project prompts.

---

## Further reading

- [VS Code Agent Skills documentation](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- [Agent Skills open standard](https://agentskills.io)
- [GitHub Docs: Adding agent skills for Copilot](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills)

<!-- markdownlint-enable MD029 MD032 -->
