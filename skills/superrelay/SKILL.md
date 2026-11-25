---
name: superrelay
description: Work in a Superrelay customer support workspace from the terminal or an agent. Read and answer conversations, triage them (status, priority, labels, assignee), open, update, reply to and close help desk tickets, look up contacts and companies, manage feature requests, changelog entries and help center articles through the `relay` CLI, the MCP server or the REST API. Use when the user mentions Superrelay or Relay support, asks to check or reply to support conversations or tickets, triage the inbox, find a customer, update a feature request, or edit help center articles.
---

# Superrelay

[Superrelay](https://superrelay.com) is a support platform: live chat widget, email and WhatsApp inbox, help desk tickets, AI agents, help center, feature request board and changelog. Everything in a workspace is reachable three ways, all backed by the same public API and the same API key scopes:

| Way | Best for | Setup |
| --- | --- | --- |
| `relay` CLI (`@superrelay/cli`) | Shell access, scripts, CI, bulk work | `npm i -g @superrelay/cli` or `npx @superrelay/cli <command>` |
| MCP server `https://api.superrelay.com/v1/mcp` | Agents with MCP tools and no shell | OAuth sign-in, or `Authorization: Bearer rl_live_…` |
| REST API `https://api.superrelay.com/v1/public` | Code that integrates Superrelay | `Authorization: Bearer rl_live_…` |

Prefer the CLI when you can run commands: every command takes `--json`, IDs accept short prefixes, and lists take `--all`. If Superrelay MCP tools (`list_conversations`, `reply_to_conversation`, …) are already connected, use them instead of installing anything.

## Setup

1. Check for credentials: `relay doctor` (or `relay config --show`). `RELAY_API_KEY` and `RELAY_API_URL` in the environment override stored config.
2. If there is no key, ask the user to create one in the Superrelay dashboard under **Connections → API keys** (it starts with `rl_live_`). Never invent one and never print it back.
3. Store it with `relay config --api-key <key>`, or export `RELAY_API_KEY` for one session or CI.

To connect the MCP server in Claude Code instead:

```bash
claude mcp add --transport http superrelay https://api.superrelay.com/v1/mcp
```

then `/mcp` to sign in. Owners and admins get every tool; members connect read-only. The dashboard's **Connections** page shows the same setup for Claude, ChatGPT and Grok, and lists every connected app.

### Scopes

A key only does what its scopes allow; a missing scope fails with an error naming it. Write scopes do not imply read. A key keeps the scopes it was created with, and keys cannot be edited: a key made before a scope existed (such as `tickets:*`, added November 2026) needs replacing. An MCP connection approved before then lists the ticket tools but gets a missing-scope error calling them until the person reconnects.

| Scope | Allows |
| --- | --- |
| `conversations:read` / `conversations:write` | Read conversations and messages / reply, create, change status, priority, labels, assignee |
| `tickets:read` / `tickets:write` | Read help desk tickets and their public replies / open, edit, reply to and delete tickets |
| `contacts:read` / `contacts:write` | Read / create and update contacts |
| `companies:read` / `companies:write` | Read / create and update companies |
| `products:read` | List products |
| `knowledge_bases:read` / `knowledge_bases:write` | Read / write help center articles and categories |
| `feature_requests:read` / `feature_requests:write` | Read / create, edit, move and link feature requests |
| `changelog:read` / `changelog:write` | Read / draft, edit and publish changelog entries |

## Concepts

- **Workspace**: one company's Superrelay account. A key belongs to one workspace.
- **Product**: what the workspace supports (an app, a site). Conversations, requests, changelog entries and knowledge bases belong to a product. `relay products --json` lists them. When a workspace has several, ask which one before writing anything.
- **Conversation**: one thread with a customer over chat, email or WhatsApp. Status `open`, `pending` or `resolved`; priority `low`, `medium`, `high`, `urgent` or none.
- **Ticket**: a help desk ticket, addressed by its display id (`HD-12`) or id. Its `status` is one of the workspace's workflow stage keys (`relay tickets statuses`; `open` and `done` always exist); priority `low`, `medium`, `high` or `urgent`. Descriptions and replies are Markdown. Internal notes are not available through the CLI, MCP or API.
- **Contact** and **company**: the customer and the organization they belong to.
- **Knowledge base**: the articles behind a help center. Articles are `draft` or `published`.
- **Feature request** and **changelog entry**: the public roadmap board and release notes. Publishing an entry completes its linked requests and tells their voters.

## Common tasks

### Read the inbox

```bash
relay list --status open --json                      # open conversations
relay list --status open --priority urgent --all --json
relay list --product none --json                     # not yet routed to a product
relay list --search "refund" --json
relay show <id> --json                               # details
relay chat <id> --limit 50 --json                    # message history
```

IDs accept a prefix of at least four characters; the CLI refuses an ambiguous prefix.

### Triage

```bash
relay priority <id> high
relay label <id> billing vip                         # replaces all labels; no args clears
relay assign <id> <userId>                           # --unassign to clear
relay pending <id>
relay resolve <id>
relay reopen <id>
```

### Reply to a customer

```bash
relay reply <id> "Thanks, I've refunded the order. It shows up in 3-5 days."
relay reply <id> --file reply.md                     # or --file - for stdin
```

A reply reaches the customer right away, in their chat or by email if they are offline. Read the conversation first, draft the reply, and show it to the user before sending unless they told you to send it.

### Help desk tickets

Needs CLI 3.4 or later (`relay --version`); the MCP tools are `list_tickets`, `get_ticket`, `create_ticket`, `update_ticket`, `reply_to_ticket` and `delete_ticket`.

```bash
relay tickets list --status open --priority urgent --json
relay tickets list --search "invoice" --json         # title, HD-12, customer, company
relay tickets show HD-12 --json                      # with public replies, newest first
relay tickets statuses --json                        # the stage keys --status takes
relay tickets create --title "Export fails" --email jane@example.com --file details.md
relay tickets update HD-12 --status done --owner <userId>
relay tickets reply HD-12 "Fixed in today's release. Can you try again?"
relay tickets delete HD-12                           # permanent, see guardrails
```

`create` emails a linked customer that the ticket was opened; a status change and every reply email them too. Show the user what will go out first.

### Customers

```bash
relay people --email jane@example.com --json
relay people --search "Acme" --json
relay person <id> --json
relay companies --search "Acme" --json
relay company:contacts <companyId> --json
relay contact:update <id> --company <companyId>
```

### Feature requests and changelog

```bash
relay requests list --product <productId> --status planned --json
relay requests create --title "Dark mode" --description "…" --product <productId>
relay requests move <id> --status in_progress
relay requests link <id> https://github.com/org/repo/pull/123 --title "Add dark mode"
relay changelog draft --file release.md --product <productId>
relay changelog publish <entryId>                    # customer-facing, see guardrails
```

New requests are private by default. For a full release flow (commits since the last tag → linked requests → draft → publish), use the `superrelay-release` skill.

### Help center articles

```bash
relay kb list --json
relay kb articles <kb> --status published --json
relay kb article <articleId>                         # markdown
relay kb publish <articleId>
relay kb suggestions list <kb> --json                # topics customers ask that no article answers
```

To keep a help center in a folder of Markdown files with pull, diff and push, use the `superrelay-docs` skill.

## Output and errors

- `--json` prints results to stdout and structured errors to stderr; a nonzero exit means failure.
- Lists return `{ data, has_more, next_cursor }`. Pass `--cursor <next_cursor>` or use `--all`.
- Reads retry on 429/502/503/504 (`--retries`, default 2). Writes are never retried automatically; `requests create`, `changelog draft`, `changelog publish` and `tickets create` take an idempotency key, so rerunning the same input resumes instead of duplicating (`tickets reply` only with `--key`).
- `relay <command> --help` lists every flag. The full command list is in [references/cli.md](references/cli.md); REST and MCP details are in [references/api.md](references/api.md).

## Guardrails

- **Ask before anything a customer sees**: `relay reply`, `relay tickets create`, `relay tickets reply`, a ticket status change, `relay changelog publish` (completes requests, emails voters, replies to and closes their tickets), `relay kb publish` and pushing published articles. Show what will go out and wait for a yes.
- **Ask before deleting a ticket**: `relay tickets delete` and `delete_ticket` remove it with its replies and history, cannot be undone and do not prompt. Prefer moving it to `done`.
- Never paste conversation content, customer emails or phone numbers into places outside the workspace (issues, commits, public docs) without the user's say-so.
- Never link a feature request from another product, and never guess a request match; a wrong link emails the wrong voters.
- Keep the API key out of files you commit, logs and replies.
