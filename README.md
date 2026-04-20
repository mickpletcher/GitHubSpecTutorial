# GitHub Copilot Agent Skills Spec Tutorial

<!-- markdownlint-disable MD032 -->

Practical documentation for learning how GitHub Copilot Agent Skills work in real projects.

This repository is a tutorial resource, not a production skill catalog. It teaches spec behavior, authoring patterns, and practical rollout paths for new and existing repositories.

## Start Here

If you are new to Agent Skills, follow this order:

1. Read [TUTORIAL.md](./TUTORIAL.md#quick-start-build-a-minimal-working-skill-in-10-minutes) and complete the Quick Start.
2. Read [TUTORIAL.md](./TUTORIAL.md#what-is-part-of-the-spec-vs-what-is-this-repos-convention).
3. Choose your implementation path:
   - New repository: [Project1-Greenfield.md](./Project1-Greenfield.md)
   - Existing repository: [Project2-Retrofit.md](./Project2-Retrofit.md)

If you already know the basics, jump to [TUTORIAL.md](./TUTORIAL.md#why-my-skill-is-not-loading-troubleshooting-checklist).

## What Is In This Repository

- [TUTORIAL.md](./TUTORIAL.md): Main reference for spec behavior, authoring, discovery, invocation controls, and troubleshooting.
- [Project1-Greenfield.md](./Project1-Greenfield.md): Agent-mode prompt to set up a new repository with skills from day one.
- [Project2-Retrofit.md](./Project2-Retrofit.md): Agent-mode prompt to audit an existing repository and add skills safely.

## Quick Comparison: What To Use When

| Tool | Use it for | Not ideal for |
| --- | --- | --- |
| Agent Skills (`SKILL.md`) | Repeatable multi-step workflows for coding tasks | Always-on global policies |
| Custom instructions | Repository or user-wide coding rules that should always apply | Multi-step task orchestration |
| Prompt files (`.prompt.md`) | Reusable one-shot prompts | Autonomous workflow behavior |
| Custom agents | Specialized persistent agent personas and broader orchestration behavior | Lightweight task snippets |

## Critical Accuracy Notes

- A skill is a directory containing `SKILL.md` and optional supporting files.
- Copilot uses `description` to decide whether a skill is relevant for a prompt.
- The `name` field in `SKILL.md` must exactly match the parent directory name.
- The `license` field is optional per spec. If the Copilot CLI rejects your skill with a missing-license error, see TUTORIAL.md for the workaround.
- Repository-level locations include:
  - `.github/skills/<skill-name>/SKILL.md`
  - `.claude/skills/<skill-name>/SKILL.md`
  - `.agents/skills/<skill-name>/SKILL.md`
- Personal skill locations follow the same pattern in your home directory.

## Allowed Tools And Safety

`allowed-tools` is optional frontmatter. Use it only when a skill must call specific tools.

High-trust warning:
- Pre-approving `shell` or `bash` tool access can execute commands that modify files, credentials, or runtime state.
- Only pre-approve shell tools for reviewed skills from trusted sources.
- Never pre-approve shell tools for unknown or copied skills without manual inspection.

See detailed guidance in [TUTORIAL.md](./TUTORIAL.md#allowed-tools-when-to-use-it-and-when-not-to).

## Suggested Validation Flow

This repo is documentation only, so no local script is shipped here. Use this reproducible check flow in any repository where you create skills:

1. Put the skill in one of the supported locations.
2. Confirm folder name equals `name`.
3. Open Copilot Chat and type `/` to verify discovery.
4. Test one natural language prompt from your `description`.
5. Test explicit `/skill-name` invocation.
6. Review behavior before trusting automated command execution.

## Further Reading

- [gh skill CLI changelog (April 2026)](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/)
- [VS Code Agent Skills docs](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- [Agent Skills open standard](https://agentskills.io)
- [GitHub Docs: Adding agent skills for Copilot](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills)
- [github/awesome-copilot](https://github.com/github/awesome-copilot)
- [anthropics/skills](https://github.com/anthropics/skills)

<!-- markdownlint-enable MD032 -->
