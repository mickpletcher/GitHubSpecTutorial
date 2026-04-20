# GitHub Copilot Agent Skills Tutorial

<!-- markdownlint-disable MD029 MD032 -->

Implementation focused guide for authoring, testing, and rolling out Agent Skills.

## Contents

1. [What Agent Skills are and are not](#what-agent-skills-are-and-are-not)
2. [Quick Start: Build a minimal working skill in 10 minutes](#quick-start-build-a-minimal-working-skill-in-10-minutes)
3. [Installing and Publishing Skills with `gh skill`](#installing-and-publishing-skills-with-gh-skill)
4. [Skill locations and discovery behavior](#skill-locations-and-discovery-behavior)
5. [SKILL.md frontmatter: required vs optional](#skillmd-frontmatter-required-vs-optional)
6. [allowed-tools: when to use it and when not to](#allowed-tools-when-to-use-it-and-when-not-to)
7. [How Copilot loads a skill](#how-copilot-loads-a-skill)
8. [Authoring patterns that work](#authoring-patterns-that-work)
9. [Keeping Skills Small: Progressive Disclosure](#keeping-skills-small-progressive-disclosure)
10. [What is part of the spec vs what is this repo's convention](#what-is-part-of-the-spec-vs-what-is-this-repos-convention)
11. [Portability Across Agent Hosts](#portability-across-agent-hosts)
12. [Why my skill is not loading: troubleshooting checklist](#why-my-skill-is-not-loading-troubleshooting-checklist)
13. [Validation and test checklist](#validation-and-test-checklist)
14. [Project paths in this repository](#project-paths-in-this-repository)

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

## Installing and Publishing Skills with `gh skill`

As of GitHub CLI v2.90.0 (released April 16, 2026), skill management is a first-class CLI primitive. This is the fastest way to discover, install, and share skills once you have authored your own.

### Prerequisites

Update the GitHub CLI to version 2.90.0 or later:

```bash
gh --version
gh auth status
```

### Installing skills from a public catalog

```bash
# Browse and install interactively from a repo
gh skill install github/awesome-copilot

# Install a specific skill directly
gh skill install github/awesome-copilot documentation-writer

# Pin to a release tag for reproducibility
gh skill install github/awesome-copilot documentation-writer@v1.2.0

# Pin to a commit SHA for maximum reproducibility
gh skill install github/awesome-copilot documentation-writer --pin abc123def

# Target a specific agent or scope
gh skill install github/awesome-copilot documentation-writer --agent claude-code --scope user
```

### Discovering skills

```bash
gh skill search mcp-apps
gh skill list
```

### Publishing your own skills

If you maintain a skills repository, `gh skill publish` validates your skills against the agentskills.io spec and checks recommended remote settings like tag protection, secret scanning, immutable releases, and code scanning:

```bash
gh skill publish
```

These settings are not required, but they are strongly recommended for supply-chain security. Enabling immutable releases is particularly important. If someone gains control of the repository, they still cannot modify existing releases, so users pinning to tags remain protected.

### Provenance via frontmatter

When `gh skill install` adds a skill, it writes tracking metadata (source repository, ref, tree SHA) directly into the installed `SKILL.md` frontmatter. This metadata travels with the skill even when it is copied, moved, or reorganized, so you can always trace a skill back to its origin.

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

### Spec reference

| Frontmatter field           | Required | Spec limit               | Notes                                                                              |
| --------------------------- | -------- | ------------------------ | ---------------------------------------------------------------------------------- |
| `name`                      | Yes      | 64 chars, [a-z0-9-]      | Must match folder name exactly. See [agentskills.io](https://agentskills.io).      |
| `description`               | Yes      | 1024 chars               | Used for auto-discovery. See [agentskills.io](https://agentskills.io).             |
| `license`                   | No       | —                        | Per spec. Values like `Apache-2.0` or `Proprietary. See LICENSE.txt` are accepted. |
| `argument-hint`             | No       | —                        | Hint shown in the slash command menu. Quote the value.                             |
| `user-invocable`            | No       | —                        | Default `true`. Set `false` to hide from the `/` menu.                             |
| `disable-model-invocation`  | No       | —                        | Default `false`. Set `true` to require explicit invocation.                        |
| `allowed-tools`             | No       | —                        | Pre-approves tool access. Never pre-approve shell/bash without review.             |
| `version`                   | No       | —                        | **Repo convention in this tutorial, not part of the spec.** Useful for manifests.  |

**A note on `license`:** There is an open issue (github/copilot-cli#894) where the Copilot CLI validator has required `license` in some versions even though the spec marks it optional. If your skills fail to load with a missing-license error, add `license: Apache-2.0` (or your actual license) to the frontmatter as a workaround.

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

### More examples by skill type

#### Scaffold

Good:
Scaffolds a new Express route module with controller, service, types, and test file.
Use this when asked to add a new endpoint, create a new resource, scaffold an API
route, or add a CRUD feature for a resource name like `user` or `order`.

Bad:
Creates route files.

#### Review

Good:
Reviews Terraform changes for security and cost impact before apply. Use this when
asked to review infrastructure changes, audit a Terraform plan, check a `.tf` diff,
or verify a module change before merging to main.

Bad:
Reviews Terraform.

#### Generate

Good:
Writes a release notes entry from merged PRs since the last tag, grouped by
feat/fix/breaking. Use this when asked to generate release notes, write a
changelog entry, summarize what shipped, or prep a release.

Bad:
Generates release notes.

#### Audit

Good:
Audits a repo for unpinned dependencies across `requirements.txt`, `pyproject.toml`,
`package.json`, and `go.mod`. Use this when asked to check dependency pinning, find
unpinned packages, audit dependency versions, or prep a repo for reproducible builds.

Bad:
Checks dependencies.

### The pattern

Every good description has two halves: a specific statement of what the skill does (with concrete technology names), and a list of natural-language trigger phrases a developer would actually type. The description is how Copilot indexes the skill for auto-discovery. Treat it as search metadata, not marketing copy.

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

## Keeping Skills Small: Progressive Disclosure

A `SKILL.md` that runs to hundreds of lines is usually a design smell. The agent pays a context cost every time it loads the skill, even for invocations that only need a slice of the content.

The fix is progressive disclosure. Keep `SKILL.md` short and point to supplementary files that load only when needed.

### Structure

```text
.github/skills/code-review/
├── SKILL.md                      # Short — workflow steps and when to use
├── reference/
│   ├── review-checklist.md       # Loaded only for full reviews
│   └── severity-rubric.md        # Loaded only when categorizing issues
└── scripts/
  └── diff-stats.sh             # Executed only when invoked
```

### When to split

Move content out of `SKILL.md` and into a `reference/` file when any of these are true:

- The content is a long checklist or rubric that is referenced but not always needed
- The content is an example that illustrates a pattern but is not the pattern itself
- The content is a data table (error codes, status mappings, severity definitions)

### How the agent finds referenced files

Copilot automatically discovers all files in the skill directory when the skill is invoked. Reference them in `SKILL.md` with relative paths:

```text
For the full severity rubric, see [reference/severity-rubric.md](./reference/severity-rubric.md).
Apply the checklist in [reference/review-checklist.md](./reference/review-checklist.md).
```

The agent loads these on demand, keeping the base skill lightweight while making deep content available when the task requires it.

### Rule of thumb

If your `SKILL.md` is over around 100 lines, ask whether a chunk of it could live in a reference file. If your skill contains multiple distinct workflows, ask whether it should be split into multiple skills. Canonical examples of this pattern are in [anthropics/skills](https://github.com/anthropics/skills).

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

## Portability Across Agent Hosts

The Agent Skills specification is an open standard. The same `SKILL.md` format works across:

- GitHub Copilot in VS Code (chat and agent mode)
- GitHub Copilot CLI (terminal)
- GitHub Copilot cloud coding agent (automated tasks)
- Claude Code, Cursor, Codex, Gemini CLI, and other agent hosts that implement the spec

This means a skill you author in this tutorial is not locked to VS Code. It runs unchanged anywhere the spec is supported. When you commit a skill to `.github/skills/`, everyone on your team gets it regardless of which agent host they use.

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
