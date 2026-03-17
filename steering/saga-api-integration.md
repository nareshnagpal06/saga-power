# Sage API Integration Guide

## Overview

This guide provides patterns and best practices for integrating Sage APIs into AI agents using AgentCore Gateway and the Sage MCP server.

## Architecture Components

### 1. Sage MCP Server
- Exposes Sage APIs as MCP tools
- Handles authentication and authorization
- Manages rate limiting and error handling
- Provides tool schemas for agent consumption

### 2. AgentCore Gateway
- Routes agent requests to Sage MCP server
- Manages authentication tokens
- Provides observability and logging
- Handles request/response transformation

### 3. Strands Agent
- Consumes Sage tools through MCP protocol
- Orchestrates multi-step workflows
- Manages conversation context
- Handles user interactions

---

## Integration Patterns

### Pattern 1: Direct API Access

**Use Case**: Simple, single-operation queries (e.g., "Get customer details")

**Flow**:
```
User → Strands Agent → AgentCore Gateway → Sage MCP Server → Sage API
```

**Example**:
```python
# Agent receives user query: "Show me customer ABC123"
# Agent calls Sage MCP tool directly
result = sage_agent.use_tool(
   tool_name="get_customer",
   parameters={"customer_id": "ABC123"}
)
```

**Best Practices**:
- Use for read-only operations
- Implement caching for frequently accessed data
- Handle API errors gracefully with user-friendly messages

---
[3:30 PM]### Pattern 2: Multi-Step Workflows

**Use Case**: Complex operations requiring multiple API calls (e.g., "Create invoice for customer")

**Flow**:
```
User → Strands Agent → [Multiple Sage API calls] → Aggregate Results → Response
```

**Example Workflow**: Creating an Invoice
```python
# Step 1: Validate customer exists
customer = sage_agent.use_tool("get_customer", {"customer_id": "ABC123"})

# Step 2: Retrieve product pricing
products = sage_agent.use_tool("get_products", {"product_ids": ["P001", "P002"]})

# Step 3: Calculate totals
total = sum([p["price"] * p["quantity"] for p in products])

# Step 4: Create invoice
invoice = sage_agent.use_tool("create_invoice", {
   "customer_id": "ABC123",
   "line_items": products,
   "total": total,
   "due_date": "2026-04-17"
})
```

**Best Practices**:
- Break complex operations into logical steps
- Validate data at each step before proceeding
- Use AgentCore Memory to store intermediate results
- Implement rollback mechanisms for failed transactions

---

### Pattern 3: Data Aggregation and Reporting

**Use Case**: Generating reports from multiple Sage data sources

**Flow**:
```
User → Strands Agent → [Parallel API calls] → Data Aggregation → Format Report → Response
```

**Example**: Financial Summary Report
```python
# Parallel data retrieval
import asyncio

async def get_financial_summary(period):
   # Fetch data in parallel
   invoices, payments, expenses = await asyncio.gather(
       sage_agent.use_tool_async("get_invoices", {"period": period}),
       sage_agent.use_tool_async("get_payments", {"period": period}),
       sage_agent.use_tool_async("get_expenses", {"period": period})
   )

   # Aggregate and format
   summary = {
       "total_invoiced": sum([i["amount"] for i in invoices]),
       "total_received": sum([p["amount"] for p in payments]),
       "total_expenses": sum([e["amount"] for e in expenses])
   }

   return summary
```

**Best Practices**:
- Use parallel API calls when operations are independent
- Implement timeout handling for slow API responses
- Cache aggregated data with appropriate TTL
- Format data for readability before presenting to users

---

## Authentication Patterns

### OAuth 2.0 Flow (Recommended)

**Setup in AgentCore Gateway**:
```yaml
authentication:
 type: oauth2
 provider: sage
 client_id: ${SAGE_CLIENT_ID}
 client_secret: ${SAGE_CLIENT_SECRET}
 token_endpoint: https://oauth.sage.com/token
 scopes:
   - read:customers
   - write:invoices
   - read:products
```

**Token Management**:
- AgentCore Gateway handles token refresh automatically
- Tokens are stored securely in AgentCore Identity
- Agent code doesn't need to manage authentication

### API Key Authentication (Alternative)

**Setup in MCP Server**:
```json
{
 "env": {
   "SAGE_API_KEY": "${SAGE_API_KEY}",
   "SAGE_API_URL": "https://api.sage.com/v1"
 }
}
```

**Best Practices**:
- Rotate API keys regularly
- Use environment variables, never hardcode keys
- Implement key expiration monitoring

---

## Error Handling Strategies

### 1. Rate Limiting
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
   stop=stop_after_attempt(3),
   wait=wait_exponential(multiplier=1, min=2, max=10)
)
def call_sage_api(tool_name, params):
   try:
       return sage_agent.use_tool(tool_name, params)
   except RateLimitError as e:
       # Log and retry with exponential backoff
       logger.warning(f"Rate limit hit: {e}")
       raise
```

### 2. API Errors
```python
def handle_sage_error(error):
   error_messages = {
       404: "The requested resource was not found in Sage",
       401: "Authentication failed. Please check your Sage credentials",
       403: "You don't have permission to access this Sage resource",
       500: "Sage API is experiencing issues. Please try again later"
   }

   return error_messages.get(error.status_code,
                            f"An unexpected error occurred: {error.message}")
```

### 3. Validation Errors
```python
def validate_customer_data(customer_data):
   required_fields = ["name", "email", "address"]
   missing = [f for f in required_fields if f not in customer_data]

   if missing:
       return {
           "valid": False,
           "message": f"Missing required fields: {', '.join(missing)}"
       }

   return {"valid": True}
```

---
[3:30 PM]## Data Transformation Patterns

### Sage API → Agent Context

**Problem**: Sage API responses may contain more data than needed

**Solution**: Transform and filter data before presenting to agent

```python
def transform_customer_data(sage_response):
   """Extract only relevant customer information"""
   return {
       "id": sage_response["customer_id"],
       "name": sage_response["display_name"],
       "email": sage_response["contact"]["email"],
       "balance": sage_response["account"]["current_balance"],
       "status": sage_response["account"]["status"]
   }
```

### User Input → Sage API Format

**Problem**: User queries may not match Sage API parameter formats

**Solution**: Normalize and validate user input

```python
def prepare_invoice_request(user_input):
   """Convert natural language to Sage API format"""
   return {
       "customer_id": extract_customer_id(user_input),
       "invoice_date": parse_date(user_input.get("date", "today")),
       "due_date": calculate_due_date(user_input.get("payment_terms", "net30")),
       "line_items": parse_line_items(user_input["items"]),
       "currency": user_input.get("currency", "USD")
   }
```

---

## Testing and Validation

### 1. Mock Sage API for Development

```python
# Use mock MCP server for testing
if os.getenv("ENVIRONMENT") == "development":
   sage_mcp_server = MockSageMCPServer()
else:
   sage_mcp_server = SageMCPServer()
```

### 2. Validate Tool Schemas

```python
def validate_tool_call(tool_name, parameters):
   """Ensure parameters match Sage API schema"""
   schema = sage_mcp_server.get_tool_schema(tool_name)

   # Validate required parameters
   required = schema.get("required", [])
   missing = [p for p in required if p not in parameters]

   if missing:
       raise ValueError(f"Missing required parameters: {missing}")

   return True
```

### 3. Integration Testing

```python
def test_create_invoice_workflow():
   """Test complete invoice creation workflow"""
   # Setup test data
   test_customer = create_test_customer()
   test_products = create_test_products()

   # Execute workflow
   result = sage_agent.execute_workflow(
       "create_invoice",
       customer_id=test_customer["id"],
       products=test_products
   )

   # Validate results
   assert result["status"] == "success"
   assert result["invoice_id"] is not None

   # Cleanup
   cleanup_test_data(test_customer, result["invoice_id"])
```

---

## Monitoring and Observability

### Key Metrics to Track

1. **API Call Success Rate**
  - Track successful vs failed Sage API calls
  - Monitor by endpoint and operation type

2. **Response Times**
  - Measure latency for each Sage API operation
  - Set alerts for slow responses (>2 seconds)

3. **Error Rates**
  - Track error types (auth, rate limit, validation)
  - Monitor error trends over time

4. **Token Usage**
  - Monitor agent token consumption
  - Track cost per conversation

### AgentCore Observability Integration

```python
# AgentCore automatically logs all API calls
# Access logs through AgentCore Console or CloudWatch

# Add custom metrics
from agentcore.observability import track_metric

track_metric("sage_api_call", {
   "endpoint": "create_invoice",
   "duration_ms": 245,
   "status": "success"
})
```

---

## Security Best Practices

### 1. Principle of Least Privilege
- Grant agents only the Sage API permissions they need
- Use separate API keys for different agent types
- Regularly audit agent permissions

### 2. Data Privacy
- Never log sensitive customer data (PII, financial info)
- Implement data masking for logs and monitoring
- Comply with GDPR, CCPA, and other regulations

### 3. Input Validation
- Sanitize all user inputs before passing to Sage APIs
- Prevent injection attacks through parameter validation
- Implement rate limiting on agent operations

### 4. Secure Credential Storage
- Store Sage API credentials in AWS Secrets Manager
- Rotate credentials regularly (every 90 days)
- Use IAM roles for AgentCore Gateway authentication

---

## Common Integration Scenarios

### Scenario 1: Customer Support Agent

**Capabilities**:
- Look up customer information
- View invoice history
- Check payment status
- Update customer contact details

**Example Agent Instructions**:
```
You are a Sage customer support assistant. Help users with:
- Finding customer records
- Checking invoice and payment status
- Updating customer information
- Answering questions about account balances

Always verify customer identity before sharing sensitive information.
Use the Sage API tools to retrieve accurate, real-time data.
```

### Scenario 2: Financial Reporting Agent

**Capabilities**:
- Generate financial reports
- Analyze revenue trends
- Track outstanding invoices
- Calculate key financial metrics

**Example Agent Instructions**:
```
You are a Sage financial analyst. Generate reports and insights on:
- Revenue and expense trends
- Outstanding receivables
- Cash flow analysis
- Period-over-period comparisons

Use Sage API data to provide accurate financial metrics.
Present data in clear, business-friendly formats.
```

### Scenario 3: Invoice Automation Agent

**Capabilities**:
- Create invoices automatically
- Send invoice reminders
- Process payments
- Update invoice status

**Example Agent Instructions**:
```
You automate invoice workflows in Sage. Your tasks include:
- Creating invoices from approved orders
- Sending payment reminders for overdue invoices
- Recording payments when received
- Updating invoice status

Always validate data before creating invoices.
Follow company policies for payment terms and pricing.
```

---

## Troubleshooting Guide

### Issue: Agent can't connect to Sage API

**Symptoms**: Connection timeout or authentication errors

**Solutions**:
1. Verify Sage MCP server is running: `ps aux | grep sage-mcp`
2. Check API credentials in environment variables
3. Validate network connectivity to Sage API endpoint
4. Review AgentCore Gateway logs for error details

### Issue: Slow API responses

**Symptoms**: Agent takes >5 seconds to respond

**Solutions**:
1. Implement caching for frequently accessed data
2. Use parallel API calls for independent operations
3. Optimize Sage API queries (use filters, pagination)
4. Consider using Sage webhooks for real-time updates

### Issue: Rate limiting errors

**Symptoms**: 429 Too Many Requests errors

**Solutions**:
1. Implement exponential backoff retry logic
2. Cache API responses with appropriate TTL
3. Batch API requests when possible
4. Contact Sage to increase rate limits if needed

---

## Next Steps

After integrating Sage APIs:

1. **Test thoroughly** - Use the testing patterns above to validate all workflows
2. **Monitor performance** - Set up dashboards in AgentCore Console
3. **Gather feedback** - Collect user feedback on agent accuracy and helpfulness
4. **Iterate and improve** - Refine agent instructions based on real usage patterns
5. **Scale gradually** - Start with limited users, expand as confidence grows

## Additional Resources

- Sage API Documentation: https://developer.sage.com/api/
- AgentCore Gateway Guide: [Reference aws-agentcore Power]
- Strands Agent SDK: [Reference strands Power]
- MCP Protocol Specification: https://modelcontextprotocol.io/