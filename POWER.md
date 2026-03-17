---
name: "saga-power"
displayName: "Build Saga AI Agents"
description: "Create AI agents on Saga Platform that interact with Saga APIs, MCP, Tools using Amazon Bedrock AgentCore and Strands SDK"
keywords: ["saga, agents", "ai", "llm", "bedrock", "strands", "agentcore", "deploy", "memory", "gateway", "mcp", "tools", "anthropic", "openai", "gemini"]
author: "SagaKiroPower"
---

# Onboarding

## Step 1: Validate Prerequisites
Before building saga agents, ensure you have:
- AWS AgentCore Power installed
- Strands Power installed
- saga API credentials configured
- MCP server for saga APIs running
- Python 3.10+
- Docker running (required for AgentCore Runtime deployments)
- AWS credentials configured (`aws configure`)

### Required Python packages
```bash
pip install strands-agents strands-agents-tools bedrock-agentcore-starter-toolkit mcp[cli]
```

### Required CLI tools
```bash
# Install the AgentCore CLI (needed for deploy/invoke commands)
pip install bedrock-agentcore
```

### Verify installation
```bash
python -c "import strands; print('Strands SDK ✓')"
python -c "import bedrock_agentcore; print('AgentCore SDK ✓')"
agentcore --version  # Should print version
aws sts get-caller-identity  # Should show your AWS account
docker info  # Should show Docker is running
```

## Step 2: saga MCP Servers
Saga provides domain-specific MCP servers for partners to use in their agents:

### saga-finance-mcp (localhost:8000)
Accounting & invoicing tools:
- `list_customers` / `get_customer` - Customer management
- `list_invoices` / `get_invoice` / `create_invoice` - Invoice operations
- `get_account_balance` - Chart of accounts lookup
- `get_profit_and_loss` - P&L summary
- `get_aged_receivables` - Receivables breakdown

### saga-payroll-mcp (localhost:8001)
Payroll & HR tools:
- `list_employees` / `get_employee` - Employee directory
- `get_payslip` - Latest payslip for an employee
- `list_pay_runs` - Pay run history and status
- `get_leave_balance` - Annual and sick leave balances
- `get_payroll_summary` - Total payroll cost summary

Start both servers:
```bash
cd /path/to/saga-mcp
source .venv/bin/activate
python mcp_server.py &          # Finance on port 8000
python saga_payroll_mcp.py &    # Payroll on port 8001
```

# When to Load Steering Files

- Creating new saga agent → saga-agent-creation.md
- Integrating saga APIs → saga-api-integration.md

# Building saga Agents

When you ask to create a saga agent, this power will:

1. **Activate aws-agentcore Power** - Provides AgentCore infrastructure and deployment patterns
2. **Activate Strands Power** - Provides Strands Agent SDK for agent development
3. **Use saga MCP Server** - Connects to your saga APIs through the MCP protocol

## Agent Creation Workflow

### Using AgentCore and Strands Together

```python
# Example: saga agent using Strands SDK with AgentCore
from strands_agents import Agent

saga_agent = Agent(
    name="saga-assistant",
    model="anthropic.claude-3-5-sonnet-20241022-v2:0",
    tools=["saga_mcp_tools"],  # Your saga MCP server tools
    instructions="You are a helpful assistant that helps users interact with saga APIs"
)