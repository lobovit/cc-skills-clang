# CLAUDE.md

## Project Overview

This is a Claude Code plugin containing AI agent skills for production-ready C/Clang projects. The repository provides reusable skill definitions that Claude Code can invoke when working on C codebases compiled with Clang/LLVM.

## Project Structure

```
skills/               # Claude Code skill definitions
  <skill-name>/
    SKILL.md          # Required: metadata + instructions
    references/       # Optional: detailed documentation loaded on demand
    scripts/          # Optional: executable code
    assets/           # Optional: templates, resources, linter configs (.clang-format, etc.)
.claude-plugin/       # Plugin metadata and configuration
.cursor-plugin/       # Plugin metadata and configuration (version must match .claude-plugin/plugin.json)
gemini-extension.json # Gemini CLI extension manifest (version must match .claude-plugin/plugin.json)
```

## Agent Skills Specification

All skills MUST conform to the [Agent Skills specification](https://agentskills.io/specification.md). Key requirements are summarized below; the spec is the source of truth when in doubt.

## Frontmatter

New skills go in `skills/<skill-name>/SKILL.md`. Each SKILL.md has YAML frontmatter. Fields per the [Agent Skills spec](https://agentskills.io/specification.md) — **this project requires all fields marked "Project-required"**:

| Field | Required | Constraints |
| --- | --- | --- |
| `name` | Spec-required | 1-64 chars. Lowercase `a-z`, digits, hyphens. No leading/trailing/consecutive hyphens. **Must match parent directory name.** |
| `description` | Spec-required | 1-1024 chars. Describes what the skill does **and when to use it** — this is the primary triggering mechanism. Be specific and slightly "pushy" to avoid under-triggering. |
| `license` | Project-required | License name or reference to a bundled license file. Use `MIT` for this project. |
| `compatibility` | Project-required | 1-500 chars. Describe actual requirements. Base: `Designed for Claude Code or similar AI coding agents.` Extend when needed: add `Requires clang`, `Requires git`, etc. |
| `metadata` | Project-required | Must include `author` (string), `version` (semver `a.b.c` string, e.g. `"1.0.0"`), and `openclaw` (object — see below). |
| `user-invocable` | Project-required | Boolean. `true` for skills invocable as slash commands (e.g. `/clang-security`), `false` (default) for contextual skills that auto-trigger. |
| `allowed-tools` | Project-required | Space-delimited list of pre-approved tools. See "Allowed tools" below. |

### ClawHub metadata (`metadata.openclaw`)

Every skill MUST include a `metadata.openclaw` block for [ClawHub](https://github.com/openclaw/clawhub) discoverability and dependency management.

| Field | Required | Description |
| --- | --- | --- |
| `emoji` | Yes | Display emoji for the skill (single emoji string) |
| `homepage` | Yes | URL to the skill's homepage. Use `https://github.com/lobor/cc-skills-clang` for this project. |
| `requires.bins` | Yes | CLI binaries that must be installed. Always includes `clang`. Add skill-specific critical bins (e.g. `clang-tidy`, `clang-format`). |
| `install` | Yes | Array of auto-installable dependencies. Use `[]` when no extra deps needed. Supported kinds: `brew`, `node`. Each entry has `kind`, `formula`/`package`, and `bins` fields. |

Example frontmatter:

```yaml
---
name: clang-example
description: "Clang skill for X. Use when doing Y."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🔧"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
---
```

## Allowed Tools

Every skill MUST declare an `allowed-tools` field. Start from the **default set** and add skill-specific extras as needed.

**Default tools** (include in every skill):

```
Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
```

**Skill-specific extras** — add only when relevant:

| Extra tool | When to add |
| --- | --- |
| `Bash(clang-tidy:*)` | Static analysis skills |
| `Bash(clang-format:*)` | Formatting skills |
| `Bash(opt:*)` | LLVM optimization skills |
| `Bash(llc:*)` | LLVM codegen skills |
| `Bash(lldb:*)` | Debugging skills |
| `Bash(git:*)` | Git-related skills |
| `WebFetch` | Library-specific skills, skills requiring deep research |
| `WebSearch` | Skills requiring deep research or analysis |
| `AskUserQuestion` | Skills that benefit from clarifying user intent |

## Skill Body

The body contains step-by-step instructions. Use secondary markdown files in `references/` for depth (referenced via relative links like `[Details](references/details.md)`). Keep file references one level deep from SKILL.md.

**Important:** When including non-markdown content (configuration files, scripts, templates, linter configs), create them as separate files in `assets/` rather than embedding them directly in markdown.

### Token budgets

- **~100 tokens per description** — loaded at startup for all skills
- **≤ 1,000 characters per description** — hard limit; keep descriptions focused and scannable
- **< 5.000 tokens per SKILL.md** (spec recommendation) — keep focused on essentials
- **< 2.500 tokens per SKILL.md** (project recommendation)
- **< 500 lines per SKILL.md** — move detailed reference material to `references/`
- **Use secondary markdown files for depth** — Claude reads these on demand, so they don't count against context until needed
- **2-4 skills loaded simultaneously** in a typical session
- **Stay below ~10k tokens of total loaded SKILL.md** to avoid degrading response quality

## Writing Guidelines

### Avoid duplicating linter rules

Skills should NOT re-explain rules already enforced by linters (e.g. clang-tidy, clang-format). If a `.clang-format` is present in the skill directory, the linter is the source of truth. Skill instructions should focus on higher-level patterns, architecture decisions, and judgment calls that linters cannot catch.

### Teach reasoning, not only rules

Skills MUST teach Claude how to think about problems, not just list prescriptive rules. Every recommendation needs a "why" — what goes wrong without it, what consequence the reader avoids.

## Best Practice Sources

- <https://clang.llvm.org/docs/ClangCommandLineReference.html>
- <https://clang.llvm.org/extra/clang-tidy/>
- <https://llvm.org/docs/LangRef.html>
- <https://llvm.org/docs/Passes.html>
- <https://developer.android.com/ndk/guides/asan>
- <https://github.com/google/sanitizers>
- <https://owasp.org/www-project-web-security-testing-guide/latest/>
- <https://cwe.mitre.org/>
- <https://faultslinux.com/>
