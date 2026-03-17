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

## Step 2: saga API Configuration
Ensure your saga MCP server is properly configured with:
- API endpoint URL
- Authentication credentials
- Required permissions

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