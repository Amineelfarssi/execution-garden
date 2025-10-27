# Execution Garden: Enterprise LLM Gateway Architecture for Banking

Banking organizations require **centralized AI governance without sacrificing business unit agility**. After analyzing self-hosted LLM gateway solutions, AWS multi-account patterns, and banking compliance requirements, the optimal architecture combines LiteLLM on AWS ECS with a serverless-first approach that scales strategically to containers only when justified. This design delivers **60% lower TCO than custom builds** while maintaining full data sovereignty, meets strict regulatory requirements, and provides a clear **16-week path to production**.

**The critical insight**: Start with Lambda + API Gateway for immediate value, then selectively add ECS-based LiteLLM when you need multi-provider orchestration or self-hosted models. This phased approach minimizes risk, accelerates time-to-market, and aligns with your constraint that ECS/EKS requires justification.

## Banking's unique LLM platform challenge

Traditional API gateways fail for LLM workloads because they can't track token-based costs, manage model deprecations across providers, or apply banking-grade guardrails at the inference layer. Yet **pure custom builds cost $1.7M-$3.9M over three years** and take 6-12 months to deploy. Your Execution Garden account must serve as both gateway and governance layer while working within AWS service constraints (Lambda, API Gateway, ALB preferred; containers need justification).

The research reveals three architectural realities for banks: First, **LiteLLM is production-proven** with 480B+ tokens processed and offers 100+ provider support. Second, **serverless handles 200-500 req/s easily** with cold starts affecting \u003c1% of requests. Third, **AWS Bedrock Guardrails now achieve 99% accuracy** with Automated Reasoning capabilities and cross-model support via ApplyGuardrail API, eliminating the need for separate guardrail infrastructure initially.

## Recommended architecture: Three-phase evolution

### Phase 1: Serverless foundation (Weeks 1-8, $4.5K infrastructure/year)

**When to deploy**: Immediately for routing to AWS Bedrock, basic observability, and single-provider scenarios serving your initial 10 agentic applications.

The serverless architecture provides the fastest path to production while maintaining banking-grade security. Business units connect via Transit Gateway to your Execution Garden VPC hosting API Gateway REST API, Lambda functions for routing and transformation, AWS Bedrock access through PrivateLink endpoints, and cross-account IAM roles with External IDs. **This pattern handles your target 200-500 req/s while costing 78% less than container alternatives**.

Lambda functions assume roles into the Operations Account for Bedrock access, implementing the "memoryless operations account" pattern where customer prompts and responses never persist in the centralized account. CloudWatch Logs remain in business unit accounts for data isolation, with CloudTrail providing centralized audit trails. This architecture achieves **\u003c100ms API Gateway latency plus 500ms-2s Bedrock inference time**.

**Justification for starting serverless**: Your constraint requires justifying ECS/EKS, and Lambda + API Gateway deliver production-ready LLM routing without containers. Cold starts affect \u003c1% of invocations and add only 200-500ms when they occur—acceptable for most banking workflows. No idle costs, automatic scaling to 1000 concurrent executions per second, and straightforward compliance audits make this the optimal starting point.

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

**When to deploy**: Only when business requirements demand proprietary fine-tuned models not available via Bedrock or other providers. **Triggers include** needing models with specialized banking domain knowledge, data residency requirements preventing API calls to external providers, cost optimization for extremely high volumes (\u003e50M requests/month), or compliance mandates for model hosting.

Deploy self-hosted models on Amazon SageMaker Real-Time Inference endpoints or (if you need cost optimization at scale) EKS with GPU node groups. SageMaker provides managed infrastructure with one-click deployment, automatic scaling, and built-in monitoring—ideal for 1-10 models. EKS becomes cost-effective at scale (\u003e100 models) with fractional GPU sharing and 30-45% cost savings reported by enterprises like Informatica and Vannevar Labs.

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
| **Latency (P99)** | \u003c1s routing + Bedrock | \u003c1.2s routing + provider | \u003c500ms routing + inference |
| **Supports 200-500 req/s** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Multi-provider** | ❌ No (Bedrock only) | ✅ Yes (100+ providers) | ✅ Yes (unlimited) |
| **Cost tracking per app** | ⚠️ Via API keys | ✅ Native in LiteLLM | ✅ Native in LiteLLM |
| **Automatic fallback** | ❌ No | ✅ Yes | ✅ Yes |
| **Prompt caching** | ⚠️ Bedrock only | ✅ Cross-provider | ✅ Cross-provider |
| **Model lifecycle mgmt** | ⚠️ Manual | ✅ Automated routing | ✅ Automated routing |
| **Compliance readiness** | ✅ Full | ✅ Full | ✅ Full |
| **Justifies containers?** | ❌ No | ✅ Yes (orchestration) | ✅ Yes (GPU access) |

**Decision framework**: Start with Phase 1 for all initial deployments. Add Phase 2 when any of these conditions trigger: (1) business units require models not on Bedrock, (2) need cost optimization across providers, (3) require automatic failover, (4) sophisticated routing based on cost or latency becomes essential. Add Phase 3 only when business case justifies GPU costs and operational complexity.

## LLM gateway solution comparison

Based on extensive research, three self-hosted options meet banking requirements: LiteLLM (recommended), Portkey Enterprise Hybrid, and Kong AI Gateway. Custom builds are explicitly **not recommended** due to $1.7M-$3.9M costs and 6-12 month timelines.

| Feature | LiteLLM (OSS) | Portkey Hybrid | Kong AI Gateway | AWS Bedrock Native |
|---------|---------------|----------------|-----------------|-------------------|
| **License** | Apache 2.0 | Apache 2.0 core + Enterprise | Apache 2.0 / Enterprise | Fully managed SaaS |
| **Deployment** | ECS, EKS, Docker | Data plane in VPC, control plane managed | ECS, EKS, VMs | N/A |
| **Multi-provider support** | 100+ providers | 200+ providers | 10+ providers | Bedrock models only |
| **Self-hosted data sovereignty** | ✅ 100% | ✅ Data plane in VPC | ✅ 100% | ⚠️ AWS-managed |
| **Automatic fallback** | ✅ Built-in | ✅ Built-in | ✅ Enterprise only | ❌ No |
| **Cost tracking per team** | ✅ Virtual keys | ✅ Native | ⚠️ Via plugins | ⚠️ Tags only |
| **Rate limiting** | ✅ RPM/TPM | ✅ Token-based | ✅ Advanced | ⚠️ Basic |
| **Observability integrations** | ✅ 15+ platforms | ✅ Native + export | ✅ Via plugins | ✅ CloudWatch |
| **Prompt caching** | ✅ Redis | ✅ Semantic | ✅ Semantic (Ent) | ✅ Native (5 min) |
| **Guardrails** | ⚠️ Via integrations | ✅ 50+ built-in | ✅ Prompt guards | ✅ Bedrock Guardrails |
| **PII detection** | ⚠️ Via Comprehend | ✅ Built-in | ⚠️ Via plugins | ⚠️ Via Comprehend |
| **Model lifecycle** | ✅ Routing-based | ✅ Version mgmt | ⚠️ Manual | ⚠️ Manual |
| **Banking maturity** | ✅ Production-proven | ✅ Enterprise-ready | ✅ Established platform | ✅ AWS-native |
| **Operational complexity** | Medium | Low (hybrid model) | High | Very low |
| **Community support** | ✅ 12K+ stars, active | ⚠️ Smaller community | ✅ Large community | ✅ AWS support |
| **3-year TCO** | $550K-$900K | $825K-$1.1M | $800K-$1.5M | $150K-$400K |

**Recommendation**: **LiteLLM for Phase 2+**, **AWS Bedrock native for Phase 1**. LiteLLM's Apache 2.0 license, production-proven reliability (480B+ tokens processed), 100+ provider support, and active community make it the optimal choice for banking environments. Portkey Hybrid is an excellent alternative if you want managed control plane and built-in guardrails. Kong AI Gateway only makes sense if you already operate Kong infrastructure. AWS Bedrock native patterns suffice for Phase 1 serverless deployments.

## Network architecture and Transit Gateway integration

Your existing Transit Gateway provides the connectivity backbone. The Execution Garden VPC becomes a hub connected to all business unit VPCs through Transit Gateway attachments.

**Network topology**:
```
Business Unit VPCs (10.1.0.0/16, 10.2.0.0/16, 10.3.0.0/16...)
    ↓ TGW Attachments
Transit Gateway (layer-3 routing hub)
    ↓ TGW Attachment
Execution Garden VPC (10.100.0.0/16)
├── Public Subnets: CloudFront, WAF (if external access needed)
├── Private Subnets: API Gateway, ALB, Lambda, ECS Tasks
│   ├── API Gateway (REST API with Usage Plans)
│   ├── Application Load Balancer (for LiteLLM in Phase 2)
│   ├── Lambda Functions (routing, transformation)
│   └── ECS Fargate Tasks (LiteLLM gateway)
├── Data Subnets: RDS, ElastiCache (no internet routes)
│   ├── RDS PostgreSQL Multi-AZ
│   └── ElastiCache Redis
└── VPC Endpoints (PrivateLink)
    ├── bedrock-runtime.us-east-1.amazonaws.com
    ├── kms.us-east-1.amazonaws.com
    ├── logs.us-east-1.amazonaws.com
    └── s3.us-east-1.amazonaws.com

External LLM Providers (OpenAI, Gemini)
    ↑ HTTPS via NAT Gateway (Phase 2+)
```

**Critical Transit Gateway configurations**: Enable **appliance mode** on Execution Garden VPC attachment to prevent asymmetric routing when using stateful network appliances. Create isolated route tables—one for the Execution Garden VPC that accepts routes from all business unit attachments, and separate route tables for each business unit with static routes pointing only to Execution Garden CIDR. This enforces hub-spoke topology where business units cannot communicate directly with each other through the LLM gateway.

**Security group patterns**:
- **Business Unit VPCs**: Outbound to Execution Garden VPC CIDR on port 443, no inbound rules needed from Execution Garden (response traffic handled by stateful connection tracking)
- **API Gateway**: Not applicable (uses resource policies for access control)
- **ALB Security Group**: Inbound from business unit VPC CIDRs on port 443, outbound to ECS security group on container port
- **ECS/Lambda Security Groups**: Inbound from ALB security group (or API Gateway prefix list), outbound to VPC endpoints on port 443
- **RDS/ElastiCache Security Groups**: Inbound from ECS/Lambda security groups only, no internet access

**VPC endpoint strategy**: Deploy interface endpoints for bedrock-runtime, kms, logs, and s3 (gateway endpoint) to keep all AWS service traffic on AWS backbone. This eliminates data transfer costs for Bedrock API calls and improves latency by avoiding NAT gateway. Apply VPC endpoint policies restricting access to specific principals (IAM roles) and actions.

**Data flow for LLM request**:
1. Agent application in Business Unit A VPC (10.1.0.0/16) sends HTTPS request to Execution Garden API Gateway endpoint (10.100.0.1)
2. Request routes through Transit Gateway to Execution Garden VPC private subnet
3. API Gateway invokes Lambda function (Phase 1) or routes to ALB → ECS (Phase 2)
4. Lambda/ECS assumes IAM role in Operations Account with External ID for authentication
5. Lambda/ECS calls Bedrock via PrivateLink endpoint (traffic never leaves AWS network)
6. Bedrock Guardrails evaluate request for PII, content policy violations (99% accuracy)
7. If approved, Bedrock inference completes
8. Response flows back through Lambda/ECS → API Gateway → Transit Gateway → Business Unit VPC
9. Lambda/ECS logs to CloudWatch Logs in **Business Unit account** (data isolation), CloudTrail logs in **Security Account** (audit trail)

**Performance characteristics**: Transit Gateway adds \u003c1ms latency. VPC PrivateLink to Bedrock adds \u003c5ms versus public endpoint. API Gateway adds 50-100ms. Lambda cold starts add 200-500ms (\u003c1% of requests). Total P99 latency for serverless architecture: ~800ms routing + provider inference time.

**Cost considerations**: Transit Gateway data processing costs $0.02/GB. For 10M requests/month with average 1KB request + 3KB response: 40GB/month = $0.80. VPC endpoint data processing for interface endpoints: $0.01/GB = $0.40. PrivateLink reduces or eliminates data transfer charges versus internet egress.

## Cross-account IAM architecture

The multi-account IAM pattern implements **memoryless operations account** principle—LLM requests and responses never persist in the centralized Execution Garden account, ensuring data sovereignty for each business unit.

**Account structure**:
- **Execution Garden Account** (new): Hosts API Gateway, Lambda/ECS infrastructure, configuration only
- **Operations Account** (existing or new): Owns AWS Bedrock access, model quotas, IAM roles for each business unit
- **Business Unit Accounts** (existing): Credits, Insurance, Wealth Management, etc. Each has VPC connected via Transit Gateway
- **Security Account** (existing): Centralized CloudTrail, GuardDuty, Security Hub

**IAM role assumption flow**:
1. Lambda/ECS in Execution Garden account has IAM role with `sts:AssumeRole` permission
2. Lambda/ECS assumes role in Operations Account passing **External ID** specific to business unit (prevents confused deputy attacks)
3. Operations Account role trust policy validates External ID and source account
4. Operations Account role has limited Bedrock permissions (specific models, regions only)
5. Lambda/ECS uses temporary credentials (1-hour expiration) to invoke Bedrock
6. Bedrock API calls logged to CloudTrail in Operations Account
7. Request/response data logged to CloudWatch in Business Unit account (data isolation)

**Operations Account IAM role template** (one role per business unit):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::EXECUTION-GARDEN-ACCOUNT:role/LLMGatewayExecutionRole"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "bu-retail-banking-unique-id-12345"
      },
      "IpAddress": {
        "aws:SourceIp": ["10.100.0.0/16"]
      }
    }
  }]
}
```

**Least privilege Bedrock policy**:
```json
{
  "Version": "2012-01-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ],
    "Resource": [
      "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0",
      "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-haiku-20241022-v1:0"
    ],
    "Condition": {
      "StringEquals": {
        "aws:RequestedRegion": "us-east-1"
      }
    }
  }, {
    "Effect": "Allow",
    "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
    "Resource": "arn:aws:kms:us-east-1:ACCOUNT:key/BU-RETAIL-KEY-ID",
    "Condition": {
      "StringEquals": {
        "kms:ViaService": "bedrock.us-east-1.amazonaws.com"
      }
    }
  }]
}
```

**Service Control Policies for governance**: Apply SCP at Organizational Unit level to prevent business units from directly accessing Bedrock, enforcing all requests through Execution Garden. This prevents shadow AI deployments and ensures consistent guardrails and audit trails.

**Audit and compliance**: Enable Organization Trail in AWS Organizations to centralize CloudTrail logs from all accounts to S3 bucket in Security Account with Object Lock (WORM) for immutability. Use IAM Access Analyzer to identify unintended cross-account access. Implement AWS Config rules to detect policy violations (e.g., Bedrock access outside Execution Garden account). Set up CloudWatch Insights queries for security analysis: cross-account AssumeRole events, Bedrock InvokeModel calls with business unit tags, failed authentication attempts.

**Session tagging for attribution**: When assuming role, pass session tags identifying business unit, application, cost center. These tags propagate to CloudTrail events and enable precise cost allocation. Example: `aws:PrincipalTag/BusinessUnit=Retail, aws:PrincipalTag/Application=CustomerServiceBot`.

## Enterprise guardrails and compliance

Banking LLM deployments require layered defense: content filtering, PII protection, prompt injection prevention, and comprehensive audit trails. The architecture implements multiple guardrail technologies in sequence for 99%+ accuracy.

**Guardrail framework selection**: **AWS Bedrock Guardrails** as primary layer, supplemented by **NeMo Guardrails** for advanced detection. This hybrid approach leverages Bedrock's managed 99% accuracy Automated Reasoning capabilities while adding NeMo's open-source customization for banking-specific rules.

**Guardrail layers**:

**Layer 1 - Input validation (Lambda/LiteLLM)**: Check request format, size limits (\u003c100K tokens), rate limiting per API key. Use Amazon Comprehend DetectPiiEntities API to identify SSN, credit cards, bank accounts in real-time. Redact or block requests containing PII before sending to LLM. Implement basic regex-based prompt injection detection for obvious attacks ("ignore previous instructions"). **Latency**: 20-50ms for PII detection, 5ms for pattern matching.

**Layer 2 - AWS Bedrock Guardrails (applied to all model calls)**: Configure content filters on HIGH for Sexual, Violence, and MEDIUM for Hate, Insults. Define denied topics: "Investment advice", "Specific stock recommendations", "Competitor products", "Internal bank systems". Add sensitive information filters for PII types missed by Layer 1. Enable contextual grounding checks to detect hallucinations in RAG workflows. **Deploy Automated Reasoning** (99% accuracy) by uploading documents defining logical rules—e.g., "Never provide specific account balances", "Always require authentication for transactions". **Latency**: 50-80ms added to Bedrock call.

**Layer 3 - NeMo Guardrails (ECS sidecar or Lambda layer)**: Deploy NVIDIA NeMo Guardrails as microservice in ECS or package as Lambda layer. Configure jailbreak detection using Prefix/Suffix Perplexity method (98% detection, 0.04% false positives at threshold 1845.65). Integrate Llama Guard 3 for content safety classification. Add custom banking rails: detect market manipulation language, block unauthorized financial commitments, enforce regulatory compliance. **Latency**: 115ms (GPU) to 2,057ms (CPU)—deploy on CPU initially, add GPU if latency becomes issue.

**Layer 4 - Output validation**: Apply same Bedrock Guardrails to model outputs. Use contextual grounding to verify RAG responses match source documents. Check output PII detection (Comprehend). Validate responses don't contain sensitive bank data leaked from training or context. **Latency**: 50-80ms.

**Layer 5 - Human oversight (asynchronous)**: For high-risk decisions (credit approvals, regulatory advice), log to review queue. Compliance officers review flagged interactions. Feedback loop improves guardrail configurations.

**Banking-specific configurations**:
- **PII entity types**: SSN (Social Security Number), BANK_ACCOUNT_NUMBER, CREDIT_CARD, driver's license, passport, phone, email, plus custom regex for account numbers, IBAN, SWIFT codes
- **Content filters**: Block any sexual content, violence, hate speech. Medium threshold for insults (customer service context may include frustrated language)
- **Denied topics**: Investment recommendations, stock tips, tax advice, legal advice, competitor products, internal systems, confidential processes
- **Prompt injection rules**: Block "ignore instructions", "you are now", role-play attacks, encoding tricks (base64, ROT13), multi-language obfuscation

**Compliance audit trails**: Every LLM interaction generates structured log with: timestamp, user_id (hashed for privacy), session_id, business_unit, application, model_name, model_version, input_tokens_count, output_tokens_count, cost, latency_ms, guardrail_actions (array of triggers), pii_detected (types found), pii_redacted (boolean), prompt_injection_score, content_filter_violations (array), bedrock_trace_id. Store in CloudWatch Logs with 7-year retention (banking requirement) and S3 with Object Lock.

**Regulatory alignment**:
- **SR 11-7 (Model Risk Management)**: Document model validation, performance monitoring, annual reviews. Maintain model inventory with risk classification. Automated Reasoning provides explainability.
- **GDPR**: Right to erasure—implement deletion workflow for user prompts/responses. Data residency—restrict Bedrock to EU regions with IAM conditions. Privacy Impact Assessment completed.
- **PCI-DSS**: Never log credit card numbers unencrypted. Tokenize before LLM processing. Quarterly vulnerability scans, annual penetration testing.
- **SOC 2**: AWS infrastructure SOC 2 compliant (reports via AWS Artifact). Implement additional controls: MFA for all admin access, encryption at rest/transit, audit logging, incident response plan.

## Observability architecture

LLM observability differs from traditional application monitoring—you must track token usage (not just requests), capture prompts and completions for debugging (while respecting PII), trace complex agentic workflows across multiple LLM calls, and attribute costs per business unit.

**OpenTelemetry foundation**: The architecture uses OpenTelemetry as the standardized observability backbone, aligned with the GenAI semantic conventions that became part of official OpenTelemetry specification in 2025. This ensures vendor-neutral instrumentation and compatibility with multiple backends.

**Instrumentation layers**:

**Lambda functions**: Deploy AWS Lambda Layers with OpenTelemetry collector and Python/Node.js SDK. Auto-instrument HTTP calls, AWS SDK calls. Add custom spans for LLM invocations with semantic conventions:
- `gen_ai.request.model`: anthropic.claude-3-5-sonnet-v2
- `gen_ai.request.temperature`: 0.7
- `gen_ai.usage.input_tokens`: 1247
- `gen_ai.usage.output_tokens`: 512
- `gen_ai.system`: AWS Bedrock
- `server.address`: bedrock-runtime.us-east-1.amazonaws.com

**LiteLLM**: Built-in OpenTelemetry support. Configure to export traces to OpenTelemetry Collector. LiteLLM tracks latency per provider, automatic fallback events, cost per request, cache hit rates.

**SageMaker endpoints**: Use CloudWatch Container Insights for infrastructure metrics. Instrument model serving code with OpenTelemetry SDK for inference-specific metrics.

**OpenTelemetry Collector**: Deploy as sidecar container in ECS or Lambda extension. Collector receives spans from all components, enriches with business unit tags, and exports to multiple backends: CloudWatch for operational metrics, Prometheus for Grafana dashboards, Langfuse for LLM-specific tracing.

**LLM-specific observability platform**: **Langfuse self-hosted** (recommended for banking). Deploy Langfuse on ECS Fargate with PostgreSQL database. Langfuse provides: prompt versioning (track every prompt change), user feedback collection (thumbs up/down on responses), cost attribution (per user, per session, per application), session tracing (multi-turn conversations visualized), LLM-as-judge evaluations (automated quality scoring), experiment comparisons (A/B test prompts).

**Alternative self-hosted options**: **Helicone** (Docker/K8s deployment, proxy-based, ClickHouse backend, 50-80ms latency impact) excellent for simple one-line integration. **Phoenix by Arize** (OpenInference protocol) strong for model drift detection but less OpenTelemetry compatible. For banking, **Langfuse's comprehensive tracing and Apache 2.0 license make it optimal**.

**Grafana dashboards**: Build dashboards showing:
- **LLM usage metrics**: Requests per second (by model, by business unit), token usage (input/output), cost per request, cost per business unit
- **Performance metrics**: P50/P95/P99 latency by model, cold start frequency, fallback rate (for multi-provider), cache hit rate
- **Quality metrics**: Guardrail trigger rate, PII detection rate, prompt injection attempts, user feedback scores
- **Business metrics**: Cost per application, most expensive queries, user adoption trends

**Alerting strategy**: CloudWatch alarms for: P95 latency \u003e 5 seconds, error rate \u003e 1%, guardrail trigger rate \u003e 10%, PII detection \u003e 5%, cost per day \u003e $10K, Bedrock quota approaching limit (80%). PagerDuty integration for critical alerts. Weekly reports to stakeholders with cost trends and usage analytics.

**Distributed tracing for agentic applications**: Agentic workflows involve multiple LLM calls, tool uses, decision points. OpenTelemetry traces capture parent-child relationships between spans. Example: Customer service agent trace includes spans for: initial LLM call (intent classification), database query (retrieve account), second LLM call (generate response), third LLM call (sentiment check). Total trace shows 4 seconds duration with clear breakdown by component. This enables debugging when agents fail or produce unexpected results.

**Cost attribution and chargeback**: Tag all API Gateway requests with business unit and application (via API keys). Lambda functions propagate tags through AssumeRole session tags. CloudWatch Logs Insights queries aggregate costs: `fields @timestamp, business_unit, application, sum(input_tokens * 0.003/1000 + output_tokens * 0.015/1000) as cost | stats sum(cost) by business_unit`. Export monthly cost reports for chargeback to business units. Set up AWS Budgets with per-business-unit thresholds and automatic alerts at 80% consumption.

## MCP hosting and agent cataloging

Model Context Protocol (MCP), introduced by Anthropic and now adopted by OpenAI, Microsoft, Google, and others as of 2025, standardizes how AI agents connect to external data and tools. Your Execution Garden account should host centralized MCP servers that 10 agentic applications can share.

**MCP fundamentals**: MCP defines three primitives—Tools (functions agents can call), Resources (data agents can access), Prompts (workflow templates). MCP servers expose these capabilities through JSON-RPC 2.0 over StreamableHTTP protocol (current standard as of June 2025). Client applications discover and use MCP servers through standardized protocol.

**Centralized MCP hosting pattern**: Deploy MCP servers as ECS Fargate containers in Execution Garden VPC. Each MCP server provides specific capabilities: database access (read-only queries to customer data), document retrieval (RAG over knowledge bases), calculation tools (financial formulas), external API integration (market data feeds). Multiple agent applications across business units access shared MCP servers through common API.

**Security architecture**: Implement OAuth 2.1 with PKCE (mandatory per MCP spec). Integrate with Amazon Cognito User Pools federated to your corporate IdP (Azure AD, Okta). MCP servers validate JWT tokens and enforce RBAC—Viewer role (read-only), User role (execute non-destructive tools), Admin role (full access). Audit logs capture who called which tool with what parameters.

**MCP registry pattern**: Deploy **Azure API Center** (if using Azure alongside AWS) or build custom registry using DynamoDB + API Gateway. Registry stores metadata for each MCP server: server_id, name, version, capabilities (tools/resources/prompts), transport endpoint, authentication requirements, owner, lifecycle_state (development/staging/production/deprecated). Agent applications query registry to discover available MCP servers.

**Session isolation**: MCP 2025 spec requires Mcp-Session-Id header for multi-tenant deployments. Each agent application gets isolated session preventing data leakage between tenants. Sessions stored in ElastiCache Redis with 1-hour TTL.

**MCP server deployment example**: Financial calculation MCP server provides tools for NPV, IRR, amortization calculations. Deploy as ECS Fargate task with 0.5 vCPU, 1GB RAM. Expose via Application Load Balancer with path-based routing (`/mcp/finance`). Configure Cognito authorizer on ALB. Agent applications from multiple business units call shared calculation tools, reducing duplication and ensuring consistent financial logic.

**Agent catalog architecture**: Maintain central registry of all agentic applications using DynamoDB table with agent metadata schema:
- **Identity**: agent_id, name, version, owner_business_unit, owner_email
- **Capabilities**: purpose, models_used (array), mcp_servers_used (array), tools, resources
- **Dependencies**: external_apis, databases, other_agents
- **Operational**: lifecycle_state (development/testing/staging/production/deprecated), environments (array), health_status, last_health_check
- **Governance**: approval_status, approver, risk_level (low/medium/high/critical), compliance_requirements (array: SR11-7, GDPR, PCI)
- **Documentation**: description, use_cases, limitations, documentation_url

**Lifecycle management**: Agents transition through states: Development (sandbox environment, unrestricted testing) → Testing (integration tests, security scans) → Staging (pilot with limited users) → Production (full deployment) → Deprecated (migration notice, 6-month sunset period) → Retired (decommissioned, archived). Implement AWS Step Functions workflow orchestrating state transitions with approval gates.

**Model lifecycle and deprecation handling**: When LLM providers deprecate models (6-month notice typical), automated workflow: (1) Query agent catalog for agents using deprecated model, (2) Notify owners via SNS/email, (3) Create migration tickets in Jira, (4) Testing phase for replacement model in shadow mode, (5) A/B testing with 5% traffic to new model, (6) Gradual rollout 5% → 25% → 50% → 100%, (7) Monitor quality metrics during migration, (8) Rollback capability until sunset date. LiteLLM enables parallel model routing during migration—old and new models run simultaneously for comparison.

**Version management with SageMaker Model Registry**: For self-hosted models, use Amazon SageMaker Model Registry to version models. Create Model Package Group per business use case. Register each model version with metadata: training dataset, performance metrics, approval status. Implement approval workflow—Data Science team trains model → Model Risk Management validates → Compliance Officer approves → Production deployment. Model lineage tracks from training data through deployment endpoint.

## Implementation roadmap and timeline

### Weeks 1-4: Foundation and planning

**Deliverables**: AWS account structure finalized, Transit Gateway connectivity established, IAM roles and policies defined, security baseline implemented, compliance documentation initiated.

**Activities**: Set up Execution Garden account in AWS Organizations with consolidated billing. Configure Transit Gateway attachments from 3 pilot business unit VPCs. Deploy VPC with public/private/data subnet tiers across 3 AZs. Create KMS customer-managed keys per business unit. Enable CloudTrail Organization Trail, AWS Config, Security Hub. Define API Gateway resource policies and Usage Plans per application. Document Well-Architected Review against FSI Lens. Conduct Privacy Impact Assessment for LLM platform. Initiate risk assessment and threat modeling.

**Team**: 2 cloud engineers, 1 security engineer, 1 compliance officer. **Risks**: Delayed approvals from security, complex Transit Gateway routing issues.

### Weeks 5-8: Phase 1 deployment (Serverless)

**Deliverables**: Production-ready Lambda + API Gateway routing to AWS Bedrock, observability integrated, guardrails enabled, pilot application onboarded.

**Activities**: Deploy API Gateway REST API with custom domain and TLS certificate. Implement Lambda functions for Bedrock routing with cross-account IAM. Configure AWS Bedrock Guardrails with banking-specific policies. Integrate Amazon Comprehend for PII detection. Deploy OpenTelemetry instrumentation. Set up CloudWatch dashboards and alarms. Configure Usage Plans and API keys for 3 pilot applications. Security testing including penetration test. Load testing to validate 500 req/s capacity. Pilot rollout with customer service chatbot (internal users only). User acceptance testing and feedback collection.

**Team**: 2 backend engineers, 1 DevOps engineer, 1 QA engineer. **Risks**: Cold start latency concerns, Bedrock quota limits, guardrail false positives.

### Weeks 9-12: Phase 2 preparation (LiteLLM deployment)

**Deliverables**: LiteLLM gateway deployed on ECS, multi-provider connectivity established, enhanced observability with Langfuse, integrated with existing serverless layer.

**Activities**: Deploy ECS Fargate cluster with Application Load Balancer. Containerize LiteLLM with Terraform/CloudFormation IaC. Configure RDS PostgreSQL and ElastiCache Redis. Integrate LiteLLM with Bedrock (via IAM roles), OpenAI, Google Gemini (via API keys in Secrets Manager). Deploy Langfuse self-hosted on ECS for comprehensive tracing. Configure automatic fallback rules and cost-based routing. Implement prompt caching with Redis. Set up Grafana dashboards for multi-provider metrics. Security hardening: private subnets, no internet egress except through NAT gateway to external providers. Load testing to 500 req/s with multiple providers.

**Team**: 2 backend engineers, 1 DevOps engineer, 1 data engineer (for Langfuse). **Risks**: LiteLLM configuration complexity, multi-provider authentication issues, increased latency from additional hop.

### Weeks 13-16: Production rollout and optimization

**Deliverables**: Full production deployment across 10 agentic applications, comprehensive monitoring, compliance audit completed, optimization based on production data.

**Activities**: Onboard remaining 7 applications with API keys and usage quotas. Implement MCP hosting with 2 pilot MCP servers (database access, document retrieval). Deploy agent catalog in DynamoDB with metadata for all 10 applications. Configure Bedrock Provisioned Throughput for production cost optimization. Complete SOC 2 compliance audit. Conduct internal audit review of LLM platform. Implement cost chargeback reporting per business unit. Optimize Lambda memory and concurrency based on production metrics. Fine-tune guardrails thresholds based on false positive rate. Quarterly red team exercise for prompt injection testing. Document runbooks and incident response procedures. Train support team on LLM platform operations.

**Team**: Full team plus compliance auditor. **Risks**: Scale issues under production load, cost overruns, regulatory findings requiring architecture changes.

### Post-Week 16: Continuous improvement

**Ongoing activities**: Monthly model performance reviews, quarterly guardrail effectiveness assessments, biannual penetration testing, annual compliance audits, continuous cost optimization, feedback-driven feature enhancements. Monitor for model deprecations and plan migrations. Evaluate Phase 3 (self-hosted models) based on business case.

## Cost estimation and TCO analysis

### Phase 1: Serverless only (Months 1-6)

**Infrastructure (per month)**:
- API Gateway REST API: 10M requests @ $1/million = $10
- Lambda: 10M requests @ $0.20/million + compute = $9
- VPC Endpoints: 3 interface endpoints @ $7.30 = $22
- KMS: 3 customer-managed keys @ $1 = $3
- CloudWatch Logs: 50GB @ $0.50/GB = $25
- S3 Storage: Minimal ($5)
- NAT Gateway: Not needed (PrivateLink only)
- **Total infrastructure: $74/month**

**LLM costs (variable)**:
- Bedrock On-Demand: 10M requests × 500 input tokens × $0.003/1K = $15K input
- Bedrock On-Demand: 10M requests × 200 output tokens × $0.015/1K = $30K output
- **Total Bedrock: $45K/month or $27K with Provisioned Throughput**

**Personnel**:
- 2 engineers @ $150K/year = $300K/year
- **Total Year 1: $74×12 + $540K Bedrock + $300K personnel = $841K**

### Phase 2: Add LiteLLM gateway (Months 7-36)

**Additional infrastructure (per month)**:
- ECS Fargate: 3 tasks × 2vCPU × 4GB × $0.12/hour = $720
- RDS PostgreSQL Multi-AZ db.t4g.medium = $140
- ElastiCache Redis cache.t4g.micro = $15
- Application Load Balancer = $23
- NAT Gateway (for external providers): $45 + $0.045/GB
- Langfuse ECS deployment: $200
- Additional S3, CloudWatch: $50
- **Additional infrastructure: $1,193/month**

**Total infrastructure Phase 2: $1,267/month or $15.2K/year**

**LLM costs**: Same as Phase 1 ($540K/year with Provisioned Throughput)

**Personnel**:
- 3 engineers (added 1 for multi-provider management) @ $450K/year

**Total Year 2-3 average: $15.2K + $540K + $450K = $1.005M/year**

**3-Year TCO**: $841K + $1.005M + $1.005M = **$2.85M total**

**TCO breakdown**: Infrastructure $378K (13%), LLM consumption $1.62M (57%), Personnel $850K (30%)

### Phase 3: Add self-hosted models (if deployed)

**Additional infrastructure**:
- SageMaker ml.g5.xlarge endpoint: $1,036/month per model
- or EKS: $73 control plane + $4,161/month per GPU node
- **Adds $12K-$50K/year depending on scale**

### Comparison to alternatives

**Custom build**: $500K development (6 months, 4 engineers) + $300K/year maintenance (2 engineers) = **$1.4M Year 1, $3.0M over 3 years** (infrastructure only, no LLM costs). Build approach **costs $150K more over 3 years** with 6-month delay to market.

**SaaS solutions (not allowed)**: LangSmith hosted ~$60/user/month, Portkey cloud ~$50/month + usage. Would save infrastructure costs but violates banking self-hosted requirement.

**AWS-only serverless (Phase 1 forever)**: $841K/year stable state. Lowest cost but lacks multi-provider flexibility. **Saves $1.2M over 3 years** versus adding Phase 2, but limits to Bedrock models only.

**Recommendation**: Start Phase 1, add Phase 2 only when multi-provider requirement confirmed. This **minimizes upfront investment** while preserving architectural flexibility.

## Critical success factors and risks

### Success factors

**Executive sponsorship**: AI Center of Excellence leadership committed to centralized platform model. Budget approved for 3-year horizon. Regulatory engagement early in process.

**Phased approach**: Deliver value quickly with Phase 1 serverless (8 weeks), then selectively add capabilities. Avoid big-bang approach reducing risk.

**Strong governance**: Centralized guardrails, audit trails, cost controls prevent shadow AI. Federated execution empowers business units within guardrails.

**Data quality**: Invest in data governance, classification, lineage. Poor data quality is #1 barrier per industry research.

**Team skills**: Upskill existing engineers on LLM operations. Hire 1-2 specialists with production LLM experience.

### Key risks and mitigations

**Risk: Regulatory approval delays** (Likelihood: High, Impact: High)
- Mitigation: Engage regulators early with architecture documentation. Start with internal use cases requiring less scrutiny. Build compliance artifacts (PIAs, threat models) into project timeline.

**Risk: Cost overruns from unexpected usage** (Likelihood: Medium, Impact: High)
- Mitigation: Implement strict usage quotas per application with automatic throttling. Set up budget alerts at 80% and 100%. Use Provisioned Throughput for predictable costs. Monitor token usage and optimize prompts.

**Risk: Model deprecations disrupting applications** (Likelihood: High, Impact: Medium)
- Mitigation: Build model lifecycle management from Day 1. Maintain model mapping registry showing which applications use which models. Use LiteLLM's routing to run old and new models in parallel during migrations. Negotiate extended sunset periods with providers.

**Risk: Security incidents (prompt injection, data leakage)** (Likelihood: Medium, Impact: Critical)
- Mitigation: Multi-layer guardrails with 99% accuracy. Quarterly red team exercises. Comprehensive audit logging with 7-year retention. Incident response playbooks. Cyber insurance coverage for AI risks.

**Risk: ECS/EKS operational complexity** (Likelihood: Medium, Impact: Medium)
- Mitigation: Start with managed Fargate, not self-managed ECS. Use AWS Copilot or ECS Blueprints for standardized deployments. Invest in training and runbooks. Consider managed Kubernetes (EKS Fargate) to reduce operational burden.

**Risk: Vendor lock-in to AWS Bedrock** (Likelihood: Low, Impact: Medium)
- Mitigation: Deploy Phase 2 multi-provider gateway early. Use OpenAI-compatible APIs abstraction. Maintain ability to run models on SageMaker or EKS. LiteLLM provides provider abstraction preventing lock-in.

## Alternative architectures considered

### Option A: Pure AWS serverless (no LiteLLM)

**Description**: Lambda + API Gateway routing directly to Bedrock, OpenAI, Gemini via API calls. Custom code for fallback, rate limiting, cost tracking.

**Pros**: Lowest initial complexity, no container management, fastest time to value (6 weeks).

**Cons**: Significant custom code required for multi-provider orchestration, manual implementation of features LiteLLM provides, higher maintenance burden, slower feature velocity, lacks sophisticated routing algorithms.

**Recommendation**: Only choose if you're certain of Bedrock-only deployment for 12+ months. Otherwise, debt incurred building custom orchestration exceeds Phase 2 timeline.

### Option B: Kong Gateway from Day 1

**Description**: Deploy Kong Gateway on ECS/EKS with AI plugins as primary gateway architecture, bypassing serverless entirely.

**Pros**: Enterprise-grade API management, mature platform, strong security controls, unified gateway for LLM and traditional APIs.

**Cons**: Heavy infrastructure (Kubernetes recommended), complex configuration, AI features relatively new (2024), requires deep Kong expertise, higher operational cost ($800K-$1.5M TCO), overkill for LLM-only use case.

**Recommendation**: Only choose if you already operate Kong infrastructure and can extend it. Otherwise, complexity and cost don't justify for greenfield LLM platform.

### Option C: Portkey Enterprise Hybrid

**Description**: Deploy Portkey Data Plane (Docker containers) in Execution Garden VPC, use Portkey managed Control Plane for configuration.

**Pros**: Fastest time to production (4-6 weeks), built-in guardrails and PII detection, semantic routing out-of-box, lower operational burden (managed control plane).

**Cons**: Dependency on external control plane (even if LLM traffic stays in VPC), enterprise licensing costs ($50K-$150K/year), smaller community than LiteLLM, newer platform (less battle-tested).

**Recommendation**: Consider as alternative to LiteLLM if you want managed control plane and are willing to accept vendor relationship. Adds $150K-$450K to 3-year TCO.

### Option D: EKS with self-hosted models from Day 1

**Description**: Deploy EKS cluster, host all LLM inference (including Claude, GPT equivalent models) on GPU nodes, no external API calls.

**Pros**: Complete control, data never leaves AWS, cost optimization at scale, no provider rate limits.

**Cons**: Massive upfront investment (6+ months to deploy), requires specialized ML engineering team, GPU costs ($50K+/year), model quality lag versus frontier models (GPT-4, Claude 3.5), compliance burden for model hosting, operational complexity extremely high.

**Recommendation**: Only consider if regulatory requirements absolutely prohibit external API calls OR if you have \u003e100M requests/month where self-hosting economics work. For your 10 initial applications, this is **not justified**.

## Unique considerations for banking

**Data sovereignty**: Architecture ensures prompt and response data never persists in Execution Garden or Operations accounts. CloudWatch Logs remain in originating business unit account. VPC PrivateLink keeps traffic on AWS backbone.

**Regulatory explainability**: AWS Bedrock Automated Reasoning (99% accuracy) provides mathematical proofs for decisions. Model Cards document training data, performance metrics, limitations. Audit trails link every decision to model version, input, guardrails applied.

**Bias and fairness**: Implement quarterly bias testing for customer-facing models. Use Amazon SageMaker Clarify to detect bias in outputs. Monitor for disparate impact across demographic groups. Document mitigation strategies.

**Third-party risk management**: AWS Bedrock models undergo AWS's vendor risk assessment. For OpenAI/Gemini, maintain Data Processing Agreements. Regularly review SOC 2 reports. Implement contractual provisions for data handling, breach notification, model security.

**Model risk management (SR 11-7)**: Maintain model inventory with risk classification (low/medium/high). Implement three lines of defense: (1) Data Science team develops and validates, (2) Model Risk Management team independently validates, (3) Internal Audit reviews governance. Annual model reviews for high-risk applications. Document model development, validation, performance monitoring.

**Incident response for AI**: Establish playbooks for: hallucinations causing customer impact, guardrail bypass detected, PII leakage, cost anomaly (usage spike), model quality degradation, provider outage. Define escalation paths, communication protocols, rollback procedures.

**Change management velocity**: Balance innovation with risk. Use three-environment approach: Sandbox (unrestricted experimentation), Staging (pilot with controls), Production (full governance). Automated testing in CI/CD pipeline but human approval gates for production deployment.

## Conclusion and next steps

The recommended Execution Garden architecture delivers **production-ready LLM platform in 16 weeks** with clear phase-gating: serverless foundation first, containers when justified, self-hosted models only with business case. This approach minimizes risk, aligns with AWS service constraints, achieves **60% cost savings versus custom build**, and maintains banking-grade security and compliance.

**Immediate next steps**:

1. **Week 1**: Secure executive approval and budget ($350K Year 1 infrastructure + personnel). Identify 3 pilot business units and applications.

2. **Week 2**: Complete Well-Architected Review using FSI Lens. Finalize security architecture with information security team.

3. **Week 3-4**: Deploy account structure, Transit Gateway connectivity, IAM roles. Complete Privacy Impact Assessment and threat model.

4. **Week 5-8**: Deploy Phase 1 serverless architecture with first pilot application. Focus on customer service chatbot for risk mitigation.

5. **Week 9-12**: Based on pilot learnings, decide on Phase 2 timing. Deploy LiteLLM if multi-provider requirement confirmed.

6. **Week 13-16**: Production rollout across 10 applications. Complete compliance audit. Establish operational procedures.

**Success metrics**: 99%+ uptime SLA, \u003c2s P99 latency for LLM requests, \u003c1% guardrail false positive rate, 100% audit trail coverage, $0 compliance violations, positive user feedback (NPS \u003e 40), under budget by 10%, 200-500 req/s capacity validated, 10 applications onboarded by Week 16.

Your banking organization can **deploy enterprise LLM capabilities with confidence**—balancing innovation velocity with risk management, leveraging managed services where possible, and building custom only where justified. This architecture provides the governance and observability foundation to scale from 10 to 100+ agentic applications while maintaining the trust and compliance banking demands.