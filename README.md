# agent-skills

[Agent Skills](https://agentskills.io) for driving Mac apps and dev workflows from an AI agent. Works with any tool that supports the Agent Skills spec, including [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Installation

```bash
npx skills add ljagiello/agent-skills
```

## Skills

| Skill | Files | Description |
|-------|-------|-------------|
| **utmapp** | 6 | Drive [UTM](https://mac.getutm.app/) virtual machines on macOS via `utmctl` and AppleScript / JXA — list/create/start/stop/suspend/clone/delete VMs, run commands inside guests, transfer files, query guest IPs, send keyboard or mouse input, forward USB devices, hand-edit `.utm` config.plist, and walk through installing Linux / Windows 11 ARM / Windows on Intel / macOS guests. Covers both the QEMU backend (cross-architecture emulation) and the Apple Virtualization backend. |
| **cleanshotx** | 2 | Drive [CleanShot X](https://cleanshot.com) on macOS via its `cleanshot://` URL scheme — capture area / window / fullscreen / scrolling / previous-area screenshots, record the screen, OCR text from images or screen regions, pin images, open files in the annotator, manage history and Quick Access overlays, toggle desktop icons, and open specific settings tabs. Covers all 19 commands with full parameter and version-gating reference. |

## Usage

Skills are loaded automatically by the agent based on the prompt — mention UTM, `utmctl`, or a `.utm` bundle and the `utmapp` skill activates; mention CleanShot, `cleanshot://`, scripted macOS screenshots, or screen-region OCR and `cleanshotx` activates. The agent reads each skill's `SKILL.md` first and then pulls in its domain-specific reference files only as needed for the task at hand.

## Contributing

Each skill follows the [Agent Skills specification](https://agentskills.io/specification): a `SKILL.md` with YAML frontmatter at the top of the skill directory, plus optional `references/`, `scripts/`, and `assets/` subdirectories. Reference files are loaded on demand so the metadata budget stays small.

When adding or editing a skill, every factual claim should trace back to a verified source — for `utmapp` that meant `UTMCtl.swift`, `UTM.sdef`, and the `Configuration/*.swift` `CodingKeys` declarations in the upstream UTM repo. Treat the spec's authoring [best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) as the bar.

## Evals

Two layers of automated checks live under [`evals/`](evals/):

- **Static validator** (`python3 evals/validate.py`) — stdlib-only, deterministic, free. Verifies frontmatter limits, name/dirname match, body line count, and that every Markdown reference resolves to a real file.
- **Behavioral evals** (`npx promptfoo eval --config evals/promptfooconfig.yaml`) — drive Claude with each skill loaded as the system prompt and assert that it emits valid commands, applies the skill's gotchas, and refuses to invent missing features. Requires `ANTHROPIC_API_KEY`.

See [`evals/README.md`](evals/README.md) for the full setup, how the prompt function wires SKILL.md into the system prompt, and how to add cases for a new skill.

## License

MIT

Created by [Lukasz Jagiello](https://lukasz.jagiello.org).
