---
date: 2026-01-16
title: So You Need Enterprise GPUs - A No-BS Guide to H100, H200, and B200
authors:
  - bet0x
description: >
  Confused about which NVIDIA GPU to pick for your AI workloads? H100 SXM vs NVL, H200, B200... the naming is a mess. Here's what actually matters when you're spending serious money on compute.
categories:
  - GPU
  - Infrastructure
  - Hardware

---

# So You Need Enterprise GPUs: A No-BS Guide to H100, H200, and B200

Let's be honest—NVIDIA's naming conventions are designed to confuse procurement teams. H100 SXM5, H100 NVL, H200 SXM, B200... it sounds like someone spilled alphabet soup on a product roadmap.

I've spent way too many hours explaining these differences to engineering teams, so here's everything you actually need to know before signing that hardware purchase order.

<!-- more -->

## The TL;DR Table

Before we dive in, here's the cheat sheet:

| Spec | H100 SXM | H100 NVL | H200 SXM | B200 |
|------|----------|----------|----------|------|
| **Memory** | 80GB HBM3 | 94GB × 2 (188GB) | 141GB HBM3e | 192GB HBM3e |
| **Bandwidth** | 3.35 TB/s | 3.9 TB/s × 2 | 4.8 TB/s | 8.0 TB/s |
| **NVLink** | 900 GB/s | 600 GB/s | 900 GB/s | 1.8 TB/s |
| **TDP** | 700W | 400W × 2 | 700W | 1000W |
| **Sweet Spot** | Training | Inference | Both | Next-gen training |

Now let's talk about *why* these numbers matter.

## H100: Still the Workhorse

The H100 has been the backbone of AI infrastructure since 2023. Two variants, completely different use cases.

### H100 SXM5 - The Training Beast

This is what goes into DGX systems and large-scale training clusters. The SXM form factor means it slots directly into specialized motherboards—no PCIe bus bottlenecks.

**Why it matters:**

- 900 GB/s NVLink bandwidth lets you scale to 8 GPUs without choking on data transfer
- The 700W TDP is manageable in purpose-built data centers
- 80GB is tight for modern LLMs, but fine for training with gradient checkpointing

The SXM5 assumes you're building dedicated AI infrastructure. If you're thinking "we'll just add it to our existing servers," stop. This isn't that kind of hardware.

### H100 NVL - The Inference Play

Here's where things get interesting. The NVL is two H100s on a single board, connected via NVLink bridge. You get 188GB of combined memory in a PCIe form factor.

**The real story:**

That 188GB memory pool is the selling point. A 70B parameter model in FP16 needs ~140GB just for weights. Single H100 SXM? Won't fit. H100 NVL? No problem.

But here's what the spec sheets don't tell you: the NVLink bridge between the two dies gives you 600 GB/s, not the 900 GB/s you get with SXM. For inference, this rarely matters. For training, it's a bottleneck.

**When to pick NVL:**

- Running inference on 70B+ models
- Your facility isn't built for 10KW+ racks
- You want to deploy in standard PCIe servers

## H200: The Memory Upgrade

The H200 is basically "what if we put better memory on an H100?" Same Hopper architecture, but with HBM3e instead of HBM3.

**What changed:**

- 141GB vs 80GB memory
- 4.8 TB/s vs 3.35 TB/s bandwidth
- Everything else? Virtually identical

That 76% memory increase is significant. Models that needed 2× H100s now fit on a single H200. The bandwidth bump helps with memory-bound workloads—which is basically everything in inference.

**The catch:**

H200 availability has been... challenging. If you can actually get them, they're a meaningful upgrade. But don't hold up your project waiting.

## B200: Welcome to Blackwell

Blackwell is a generational leap, not an incremental upgrade. The B200 rewrites what's possible.

**The headline numbers:**

- 192GB HBM3e
- 8.0 TB/s memory bandwidth (2.4× the H100)
- 1.8 TB/s NVLink 5 (2× the H100)
- FP4 tensor support (new)

Fifth-generation tensor cores with dual transformer engines. In MLPerf benchmarks, B200 hits roughly 2× the performance of H200 on LLM training. That's not marketing fluff—it's measured.

**The infrastructure problem:**

B200 draws 1000W per GPU. An 8-GPU node needs 8KW+ just for compute, before you count networking, storage, and cooling. Your existing data center probably can't handle it without upgrades.

Also: CUDA 12.4+ required. If you're running older software stacks, expect a migration project.

## The Memory Bandwidth Story

Here's the thing nobody talks about enough: memory bandwidth is often the real bottleneck, not compute.

```
H100 SXM:  3.35 TB/s
H100 NVL:  3.9 TB/s (per die)
H200:      4.8 TB/s
B200:      8.0 TB/s
```

During inference, you're constantly shuffling weights and KV cache in and out of HBM. More bandwidth = more tokens per second, period. The B200's 8 TB/s is why it demolishes everything else in inference benchmarks despite "only" having 2× the raw compute.

## NVLink: When Interconnect Matters

NVLink bandwidth determines how well multi-GPU setups scale:

```
H100/H200: 900 GB/s (NVLink 4)
B200:      1.8 TB/s (NVLink 5)
```

For single-GPU inference, this is irrelevant. For 8-GPU training runs, it's everything.

Tensor parallelism means shipping activation tensors between GPUs every single layer. Slow interconnect = GPUs waiting on data = expensive hardware sitting idle.

The jump from 900 GB/s to 1.8 TB/s isn't just "better"—it enables training paradigms that were impractical before.

## What Should You Actually Buy?

Cut through the noise:

**"We're deploying inference for a 70B model"**
→ H100 NVL or wait for H200 availability. The memory is what matters.

**"We're training models up to 30B parameters"**
→ H100 SXM is still cost-effective if you can get good pricing. 8-GPU nodes work well.

**"We're training 100B+ parameter models"**
→ H200 if available, B200 if you can wait and your facility supports the power draw.

**"We need the absolute best performance and budget isn't the constraint"**
→ B200. Not close.

**"We're a startup figuring things out"**
→ Rent H100 SXM time from cloud providers. Don't buy hardware until you know your workload.

## The Hidden Costs

Everyone focuses on GPU price. The real costs:

1. **Power infrastructure**: B200 nodes need serious electrical work. Budget $50-100K per rack for upgrades.

2. **Cooling**: 8KW+ of heat per node. Traditional air cooling won't cut it for dense deployments.

3. **Networking**: You need 400GbE minimum between nodes for distributed training. That's $10K+ per port.

4. **Software stack**: Blackwell needs CUDA 12.4+. Older frameworks may need updates.

5. **Lead times**: Enterprise GPU lead times are still measured in months, not weeks.

## Final Thoughts

The GPU landscape moves fast. By the time you read this, there might be new variants I haven't covered. But the fundamentals stay the same:

- Memory size determines what models fit
- Memory bandwidth determines inference speed
- NVLink bandwidth determines training scale
- TDP determines infrastructure requirements

Match your workload to these constraints, not to marketing materials.

And if someone tells you "just get the newest thing"—ask them if they've actually priced out the infrastructure upgrades. Hardware is the easy part. Everything around it is where projects die.

---

*Have questions about GPU selection for your specific workload? That's literally what we do at Rackspace Private Cloud AI. [Reach out](https://www.rackspace.com/cloud/private/ai) and let's talk specifics.*
