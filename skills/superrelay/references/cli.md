# `relay` CLI commands

Package: [`@superrelay/cli`](https://www.npmjs.com/package/@superrelay/cli) (Node 20+). Installs `relay` and `superrelay`, which are the same. Full docs: https://docs.superrelay.com/docs/api-reference/cli

Global options: `--json`, `--profile <name>` (or `RELAY_PROFILE`), `--target <name>` (KB folder target), `--timeout <ms>` (default 30000), `--retries <n>` (reads only, default 2). Every command answers `--help`.

## Setup

| Command | Does |
| --- | --- |
| `relay config --api-key <key>` | Store the key (`--api-url` for another API host, `--show` to print the config path) |
| `relay doctor [dir]` | Read-only check of credentials, API access, selected knowledge base and pending sync |

## Conversations

IDs accept a prefix of at least four characters. Each also works as `relay conversations <command>`.

| Command | Does |
| --- | --- |
| `relay list` (`ls`) | List (`--limit`, `--cursor`, `--all`, `--product <id\|none>`, `--search`, `--status open\|pending\|resolved`, `--priority`, `--assignee <userId\|none>`, `--label`, `--contact`) |
| `relay show <id>` | Details |
| `relay chat <id>` | Messages (`--limit`, `--cursor` for older) |
| `relay reply <id> <msg>` | Send an agent message (`--file <path\|->`, `--agent <userId>`, `--incoming` to log a customer message) |
| `relay resolve <id>` / `reopen <id>` / `pending <id>` | Change status |
| `relay priority <id> <low\|medium\|high\|urgent\|none>` | Set priority |
| `relay label <id> [labels...]` | Replace labels; no labels clears |
| `relay assign <id> [userId]` | Assign (`--unassign`) |
| `relay create --contact <id>` | New conversation (`--inbox`, `--product <id\|none>`) |

## Contacts, companies, products

| Command | Does |
| --- | --- |
| `relay people` (`contacts`) | List (`--search`, `--email`, `--external-id`, `--all`) |
| `relay person <id>` (`contact`) | View |
| `relay contact:create` | `--name`, `--email`, `--shop` |
| `relay contact:update <id>` | `--name`, `--email`, `--shop`, `--company <id\|none>` |
| `relay companies` | List (`--search`) |
| `relay company <id>` | View, with contact count |
| `relay company:contacts <id>` | A company's contacts |
| `relay company:create` / `company:update <id>` | `--name`, `--website`, `--industry`, `--notes` (`none` clears) |
| `relay products` | List (`--archived`) |
| `relay product <id>` | View |

## Feature requests

`relay feature-requests` is an alias. New requests are private by default.

| Command | Does |
| --- | --- |
| `relay requests list` | `--product <id\|none>`, `--status`, `--visibility`, `--sort newest\|votes\|roadmap` |
| `relay requests show <id>` | Request and its links |
| `relay requests create --title <t>` | `--description`, `--file`, `--product`, `--status`, `--visibility`, `--target`, `--key` |
| `relay requests update <id>` | Edit title, description, status, visibility or target |
| `relay requests move <id>` | `--status`, `--visibility`, `--target` |
| `relay requests link <id> <url>` | Attach the PR, commit or issue that ships it (`--title`, `--kind`) |

## Changelog

| Command | Does |
| --- | --- |
| `relay changelog list` | `--product`, `--status draft\|published` |
| `relay changelog show <id>` | Markdown (`--html`) |
| `relay changelog draft` | From `--file` (front matter: `title`, `version`, `type`, `summary`, `requests`) or flags; same file resumes the same draft |
| `relay changelog update <id>` | Edit without publishing (`--request <id...>`, `--clear-requests`) |
| `relay changelog publish <id>` | Completes linked requests, emails voters, replies to and closes their tickets (`--no-notify-voters`, `--no-reply-to-tickets`, `--reply-text`) |

## Knowledge bases

`<kb>` is an id, id prefix or exact name.

| Command | Does |
| --- | --- |
| `relay kb list` | Knowledge bases (`--full-ids`) |
| `relay kb articles <kb>` | Articles (`--status`, `--category <id\|none>`) |
| `relay kb article <id>` | Article as Markdown (`--html`) |
| `relay kb publish <id>` / `unpublish <id>` / `delete <id>` | Change one article |
| `relay kb upload <files...>` | Upload images or files, print their URLs |
| `relay kb suggestions list <kb>` | Unanswered topics (`--status suggested\|approved\|dismissed`) |
| `relay kb suggestions approve <id>` / `dismiss <id>` | Turn into a draft article (`--publish`) / silence (`--for 30d\|90d\|forever`) |
| `relay kb init`, `new`, `check`, `status`, `diff`, `preview`, `stage`, `pull`, `push` | Markdown folder sync; see the `superrelay-docs` skill |

## WhatsApp groups

| Command | Does |
| --- | --- |
| `relay wa:group:create` | `--name`, `--participant` (repeat or comma-separated), `--idempotency-key` |
| `relay wa:group:add <conversationId>` | Add participants |
| `relay wa:group:admin <conversationId> <phone>` | Promote (`--demote`) |

## Migration

`relay migrate --help` imports conversations and contacts from Crisp.
