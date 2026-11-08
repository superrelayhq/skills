---
name: superrelay-widget
description: Add the Superrelay live chat widget to a website or web app, identify signed-in users, and sign their identity on the server so they get their help desk tickets and one conversation across devices. Use when the user asks to add support chat, a help widget, live chat or the Superrelay widget to their site, to pass the logged-in user to Superrelay, or to set up Superrelay identity verification.
---

# Superrelay chat widget

Install the [Superrelay](https://superrelay.com) chat widget in the user's app. Read the app first: find the root layout or HTML shell every page renders through, how the app knows who is signed in, and where server-side secrets live. Then make the smallest change that fits that structure.

## What the user needs from the dashboard

The agent cannot create these; ask the user for them.

1. **Widget token**: Superrelay dashboard → **Settings → Inboxes → Widgets** → create a widget and connect it to a product. The install snippet carries `data-relay-token`. It is public; it goes in client code.
2. **Relay Secret** (only for identity verification, step 3): **Settings → Products → the product → Identity → Generate**. Starts with `rlys_`. Server-only: an environment variable, never client code, never committed.

Use placeholders (`YOUR_WIDGET_TOKEN`, `RELAY_SECRET`) until the user gives you real values, and say which ones are still placeholders when you finish.

## 1. Load the widget on every page

```html
<script
  src="https://app.superrelay.com/widget/widget.js"
  data-relay-token="YOUR_WIDGET_TOKEN"
  defer
></script>
```

Put it once in the shared layout, not per page. Framework notes:

- **Next.js (App Router)**: in `app/layout.tsx`, `import Script from "next/script"` and render `<Script src="https://app.superrelay.com/widget/widget.js" data-relay-token={process.env.NEXT_PUBLIC_RELAY_WIDGET_TOKEN} strategy="afterInteractive" />` inside `<body>`.
- **Next.js (Pages Router)**: the same `<Script>` in `pages/_app.tsx`.
- **Vite / React / Vue / Svelte SPA**: the plain tag in `index.html`.
- **Rails, Django, Laravel, Express templates**: the plain tag in the base layout.
- **Astro**: the plain tag in the base layout, with `is:inline`.

Optional attributes: `data-widget-position="bottom-left"` (default `bottom-right`) when the corner is taken; `data-relay-api-url` only for a self-hosted API.

The loader is small and fetches the full widget after the page settles, so it does not need to be lazy-loaded by hand.

## 2. Tell Superrelay who the visitor is

When the app knows the signed-in user on the client, queue an `identify` call. The queue works before the script loads:

```html
<script>
  window.$relay = window.$relay || []
  window.$relay.push([
    "identify",
    {
      name: "Jane Doe",
      email: "jane@example.com",
      data: { plan: "pro", userId: "123", signedUpAt: "2026-10-27T09:14:00Z" },
      segments: ["Beta"],
    },
  ])
</script>
```

Fill it from the app's real session object, only with fields the app has. `data` keys become contact attributes that agents see and automations can filter on; send dates as ISO strings or Unix timestamps. Calling `identify` again later (after the user loads) is fine. Never put secrets or payment data in `data`.

Other commands use the same queue: `["open"]`, `["close"]`, `["toggle"]`, `["hide"]`, `["show"]`, `["position", "bottom-left"]`, `["track", "plan_upgraded", { plan: "Pro" }]`. Wire `open` to an existing "Contact support" button instead of adding a new one.

`identify` is not proof of identity. For the help desk tab, verified contacts and one conversation across devices, do step 3.

## 3. Verify the identity on the server

The server signs a short-lived HS256 JWT with `RELAY_SECRET` on each page load and exposes it as `window.RelayEmbed.identity` **before** the widget script tag.

Claims:

| Claim | Required | Value |
| --- | --- | --- |
| `aud` | yes | exactly `"superrelay"` |
| `sub` | yes | the user's stable id in the app; never reused for another person |
| `iat`, `exp` | yes | Unix **seconds**; `exp - iat` at most 86400 (use 300) |
| `email`, `name` | no | matches an existing contact by email, fills the name |
| `organization` | no | `{ externalId, name?, attributes? }`: the account, team or company the user belongs to |
| `attributes` | no | `{ plan: "pro", role: "admin" }`: facts agents see on the contact |

Node (Express, `jsonwebtoken`):

```ts
import jwt from "jsonwebtoken"

const now = Math.floor(Date.now() / 1000)
const relayIdentity = jwt.sign(
  {
    aud: "superrelay",
    sub: user.id,
    email: user.email,
    name: user.name,
    organization: { externalId: user.accountId, name: user.accountName },
    iat: now,
    exp: now + 300,
  },
  process.env.RELAY_SECRET!,
  { algorithm: "HS256" },
)
```

Next.js App Router (`jose`), in the root layout:

```tsx
import { SignJWT } from "jose"

async function relayIdentity(user: { id: string; email: string; name: string }) {
  return new SignJWT({ aud: "superrelay", sub: user.id, email: user.email, name: user.name })
    .setProtectedHeader({ alg: "HS256" })
    .setIssuedAt()
    .setExpirationTime("5m")
    .sign(new TextEncoder().encode(process.env.RELAY_SECRET!))
}

// in <body>, before the widget <Script>:
// <script dangerouslySetInnerHTML={{ __html: `window.RelayEmbed=${JSON.stringify({ identity: { token } })}` }} />
```

Any other stack: any HS256 JWT library with the same claims (PHP `firebase/php-jwt`, Python `PyJWT`, Ruby `jwt`, Go `golang-jwt`). Then on the page:

```html
<script>
  window.RelayEmbed = { identity: { token: "<signed token>" } }
</script>
<script src="https://app.superrelay.com/widget/widget.js" data-relay-token="YOUR_WIDGET_TOKEN" defer></script>
```

Only render the token for signed-in users, only into their own pages. Signed-out visitors get the widget without it and can still chat.

The user then turns on **Helpdesk** for the widget in **Settings → Inboxes → Widgets** if they want the tickets tab.

## Check your work

- The script tag appears exactly once in the rendered HTML of a few different pages.
- `RELAY_SECRET` appears only in server code and the environment example file (`.env.example` with an empty value), never in client bundles. Search the client build output for `rlys_` if there is one.
- The identity script comes before the widget script in the HTML.
- If the app has a Content Security Policy, allow at least `https://app.superrelay.com` in `script-src` and `https://api.superrelay.com` plus `wss://api.superrelay.com` in `connect-src`, then check the browser console for further blocked requests.
- Run the app and open a page if you can: a launcher appears in the bottom corner.

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| No launcher | Wrong or missing `data-relay-token`, or the script or socket blocked by a Content Security Policy (check the browser console) |
| `Signature verification failed` | Token signed with a different secret or not HS256 |
| `aud must be "superrelay"` / `sub is required` | Missing claim |
| `Identity token has expired` / `Token lifetime exceeds 24 hours` | `iat`/`exp` in milliseconds instead of seconds, or a cached token |
| Help desk tab says identity is missing | `window.RelayEmbed` set after the widget script, or malformed |

Docs: https://docs.superrelay.com/docs/implementation/widget and https://docs.superrelay.com/docs/implementation/identity-verification
