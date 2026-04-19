# Copilot Task: Greenfield Project — GitHub Copilot Agent Skills Setup

Paste this prompt into GitHub Copilot **Agent mode** in VS Code. Agent mode is required — this prompt creates files, runs terminal commands, and validates the result. Ask mode will not work.

Before starting, open a new empty folder in VS Code. This prompt assumes an empty workspace. Run it once per project.

---

## Context

You are setting up a new software project with GitHub Copilot Agent Skills wired in from the first commit. Agent Skills are folders of instructions that Copilot loads when relevant, making repeatable workflows available to every developer who clones this repository. This prompt builds the full skill infrastructure — directory structure, skill files, packaging script, CI workflow, and a human-readable setup guide — before any application code is written.

Your goal is to produce a repository that is ready to push to GitHub and immediately useful to any Copilot user who opens it.

---

## Step 1 — Collect project information

Before creating any files, ask the user for the following. Do not proceed until you have all four answers.

1. **Project name** — used for the root `README.md` heading and the `package.json` or project manifest if one exists. Use kebab-case for directory and file naming (e.g., `my-api-service`).
2. **Project description** — one sentence describing what this project does. Used in `README.md` and `instructions.md`.
3. **Primary language or stack** — used to make the feature-scaffold skill concrete (e.g., TypeScript/Node, Python/FastAPI, C#/.NET, Go). If the user says "generic," use language-agnostic examples.
4. **GitHub username or org** — used to pre-fill the remote URL in `instructions.md`.

Store the answers as variables and use them throughout. Do not use placeholder text like `[PROJECT_NAME]` in any output file — substitute the real values everywhere.

---

## Step 2 — Create the directory structure

Run the following commands in the VS Code terminal. Create every directory before writing any files.

```bash
# Core repo directories
mkdir -p .github/skills/feature-scaffold
mkdir -p .github/skills/code-review
mkdir -p .github/skills/git-commit
mkdir -p .github/skills/skill-scaffold
mkdir -p .github/workflows
mkdir -p scripts
mkdir -p docs
```

Confirm all directories exist before proceeding:

```bash
find .github scripts docs -type d | sort
```

Expected output should show all eight directories. If any are missing, create them before continuing.

---

## Step 3 — Create `.gitignore`

Create `.gitignore` with sensible defaults for the project's language. At minimum include:

```
# Build output
dist/
build/
out/

# Skillpack output — generated, not committed
dist/skillpack/
dist/skillpack.tar.gz

# OS
.DS_Store
Thumbs.db

# Editor
.vscode/settings.json
*.swp
```

Add language-specific entries based on the stack the user provided in Step 1. For Node: `node_modules/`. For Python: `__pycache__/`, `*.pyc`, `.venv/`. For .NET: `bin/`, `obj/`. For Go: vendor-specific patterns.

---

## Step 4 — Create `README.md`

Create the root `README.md` using the project name and description collected in Step 1.

```markdown
# [PROJECT_NAME]

[PROJECT_DESCRIPTION]

---

## Copilot Skills

This repository includes GitHub Copilot Agent Skills that automate repeatable development workflows. Skills are stored in `.github/skills/` and load automatically when Copilot detects a matching prompt.

| Skill | Slash Command | What it does |
|---|---|---|
| feature-scaffold | `/feature-scaffold` | Scaffolds a new feature with the project's standard file structure |
| code-review | `/code-review` | Reviews staged changes and produces structured feedback |
| git-commit | `/git-commit` | Generates a Conventional Commits message and commits |
| skill-scaffold | `/skill-scaffold` | Creates a new skill folder with valid SKILL.md and README |

Type `/` in Copilot Chat to invoke any skill directly, or describe what you want and Copilot will load the relevant skill automatically.

## Getting started

See [instructions.md](./instructions.md) for setup, skill usage, and how to add new skills to this project.

## Repository structure

```
.github/
  skills/          # Copilot Agent Skills
  workflows/       # GitHub Actions — skill validation and packaging
docs/              # Project documentation
scripts/           # Developer tooling — skillpack.sh for validation and packaging
instructions.md    # How to use Copilot Skills in this project
```
```

---

## Step 5 — Create the four SKILL.md files

Write each `SKILL.md` with the actual project language substituted into examples. These are production-quality skills, not placeholders. Every field is populated. Every step is specific.

### `.github/skills/feature-scaffold/SKILL.md`

Customize the file structure, templates, and registration steps for the language the user provided. The example below uses TypeScript/Express — adapt it to the actual stack.

```markdown
---
name: feature-scaffold
version: 1.0.0
description: Scaffolds a new feature with the project's standard file structure. Use this when asked to add a new feature, create a new module, scaffold a resource, add an endpoint, or create a new component. Generates all required files and registers the feature in the appropriate entry point.
argument-hint: "[feature-name] the name of the feature or resource to create (e.g., user, order, report)"
---

# Feature Scaffold

Creates the files required for a new feature following this project's established structure.

## When to use this skill

Use this skill when asked to:
- Add a new feature, module, or resource
- Create a new endpoint or component
- Scaffold files for a new area of functionality

## Required input

A feature name in kebab-case (e.g., `user-profile`, `order-history`, `payment-method`). If the user did not provide one, ask before creating any files.

## File structure

For a feature named `{feature}`, create these files:

```
src/features/{feature}/
├── {feature}.routes.ts       # Route definitions
├── {feature}.controller.ts   # Request handling — no business logic
├── {feature}.service.ts      # Business logic and data access
├── {feature}.types.ts        # Type definitions and interfaces
└── {feature}.test.ts         # Unit tests
```

## Route file template

```typescript
import { Router } from 'express';
import { {Feature}Controller } from './{feature}.controller';

const router = Router();
const controller = new {Feature}Controller();

router.get('/', controller.getAll.bind(controller));
router.get('/:id', controller.getById.bind(controller));
router.post('/', controller.create.bind(controller));
router.put('/:id', controller.update.bind(controller));
router.delete('/:id', controller.remove.bind(controller));

export { router as {feature}Router };
```

## Controller rules

- Thin layer only — delegate all logic to the service
- Validate request shape at this layer, not in the service
- Return consistent response shape: `{ data, error, meta }`
- Do not import database clients directly

## Service rules

- All business logic lives here
- Return typed results, never raw database rows
- Handle errors by throwing typed exceptions — let the controller format the response

## Registration

After creating the files, open `src/app.ts` and register the new router:

```typescript
import { {feature}Router } from './features/{feature}/{feature}.routes';
app.use('/api/{feature}s', {feature}Router);
```

## Edge cases

- If a feature with this name already exists, stop and report the conflict. Do not overwrite existing files.
- If the feature name contains spaces, convert to kebab-case before creating files.
- If `src/app.ts` does not exist or uses a different entry point, ask the user where to register the route before adding it.
```

### `.github/skills/feature-scaffold/README.md`

```markdown
# feature-scaffold

Scaffolds new features following the project's route/controller/service/types/test structure.

## Usage

- Type `/feature-scaffold` in Copilot Chat, or
- Ask: "add a new feature for user profiles"

## What it creates

Five files under `src/features/{feature}/`: routes, controller, service, types, and tests.
Registers the new router in `src/app.ts`.

## Notes

Adapt the file templates in `SKILL.md` if the project structure changes.
Keep this skill updated when the team adopts new patterns.
```

### `.github/skills/code-review/SKILL.md`

```markdown
---
name: code-review
version: 1.0.0
description: Reviews staged or committed code changes and produces structured feedback organized by severity. Use this when asked to review a PR, check a diff, review code, give feedback on changes, audit a change set, identify risks, check for issues, or summarize what changed before merging.
argument-hint: "[scope] optional file, module, or concern to focus the review on"
---

# Code Review

Produces a structured review of the current change set grounded in the actual diff.

## When to use this skill

Use this skill when asked to:
- Review a pull request or branch
- Give feedback on staged changes
- Check code quality before committing
- Identify risks or missing test coverage

## Process

### Step 1 — Read the diff

Run `git diff --staged` if changes are staged. If nothing is staged, run `git diff HEAD` to review the last commit. If neither has output, ask the user which files or commits to review.

### Step 2 — Identify the intent

Determine what the change is trying to accomplish before evaluating how it does it. A change that is correct for its intent but wrong in approach gets a different note than a change with flawed intent.

### Step 3 — Produce structured output

```
## Code Review

### Summary
[One sentence: what does this change do?]

### Critical issues
[Issues that should block merge — correctness, security, data integrity]
- [file:line] description of the issue and why it matters

### Advisory
[Suggestions that improve quality but do not block merge]
- [file:line] suggestion with rationale

### Missing coverage
[Behaviors that are changed but not tested]
- [description of what should be tested]

### Verdict
[ ] Approved
[ ] Approved with minor changes
[ ] Changes required before merge
```

## Rules

- Ground every comment in the actual diff. Do not raise issues about unchanged code.
- Reference file name and line number for every critical issue and advisory note.
- Do not flag style issues handled by the project linter or formatter.
- If the change is clean, say so. An empty critical issues section is a valid review.
- Do not approve changes that lack tests for new behavior.
```

### `.github/skills/code-review/README.md`

```markdown
# code-review

Reviews staged or committed changes and produces structured feedback.

## Usage

- Type `/code-review` in Copilot Chat, or
- Ask: "review my changes" or "check this diff"

## Output

Structured review with: summary, critical issues, advisories, missing coverage, and verdict.

## Notes

The skill reads the actual diff. Stage your changes before invoking for best results.
```

### `.github/skills/git-commit/SKILL.md`

```markdown
---
name: git-commit
version: 1.0.0
description: Generates a Conventional Commits message from staged changes, presents it for confirmation, and commits to the current branch. Use this when asked to commit changes, write a commit message, save work to git, stage and commit files, or generate a commit.
argument-hint: "[files] optional specific files to stage before committing"
---

# Git Commit

Generates a Conventional Commits message from the staged diff, confirms with the user, then commits.

## When to use this skill

Use this skill when asked to:
- Commit staged or unstaged changes
- Generate a commit message
- Save work to git

## Process

### Step 1 — Assess state

Run `git status` to identify staged, unstaged, and untracked files.

### Step 2 — Stage files

If the user specified files, stage those only:
```bash
git add [files]
```

If the user said "everything" or nothing was specified and there are unstaged changes, stage all:
```bash
git add -A
```

If files are already staged and no additional files were specified, skip this step.

### Step 3 — Read the diff

Run `git diff --staged` to read what will be committed. Use this output — not filenames — to write the commit message.

### Step 4 — Generate the message

Use the Conventional Commits format:

```
type(scope): short description

[optional body — include when the change is non-obvious or has side effects]
```

**Type selection:**

| Type | When to use |
|---|---|
| feat | New capability or behavior |
| fix | Bug or broken behavior corrected |
| docs | Documentation only |
| refactor | Code restructured, behavior unchanged |
| perf | Performance improvement |
| test | Tests added or updated |
| chore | Build, deps, tooling |
| ci | CI/CD pipeline |
| style | Formatting only, no logic change |

**Scope:** the affected module, feature, or area of the codebase. Omit if the change is broad.

**Short description:** imperative mood, lowercase, no period, 72 characters or less.

### Step 5 — Present for confirmation

Show the message before committing:

```
Proposed commit:

  type(scope): short description

  [body if applicable]

Commit? (yes to proceed, or provide changes)
```

Wait for confirmation. Revise if the user requests changes. Do not commit until confirmed.

### Step 6 — Commit

```bash
git commit -m "type(scope): short description"
```

For messages with a body:
```bash
git commit -m "type(scope): short description" -m "body text"
```

### Step 7 — Report result

After a successful commit, show:
- The commit hash (short form)
- The branch name
- Whether the branch has an upstream set

Do not push. Committing and pushing are separate decisions.

## Edge cases

- **Nothing to commit:** report the clean state and stop
- **Merge conflicts present:** stop immediately and tell the user to resolve conflicts first
- **Detached HEAD:** warn the user and do not commit until they are on a named branch
- **Many unrelated changes:** flag this and recommend splitting into multiple commits before proceeding
```

### `.github/skills/git-commit/README.md`

```markdown
# git-commit

Generates a Conventional Commits message from staged changes and commits after confirmation.

## Usage

- Type `/git-commit` in Copilot Chat, or
- Ask: "commit my changes" or "write a commit message"

## What it does

Reads the staged diff, generates a typed commit message, asks for confirmation, then commits. Does not push.

## Format

Follows Conventional Commits: `type(scope): description`
```

### `.github/skills/skill-scaffold/SKILL.md`

```markdown
---
name: skill-scaffold
version: 1.0.0
description: Creates a new GitHub Copilot skill folder with a valid SKILL.md and README.md. Use this when asked to add a new skill, create a skill, scaffold a Copilot skill, or add a repeatable workflow to the skill catalog.
argument-hint: "[skill-name] the name of the new skill in kebab-case"
disable-model-invocation: true
---

# Skill Scaffold

Creates a new skill folder following the project's skill structure and conventions.

## When to use this skill

Use this skill when asked to:
- Add a new skill to this project
- Create a new Copilot workflow
- Scaffold a skill folder

## Required input

A skill name in kebab-case (e.g., `db-migration`, `api-test`, `env-setup`). If not provided, ask before creating any files.

## What this skill creates

```
.github/skills/{skill-name}/
├── SKILL.md     # Machine-readable skill definition with full frontmatter
└── README.md    # Human-readable documentation
```

## SKILL.md template

```markdown
---
name: {skill-name}
version: 1.0.0
description: [What this skill does and when to use it. Include the natural-language phrases a developer would type to trigger it.]
argument-hint: "[optional argument hint shown in the slash command menu]"
---

# [Skill Title]

## When to use this skill

## Process

## Edge cases
```

## README.md template

```markdown
# {skill-name}

[One sentence: what does this skill do?]

## Usage

- Type `/{skill-name}` in Copilot Chat, or
- Ask: "[example natural language prompt]"

## Notes
```

## Rules

- The `name` field must match the folder name exactly — VS Code will not load the skill if they differ
- The `description` must contain the phrases a developer would type, not just what the skill does
- Do not create files with placeholder text — fill in the description before saving
- After creating the files, run `./scripts/skillpack.sh --validate` to confirm the skill passes validation
- Update the skills table in `README.md` to include the new skill

## Edge cases

- If a skill with this name already exists, stop and report the conflict
- If the name contains uppercase letters or spaces, convert to kebab-case before creating the folder
```

### `.github/skills/skill-scaffold/README.md`

```markdown
# skill-scaffold

Creates new Copilot Agent Skill folders following this project's conventions.

## Usage

Type `/skill-scaffold` in Copilot Chat (explicit invocation only — this skill does not auto-invoke).

## What it creates

Two files under `.github/skills/{skill-name}/`: a populated `SKILL.md` and a `README.md`.
Reminds you to update the skills table in the root `README.md`.

## Notes

After creating a skill with this tool, run `./scripts/skillpack.sh --validate` to confirm
the new skill passes all validation checks before committing.
```

---

## Step 6 — Create `scripts/skillpack.sh`

Create `scripts/skillpack.sh` with the following content exactly. Make it executable after writing.

```bash
#!/usr/bin/env bash

set -euo pipefail

script_dir=$(cd "$(dirname "$0")" && pwd)
repo_root=$(cd "$script_dir/.." && pwd)
default_output_dir="$repo_root/dist/skillpack"
output_dir="$default_output_dir"
manifest_format="text"
validate_only=false

usage() {
cat <<'EOF'
Usage:
  ./scripts/skillpack.sh [--output DIR] [--manifest text|json] [--validate]

Options:
  --output DIR      Write the skillpack to DIR.
  --manifest FORMAT Manifest format: text or json.
  --validate        Validate skill folders and exit without packaging.
  --help            Show this help text.
EOF
}

json_escape() {
    local value="$1"
    value=${value//\\/\\\\}
    value=${value//\"/\\\"}
    value=${value//$'\n'/\\n}
    value=${value//$'\r'/\\r}
    value=${value//$'\t'/\\t}
    printf '%s' "$value"
}

trim_quotes() {
    local value="$1"
    if [[ "$value" == \"*\" ]]; then
        value=${value:1:-1}
    elif [[ "$value" == \'*\' ]]; then
        value=${value:1:-1}
    fi
    printf '%s' "$value"
}

file_sha256() {
    local file_path="$1"
    if command -v shasum >/dev/null 2>&1; then
        shasum -a 256 "$file_path" | awk '{print $1}'
        return
    fi
    if command -v sha256sum >/dev/null 2>&1; then
        sha256sum "$file_path" | awk '{print $1}'
        return
    fi
    if command -v openssl >/dev/null 2>&1; then
        openssl dgst -sha256 -r "$file_path" | awk '{print $1}'
        return
    fi
    echo "No SHA-256 tool available" >&2
    exit 1
}

extract_frontmatter_value() {
    local file_path="$1"
    local key="$2"
    local value
    value=$(sed -n "s/^$key:[[:space:]]*//" "$file_path" | head -n 1)
    trim_quotes "$value"
}

skill_dirs=()

collect_skills() {
    skill_dirs=()
    local skills_root="$repo_root/.github/skills"
    if [[ ! -d "$skills_root" ]]; then
        echo "No .github/skills directory found under $repo_root" >&2
        exit 1
    fi
    for entry in "$skills_root"/*/; do
        if [[ -d "$entry" && -f "${entry}SKILL.md" ]]; then
            skill_dirs+=("$entry")
        fi
    done
}

validate_skills() {
    local skill_dir skill_name declared_name name_length desc_length
    collect_skills

    if [[ ${#skill_dirs[@]} -eq 0 ]]; then
        echo "No skill directories found under .github/skills/" >&2
        exit 1
    fi

    for skill_dir in "${skill_dirs[@]}"; do
        skill_name=$(basename "$skill_dir")

        if [[ ! -f "${skill_dir}README.md" ]]; then
            echo "Missing README.md for skill: $skill_name" >&2
            exit 1
        fi
        if ! grep -q '^---$' "${skill_dir}SKILL.md"; then
            echo "Missing YAML frontmatter markers in $skill_name/SKILL.md" >&2
            exit 1
        fi
        if ! grep -q '^name:' "${skill_dir}SKILL.md"; then
            echo "Missing name field in $skill_name/SKILL.md" >&2
            exit 1
        fi
        if ! grep -q '^description:' "${skill_dir}SKILL.md"; then
            echo "Missing description field in $skill_name/SKILL.md" >&2
            exit 1
        fi
        if ! grep -q '^version:' "${skill_dir}SKILL.md"; then
            echo "Missing version field in $skill_name/SKILL.md" >&2
            exit 1
        fi

        declared_name=$(extract_frontmatter_value "${skill_dir}SKILL.md" "name")
        if [[ "$declared_name" != "$skill_name" ]]; then
            echo "Name mismatch in $skill_name/SKILL.md: frontmatter declares '$declared_name' but folder is '$skill_name'" >&2
            echo "VS Code will not load this skill until these match." >&2
            exit 1
        fi

        if [[ ! "$declared_name" =~ ^[a-z0-9-]+$ ]]; then
            echo "Invalid name '$declared_name' in $skill_name/SKILL.md: must be lowercase letters, digits, and hyphens only" >&2
            exit 1
        fi

        name_length=${#declared_name}
        if [[ $name_length -gt 64 ]]; then
            echo "Name too long in $skill_name/SKILL.md: $name_length characters (max 64)" >&2
            exit 1
        fi

        desc_length=$(extract_frontmatter_value "${skill_dir}SKILL.md" "description" | wc -c)
        if [[ $desc_length -gt 1024 ]]; then
            echo "Description too long in $skill_name/SKILL.md: $desc_length characters (max 1024)" >&2
            exit 1
        fi
    done

    echo "Validated ${#skill_dirs[@]} skill(s): $(IFS=', '; echo "${skill_dirs[*]##*/}")" | tr -d '/'
}

write_json_manifest() {
    local manifest_file="$output_dir/manifest.json"
    local generated_at skill_dir skill_name skill_version skill_description
    local skill_readme_hash skill_file_hash first=true

    generated_at=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

    {
        printf '{\n'
        printf '  "kind": "skillpack",\n'
        printf '  "schema_version": 1,\n'
        printf '  "generated_at": "%s",\n' "$generated_at"
        printf '  "skill_count": %s,\n' "${#skill_dirs[@]}"
        printf '  "skills": [\n'

        for skill_dir in "${skill_dirs[@]}"; do
            skill_name=$(basename "$skill_dir")
            skill_version=$(extract_frontmatter_value "${skill_dir}SKILL.md" "version")
            skill_description=$(extract_frontmatter_value "${skill_dir}SKILL.md" "description")
            skill_readme_hash=$(file_sha256 "${skill_dir}README.md")
            skill_file_hash=$(file_sha256 "${skill_dir}SKILL.md")

            if [[ "$first" == true ]]; then
                first=false
            else
                printf ',\n'
            fi

            printf '    {"name": "%s", "version": "%s", "description": "%s", "skill_file_sha256": "%s", "readme_sha256": "%s"}' \
                "$(json_escape "$skill_name")" \
                "$(json_escape "$skill_version")" \
                "$(json_escape "$skill_description")" \
                "$skill_file_hash" \
                "$skill_readme_hash"
        done

        printf '\n  ]\n'
        printf '}\n'
    } > "$manifest_file"
}

while [[ $# -gt 0 ]]; do
    case "$1" in
        --output) output_dir="$2"; shift 2 ;;
        --output=*) output_dir="${1#*=}"; shift ;;
        --manifest) manifest_format="$2"; shift 2 ;;
        --manifest=*) manifest_format="${1#*=}"; shift ;;
        --validate) validate_only=true; shift ;;
        --help) usage; exit 0 ;;
        -*) echo "Unknown option: $1" >&2; usage >&2; exit 1 ;;
        *) output_dir="$1"; shift ;;
    esac
done

if [[ "$manifest_format" != "text" && "$manifest_format" != "json" ]]; then
    echo "Invalid manifest format: $manifest_format" >&2
    exit 1
fi

validate_skills

if [[ "$validate_only" == true ]]; then
    exit 0
fi

skills_dir="$output_dir/skills"
rm -rf "$output_dir"
mkdir -p "$skills_dir"

for skill_dir in "${skill_dirs[@]}"; do
    skill_name=$(basename "$skill_dir")
    cp -R "$skill_dir" "$skills_dir/$skill_name"
done

if [[ "$manifest_format" == "json" ]]; then
    write_json_manifest
fi

tar -czf "$output_dir.tar.gz" -C "$(dirname "$output_dir")" "$(basename "$output_dir")"

echo "Skillpack written to $output_dir"
echo "Archive written to $output_dir.tar.gz"
```

After writing the file, make it executable:

```bash
chmod +x scripts/skillpack.sh
```

---

## Step 7 — Create `.github/workflows/validate-skills.yml`

```yaml
name: Validate Skills

on:
  push:
    branches: [main]
  pull_request:
    paths:
      - '.github/skills/**'
      - 'scripts/skillpack.sh'

jobs:
  validate:
    name: Validate Agent Skills
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Validate skill directories
        run: ./scripts/skillpack.sh --validate

      - name: Build skillpack (on main push)
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: ./scripts/skillpack.sh --manifest json

      - name: Upload skillpack artifact
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        uses: actions/upload-artifact@v4
        with:
          name: skillpack
          path: dist/skillpack/
          retention-days: 30
```

---

## Step 8 — Create `instructions.md`

This is the primary human-facing document. It lives in the project root and tells every developer how to use the Copilot skills, how to add new ones, and how to push the project to GitHub. Substitute the actual project name, description, and GitHub username collected in Step 1.

```markdown
# How to Use Copilot Skills in [PROJECT_NAME]

This project is set up with GitHub Copilot Agent Skills. Skills are folders of instructions stored in `.github/skills/` that Copilot loads automatically when your prompt matches, or that you can invoke directly with a slash command. Every developer who opens this repository in VS Code gets the same Copilot-assisted workflows without any additional configuration.

---

## Prerequisites

Before using skills in this project:

- VS Code 1.108 or later
- GitHub Copilot subscription (Free, Pro, Pro+, Business, or Enterprise)
- Copilot authenticated and active in VS Code

Verify Copilot is running: open Chat with `Ctrl+Alt+I` (Windows/Linux) or `Cmd+Ctrl+I` (macOS). If it loads without an auth error, you are ready.

---

## Skills in this project

### feature-scaffold

Creates the full file structure for a new feature.

**Invoke:** type `/feature-scaffold` in Copilot Chat, or ask "add a new feature for [name]"

**What it creates:** a route file, controller, service, types file, and test file under `src/features/{name}/`. Registers the route in the app entry point.

**When to use:** any time you are starting work on a new resource or module.

---

### code-review

Reviews staged changes and produces structured feedback organized by severity.

**Invoke:** type `/code-review` in Copilot Chat, or ask "review my changes" or "check this diff"

**What it produces:** a summary of what changed, critical issues that block merge, advisory suggestions, missing test coverage, and a verdict.

**When to use:** before opening a PR, or to get a second opinion on changes before committing.

---

### git-commit

Generates a Conventional Commits message from staged changes, confirms with you, then commits.

**Invoke:** type `/git-commit` in Copilot Chat, or ask "commit my changes"

**What it does:** reads the staged diff, writes a typed commit message, shows it to you for approval, then commits. Does not push.

**When to use:** any time you are ready to commit. Works for any size change.

---

### skill-scaffold

Creates a new skill folder with a valid `SKILL.md` and `README.md`.

**Invoke:** type `/skill-scaffold` in Copilot Chat. This skill requires explicit invocation — it does not auto-load.

**What it creates:** a populated skill folder under `.github/skills/{skill-name}/` following project conventions.

**When to use:** when you identify a repeatable workflow that should be encoded as a skill. See the "Adding a new skill" section below.

---

## Verifying skills are discovered

Type `/` in the Copilot Chat input. All four project skills should appear in the list alongside any personal skills you have configured.

If a skill is missing:
1. Confirm the skill folder exists under `.github/skills/`
2. Confirm the `name` field in `SKILL.md` matches the folder name exactly — VS Code silently skips skills where these differ
3. Run `./scripts/skillpack.sh --validate` — if the skill has a validation error, the script will tell you exactly what is wrong

---

## Adding a new skill

When you find yourself doing the same multi-step task repeatedly, that is a skill candidate. The threshold is roughly: more than two steps, done at least weekly, where getting it wrong costs real time.

**Step 1 — Invoke the scaffold skill:**

```
/skill-scaffold [new-skill-name]
```

Copilot creates the folder and file structure. Fill in the `SKILL.md` description and body.

**Step 2 — Write the description carefully:**

The description is what Copilot uses to decide when to load the skill. It must contain the natural-language phrases a developer in this project would actually type. A weak description means the skill never auto-invokes.

Good description:
```
Reviews database migrations for correctness and safety. Use this when asked to
review a migration, check a schema change, audit a migration file, or validate
an ALTER TABLE statement before it runs in production.
```

Poor description:
```
Helps with database migrations.
```

**Step 3 — Validate:**

```bash
./scripts/skillpack.sh --validate
```

All skills must pass before committing. The validator checks:
- `name` matches folder name exactly
- `name` is lowercase letters, digits, and hyphens only
- `name` is 64 characters or less
- `description` is 1024 characters or less
- `version` field is present
- `README.md` exists

**Step 4 — Update the skills table in `README.md`:**

Add a row for the new skill to the table in the root `README.md`.

**Step 5 — Commit:**

```bash
git add .github/skills/[new-skill-name]/
git commit -m "feat(skills): add [new-skill-name] skill"
```

---

## Pushing to GitHub for the first time

If you have not yet created the remote repository:

```bash
# Create the repo on GitHub (requires gh CLI)
gh repo create [GITHUB_USERNAME]/[PROJECT_NAME] --public

# Add the remote and push
git remote add origin https://github.com/[GITHUB_USERNAME]/[PROJECT_NAME].git
git push -u origin main
```

If you prefer to create the repo through the GitHub UI first, then:

```bash
git remote add origin https://github.com/[GITHUB_USERNAME]/[PROJECT_NAME].git
git push -u origin main
```

After the first push, the `validate-skills.yml` workflow runs automatically. Open the Actions tab on GitHub to confirm all skills pass validation.

---

## Skill validation reference

```bash
# Validate all skills — exits non-zero on any error
./scripts/skillpack.sh --validate

# Build the full skillpack archive
./scripts/skillpack.sh --manifest json

# See all options
./scripts/skillpack.sh --help
```

---

## Spec reference

Agent Skills follow the open standard at [agentskills.io](https://agentskills.io).

| Frontmatter field | Required | Notes |
|---|---|---|
| `name` | Yes | Must match folder name. Lowercase + hyphens. Max 64 chars. |
| `description` | Yes | Used for auto-discovery. Max 1024 chars. |
| `version` | No (repo convention) | Used by `skillpack.sh` for the manifest. |
| `argument-hint` | No | Hint shown in the slash command menu. Quote the value. |
| `user-invocable` | No | Defaults to `true`. Set `false` to hide from `/` menu. |
| `disable-model-invocation` | No | Defaults to `false`. Set `true` for explicit-only skills. |
```

---

## Step 9 — Run validation

Run the validator against all four skills:

```bash
./scripts/skillpack.sh --validate
```

Expected output:

```
Validated 4 skill(s): feature-scaffold, code-review, git-commit, skill-scaffold
```

If any skill fails, report the exact error and fix it before proceeding. Do not continue to Step 10 until validation passes cleanly.

---

## Step 10 — Stage, commit, and report status

Once validation passes, stage everything and make the initial commit:

```bash
git init
git add .
git status
```

Show the user the full git status output so they can verify every expected file is staged. Then commit:

```bash
git commit -m "feat: initialize project with GitHub Copilot Agent Skills

Adds four skills (feature-scaffold, code-review, git-commit, skill-scaffold),
the skillpack validation script, a GitHub Actions workflow, and instructions.md
documenting skill usage and the process for adding new skills."
```

After the commit, show the user:

```bash
git log --oneline -1
```

---

## Step 11 — Provide final summary to the user

After all steps complete successfully, output this summary:

```
Setup complete. Here is what was created:

  .github/skills/feature-scaffold/   — scaffolds new features
  .github/skills/code-review/        — reviews staged changes
  .github/skills/git-commit/         — generates Conventional Commits messages
  .github/skills/skill-scaffold/     — creates new skill folders
  scripts/skillpack.sh               — validates and packages skills
  .github/workflows/validate-skills.yml  — CI validation on push and PR
  instructions.md                    — setup and usage guide for this project
  README.md                          — project overview with skills table
  .gitignore

Next steps:

1. Open Copilot Chat (Ctrl+Alt+I / Cmd+Ctrl+I) and type / to confirm all four skills appear.
2. Push to GitHub:
     gh repo create [GITHUB_USERNAME]/[PROJECT_NAME] --public
     git remote add origin https://github.com/[GITHUB_USERNAME]/[PROJECT_NAME].git
     git push -u origin main
3. Open the Actions tab on GitHub and confirm the validate-skills workflow passes.
4. Start building. Use /feature-scaffold when adding the first feature.

See instructions.md for full usage details and how to add new skills as the project grows.
```

Substitute the actual GitHub username and project name from Step 1 into the Next steps output.

---

## Out of scope for this task

- Do not create any application source code (`src/`, `lib/`, etc.)
- Do not create a `package.json`, `pyproject.toml`, `.csproj`, or any language-specific project file — the user sets up their stack after this prompt runs
- Do not push to GitHub — provide the commands but let the user execute the push
- Do not create GitHub Issues, Projects, or repository settings
- Do not install any VS Code extensions