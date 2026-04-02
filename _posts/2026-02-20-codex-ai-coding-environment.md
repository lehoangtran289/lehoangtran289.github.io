---
title: "Setting Up an AI Coding Environment with Codex"
layout: single
date: 2026-02-20
permalink: /posts/codex-ai-coding-environment/
author_profile: false
tags:
  - AI
  - Developer Tools
---

**This post is AI-generated**. It is a practical summary of the official OpenAI guide: [Codex Best Practices](https://developers.openai.com/codex/learn/best-practices/). The core idea is that Codex works best when you treat it less like a one-off assistant and more like a teammate that you configure, guide, and improve over time.

{% include toc %}

## 1. Start with strong context and a better prompt

The official guide says Codex is already useful even if your prompt is imperfect, but better prompts make results more reliable, especially in larger repositories.

A good default is to include four things in your prompt:

- **Goal**: what are you trying to change or build?
- **Context**: which files, folders, docs, examples, or errors matter?
- **Constraints**: what standards, architecture, safety rules, or conventions must Codex follow?
- **Done when**: what should be true before the task is complete?

This keeps the task scoped and makes the result easier to review.

### 1.1. Example of a good prompt

```text
Goal:
Write a new blog post summarizing OpenAI Codex best practices.

Context:
- Follow @_posts/2026-03-05-k8s-101.md for structure
- Create the file under @_posts/
- Use the official guide at https://developers.openai.com/codex/learn/best-practices/

Constraints:
- Match the markdown style of existing posts
- Keep the tone practical and concise
- Use official OpenAI docs for Codex-specific claims

Done when:
- The post includes prompting, AGENTS.md, PLANS.md, config, MCP, skills, review, and automations
- The site builds successfully with `bundle exec jekyll build`
```

The guide also recommends choosing reasoning effort based on task difficulty:

- `Low` for fast, well-scoped tasks
- `Medium` or `High` for debugging or more complex changes
- `Extra High` for long, agentic, reasoning-heavy work

## 2. Plan first for difficult tasks

If the task is complex, ambiguous, or hard to describe clearly, the official recommendation is to plan before coding.

The guide suggests three good approaches:

- use Plan mode
- ask Codex to interview you first
- use a `PLANS.md` template for longer-running work

### 2.1. How to use `PLANS.md`

`PLANS.md` is useful for multi-step work where you want Codex to follow an execution plan instead of making up the structure as it goes.

Common cases:

- refactors
- migrations
- large features
- long debugging sessions

Example:

```md
# PLANS.md

## Goal
Summarize the official Codex best-practices guide into a new blog post.

## Scope
- Add one new post under `_posts/`
- Do not modify layout files or unrelated pages

## Steps
1. Inspect recent posts and match their front matter
2. Read the official guide and note the key sections
3. Draft the article with examples for prompts, AGENTS.md, and PLANS.md
4. Run `bundle exec jekyll build`

## Constraints
- Keep the article concise
- Use official OpenAI docs for Codex guidance
- Stay consistent with the current Jekyll site style

## Done when
- The post is added in the expected format
- The content is accurate to the official guide
- The site build passes
```

## 3. Move durable guidance into `AGENTS.md`

Once a prompt pattern works, the official guide recommends moving the repeated guidance into `AGENTS.md`.

`AGENTS.md` is described as an open-format README for agents. It loads into context automatically and is the best place to encode how you and your team want Codex to work in a repository.

A good `AGENTS.md` covers:

- repo layout and important directories
- how to run the project
- build, test, and lint commands
- engineering conventions and PR expectations
- constraints and do-not rules
- what "done" means and how to verify work

The CLI also provides `/init` to scaffold a starter `AGENTS.md`.

### 3.1. Where to put `AGENTS.md`

The official guide says you can create it at multiple levels:

- `~/.codex/AGENTS.md` for personal defaults
- repo-level `AGENTS.md` for shared standards
- subdirectory-level `AGENTS.md` for local rules

If there is a more specific file closer to the current directory, that guidance wins.

### 3.2. Example `AGENTS.md`

```md
# AGENTS.md

## Repo layout
- `_posts/` contains blog posts
- `_pages/` contains static pages
- `images/` contains post images

## Commands
- `bundle exec jekyll build`
- `bundle exec jekyll serve`

## Conventions
- Match front matter used by recent posts
- Use `{% include toc %}` for long technical posts
- Keep markdown compatible with the current Jekyll theme

## Constraints
- Do not modify unrelated posts
- Prefer small diffs
- Use official OpenAI docs for Codex-specific behavior

## Done when
- The requested content is added in the correct format
- The site builds successfully
```

The official advice here is to keep `AGENTS.md` short, practical, and grounded in real mistakes or repeated friction.

## 4. Configure Codex for consistency

The guide recommends separating personal defaults from repo-specific configuration.

A good pattern is:

- keep personal defaults in `~/.codex/config.toml`
- keep repo-specific behavior in `.codex/config.toml`
- use command-line overrides only for one-off situations

Example:

```toml
model = "gpt-5.4"
model_reasoning_effort = "medium"
personality = "pragmatic"

[projects."/Users/hoangtl/Workspace/lehoangtran289.github.io"]
trust_level = "trusted"

[features]
prevent_idle_sleep = true
```

The guide also highlights two important controls:

- approval mode decides when Codex asks permission to run commands
- sandbox mode decides what files and directories it can access

The safe default is to keep both tight at first, then loosen them only for trusted repositories or stable workflows.

## 5. Improve reliability with testing and review

The official guide is explicit that Codex should not just generate code. It should also help test it, check it, and review it.

That includes:

- writing or updating tests
- running the relevant test suite
- checking lint, formatting, or type checks
- confirming the requested behavior
- reviewing the diff for bugs, regressions, or risky patterns

The Codex app supports local diff review, and the `/review` command can be used for PR-style or change-based review. If your team has a `code_review.md`, the guide suggests referencing it from `AGENTS.md` so review behavior stays consistent.

## 6. Use MCP when the context is outside the repo

The official guidance for MCP is simple: use it when the information Codex needs lives outside the repository and changes frequently.

Use MCP when:

- the context lives outside the repo
- the data changes often
- Codex should call a tool instead of relying on pasted text
- the integration should be reusable across users or projects

Do not connect everything at once. Start with one or two tools that remove a real manual loop.

## 7. Turn repeated work into skills

Once a workflow becomes repeatable, the guide recommends packaging it as a skill instead of repeating long prompts.

A skill combines instructions in `SKILL.md` with optional supporting scripts or assets. The official advice is:

- keep each skill scoped to one job
- start with 2 to 3 concrete use cases
- describe what the skill does and when to use it
- add scripts only when they improve reliability

Good examples include:

- log triage
- release note drafting
- PR review against a checklist
- migration planning
- incident summaries
- standard debugging flows

### 7.1. Where to store skills

The official guide says:

- personal skills live in `$HOME/.agents/skills`
- shared team skills can live in `.agents/skills` inside the repository

In some Codex setups, installed skills may also appear under `~/.codex/skills`. The important part is to follow the storage path used by your installation and keep shared skills versioned in the repo when the team should use the same workflow.

## 8. Automate only after the workflow is stable

The guide draws a useful distinction:

- skills define the method
- automations define the schedule

Good automation candidates include:

- summarizing recent commits
- scanning for likely bugs
- drafting release notes
- checking CI failures
- producing standup summaries

If a workflow still needs a lot of steering, it should become a skill first. Automate it only after it becomes predictable.

## 9. Manage sessions and long-running work carefully

The guide also covers session controls and thread management.

The practical advice is:

- keep one thread per coherent task
- stay in the same thread when it is still the same problem
- fork only when the work truly branches
- use subagents for bounded work, not for everything

Useful slash commands mentioned in the guide include:

- `/resume`
- `/fork`
- `/compact`
- `/agent`
- `/status`

## 10. Common mistakes to avoid

The official guide lists several mistakes that are worth avoiding:

- putting durable rules into prompts instead of `AGENTS.md` or a skill
- not telling Codex how to build, test, or verify its own work
- skipping planning on multi-step tasks
- granting broad permissions too early
- automating a workflow before it is reliable manually
- using one thread per project instead of one thread per task

## 11. Final takeaway

The official best-practices guide can be summarized into one workflow:

1. start with a clear prompt
2. plan first when the task is difficult
3. move durable repo guidance into `AGENTS.md`
4. configure Codex to match your environment
5. validate with tests and review
6. use MCP for external context
7. turn repeatable workflows into skills
8. automate only after the workflow is stable

That is the setup that makes Codex more reliable across the CLI, IDE extension, and Codex app.

## References

- OpenAI Developers, [Codex Best Practices](https://developers.openai.com/codex/learn/best-practices/)
- OpenAI Developers, [AGENTS.md Guide](https://developers.openai.com/codex/guides/agents-md)
- OpenAI Developers, [Execution Plans Guide](https://developers.openai.com/cookbook/articles/codex_exec_plans)
