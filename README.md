# ApproveKit

Get AI-built apps through App Store review, Google Play review and Google OAuth verification.

ApproveKit is a Claude Code plugin plus a remote MCP server. Your coding agent reads the repo, builds an inventory of what the app collects, and runs a free audit against the rules reviewers actually apply. If you want the documents those reviews require, the agent gets them generated and hosted for a one-time fee per app.

## Install

```bash
claude plugin marketplace add yateshaw/approvekit
claude plugin install approvekit@approvekit
```

Then, inside any app repo, tell Claude:

> get my app ready for the App Store

or "prepare the Google OAuth verification", "why would Google Play reject this app", "I need a privacy policy and an account deletion page".

Any MCP-capable agent can use the server without the plugin:

```bash
claude mcp add --transport http approvekit https://approvekit.dev/mcp
```

## What it does

1. **Inventory.** The skill tells the agent how to derive platforms, sign-in methods, permissions, data collected, third-party SDKs, Google OAuth scopes and payments from the repo (Expo, React Native, Flutter, native iOS and Android, web). Your source code never leaves your machine; only the inventory is sent.
2. **Free audit.** 23 rules across Apple's App Review Guidelines, Google Play policy and Google's OAuth verification. Every finding comes with its severity, the fix, and a link to the official source.
3. **Documents (paid, optional).** A privacy policy and an account deletion page hosted at `yourapp.approvekit.dev`, reviewer notes for App Store Connect and Play Console, and for Google OAuth the per-scope justifications and the demo video shot list. Anything the repo cannot answer is marked `[CONFIRM: ...]` for you to fill in; pages go live only when the text is final.

The agent never buys on its own: it reports the findings and the price, and creates a checkout link only after you say yes.

## Pricing

| | |
|---|---|
| Audit | Free |
| Launch Pass (privacy policy, account deletion page, reviewer notes) | US$49 per app |
| Google OAuth Verification Pack (Launch Pass + scope justifications + demo video script) | US$249 per app |

## MCP tools

| Tool | Purpose |
|---|---|
| `audit_app` | Send the inventory, get findings and offers. Returns an `app_token` to reuse. |
| `create_checkout` | Stripe link for a package, after the user agrees. |
| `get_order` | Payment status and, once paid, the documents as Markdown. |
| `publish_document` | Save the final text and put the hosted pages live. |

Endpoint: `https://approvekit.dev/mcp` (Streamable HTTP, no authentication; the app token identifies the app).

## Links

- Site: https://approvekit.dev
- Terms: https://approvekit.dev/terms
- Privacy: https://approvekit.dev/privacy
- Support: support@approvekit.dev

ApproveKit is a product of WyeTech LLC. It is a software tool, not legal advice, and it does not guarantee approval by any platform.
