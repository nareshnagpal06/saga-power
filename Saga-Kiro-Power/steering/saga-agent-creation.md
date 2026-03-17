# Sage Agent Creation Workflow

## Architecture Pattern

When building a Sage agent, coordinate between:

### AgentCore Layer (aws-agentcore Power)
- Use AgentCore Runtime for hosting the agent
- Configure AgentCore Gateway to expose Sage APIs as MCP tools
- Set up AgentCore Memory for conversation context
- Use AgentCore Identity for authentication

### Agent Development Layer (Strands Power)
- Use Strands Agent SDK to build the agent logic
- Define agent instructions and behavior
- Configure tool usage patterns
- Implement multi-agent orchestration if needed

### Integration Layer (Sage MCP Server)
- Connect to Sage APIs through MCP protocol
- Handle authentication and authorization
- Map Sage API operations to MCP tools
- Implement error handling and retries

## Step-by-Step Process

1. **Define Agent Requirements**
   - What Sage operations does the agent need?
   - What user interactions are required?
   - What data needs to be accessed?

2. **Configure AgentCore Gateway**
   - Create gateway to expose Sage APIs
   - Set up authentication (OAuth, API keys)
   - Define tool schemas

3. **Build Agent with Strands**
   - Create agent using Strands SDK
   - Connect to Sage MCP tools
   - Define agent instructions
   - Test agent behavior

4. **Deploy to AgentCore Runtime**
   - Package agent code
   - Deploy to AgentCore
   - Configure scaling and monitoring
   - Test production deployment

## Example Integration

```python
# Sage agent that uses both AgentCore and Strands
from strands_agents import Agent

# Agent connects to Sage APIs via AgentCore Gateway
sage_agent = Agent(
    name="sage-financial-assistant",
    model="anthropic.claude-3-5-sonnet-20241022-v2:0",
    instructions='''
    You help users interact with Sage financial systems.
    Use the available Sage API tools to:
    - Query financial data
    - Create invoices
    - Manage customers
    - Generate reports
    ''',
    tools=["sage_mcp_tools"]
)