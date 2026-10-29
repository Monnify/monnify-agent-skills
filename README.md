# Monnify Agent Skill

Give your AI coding assistant working knowledge of the [Monnify](https://monnify.com) payments API, so it writes correct integrations the first time: right base URLs, token handling, payment verification, webhook signature checks, reserved accounts, transfers and more.

This is an [Agent Skill](https://agentskills.io): a folder with a `SKILL.md` and reference files. The assistant reads `SKILL.md` when a task involves Monnify and opens only the reference it needs.

```
monnify/
├── SKILL.md                         # Overview, golden rules, where to look
└── references/
    ├── auth-and-environments.md     # Keys, base URLs, tokens (Node + Python clients)
    ├── collections.md               # Checkout SDK, init transaction, card/transfer/USSD/Pay with Bank, verification
    ├── reserved-accounts.md         # Dedicated virtual accounts
    ├── disbursements.md             # Single & bulk transfers, OTP, status
    ├── webhooks.md                  # Events + HMAC-SHA512 verification (Node, Next.js, Python, PHP)
    ├── other-apis.md                # Refunds, splits, invoices, mandates, verification, bills, wallets
    ├── testing.md                   # Sandbox credentials, test cards, transfer simulator
    └── errors-and-gotchas.md        # Error envelope and the mistakes that break integrations
```

## Install

### Claude Code

```bash
# for one project
mkdir -p .claude/skills && cp -r monnify .claude/skills/
# or for every project on your machine
mkdir -p ~/.claude/skills && cp -r monnify ~/.claude/skills/
```

Then ask something like *"Add Monnify checkout to this Express app and verify payments with a webhook."*

### Claude apps (claude.ai / desktop)

Zip the `monnify` folder and upload it under **Settings → Capabilities → Skills**.

### Other assistants (Cursor, Copilot, Windsurf, Codex, Gemini CLI)

Tools that support the Agent Skills format can load the folder directly. For tools that don't, point the assistant at `monnify/SKILL.md` as a rules/context file, or paste it into the project's agent instructions file (e.g. `AGENTS.md`).

## Try it

- "Integrate Monnify checkout into my Next.js app and confirm payment on the server."
- "Create a reserved account for each new user and credit their wallet from the webhook."
- "Write a payout service that validates the account name, sends the transfer and handles OTP."
- "Review my Monnify webhook handler for security issues."

## Pair it with the Monnify MCP server

The skill teaches an assistant how to **build** with Monnify. The [Monnify MCP server](https://developers.monnify.com/docs/integration/mcp-server) (`@monnify/mcp-server`) lets it **act** on a Monnify account from chat. Use both.

## Source of truth

Content is based on the [Monnify developer docs](https://developers.monnify.com) and the [OpenAPI reference](https://developers.monnify.com/collection/monnify-collection.yml). If anything here disagrees with the live docs, the live docs win. Please open an issue or PR.

## Contributing

Spot an outdated endpoint, missing product, or a mistake developers keep making? Open an issue or PR. Keep `SKILL.md` short; put detail in `references/`.
