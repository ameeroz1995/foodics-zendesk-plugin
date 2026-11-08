# foodics-zendesk plugin

Adds the Foodics Zendesk MCP server and its skill to Claude Code in one step.

```bash
claude plugin marketplace add ameeroz1995/foodics-zendesk-plugin
claude plugin install foodics-zendesk@foodics-coe
```

Then run `/mcp`, choose **zendesk → Authenticate** and sign in with your foodics.com Google account.
Everyone starts as a viewer; ask an MCP admin for writer or admin access.

If you added the server or skill by hand before, remove them first:
`claude mcp remove zendesk -s user` and delete `~/.claude/skills/foodics-zendesk`.

Updates: `claude plugin marketplace update foodics-coe`.

The server's code lives in the zendesk-mcp project; this repo holds only the plugin.
