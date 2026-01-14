# MCP Docker Lambda Template

This repository provides a template for deploying MCP (Model Context Protocol) tools as Docker containers running on AWS Lambda.

## Project Structure

```
mcp-docker-lambda-template/
├── mcp-tool/                    # Python MCP server
│   ├── app.py                   # MCP tool implementation
│   ├── requirements.txt         # Python dependencies
│   ├── Dockerfile               # Lambda-compatible container
│   └── Dockerfile.local         # Local development container
├── infrastructure/              # CDK infrastructure (TypeScript)
│   ├── bin/app.ts               # CDK app entry point
│   ├── lib/
│   │   ├── config.ts            # Configuration loader
│   │   └── mcp-lambda-stack.ts  # Lambda + ECR stack
│   ├── cdk.json                 # CDK configuration
│   └── package.json             # Node dependencies
├── scripts/                     # CI/CD automation scripts
│   ├── common/load-env.sh       # Environment configuration
│   └── stack-mcp-lambda/        # Stack-specific scripts
│       ├── install.sh           # Install CDK dependencies
│       ├── build-docker.sh      # Build Docker image
│       ├── push-docker.sh       # Push to ECR
│       ├── synth.sh             # Synthesize CDK templates
│       ├── diff.sh              # Show CDK diff
│       └── deploy.sh            # Deploy to AWS
├── .github/workflows/
│   └── deploy.yml               # GitHub Actions workflow
├── docker-compose.yml           # Local development
└── CLAUDE.md                    # This file
```

## Key Concepts

### MCP Tool (`mcp-tool/`)

The MCP tool uses FastMCP with Streamable HTTP transport, suitable for Lambda's request-response model. The `app.py` file contains:

- Tool definitions using `@mcp.tool()` decorator
- Starlette app creation via `mcp.streamable_http_app()`
- Lambda handler using Mangum ASGI adapter

### Infrastructure (`infrastructure/`)

TypeScript CDK stack that creates:

- **ECR Repository**: Stores Docker images
- **Lambda Function**: Runs the containerized MCP tool
- **Function URL**: Provides HTTP endpoint for MCP access
- **IAM Role**: Lambda execution permissions
- **CloudWatch Logs**: Log group with 2-week retention

### Configuration Flow

Configuration flows through the system via environment variables:

```
GitHub Secrets/Variables
    ↓
.github/workflows/deploy.yml (env section)
    ↓
scripts/common/load-env.sh (exports)
    ↓
scripts/stack-mcp-lambda/*.sh (--context flags)
    ↓
infrastructure/lib/config.ts (loadConfig)
    ↓
infrastructure/lib/mcp-lambda-stack.ts (resources)
```

Key configuration variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `CDK_PROJECT_PREFIX` | Resource naming prefix | `mcp-docker-lambda` |
| `CDK_AWS_REGION` | AWS region | `us-west-2` |
| `CDK_AWS_ACCOUNT_ID` | AWS account ID | Required for deploy |
| `CDK_IMAGE_TAG` | Docker image tag | `latest` |
| `CDK_LAMBDA_MEMORY_MB` | Lambda memory allocation | `512` |
| `CDK_LAMBDA_TIMEOUT_SECONDS` | Lambda timeout | `30` |

## Development Commands

### Local Development

```bash
# Start MCP server locally
docker-compose up --build

# Server runs at http://localhost:8000
# MCP endpoint: POST http://localhost:8000/mcp/v1
```

### Manual Deployment

```bash
# Install CDK dependencies
cd infrastructure && npm install

# Build and push Docker image (requires AWS credentials)
export CDK_AWS_ACCOUNT_ID="123456789012"
export CDK_AWS_REGION="us-west-2"

./scripts/stack-mcp-lambda/build-docker.sh
./scripts/stack-mcp-lambda/push-docker.sh

# Deploy stack
./scripts/stack-mcp-lambda/synth.sh
./scripts/stack-mcp-lambda/deploy.sh
```

## CI/CD Pipeline

The GitHub Actions workflow (`.github/workflows/deploy.yml`) follows these patterns:

1. **Job Isolation**: Each job runs on a fresh runner
2. **Artifact-Driven**: Docker images and CDK output passed as artifacts
3. **Synth Once**: CDK templates synthesized once, reused for diff and deploy
4. **Script-Based**: All logic in `scripts/`, workflow YAML is a thin wrapper
5. **OIDC Authentication**: Uses AWS IAM role assumption via GitHub OIDC provider (no static credentials)

### Required GitHub Secrets

- `AWS_DEPLOY_ROLE_ARN`: IAM role ARN for GitHub Actions to assume via OIDC

### Optional GitHub Variables

- `CDK_PROJECT_PREFIX`: Project prefix (default: `mcp-docker-lambda`)
- `CDK_AWS_REGION`: AWS region (default: `us-west-2`)
- `CDK_LAMBDA_MEMORY_MB`: Lambda memory (default: `512`)
- `CDK_LAMBDA_TIMEOUT_SECONDS`: Lambda timeout (default: `30`)

### Setting up AWS OIDC for GitHub Actions

1. Create an OIDC identity provider in AWS IAM:
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`

2. Create an IAM role with trust policy for your GitHub repository:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
           },
           "StringLike": {
             "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
           }
         }
       }
     ]
   }
   ```

3. Attach policies to the role for ECR, Lambda, CloudFormation, IAM, and CloudWatch access.

4. Add the role ARN as `AWS_DEPLOY_ROLE_ARN` secret in GitHub repository settings.

## Adding New MCP Tools

To add new tools to the MCP server, edit `mcp-tool/app.py`:

```python
@mcp.tool()
def my_new_tool(param1: str, param2: int = 10) -> dict:
    """
    Description of what this tool does.

    Args:
        param1: Description of param1
        param2: Description of param2 (default: 10)

    Returns:
        A dictionary with the result
    """
    return {
        "result": f"Processed {param1} with {param2}"
    }
```

## Common Tasks

### Updating Lambda Configuration

1. Modify `infrastructure/lib/mcp-lambda-stack.ts`
2. Update CDK context in `infrastructure/cdk.json` or environment variables
3. Run `./scripts/stack-mcp-lambda/synth.sh` to verify changes
4. Commit and push to trigger deployment

### Adding Dependencies

1. Add to `mcp-tool/requirements.txt`
2. Rebuild Docker image: `./scripts/stack-mcp-lambda/build-docker.sh`
3. Test locally: `docker-compose up --build`

### Debugging

- **Lambda logs**: Check CloudWatch Logs at `/aws/lambda/{prefix}-mcp-tool`
- **Local debugging**: Run `docker-compose up` and check stdout
- **CDK issues**: Run `./scripts/stack-mcp-lambda/diff.sh` to see pending changes

## Architecture Notes

- **Transport**: Uses Streamable HTTP (single request-response), ideal for Lambda
- **Cold Starts**: Docker Lambda images have ~1-3s cold start; consider provisioned concurrency for production
- **Timeout**: Default 30s; MCP operations should complete within this window
- **Memory**: Default 512MB; increase if processing large payloads
