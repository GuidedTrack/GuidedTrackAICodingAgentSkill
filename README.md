# GuidedTrack AI Coding Agent Skill

Teaches AI coding agents (Claude Code, Codex, and similar) to write, edit, review, push, and debug [GuidedTrack](https://www.guidedtrack.com/) programs correctly. GuidedTrack has many undocumented sharp edges — this skill encodes them (plus tooling for pushing programs and downloading run data) so agents get things right the first time.

This GuidedTrack skill provides the most effective and efficient way to have AI agents/LLMs create surveys, studies, and behavior change tools which you can then easily publish to collect data. If you want your AI to create and publish surveys or studies, this is the skill for you.

## Prerequisites

Claude Code (or another agent framework that supports skills) and git. The push/pull/data features also need a GuidedTrack account — the agent will ask for your email and password when it needs them.

## Install

**Claude Code, all your projects (recommended):**

```bash
git clone https://github.com/willfind/GuidedTrackAICodingAgentSkill.git ~/.claude/skills/guidedtrack-builder
```

**Claude Code, one project only:** clone into `<project>/.claude/skills/guidedtrack-builder` instead (note the `.claude` — skills elsewhere in a project are not discovered). The folder name is cosmetic — the skill's name comes from SKILL.md.

**Codex / other frameworks:** point your framework's skill mechanism at this folder; `agents/openai.yaml` carries the Codex metadata. See your framework's skill docs.

Works on Windows too: the agent drives the `gt` CLI via Git Bash, with details in SKILL.md's Windows notes.

**Then install `jq`.** Pushing, pulling, and downloading run data need it. Check with `command -v jq || ls ~/bin/jq ~/bin/jq.exe`. If it's missing:

- macOS: `brew install jq`
- Linux (no sudo needed): `mkdir -p ~/bin && curl -fsSL -o ~/bin/jq https://github.com/jqlang/jq/releases/latest/download/jq-linux-amd64 && chmod +x ~/bin/jq` (use `jq-linux-arm64` on ARM)
- Windows (Git Bash): `mkdir -p ~/bin && curl -fsSL -o ~/bin/jq.exe https://github.com/jqlang/jq/releases/latest/download/jq-windows-amd64.exe`

**AI agents installing this skill:** installing jq is a required install step, not an optional extra. After cloning, run the check above. If jq is missing, install it with the matching command, and then tell the user what you installed.

## Use

Nothing to configure — Claude Code loads the skill automatically whenever a task involves GuidedTrack, or invoke it explicitly with `/guidedtrack-builder`. To confirm it's installed, ask Claude "what skills do you have?"

Try it end to end:

> Using the guidedtrack-builder skill, create a 3-question mood survey and push it to GuidedTrack as `zz_skill_smoke_test`

(then delete `zz_skill_smoke_test` from your GuidedTrack account afterward).

## Stay updated

The skill improves regularly — run `git pull` in the cloned folder now and then, or before starting significant GuidedTrack work.

## What's in here

| Path | Purpose |
|---|---|
| `SKILL.md` | Entry point: workflow, validation checklist, pushing to GuidedTrack.com (Mac/Linux/Windows), downloading run data |
| `references/complete_guide.md` | The GuidedTrack language guide: syntax, patterns, keywords, pitfalls |
| `references/custom-services.md` | Custom services: server-side JavaScript routes, the `guidedtrack-db` database library, and calling them with `*service:` |
| `references/gtlint.md` | Using GTLint, an optional third-party linter/formatter for `.gt` files, to automate most of the validation checklist |
| `bin/gt` | The GuidedTrack CLI (bash), with portable fixes and an added `pull` subcommand. Optional install: `mkdir -p ~/bin && cp bin/gt ~/bin/gt && chmod +x ~/bin/gt` (needs `jq`; per-OS instructions in SKILL.md) |
| `agents/openai.yaml` | Codex metadata (ignored by Claude Code) |

## Provenance

Merged 2026-07-24 from the original skill, study-recreation additions (Isaac), a Windows adaptation (Greg), and field-tested improvements from building and launching a production research study. Every non-obvious claim is verified against the [official docs](https://docs.guidedtrack.com/) or observed in production.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) — the short version: every language claim needs a citation or a tested program.
