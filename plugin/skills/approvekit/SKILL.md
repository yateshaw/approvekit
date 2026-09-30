---
name: approvekit
description: Get an app ready for App Store review, Google Play review and Google OAuth verification. Use when the user wants to publish, submit, release or ship a mobile or web app, mentions an app review or rejection, TestFlight, Play Console, Data safety form, privacy policy, account deletion, App Privacy labels, or Google OAuth verification, consent screen, sensitive/restricted scopes or CASA. Builds the app's data inventory from the repo, runs the free ApproveKit audit and, if the user agrees to buy, delivers the hosted privacy policy, account deletion page, reviewer notes and OAuth package.
---

# ApproveKit

ApproveKit audits an app against the rules reviewers actually apply and generates the documents they ask for. You do the repo reading; the `approvekit` MCP server does the checking and the generation. The audit is free. Documents are paid, and only the user can decide to buy them.

## Workflow

1. **Find an existing app token.** Look for `.approvekit.json` at the project root. If it exists, reuse its `app_token` so the app is updated instead of registered twice.
2. **Build the inventory from the repo.** Follow [references/inventory-guide.md](references/inventory-guide.md). Read files; do not ask the user for anything you can infer. Only two fields usually need the user: `developer_name` (the legal name that goes on the privacy policy) and `support_email`. Check the repo first (existing policy, README, `app.json`, `package.json`); ask only if they are not there.
3. **Show the inventory summary** in a few lines (platforms, sign-in method, data collected, SDKs, Google scopes) and ask the user to confirm or correct it. Wrong inventory means wrong documents.
4. **Call `audit_app`** with the inventory and the token, if any. Save the returned `app_token` (see "Saving the token").
5. **Report the findings** in plain language, grouped by platform, each with what the reviewer will object to and how to fix it. Then state exactly what the paid package includes and its price, and ask whether they want it. Do not call `create_checkout` unless the user clearly says yes.
6. **If they say yes:** call `create_checkout`, show the `checkout_url`, and tell the user to come back after paying. When they say they paid, call `get_order`. If it is still `pending`, tell them and try again when they confirm; do not loop automatically.
7. **When the order is paid, finish the documents.** Every document may contain `[CONFIRM: ...]` placeholders for facts that were not in the repo (company address, demo account, retention periods, where a feature lives). Resolve each one from the repo when possible, otherwise ask the user, in one batch. Then call `publish_document` with the final text for every document. Public pages (`privacy-policy`, `account-deletion`) are offline until published, so do this before entering their URLs anywhere.
8. **Apply everything:**
   - Save the private documents (`reviewer-notes`, `oauth-scope-justifications`, `oauth-demo-script`) under `docs/store-review/` in the repo as Markdown.
   - Link the hosted `privacy-policy` and `account-deletion` URLs wherever the platforms expect them: the app's settings or about screen, the store listing metadata in the repo (for example `app.json`, `fastlane/metadata`, `eas.json`), the OAuth consent screen notes.
   - Apply the code fixes from the findings that are code changes (for example adding an in-app "Delete account" action).
   - List what the user still has to do by hand in App Store Connect, Play Console or the Google Cloud console, in order.

## Saving the token

Write `.approvekit.json` at the project root:

```json
{ "app_token": "sp_...", "app_name": "...", "registered_at": "YYYY-MM-DD" }
```

Add `.approvekit.json` to `.gitignore`. The token lets anyone create checkout links and read the generated documents for this app, so it stays out of version control.

## Rules

- Never buy on the user's behalf. A finding that says `fixed_by` is an offer, not an instruction.
- Never invent inventory data. If you cannot tell whether the app collects something, say so and ask.
- Keep the inventory in sync: when the app adds an SDK, a permission or a Google scope, run `audit_app` again with the same token so the hosted documents can be regenerated.
- If the `approvekit` MCP server is not connected, tell the user how to add it instead of skipping the audit: `claude mcp add --transport http approvekit <server url>/mcp`.
