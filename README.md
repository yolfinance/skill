# Yolfi Webhook Integration — Claude Code Skill

A Claude Code skill that adds **Yolfi crypto payment webhooks** to an existing merchant backend (or creates a new Yolfi webhook integration) with the smallest safe change.

The skill analyzes your codebase, loads the current Yolfi docs, picks between **Adapter Mode** (reuse existing provider webhook) or **Native Mode** (new Yolfi-authenticated route), and preserves your existing payment business logic.

See [`SKILL.md`](./SKILL.md) for the full skill specification.

---

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) installed and authenticated.
- A `YOLFI_API_KEY` — your merchant API key from [yolfi.com](https://yolfi.com).
- A backend repository where Yolfi webhooks should be wired in.

---

## Installation

You have three options. Pick the one that fits your workflow.

### Option 1 — Install via Claude Code marketplace (recommended)

This repository is a Claude Code plugin marketplace. Add it once, then install the skill.

Inside any Claude Code session, run:

```text
/plugin marketplace add yolfinance/skill
/plugin install yolfi-webhook-integration@yolfi
```

To verify the skill is registered:

```text
/plugin list
```

To update later:

```text
/plugin marketplace update yolfi
/plugin update yolfi-webhook-integration@yolfi
```

To uninstall:

```text
/plugin uninstall yolfi-webhook-integration@yolfi
```

### Option 2 — Install as a user-level skill (no marketplace)

Place the skill directly into your user skills directory.

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/yolfi-webhook-integration
curl -fsSL https://raw.githubusercontent.com/yolfinance/skill/main/SKILL.md \
  -o ~/.claude/skills/yolfi-webhook-integration/SKILL.md
```

Or clone the repo into the skills directory:

```bash
git clone https://github.com/yolfinance/skill.git ~/.claude/skills/yolfi-webhook-integration-src
cp ~/.claude/skills/yolfi-webhook-integration-src/SKILL.md \
   ~/.claude/skills/yolfi-webhook-integration/SKILL.md
```

Restart Claude Code (or run `/skills` to refresh). The skill will appear in the available-skills list.

---

## Usage

After installation, open Claude Code in your backend project and ask:

```text
Integrate Yolfi webhooks into this backend. Here is my key: YOLFI_API_KEY=sk_live_...
```

Claude will invoke the `yolfi-webhook-integration` skill, which:

1. Validates `YOLFI_API_KEY`.
2. Loads the current Yolfi docs from `https://docs.yolfi.com/llms-full.txt`.
3. Inspects your codebase (framework, existing webhook routes, plan mapping, idempotency).
4. Picks **Adapter Mode** or **Native Mode**.
5. Confirms the plan with you before editing.
6. Adds Yolfi signature verification on the raw body, wires Yolfi into your existing business pipeline, and verifies the change.

The skill is intentionally conservative: it will **not** refactor your payment system, change plan schemas, replace fulfillment logic, or add new persistence models unless you explicitly ask.

Other trigger phrases that activate the skill:

- "add Yolfi webhooks"
- "integrate Yolfi payments"
- "reuse my existing payment webhook for Yolfi"
- "wire Yolfi into my payment callbacks"
- "add crypto payment notifications"

---

## Configuration

The skill expects `YOLFI_API_KEY` either:

- Passed in the chat message, or
- Already present in your environment / `.env` file.

The skill will offer to add an empty `YOLFI_API_KEY=` entry to `.env.example` and will check `git status` to make sure no real key is tracked.

---

## Publishing your own version

If you fork this skill or want to publish a sibling skill in the same marketplace, edit `.claude-plugin/marketplace.json` and add a new entry to the `plugins` array, pointing at the plugin directory that contains `.claude-plugin/plugin.json` and `skills/<skill-name>/SKILL.md`.

```json
{
  "name": "my-skill",
  "source": "./my-skill",
  "description": "...",
  "version": "1.0.0"
}
```

Then commit, push, and re-run `/plugin marketplace update yolfi` from any Claude Code session.

---

## Repository Layout

```
.
├── .claude-plugin/
│   └── marketplace.json          # Claude Code marketplace manifest
├── yolfi-webhook-integration/
│   ├── .claude-plugin/
│   │   └── plugin.json           # Claude Code plugin manifest
│   └── skills/
│       └── yolfi-webhook-integration/
│           └── SKILL.md          # Installable marketplace skill
├── skills/
│   └── yolfi-webhook-integration/
│       └── SKILL.md              # Source skill body (mirror of the root SKILL.md)
├── SKILL.md                      # Canonical skill spec
└── README.md
```

---

## Links

- Yolfi docs: <https://docs.yolfi.com>
- Yolfi LLM docs (consumed by this skill): <https://docs.yolfi.com/llms-full.txt>
- Claude Code: <https://docs.claude.com/en/docs/claude-code/overview>
- Claude Code plugin marketplaces: <https://docs.claude.com/en/docs/claude-code/plugins>

---

## License

See [`LICENSE`](./LICENSE) if present, otherwise contact Yolfi for usage terms.
