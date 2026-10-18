# GPU Inference Hosting: How to Size VRAM, Compare Real Prices, and Pick a Dedicated GPU for Your LLM Endpoint

Serving your own model comes down to one decision before anything else: how you pay for GPU time. Per-token serverless APIs work for light, unpredictable traffic. Per-second serverless works for spiky workloads. And for an endpoint that has to answer 24/7, an hourly-billed cloud GPU with a monthly cap — or a flat monthly dedicated server — is almost always the cheapest option over time.

This article walks through the actual math, the VRAM you really need, and where Vultr's Cloud GPU plans fit into that picture.

> **Quick pick:** For a 7B–13B model running around the clock, Vultr's A16 cloud GPU lists at **$0.24/hr** with a monthly cap of roughly **$175** — the cheapest always-on entry among the providers compared below. For 70B-class serving, look at the A100 80 GB ($1.19/hr) or H100 ($2.99/hr). 👉 [Check Vultr's current Cloud GPU pricing](https://www.vultr.com/products/cloud-gpu/?ref=7268283)

## The three pricing models behind GPU inference hosting

GPU inference hosting is sold in three shapes, and picking the wrong one is the most common way to overpay.

**Per-token / per-request serverless.** Providers like Together AI bill by token on hosted models — Llama 3.3 70B at $0.88 per 1M tokens, for example. Replicate bills image and video models per output unit ($0.04 per Flux 1.1 Pro image, $0.09 per second of 480p Wan 2.1 video). No infrastructure to manage, but rate limits apply and the meter never stops.

**Per-second or hourly GPU rental.** Modal bills GPU time by the second and scales to zero when idle, with cold starts of 1–5 seconds. Hugging Face Inference Endpoints sell dedicated servers by the hour (with scale-to-zero cold starts of 30–90 seconds on most tiers). Vultr's cloud GPU line follows the hourly model with a monthly cap.

**Flat monthly dedicated.** GPUCloudHQ rents whole bare-metal servers by the month (an L40S at $429/mo, an H200 at $1,690/mo). GPU Mart starts its RTX Pro line at $179/mo — though that figure is a monthly-equivalent price based on a 24-month billing term.

One managed-GPU provider, ZenoCloud, puts the crossover point between shared APIs and dedicated inference at roughly **150,000–300,000 requests per month**. Below that, serverless usually wins. Above it, you're feeding a meter that a fixed monthly cost would beat — and you inherit rate limits and shared-infrastructure latency on top.

## Vultr's GPU plans and what they actually cost

Vultr sells dedicated NVIDIA GPUs attached to virtual cloud instances, billed hourly with a monthly cap: a short-lived test pays by the hour, while an always-on endpoint settles at roughly the cap. Rates below are Vultr's listed prices at the time of writing; stock and exact rates vary by region, so the order page is the source of truth.

| Plan | GPU & VRAM | Listed rate | Best for | Link |
| --- | --- | --- | --- | --- |
| Cloud GPU – A16 | NVIDIA A16, 64 GB (4× 16 GB), Ampere | $0.24/hr, ≈ $175/mo cap | 7B–13B models, embeddings, Whisper, VDI | [Check the A16 plan](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |
| Cloud GPU – A100 | NVIDIA A100, 80 GB | $1.19/hr, ≈ $869/mo cap | 30B-class at FP8, 70B at Q4 | [Check the A100 plan](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |
| Cloud GPU – H100 | NVIDIA H100, 80 GB | $2.99/hr, ≈ $2,183/mo cap | 70B FP8, high concurrency, long context | [Check the H100 plan](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |

The A16 deserves a closer look, because most spec sheets hide the catch: its "64 GB" is actually **four 16 GB GPU dies on one board**, not one contiguous 64 GB pool. That's fine for a 7B FP16 model (~14 GB fits a single die, and small models parallel across dies well), quantized 13B, embeddings, rerankers, and Whisper. It's the wrong tool for a 70B model at anything above 4-bit. If your plan is "one big model, lots of context," start at the A100.

Two practical notes. First, GPU availability is region-dependent — check your target data center's stock before committing to a deployment plan. Second, if you specifically need a non-virtualized machine, Vultr's separate bare metal line covers that at higher fixed rates.

## How much VRAM your model actually needs

VRAM is the binding constraint; everything else is speed. If the model doesn't fit in GPU memory, the runtime spills to system RAM and throughput collapses. Here's the sizing math, using 2 bytes per parameter at FP16, 1 byte at FP8, and roughly 0.5 bytes at 4-bit:

| Model size | Precision | Weights in VRAM | Minimum GPU |
| --- | --- | --- | --- |
| 7B | FP16 | ~14 GB | 24 GB |
| 7B | Q4 | ~4 GB | 8 GB |
| 13B | FP16 | ~26 GB | 32 GB |
| 70B | Q4 | ~38 GB | 48 GB |
| 70B | FP8 | ~70 GB | 80 GB |
| 70B | FP16 | ~140 GB | 141 GB |

Weights are only half the story, though. The KV cache scales with context length times concurrency, and at 16-bit a Llama-70B setup burns about 0.3 MB per token — a 128k-token context needs around 42 GB on its own. Run the addition before you pick a card: a 70B at FP8 needs ~70 GB for weights, which means weights plus a serious KV cache blow past a single 80 GB GPU. Your realistic options are 4-bit weights on an 80 GB card, multi-GPU, or a 141 GB card like the H200.

For throughput expectations, one managed provider's published benchmarks are a useful rule of thumb: on an A100 80 GB with vLLM at batch size 8, a 7B FP16 model delivers roughly 2,000–4,000 tokens/second, a 70B FP16 model roughly 400–800 tokens/second, and the H100 roughly doubles the A100 thanks to memory bandwidth. Actual numbers depend on prompt length and concurrency.

Mapping that to the plans above: 7B–13B → A16. 30B at FP8 (~30 GB) or 70B at Q4 (~38 GB) with moderate context → A100 80 GB. 70B at FP8 with high concurrency, or anything latency-sensitive at scale → H100.

## GPU inference hosting prices, compared

On a pure hourly-rate basis, Vultr's list prices undercut the dedicated-endpoint rates published by the big managed platforms in spring 2026:

| Provider | GPU | Published rate | Billing model |
| --- | --- | --- | --- |
| Vultr | A100 80 GB | $1.19/hr | hourly, monthly cap |
| Modal | A100 80 GB | $2.50/hr | per-second |
| Hugging Face | A100 80 GB | $2.50/hr | per-hour endpoint |
| Replicate | A100 80 GB | $5.04/hr | per-second |
| Vultr | H100 80 GB | $2.99/hr | hourly, monthly cap |
| Modal | H100 | $3.95/hr | per-second |
| Together AI | H100 (dedicated) | $3.99/hr | hourly |
| Hugging Face | H100 | $4.50/hr | per-hour endpoint |
| Replicate | H100 | $5.49/hr | per-second |

The always-on math is where this matters. A Hugging Face A100 endpoint running 24/7 costs $2.50 × 730 hours ≈ **$1,825/month**. Vultr's A100 caps out around **$869**. Same GPU class, under half the bill, for a workload that never sleeps. On the H100 side, Vultr's $2.99/hr sits about 24% below the next-best published rate.

Flat-monthly specialists are worth checking too, especially if you want bare metal. GPUCloudHQ's ladder runs from an L4 at $169/mo (fine for 7B–8B models and embeddings) through an L40S at $429 and an RTX PRO 6000 at $749, up to an H200 at $1,690 — with 20 TB of monthly transfer included and no per-token charges. GPU Mart's Blackwell RTX Pro line starts at $179/mo, but read the fine print: that's pricing on a 24-month term, and shorter terms cost more per month. If your traffic is steady, an hourly-with-cap plan and a flat monthly plan land in similar territory; if your traffic is genuinely bursty with long idle stretches, per-second serverless with scale-to-zero remains the cheaper route. 👉 [Compare Vultr's A100 and H100 rates for your region](https://www.vultr.com/pricing/?ref=7268283)

## Setting up an inference server: the short version

From order to first API call takes about 15–30 minutes on a typical GPU VPS walkthrough. The path looks like this:

1. Provision the GPU instance with an Ubuntu 22.04 or 24.04 image. Run `nvidia-smi` to confirm the driver loaded — 535.x or higher on pre-Blackwell GPUs (A16, A100), 570.x or higher on Blackwell cards, which require the open GPU kernel module.
2. Pick a runtime. **vLLM** is the production default once you exceed roughly 5 concurrent requests — it uses PagedAttention for KV cache efficiency and continuous batching for throughput, and it exposes an OpenAI-compatible API you can point existing code at. **Ollama** installs in one command and is the faster route for single-user or dev setups. For vision, speech, or ONNX models, NVIDIA Triton is the right tool instead.
3. Load your model and budget the disk: a 7B model in FP16 is about 14 GB on disk, a 70B at Q4 around 40 GB. The initial load takes under 30 seconds for a 7B model and 60–120 seconds for a 70B; after that, vLLM stays warm with no per-request cold start.
4. Point your application at the endpoint. If you've been calling a managed API, swapping the base URL to your own server usually requires no code changes beyond configuration.

## When Vultr is the wrong pick

Honest limits, because they'll save some of you money:

- **Your traffic is spiky and idle most of the day.** Per-second billing with scale-to-zero (Modal-style, cold starts of 1–5 seconds) will beat a capped monthly bill. A dedicated GPU running empty at 3 a.m. is only "free" if your rate caps out anyway.
- **You want a flat, predictable invoice and single-tenant bare metal.** The flat-monthly providers do that natively; Vultr's cloud GPU line is virtualized and hourly-based (capped, but still metered within the month).
- **You need more than 80 GB on a single card.** A 70B at FP16 wants 141 GB. That means an H200 — $1,690/mo flat at GPUCloudHQ, or $5.00/hr on Hugging Face — rather than anything in Vultr's current cloud GPU lineup.
- **You want fully managed operations, monitoring dashboards, and compliance paperwork.** Managed serving providers handle runtime configuration, latency monitoring, and data processing agreements as the product.

One more point in favor of datacenter GPUs generally: consumer cards like the RTX 4090 lack ECC memory, which matters for a model serving 24/7 for weeks — undetected VRAM errors quietly corrupt outputs. Every option in Vultr's lineup is a datacenter part, so that failure mode is handled.

## FAQ

**What's the cheapest way to serve a 7B–8B model around the clock?**
A flat L4 at $169/mo or Vultr's A16 at roughly a $175/mo cap are the two cheapest always-on options among the providers compared here. Both beat most per-token bills once traffic is steady, and neither charges per request.

**Does hourly billing end up costing more than a monthly plan?**
With a monthly cap, no. A GPU that runs 24/7 settles at approximately the cap; you only save with hourly billing if the instance is actually off part of the month.

**How much VRAM does a 70B model need?**
About 38 GB at 4-bit (fits a 48 GB card), about 70 GB at FP8 (needs 80 GB plus KV cache headroom — tight), and about 140 GB at FP16 (needs an H200 or multi-GPU). Always add KV cache on top: ~42 GB for a 128k context on a 70B at 16-bit.

**Can I run several models on one GPU?**
Yes, as long as combined VRAM fits. Multiple vLLM instances or Triton with multiple backends both work; memory is the only real limit.

## Before you order, check two things

First, confirm GPU stock in your target region — availability differs by data center, and the deploy page shows what's actually live. Second, size your disk for model weights plus a working buffer, and skim the bandwidth terms on the order page so egress doesn't surprise you later.

If you've made it this far, the decision usually collapses to a single line: 7B–13B steady traffic → A16; 30B FP8 or 70B Q4 → A100 80 GB; 70B FP8 at high concurrency → H100; long idle periods → per-second serverless instead. For the first three cases, 👉 [open Vultr's Cloud GPU plans](https://www.vultr.com/products/cloud-gpu/?ref=7268283), pick your region, and you can be at a root prompt loading weights tonight.
