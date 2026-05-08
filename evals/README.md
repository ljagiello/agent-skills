# Skill evals

Two layers of automated checks for the skills in this repo.

## Layer 1 — Static validator (`validate.py`)

Stdlib-only Python script. Walks every `SKILL.md` under the repo and checks it
against the [Agent Skills specification](https://agentskills.io/specification):

- Frontmatter is well-formed; required `name` and `description` are present
- `name` ≤ 64 chars, lowercase a–z / 0–9 / hyphens, no leading/trailing/double
  hyphens, no reserved tokens (`anthropic`, `claude`), no XML
- `name` matches the parent directory
- `description` ≤ 1024 chars, non-empty
- `compatibility` (if present) ≤ 500 chars
- `SKILL.md` body ≤ 500 lines (warns at 90% of cap)
- Every Markdown link to a `.md` file resolves to a real file
- Reference files are at most one level deep from `SKILL.md`
- No backslash paths or Windows-style paths in the body

```bash
python3 evals/validate.py            # validate every skill under cwd
python3 evals/validate.py utmapp     # validate a single skill
```

Exits non-zero if any skill has ERRORs (warnings are reported but not fatal).

## Layer 2 — Behavioral evals (promptfoo)

Drive Claude with each skill loaded as the system prompt and assert that
its output respects the skill — emits valid commands/URLs, applies the
documented gotchas, and refuses to invent missing features.

### Setup

```bash
export ANTHROPIC_API_KEY=sk-ant-...
npm install -g promptfoo            # or use npx (no install)
```

### Run

```bash
# Full suite (all skills, all scenarios)
npx promptfoo eval --config evals/promptfooconfig.yaml

# One skill while iterating
npx promptfoo eval --config evals/promptfooconfig.yaml \
  --filter-pattern cleanshotx

# Browse the last run as a UI
npx promptfoo view
```

### How it's wired

- `promptfooconfig.yaml` — top-level config: declares the prompt function, the
  Anthropic provider, and includes per-skill test files.
- `prompts/build_prompt.js` — prompt function. Reads `<skillDir>/SKILL.md`,
  strips the frontmatter, and emits a `[system, user]` chat-message pair.
- `tests/<skill>.yaml` — per-skill test cases. Each case sets `vars.skillDir`
  (which skill to load) and `vars.userPrompt` (the user turn), then declares
  promptfoo `assert` blocks.

This layer simulates *content correctness*: given that the agent has loaded
the skill, does it use the skill correctly? It does **not** test *activation*
(does the agent decide to load the skill from just the description?). Treat
description-tuning as a separate problem.

### Adding a new test case

```yaml
# evals/tests/<skill>.yaml
- description: short summary of what this case proves
  vars:
    skillDir: <skill>
    userPrompt: |
      The user message to send.
  assert:
    - type: contains            # or regex, not-contains, llm-rubric, ...
      value: 'something_expected'
    - type: llm-rubric
      value: |
        Plain-English rubric for what the response must demonstrate.
```

See [promptfoo's assertion reference](https://www.promptfoo.dev/docs/configuration/expected-outputs/)
for the full assertion catalog (`equals`, `contains`, `contains-all`,
`not-contains`, `regex`, `javascript`, `similar`, `llm-rubric`, etc.).

### Adding a new skill

1. Create `<skill>/SKILL.md` and any reference files (see [the spec](https://agentskills.io/specification)).
2. Run `python3 evals/validate.py` to confirm it passes the static layer.
3. Add `evals/tests/<skill>.yaml` with at least three behavioral cases.
4. Add a line to `promptfooconfig.yaml`:
   ```yaml
   tests:
     - file://tests/<skill>.yaml
   ```

## What's not here

Per the project decision, neither layer is wired into CI by default. To gate
PRs on the static validator, add a workflow that runs `python3 evals/validate.py`.
The behavioral layer is intentionally manual — it costs API credits and can
be flaky on prompt-sensitive cases; run it before publishing skill changes.
