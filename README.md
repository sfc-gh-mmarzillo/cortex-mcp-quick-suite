# Connect Snowflake Cortex Agent to Amazon QuickSight via MCP

This guide walks you through exposing a Snowflake Cortex Agent to Amazon QuickSight using Snowflake's Managed MCP (Model Context Protocol) server.

> **Battle-Tested**: This guide incorporates real-world fixes and workarounds discovered during production deployments. Follow each step carefully to avoid common pitfalls.

## Overview

Amazon QuickSight supports MCP integration for both action execution and data access. By creating an MCP server in Snowflake that wraps your Cortex Agent, QuickSight users can interact with your agent directly from the QuickSight interface.

## Critical Requirements (Don't Skip These!)

Before you begin, be aware of these **critical issues** that are NOT well documented elsewhere:

| Issue | Why It Matters | Solution |
|-------|----------------|----------|
| **QuickSight sends `scope=ALL`** | Snowflake interprets this as requesting a role named "ALL" | Create a role literally called `"ALL"` |
| **Agent needs `execution_environment`** | Without it, agent responds to "hello" but fails on data questions | Must configure via SQL (NOT available in UI!) |
| **`GRANT USAGE ON INTEGRATION`** | OAuth works but data queries fail silently | Must grant the OAuth integration itself |
| **MFA/WebAuthn breaks OAuth** | OAuth flows fail with passkey authentication | Create a dedicated service user |
| **Account URL format** | Wrong format causes "server not found" | Use account locator (e.g., `abc12345`), not org name |

## Prerequisites

Before you begin, ensure you have:

- **Snowflake Requirements**
  - An existing Cortex Agent in Snowflake (or you'll create one via SQL)
  - A Semantic View for Cortex Analyst
  - ACCOUNTADMIN role access
  - A warehouse for query execution

- **Amazon QuickSight Requirements**
  - Amazon QuickSight Author subscription or higher
  - Access to the QuickSight console

## Step-by-Step Guide

### Step 1: Create the "ALL" Role

**Why?** QuickSight sends `scope=ALL` in its OAuth request. Snowflake interprets this as requesting a role named "ALL". Without this role, you'll get: *"The role ALL requested has been explicitly blocked"*

```sql
USE ROLE ACCOUNTADMIN;

-- Create the "ALL" role (quotes preserve the exact case)
CREATE ROLE IF NOT EXISTS "ALL";

-- Verify it was created
SHOW ROLES LIKE 'ALL';
```

### Step 2: Create a Dedicated OAuth User

**Why?** Users with MFA/WebAuthn (passkeys) enabled cannot complete OAuth flows. You'll see: *"The WebAuthn assertion is invalid"*

```sql
-- Create a dedicated user for QuickSight OAuth
CREATE USER IF NOT EXISTS QUICKSIGHT_MCP_USER
    PASSWORD = 'YourSecurePassword123!'  -- Change this!
    DEFAULT_ROLE = "ALL"
    DEFAULT_WAREHOUSE = <YOUR_WAREHOUSE>
    MUST_CHANGE_PASSWORD = FALSE
    COMMENT = 'Service user for QuickSight MCP OAuth';

-- Grant the ALL role to this user
GRANT ROLE "ALL" TO USER QUICKSIGHT_MCP_USER;
```

> **Note**: Snowflake may prompt MFA setup on first login via Snowsight. Complete this once - OAuth will work afterward without MFA prompts.

### Step 3: Create the Agent via SQL (CRITICAL!)

**Why?** Agents created through the Snowsight UI are **missing** the `execution_environment` configuration. Without this, the agent will respond to general questions like "hello" but **fail on all data questions** with: *"The Analyst tool is missing an execution environment"*

> **WARNING**: The Snowsight UI does NOT expose the `execution_environment` setting. You MUST create or update your agent via SQL!

```sql
-- CREATE AGENT VIA SQL (NOT via Snowsight UI!)
CREATE OR REPLACE AGENT <DATABASE>.<SCHEMA>.<AGENT_NAME>
  COMMENT = 'Your agent description'
  FROM SPECIFICATION $$
models:
  orchestration: "auto"
instructions:
  response: "You are a helpful analytics assistant. Provide clear answers with relevant numbers."
  orchestration: "Use the analytics tool to answer data questions."
tools:
  - tool_spec:
      type: "cortex_analyst_text_to_sql"
      name: "analytics"
      description: "Query your data for analytics"
tool_resources:
  analytics:
    semantic_view: "<DATABASE>.<SCHEMA>.<YOUR_SEMANTIC_VIEW>"
    execution_environment:                    # <-- THIS IS CRITICAL!
      type: "warehouse"
      warehouse: "<YOUR_WAREHOUSE>"           # <-- Your warehouse name
      query_timeout: 60                       # <-- Timeout in seconds
$$;
```

**Verify the agent works directly in Snowflake:**

```sql
-- Test with a general question first
SELECT SNOWFLAKE.CORTEX.DATA_AGENT_RUN(
    '<DATABASE>.<SCHEMA>.<AGENT_NAME>',
    $${"messages": [{"role": "user", "content": [{"type": "text", "text": "hello"}]}]}$$
);

-- Then test with a data question
SELECT SNOWFLAKE.CORTEX.DATA_AGENT_RUN(
    '<DATABASE>.<SCHEMA>.<AGENT_NAME>',
    $${"messages": [{"role": "user", "content": [{"type": "text", "text": "What is the total transaction volume?"}]}]}$$
);
```

### Step 4: Create the MCP Server

Create an MCP server that exposes your Cortex Agent:

```sql
CREATE OR REPLACE MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER
  FROM SPECIFICATION $$
tools:
  - title: "<AGENT_DISPLAY_NAME>"
    name: "<AGENT>"
    type: "CORTEX_AGENT_RUN"
    identifier: "<DATABASE>.<SCHEMA>.<AGENT>"
    description: "Your agent description for QuickSight"
$$;
```

> **Tool Types:**
> - `CORTEX_AGENT_RUN`: Runs the agent and **executes SQL**, returning actual results (recommended)
> - `CORTEX_ANALYST_MESSAGE`: Only generates SQL without executing - returns SQL query text only

Verify the server was created:

```sql
SHOW MCP SERVERS IN SCHEMA <DATABASE>.<SCHEMA>;
DESCRIBE MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER;
```

### Step 5: Create OAuth Security Integration

```sql
CREATE OR REPLACE SECURITY INTEGRATION <AGENT>_QUICKSIGHT_OAUTH
    TYPE = OAUTH
    ENABLED = TRUE
    OAUTH_CLIENT = CUSTOM
    OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
    OAUTH_REDIRECT_URI = 'https://quicksight.aws.amazon.com/sn/oauthcallback'
    OAUTH_ISSUE_REFRESH_TOKENS = TRUE
    OAUTH_REFRESH_TOKEN_VALIDITY = 86400
    OAUTH_USE_SECONDARY_ROLES = IMPLICIT
    PRE_AUTHORIZED_ROLES_LIST = ('ALL', 'PUBLIC');
```

Get the OAuth credentials:

```sql
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('<AGENT>_QUICKSIGHT_OAUTH');
```

**Save these values** - you'll need them for QuickSight:
- `OAUTH_CLIENT_ID`
- `OAUTH_CLIENT_SECRET`

### Step 6: Grant All Required Permissions

This is where most setups fail. Grant **ALL** of these permissions:

```sql
USE ROLE ACCOUNTADMIN;

-- =====================================================
-- CRITICAL: Grant usage on the OAuth INTEGRATION itself
-- =====================================================
-- This is often missed and causes silent failures!
GRANT USAGE ON INTEGRATION <AGENT>_QUICKSIGHT_OAUTH TO ROLE "ALL";
GRANT USAGE ON INTEGRATION <AGENT>_QUICKSIGHT_OAUTH TO ROLE PUBLIC;

-- =====================================================
-- Database and Schema Access
-- =====================================================
GRANT USAGE ON DATABASE <DATABASE> TO ROLE "ALL";
GRANT USAGE ON ALL SCHEMAS IN DATABASE <DATABASE> TO ROLE "ALL";
GRANT ALL PRIVILEGES ON DATABASE <DATABASE> TO ROLE "ALL";
GRANT ALL PRIVILEGES ON ALL SCHEMAS IN DATABASE <DATABASE> TO ROLE "ALL";

-- Also grant to PUBLIC as fallback
GRANT USAGE ON DATABASE <DATABASE> TO ROLE PUBLIC;
GRANT USAGE ON ALL SCHEMAS IN DATABASE <DATABASE> TO ROLE PUBLIC;

-- =====================================================
-- Table and View Access
-- =====================================================
GRANT SELECT ON ALL TABLES IN DATABASE <DATABASE> TO ROLE "ALL";
GRANT SELECT ON ALL VIEWS IN DATABASE <DATABASE> TO ROLE "ALL";
GRANT SELECT ON FUTURE TABLES IN DATABASE <DATABASE> TO ROLE "ALL";
GRANT SELECT ON FUTURE VIEWS IN DATABASE <DATABASE> TO ROLE "ALL";

-- Same for PUBLIC
GRANT SELECT ON ALL TABLES IN DATABASE <DATABASE> TO ROLE PUBLIC;
GRANT SELECT ON ALL VIEWS IN DATABASE <DATABASE> TO ROLE PUBLIC;

-- =====================================================
-- CRITICAL: Semantic View Access
-- =====================================================
-- Cortex Analyst queries the semantic view - this grant is required!
GRANT SELECT ON SEMANTIC VIEW <DATABASE>.<SCHEMA>.<SEMANTIC_VIEW> TO ROLE "ALL";
GRANT SELECT ON SEMANTIC VIEW <DATABASE>.<SCHEMA>.<SEMANTIC_VIEW> TO ROLE PUBLIC;

-- =====================================================
-- Warehouse Access
-- =====================================================
GRANT USAGE ON WAREHOUSE <WAREHOUSE> TO ROLE "ALL";
GRANT OPERATE ON WAREHOUSE <WAREHOUSE> TO ROLE "ALL";
GRANT USAGE ON WAREHOUSE <WAREHOUSE> TO ROLE PUBLIC;
GRANT OPERATE ON WAREHOUSE <WAREHOUSE> TO ROLE PUBLIC;

-- =====================================================
-- Agent and MCP Server Access
-- =====================================================
GRANT USAGE ON AGENT <DATABASE>.<SCHEMA>.<AGENT> TO ROLE "ALL";
GRANT USAGE ON AGENT <DATABASE>.<SCHEMA>.<AGENT> TO ROLE PUBLIC;
GRANT USAGE ON MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER TO ROLE "ALL";
GRANT USAGE ON MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER TO ROLE PUBLIC;

-- =====================================================
-- Cortex Database Roles (Required for Cortex features)
-- =====================================================
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE "ALL";
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_ANALYST_USER TO ROLE "ALL";
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_AGENT_USER TO ROLE "ALL";

GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE PUBLIC;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_ANALYST_USER TO ROLE PUBLIC;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_AGENT_USER TO ROLE PUBLIC;
```

### Step 7: Get Your MCP Endpoint URL

Your MCP endpoint URL follows this format:

```
https://<ACCOUNT_LOCATOR>.<REGION>.aws.snowflakecomputing.com/api/v2/databases/<DATABASE>/schemas/<SCHEMA>/mcp-servers/<MCP_SERVER_NAME>
```

**Find your account locator:**

```sql
SELECT CURRENT_ACCOUNT(), CURRENT_REGION();
-- Example result: ABC12345, AWS_US_WEST_2
```

> **WARNING**: Use the account locator format (e.g., `abc12345.us-west-2.aws`), NOT the organization name format (e.g., `myorg-account.us-west-2.aws`). The org name format will cause connection failures!

**Example endpoint:**
```
https://abc12345.us-west-2.aws.snowflakecomputing.com/api/v2/databases/MY_DB/schemas/MY_SCHEMA/mcp-servers/MY_AGENT_MCP_SERVER
```

### Step 8: Configure QuickSight MCP Integration

1. Open the **Amazon QuickSight console**
2. Navigate to **Integrations** and click **Add** (+)
3. Enter your **MCP Server Endpoint** URL from Step 7
4. Click **Next**
5. Select **User authentication (OAuth)**
6. Select **Manual configuration**
7. Enter OAuth details:

   | Field | Value |
   |-------|-------|
   | Client ID | `<OAUTH_CLIENT_ID from Step 5>` |
   | Client Secret | `<OAUTH_CLIENT_SECRET from Step 5>` |
   | Token URL | `https://<ACCOUNT_LOCATOR>.<REGION>.aws.snowflakecomputing.com/oauth/token-request` |
   | Auth URL | `https://<ACCOUNT_LOCATOR>.<REGION>.aws.snowflakecomputing.com/oauth/authorize` |

8. Click **Create and continue**
9. When the Snowflake login appears, use your **QUICKSIGHT_MCP_USER** credentials
10. Complete the authorization flow

### Step 9: Test the Integration

1. In QuickSight, click on your new integration
2. **Test with "hello" first** - this verifies the MCP connection works
3. **Then test with a data question** like "What is the total transaction volume?"

**Expected results:**
- "Hello" → Agent responds with a greeting
- Data question → Agent returns **actual data** (e.g., "The total transaction volume is $2,614,086,316.00")

> **If you see SQL queries instead of results**, the `execution_environment` is missing from your agent. Go back to Step 3 and recreate the agent via SQL.

## Troubleshooting

### Error: "The role ALL requested has been explicitly blocked"

**Cause**: The "ALL" role doesn't exist

**Fix**:
```sql
CREATE ROLE IF NOT EXISTS "ALL";
```

### Error: "The WebAuthn assertion is invalid"

**Cause**: User has MFA with passkeys enabled

**Fix**: Create a dedicated service user without WebAuthn MFA (Step 2)

### Error: "The Analyst tool is missing an execution environment"

**Cause**: Agent was created via Snowsight UI without warehouse config

**Fix**: Recreate the agent via SQL with `execution_environment` block (Step 3)

### Agent responds to "hello" but fails on data questions

**Cause**: Missing `GRANT USAGE ON INTEGRATION`

**Fix**:
```sql
GRANT USAGE ON INTEGRATION <AGENT>_QUICKSIGHT_OAUTH TO ROLE "ALL";
```

### QuickSight returns SQL query instead of actual results

**Cause**: Using `CORTEX_ANALYST_MESSAGE` instead of `CORTEX_AGENT_RUN`

**Fix**: Update MCP server to use `type: "CORTEX_AGENT_RUN"`

### Connection timeout / Server not found

**Cause**: Wrong account URL format

**Fix**: Use account locator format (e.g., `abc12345.us-west-2.aws`), not organization name format

### "Authorization Error" / "Data View Doesn't Exist"

**Cause**: Missing SELECT permission on semantic view

**Fix**:
```sql
GRANT SELECT ON SEMANTIC VIEW <DATABASE>.<SCHEMA>.<SEMANTIC_VIEW> TO ROLE "ALL";
```

### OAuth client integration not found

**Cause**: Client ID is incorrect or OAuth integration was recreated

**Fix**: Get fresh credentials:
```sql
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('<AGENT>_QUICKSIGHT_OAUTH');
```

### 60-Second Timeout Errors

MCP operations in QuickSight have a fixed 60-second timeout.

**Fix**:
- Optimize your agent's queries
- Use simpler prompts
- Ensure `query_timeout` in agent spec is < 60 seconds

## Complete Setup Script

Here's everything in one script. Replace all `<PLACEHOLDERS>`:

```sql
-- ============================================
-- QUICKSIGHT MCP INTEGRATION - COMPLETE SETUP
-- ============================================

USE ROLE ACCOUNTADMIN;

-- Step 1: Create "ALL" role
CREATE ROLE IF NOT EXISTS "ALL";

-- Step 2: Create OAuth user
CREATE USER IF NOT EXISTS QUICKSIGHT_MCP_USER
    PASSWORD = 'YourSecurePassword123!'
    DEFAULT_ROLE = "ALL"
    DEFAULT_WAREHOUSE = <YOUR_WAREHOUSE>
    MUST_CHANGE_PASSWORD = FALSE;
GRANT ROLE "ALL" TO USER QUICKSIGHT_MCP_USER;

-- Step 3: Create Agent WITH execution_environment (MUST use SQL!)
CREATE OR REPLACE AGENT <DATABASE>.<SCHEMA>.<AGENT>
  COMMENT = 'Agent for QuickSight MCP'
  FROM SPECIFICATION $$
models:
  orchestration: "auto"
instructions:
  response: "You are a helpful analytics assistant."
tools:
  - tool_spec:
      type: "cortex_analyst_text_to_sql"
      name: "analytics"
      description: "Query your data"
tool_resources:
  analytics:
    semantic_view: "<DATABASE>.<SCHEMA>.<SEMANTIC_VIEW>"
    execution_environment:
      type: "warehouse"
      warehouse: "<YOUR_WAREHOUSE>"
      query_timeout: 60
$$;

-- Step 4: Create MCP Server
CREATE OR REPLACE MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER
  FROM SPECIFICATION $$
tools:
  - title: "<AGENT_DISPLAY_NAME>"
    name: "<AGENT>"
    type: "CORTEX_AGENT_RUN"
    identifier: "<DATABASE>.<SCHEMA>.<AGENT>"
    description: "Your agent description"
$$;

-- Step 5: Create OAuth Integration
CREATE OR REPLACE SECURITY INTEGRATION <AGENT>_QUICKSIGHT_OAUTH
    TYPE = OAUTH
    ENABLED = TRUE
    OAUTH_CLIENT = CUSTOM
    OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
    OAUTH_REDIRECT_URI = 'https://quicksight.aws.amazon.com/sn/oauthcallback'
    OAUTH_ISSUE_REFRESH_TOKENS = TRUE
    OAUTH_REFRESH_TOKEN_VALIDITY = 86400
    OAUTH_USE_SECONDARY_ROLES = IMPLICIT
    PRE_AUTHORIZED_ROLES_LIST = ('ALL', 'PUBLIC');

-- Step 6: Grant ALL permissions
GRANT USAGE ON INTEGRATION <AGENT>_QUICKSIGHT_OAUTH TO ROLE "ALL";
GRANT USAGE ON INTEGRATION <AGENT>_QUICKSIGHT_OAUTH TO ROLE PUBLIC;

GRANT USAGE ON DATABASE <DATABASE> TO ROLE "ALL";
GRANT USAGE ON ALL SCHEMAS IN DATABASE <DATABASE> TO ROLE "ALL";
GRANT SELECT ON ALL TABLES IN DATABASE <DATABASE> TO ROLE "ALL";
GRANT SELECT ON ALL VIEWS IN DATABASE <DATABASE> TO ROLE "ALL";
GRANT SELECT ON SEMANTIC VIEW <DATABASE>.<SCHEMA>.<SEMANTIC_VIEW> TO ROLE "ALL";
GRANT USAGE ON WAREHOUSE <YOUR_WAREHOUSE> TO ROLE "ALL";
GRANT OPERATE ON WAREHOUSE <YOUR_WAREHOUSE> TO ROLE "ALL";
GRANT USAGE ON AGENT <DATABASE>.<SCHEMA>.<AGENT> TO ROLE "ALL";
GRANT USAGE ON MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER TO ROLE "ALL";

GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE "ALL";
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_ANALYST_USER TO ROLE "ALL";
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_AGENT_USER TO ROLE "ALL";

-- Also grant to PUBLIC
GRANT USAGE ON DATABASE <DATABASE> TO ROLE PUBLIC;
GRANT USAGE ON MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER TO ROLE PUBLIC;
GRANT USAGE ON AGENT <DATABASE>.<SCHEMA>.<AGENT> TO ROLE PUBLIC;

-- Step 7: Get OAuth credentials (save these!)
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('<AGENT>_QUICKSIGHT_OAUTH');

-- Done! Now configure QuickSight with the OAuth credentials.
```

## Known Limitations

- **60-second timeout**: MCP operations in QuickSight have a fixed 60-second timeout
- **No custom headers**: Custom HTTP headers are not supported
- **Static tool lists**: Tool lists remain static after initial registration - delete and recreate integration to refresh
- **No VPC connectivity**: VPC connectivity is not supported for MCP integrations
- **MFA not supported**: Users with WebAuthn/passkey MFA cannot use OAuth - use dedicated service user
- **UI limitation**: `execution_environment` is NOT configurable in Snowsight UI - must use SQL

## Cleanup

To remove the integration:

```sql
-- Drop the MCP server
DROP MCP SERVER IF EXISTS <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER;

-- Drop the OAuth integration
DROP SECURITY INTEGRATION IF EXISTS <AGENT>_QUICKSIGHT_OAUTH;

-- Drop the role (optional)
DROP ROLE IF EXISTS "ALL";

-- Drop the user (optional)
DROP USER IF EXISTS QUICKSIGHT_MCP_USER;
```

Also remove the integration from the QuickSight console under **Integrations**.

## Additional Resources

- [Amazon QuickSight MCP Integration Documentation](https://docs.aws.amazon.com/quick/latest/userguide/mcp-integration.html)
- [Snowflake Cortex Agents Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- [Snowflake MCP Server Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents/mcp-server)
- [Snowflake OAuth Security Integration](https://docs.snowflake.com/en/user-guide/oauth-custom)
