---
date: 2025-12-31
title: The Business Case for CPU-Based AI Inference
authors:
  - cloudnull
description: >
  The Business Case for CPU-Based AI Inference
categories:
  - OpenStack
  - Zen
  - AMD
  - ZenDNN
  - ZenTorch
  - AI

---

# The Business Case for CPU-Based AI Inference

**Your finance team doesn't care about tokens per second. They care about predictable costs, compliance risk, and vendor lock-in. Here's how CPU inference stacks up.**

The other week I published a [technical deep-dive on running LLM inference with AMD EPYC processors and ZenDNN](2025-12-17-ZenDNN-ai-enablement.md). The benchmarks showed that a $0.79/hour VM can push 40-125 tokens per second depending on model size, genuinely usable performance for a surprising range of workloads.

But benchmarks don't answer the question that actually matters: **Should you do this?**

<!-- more -->

That depends on your volume, your compliance requirements, and your appetite for vendor dependency. Let's work through the math and the strategy.

## The Economics: What Does CPU Inference Actually Cost?

The $0.79/hour headline is useful, but CIOs and finance teams think in cost-per-unit. For AI inference, that unit is tokens. Let's translate.

### Cost-Per-Token Calculation

Using results from the Qwen3-4B model, a capable 4-billion parameter model suitable for production workloads:

| Metric | Value |
|--------|-------|
| Output throughput | 60.00 tokens/sec |
| Tokens per hour | 216,000 |
| Instance cost | $0.79/hour |
| **Cost per million output tokens** | **$3.66** |

For other models in the benchmark suite:

| Model | Parameters | Tokens/sec | Tokens/Hour | Cost per Million Tokens |
|-------|------------|------------|-------------|------------------------|
| Qwen3-0.6B | 0.6B | 124.81 | 449,316 | $1.76 |
| Llama-3.2-1B | 1B | 123.95 | 446,220 | $1.77 |
| Gemma-3-1b-it | 1B | 97.35 | 350,460 | $2.25 |
| Qwen3-1.7B | 1.7B | 95.55 | 343,980 | $2.30 |
| Llama-3.2-3B | 3B | 71.87 | 258,732 | $3.05 |
| Phi-4-mini-instruct | 4B | 67.32 | 242,352 | $3.26 |
| Qwen3-4B | 4B | 60.00 | 216,000 | $3.66 |
| Gemma-3-4b-it | 4B | 53.83 | 193,788 | $4.08 |
| Qwen3-8B | 8B | 39.75 | 143,100 | $5.52 |
| Gemma-3-12b-it | 12B | 25.36 | 91,296 | $8.65 |
| Phi-4 | 15B | 23.83 | 85,788 | $9.21 |

These numbers assume continuous operation at observed throughput rates. Real-world utilization varies, but the math scales linearly.

### How Does This Compare to Commercial APIs?

Here's where honesty matters more than advocacy.

| Provider | Model | Output $/Million | Capability Class |
|----------|-------|------------------|------------------|
| OpenAI | GPT-4o-mini | $0.60 | Small, fast |
| **Self-hosted** | Qwen3-0.6B | $1.76 | Small, fast |
| **Self-hosted** | Llama-3.2-1B | $1.77 | Small, fast |
| **Self-hosted** | Qwen3-4B | $3.66 | Medium, capable |
| Anthropic | Claude Haiku 3.5 | $4.00 | Small, capable |
| OpenAI | GPT-4o | $10.00 | Large, sophisticated |
| Anthropic | Claude Sonnet 4.5 | $15.00 | Large, sophisticated |

**The honest take:** On pure cost-per-token for the smallest models, GPT-4o-mini wins. At $0.60 per million output tokens, OpenAI's pricing reflects massive scale economies that self-hosted infrastructure can't match at low volumes.

But that comparison is incomplete.

### The Crossover: Where Self-Hosted Wins

Commercial API pricing assumes you pay per token. Self-hosted infrastructure costs the same whether you process one token or one billion.

#### Scenario: Customer support automation

Processing 100,000 tickets monthly, averaging 500 tokens input and 300 tokens output each.

Monthly volume: 50 million input tokens + 30 million output tokens

| Approach | Model | Monthly Cost |
|----------|-------|--------------|
| OpenAI API | GPT-4o-mini | $25.50 |
| Anthropic API | Claude Sonnet 4.5 | $600.00 |
| **Self-hosted** | Qwen3-4B (continuous) | $575.00 |

At 80 million tokens monthly, self-hosted CPU inference matches Claude Sonnet pricing while providing a model you fully control.

**Scale it further:** 500,000 tickets monthly (400 million tokens):

| Approach | Model | Monthly Cost |
|----------|-------|--------------|
| OpenAI API | GPT-4o-mini | $127.50 |
| Anthropic API | Claude Sonnet 4.5 | $3,000.00 |
| **Self-hosted** | Qwen3-4B (continuous) | $575.00 |

Now self-hosted beats everything except the cheapest API tier, and that cheapest tier comes with constraints that matter.

### The Hidden Costs APIs Don't Show

Commercial API pricing excludes costs that don't appear on invoices but absolutely appear in your operations:

| Factor | API Reality | Self-Hosted Reality |
|--------|-------------|---------------------|
| Rate limits | Throttling during demand spikes | Your capacity, your limits |
| Outages | Service degradation outside your control | Hardware fails predictably, locally |
| Price changes | OpenAI has changed pricing multiple times | Hardware costs are commodity markets |
| Model deprecation | GPT-3 is gone; integrations break | Downloaded weights are permanent |
| Compliance burden | Proving data handling to auditors | Data never leaves your infrastructure |

Self-hosted infrastructure has its own hidden costs, operations, maintenance, expertise. But those costs are visible and controllable. API costs are opaque until the invoice arrives or the service degrades.

### The Break-Even Framework

**When APIs make sense:**

- Under 20 million tokens monthly
- No compliance constraints on data handling
- Variable, unpredictable workload patterns
- Time-to-value priority over long-term cost optimization

**When self-hosted makes sense:**

- Over 50 million tokens monthly
- Regulated data requiring sovereignty
- Predictable, sustained workload patterns
- Strategic priority on vendor independence

**The hybrid approach:** Self-hosted for baseline capacity, API overflow for burst demand. Fixed costs stay controlled; spikes get absorbed without over-provisioning.

## Beyond Cost: Control, Compliance, and Continuity

For many organizations, cost-per-token isn't the deciding factor. Three strategic considerations often matter more.

### Data Sovereignty: The Compliance Imperative

Every token sent to a commercial API leaves your infrastructure. For organizations handling protected health information, personally identifiable information, financial data, or classified information, this creates immediate complexity.

**The HIPAA reality:** When patient data goes to OpenAI's API, you're trusting their Business Associate Agreement, their security practices, their employee access controls, and their data retention policies. You're also trusting that no prompt injection or model behavior will cause data leakage across tenant boundaries.

Self-hosted inference eliminates this trust chain. PHI never leaves your network. Audit trails exist on systems you control. Data residency is whatever you configure.

| Sector | API Challenge | Self-Hosted Advantage |
|--------|---------------|----------------------|
| Healthcare | BAA complexity, PHI exposure | Complete data isolation |
| Financial Services | SOX audit requirements | Full audit trail ownership |
| Legal | Attorney-client privilege | Air-gapped deployment option |
| Government | FedRAMP, ITAR, classification | On-premise or sovereign cloud |

> This isn't theoretical, multiple healthcare organizations in the USE and several European medical centers run production AI workloads on OpenStack specifically because data sovereignty requirements preclude commercial API usage.

### Vendor Independence: The Lock-In Calculation

Commercial AI APIs create dependencies that compound over time.

**Model availability risk:** OpenAI has deprecated models multiple times. GPT-3 is gone. GPT-4 variants come and go. If your application depends on specific model behavior, you're one deprecation notice away from an emergency rewrite.

**Pricing power asymmetry:** Once your application is built on a specific API, switching costs are substantial. GPT-4 launched at $60/$30 per million tokens and dropped to $10/$2.50. Good for customers, but it demonstrates pricing is a strategic decision by the vendor, not a stable input to your financial planning.

**The open alternative:** Self-hosted inference on open-weight models (Llama, Qwen, Gemma, Phi) provides permanent model availability, commodity-market pricing for hardware, and portability across any deployment target. If your cloud provider disappeared tomorrow, vLLM containers run identically elsewhere. The same cannot be said for applications built on proprietary API features.

### Operational Continuity: Failure Modes Matter

Commercial AI services experience outages. When the API is down, your AI-powered features are down.

Self-hosted infrastructure fails differently. Hardware fails predictably and locally. You build redundancy. You control failover. You own the blast radius.

| Failure Mode | API Impact | Self-Hosted Impact |
|--------------|------------|-------------------|
| Provider outage | Complete service loss | N/A |
| Rate limiting | Degraded service, queuing | N/A |
| Network partition | Service loss | Local operations continue |
| Hardware failure | N/A | Failover to redundant capacity |
| Demand spike | Throttling, latency | Scale horizontally |

For applications where AI is core functionality, availability on your terms matters.

## The Strategic Framework

Evaluate AI infrastructure across three dimensions:

### Data Sensitivity

- Low sensitivity, non-regulated → APIs are fine
- Moderate sensitivity, compliance-conscious → Evaluate carefully
- High sensitivity, heavily regulated → Self-hosted likely required

### Volume and Predictability

- Low volume, variable → APIs offer flexibility
- High volume, predictable → Self-hosted offers cost stability
- Burst capacity needs → Hybrid approach

### Strategic Importance

- Experimental/exploratory → APIs minimize commitment
- Core business capability → Ownership reduces risk
- Competitive differentiation → Control enables customization

## Is CPU Inference Production-Ready?

Yes, with appropriate expectations.

The [technical testing](2025-12-17-ZenDNN-ai-enablement.md) demonstrated zero failed requests across all benchmark runs, stable memory utilization, and consistent throughput. The benchmarks include both ZenDNN-optimized and baseline configurations, showing measurable performance improvements from AMD's optimization library.

!!! tip "Version Compatibility"

    The vLLM and ZenTorch ecosystems are evolving rapidly. The technical post includes guidance on version compatibility, currently v0.11.0 is the recommended vLLM version for ZenTorch integration. Running from HEAD works but may produce version warnings. Pin your versions for production deployments.

**What requires attention:**

- Version compatibility between vLLM and ZenTorch needs monitoring
- Larger models (12B+) push memory limits; size your instances appropriately
- Operational expertise for container orchestration and monitoring is required

This is mature enough for production use cases where operational requirements align with organizational capabilities. It's not "deploy and forget", but neither is any production AI infrastructure.

## The Decision

| If you need... | Consider... |
|----------------|-------------|
| Fastest time-to-value | Commercial APIs |
| Lowest cost at small scale | Commercial APIs (GPT-4o-mini, Claude Haiku) |
| Predictable costs at scale | Self-hosted |
| Data sovereignty | Self-hosted |
| Regulatory compliance | Self-hosted |
| Vendor independence | Self-hosted |
| Development/testing environment | Self-hosted CPU |

The answer isn't universal. Commercial APIs are excellent for many use cases, fast, capable, constantly improving.

But the assumption that APIs are always the right choice deserves scrutiny. For organizations where data control, cost predictability, and vendor independence are strategic priorities, CPU inference on your own infrastructure is a legitimate option that the industry has under-discussed.

The 24-core AMD EPYC VM processing 60 tokens per second on a 4B parameter model isn't competing with H100 clusters. It's competing with the default assumption that AI inference requires external dependencies. For a substantial category of workloads, it doesn't.

---

## Getting Started

The [companion technical post](2025-12-17-ZenDNN-ai-enablement.md) covers the complete setup: Docker builds, vLLM configuration, ZenTorch integration, and benchmark methodology. Start there if you want to replicate the testing or deploy CPU inference in your own environment.

For Rackspace OpenStack Flex customers, the `gp.5` flavor family provides AMD EPYC instances ready for this workload. Adjust your KV cache and CPU binding settings based on instance size, and you're running inference within the hour.

---

*API pricing reflects published rates as of December 2025. Pricing changes frequently; verify current rates before making infrastructure decisions.*
