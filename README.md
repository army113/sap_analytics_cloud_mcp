# [SAP Analytics Cloud MCP Server](https://github.com/JumenEngels/sap_analytics_cloud_mcp)

An [MCP](https://modelcontextprotocol.io/) (Model Context Protocol) server that exposes the SAP Analytics Cloud REST API as 105 tools consumable by any MCP-compatible client (Custom Agent, Claude Desktop, Claude Code, Cursor, etc.).

> **This fork** of [JumenEngels/sap_analytics_cloud_mcp](https://github.com/JumenEngels/sap_analytics_cloud_mcp) carries the fixes and additions below on `main`. The branch `upstream` is a plain copy of the original.
>
> - **OData parameter names**: Claude rejects tool parameters that start with `$`, so the 18 tools with `$top`, `$filter`, `$select`, ... failed to load. They are now `odata_top`, `odata_filter`, `odata_select`, ... and are mapped back before the request ([#1](https://github.com/JumenEngels/sap_analytics_cloud_mcp/issues/1)).
> - **stdout stays clean**: the CSRF fetch logged to stdout, which is the MCP channel, and broke the stream on the first write. It logs to stderr now.
> - **Activity log**: `sac_audit_export` returned the whole log (several MB). It now pages (100 rows by default) and filters by type, user, activity, status, file, text and `from`/`till`.
> - **Monitoring**: `sac_monitoring_get` takes a statistic (`error`, `license`, `user`, `dataManagement`, `backend`, ...), a date range, paging and the performance variant.
> - **New tools**: Content Network lists (private, and public with compact rows and search), public item details and permissions, SCIM2 team by id, user and team search, SCIM2 config/schemas/resource types, file repository transfer and its status, team update and delete, content permissions edit, and `$filter` on the import job list.

## What it does

The server authenticates against an SAC tenant using OAuth 2.0 Client Credentials and manages CSRF tokens and session cookies automatically. It then exposes the full SAC REST API surface as discrete MCP tools grouped into 11 service areas:

| Module | Tools | Covers |
|---|---|---|
| Content | 12 | Stories, Resources, File Repository (list, transfer, transfer status), Repositories, Widget Query |
| Data Export | 22 | Namespaces, Providers, Subscriptions, FactData, MasterData, AuditData, Aggregation, Difference, Currency/Unit tables, Public Dimensions |
| Data Import | 24 | Models, Import Jobs lifecycle (create/upload/validate/run/status), One-click Import, Public Dimensions, Currency Conversions, Unit Conversions |
| Multi Actions | 2 | Execute and poll status |
| Calendar | 3 | Get, update, copy calendar events |
| Content Transport | 12 | Export/import jobs, private and public Content Network items, item details, permissions (read and edit) |
| User Management | 18 | SCIM v2 Users, Teams/Groups (get, search, create, update, patch, delete), Bulk operations, ServiceProviderConfig, Schemas, ResourceTypes |
| Monitoring | 2 | Activity log (paged, filtered), system monitoring statistics and performance statistics |
| Schedule & Publication | 4 | Schedule CRUD |
| Translation | 4 | Artifact metadata, XLIFF download (single + bulk) |
| Smart Query | 1 | Intelligent routing (Relational vs Analytical) |

Plus a `ping` connectivity check tool (105 total), and **Prompt** support for guided workflows.


## Prerequisites

- **Node.js >= 18.0** (required for native `fetch` and `Headers.getSetCookie`)
- **npm**
- An SAP Analytics Cloud tenant with an OAuth client (see below).

### SAC Tenant Setup

1. Open your SAC tenant and go to **System > Administration > App Integration > Add a New OAuth Client**.
2. Set the purpose to **Interactive Usage and API Access**.
3. Note down the following four values:
   - **Base URL** -- `https://<tenant>.<datacenter>.sapanalytics.cloud`
   - **Token URL** -- shown on the OAuth client detail page
   - **Client ID**
   - **Client Secret**

## Installation & Build

```bash
git clone https://github.com/JumenEngels/sap_analytics_cloud_mcp
cd sap_analytics_cloud_mcp
npm install
npm run build
```

## Quick Start: Agent Client

The easiest way to test the server locally (after doing mcp setup above) is using the included **Agent Client**. It connects to the server and uses an LLM (SAP AI Core, Anthropic, OpenAI, etc.) to interact with your SAC tenant.

### 1. Setup Configuration
Navigate to the client directory and run the setup script:

```bash
cd AgentClient
npm install
npm run setup
```

This creates `mcp_agentclient.json`. **Open this file and fill in your credentials:**
1.  **SAC Credentials**: Fill in `sac` section with the values from "SAC Tenant Setup".
2.  **LLM Credentials**: Fill in `aicore`, `anthropic`, or `openai` sections. 
3.  **Default Provider**: Set `"defaultProvider"` to your preferred LLM (e.g., `"genaicore"`, `"anthropic"`).

### 2. Run the Client
```bash
npm run build
npm start
```

The client will automatically load all necessary configuration (both for the LLM and the SAC Server) from your JSON file.

---

## Configuration (For standard MCP Clients)

If you are using **Claude Desktop**, **Cursor**, or another MCP client, you need to configure the server environment variables directly.

The server reads four required environment variables at runtime:

| Variable | Description | Example |
|---|---|---|
| `SAC_BASE_URL` | Tenant root URL (no trailing slash) | `https://mytenant.eu10.sapanalytics.cloud` |
| `SAC_TOKEN_URL` | OAuth token endpoint | `https://mytenant.authentication.eu10.hana.ondemand.com/oauth/token` |
| `SAC_CLIENT_ID` | OAuth Client ID | `sb-abc123-def456` |
| `SAC_CLIENT_SECRET` | OAuth Client Secret | `AbC123...` |

### MCP Client Configuration

#### Claude Desktop
To automatically generate a configuration file with the correct absolute path:

```bash
npm run setup
```

This creates an `mcp_config.json` file in the project root. fill in your SAC credentials there and copy the content to your Claude Desktop config file.

#### Cursor / Other Clients
Add the server with the command `node build/index.js` and set the 4 environment variables in the client's configuration UI.

## Smart Query

The `smart_query` tool simplifies data access by intelligently routing your request:

- **Relational**: Simple `SELECT *` queries are routed to the OData API for raw entity data.
- **Analytical**: Queries with aggregations (`SUM`, `COUNT`, `GROUP BY`) are analyzed and routed to the Widget Query API.

```sql
SELECT SUM(Amount) FROM Model123 GROUP BY Region
```

## Prompts

The server provides pre-defined prompts to guide users through common workflows:

1.  **explore_content**: Discover stories and files in your tenant.
2.  **analyze_story**: Deep dive into a specific story's metadata and dependencies.
3.  **system_health_check**: Quick status check of connectivity and recent failures.

## Deployment

### Docker
```bash
docker build -t sac-mcp-server .
docker run -e SAC_BASE_URL="..." ... sac-mcp-server
```

## Development

Watch mode recompiles on every file change:

```bash
npm run dev
```

### Security Notes

- **Write Operations**: All tools that modify data require `allowalteration=true` argument.
- **Read-only use**: the filtered activity log, the Content Network lists and the SCIM2 searches use POST, because the API takes their filter as a body. They change nothing. Data Export reads with `trackChanges` create a delta subscription on the tenant, and reads with a `deltaid` move that subscription's delta position. Block both if a client must stay read-only.
- **Large answers**: the public Content Network list is several hundred KB with `detail=true`. The default compact rows are about 50 KB for ~100 packages.
- **stdout**: it carries MCP messages only. Log to stderr.

