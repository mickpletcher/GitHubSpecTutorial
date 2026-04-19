# GitHub Copilot Agent Skills — Setup and Authoring Guide

This tutorial walks through the Agent Skills specification from scratch to a fully packaged and CI-deployed skillpack. It assumes you are comfortable with VS Code, git, and shell scripting. No hand-holding on basics.

---

## Contents

1. [What Agent Skills are and are not](#1-what-agent-skills-are-and-are-not)
2. [Prerequisites](#2-prerequisites)
3. [Skill storage locations](#3-skill-storage-locations)
4. [SKILL.md specification](#4-skillmd-specification)
5. [How Copilot loads a skill — the three-phase process](#5-how-copilot-loads-a-skill--the-three-phase-process)
6. [Writing a skill that actually works](#6-writing-a-skill-that-actually-works)
7. [Testing skills locally](#7-testing-skills-locally)
8. [Packaging with skillpack.sh](#8-packaging-with-skillpacksh)
9. [CI with GitHub Actions](#9-ci-with-github-actions)
10. [Common mistakes](#10-common-mistakes)
11. [Spec field reference](#11-spec-field-reference)
12. [Project 1: Greenfield](#12-project-1-greenfield)
13. [Project 2: Retrofit](#13-project-2-retrofit)

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

Type `/` in the Copilot Chat input. Every discovered skill appears in the menu alongside prompt files. If a skill is missing, the folder name or `name` frontmatter field is mismatched — see [Common mistakes](#10-common-mistakes).

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

## 5. How Copilot loads a skill — the three-phase process

Before writing a skill, you need to understand exactly what Copilot does with it. There are three distinct phases. Understanding them explains every authoring decision that follows.

### Phase 1 — Discovery

When you open Copilot Chat, VS Code scans all skill directories and reads only the frontmatter of each `SKILL.md`. It extracts `name` and `description` and builds an index. This is the only thing that runs at startup — the skill body is not loaded.

When you type a prompt, Copilot runs a semantic match against every description in that index. If your prompt matches a description, the skill moves to Phase 2. If it does not match, the skill is never loaded.

**Implication:** the description field is not documentation — it is a matching surface. Every phrase in it is a potential match target. A description that accurately describes the skill but does not contain the phrases developers actually type will never trigger auto-invocation.

### Phase 2 — Instruction loading

When a skill is matched, Copilot loads the full body of `SKILL.md` into context. The instructions, workflow steps, rules, and examples all become available to Copilot for that conversation turn.

This is why the body can be long and detailed without performance cost. It is only loaded when relevant.

**Implication:** write the body for correctness and completeness, not for brevity. A detailed step-by-step workflow is better than a vague high-level summary. Copilot follows explicit instructions more reliably than implicit ones.

### Phase 3 — Resource access

As Copilot works through the skill body, it encounters references to supporting files — scripts, templates, checklists, examples. It loads those files only when the instruction referencing them is reached.

This lazy loading means you can have a skill that references a 500-line template file without that template consuming context on every turn. It is loaded precisely when needed and not before.

**Implication:** move supporting material into separate files in the skill directory. Reference them in the skill body at the point where they are needed. This keeps the `SKILL.md` body focused on instructions while making detailed reference material available on demand.

### What this means for authoring

Every authoring decision in the next section maps to one of these three phases:

| Decision | Phase it affects |
|---|---|
| Description phrase selection | Phase 1 — discovery |
| `disable-model-invocation` setting | Phase 1 — blocks automatic match |
| Body structure and detail level | Phase 2 — instruction quality |
| Supporting file references | Phase 3 — resource access |

A skill that fails at Phase 1 never loads, regardless of how good the body is. A skill that passes Phase 1 but has a vague body produces inconsistent results. A skill that passes Phase 1 and 2 but keeps all content inline instead of using supporting files wastes context on every invocation.

---

## 6. Writing a skill that actually works

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

Copilot matches your prompt against the `description` field during Phase 1 discovery. The match is semantic, not exact — but it heavily favors phrases that appear verbatim in the description. A description that accurately describes what a skill does but does not contain the words a developer types when asking for help will miss most invocations.

**Weak description — correct but untriggered:**
```
Reviews code changes and produces structured feedback.
```
This is accurate. A developer asking "review my PR" may not trigger it because "PR" and "review" appear in the description but "check my diff", "give me feedback", "is this safe to merge" — common phrasings — do not.

**Strong description — same skill, broad match surface:**
```
Reviews staged or committed code changes and produces structured feedback organized
by severity. Use this when asked to review a PR, check a diff, give feedback on
code quality, audit a change set, identify risks in a branch, summarize what
changed before merging, or check if code is safe to merge.
```

The spec allows 1024 characters. The strong version above uses 328. There is room for more.

### How to derive description phrases

The instinct is to describe what the skill does. The correct approach is to list how a developer asks for what the skill does. These are different questions.

**Step 1 — State the outcome the skill produces:**
"Produces a structured code review with critical issues and a verdict."

**Step 2 — List every way a developer might ask for that outcome:**
- "review my PR"
- "check this diff"
- "give me feedback on my changes"
- "is this safe to merge"
- "audit this change set"
- "what are the risks in this branch"
- "summarize what changed"

**Step 3 — Add synonyms for the action and the subject:**
- Action synonyms: review, check, audit, inspect, analyze, look at
- Subject synonyms: PR, pull request, diff, change set, branch, code, changes

**Step 4 — Assemble into the description field:**
Combine the outcome statement with the phrase list. Start with what the skill does. Follow with "Use this when asked to..." and list every phrase from Step 2 and Step 3.

**Step 5 — Test against your own usage:**
For three days, every time you would naturally ask Copilot for something this skill handles, write down exactly what you typed. If any of those phrases are not in the description, add them.

### When to set `disable-model-invocation: true`

This field controls Phase 1 behavior. When set to `true`, Copilot will never auto-load the skill based on description matching — it only loads when the user types `/skill-name` explicitly.

Use it when accidental auto-invocation would be disruptive or wrong:

**Good candidates for `disable-model-invocation: true`:**
- Skills that scaffold files or create directories — you want deliberate opt-in, not accidental creation
- Skills that run terminal commands — unintended execution is a real risk
- Skills that produce long-form output (release notes, changelogs) — these consume significant context and should be intentional
- Meta-skills that manage the skill catalog — you never want these loading during normal coding conversations

**Good candidates to leave at default (auto-invocable):**
- Code review skills — the trigger is obvious ("review my changes") and the cost of accidental invocation is low
- Git commit skills — the trigger is unambiguous and the skill always asks for confirmation before acting
- Analysis and advisory skills — read-only, no side effects, safe to auto-load

**The `user-invocable: false` combination:**

Setting `user-invocable: false` with `disable-model-invocation` left at default creates a background knowledge skill — Copilot can auto-load it when relevant but it does not appear in the `/` menu. Use this for skills that provide context or guidelines Copilot should apply automatically without surfacing as a user command. Example: a skill that encodes your team's code review criteria, loaded automatically whenever code is discussed but not something a developer would invoke directly.

### Invocation control scenarios

| What you want | Settings |
|---|---|
| Skill auto-invokes and appears in `/` menu | both fields omitted (default) |
| Background knowledge — auto-invokes, hidden from `/` menu | `user-invocable: false` |
| Explicit-only — visible in `/` menu, never auto-invokes | `disable-model-invocation: true` |
| Disabled | both set |

### Referencing supporting files — end-to-end example

The skill body can reference files inside the skill directory using relative paths. Copilot loads them at Phase 3 — only when the instruction referencing them is executed.

Consider a skill that enforces a PR checklist. The checklist is long and detailed. Instead of embedding it in the skill body, you put it in a separate file and reference it:

**Directory structure:**
```
.github/skills/code-review/
├── SKILL.md
├── review-checklist.md     ← supporting file
└── README.md
```

**Reference in SKILL.md body:**
```markdown
## Review process

Work through every item in the [review checklist](./review-checklist.md) before
producing the verdict. Do not skip items marked as required.
```

**Contents of `review-checklist.md`:**
```markdown
## Required checks
- [ ] No hardcoded credentials or secrets
- [ ] Error paths are handled and tested
- [ ] No N+1 query patterns introduced
- [ ] Input validation present on all user-facing routes
- [ ] No commented-out code committed

## Advisory checks
- [ ] Function names describe behavior, not implementation
- [ ] No magic numbers without named constants
- [ ] Tests cover the new behavior, not just the happy path
```

When Copilot reaches the instruction that references `./review-checklist.md`, it loads the file. The checklist is now in context for that turn. On turns where the code-review skill is not invoked, the checklist never loads.

This pattern scales cleanly. A scaffold skill might reference a 200-line template file. A migration skill might reference a SQL patterns guide. None of those files consume context unless the skill that references them is active.

### Complete worked example — from blank directory to working skill

This example builds a complete `db-migration` skill from scratch, showing every decision.

**Step 1 — Create the directory:**
```bash
mkdir -p .github/skills/db-migration
```

**Step 2 — Derive the description phrases:**
Outcome: "Creates a database migration file following the project's naming convention and schema change patterns."

Developer phrases: "add a migration", "create a migration", "add a database migration", "write a migration for...", "schema change", "add a column", "create a table migration", "db migration"

**Step 3 — Write the SKILL.md:**
```markdown
---
name: db-migration
version: 1.0.0
description: Creates a new database migration file following project conventions. Use this when asked to add a migration, create a database migration, write a migration for a schema change, add a column, create a table, or make any database schema modification.
argument-hint: "[description] brief description of the schema change (e.g., add-users-email-index)"
disable-model-invocation: true
---

# Database Migration

Creates a new migration file following this project's naming convention and schema change patterns.

## When to use this skill

Use this skill when asked to:
- Add a new migration
- Create a schema change
- Add or remove columns from a table
- Create or drop a table
- Add indexes or constraints

## Required input

A description of the schema change in kebab-case (e.g., `add-users-email-index`, `create-products-table`). If not provided, ask before creating any files.

## Migration naming convention

```
{timestamp}_{description}.ts

Example: 20260419_add-users-email-index.ts
```

Generate the timestamp with: `date +%Y%m%d`

## File location

```
db/migrations/{timestamp}_{description}.ts
```

## Template

Follow the pattern in [./migration-template.ts](./migration-template.ts).

## Rules

- Every migration must have both an `up()` and a `down()` method
- The `down()` method must exactly reverse what `up()` does
- Never modify an existing migration file — create a new one
- Run `npm run db:migrate:validate` after creating the file to check syntax

## Edge cases

- If `db/migrations/` does not exist, create it before creating the file
- If a migration with this timestamp already exists, increment the timestamp by 1 second
- If the description is missing, stop and ask before creating anything
```

**Step 4 — Add the supporting template file:**
```bash
touch .github/skills/db-migration/migration-template.ts
```

Write the template — the actual content depends on your ORM. The key point is that it lives in the skill directory and loads only when Copilot reaches the instruction that references it.

**Step 5 — Write README.md:**
```markdown
# db-migration

Creates database migration files following the project's naming convention.

## Usage

Type `/db-migration` in Copilot Chat (explicit invocation only — this skill does not auto-invoke).

## What it creates

A timestamped migration file under `db/migrations/` following the project template.
Always includes both `up()` and `down()` methods.
```

**Step 6 — Validate:**
```bash
./scripts/skillpack.sh --validate
# Expected: Validated 1 skill(s): db-migration
```

**Step 7 — Test:**
Type `/db-migration add-users-email-index` in Copilot Chat. The skill loads, asks for confirmation of the file path and template, and creates the migration. Adjust the body if the output does not match your project's conventions.

### Skill conflict avoidance

When two skills have overlapping descriptions, Copilot may load both or choose unpredictably. This matters as your catalog grows past four or five skills.

**Signs of description collision:**
- Two skills both load when you type a prompt that should only match one
- A skill loads unexpectedly when a different skill's trigger phrase is used

**How to avoid collisions:**

Use distinct anchor phrases in each description. The more specific the trigger, the narrower the match surface.

Instead of two skills that both say "review code":
```
# Skill A — too broad
Use this when asked to review code or check a change.

# Skill B — too broad
Use this when reviewing code or checking files.
```

Give each a distinct primary trigger:
```
# Skill A — code-review
Use this when asked to review a PR, check a diff, or audit staged changes.
Produces structured feedback with a merge verdict.

# Skill B — security-review
Use this when asked to security review code, check for vulnerabilities,
audit for secrets, or assess code for security risks.
```

The primary trigger phrase anchors each skill to its intent. Overlap in secondary phrases is acceptable.

---

## 7. Testing skills locally

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

## 8. Packaging with skillpack.sh

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

## 9. CI with GitHub Actions

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

## 10. Common mistakes

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

## 11. Spec field reference

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

## 12. Project 1: Greenfield

See **[Project1-Greenfield.md](./Project1-Greenfield.md)**.

Use this project if you are starting a new repository. The file is a GitHub Copilot Agent mode prompt — paste it directly into Copilot Chat in Agent mode and it builds the complete skills infrastructure for a new project from the first commit. It collects your project name, description, stack, and GitHub username before creating anything.

**Use Project 1 when:**
- The repository does not exist yet
- You want skills wired in before any application code is written
- You want the full four-skill system (feature scaffold, code review, git commit, skill scaffold) as a starting point
- You want the skillpack validator and CI workflow set up automatically

After the prompt completes, read the **Tutorial** section at the end of `Project1-Greenfield.md` for a walkthrough of every decision the prompt made and how to customize the output for your stack.

---

## 13. Project 2: Retrofit

See **[Project2-Retrofit.md](./Project2-Retrofit.md)**.

Use this project if you have an existing populated repository. The file is a GitHub Copilot Agent mode prompt — paste it directly into Copilot Chat in Agent mode and it audits the codebase, identifies skill candidates from commit history and existing documentation, and adds skills without modifying any application code.

**Use Project 2 when:**
- The repository already has application code, history, and conventions
- You want skills extracted from patterns that are already present in the codebase
- You want all changes isolated in a single PR that is easy to review and revert

After the prompt completes, read the **Tutorial** section at the end of `Project2-Retrofit.md` for a walkthrough of the audit methodology, the extraction decisions, and how to maintain the skills as the codebase evolves.

---

## Further reading

- [VS Code Agent Skills documentation](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- [Agent Skills open standard](https://agentskills.io)
- [GitHub Docs: Adding agent skills for Copilot](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills)
- [github/awesome-copilot](https://github.com/github/awesome-copilot) — community skill catalog
- [anthropics/skills](https://github.com/anthropics/skills) — reference skills from Anthropic
