# Microsoft Graph MCP Server

A Model Context Protocol (MCP) server providing Claude Code with full access to Microsoft 365 services via the Microsoft Graph API.

## Features

**100+ tools** across Microsoft 365 services:

- **Email**: Read, search, send, reply, forward, draft management, attachments, categories, folders
- **Calendar**: Events, meetings, availability, scheduling, responses
- **Contacts**: Contact management, search, company lookup
- **To Do**: Task lists, tasks, checklist items
- **Planner**: Plans, buckets, tasks, assignments
- **Groups**: M365 groups, membership management

## Requirements

- Docker and Docker Compose
- Azure AD application with Microsoft Graph permissions
- Network access (WireGuard tunnel recommended for remote deployment)

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/keelinglogic/ms-graph-mcp.git
cd ms-graph-mcp
```

### 2. Configure Azure AD credentials

```bash
cp .env.example deployment/hermes/.env
# Edit .env with your CLIENT_ID and TENANT_ID
```

### 3. Deploy

```bash
# Local deployment
cd deployment/hermes
docker compose up -d

# Remote deployment (Hermes VPS)
./scripts/deploy-to-hermes.sh
```

### 4. Authenticate

On first run, the server displays a device code. Visit https://microsoft.com/devicelogin and enter the code to authenticate.

### 5. Configure Claude Code

Add to `~/.mcp.json`:

```json
{
  "mcpServers": {
    "microsoft-graph-email": {
      "url": "http://10.100.0.1:8000/mcp",
      "transport": "http"
    }
  }
}
```

## Project Structure

```
ms-graph-mcp/
├── src/
│   ├── mcp_server_v2.py    # Main MCP server
│   └── requirements.txt     # Python dependencies
├── deployment/
│   └── hermes/
│       └── docker-compose.yml
├── scripts/
│   └── deploy-to-hermes.sh  # Deployment automation
├── docs/
│   ├── deployment.md        # Detailed deployment guide
│   └── guides/              # Operational guides
├── .env.example             # Environment template
└── README.md
```

## OAuth Token Persistence

Tokens are stored in a Docker volume (`m365-mcp-data`) at:
```
/data/.athena/credentials/graph_mcp_cache.json
```

Tokens persist across container restarts. No re-authentication needed unless tokens are revoked.

## Documentation

- [Deployment Guide](docs/deployment.md) - Detailed setup instructions
- [Attachment Tool Selection](docs/guides/attachment-tool-selection.md) - When to use which attachment tool
- [HTML Email Formatting](docs/guides/html-email-formatting.md) - Sending formatted emails

## License

MIT License - See [LICENSE](LICENSE) for details.

## Contributing

Contributions welcome. Please open an issue first to discuss proposed changes.
