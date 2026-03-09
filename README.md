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
