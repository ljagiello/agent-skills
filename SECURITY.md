# Security Policy

## About This Repository

This repository contains [Agent Skills](https://agentskills.io) — Markdown documentation that AI agents read and follow. The skills here drive real macOS apps: they teach agents how to start and stop VMs, run commands inside guests, transfer files, send keyboard and mouse input, and hand-edit on-disk configuration. Following them, an agent can perform destructive actions (e.g. `utmctl delete`, force-stopping a VM, overwriting a `.utm/config.plist`).

The skills do not ship executable code beyond shell snippets the agent may run. The security surface is therefore primarily about **content integrity** — making sure the instructions an agent reads are accurate, scoped to the documented tools, and free of injected payloads.

## Reporting Security Issues

Please report the following via [GitHub Security Advisories](https://github.com/ljagiello/agent-skills/security/advisories/new):

- **Prompt injection in skill content** — text that tries to coerce an agent into actions outside the skill's documented scope (e.g. exfiltrating secrets, contacting external servers, running unrelated shell commands).
- **Malicious payloads in example commands** — shell, AppleScript, or JXA snippets that do something other than what the surrounding prose claims.
- **Inaccurate destructive operations** — commands documented as safe that actually destroy or corrupt data (e.g. a wrong `--flag` that silently deletes a VM).
- **Leaked credentials, tokens, or PII** — anything that should not have made it into the documentation.
- **Links to live malicious infrastructure** — URLs in docs pointing at attacker-controlled hosts rather than vendor documentation.

When reporting, please include the file path, the exact line(s), and a brief description of the impact. We aim to acknowledge within 5 business days.

## What Is NOT a Security Issue

- Documentation of fragile or destructive operations clearly labelled as such (e.g. `utmctl delete` having no confirmation, `--force` stop semantics) — the documentation is the warning.
- Snippets that would harm a system if run blindly — agents are expected to apply judgement, and the skill content includes the necessary caveats.
- Disagreements about coding style, terminology, or doc layout — open a regular issue or PR.

## Responsible Use

These skills are intended for the operator's own machines and VMs. The skills assume an interactive Aqua session on macOS and an installed copy of UTM under the operator's control. Running them against systems you do not own or are not authorized to operate is not the intended use case.

If you find a way that the documented workflow can be redirected against a system the operator does not control, please report it through the channel above.
