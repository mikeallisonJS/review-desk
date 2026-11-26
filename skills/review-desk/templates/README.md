# Templates

- `reviewer.md`: the reviewer's instructions. Fill in `{{OWNER}}`, `{{OWNER_LOGIN}}` and `{{CHAT}}`, and install it
  as the agent-instructions file in the reviewer's workspace.
- `reviewer-permissions.json`: the reviewer's lockdown, in Claude Code's settings format. Install it as
  `.claude/settings.json` in the reviewer's workspace, a folder with no MCP config. Deny rules win over allow rules
  and hold in bypass mode and in subagents, but test that on your version. Add your harness's own tool names to
  `deny`: anything that runs commands, writes files, fetches, messages other sessions, schedules or publishes. For
  another harness, translate the idea: allow reading, searching and read-only subagents, and deny everything else.
- `prompts.md`: the watcher, chat relay and fix dispatcher prompts.
