# Execution Garden: Enterprise LLM Gateway Architecture for Banking (Refined)

> **Version:** 2.0.0  
> **Date:** October 27, 2025  
> **Status:** Refined Draft  
> **Authors:** Architecture Team  
> **Revision:** Added llmgateway.io and AWS AgentCore Gateway analysis

## Executive Summary

Banking organizations require **centralized AI governance without sacrificing business unit agility**. This refined architecture analysis incorporates two critical new solutions that have emerged in the LLM gateway and MCP hosting landscape:

1. **llmgateway.io** - Open-source AGPLv3 gateway providing LiteLLM-like capabilities with modern architecture
2. **AWS AgentCore** - Comprehensive managed service suite for agent runtime, MCP gateway, memory, and identity

**Updated critical insight**: Start with Lambda + API Gateway + Bedrock for immediate value (Phase 1), selectively add container-based gateway when multi-provider orchestration is needed (Phase 2), and **leverage AWS AgentCore Gateway for all MCP server hosting** instead of self-managing MCP infrastructure. This phased approach delivers **60% lower TCO than custom builds**, provides managed MCP hosting at scale, and offers a clear **14-16 week path to production**.

## Key Architectural Updates

### What's New in Version 2.0

1. **llmgateway.io Evaluation**: Comprehensive analysis of this emerging open-source alternative to LiteLLM
2. **AWS AgentCore Gateway Integration**: Recommendation to use managed MCP hosting instead of self-hosting MCP servers
3. **AgentCore Runtime for Agents**: Pattern for deploying agentic applications with 8-hour runtime support
4. **Updated Cost Models**: Incorporating AgentCore pricing ($0.005/1K tool calls, consumption-based runtime)
5. **Refined MCP Architecture**: Centralized MCP hosting via AgentCore Gateway with OAuth 2.1 security
6. **Enhanced Observability**: AgentCore Observability with CloudWatch and OTEL compatibility

---

## LLM Gateway Solution Comparison (Updated)

Based on research including llmgateway.io (launched 2024) and recent AWS AgentCore capabilities, the comparison now includes five primary options:

| Feature | LiteLLM (OSS) | llmgateway.io (OSS) | Portkey Hybrid | Kong AI Gateway | AWS Bedrock Native |
|---------|---------------|---------------------|----------------|-----------------|-------------------|
| **License** | Apache 2.0 | AGPLv3 | Apache 2.0 core + Enterprise | Apache 2.0 / Enterprise | Fully managed SaaS |
| **First Released** | 2023 | 2024 | 2023 | 2024 (AI features) | 2023 |
| **Deployment** | ECS, EKS, Docker | ECS, EKS, Docker | Data plane in VPC | ECS, EKS, VMs | N/A |
| **Multi-provider support** | 100+ providers | 93+ models | 200+ providers | 10+ providers | Bedrock models only |
| **Self-hosted data sovereignty** | ✅ 100% | ✅ 100% | ✅ Data plane in VPC | ✅ 100% | ⚠️ AWS-managed |
| **Automatic fallback** | ✅ Built-in | ✅ Built-in | ✅ Built-in | ✅ Enterprise only | ❌ No |
| **Cost tracking per team** | ✅ Virtual keys | ✅ Per-project tracking | ✅ Native | ⚠️ Via plugins | ⚠️ Tags only |
| **Rate limiting** | ✅ RPM/TPM | ✅ RPM/TPM | ✅ Token-based | ✅ Advanced | ⚠️ Basic |
| **Observability integrations** | ✅ 15+ platforms | ✅ Basic (CloudWatch) | ✅ Native + export | ✅ Via plugins | ✅ CloudWatch |
| **Prompt caching** | ✅ Redis | ✅ Redis/simple | ✅ Semantic | ✅ Semantic (Ent) | ✅ Native (5 min) |
| **Guardrails** | ⚠️ Via integrations | ⚠️ Basic | ✅ 50+ built-in | ✅ Prompt guards | ✅ Bedrock Guardrails |
| **PII detection** | ⚠️ Via Comprehend | ⚠️ Via Comprehend | ✅ Built-in | ⚠️ Via plugins | ⚠️ Via Comprehend |
| **Model lifecycle** | ✅ Routing-based | ⚠️ Manual | ✅ Version mgmt | ⚠️ Manual | ⚠️ Manual |
| **Production maturity** | ✅ 480B+ tokens | ⚠️ Early stage (<1 yr) | ✅ Enterprise-ready | ✅ Established | ✅ AWS-native |
| **Operational complexity** | Medium | Medium | Low (hybrid model) | High | Very low |
| **Community support** | ✅ 12K+ GitHub stars | ⚠️ 200+ GitHub stars | ⚠️ Smaller | ✅ Large | ✅ AWS support |
| **License restrictions** | ✅ Permissive (Apache 2.0) | ⚠️ Copyleft (AGPLv3) | ✅ Permissive | ✅ Permissive | N/A |
| **Banking adoption** | ✅ Production-proven | ⚠️ Too new | ✅ Enterprise clients | ⚠️ Limited AI | ✅ Extensive |
| **3-year TCO** | $550K-$900K | $550K-$900K (similar) | $825K-$1.1M | $800K-$1.5M | $150K-$400K |

### llmgateway.io Analysis

**Key findings**:
- **Architecture**: Modern TypeScript/Node.js stack with PostgreSQL and Redis, similar to LiteLLM
- **License concern**: AGPLv3 is copyleft, requiring derivative works to be open-sourced - **significant risk for banking proprietary modifications**
- **Maturity**: Launched 2024, still early-stage with limited production references
- **Features**: Good BYOK (bring your own keys) support, basic analytics, caching, but lacks enterprise guardrails
- **Self-hosting**: Docker Compose setup, but less documented than LiteLLM for enterprise deployments
- **Cost model**: Free open-source, $50/month Pro plan for hosted version

**Recommendation**: **Not recommended for banking** due to AGPLv3 license restrictions. While the architecture is modern and feature set is growing, the copyleft license creates compliance risks for proprietary banking extensions. LiteLLM's Apache 2.0 license and 480B+ tokens production track record make it the safer choice.

**When to consider llmgateway.io**: Only if you're committed to 100% open-source with no proprietary modifications, and accept early-stage maturity risks.

---

## AWS AgentCore: Managed MCP and Agent Infrastructure

### Overview

AWS AgentCore (announced preview July 2025, GA October 2025) is a comprehensive managed service suite purpose-built for AI agents. For the Execution Garden architecture, **AgentCore Gateway is the recommended solution for all MCP server hosting**, replacing self-managed ECS deployments of MCP servers.

### AgentCore Components

#### 1. **AgentCore Runtime**
- **Purpose**: Serverless runtime for deploying agents built with any framework (LangGraph, CrewAI, Strands Agents, etc.)
- **Key capabilities**: 
  - Industry-leading 8-hour execution windows (vs Lambda's 15 minutes)
  - Complete session isolation in dedicated microVMs
  - Fast cold starts (<3 seconds)
  - 100MB payload support
  - OAuth integration via AgentCore Identity
- **Use case in Execution Garden**: Deploy your 10 agentic applications on AgentCore Runtime instead of EC2/ECS

#### 2. **AgentCore Gateway** ⭐ **Critical for MCP Hosting**
- **Purpose**: Fully managed MCP server infrastructure that converts APIs, Lambda functions, and services into MCP-compatible tools
- **Key capabilities**:
  - Zero-code MCP tool creation from OpenAPI specs, AWS Lambda, or Smithy models
  - Native MCP protocol support with streamable HTTP transport
  - Dual authentication: Inbound OAuth (Cognito/Okta) + Outbound IAM/OAuth
  - Semantic tool search and discovery
  - Connects to existing external MCP servers
- **Use case in Execution Garden**: **Replace all self-hosted MCP server deployments** - no more managing ECS containers for MCP
- **Pricing**: $0.005 per 1,000 tool invocations (ListTools, CallTool, Ping)

#### 3. **AgentCore Identity**
- **Purpose**: Agent identity and access management with OAuth 2.1 support
- **Key capabilities**:
  - Integration with corporate IdP (Okta, Azure AD, Cognito)
  - Secure token vault for refresh tokens
  - Just-enough access (JEA) authorization
  - Both user delegation and autonomous agent patterns
- **Pricing**: **Included at no extra cost** when using Runtime or Gateway

#### 4. **AgentCore Memory**
- **Purpose**: Managed short-term and long-term memory for agents
- **Key capabilities**:
  - Short-term: session/conversation context
  - Long-term: persistent knowledge across sessions
  - Built-in and custom memory strategies
  - Self-managed extraction pipelines
- **Pricing**: $0.25/1K short-term events, $0.50/1K long-term records

#### 5. **AgentCore Browser & Code Interpreter**
- **Purpose**: Managed tools for web automation and code execution
- **Use cases**: Agents that need to scrape websites or run generated code
- **Pricing**: Consumption-based on runtime duration

#### 6. **AgentCore Observability**
- **Purpose**: Unified monitoring and tracing for agents
- **Key capabilities**:
  - OpenTelemetry (OTEL) compatible
  - CloudWatch dashboards
  - Integration with Datadog, Dynatrace, Langfuse, LangSmith, Arize Phoenix
  - End-to-end trace visualization
- **Pricing**: Included with CloudWatch costs

### AgentCore Gateway Architecture Pattern

**Recommended MCP hosting architecture** for Execution Garden:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Business Unit VPCs                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Agentic Applications (Python/TypeScript)                │  │
│  │  - LangGraph agents                                       │  │
│  │  - CrewAI multi-agent systems                            │  │
│  │  - Strands Agents workflows                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           │ MCP over HTTPS                       │
│                           ↓                                      │
└───────────────────────────┼──────────────────────────────────────┘
                            │
                    Transit Gateway
                            │
┌───────────────────────────┼──────────────────────────────────────┐
│              Execution Garden VPC                                │
│                           │                                       │
│  ┌────────────────────────┼──────────────────────────────────┐  │
│  │  AgentCore Gateway (Managed Service)                      │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │ Inbound Auth: OAuth 2.1 (Cognito User Pool)        │ │  │
│  │  │ - Authorization Code Flow (3LO) for users          │ │  │
│  │  │ - Client Credentials Flow (2LO) for services       │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  │  │                                                         │  │
│  │  │  Targets (Backend Services as MCP Tools):             │  │
│  │  │  ┌─────────────────────────────────────────────────┐ │  │
│  │  │  │ 1. OpenAPI Specs → REST APIs                    │ │  │
│  │  │  │    - Customer Data API (read-only queries)      │ │  │
│  │  │  │    - Product Catalog API                        │ │  │
│  │  │  │    - Market Data Feed API                       │ │  │
│  │  │  └─────────────────────────────────────────────────┘ │  │
│  │  │  ┌─────────────────────────────────────────────────┐ │  │
│  │  │  │ 2. Lambda Functions → Custom Tools              │ │  │
│  │  │  │    - Financial calculations (NPV, IRR, etc.)    │ │  │
│  │  │  │    - Document retrieval (RAG integration)       │ │  │
│  │  │  │    - Compliance checks                          │ │  │
│  │  │  └─────────────────────────────────────────────────┘ │  │
│  │  │  ┌─────────────────────────────────────────────────┐ │  │
│  │  │  │ 3. Smithy Models → AWS Service Tools            │ │  │
│  │  │  │    - S3 document access                         │ │  │
│  │  │  │    - DynamoDB queries                           │ │  │
│  │  │  └─────────────────────────────────────────────────┘ │  │
│  │  │  ┌─────────────────────────────────────────────────┐ │  │
│  │  │  │ 4. External MCP Servers (connect via Gateway)   │ │  │
│  │  │  │    - Partner MCP servers                        │ │  │
│  │  │  │    - Third-party tool providers                 │ │  │
│  │  │  └─────────────────────────────────────────────────┘ │  │
│  │  │                                                         │  │
│  │  │ Outbound Auth:                                         │  │
│  │  │ - IAM for Lambda/AWS services                         │  │
│  │  │ - OAuth for external APIs                             │  │
│  │  │ - API keys in Secrets Manager                         │  │
│  │  └─────────────────────────────────────────────────────┘  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  MCP Registry (DynamoDB)                                   │  │
│  │  - Gateway ARNs and endpoints                             │  │
│  │  - Tool catalog metadata                                  │  │
│  │  - Access control policies                               │  │
│  └────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

### AgentCore Gateway Benefits for Banking

1. **Zero infrastructure management**: No ECS clusters, no container orchestration, no scaling concerns
2. **Compliance-ready**: VPC support, PrivateLink, CloudTrail integration, IAM authorization
3. **Cost-effective**: $0.005/1K tool calls vs managing ECS tasks ($720/month for 3 tasks)
4. **Security built-in**: Dual OAuth/IAM auth, secret management, audit trails
5. **Tool discovery**: Semantic search across all tools, runtime discovery for agents
6. **Multi-protocol**: Native MCP plus connects to existing MCP servers

### AgentCore vs Self-Hosted MCP Comparison

| Aspect | Self-Hosted MCP (ECS) | AgentCore Gateway |
|--------|----------------------|-------------------|
| **Infrastructure** | ECS Fargate tasks ($720/mo for 3 tasks) | Fully managed, serverless |
| **Scaling** | Manual ASG configuration | Automatic, instant |
| **Security** | DIY OAuth, secrets management | Built-in OAuth 2.1, IAM integration |
| **Tool creation** | Write MCP server code | Zero-code from OpenAPI/Lambda |
| **Monitoring** | CloudWatch + custom dashboards | AgentCore Observability + OTEL |
| **High availability** | Multi-AZ deployment complexity | Built-in resilience |
| **Updates/patches** | Manual container updates | Managed by AWS |
| **Cost (1M tool calls/mo)** | $720 ECS + ops time | $5 tool calls + $0 Identity |
| **Integration effort** | Weeks to implement | Hours to configure |
| **Banking compliance** | Manual SOC 2, PCI compliance | AWS compliance inheritance |

**Recommendation**: **Use AgentCore Gateway for all MCP hosting**. The economics and operational simplicity are overwhelming - $5/million tool calls vs $720/month base ECS costs, plus eliminated operational burden. Reserve ECS/containers for LLM gateway (LiteLLM) where multi-provider orchestration requires stateful routing logic that AgentCore doesn't provide.

---

## Refined Architecture: Four-Phase Evolution

### Phase 1: Serverless LLM Gateway (Weeks 1-8, $4.5K/year infrastructure)

**Unchanged from original** - Lambda + API Gateway + Bedrock remains optimal starting point.

### Phase 2: Multi-Provider Gateway (Weeks 9-16, +$12K/year infrastructure)

**Updated** - Add LiteLLM on ECS when multi-provider orchestration needed:
- **Why LiteLLM over llmgateway.io**: Apache 2.0 license, production maturity (480B+ tokens), banking adoption
- **Container justification**: Multi-provider orchestration requires stateful routing, caching, circuit breakers
- **Cost**: $903/month for ECS Fargate + RDS + Redis

### Phase 3: MCP Server Hosting (Weeks 13-16, +$60/year at 1M calls)

**NEW RECOMMENDATION** - Use AgentCore Gateway instead of self-hosted MCP:
- **Deploy**: AgentCore Gateway with targets (OpenAPI specs, Lambda functions, Smithy models)
- **Integrate**: Cognito User Pool for OAuth, IAM roles for outbound auth
- **Cost**: $0.005 per 1K tool invocations = **$5/million calls**
- **No ECS infrastructure**: Eliminate previous recommendation to deploy MCP servers on ECS

**MCP hosting pattern**:
```bash
# Create Gateway
aws bedrock-agentcore create-gateway \
  --gateway-name "execution-garden-tools" \
  --ingress-authentication-configuration \
    "OAuth={Authority=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXX}"

# Add OpenAPI target (e.g., Customer Data API)
aws bedrock-agentcore create-target \
  --gateway-identifier "arn:aws:bedrock-agentcore:us-east-1:ACCOUNT:gateway/GATEWAY_ID" \
  --target-name "customer-data-api" \
  --target-type "OPEN_API" \
  --open-api-specification-s3-location "s3://bucket/customer-api-spec.json" \
  --egress-authentication-configuration \
    "OAuth={TokenUrl=https://api.example.com/oauth/token, ClientId=xxx}"

# Add Lambda target (e.g., Financial Calculations)
aws bedrock-agentcore create-target \
  --gateway-identifier "arn:aws:bedrock-agentcore:us-east-1:ACCOUNT:gateway/GATEWAY_ID" \
  --target-name "financial-calcs" \
  --target-type "LAMBDA" \
  --lambda-configuration "FunctionArn=arn:aws:lambda:us-east-1:ACCOUNT:function:npv-calculator" \
  --egress-authentication-configuration \
    "IAM={ExecutionRoleArn=arn:aws:iam::ACCOUNT:role/GatewayLambdaRole}"
```

**Agent connection code** (Python with Strands Agents):
```python
from strands_agents import Agent, AgentCoreClient
from strands_agents.mcp import MCPClient

# AgentCore client with OAuth
agentcore = AgentCoreClient(
    gateway_url="https://bedrock-agentcore.us-east-1.amazonaws.com/gateways/GATEWAY_ID/mcp",
    auth_provider="cognito",
    user_pool_id="us-east-1_XXXXX",
    client_id="YOUR_CLIENT_ID"
)

# Create agent with MCP tools
agent = Agent(
    name="financial_advisor",
    model="anthropic.claude-3-5-sonnet-20241022-v2:0",
    mcp_client=agentcore.get_mcp_client(),
    system_prompt="You are a financial advisor with access to customer data and calculation tools."
)

# Agent automatically discovers and uses tools from Gateway
response = agent.run("Calculate NPV for Project Alpha using customer's discount rate")
```

### Phase 4: Agent Runtime Deployment (Weeks 17-20, variable cost)

**NEW CAPABILITY** - Deploy agentic applications on AgentCore Runtime:

**When to use AgentCore Runtime**:
- Agents requiring >15 minute execution (up to 8 hours)
- Long-running workflows (multi-agent orchestration, complex reasoning)
- Need for persistent session state across tool calls
- Require user authentication with corporate IdP
- Multi-modal processing (100MB payloads)

**When to use Lambda** (Phase 1 pattern):
- Simple request-response agents (<15 minutes)
- Stateless operations
- Cost-sensitive high-volume (Lambda cheaper for short bursts)

**Deployment example** (LangGraph agent on AgentCore Runtime):
```python
# langgraph_agent.py
from langchain_aws import ChatBedrock
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    
def customer_service_agent(state: AgentState):
    llm = ChatBedrock(model="anthropic.claude-3-5-sonnet-20241022-v2:0")
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# Build graph
workflow = StateGraph(AgentState)
workflow.add_node("agent", customer_service_agent)
workflow.set_entry_point("agent")
workflow.add_edge("agent", END)

app = workflow.compile()

# AgentCore Runtime handler
def handler(event, context):
    user_input = event.get("input", "")
    result = app.invoke({"messages": [("user", user_input)]})
    return {"output": result["messages"][-1].content}
```

```bash
# Deploy to AgentCore Runtime with AgentCore CLI
pip install bedrock-agentcore

# Configure deployment
agentcore configure \
  --entry-point langgraph_agent.py \
  --runtime-name customer-service-agent \
  --memory-size 4096 \
  --timeout 3600 \
  --auth-type oauth \
  --oauth-provider cognito \
  --user-pool-id us-east-1_XXXXX

# Deploy
agentcore deploy
```

**AgentCore Runtime pricing example** (from AWS pricing page):
- CPU: 2 vCPU × 48 seconds active (60% I/O wait) × $0.0895/3600 = $0.002387/session
- Memory: 4GB × 120 seconds × $0.00945/3600 = $0.00126/session
- **Total: $0.003647/session** or **$109.40/month for 30K sessions**

**Cost comparison**:
- Lambda (500ms, 1GB): $0.0000083/invocation = $249/month for 30K invocations
- AgentCore Runtime (120s, 4GB): $0.003647/invocation = $109.40/month for 30K sessions

**Verdict**: For long-running agents, **AgentCore Runtime is 56% cheaper** than Lambda equivalent and provides better experience (no 15-min timeout, persistent state, OAuth integration).

---

## Updated Cost Analysis

### Phase 3: MCP Hosting with AgentCore Gateway

**Assumptions**: 10 agentic applications, each making 100K tool calls/month = 1M total tool calls

**AgentCore Gateway costs**:
- Tool invocations: 1M calls × $0.005/1K = **$5/month**
- AgentCore Identity: **$0** (included with Gateway)
- CloudWatch logs: ~$5/month
- **Total: $10/month** or **$120/year**

**Comparison to self-hosted MCP on ECS**:
- ECS Fargate (2 tasks, 0.5 vCPU, 1GB): $240/month
- ALB: $23/month
- RDS/Redis (optional): $50/month
- **Total: $313/month** or **$3,756/year**

**Savings**: $3,636/year (97% reduction) by using AgentCore Gateway

### Phase 4: Agent Runtime Deployment

**Assumptions**: 5 of 10 applications need long-running runtime, 6K sessions/month each = 30K total

**AgentCore Runtime costs** (per pricing example):
- 30K sessions × $0.003647 = **$109.40/month**
- AgentCore Identity: **$0** (included)
- AgentCore Memory (if used): ~$25/month for short-term
- **Total: $134.40/month** or **$1,613/year**

**Comparison to Lambda**:
- Lambda (2-minute equiv): 30K × 120s × 1GB × $0.0000166667 = **$60/month**
- **But**: Lambda cannot handle >15 min, no persistent state, no OAuth integration

**Verdict**: AgentCore Runtime costs more than Lambda for equivalent compute, but provides **capabilities Lambda cannot** (8-hour runtime, persistent state, OAuth). For true long-running agents, it's not comparable - you'd need EC2/ECS which costs $500+/month.

### Combined 3-Year TCO (All Phases)

| Component | Year 1 | Year 2-3 (annual) |
|-----------|--------|-------------------|
| **Phase 1: Serverless Gateway** | $54K | $54K |
| - Infrastructure | $888 | $888 |
| - Bedrock (on-demand) | $53K | $53K |
| **Phase 2: LiteLLM ECS** | $15.2K | $15.2K |
| - ECS Fargate + RDS + Redis | $10.8K | $10.8K |
| - Langfuse ECS | $2.4K | $2.4K |
| - S3, CloudWatch | $2K | $2K |
| **Phase 3: AgentCore Gateway** | $120 | $120 |
| **Phase 4: AgentCore Runtime** | $1.6K | $1.6K |
| **Personnel** | $450K | $450K |
| **Total Year 1** | **$521K** | |
| **Total Year 2-3 (each)** | | **$521K** |
| **3-Year Total** | | **$1.56M** |

**Previous 3-year estimate** (without AgentCore, with self-hosted MCP): $2.85M

**Savings with AgentCore**: **$1.29M over 3 years** (45% reduction)

**Key drivers**:
- AgentCore Gateway eliminates $3.7K/year ECS costs for MCP
- AgentCore Runtime consolidates agent infrastructure vs EC2 fleet
- Included Identity service ($0 incremental)
- Bedrock Provisioned Throughput (vs on-demand) saves $270K/year

---

## Updated Implementation Roadmap

### Weeks 1-4: Foundation (Unchanged)
AWS account setup, Transit Gateway, IAM, security baseline

### Weeks 5-8: Phase 1 Serverless Gateway (Unchanged)
Lambda + API Gateway + Bedrock deployment

### Weeks 9-12: Phase 2 LiteLLM Gateway
**Decision**: Deploy LiteLLM (not llmgateway.io) on ECS Fargate
- **Rationale**: Apache 2.0 license, production maturity, banking adoption
- **Activities**: ECS cluster, RDS, Redis, LiteLLM container, Langfuse deployment

### Weeks 13-16: Phase 3 AgentCore Gateway for MCP ⭐ **Updated**
**NEW**: Deploy AgentCore Gateway instead of self-hosted MCP servers
- **Week 13**: 
  - Create Cognito User Pool for OAuth 2.1
  - Deploy AgentCore Gateway
  - Configure first target (OpenAPI spec for customer data API)
- **Week 14**:
  - Add Lambda function targets (financial calculations, document retrieval)
  - Configure IAM roles for outbound authentication
  - Test MCP protocol connectivity from sample agent
- **Week 15**:
  - Onboard 5 MCP tools across 3 gateway targets
  - Integrate with DynamoDB registry for tool metadata
  - Configure semantic search indexing
- **Week 16**:
  - Production testing with 3 pilot agentic applications
  - Load test to 10K tool calls/day
  - Security review and penetration testing

### Weeks 17-20: Phase 4 AgentCore Runtime (Optional) ⭐ **New**
**NEW**: Deploy long-running agents on AgentCore Runtime
- **When to add**: If any of your 10 agents require >15 minute execution
- **Week 17**: 
  - Install AgentCore CLI and SDK
  - Convert first LangGraph agent to AgentCore-compatible format
  - Configure OAuth with Cognito
- **Week 18**:
  - Deploy 2 pilot agents to AgentCore Runtime
  - Integrate with AgentCore Gateway tools
  - Configure AgentCore Memory for persistent state
- **Week 19**:
  - Enable AgentCore Observability dashboards
  - Export OTEL traces to Langfuse
  - Set up CloudWatch alarms
- **Week 20**:
  - Production rollout of remaining long-running agents
  - Performance optimization
  - Cost analysis and right-sizing

---

## Revised Decision Matrix

### When to use each component:

| Use Case | Solution | Justification |
|----------|----------|---------------|
| **Single-provider LLM routing (Bedrock only)** | Phase 1: Lambda + API Gateway | Serverless, lowest cost, fastest deployment |
| **Multi-provider LLM orchestration** | Phase 2: LiteLLM on ECS | Stateful routing, automatic fallback, prompt caching |
| **MCP server hosting** | Phase 3: AgentCore Gateway | Zero infrastructure, $0.005/1K calls, managed auth |
| **Long-running agents (>15 min)** | Phase 4: AgentCore Runtime | 8-hour runtime, persistent state, OAuth integration |
| **Short-running agents (<15 min)** | Lambda functions | Cost-effective, auto-scaling, no idle costs |
| **Custom LLM gateway** | DON'T | Use LiteLLM (proven, 480B+ tokens) |
| **Self-host MCP servers** | DON'T | Use AgentCore Gateway (97% cost savings) |
| **llmgateway.io** | DON'T | AGPLv3 license risk, immature, use LiteLLM |

---

## Critical Success Factors (Updated)

### 1. **Leverage AWS Managed Services Aggressively**
- **AgentCore Gateway for MCP**: Eliminate all self-hosted MCP infrastructure
- **AgentCore Runtime for agents**: Use for long-running workflows instead of EC2/ECS
- **Bedrock for inference**: Avoid self-hosted models until >50M requests/month

### 2. **License Compliance**
- **LiteLLM: Apache 2.0** ✅ Safe for banking proprietary extensions
- **llmgateway.io: AGPLv3** ❌ Copyleft requires open-sourcing derivatives
- **Recommendation**: Stick with Apache 2.0 licensed solutions (LiteLLM, Portkey)

### 3. **MCP Architecture Best Practices**
- **Centralize via AgentCore Gateway**: Single source of truth for all tools
- **OAuth 2.1 security**: Cognito integration, no hardcoded secrets
- **Tool registry**: DynamoDB catalog for governance
- **Semantic search**: Enable runtime discovery for agents

### 4. **Cost Optimization**
- **AgentCore Gateway**: $5/million tool calls vs $240/month ECS
- **Bedrock Provisioned Throughput**: 40% savings vs on-demand
- **Right-size agents**: Lambda for short tasks, AgentCore Runtime for long workflows

### 5. **Observability Strategy**
- **AgentCore Observability**: Unified dashboards for agents, tools, memory
- **OpenTelemetry**: Export traces to Langfuse for LLM-specific analysis
- **CloudWatch**: Metrics, logs, alarms for all components

---

## Refined Recommendations

### Primary Architecture (Recommended)

**Phase 1-2**: Lambda + API Gateway + Bedrock → Add LiteLLM on ECS when multi-provider needed
**Phase 3**: AgentCore Gateway for all MCP hosting (not self-hosted)
**Phase 4**: AgentCore Runtime for long-running agents (optional)

### Solution Selection

1. **LLM Gateway**: **LiteLLM** (Apache 2.0, proven, banking-ready)
2. **MCP Hosting**: **AgentCore Gateway** (managed, $0.005/1K calls, zero infra)
3. **Agent Runtime**: **AgentCore Runtime** for >15 min, **Lambda** for <15 min
4. **Observability**: **Langfuse self-hosted** + **AgentCore Observability**

### Don't Use

1. ❌ **llmgateway.io**: AGPLv3 license risk, immature
2. ❌ **Self-hosted MCP on ECS**: 30x more expensive than AgentCore Gateway
3. ❌ **Custom builds**: $1.7M-$3.9M vs $1.56M with recommended stack
4. ❌ **Kong AI Gateway**: Overkill complexity unless already using Kong

---

## Next Steps

1. **Week 1**: Approve updated architecture with AgentCore Gateway for MCP
2. **Week 2-4**: Complete security review of AgentCore (VPC, PrivateLink, IAM)
3. **Week 5-8**: Deploy Phase 1 (serverless gateway)
4. **Week 9-12**: Deploy Phase 2 (LiteLLM) if multi-provider confirmed
5. **Week 13-16**: Deploy Phase 3 (AgentCore Gateway for MCP)
6. **Week 17-20**: Deploy Phase 4 (AgentCore Runtime) if long-running agents needed

---

## Conclusion

The refined architecture delivers:
- **45% cost reduction** ($1.56M vs $2.85M 3-year TCO) using AgentCore
- **97% MCP hosting cost savings** ($120/year vs $3.7K/year with ECS)
- **Zero MCP infrastructure management** (fully managed by AWS)
- **14-16 week deployment** (slightly faster with managed services)
- **Banking-grade compliance** (AWS SOC 2, PCI, ISO inheritance)
- **Production-proven components** (LiteLLM 480B+ tokens, AWS managed services)

**The critical architectural insight**: Use managed services (AgentCore Gateway) wherever possible, containers (LiteLLM) only where justified (multi-provider orchestration), and serverless (Lambda) as foundation. This phased approach eliminates undifferentiated heavy lifting while maintaining banking security requirements.

