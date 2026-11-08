---
name: superrelay-release
description: Turn the commits since the last release into a Superrelay changelog entry, link the feature requests it ships, and publish it so voters and tickets hear about it. Use when the user says "cut a release", "write the changelog", "ship it", or after tagging a version.
---

# Superrelay release

Draft and publish a changelog entry for what shipped since the last release. Everything goes through the `relay` CLI (`npm i -g @superrelay/cli`, then `relay config --api-key rl_live_…` or set `RELAY_API_KEY`) or the MCP tools with the same names. The key needs `feature_requests:read`, `feature_requests:write`, `changelog:read`, `changelog:write` and `products:read`; write scopes do not imply read.

## Steps

1. **Pick the product.** A workspace can hold several products, and every request and entry belongs to one. Ask the user which product this release is for if the repo does not make it obvious, then find its id:
   ```bash
   relay products --json
   ```
   Every later query is filtered to that product id; never match requests across the whole workspace.

2. **Collect what shipped.** Work out the release range on the branch being released (`main`, or the branch the user names). Only tags reachable from the endpoint count, so a tag cut on a maintenance branch can never become the base:
   ```bash
   endpoint=HEAD                                   # or the tag the user just created
   git tag --merged "$endpoint" --sort=-creatordate | head -3
   ```
   If the newest reachable tag points at the endpoint itself (`git describe --tags --exact-match "$endpoint"` succeeds), the user just tagged: that tag is the endpoint and the next one in the list is the base. Otherwise the newest reachable tag is the base and `HEAD` the endpoint. Then list the commits and the PRs they came from:
   ```bash
   git log <base>..<endpoint> --pretty='%h %s' --no-merges
   git log <base>..<endpoint> --pretty='%s' | grep -oE '#[0-9]+' | sort -u   # PR numbers in the range
   ```
   Use only PRs whose merge or squash commit is inside that range; do not search PRs by date, since that includes work merged after the endpoint or into other branches. If the range is empty, stop and ask the user which tags to compare. Ignore chores, dependency bumps and internal refactors unless they change behavior users can notice.

3. **Match commits to feature requests.** Pull the product's requests that are in progress or planned and pair each user-visible change with one:
   ```bash
   relay requests list --product <product-id> --status in_progress --json
   relay requests list --product <product-id> --status planned --json
   ```
   A commit in the range (or a PR whose number appears in one) that names a request id, or whose subject clearly matches a request title, ships that request. Work not in the range is not shipped, even if its PR is merged. Before linking, confirm with `relay requests show <id> --json` that `productId` is the release product. When unsure, leave the request unlinked and say so; a wrong link emails the wrong voters and closes unrelated tickets.

4. **Attach the work to the requests.** For every matched request, link the PR (or commit) that shipped it:
   ```bash
   relay requests link <request-id> <pr-url> --title "<PR title>"
   ```

5. **Draft the entry.** Write a markdown file with front matter and create the draft for the product:
   ```markdown
   ---
   title: <Version or headline>
   version: <x.y.z>
   type: feature | improvement | bugfix | announcement
   requests: <request-id>, <request-id>
   ---
   ## What's new
   - <user-facing change, one line, no commit hashes>
   ## Fixes
   - <...>
   ```
   ```bash
   relay changelog draft --file release.md --product <product-id>
   ```
   Write for customers: what changed for them and why it matters, not how it was built. Re-running `draft` with the same file resumes the same draft instead of creating a second one.

6. **Confirm with the user before publishing.** Show the draft (`relay changelog show <id>`) and state exactly what publishing does: the linked requests move to *completed*, their opted-in voters get an email, linked help desk tickets get a reply and are closed, and knowledge base articles that mention the shipped work are flagged for review. Wait for an explicit yes.

7. **Publish.**
   ```bash
   relay changelog publish <id>
   ```
   Use `--no-notify-voters` or `--no-reply-to-tickets` when the user asks to skip those. `publish` is idempotent per entry, so a retried command never notifies twice.

8. **Report back** with the entry id, the requests completed, and the counts the publish command prints.

## Guardrails

- Never publish without step 6. Publishing is customer-facing and cannot be undone for the emails already sent.
- Never link a request from another product, and never invent request links; unmatched changes go in the entry text only.
- Keep private requests private: a linked request's title appears in the entry's "ships" list, so check with the user before linking anything sensitive.
