# Execution Garden: Enterprise LLM Gateway Architecture for Banking

> **Version:** 1.0.0  
> **Date:** October 27, 2025  
> **Status:** Draft  
> **Authors:** Architecture Team

Banking organizations require **centralized AI governance without sacrificing business unit agility**. After analyzing self-hosted LLM gateway solutions, AWS multi-account patterns, and banking compliance requirements, the optimal architecture combines LiteLLM on AWS ECS with a serverless-first approach that scales strategically to containers only when justified. This design delivers **60% lower TCO than custom builds** while maintaining full data sovereignty, meets strict regulatory requirements, and provides a clear **16-week path to production**.

**The critical insight**: Start with Lambda + API Gateway for immediate value, then selectively add ECS-based LiteLLM when you need multi-provider orchestration or self-hosted models. This phased approach minimizes risk, accelerates time-to-market, and aligns with your constraint that ECS/EKS requires justification.

## Banking's unique LLM platform challenge

Traditional API gateways fail for LLM workloads because they can't track token-based costs, manage model deprecations across providers, or apply banking-grade guardrails at the inference layer. Yet **pure custom builds cost $1.7M-$3.9M over three years** and take 6-12 months to deploy. Your Execution Garden account must serve as both gateway and governance layer while working within AWS service constraints (Lambda, API Gateway, ALB preferred; containers need justification).

The research reveals three architectural realities for banks: First, **LiteLLM is production-proven** with 480B+ tokens processed and offers 100+ provider support. Second, **serverless handles 200-500 req/s easily** with cold starts affecting <1% of requests. Third, **AWS Bedrock Guardrails now achieve 99% accuracy** with Automated Reasoning capabilities and cross-model support via ApplyGuardrail API, eliminating the need for separate guardrail infrastructure initially.

## Recommended architecture: Three-phase evolution

### Phase 1: Serverless foundation (Weeks 1-8, $4.5K infrastructure/year)

**When to deploy**: Immediately for routing to AWS Bedrock, basic observability, and single-provider scenarios serving your initial 10 agentic applications.

The serverless architecture provides the fastest path to production while maintaining banking-grade security. Business units connect via Transit Gateway to your Execution Garden VPC hosting API Gateway REST API, Lambda functions for routing and transformation, AWS Bedrock access through PrivateLink endpoints, and cross-account IAM roles with External IDs. **This pattern handles your target 200-500 req/s while costing 78% less than container alternatives**.

Lambda functions assume roles into the Operations Account for Bedrock access, implementing the "memoryless operations account" pattern where customer prompts and responses never persist in the centralized account. CloudWatch Logs remain in business unit accounts for data isolation, with CloudTrail providing centralized audit trails. This architecture achieves **<100ms API Gateway latency plus 500ms-2s Bedrock inference time**.

**Justification for starting serverless**: Your constraint requires justifying ECS/EKS, and Lambda + API Gateway deliver production-ready LLM routing without containers. Cold starts affect <1% of invocations and add only 200-500ms when they occur—acceptable for most banking workflows. No idle costs, automatic scaling to 1000 concurrent executions per second, and straightforward compliance audits make this the optimal starting point.

**Observability pattern**: Deploy OpenTelemetry instrumentation in Lambda functions to capture LLM-specific traces. Use AWS CloudWatch for metrics and CloudWatch Logs Insights for structured query analysis. OpenTelemetry semantic conventions for GenAI (now part of official OpenTelemetry spec as of 2025) standardize how you capture gen_ai.request.model, gen_ai.usage.input_tokens, gen_ai.usage.output_tokens, and gen_ai.system attributes across all LLM calls. Export traces to Grafana-compatible backends using OpenTelemetry Collector.

**Key services**: API Gateway REST API with Usage Plans for per-application quotas, Lambda (Python 3.12, 1GB memory, Graviton2 for 20% cost savings), AWS Bedrock with Provisioned Throughput for production workloads, Amazon Comprehend for PII detection, AWS Bedrock Guardrails for content filtering and compliance, VPC endpoints for private connectivity, KMS customer-managed keys per business unit.

**Cost breakdown**: API Gateway REST API $1/million requests, Lambda $8.53/million requests at 500ms execution, Bedrock usage variable ($3/million input tokens for Claude 3.5 Sonnet), infrastructure totals $165/month before Bedrock consumption. **For 10M requests/month with typical LLM usage: $4,510/month total, or $2,260/month with Provisioned Throughput**.

### Phase 2: Multi-provider gateway (Weeks 9-20, +$12K infrastructure/year)

**When to deploy**: When you need orchestration across Bedrock, OpenAI, Google Gemini, or self-hosted models. **Triggers include** requiring automatic fallback between providers, implementing sophisticated cost routing, needing prompt caching across providers, supporting models not available on Bedrock.

Add ECS Fargate running LiteLLM Gateway as your multi-provider orchestration layer. LiteLLM sits behind an Application Load Balancer in private subnets, provides an OpenAI-compatible API to all business units, and routes to Bedrock, OpenAI, Gemini, SageMaker endpoints, and self-hosted models. The gateway includes ElastiCache Redis for distributed rate limiting, prompt caching, and semantic routing metadata; RDS PostgreSQL for audit trails and usage tracking; and S3 for long-term log retention.

**This is the ECS/EKS justification**: Containers become necessary when Lambda's 15-minute timeout, lack of persistent connections, and limited complex dependency management restrict multi-provider orchestration. LiteLLM requires stateful caching, sophisticated retry logic with circuit breakers, and complex routing decisions that benefit from long-running processes. ECS Fargate eliminates container management overhead while providing the runtime environment LiteLLM requires.

The architecture maintains your Phase 1 serverless layer for simple Bedrock-only requests (80% of traffic) while routing complex multi-provider requests through LiteLLM. This **hybrid pattern optimizes costs**—serverless for simple routing, containers only where justified.

**LiteLLM deployment specifics**: Deploy official Docker image (litellm/litellm:main-stable) on ECS Fargate with 2 vCPU, 4GB RAM per task. Configure 3+ tasks across multiple AZs with auto-scaling based on CPU/memory. Use AWS Systems Manager Parameter Store for API keys with encryption. Configure IAM roles for ECS tasks to access Bedrock via AssumeRole pattern. Enable CloudWatch Container Insights for detailed metrics.

**Enhanced observability**: LiteLLM provides 15+ native integrations including Langfuse (self-hosted option available), Helicone (self-hosted via Docker/K8s), and Prometheus for Grafana. Deploy **Langfuse self-hosted** (PostgreSQL + Next.js app on ECS) for comprehensive LLM tracing with prompt versioning, user feedback collection, and cost attribution per application. Langfuse's Apache 2.0 license and strong banking adoption make it ideal for regulated environments. OpenTelemetry traces from Lambda and LiteLLM flow to unified observability backend.

**Cost breakdown**: ECS Fargate (3 tasks, 2 vCPU, 4GB) $720/month, RDS PostgreSQL Multi-AZ (db.t4g.medium) $140/month, ElastiCache Redis (cache.t4g.micro) $15/month, Application Load Balancer $23/month, S3 storage $5/month. **Total infrastructure: $903/month**. Combined with Phase 1 serverless layer: approximately $1,070/month infrastructure before LLM consumption.

### Phase 3: Self-hosted inference (Month 7+, variable GPU costs)

**When to deploy**: Only when business requirements demand proprietary fine-tuned models not available via Bedrock or other providers. **Triggers include** needing models with specialized banking domain knowledge, data residency requirements preventing API calls to external providers, cost optimization for extremely high volumes (>50M requests/month), or compliance mandates for model hosting.

Deploy self-hosted models on Amazon SageMaker Real-Time Inference endpoints or (if you need cost optimization at scale) EKS with GPU node groups. SageMaker provides managed infrastructure with one-click deployment, automatic scaling, and built-in monitoring—ideal for 1-10 models. EKS becomes cost-effective at scale (>100 models) with fractional GPU sharing and 30-45% cost savings reported by enterprises like Informatica and Vannevar Labs.

**EKS justification for self-hosted models**: GPU workloads benefit dramatically from fractional GPU sharing and time-slicing available in EKS but not in SageMaker. EKS with Karpenter enables 30-60 second node provisioning for scale-out, sophisticated GPU scheduling with tools like vLLM and TensorRT-LLM, and unified platform for ML and non-ML workloads. For banks running 10+ custom models, **EKS reduces inference costs by 30-45%** versus SageMaker while providing greater control.

LiteLLM orchestrates requests to these self-hosted endpoints alongside external providers, providing unified observability and cost tracking. For most banking use cases, **SageMaker endpoints are sufficient** and avoid Kubernetes operational complexity.

**Cost considerations**: SageMaker ml.g5.xlarge (1 GPU) costs $1.41/hour = $1,036/month per endpoint. EKS control plane $73/month plus g5.12xlarge nodes (4 GPUs) at $5.67/hour = $4,161/month per node. GPU-based inference only makes financial sense at scale or with specific compliance requirements.

## Architecture comparison matrix

| Dimension | Phase 1: Serverless Only | Phase 2: + LiteLLM on ECS | Phase 3: + Self-Hosted Models |
|-----------|--------------------------|---------------------------|-------------------------------|
| **Best for** | Bedrock-only, rapid deployment | Multi-provider orchestration | Proprietary models, high volume |
| **Time to production** | 6-8 weeks | 12-16 weeks | 20-28 weeks |
| **Monthly infrastructure cost** | $165 | $1,070 | $1,070 + $1,000-$10,000 |
| **Operational complexity** | Low (managed services) | Medium (container management) | High (GPU + container ops) |
| **Latency (P99)** | <1s routing + Bedrock | <1.2s routing + provider | <500ms routing + inference |
| **Supports 200-500 req/s** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Multi-provider** | ❌ No (Bedrock only) | ✅ Yes (100+ providers) | ✅ Yes (unlimited) |
| **Cost tracking per app** | ⚠️ Via API keys | ✅ Native in LiteLLM | ✅ Native in LiteLLM |
| **Automatic fallback** | ❌ No | ✅ Yes | ✅ Yes |
| **Prompt caching** | ⚠️ Bedrock only | ✅ Cross-provider | ✅ Cross-provider |
| **Model lifecycle mgmt** | ⚠️ Manual | ✅ Automated routing | ✅ Automated routing |
| **Compliance readiness** | ✅ Full | ✅ Full | ✅ Full |
| **Justifies containers?** | ❌ No | ✅ Yes (orchestration) | ✅ Yes (GPU access) |

**Decision framework**: Start with Phase 1 for all initial deployments. Add Phase 2 when any of these conditions trigger: (1) business units require models not on Bedrock, (2) need cost optimization across providers, (3) require automatic failover, (4) sophisticated routing based on cost or latency becomes essential. Add Phase 3 only when business case justifies GPU costs and operational complexity.

## Implementation roadmap

See detailed 16-week implementation roadmap in the full architecture document.

## Next steps

1. **Week 1**: Secure executive approval and budget
2. **Week 2**: Complete Well-Architected Review
3. **Week 3-4**: Deploy account structure and connectivity
4. **Week 5-8**: Deploy Phase 1 with pilot application
5. **Week 9+**: Evaluate Phase 2 based on requirements

---

For detailed sections on network architecture, IAM patterns, guardrails, observability, MCP hosting, and cost analysis, see the full documentation.
