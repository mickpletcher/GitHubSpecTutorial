# GitHub Copilot Agent Skills — Setup and Authoring Guide

This tutorial walks through the Agent Skills specification from scratch to a fully packaged and CI-deployed skillpack. It assumes you are comfortable with VS Code, git, and shell scripting. No hand-holding on basics.

---

## Contents

1. [What Agent Skills are and are not](#1-what-agent-skills-are-and-are-not)
2. [Prerequisites](#2-prerequisites)
3. [Skill storage locations](#3-skill-storage-locations)
4. [SKILL.md specification](#4-skillmd-specification)
5. [Writing a skill that actually works](#5-writing-a-skill-that-actually-works)
6. [Testing skills locally](#6-testing-skills-locally)
7. [Packaging with skillpack.sh](#7-packaging-with-skillpacksh)
8. [CI with GitHub Actions](#8-ci-with-github-actions)
9. [Common mistakes](#9-common-mistakes)
10. [Spec field reference](#10-spec-field-reference)

---

## 1. What Agent Skills are and are not

Agent Skills are directories that teach Copilot a repeatable workflow. When Copilot detects that your prompt matches a skill's description, it loads the skill's instructions into context and follows them. You can also invoke a skill directly with a `/skill-name` slash command.

**Agent Skills are:**
- An open standard ([agentskills.io](https://agentskills.io)) that works across VS Code, GitHub Copilot CLI, and the Copilot cloud coding agent
- Context loaded on demand — Copilot reads the name and description first, then loads the body only when relevant
- Portable across machines and teams when stored in a repository

**Agent Skills are not:**
- Custom instructions (`github.copilot.instructions`) — those always apply globally
- Prompt files (`.prompt.md`) — those are one-shot reusable prompts, not workflows
- MCP servers — skills cannot expose tools or call APIs; they are instruction sets only
- A replacement for `.github/copilot-instructions.md` for project-wide coding standards

**Choosing between skills, custom instructions, and prompt files:**

| Need | Use |
|---|---|
| Coding standards that always apply | Custom instructions |
| One-shot reusable prompt | Prompt file |
| Multi-step repeatable workflow | Agent Skill |
| External API integration | MCP server |

---

## 2. Prerequisites

- VS Code 1.108 or later (Agent Skills shipped in December 2025)
- GitHub Copilot subscription (Free, Pro, Pro+, Business, or Enterprise)
- Git configured with a remote on GitHub
- Bash available for the skillpack script (macOS/Linux native; Windows: Git Bash or WSL)

Verify your VS Code version:

```bash
code --version
```

Verify Copilot is active: open the Chat panel (`Ctrl+Alt+I` / `Cmd+Ctrl+I`) and confirm it loads without an auth error.

---

## 3. Skill storage locations

VS Code checks two roots for skills on startup.

### Project skills

Stored inside the repository. Scoped to anyone who opens the workspace.

VS Code recognizes any of these directories:

```
.github/skills/<skill-name>/SKILL.md
.claude/skills/<skill-name>/SKILL.md
.agents/skills/<skill-name>/SKILL.md
```

### Personal skills

Stored in your home directory. Available across all workspaces.

```
~/.copilot/skills/<skill-name>/SKILL.md
~/.claude/skills/<skill-name>/SKILL.md
~/.agents/skills/<skill-name>/SKILL.md
```

### Additional search paths

To load skills from a shared location — a central repo, a network drive, a monorepo parent — configure extra paths in VS Code settings:

```json
"chat.agentSkillsLocations": [
    "/path/to/shared/skills",
    "${workspaceFolder}/../team-skills"
]
```

### Confirming VS Code sees your skills

Type `/` in the Copilot Chat input. Every discovered skill appears in the menu alongside prompt files. If a skill is missing, the folder name or `name` frontmatter field is mismatched — see [Common mistakes](#9-common-mistakes).

---

## 4. SKILL.md specification

Each skill lives in its own directory. The only required file is `SKILL.md`. Everything else — scripts, templates, examples — is optional supporting material that the skill body can reference.

### Required directory structure

```
<skill-name>/
├── SKILL.md           # required by the spec
├── README.md          # required by this repo's conventions; optional per spec
└── <supporting files> # scripts, examples — loaded only when referenced
```

### File format

`SKILL.md` is a Markdown file with YAML frontmatter:

```markdown
---
name: skill-name
description: What this skill does and when to use it.
---

# Skill body

Instructions go here.
```

### Frontmatter fields

| Field | Required | Limit | Notes |
|---|---|---|---|
| `name` | Yes | 64 chars, lowercase + hyphens | **Must match the folder name exactly.** VS Code silently skips the skill if these differ. |
| `description` | Yes | 1024 chars | Used for auto-discovery. Include all natural-language trigger phrases developers would type. |
| `argument-hint` | No | — | Hint shown in the `/` menu when the skill is invoked as a slash command. Quote the value to prevent GitHub Markdown rendering issues. |
| `user-invocable` | No | — | Defaults to `true`. Set `false` to hide from the `/` menu while still allowing Copilot to auto-load it. |
| `disable-model-invocation` | No | — | Defaults to `false`. Set `true` to require explicit `/skill-name` invocation — blocks auto-loading entirely. |

Fields not in this list are ignored by VS Code but will not cause failures. This repository uses a `version` field as a repo convention for the skillpack manifest — it is not part of the spec.

---

## 5. Writing a skill that actually works

### The name-folder match is enforced by VS Code, not by git

This is the single most common breaking mistake. If your folder is `code-review` but your frontmatter says `name: pr-review`, VS Code silently skips the skill. No error. No warning. It simply does not appear.

Always set them together:

```bash
mkdir -p .github/skills/code-review
cat > .github/skills/code-review/SKILL.md << 'EOF'
---
name: code-review
description: ...
---
EOF
```

### Description quality determines auto-invocation

Copilot matches skills against the description field alone during the discovery phase. If your description is too narrow, the skill will not fire when it should.

**Weak description:**
```
Helps with code reviews.
```

**Strong description:**
```
Reviews code changes and produces structured feedback. Use this when asked
to review a PR, check a diff, give feedback on code quality, audit a change
set, identify risks in a branch, or summarize what changed before merging.
```

Enumerate the phrases a developer would actually type. The spec allows 1024 characters — use them.

### When to set `disable-model-invocation: true`

Use this for skills where accidental auto-invocation would be disruptive:

- Skills that scaffold files or directories (you want explicit opt-in)
- Skills that run terminal commands
- Skills that produce long-form output like release notes or changelogs
- Meta-skills that manage the skill catalog itself

Leave it at default for skills that analyze, review, or advise.

### Referencing supporting files

The skill body can reference files in the same directory using relative paths. Copilot loads them lazily — only when the instruction that references them is reached.

```markdown
## Setup

Follow the steps in [./setup-checklist.md](./setup-checklist.md) before running the scan.

## Script

Run the included validation script:

```bash
bash ./validate.sh
```
```

This keeps `SKILL.md` concise while making detailed supporting material available on demand.

### Body structure that performs well

The spec has no required body format. This pattern works reliably:

```markdown
## When to use this skill

Explicit list of scenarios. Tells Copilot exactly when the skill applies.

## Inputs

What context or files Copilot should look for before starting.

## Workflow

Numbered steps. Be specific. Vague instructions produce vague output.

## Output format

What the result should look like. Include an example if the format is non-obvious.

## Edge cases

What to do when inputs are missing, ambiguous, or out of scope.
```

---

## 6. Testing skills locally

### Verify discovery

1. Open VS Code in the workspace where your skill is stored
2. Open Copilot Chat (`Ctrl+Alt+I`)
3. Type `/` and look for your skill in the list
4. If it is missing: the folder name does not match the `name` field, or the skill is not in a recognized directory

### Verify auto-invocation

Type a prompt that matches phrases in your description. If Copilot loads the skill, you will see it referenced in the response. If it does not load, your description lacks enough matching phrases for that specific prompt. Add them.

### Invoke directly

Type `/your-skill-name` to bypass description matching and load the skill body directly. Useful for testing before description tuning is done.

### Debug with the Chat Customizations editor

Open the Command Palette (`Ctrl+Shift+P`) and run `Chat: Open Chat Customizations`. This panel shows all discovered skills and their invocation settings in one view — faster than inferring discovery from the `/` menu.

---

## 7. Packaging with skillpack.sh

`scripts/skillpack.sh` validates all skills, builds a distributable directory, writes a manifest, and produces a `tar.gz` archive suitable for attaching to a GitHub release.

### What the script does

1. Collects every top-level directory that contains a `SKILL.md`
2. Validates each skill against the rules listed below
3. Copies each skill directory into `dist/skillpack/skills/`
4. Copies the root `README.md` and `LICENSE` into `dist/skillpack/`
5. Writes a text or JSON manifest
6. Creates `dist/skillpack.tar.gz`

### Usage

```bash
# Validate only — no output produced, exits non-zero on any error
./scripts/skillpack.sh --validate

# Build with JSON manifest (recommended for releases)
./scripts/skillpack.sh --manifest json

# Build with text manifest
./scripts/skillpack.sh --manifest text

# Write to a custom output directory
./scripts/skillpack.sh --output /tmp/my-skillpack --manifest json
```

### Validation rules

The script hard-fails on any of these:

| Check | Why it fails |
|---|---|
| Missing `README.md` | Skill is undocumented |
| Missing frontmatter `---` markers | Malformed `SKILL.md` |
| Missing `name` field | Required by spec |
| Missing `description` field | Required by spec |
| Missing `version` field | Required by this repo's convention |
| `name` does not match folder name | Skill will not load in VS Code |
| `name` contains uppercase or non-hyphen characters | Violates spec character rules |
| `name` exceeds 64 characters | Violates spec length limit |
| `description` exceeds 1024 characters | Violates spec length limit |

The name-folder match check is the most important. It catches the class of bug where a skill is renamed or the frontmatter is edited without updating the other side.

### JSON manifest structure

```json
{
  "kind": "skillpack",
  "schema_version": 3,
  "generated_at": "2026-04-19T12:00:00Z",
  "skill_count": 4,
  "bundle_files": [
    {"path": "README.md", "sha256": "..."},
    {"path": "LICENSE", "sha256": "..."}
  ],
  "skills": [
    {
      "name": "git-autopilot",
      "version": "1.0.0",
      "path": "skills/git-autopilot",
      "description": "...",
      "readme": "skills/git-autopilot/README.md",
      "skill_file": "skills/git-autopilot/SKILL.md",
      "files": [
        {"path": "skills/git-autopilot/README.md", "sha256": "..."},
        {"path": "skills/git-autopilot/SKILL.md", "sha256": "..."}
      ]
    }
  ]
}
```

SHA-256 hashes are computed with `shasum`, `sha256sum`, or `openssl` — whichever is available first. Consumers can verify archive integrity against the manifest before installing skills.

---

## 8. CI with GitHub Actions

The repository ships two workflows under `.github/workflows/`.

### build-skillpack.yml

**Triggers:** push to `main`, pull requests, published releases, manual dispatch.

**What it does:**

1. Validates all skill directories with `--validate`
2. Builds the skillpack with `--manifest json`
3. Uploads `dist/skillpack/` as a workflow artifact (available for 90 days)
4. On published releases: attaches `skillpack.tar.gz` and `manifest.json` as release assets

The release asset attachment is what powers the download badges in the root README. The badge URLs point to `releases/latest/download/skillpack.tar.gz` and `releases/latest/download/manifest.json`, both populated by this step.

Every pull request triggers a validation and build so broken skills are caught before merging. The artifact upload means you can download and inspect the packaged output directly from the Actions run without publishing a release.

### create-release-tag.yml

**Triggers:** manual dispatch only.

Accepts a semantic version string and a target branch or commit SHA. Creates a git tag and publishes a GitHub release at that ref. Publishing the release then triggers `build-skillpack.yml`, which handles archive generation and asset attachment.

**End-to-end release flow:**

```
1. Run create-release-tag.yml manually (input: version, ref)
     ↓
2. Git tag created, GitHub release published
     ↓
3. build-skillpack.yml triggered by the release event
     ↓
4. Skillpack validated, built, and archived
     ↓
5. skillpack.tar.gz and manifest.json attached to the release
     ↓
6. Download badge URLs resolve to the new version
```

### Adapting the workflows to a new repository

Copy both workflow files into `.github/workflows/`. The only dependencies are `bash` and standard POSIX tools — no third-party Actions are used. The packaging script auto-detects a SHA-256 utility from `shasum`, `sha256sum`, and `openssl` in priority order.

If your skill directories are not at the repository root, adjust the `repo_root` derivation at the top of `skillpack.sh`, or use the `--output` flag to redirect build output.

---

## 9. Common mistakes

**Skill does not appear in the `/` menu**

The `name` frontmatter field does not match the folder name. They must be identical strings. Check both — it is easy to rename a folder in a refactor and forget to update the frontmatter, or to copy a skill template and forget to update the name inside it.

**Skill appears but never auto-invokes**

The description does not contain phrases that match your prompts. Read the description and ask whether someone who did not write it would use those exact words when asking Copilot for help. If not, add the phrases they would actually use.

**Validation passes locally but CI fails**

The script validates the working tree, not committed state. If you have unstaged or uncommitted edits to a `SKILL.md`, local validation passes against the dirty file while CI sees the last committed version. Commit before triggering the workflow.

**`argument-hint` renders as a broken table on GitHub**

YAML pipe characters in unquoted values are parsed as Markdown table syntax by GitHub's renderer. Quote the value:

```yaml
# Broken on GitHub
argument-hint: [branch] or file scope

# Correct
argument-hint: "[branch] optional branch name or file scope"
```

**Skill body references a file that does not exist in the directory**

Copilot attempts to load the referenced file and fails silently if it is missing. Keep all referenced files inside the skill directory and use relative paths.

**`disable-model-invocation: true` on a skill you want to auto-invoke**

With this flag set, the skill only responds to explicit `/skill-name` invocation. Remove the field or set it to `false` to restore auto-invocation.

---

## 10. Spec field reference

Complete frontmatter with all fields and their defaults:

```yaml
---
# Required — spec fields
name: my-skill                     # Lowercase, hyphens + digits only, max 64 chars.
                                   # Must match the folder name exactly.
description: >                     # Max 1024 chars.
  What this skill does and when    # Include all trigger phrases developers would type.
  to use it. Use this when asked
  to A, B, C, or D.

# Optional — spec fields
argument-hint: "[scope] hint"      # Text shown in the slash command input.
                                   # Quote the value to prevent rendering issues.
user-invocable: true               # Default true. false = hidden from / menu.
disable-model-invocation: false    # Default false. true = explicit invocation only.

# Optional — repo convention (not in spec)
version: 1.0.0                     # Used by skillpack.sh to populate the JSON manifest.
---
```

### Invocation behavior matrix

| `user-invocable` | `disable-model-invocation` | Appears in `/` menu | Copilot auto-loads |
|---|---|---|---|
| `true` (default) | `false` (default) | Yes | Yes |
| `false` | `false` (default) | No | Yes |
| `true` (default) | `true` | Yes | No |
| `false` | `true` | No | No |

---

## Further reading

- [VS Code Agent Skills documentation](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- [Agent Skills open standard](https://agentskills.io)
- [GitHub Docs: Adding agent skills for Copilot](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills)
- [github/awesome-copilot](https://github.com/github/awesome-copilot) — community skill catalog
- [anthropics/skills](https://github.com/anthropics/skills) — reference skills from Anthropic
