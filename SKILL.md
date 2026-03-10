---
name: expose-agent-to-quick-suite
description: "Expose a Cortex Agent to Quick Suite via Snowflake Managed MCP. Use when: user wants to connect agent to Quick Suite, set up MCP server for Quick Suite, integrate Snowflake with Quick Suite, enable OAuth for Quick Suite MCP. Triggers: quick suite, MCP server, expose agent, quick suite action, aws integration."
---

# Expose Cortex Agent to Quick Suite

Exposes an existing Snowflake Cortex Agent to Quick Suite via Snowflake Managed MCP (Model Context Protocol).

## Prerequisites

- An existing Cortex Agent in Snowflake
- ACCOUNTADMIN role access (or equivalent privileges)
- Quick Suite Author or higher subscription

## Workflow

### Step 1: Gather Agent Information

**Ask user for the following:**

```
To expose your agent to Quick Suite, provide:

1. DATABASE     : Database containing your agent
2. SCHEMA       : Schema containing your agent  
3. AGENT        : Your Cortex Agent name
4. ACCOUNT      : Snowflake account identifier (e.g., abc12345)
5. WAREHOUSE    : Warehouse for query execution
```

If user doesn't know their agent location, suggest:
```sql
SHOW AGENTS IN ACCOUNT;
```

**Store values as:**
- `<DATABASE>` - Database name
- `<SCHEMA>` - Schema name
- `<AGENT>` - Agent name
- `<ACCOUNT>` - Account identifier
- `<WAREHOUSE>` - Warehouse name

**Stop**: Confirm all values before proceeding.

---

### Step 2: Create MCP Server

**Execute:**

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER
  FROM SPECIFICATION $$
    tools:
      - title: "<AGENT>"
        name: "<AGENT>"
        type: "CORTEX_AGENT_RUN"
        identifier: "<DATABASE>.<SCHEMA>.<AGENT>"
        description: "Cortex Agent exposed via MCP"
  $$;
```

**Verify creation:**
```sql
SHOW MCP SERVERS IN SCHEMA <DATABASE>.<SCHEMA>;
```

---

### Step 3: Grant Permissions

**Review these grants with the user before executing:**

```sql
-- Required grants for MCP access
GRANT USAGE ON DATABASE <DATABASE> TO ROLE PUBLIC;
GRANT USAGE ON SCHEMA <DATABASE>.<SCHEMA> TO ROLE PUBLIC;
GRANT USAGE ON MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER TO ROLE PUBLIC;
GRANT USAGE ON AGENT <DATABASE>.<SCHEMA>.<AGENT> TO ROLE PUBLIC;
GRANT USAGE ON WAREHOUSE <WAREHOUSE> TO ROLE PUBLIC;
GRANT SELECT ON ALL TABLES IN SCHEMA <DATABASE>.<SCHEMA> TO ROLE PUBLIC;
```

**Optional grants (ask user if needed):**

```sql
-- If agent uses a semantic model stage:
GRANT READ ON STAGE <DATABASE>.<SCHEMA>.<STAGE_NAME> TO ROLE PUBLIC;

-- If agent uses Cortex Search:
GRANT USAGE ON CORTEX SEARCH SERVICE <DATABASE>.<SCHEMA>.<SERVICE_NAME> TO ROLE PUBLIC;
```

**Stop**: Get explicit approval before granting. User may prefer a custom role instead of PUBLIC.

**If user wants a custom role:**
```sql
CREATE ROLE IF NOT EXISTS <CUSTOM_ROLE>;
-- Replace PUBLIC with <CUSTOM_ROLE> in all grants above
-- Then grant the custom role to users who need access
```

---

### Step 4: Create OAuth Integration

**Execute:**

```sql
CREATE OR REPLACE SECURITY INTEGRATION <AGENT>_QUICK_SUITE_OAUTH
    TYPE = OAUTH
    ENABLED = TRUE
    OAUTH_CLIENT = CUSTOM
    OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
    OAUTH_REDIRECT_URI = 'https://placeholder.example.com/callback'
    OAUTH_ISSUE_REFRESH_TOKENS = TRUE
    OAUTH_REFRESH_TOKEN_VALIDITY = 86400;
```

The redirect URI is a placeholder - it will be updated in Step 7 after Quick Suite generates the real one.

---

### Step 5: Get OAuth Credentials

**Execute:**

```sql
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('<AGENT>_QUICK_SUITE_OAUTH');
```

**Present credentials to user:**

```
Save these credentials - you'll need them for Quick Suite:

OAUTH_CLIENT_ID:     <value from query>
OAUTH_CLIENT_SECRET: <value from query>
```

**Stop**: Ensure user has copied both values before proceeding.

---

### Step 6: Configure Quick Suite (Manual Steps)

**Present these instructions to the user:**

```
QUICK SUITE CONFIGURATION
=========================

1. Open the Quick Suite console

2. Navigate to: Integrations -> Click "Add" (+ button)

3. On the "Create Integration" page, enter:

   Name:                <any descriptive name>
   Description:         <optional description>
   MCP server endpoint: https://<ACCOUNT>.snowflakecomputing.com/api/v2/databases/<DATABASE>/schemas/<SCHEMA>/mcp-servers/<AGENT>_MCP_SERVER

4. Click "Next"

5. Select authentication method: "User authentication (OAuth)"

6. Select configuration: "Manual configuration"

7. Fill in OAuth details:

   Client ID:      <OAUTH_CLIENT_ID from Step 5>
   Client Secret:  <OAUTH_CLIENT_SECRET from Step 5>
   Token URL:      https://<ACCOUNT>.snowflakecomputing.com/oauth/token-request
   Auth URL:       https://<ACCOUNT>.snowflakecomputing.com/oauth/authorize
   Redirect URL:   (leave blank - Quick Suite will auto-generate)
   
8. Click "Create and continue"

9. On the success screen, COPY the generated Redirect URL

10. Click "Next" to review capabilities (optional)

11. Click "Next" to share with users (optional)

Return here with the Redirect URL from step 9.
```

**Stop**: Wait for user to provide the Redirect URL from Quick Suite.

---

### Step 7: Update Redirect URL

**Once user provides the Redirect URL, execute:**

```sql
ALTER SECURITY INTEGRATION <AGENT>_QUICK_SUITE_OAUTH 
    SET OAUTH_REDIRECT_URI = '<REDIRECT_URL_FROM_QUICK_SUITE>';
```

**Verify the update:**
```sql
DESCRIBE SECURITY INTEGRATION <AGENT>_QUICK_SUITE_OAUTH;
```

Confirm `OAUTH_REDIRECT_URI` shows the new URL.

---

### Step 8: Complete Connection

**Present final instructions:**

```
COMPLETE THE CONNECTION
=======================

1. Return to Quick Suite

2. Navigate to Integrations

3. Find your MCP integration (may show as pending)

4. Click to open and authenticate

5. Sign in with your Snowflake credentials

6. Done! Your agent is now available as an action in Quick Suite.

TEST YOUR AGENT
===============

In Quick Suite, use your agent through the action connector.
The agent should respond using your Snowflake Cortex Agent.

IMPORTANT LIMITATIONS
=====================

- MCP operations have a 60-second timeout
- Tool lists are static after registration (refresh manually if server changes)
- Custom HTTP headers are not supported
```

---

## Quick Reference

### Generated Object Names

| Object | Name |
|--------|------|
| MCP Server | `<DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER` |
| OAuth Integration | `<AGENT>_QUICK_SUITE_OAUTH` |

### Key URLs (replace placeholders)

| URL | Value |
|-----|-------|
| MCP Server URL | `https://<ACCOUNT>.snowflakecomputing.com/api/v2/databases/<DATABASE>/schemas/<SCHEMA>/mcp-servers/<AGENT>_MCP_SERVER` |
| Authorization URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/authorize` |
| Token URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/token-request` |

### Scopes

| Role | Scope Value |
|------|-------------|
| PUBLIC | `session:role:PUBLIC` |
| Custom role | `session:role:<CUSTOM_ROLE>` |

---

## Troubleshooting

### "Object does not exist" in Quick Suite
- Verify MCP server exists: `SHOW MCP SERVERS IN SCHEMA <DATABASE>.<SCHEMA>;`
- Check grants were applied: `SHOW GRANTS ON MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER;`

### OAuth errors
- Verify redirect URL matches exactly: `DESCRIBE SECURITY INTEGRATION <AGENT>_QUICK_SUITE_OAUTH;`
- Ensure OAuth integration is enabled: `ENABLED = TRUE`

### Permission denied
- Check role grants: `SHOW GRANTS TO ROLE PUBLIC;`
- Verify warehouse access: `SHOW GRANTS ON WAREHOUSE <WAREHOUSE>;`

### Timeout errors (HTTP 424)
- Quick Suite MCP has a 60-second timeout limit
- Optimize agent queries or use simpler prompts
- Consider breaking complex operations into smaller steps

### Agent not responding
- Test agent directly in Snowflake first
- Check agent has proper tool access and semantic model permissions

### Tools not appearing
- Tool lists are static after initial registration
- Refresh the action connector in Quick Suite to detect server-side changes

---

## Cleanup (if needed)

To remove the integration:

```sql
DROP MCP SERVER IF EXISTS <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER;
DROP SECURITY INTEGRATION IF EXISTS <AGENT>_QUICK_SUITE_OAUTH;
-- Revoke grants manually if using custom role
```

---

## Stopping Points

- After Step 1: Confirm agent coordinates
- After Step 3: Review permissions before granting (security checkpoint)
- After Step 5: Ensure credentials are saved
- After Step 6: Wait for Redirect URL from user
- After Step 7: Verify redirect URL update

## Output

- MCP Server exposing Cortex Agent
- OAuth security integration for Quick Suite
- Working connection between Quick Suite and Snowflake Agent
