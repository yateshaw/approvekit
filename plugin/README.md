# ApproveKit plugin for Claude Code

Adds the `approvekit` skill and connects the ApproveKit MCP server.

```bash
claude plugin marketplace add yateshaw/approvekit
claude plugin install approvekit@approvekit
```

Then, inside any app repo: "get my app ready for the App Store" or "prepare Google OAuth verification".

To point the plugin at a different server (for example a local one), set `APPROVEKIT_MCP_URL` before starting Claude Code:

```bash
APPROVEKIT_MCP_URL=http://localhost:8787/mcp claude --plugin-dir ./plugin
```
