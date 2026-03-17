# Shared Patterns — Strands + AgentCore End-to-End

## Pattern 1: Build → Test → Deploy (Minimal)

The simplest path from zero to a deployed agent.

```bash
# 1. Install everything
pip install strands-agents strands-agents-tools bedrock-agentcore-starter-toolkit

# 2. Set credentials
export AWS_BEDROCK_API_KEY=your_key

# 3. Create project
agentcore create --non-interactive --project-name MyAgent
cd MyAgent

# 4. Start dev server + test
agentcore dev                    # Terminal 1
agentcore invoke --dev "Hello"   # Terminal 2

# 5. Deploy
agentcore launch
agentcore invoke '{"prompt": "Hello from the cloud"}'
```

## Pattern 2: Agent with Memory + Custom Tools

A production-ready agent with conversation persistence and custom tools.

```python
import os
from datetime import datetime
from strands import Agent, tool
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.memory.integrations.strands.config import AgentCoreMemoryConfig
from bedrock_agentcore.memory.integrations.strands.session_manager import AgentCoreMemorySessionManager

@tool
def lookup_order(order_id: str) -> str:
    """Look up an order by ID.

    Args:
        order_id: The order identifier
    """
    # Replace with real logic
    return f"Order {order_id}: Shipped, arriving tomorrow"

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    # Memory
    config = AgentCoreMemoryConfig(
        memory_id=os.environ.get("AGENTCORE_MEMORY_ID"),
        session_id=payload.get("session_id", "default"),
        actor_id=payload.get("actor_id", "user")
    )
    session_manager = AgentCoreMemorySessionManager(
        agentcore_memory_config=config, region_name="us-west-2"
    )

    # Agent
    agent = Agent(
        tools=[lookup_order],
        system_prompt="You are a customer support assistant. Remember user context across messages.",
        session_manager=session_manager
    )
    response = agent(payload.get("prompt", "Hello"))
    return {"response": str(response), "session_id": config.session_id}

if __name__ == "__main__":
    app.run()
```

## Pattern 3: Agent with Gateway (External Tools via MCP)

An agent that accesses Lambda functions and APIs through AgentCore Gateway.

```python
from strands import Agent, tool
from bedrock_agentcore import BedrockAgentCoreApp
from mcp_client.client import get_streamable_http_mcp_client

@tool
def local_calculator(a: int, b: int) -> int:
    """Add two numbers.

    Args:
        a: First number
        b: Second number
    """
    return a + b

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    mcp_client = get_streamable_http_mcp_client()
    with mcp_client:
        gateway_tools = mcp_client.list_tools_sync()
        agent = Agent(
            tools=[local_calculator] + gateway_tools,
            system_prompt="You have access to local and remote tools."
        )
        response = agent(payload.get("prompt", "Hello"))
        return {"response": str(response)}

if __name__ == "__main__":
    app.run()
```

## Pattern 4: Full Stack — Memory + Gateway + Custom Tools

The complete pattern combining everything.

```python
import os
from datetime import datetime
from strands import Agent, tool
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.memory.integrations.strands.config import AgentCoreMemoryConfig
from bedrock_agentcore.memory.integrations.strands.session_manager import AgentCoreMemorySessionManager
from mcp_client.client import get_streamable_http_mcp_client

@tool
def get_time() -> str:
    """Get the current time."""
    return datetime.now().strftime("%H:%M:%S")

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    # Memory
    config = AgentCoreMemoryConfig(
        memory_id=os.environ.get("AGENTCORE_MEMORY_ID"),
        session_id=payload.get("session_id", "default"),
        actor_id=payload.get("actor_id", "user")
    )
    session_manager = AgentCoreMemorySessionManager(
        agentcore_memory_config=config, region_name="us-west-2"
    )

    # Gateway tools
    mcp_client = get_streamable_http_mcp_client()
    with mcp_client:
        gateway_tools = mcp_client.list_tools_sync()

        # Agent with everything
        agent = Agent(
            tools=[get_time] + gateway_tools,
            system_prompt="You are a full-featured assistant with memory, local tools, and remote tools.",
            session_manager=session_manager
        )
        response = agent(payload.get("prompt", "Hello"))
        return {"response": str(response), "session_id": config.session_id}

if __name__ == "__main__":
    app.run()
```

## Deployment Checklist

1. ✅ Agent works locally with `agentcore dev` + `agentcore invoke --dev`
2. ✅ Deploy with `mode: NO_MEMORY` first
3. ✅ Verify cloud deployment with `agentcore invoke`
4. ✅ Add memory (`STM_ONLY`) and redeploy
5. ✅ Add gateway targets if needed
6. ✅ Test full integration end-to-end
7. ✅ No secrets in version control
