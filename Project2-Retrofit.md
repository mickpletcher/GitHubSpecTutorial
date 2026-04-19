# Copilot Task: Retrofit — Adding GitHub Copilot Agent Skills to an Existing Repository

<!-- markdownlint-disable MD031 MD032 MD036 MD040 MD058 MD060 -->

Paste this prompt into GitHub Copilot **Agent mode** in VS Code. Agent mode is required — this prompt reads repository files, runs terminal commands, creates new files, and validates the result. Ask mode will not work.

Before starting, open the existing repository in VS Code. This prompt assumes a populated codebase is already present. It does not touch application code, existing CI workflows, or any file outside of `.github/skills/`, `scripts/`, and the two new files it creates (`instructions.md` and the CI workflow).

---

## Context

You are retrofitting an existing software project with GitHub Copilot Agent Skills. Unlike a greenfield setup, the patterns, conventions, and tribal knowledge already exist in this repository — in commit history, contributing guides, PR templates, recurring CI failures, and code structure. Your job is to find those patterns, extract them into skills, and make them available to every Copilot user who opens this repository.

The approach is audit first, build second. No file is created until you understand what already exists.

---

## Step 1 — Establish baseline: what is already here

Run the following commands to build a picture of the repository before touching anything. Collect the output of each — you will use it throughout the remaining steps.

### Repository overview

```bash
# Top-level structure
ls -la

# How large is this repo?
git log --oneline | wc -l

# Who contributes and how often?
git shortlog -sn --no-merges | head -10

# How old is this repo?
git log --oneline --reverse | head -1
```

### Existing documentation

```bash
# Check for contributing guide
cat CONTRIBUTING.md 2>/dev/null || cat .github/CONTRIBUTING.md 2>/dev/null || echo "No CONTRIBUTING.md found"

# Check for PR template
cat .github/pull_request_template.md 2>/dev/null || cat .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null || echo "No PR template found"

# Check for issue templates
ls .github/ISSUE_TEMPLATE/ 2>/dev/null || echo "No issue templates found"

# Check for existing custom instructions
cat .github/copilot-instructions.md 2>/dev/null || echo "No Copilot instructions found"
```

### Existing CI

```bash
# List all existing workflows
ls .github/workflows/ 2>/dev/null || echo "No workflows directory found"

# Read each workflow name and trigger to understand what already runs
for f in .github/workflows/*.yml .github/workflows/*.yaml; do
    [ -f "$f" ] && echo "--- $f ---" && grep -E "^name:|^on:" "$f" | head -4
done
```

### Commit message patterns

```bash
# Most common commit prefixes — reveals what types of work happen most
git log --oneline --no-merges -100 | \
    grep -oE '^[a-f0-9]+ (feat|fix|docs|refactor|perf|test|chore|ci|style|add|update|remove|fix)' | \
    awk '{print $2}' | sort | uniq -c | sort -rn | head -10

# Commit messages that suggest repeated manual workflows
git log --oneline --no-merges -200 | \
    grep -iE "(scaffold|boilerplate|template|add test|fix lint|update docs|migration|schema|release|changelog)" | head -20

# Most frequently changed directories — reveals where the action is
git log --no-merges --name-only --pretty=format: -200 | \
    grep -v '^$' | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn | head -10
```

### Code structure

```bash
# Top-level source directories
find . -maxdepth 3 -type d \
    -not -path './.git/*' \
    -not -path './node_modules/*' \
    -not -path './.venv/*' \
    -not -path './dist/*' \
    -not -path './build/*' \
    -not -path './bin/*' \
    -not -path './obj/*' | sort

# What languages are present?
git ls-files | grep -oE '\.[^.]+$' | sort | uniq -c | sort -rn | head -10
```

### Existing skill infrastructure

```bash
# Check if any skills are already present
ls .github/skills/ 2>/dev/null && echo "Skills found:" && ls .github/skills/ || echo "No existing skills directory"

# Check for a skillpack script
ls scripts/skillpack.sh 2>/dev/null || echo "No skillpack script found"
```

After running all of the above, pause and summarize what you found in this format before continuing:

```
Repository summary:
- Language(s): [detected languages]
- Approx. commit count: [n]
- CONTRIBUTING.md: [found / not found]
- PR template: [found / not found]
- Existing CI workflows: [list names]
- Existing Copilot skills: [found n / not found]
- Existing skillpack script: [found / not found]
- Top changed directories: [list top 3]
```

---

## Step 2 — Identify skill candidates

Based on the audit output from Step 1, identify candidate workflows to encode as skills. Apply the following criteria to each candidate before including it.

### Candidate scoring criteria

A strong skill candidate meets all three:
1. **Frequency** — the workflow happens at least weekly for at least one developer
2. **Multiple steps** — more than two steps where order matters or steps are easy to miss
3. **Cost of mistakes** — getting it wrong causes real rework, broken builds, or missed files

Candidates that fail any criterion should be noted but not built in this pass.

### Sources to mine

**From CONTRIBUTING.md** — any "How to" section is a candidate. Extract the heading and the step count:

```bash
grep -n "^## \|^### " CONTRIBUTING.md 2>/dev/null | grep -iE "how to|adding|creating|running|setting up|deploying|testing|migrating"
```

**From PR template** — each checklist item that a reviewer must manually verify is a code-review skill criterion:

```bash
cat .github/pull_request_template.md 2>/dev/null
```

**From recurring CI failures** — if the repo has a CI history, find the checks that fail most:

```bash
# Requires gh CLI authenticated
gh run list --limit 100 --json conclusion,name,status 2>/dev/null | \
    python3 -c "
import json, sys, collections
data = json.load(sys.stdin)
failures = [r['name'] for r in data if r['conclusion'] == 'failure']
counter = collections.Counter(failures)
for name, count in counter.most_common(10):
    print(f'{count:3d}  {name}')
" 2>/dev/null || echo "gh CLI not authenticated or no run history"
```

**From git log patterns** — commits that show the same manual steps being done repeatedly:

```bash
git log --oneline --no-merges -500 | \
    grep -iE "migration|schema|scaffold|boilerplate|release note|changelog|add endpoint|new feature" | head -20
```

**From code repetition** — files that follow an obvious template pattern:

```bash
# Find files that share a naming pattern — often indicates a scaffold workflow
git ls-files | sed 's|.*/||' | sort | uniq -d | head -20
```

### Present candidates to the user

After mining all sources, present your findings as a ranked list. Ask the user to select which ones to build in this pass. Cap this session at five skills — better to ship five strong ones than ten shallow ones.

Format:

```
Skill candidates identified:

[1] feature-scaffold — CONTRIBUTING.md has a 6-step "Adding a new endpoint" section.
    Source: CONTRIBUTING.md lines 42-61. Frequency: high (12 matching commits in last 200).
    Verdict: strong candidate.

[2] code-review — PR template has 8 manual checklist items reviewers verify on every PR.
    Source: .github/pull_request_template.md. Frequency: every PR.
    Verdict: strong candidate.

[3] db-migration — git log shows 18 commits in the last 6 months matching "migration" or "schema".
    Source: git log pattern. Frequency: high.
    Verdict: strong candidate — but ask the user for migration conventions before writing the skill.

[4] release-notes — 4 commits in the last 6 months matching "release note" or "changelog".
    Source: git log pattern. Frequency: low.
    Verdict: moderate. Include only if the user does this manually today.

[5] ...

Which skills should I build in this session? (Select up to 5, or type "all" to build all strong candidates.)
```

Wait for the user to respond before proceeding. Do not create any files until the skill list is confirmed.

---

## Step 3 — Collect missing context for each selected skill

For each skill the user selected, run targeted commands to gather the specific context needed to write a concrete skill body. Generic skill bodies fail — the instructions must match what this codebase actually does.

### For a feature scaffold skill

```bash
# Find an example of the file pattern
git log --diff-filter=A --name-only --pretty=format: -50 | grep -v '^$' | head -30

# Read an existing feature to understand the structure
# (adjust path based on what the audit revealed)
find src -name "*.routes.*" -o -name "*.controller.*" -o -name "*.service.*" 2>/dev/null | head -10

# Find where routes are registered
grep -rn "app.use\|router.use\|include_router\|app.route\|MapControllers" src/ --include="*.ts" --include="*.py" --include="*.cs" --include="*.go" 2>/dev/null | head -10
```

### For a code review skill

```bash
# Read the full PR template
cat .github/pull_request_template.md 2>/dev/null

# Check if there is a code style guide referenced anywhere
grep -rn "style guide\|coding standard\|eslint\|pylint\|rubocop\|golangci" . \
    --include="*.md" --include="*.json" --include="*.yml" --include="*.yaml" 2>/dev/null | \
    grep -v ".git" | head -10

# Find what the linter/formatter is
ls .eslintrc* .eslintrc.json .eslintrc.js .pylintrc .rubocop.yml .golangci.yml \
   pyproject.toml setup.cfg 2>/dev/null | head -5
```

### For a migration skill

```bash
# Find the migration directory
find . -type d -name "migrations" -o -name "migration" -o -name "db/migrate" 2>/dev/null | grep -v ".git"

# Read a recent migration file as a template
ls -lt migrations/ 2>/dev/null | head -5

# Find the migration command
grep -rn "migrate\|knex\|alembic\|flyway\|liquibase\|ef.*migration\|sequelize" \
    package.json Makefile Dockerfile README.md 2>/dev/null | head -10
```

### For a release notes skill

```bash
# Find existing release notes or changelog
ls CHANGELOG.md RELEASES.md HISTORY.md docs/CHANGELOG.md 2>/dev/null

# Read the format if found
head -60 CHANGELOG.md 2>/dev/null || head -60 RELEASES.md 2>/dev/null || echo "No existing changelog found"

# Find how releases are tagged
git tag --sort=-version:refname | head -5
```

After running the targeted commands for each selected skill, summarize what you found and any gaps that require user input before the skill body can be written. Ask all remaining questions at once — do not interrupt the build phase with questions.

---

## Step 4 — Check for conflicts before creating anything

Before creating any directories, verify the skill infrastructure does not already partially exist in a conflicting state:

```bash
# Check for existing skills directory
ls .github/skills/ 2>/dev/null && echo "Existing skills found — will add alongside" || echo "No existing skills directory"

# Check for a conflicting skillpack script
ls scripts/skillpack.sh 2>/dev/null && echo "Existing skillpack.sh found — will not overwrite" || echo "No existing skillpack.sh"

# Check for a conflicting validate-skills workflow
ls .github/workflows/validate-skills.yml 2>/dev/null && echo "Existing validate-skills.yml found — will not overwrite" || echo "No existing validate-skills.yml"
```

**If a `skillpack.sh` already exists:** read it and determine whether it covers the checks in Step 7. If it does, skip Step 7. If it is missing any checks, note the gaps and ask the user whether to update it or add the missing checks to a separate script.

**If a `validate-skills.yml` already exists:** skip Step 8 and note to the user that validation CI is already present.

**If skills already exist in `.github/skills/`:** list them and ask the user whether any should be updated or replaced in this pass. Do not overwrite existing skill files without explicit user confirmation.

---

## Step 5 — Create the skill directory structure

Create only the directories needed for the skills the user selected in Step 2. Do not create placeholder directories for skills that were not selected.

```bash
mkdir -p .github/skills
for skill in [SELECTED_SKILL_NAMES]; do
    mkdir -p ".github/skills/$skill"
done
```

If `scripts/` does not already exist:

```bash
mkdir -p scripts
```

Confirm the structure:

```bash
find .github/skills scripts -type d | sort
```

---

## Step 6 — Write SKILL.md files extracted from the repository's actual patterns

For each selected skill, write the `SKILL.md` by drawing directly from the context gathered in Step 3. Do not use generic templates — every step in the skill body must reference actual file paths, commands, and conventions found in this codebase.

The rules for every SKILL.md written in this step:

1. The `name` field must exactly match the folder name
2. The `description` must contain the natural-language phrases a developer in this project would actually type — not abstract descriptions of what the skill does
3. Every step in the workflow must be grounded in what the audit found — reference real file paths, real commands, real registration patterns
4. The `version` field must be `1.0.0`
5. The `argument-hint` must be quoted
6. Skills that the user should only invoke explicitly (scaffolding, migration creation) must include `disable-model-invocation: true`

### Writing each skill

For each skill:

**Step 6a — Write the frontmatter**

Pull the description from the CONTRIBUTING.md section, PR template, or commit pattern that surfaced this skill. The description should read like what a developer types in chat, not like a spec.

**Step 6b — Write the workflow body**

Convert the source material (contributing section, PR checklist, migration steps) into procedural Copilot instructions. The difference between a contributing guide section and a skill body is specificity:

- Contributing guide: "Create a migration file"
- Skill body: "Run `npm run db:migration:create -- --name={description}`. This creates `migrations/{timestamp}_{description}.ts`. Open it and add the `up()` and `down()` methods following the pattern in `migrations/20250101_add_users_table.ts`."

Reference an actual existing file wherever a template or pattern is involved. Copilot will read it.

**Step 6c — Write the README.md**

Two to four sentences: what the skill does, how to invoke it, what it produces.

**Step 6d — Write the edge cases section**

At minimum cover: what to do if inputs are missing, what to do if the relevant files do not exist, and what to do if a conflict is detected.

---

## Step 7 — Create `scripts/skillpack.sh`

Skip this step if a `skillpack.sh` already exists (detected in Step 4).

Create `scripts/skillpack.sh` with the following content exactly, then make it executable:

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

```bash
chmod +x scripts/skillpack.sh
```

Add `dist/skillpack/` and `dist/skillpack.tar.gz` to `.gitignore` if they are not already present:

```bash
grep -q "dist/skillpack" .gitignore 2>/dev/null || printf '\n# Skillpack output\ndist/skillpack/\ndist/skillpack.tar.gz\n' >> .gitignore
```

---

## Step 8 — Add skill validation to CI

Skip this step if a `validate-skills.yml` workflow already exists (detected in Step 4).

Read the existing workflow files before creating the new one to understand what runner, checkout action version, and job naming conventions the project uses:

```bash
head -30 .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null | \
    grep -E "runs-on:|uses: actions/checkout"
```

Match the checkout action version and runner to what the project already uses. Then create `.github/workflows/validate-skills.yml`:

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

      - name: Build skillpack
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

The `paths` filter on `pull_request` is intentional. This workflow only fires on PRs that touch skill files or the validation script — it does not add latency to unrelated PRs.

---

## Step 9 — Create `instructions.md`

Create `instructions.md` in the repository root. This file documents what was added, how to use each skill, and how to add new skills going forward. Populate it with the actual skill names and descriptions from this session — do not use generic placeholder content.

```markdown
# Copilot Skills — [REPO_NAME]

This repository now includes GitHub Copilot Agent Skills. Skills are stored in `.github/skills/` and load automatically in VS Code when Copilot detects a matching prompt, or you can invoke them directly with a slash command.

These skills were extracted from patterns already present in this codebase — from the contributing guide, PR template, commit history, and code structure. They encode workflows the team was already doing manually.

---

## Skills added in this session

[For each skill added, write a block like this:]

### [skill-name]

[One sentence: what does this skill do?]

**Invoke:** type `/[skill-name]` in Copilot Chat, or ask "[example natural language prompt from the skill description]"

**Source:** extracted from [CONTRIBUTING.md section / PR template / git log pattern / code structure]

**What it does:** [two to three sentences describing the workflow — not the spec, the actual outcome]

**When to use:** [specific situations — "before opening any PR that touches the database schema"]

---

## Verifying skills are active

Type `/` in the Copilot Chat input (`Ctrl+Alt+I` / `Cmd+Ctrl+I`). All skills added in this session should appear in the list.

If a skill is missing:

```bash
./scripts/skillpack.sh --validate
```

The validator reports exactly what is wrong. Common causes: the `name` field in `SKILL.md` does not match the folder name (VS Code silently skips mismatched skills), or a required field is missing.

---

## Adding new skills

When you find yourself doing the same multi-step task repeatedly and it is not already a skill, encode it.

**Rule of thumb:** more than two steps, done at least weekly, where missing a step causes real rework.

**Step 1 — Create the skill folder:**

```bash
mkdir -p .github/skills/[new-skill-name]
```

The folder name becomes the slash command name. Use kebab-case.

**Step 2 — Write `SKILL.md`:**

The frontmatter minimum:

```yaml
---
name: [new-skill-name]        # Must match the folder name exactly
version: 1.0.0
description: [What this skill does. Include every phrase a developer would type to trigger it.]
argument-hint: "[optional hint shown in the slash command menu]"
---
```

The body should be a procedural workflow: numbered steps, specific file paths, real commands, and edge cases.

**Step 3 — Write `README.md`:**

Two to four sentences covering what the skill does, how to invoke it, and what it produces.

**Step 4 — Validate:**

```bash
./scripts/skillpack.sh --validate
```

All skills must pass before committing.

**Step 5 — Update `CONTRIBUTING.md`:**

Add the new skill to the skills table in the contributing guide so the team knows it exists.

**Step 6 — Commit:**

```bash
git add .github/skills/[new-skill-name]/
git commit -m "feat(skills): add [new-skill-name] skill"
```

---

## Skill validation reference

```bash
# Validate all skills — exits non-zero on any error, reports exactly what failed
./scripts/skillpack.sh --validate

# Build a distributable skillpack archive with JSON manifest
./scripts/skillpack.sh --manifest json

# Write to a custom output directory
./scripts/skillpack.sh --output /tmp/skillpack-preview --manifest json
```

Validation checks enforced:

| Check | What it catches |
|---|---|
| `name` matches folder name | Prevents VS Code from silently skipping the skill |
| `name` is lowercase + hyphens only | Spec compliance |
| `name` ≤ 64 characters | Spec compliance |
| `description` ≤ 1024 characters | Spec compliance |
| `version` field present | Repo convention for manifest |
| `README.md` exists | Skills must be documented |
| Frontmatter markers present | Catches malformed SKILL.md files |

---

## CI validation

The `.github/workflows/validate-skills.yml` workflow runs automatically:

- On every push to `main`
- On every pull request that touches `.github/skills/` or `scripts/skillpack.sh`

The PR path filter means skill validation does not add latency to PRs that do not touch skills. On push to `main`, it also builds and archives the skillpack as a workflow artifact available in the Actions tab.

---

## Agent Skills spec reference

Skills follow the open standard at [agentskills.io](https://agentskills.io).

| Field | Required | Spec limit | Notes |
|---|---|---|---|
| `name` | Yes | 64 chars, lowercase + hyphens | Must match folder name |
| `description` | Yes | 1024 chars | Used for auto-discovery |
| `version` | No (repo convention) | — | Used by skillpack.sh manifest |
| `argument-hint` | No | — | Quote the value |
| `user-invocable` | No | — | Default `true` |
| `disable-model-invocation` | No | — | Default `false` — set `true` for explicit-only skills |
```

---

## Step 10 — Run full validation

Run the validator against all skills created in this session:

```bash
./scripts/skillpack.sh --validate
```

Expected output names every skill that was added and reports clean validation for each. If any skill fails, fix it before continuing. The most common failure at this stage is a `name` field that does not match the folder — fix the frontmatter, not the folder name, unless the folder name itself violates the spec.

After validation passes, run a full build to confirm the manifest generates cleanly:

```bash
./scripts/skillpack.sh --manifest json
cat dist/skillpack/manifest.json
```

Confirm the manifest contains an entry for every skill added in this session with a populated `name`, `version`, and `description`.

---

## Step 11 — Create a focused branch and commit

All changes from this session go in one branch and one PR. Do not mix skill additions with application code changes.

```bash
# Create the branch
git checkout -b feat/add-copilot-skills

# Stage only the files created in this session
git add .github/skills/
git add scripts/skillpack.sh
git add .github/workflows/validate-skills.yml
git add instructions.md
git add .gitignore

# Confirm nothing unintended is staged
git status
```

Show the user the full `git status` output and wait for confirmation that the staged files are correct before committing. Do not commit automatically.

After confirmation:

```bash
git commit -m "feat(skills): add GitHub Copilot Agent Skills

Extracted [N] skills from existing repository patterns:
[List each skill and its one-line description]

Adds skillpack validation script and CI workflow.
No application code changes. See instructions.md for usage."
```

Substitute the actual skill names and descriptions collected during this session.

---

## Step 12 — Provide PR creation command and final summary

After the commit is made, provide the exact commands to push and open the PR:

```bash
git push -u origin feat/add-copilot-skills

gh pr create \
  --title "feat: add GitHub Copilot Agent Skills" \
  --body "## What this adds

Adds [N] Copilot Agent Skills extracted from existing repository patterns.

| Skill | Source | What it does |
|---|---|---|
[One row per skill with source and description]

## What this does not change

- No application code modified
- No existing CI workflows modified
- No dependencies added

## How to verify

1. Pull the branch and open in VS Code
2. Open Copilot Chat and type \`/\` — all added skills should appear
3. Check the Actions tab after push — validate-skills workflow should pass

## Reviewer notes

See \`instructions.md\` for usage documentation and how to add new skills going forward."
```

Then output the final summary:

```
Retrofit complete.

Skills added:
[List each skill with name, source, and slash command]

Files created:
  .github/skills/[skill-name]/   — for each skill added
  scripts/skillpack.sh           — validation and packaging
  .github/workflows/validate-skills.yml  — CI validation
  instructions.md                — usage guide and conventions

Files modified:
  .gitignore                     — added dist/skillpack/ exclusions

Files not touched:
  All application source code
  All existing CI workflows
  All existing documentation (except .gitignore)

Next steps:
1. Merge the PR
2. Open the repo in VS Code and type / to confirm all skills appear
3. Add a Copilot Skills section to CONTRIBUTING.md pointing to instructions.md
4. Run /skill-scaffold when the next repeatable workflow is identified
```

---

---

## Tutorial: Understanding the retrofit process

This section explains the reasoning behind the audit methodology, why each extraction pattern produces good skills, and how to maintain the skills you added as the codebase evolves. Read it after the prompt completes.

---

### Why audit-first is non-negotiable

The most common mistake in a retrofit is to write skills from memory — a developer thinks about what their team does and writes instructions from their mental model of the process. These skills are almost always wrong in ways that only become visible when Copilot tries to follow them.

The actual workflow stored in a codebase is almost always different from what developers believe it to be. The file paths are slightly different. The command has an extra flag. The registration step has an order dependency nobody remembers consciously. The skill body must reflect the actual codebase, not the remembered version.

The audit extracts from real sources: git history (what actually happened), CONTRIBUTING.md (what was written down), PR templates (what reviewers actually check), and CI failures (what developers actually get wrong). Every one of these sources is grounded in reality. Memory is not.

---

### Why the three sources reliably produce good skills

**CONTRIBUTING.md — the workflow already written down**

A CONTRIBUTING.md "How to add a new endpoint" section is a skill body first draft. The author wrote it because the workflow is non-obvious enough to need documentation. That is exactly the criterion for a good skill: complex enough to need documentation, repeated often enough to automate.

The transformation from CONTRIBUTING.md section to skill body is about specificity. The contributing guide tells a human what to do. The skill tells Copilot exactly where to look and what to check:

Contributing guide step: "Create a route file in `src/routes/`"

Skill body step: "Create `src/routes/{resource}.routes.ts` using `src/routes/user.routes.ts` as the template. Copy the import structure and replace `User` with `{Resource}`. Register the route in `src/app.ts` by adding `app.use('/api/{resource}s', {resource}Router)` after line 47."

The content is the same. The specificity is completely different. Copilot cannot follow vague instructions reliably. It follows specific ones.

**PR templates — the review already encoded**

A PR checklist is a list of things reviewers need to verify before merging. Each checklist item is a code review skill criterion. The team already did the work of deciding what matters — the skill just automates the checking.

The transformation is direct: each `- [ ] item` in the PR template becomes a review criterion in the `code-review` skill body. The difference is that the skill actually reads the diff and evaluates each criterion rather than asking a developer to remember to check.

**CI failures — the mistakes developers already make**

If the same CI check fails repeatedly, a developer is consistently missing a step. That missing step is a skill insertion point. The skill body should include that step explicitly, before the work is committed, so CI never sees the failure again.

Finding recurring CI failures is one of the highest-ROI things the audit does. A skill that prevents a CI failure is worth more than one that codifies a workflow that already runs reliably.

---

### The extraction transformation explained

The single most important concept in a retrofit is the difference between what a contributing guide tells a human and what a skill body tells Copilot.

A contributing guide is written for a human developer reading it once, understanding the intent, and applying judgment to fill in the gaps. It can be brief because the reader has context.

A skill body is written for Copilot executing it on every invocation, with no memory of previous invocations, and no judgment to fill in gaps. It must be complete and explicit.

**What a contributing guide can leave implicit:**
- "Create the route file" — the developer knows what format, where to put it, what to name it
- "Write tests" — the developer knows your test framework, naming conventions, and what to cover
- "Update the docs" — the developer knows which docs and what format

**What a skill body must make explicit:**
- "Create `src/routes/{resource}.routes.ts` using `src/routes/user.routes.ts` as the template"
- "Create `src/__tests__/{resource}.test.ts`. Cover: happy path for GET/POST, missing required fields (expect 400), unauthorized access (expect 401)"
- "Open `docs/openapi.yml`. Add paths and schema under the `paths` and `components/schemas` keys. Run `npm run validate:openapi` to confirm"

Every implicit assumption in the contributing guide becomes an explicit instruction in the skill body. This is the core intellectual work of a retrofit.

---

### The scoring criteria mapped to real value

The audit scores skill candidates on frequency and cost of mistakes. Here is what those criteria actually mean in practice.

**Frequency** is not about how often a task happens — it is about how often a developer has to think about it. A task done weekly that is muscle memory (running the test suite) is not a good skill candidate. A task done weekly that requires remembering a non-obvious sequence of steps (adding a new endpoint with its five required files) is an excellent candidate.

**Cost of mistakes** is the time and frustration caused when a step is missed. A missing file that causes a build error is a cost. A wrong naming convention that requires renaming files across the codebase is a higher cost. A missed security check that ships to production is a much higher cost.

The best skill candidates score high on both: workflows done frequently enough that small time savings compound, with mistakes costly enough that preventing even one pays for the skill setup time.

---

### Maintaining skills as the codebase evolves

Skills extracted from an existing codebase are coupled to that codebase. When the patterns change, the skills must change too. This is the maintenance obligation that does not exist in a greenfield setup.

**Triggers that mean a skill needs updating:**
- A file path referenced in a skill body no longer exists
- A command in a skill body has changed flags or syntax
- A registration step has moved to a different file or changed format
- A new required step was added to the workflow but is not in the skill body
- A skill's CI validation is failing because the skill body produces output that no longer matches conventions

**The skill update process:**
```bash
# Locate the skill
ls .github/skills/

# Edit the body — make it match current reality
code .github/skills/[skill-name]/SKILL.md

# Update the version field
# 1.0.0 → 1.1.0 for body changes that maintain the same workflow
# 1.0.0 → 2.0.0 for workflow changes that break backward compatibility

# Validate
./scripts/skillpack.sh --validate

# Commit
git add .github/skills/[skill-name]/
git commit -m "fix(skills): update [skill-name] for [change description]"
```

**Who owns skill maintenance:**
The developer who notices the skill is wrong owns the fix. This is the same as fixing a stale comment in code — whoever sees it fixes it. The validator catches structural issues at commit time. Semantic drift (the skill body is valid but no longer accurate) is caught by usage — when Copilot produces incorrect output following a skill, that is the signal that the skill needs updating.

**Skills as living documentation:**
A skill is more authoritative than a CONTRIBUTING.md section because it is what Copilot actually executes. When a skill and the contributing guide disagree, fix the skill first — it is the executable specification of the workflow. Then update the contributing guide to match.

---

## Out of scope for this task

- Do not modify any application source code
- Do not modify any existing `.github/workflows/` files — only add new ones
- Do not modify `CONTRIBUTING.md`, `README.md`, or any existing documentation
- Do not install dependencies or modify lock files
- Do not push to remote — provide the commands but let the user execute the push
- Do not create GitHub Issues or Projects
- Do not overwrite existing skill files without explicit user confirmation

<!-- markdownlint-enable MD031 MD032 MD036 MD040 MD058 MD060 -->
