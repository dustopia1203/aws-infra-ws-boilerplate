# AWS Infrastructure Workspace Template

A pre-configured Cursor IDE workspace for AWS Terraform projects, equipped with MCP servers, agent skills, and the superpowers plugin.

## Prerequisites

| Tool | Purpose |
|------|---------|
| [Terraform CLI](https://developer.hashicorp.com/terraform/install) >= 1.x | Infrastructure-as-code engine |
| [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) | AWS authentication and profile management |
| Python 3.x + [uvx](https://docs.astral.sh/uv/) | Required to run MCP servers locally |
| [Cursor IDE](https://cursor.sh) | Editor with AI agent support |
| Superpowers plugin | Pre-enabled in `.cursor/settings.json` |

## Quick Start

1. Clone or copy this boilerplate into a new project directory
2. Open the directory in Cursor IDE
3. Update `AWS_PROFILE` and `AWS_REGION` in `.cursor/mcp.json` to match your target AWS account
4. Verify MCP servers connect in Cursor's MCP panel (Tools icon in sidebar)
5. Follow the [Workflow](#workflow) section below

## Project Structure

```
.
├── .cursor/
│   ├── mcp.json              # AWS MCP server configurations
│   ├── settings.json          # Superpowers plugin enabled
│   ├── skills/                # Installed agent skills
│   │   └── terraform-skill/   # Terraform/OpenTofu guidance (v1.6.0)
│   └── rules/                 # Agent behavior rules
│       └── workflow.mdc       # Workspace workflow rule
├── docs/
│   └── plans/                 # Design and implementation plans
└── README.md
```

## What's Included

### MCP Servers

Four AWS MCP servers are pre-configured in `.cursor/mcp.json`:

| Server | Type | Purpose |
|--------|------|---------|
| `aws-api-mcp-server` | Local (uvx) | Direct AWS API calls for resource inspection and validation |
| `aws-knowledge-mcp-server` | Remote (HTTP) | AWS architecture knowledge base and best practices |
| `aws-documentation-mcp-server` | Local (uvx) | AWS service documentation lookup |
| `aws-pricing-mcp-server` | Local (uvx) | AWS pricing queries and cost estimation |

> **Setup:** Update `AWS_PROFILE` and `AWS_REGION` in `.cursor/mcp.json` before first use.

### Skills

**Terraform Skill (v1.6.0)** -- Activates automatically when working with Terraform or OpenTofu. Provides guidance on:
- Module structure and patterns
- Testing frameworks (native tests, Terratest)
- CI/CD pipeline configuration
- Security scanning (Trivy, Checkov)
- State management and debugging

### Superpowers

The superpowers plugin is enabled and provides structured workflows:

| Workflow | When to Use |
|----------|-------------|
| Brainstorming | Before any creative work -- explores design before implementation |
| Writing Plans | Turn specs into bite-sized implementation plans |
| Executing Plans | Implement plans task-by-task with checkpoints |
| Systematic Debugging | Investigate bugs with evidence-based troubleshooting |
| Code Review | Validate work against plan and coding standards |
| Verification | Confirm work is complete before claiming done |
