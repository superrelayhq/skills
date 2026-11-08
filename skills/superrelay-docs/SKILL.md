---
name: superrelay-docs
description: Keep a Superrelay help center in a repository as Markdown files. Pull the knowledge base into a folder, write or update articles, validate, diff and push them back with the `relay kb` commands. Use when the user wants to write help center or knowledge base articles, document a feature for customers, sync docs to Superrelay, or set up docs-as-code for their help center.
---

# Superrelay help center as code

A Superrelay knowledge base (the articles behind a help center) can live in the repository as Markdown and sync both ways with the `relay` CLI (`npm i -g @superrelay/cli` or `npx @superrelay/cli`). The key needs `knowledge_bases:read` and `knowledge_bases:write`.

## First look

```bash
relay doctor                 # credentials, selected knowledge base, pending recovery
cat relay.kb.json 2>/dev/null # repository targets, if the repo is already set up
relay kb list --json         # knowledge bases the key can see
```

Some knowledge bases **deploy from GitHub** through the Superrelay GitHub app; `push` and `stage` then refuse and name the branch to merge to. For those, edit the Markdown files and commit them on a branch; merging to the tracked branch deploys them, and edits made in Superrelay come back as a pull request.

## Set up a folder (once per repository)

```bash
relay kb init docs --kb "Acme Help" --target production   # writes relay.kb.json and docs/.relay-kb.json
relay kb pull                                             # download existing articles
```

`relay.kb.json` holds no API key. Commit it, the article files and `docs/.relay-kb.json` together. Use a separate `--profile` and directory per environment so staging and production article IDs never mix.

## Layout and frontmatter

```text
docs/
  getting-started.md          # no category
  Billing/                    # first folder level = category
    refunds.md
    Advanced/invoicing.md     # deeper folders only organize files
  images/setup.png
```

```md
---
id: 9c8d7e6f-aaaa-4bbb-8ccc-dddd1111eeee
title: How refunds work
status: draft
---

See [Invoices](./Advanced/invoicing.md#invoice-details).

![Setup](../images/setup.png)
```

- Omit `id` on a new article; the first push writes it. Keep it when moving or renaming the file, and never copy it into another article.
- `status` is `draft` (default) or `published`. `icon` and integer `order` are optional. Unknown fields are rejected.
- Relative links to other `.md` files become stable help center links. Local images (JPEG, PNG, WebP, GIF up to 5 MB) and files are uploaded on push. A YouTube, Vimeo or Loom URL on its own line becomes an embed.
- Files and folders starting with `.` or `_`, and the root `README.md`, are ignored; `.relayignore` adds patterns.

Start a new article from a template instead of a blank file:

```bash
relay kb new "Connect your calendar" --template how-to --file Integrations/calendar.md
```

Templates: `blank`, `how-to`, `troubleshooting`, `faq`.

## Writing articles

Before writing, read a few existing articles to match their tone, heading style and length, and read the product code or PR the article describes so every step is accurate. Write for customers: what to click and what they will see, not how it was built. One task per article, a short intro, numbered steps, and a "Troubleshooting" section only when there are real failure cases. New articles start as `status: draft`.

## Review and sync

```bash
relay kb check               # offline validation: frontmatter, IDs, links, assets
relay kb status              # destination and planned changes
relay kb diff                # unified diff against what is live, with publication status
relay kb preview --file Billing/refunds.md   # local HTML through the real sanitizer
relay kb push --dry-run      # the exact plan, no writes
relay kb push                # apply
```

Show the user the `diff` and the dry-run plan, and get a yes before `push`: **edits to published articles go live on push**. To publish exactly what was reviewed, `relay kb stage` then `relay kb push --staged`. Removals need `--prune` on `diff`, `stage` and `push`; never prune without asking.

A published article may link to a draft: the help center shows the link as plain text until the draft is published, so articles can go live one at a time.

## When something goes wrong

- **Remote change or conflict**: someone edited the article in Superrelay. `push` stops before writing anything new. Inspect it with `relay kb diff <file>`. If only the dashboard changed, `relay kb pull` first. If both sides changed, ask the user which wins: `relay kb pull --force` takes the dashboard version, `relay kb push --force` keeps the local one.
- **Interrupted push**: run `relay kb push` again; it replays the pending write from `.relay-kb.json` with the same idempotency key before planning new work. Do not edit `.relay-kb.json` by hand, and in CI keep the folder from the failed run instead of retrying from a fresh checkout.
- **Generated files lost their `id:`**: `relay kb check --restore-ids` puts them back from the manifest.

## CI

`examples/kb-check.yml` in the CLI package is a ready GitHub Actions workflow. `relay kb check --all-targets --format github` validates every target offline, with annotations and no credentials.

Docs: https://docs.superrelay.com/docs/api-reference/knowledge-base-sync
