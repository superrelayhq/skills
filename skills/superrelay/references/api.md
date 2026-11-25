# Superrelay REST API and MCP server

Full reference with request and response schemas: https://docs.superrelay.com/docs/api-reference (OpenAPI document behind it is the source of truth).

## REST

Base URL `https://api.superrelay.com`. Every request sends `Authorization: Bearer rl_live_…`.

```bash
curl https://api.superrelay.com/v1/public/conversations?status=open \
  -H "Authorization: Bearer $RELAY_API_KEY"
```

### Endpoints

| Method | Path | Notes |
| --- | --- | --- |
| GET | `/v1/public/conversations` | `limit`, `cursor`, `productId` (`none` = untriaged), `search`, `status`, `priority`, `assignee`, `label`, `contactId` |
| POST | `/v1/public/conversations` | `{ contactId, inboxId?, productId? }` |
| GET | `/v1/public/conversations/{id}` | Conversation plus a page of messages |
| PATCH | `/v1/public/conversations/{id}` | `status` (`open`/`pending`/`resolved`), `priority` (`low`/`medium`/`high`/`urgent`/`null`), `assignee`, `labels`, `customAttributes` |
| POST | `/v1/public/conversations/{id}/messages` | `{ content, role?: "outgoing" \| "incoming", agentUserId? }`; `outgoing` reaches the customer |
| GET, POST | `/v1/public/tickets` | List (`status`, `priority`, `ownerId`, `contactId`, `companyId`, `productId`, `conversationId`, `search`, `limit` 1–100, `cursor`) / create (`title`, `description` Markdown, `contactId` or `contactEmail`, `status`, `priority`, `ownerId`, `dueAt`, …; `Idempotency-Key`) |
| GET | `/v1/public/tickets/statuses` | Workflow stage keys |
| GET, PATCH, DELETE | `/v1/public/tickets/{id}` | `{id}` is an id or display id (`HD-12`); GET includes public replies (`replies_limit`, `replies_cursor`); DELETE is permanent |
| POST | `/v1/public/tickets/{id}/replies` | `{ content }` Markdown; emails the customer; `Idempotency-Key` |
| GET, POST | `/v1/public/contacts` | List (`search`, `email`, `externalId`) / create |
| GET, PATCH | `/v1/public/contacts/{id}` | `companyId: null` unlinks the company |
| GET, POST | `/v1/public/companies` | |
| GET, PATCH | `/v1/public/companies/{id}` | |
| GET | `/v1/public/companies/{id}/contacts` | |
| GET | `/v1/public/products`, `/v1/public/products/{id}` | Read-only |
| GET | `/v1/public/knowledge-bases`, `/v1/public/knowledge-bases/{id}` | Read-only |
| GET, POST | `/v1/public/knowledge-bases/{id}/categories` | |
| GET, PATCH, DELETE | `/v1/public/categories/{id}` | |
| GET, POST | `/v1/public/knowledge-bases/{id}/articles` | `include_content=true&limit=200` returns whole articles; create takes `title` plus `contentMarkdown` or `content` (HTML) |
| GET, PATCH, DELETE | `/v1/public/articles/{id}` | `status: "published"` publishes |
| POST | `/v1/public/knowledge-bases/{id}/preview` | Render Markdown through the sanitizer without saving |
| POST | `/v1/public/uploads` | Multipart; returns a URL for article images and files |
| GET | `/v1/public/knowledge-bases/{id}/suggestions` | Topics the help center does not answer yet |
| GET | `/v1/public/kb-suggestions/{id}` | |
| POST | `/v1/public/kb-suggestions/{id}/approve`, `…/dismiss` | |
| GET, POST | `/v1/public/feature-requests` | |
| GET, PATCH | `/v1/public/feature-requests/{id}` | |
| GET, POST | `/v1/public/feature-requests/{id}/links` | Attach the PR, commit or issue that ships it |
| GET, POST | `/v1/public/changelog` | Create makes a draft |
| GET, PATCH | `/v1/public/changelog/{id}` | |
| POST | `/v1/public/changelog/{id}/publish` | Customer-facing: completes linked requests, emails voters, closes tickets |
| POST | `/v1/public/whatsapp/groups` | Plus `/{conversationId}/participants` and `/{conversationId}/admins` |

`/v1/public/stores*` is a deprecated alias of `/companies*` until 16 February 2027. Use `/companies`.

### Conventions

- Lists return `{ "data": [], "has_more": true, "next_cursor": "…" }`. Pass `cursor` back with `limit` (1–200). Cursors are opaque.
- Errors return `{ "error": "message" }`: `401` bad key, `403` missing scope, `404` not in this workspace, `400` validation.
- Creates that can be retried accept an `Idempotency-Key` header (articles, feature requests, changelog draft and publish, tickets and ticket replies).
- Article reads return both `content` (HTML) and `contentMarkdown`.

## MCP server

`POST https://api.superrelay.com/v1/mcp`, Streamable HTTP, stateless (one JSON-RPC message per request, JSON responses). Auth: OAuth 2.1 sign-in (Claude, ChatGPT, Grok, Claude Code) or `Authorization: Bearer rl_live_…`.

```bash
claude mcp add --transport http superrelay https://api.superrelay.com/v1/mcp
claude mcp add --transport http superrelay https://api.superrelay.com/v1/mcp --header "Authorization: Bearer rl_live_…"
```

| Scope | Tools |
| --- | --- |
| `conversations:read` | `list_conversations`, `get_conversation` |
| `conversations:write` | `reply_to_conversation`, `update_conversation`, `create_conversation` |
| `tickets:read` | `list_tickets`, `get_ticket` |
| `tickets:write` | `create_ticket`, `update_ticket`, `reply_to_ticket`, `delete_ticket` |
| `products:read` | `list_products` |
| `feature_requests:read` | `list_feature_requests`, `get_feature_request`, `list_feature_request_links` |
| `feature_requests:write` | `create_feature_request`, `update_feature_request`, `add_feature_request_link` |
| `changelog:read` | `list_changelog_entries`, `get_changelog_entry` |
| `changelog:write` | `create_changelog_entry`, `update_changelog_entry`, `publish_changelog_entry` |
| `knowledge_bases:read` | `list_knowledge_bases`, `list_kb_articles`, `get_kb_article` |
| `knowledge_bases:write` | `create_kb_article`, `update_kb_article` |
| `contacts:read` | `list_contacts`, `get_contact` |
| `contacts:write` | `create_contact`, `update_contact` |

Lists default to 20 items; pass `cursor` from a result to continue. `reply_to_conversation` and `reply_to_ticket` over OAuth send as the person who connected; with an API key as the workspace bot. Over OAuth, `tools/list` shows every tool the person's role allows; a tool the connection was not approved for returns a missing-scope result with an `mcp/www_authenticate` challenge, and reconnecting approves it.
