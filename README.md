# Cortex Agent → Quick Suite (MCP)

Connect a Snowflake Cortex Agent to Quick Suite using Snowflake's Managed MCP server with OAuth authentication.

## Cortex Code Skill

This repo contains a Cortex Code skill. To use it:

```bash
cortex skill add skills/expose-agent-to-quick-suite
```

Then ask Cortex Code:

> "How do I expose my agent to Quick Suite?"

## What's Inside

| Path | Purpose |
|------|---------|
| `skills/expose-agent-to-quick-suite/SKILL.md` | Interactive skill definition (8 steps) |
| `skills/expose-agent-to-quick-suite/skill_evidence.yaml` | Promotion tracking |

## Prerequisites

### Snowflake
- An existing Cortex Agent in Snowflake
- ACCOUNTADMIN role access (or equivalent privileges)
- A warehouse for query execution

### Quick Suite
- Quick Suite Author subscription or higher
- Access to the Quick Suite console

## Quick Reference

| Object | Name Pattern |
|--------|-------------|
| MCP Server | `<DB>.<SCHEMA>.<AGENT>_MCP_SERVER` |
| OAuth Integration | `<AGENT>_QUICK_SUITE_OAUTH` |

### Key URLs

| URL Type | Template |
|----------|----------|
| MCP Server Endpoint | `https://<ACCOUNT>.snowflakecomputing.com/api/v2/databases/<DATABASE>/schemas/<SCHEMA>/mcp-servers/<AGENT>_MCP_SERVER` |
| Authorization URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/authorize` |
| Token URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/token-request` |

## Known Limitations

- **60-second timeout**: MCP operations have a fixed 60-second timeout (HTTP 424 on exceed)
- **No custom headers**: Custom HTTP headers are not supported in MCP operations
- **Static tool lists**: Tool lists remain static after initial registration; refresh manually
- **No VPC connectivity**: VPC connectivity is not supported for MCP integrations

## Resources

- [Quick Suite MCP Integration Documentation](https://docs.aws.amazon.com/quick/latest/userguide/mcp-integration.html)
- [Snowflake Cortex Agents Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
