<!--
Before opening this PR, please skim CONTRIBUTING.md. The most common reason
PRs need rework is missing source citations for factual claims. Quoting the
upstream file:line you read makes review fast.
-->

## Summary

<!-- One or two sentences. What does this PR do, and why? -->

## Type of change

- [ ] New skill (added a directory at the repo root with `SKILL.md`)
- [ ] Addition to an existing skill (new section, new reference file, new example)
- [ ] Skill bug fix (factual correction, wrong flag, wrong plist key, broken anchor)
- [ ] Repo housekeeping (README, CONTRIBUTING, CI, templates, etc.)
- [ ] Other (describe below)

## Source citations

<!--
For factual claims about an external tool's surface (CLI flags, config keys,
APIs), cite the upstream file and line range you verified against. Examples:

- "Verified against `utmctl/UTMCtl.swift:230-256` (Start subcommand)"
- "Per `Configuration/UTMQemuConfigurationDrive.swift:39-44` CodingKeys"
- "Tested with UTM 4.6.5 on macOS 15.4 Apple Silicon"

If the change is housekeeping (README, templates, etc.), write "n/a".
-->

## Spec & best-practices checklist

For changes that touch any `SKILL.md` or `references/*.md`:

- [ ] `SKILL.md` body is under 500 lines (and roughly under 5,000 tokens).
- [ ] Frontmatter `name` matches the parent directory name; lowercase + hyphens; no reserved words (`anthropic`, `claude`).
- [ ] `description` is third-person (`Operates`, `Extracts` — not "I" or "Use this to") and lists trigger keywords.
- [ ] `description` ≤ 1024 chars; `compatibility` ≤ 500 chars (if present).
- [ ] All `references/*.md` longer than 100 lines start with a `## Contents` table of contents.
- [ ] Every reference linked from `SKILL.md` is one level deep (no chains).
- [ ] All anchor links resolve.
- [ ] No time-sensitive language (`today`, `currently`, dated transitions).
- [ ] Forward slashes only.
- [ ] No leaked credentials, API keys, or personal paths.

## Test plan

<!--
A bulleted checklist of how a reviewer (or you) can verify the change.
Example for a skill bug fix:

- [ ] On macOS with UTM 4.x, run the documented `utmctl ...` command and confirm output matches.
- [ ] `plutil -p` an existing `.utm/config.plist` and confirm the documented key actually appears.

Example for a new skill:

- [ ] Activate the skill on the trigger prompt listed in `description`.
- [ ] Walk through the SKILL.md as a fresh agent and reach the right reference.
- [ ] Spot-check three claims against the cited upstream sources.
-->
