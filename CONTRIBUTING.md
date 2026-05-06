# Contributing to agent-skills

Thanks for helping. This guide covers how to set up a working copy, add to an existing skill, create a new skill, and get your PR through review.

## Development setup

The repository is plain Markdown — there is no build step, no compiled code, and no language runtime to install. A normal `git clone` is enough to start editing.

Optional but recommended:

```bash
# Markdown linter (matches the spec's lowercase / hyphen / no-XML rules informally)
npx markdownlint-cli2 "**/*.md"
```

If your editor supports it, enable a Markdown linter and a YAML linter — every `SKILL.md` starts with YAML frontmatter that has hard validation rules (see below).

## Adding to an existing skill

This is the most common contribution.

### 1. Pick the right reference file

Each skill has a `SKILL.md` (the dispatcher) and one or more files under `references/`. Per the [Agent Skills spec](https://agentskills.io/specification#progressive-disclosure), `SKILL.md` is loaded whenever the skill activates, while `references/*.md` are loaded only when `SKILL.md` points at them. Place new content where it actually belongs:

- **Goes in `SKILL.md`** — gotchas the agent must see before acting, the decision logic for choosing between subfeatures, and a quick-start with the ten most common commands.
- **Goes in `references/`** — exhaustive reference material, advanced workflows, troubleshooting, and anything more than a few hundred tokens.

### 2. Follow the source-grounded convention

For skills that document an external tool (UTM, a CLI, an API, etc.), **every factual claim should trace to a verified source**. Things that earlier review rounds caught in `utmapp/`:

- AppleScript property names that did not match `UTM.sdef`.
- On-disk plist keys that did not match the `CodingKeys` declarations in `Configuration/*.swift`.
- Enum raw values with the wrong casing.
- Command-line flags whose behaviour was misdescribed.

When you change or add a claim, cite the file and line range you read in the upstream source (in your PR description, not in the skill itself). If you can't cite a source, mark the passage as a heuristic or omit it.

### 3. Update cross-references

If you add a new reference file or a new top-level section in an existing one:

- Add a link from `SKILL.md` if an agent needs to know the section exists.
- Update the `## Contents` table of contents at the top of the reference file (required for any reference file longer than 100 lines per the [authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices#structure-longer-reference-files-with-table-of-contents)).
- If you renamed a section heading, search the repo for the old anchor and update every link.

### 4. Update the skills table in `README.md`

Bump the **Files** count and add the new capability to the **Description** column.

## Creating a new skill

A new skill is a top-level directory containing at minimum a `SKILL.md` file.

### Directory layout

```
my-skill/
├── SKILL.md            # required
├── references/         # optional, one level deep
│   └── *.md
├── scripts/            # optional, executable helpers
└── assets/             # optional, templates / data files
```

The directory name **must match** the `name` field in `SKILL.md` frontmatter (spec rule).

### Required frontmatter

```yaml
---
name: my-skill
description: >-
  Operates [thing] on [platform] — [verb list]. Use when the user mentions
  [trigger keywords] or wants to [verb] [thing].
license: MIT
---
```

Frontmatter rules from the [spec](https://agentskills.io/specification#frontmatter):

| Field           | Requirement                                                                                   |
|-----------------|-----------------------------------------------------------------------------------------------|
| `name`          | 1–64 chars; lowercase letters, digits, hyphens; must match the parent directory; no leading/trailing/double hyphens; no reserved words (`anthropic`, `claude`). |
| `description`   | 1–1024 chars; non-empty; third-person verbs ("Operates", "Extracts" — not "I" or "Use this to"); list both **what** the skill does and **when** it should activate. |
| `license`       | Optional; recommend `MIT` to match the repo's existing `LICENSE`.                             |
| `compatibility` | Optional; ≤500 chars. Use only if your skill has real environment requirements.               |
| `metadata`      | Optional; string-to-string map.                                                               |

### Body shape

- Keep `SKILL.md` under 500 lines / 5000 tokens. Move bulk into `references/`.
- All references must be **one level deep** from `SKILL.md`. Do not chain links (`SKILL.md → A.md → B.md`); link from `SKILL.md` directly.
- Use forward slashes in paths.
- Avoid time-sensitive language (`today`, `currently`, dated transitions).
- Tell the agent *when* to load each reference (e.g. "See `references/foo.md` when X"), not just "see references/".

## Local sanity checks

There is no CI yet — please run these before opening a PR:

```bash
# 1. SKILL.md size
wc -l my-skill/SKILL.md             # should be < 500

# 2. References > 100 lines have a TOC
for f in my-skill/references/*.md; do
  l=$(wc -l < "$f")
  h=$(grep -c '^## Contents' "$f")
  echo "$f: $l lines, Contents=$h"
done

# 3. No time-sensitive language
grep -nE 'today|currently[^-]' my-skill/SKILL.md my-skill/references/*.md

# 4. All anchors referenced from SKILL.md resolve
grep -oE '\(references/[a-z]+\.md(#[a-z-]+)?\)' my-skill/SKILL.md
```

Frontmatter can be sanity-checked with `python3 -c 'import yaml,sys; print(yaml.safe_load(open(sys.argv[1]).read().split("---",2)[1]))' my-skill/SKILL.md`.

## Pull request process

### Before submitting

1. Run the sanity checks above.
2. Confirm the parent directory name matches `name` in frontmatter.
3. Check that every anchor link from `SKILL.md` resolves.
4. Run a Markdown linter if you have one installed.

### What reviewers look for

- **Source grounding** — claims trace to verifiable sources (cite them in the PR body).
- **Spec compliance** — frontmatter passes all rules; SKILL.md ≤ 500 lines; refs one level deep; TOC in long refs.
- **Best practices** — concise (no explanations of what Claude already knows); third-person description; defaults rather than menus; gotchas for non-obvious environment facts only; consistent terminology.
- **No time-sensitive language** and no leaked credentials, personal paths, or API keys.

A useful self-review heuristic: imagine a fresh agent given the skill and a representative task. Does the description trigger? Does the agent reach the right reference file? Are example commands runnable as-shown?

### Tone and accuracy

- Prefer fewer, well-grounded claims over many shaky ones. Earlier review rounds on this repo turned up multiple cases where the documented schema disagreed with the source — fixing those is more valuable than adding new content.
- If you discover an inaccuracy in an existing skill, a one-line PR that fixes it is welcome.

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md). By participating, you agree to abide by its terms.

## Reporting security issues

See [SECURITY.md](SECURITY.md). Do not open public issues for prompt injection, leaked credentials in skill content, or inaccurate destructive commands — use [GitHub Security Advisories](https://github.com/ljagiello/agent-skills/security/advisories/new).
