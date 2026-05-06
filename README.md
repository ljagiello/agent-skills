# agent-skills

[Agent Skills](https://agentskills.io) for driving Mac apps and dev workflows from an AI agent. Works with any tool that supports the Agent Skills spec, including [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Installation

```bash
npx skills add ljagiello/agent-skills
```

## Run with Friday Studio

Want these skills as part of a real workflow — schedules, signals, MCP tools, memory, the works? Drop them into [Friday](https://hellofriday.ai/), the shareable AI workspace runtime from [Tempest Labs](https://hellofriday.ai/).

Friday Studio loads skills into agent context on demand and runs them inside reproducible workspaces that you can trigger from chat, on a cron, or over HTTP. Everything runs locally, your data stays on your machine, and every step is logged so you can see exactly what the agent did.

To add these skills to Friday Studio:

1. Install Friday from [hellofriday.ai](https://hellofriday.ai/) (macOS).
2. Open **Skills** in the Studio sidebar and click **+ Add**.
3. Import individual skills by reference (e.g. `ljagiello/agent-skills/utmapp`), or upload this repo as a folder.
4. Reference them from any `workspace.yml`, or let agents load them automatically based on the skill description.

See the [Friday Skills docs](https://docs.hellofriday.ai/core-concepts/skills) for the full workflow, and the [Friday blog](https://blog.hellofriday.ai/) — including [AI Drift: The Hidden Cost of Building with AI](https://blog.hellofriday.ai/ai-drift-the-hidden-cost-of-building-with-ai-e2b51415b3b0) — for the philosophy behind it.

## Skills

| Skill | Files | Description |
|-------|-------|-------------|
| **utmapp** | 6 | Drive [UTM](https://mac.getutm.app/) virtual machines on macOS via `utmctl` and AppleScript / JXA — list/create/start/stop/suspend/clone/delete VMs, run commands inside guests, transfer files, query guest IPs, send keyboard or mouse input, forward USB devices, hand-edit `.utm` config.plist, and walk through installing Linux / Windows 11 ARM / Windows on Intel / macOS guests. Covers both the QEMU backend (cross-architecture emulation) and the Apple Virtualization backend. |

## Usage

Skills are loaded automatically by the agent based on the prompt — mention UTM, `utmctl`, a `.utm` bundle, QEMU on a Mac, or Apple Virtualization.framework via UTM and the `utmapp` skill activates. The agent reads `SKILL.md` first and then pulls in domain-specific reference files (`utmctl.md`, `applescript.md`, `configuration.md`, `workflows.md`, `troubleshooting.md`) only as needed for the task at hand.

## Contributing

Each skill follows the [Agent Skills specification](https://agentskills.io/specification): a `SKILL.md` with YAML frontmatter at the top of the skill directory, plus optional `references/`, `scripts/`, and `assets/` subdirectories. Reference files are loaded on demand so the metadata budget stays small.

When adding or editing a skill, every factual claim should trace back to a verified source — for `utmapp` that meant `UTMCtl.swift`, `UTM.sdef`, and the `Configuration/*.swift` `CodingKeys` declarations in the upstream UTM repo. Treat the spec's authoring [best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) as the bar.

## License

MIT
