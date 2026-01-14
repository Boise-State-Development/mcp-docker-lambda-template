# MCP Docker Lambda Template

A template for deploying [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) tools as Docker containers running on AWS Lambda.

## Features

- **Python MCP Server**: Simple time-based MCP tool using FastMCP
- **Docker Container**: Lambda-compatible container image
- **AWS CDK Infrastructure**: TypeScript CDK for ECR + Lambda deployment
- **GitHub Actions CI/CD**: Automated build, push, and deploy pipeline
- **Local Development**: Docker Compose for local testing

## Quick Start

### Local Development

```bash
# Start the MCP server locally
docker-compose up --build

# The server will be available at http://localhost:8000
```

### Deploy to AWS

1. Fork this repository
2. Set up AWS OIDC provider for GitHub Actions (see [CLAUDE.md](CLAUDE.md) for details)
3. Add GitHub Secret:
   - `AWS_DEPLOY_ROLE_ARN`: IAM role ARN for GitHub Actions to assume
4. Push to `main` branch to trigger deployment

### Manual Deployment

```bash
# Install CDK dependencies
cd infrastructure && npm install && cd ..

# Set environment variables
export CDK_AWS_ACCOUNT_ID="your-account-id"
export CDK_AWS_REGION="us-west-2"

# Build and deploy
./scripts/stack-mcp-lambda/build-docker.sh
./scripts/stack-mcp-lambda/push-docker.sh
./scripts/stack-mcp-lambda/deploy.sh
```

## Project Structure

```
├── mcp-tool/                    # Python MCP server
│   ├── app.py                   # Tool implementation
│   ├── Dockerfile               # Lambda container
│   └── Dockerfile.local         # Local dev container
├── infrastructure/              # CDK infrastructure
│   ├── lib/mcp-lambda-stack.ts  # Lambda + ECR stack
│   └── bin/app.ts               # CDK entry point
├── scripts/                     # CI/CD scripts
└── .github/workflows/           # GitHub Actions
```

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `CDK_PROJECT_PREFIX` | Resource naming prefix | `mcp-docker-lambda` |
| `CDK_AWS_REGION` | AWS region | `us-west-2` |
| `CDK_LAMBDA_MEMORY_MB` | Lambda memory | `512` |
| `CDK_LAMBDA_TIMEOUT_SECONDS` | Lambda timeout | `30` |

## Extending the Template

Add new tools in `mcp-tool/app.py`:

```python
@mcp.tool()
def my_tool(param: str) -> dict:
    """Tool description."""
    return {"result": param}
```

See [CLAUDE.md](CLAUDE.md) for detailed documentation.

## License

MIT
