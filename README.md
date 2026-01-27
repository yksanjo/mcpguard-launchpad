# MCPGuard Launchpad

Deploy your own MCP Security Scanner as a service. Scan MCP configurations for vulnerabilities before they reach production.

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?stackName=mcpguard&templateURL=https://mcpguard-launchpad.s3.amazonaws.com/template.yaml)

## Why Deploy Your Own?

- **Private Scanning** - Keep your configs confidential
- **Unlimited Scans** - No rate limits with API key
- **CI/CD Integration** - Webhook support for automated checks
- **Scan History** - Track security posture over time
- **Customization** - Add your own rules and trusted registries

## What It Detects

| Risk | Detection |
|------|-----------|
| **CRITICAL** | Prompt injection vectors (CVE-2025-49596) |
| **HIGH** | Tool poisoning, suspicious commands |
| **HIGH** | Missing authentication |
| **HIGH** | Unrestricted filesystem access |
| **MEDIUM** | Localhost exposure |
| **MEDIUM** | Exposed environment secrets |
| **MEDIUM** | Unsafe tool metadata |

## Quick Start (5 minutes)

### Prerequisites

1. **AWS Account** - [Create one here](https://aws.amazon.com/free/)

### Step 1: Launch

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?stackName=mcpguard&templateURL=https://mcpguard-launchpad.s3.amazonaws.com/template.yaml)

### Step 2: Configure

| Parameter | What to Enter |
|-----------|---------------|
| Stack name | `mcpguard` |
| Instance Type | `t3.micro` (sufficient) |
| Admin Email | Your email |
| API Key | Leave blank (auto-generated) |

### Step 3: Deploy

1. Click **Next** through options
2. Check "I acknowledge IAM resources"
3. Click **Create stack**
4. Wait 5 minutes

### Step 4: Start Scanning

1. Go to **Outputs** tab
2. Click **ScannerURL**
3. Paste your MCP config
4. Click **Scan Configuration**

## Usage

### Web Interface

Visit your Scanner URL and paste your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/safe-folder"]
    },
    "suspicious-server": {
      "command": "npx",
      "args": ["-y", "unknown-mcp-server"],
      "env": {
        "API_KEY": "sk-secret-key-exposed"
      }
    }
  }
}
```

### API (Public - Rate Limited)

```bash
curl -X POST https://your-mcpguard.com/api/scan \
  -H "Content-Type: application/json" \
  -d '{"config": {"mcpServers": {...}}}'
```

Rate limit: 10 scans/hour per IP

### API (Authenticated - Unlimited)

```bash
# Get your API key
ssh ubuntu@YOUR_IP "cat /opt/mcpguard-service/API_KEY.txt"

# Scan with API key
curl -X POST https://your-mcpguard.com/api/v1/scan \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"config": {"mcpServers": {...}}}'
```

### Response Format

```json
{
  "scan_id": "abc123-...",
  "servers": [
    {
      "name": "suspicious-server",
      "riskLevel": "HIGH",
      "riskScore": 0.82,
      "findings": [
        {
          "id": "MCPG-006",
          "name": "Sensitive Env Vars",
          "severity": "MEDIUM",
          "description": "API key exposed in environment variables"
        }
      ]
    }
  ],
  "summary": {
    "totalServers": 2,
    "highRisk": 1,
    "mediumRisk": 0,
    "lowRisk": 1,
    "highestRisk": "HIGH"
  }
}
```

## CI/CD Integration

### GitHub Actions

```yaml
name: MCP Security Check

on:
  push:
    paths:
      - '**/mcp*.json'
      - '**/claude_desktop_config.json'

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Scan MCP Config
        run: |
          RESULT=$(curl -s -X POST ${{ secrets.MCPGUARD_URL }}/api/v1/scan \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${{ secrets.MCPGUARD_API_KEY }}" \
            -d "{\"config\": $(cat ./mcp-config.json)}")

          echo "$RESULT" | jq .

          HIGH_RISK=$(echo "$RESULT" | jq -r '.summary.highRisk')
          if [ "$HIGH_RISK" -gt 0 ]; then
            echo "::error::High risk MCP servers detected!"
            exit 1
          fi
```

### Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

CONFIG_FILES=$(git diff --cached --name-only | grep -E '(mcp.*\.json|claude_desktop_config\.json)')

for FILE in $CONFIG_FILES; do
  RESULT=$(curl -s -X POST https://your-mcpguard.com/api/v1/scan \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $MCPGUARD_API_KEY" \
    -d "{\"config\": $(cat $FILE)}")

  HIGH_RISK=$(echo "$RESULT" | jq -r '.summary.highRisk')
  if [ "$HIGH_RISK" -gt 0 ]; then
    echo "BLOCKED: High risk MCP config in $FILE"
    echo "$RESULT" | jq '.servers[] | select(.riskLevel == "HIGH")'
    exit 1
  fi
done
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Your AWS Account                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              EC2 Instance (t3.micro)                    │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │
│  │  │  Web UI &    │  │   MCPGuard   │  │   SQLite     │  │ │
│  │  │  REST API    │  │   Scanner    │  │   Database   │  │ │
│  │  │   (Express)  │  │    Engine    │  │ (Scan History│  │ │
│  │  └──────┬───────┘  └──────────────┘  └──────────────┘  │ │
│  │         │                                               │ │
│  │  ┌──────▼─────────────────────────┐                    │ │
│  │  │       Caddy (Reverse Proxy)    │                    │ │
│  │  │            :80/:443            │                    │ │
│  │  └────────────────────────────────┘                    │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
       ┌─────────────┐                ┌─────────────┐
       │  Your CI/CD │                │  Web Users  │
       │  Pipelines  │                │  (Browser)  │
       └─────────────┘                └─────────────┘
```

## Monthly Costs

| Resource | Cost |
|----------|------|
| EC2 t3.micro | ~$8/month |
| Elastic IP | ~$3.65/month |
| Data transfer | ~$1/month |
| **Total** | **~$12-15/month** |

## Detection Rules

| ID | Name | Severity | Description |
|----|------|----------|-------------|
| MCPG-001 | Prompt Injection | Critical | Detects prompt injection in configs |
| MCPG-002 | Tool Poisoning | High | Suspicious shell/exec commands |
| MCPG-003 | Localhost Exposure | Medium | Localhost without auth |
| MCPG-004 | Missing Auth | High | No authentication configured |
| MCPG-005 | Metadata Issues | Medium | Unsafe content in tool metadata |
| MCPG-006 | Sensitive Env Vars | Medium | Exposed secrets in environment |
| MCPG-007 | File System Access | High | Unrestricted file access |

## Known CVEs

MCPGuard detects:

- **CVE-2025-49596**: Prompt injection via tool descriptions
- **CVE-2025-49597**: Tool poisoning via malicious registries
- **CVE-2025-49598**: Dynamic tool modification (rug pull)
- **CVE-2025-49599**: Cross-server data exfiltration
- **CVE-2025-49600**: Token exhaustion attacks

## Pricing Tiers

### Free (This Repo)
- Self-hosted
- Unlimited private scans
- Full API access

### Support Tier ($9/month)
- Priority email support
- Setup assistance
- [Contact us](mailto:support@mcpguard-launchpad.com)

### Setup Service ($99 one-time)
- We deploy and configure
- Custom rule development
- [Book setup](mailto:setup@mcpguard-launchpad.com)

## Security

- **Your configs never leave your infrastructure**
- **API key authentication** for private endpoints
- **Rate limiting** on public endpoints
- **Scan history stored locally** in SQLite

## Contributing

PRs welcome! See [CONTRIBUTING.md](docs/CONTRIBUTING.md).

## License

MIT License - See [LICENSE](LICENSE)

---

**"The S in MCP Stands for Security"**
