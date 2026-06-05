---
layout: distill
title: "Putting the NVIDIA DGX Station to Work at Cornell"
date: 2026-05-28
description: "Lessons learned from putting an NVIDIA DGX Station to work on RL fine-tuning of LLMs, diffusion language modeling, and large-scale synthetic data generation."
tags: nvidia dgx-station llm reinforcement-learning diffusion synthetic-data
categories: research
featured: false
giscus_comments: false
related_posts: false
thumbnail: assets/img/posts/dgx-station/dgx-station.jpg

authors:
  - name: Anmol Kabra
    url: "https://anmolkabra.com/"
    affiliations:
      name: Cornell University
  - name: Paul Jünger
    url: "https://www.linkedin.com/in/pauljuenger"
    affiliations:
      name: Cornell University
  - name: Christian Belardi
    url: "https://christianbelardi.github.io/"
    affiliations:
      name: Cornell University

bibliography: 2026-05-28-nvidia-dgx-station.bib

toc:
  - name: Scaling RL Fine-tuning to Larger LLMs
  - name: Accelerating Iteration on Diffusion-LM Retrieval
  - name: Generating Synthetic Data for Language Model Development
  - name: Takeaways
---

Our ML group at Cornell University recently received early access to an [NVIDIA DGX Station](https://build.nvidia.com/station)™. It is powered by the latest NVIDIA GB300 Grace Blackwell Ultra Desktop Superchip with 748GB of coherent memory. It brings data-center-class compute into a deskside form factor designed for an office or computing lab. We used the DGX Station remotely, and we are thrilled to get this early access!

{% include figure.liquid loading="eager" path="assets/img/posts/dgx-station/dgx-station.jpg" class="img-fluid rounded z-depth-1" %}
<div class="caption">
    NVIDIA DGX Station.
</div>


Through the course of February, three of us in Professor [Kilian Weinberger](https://www.cs.cornell.edu/~kilian/)'s group put the machine to test on very different research workloads: RL fine-tuning LLMs, diffusion language modeling, and large-scale synthetic data generation. In each case, the DGX Station accelerated our research tasks beyond what our shared university cluster could provide. In this post, we share concrete benchmarks, lessons learned, and how the machine has changed the way we iterate on ideas.

## Scaling RL Fine-tuning to Larger LLMs

**[Anmol Kabra](https://anmolkabra.com/)**

LLMs have made rapid progress on math and coding reasoning tasks, but underneath every kind of reasoning lies a more fundamental skill: composing knowledge across multiple pieces of evidence to reach an answer.

In our work published at ICLR 2026 <d-cite key="kabra2026learning"></d-cite>, we target that fundamental skill directly. We post-trained LLMs with reinforcement learning (RL), specifically the GRPO algorithm following DeepSeek-R1. Our paper showed that LLMs can in fact be taught to compose knowledge. The paper focused on full fine-tuning of smaller LMs (<4B parameters). To push the method further, we needed to **scale to 4–7B parameter models**.

I had developed the [project](https://github.com/kilian-group/phantom-reasoning) with [Huggingface's TRL library](https://huggingface.co/docs/trl/en/index) on a cluster of 4×H100 GPU nodes, where scaling to larger LMs was challenging. RL fine-tuning with GRPO is extremely memory-intensive: you need space for the trainable parameters, the rollout buffer, and optimizer states. For example, on the 4×H100 nodes, I could only RL fine-tune the Qwen3-4B model with batch size 8 and 4 rollouts, before hitting CUDA out-of-memory errors. This was even with aggressive DeepSpeed CPU offloading. However, these small batch and rollout size configurations were suboptimal for effective GRPO fine-tuning. To effectively fine-tune 4B+ LLMs, I needed larger batches and more rollouts, and the 4×H100 GPU node cluster did not have the resources I required.

The DGX Station lifted that resource ceiling. With the large coherent memory on a single device, I could comfortably run batch size 32 with 16 rollouts — a 4× increase in both batch size and rollouts. Training runs that previously crashed or required aggressive memory workarounds now ran cleanly to completion. Parameter scaling no longer felt daunting; I was quickly onto full-fine-tuning 7B parameter models, confirming that our findings held as model size grew.

While the machine was enabling larger model training, I wanted to get a rough sense of training efficiency on the DGX Station compared to the 4×H100 GPU setting. In the following figures, I compare GPU hours and wall clock time estimates of our RL fine-tuning experiments at different model sizes (lower is better for both).

<div style="width: 80%; margin: 0 auto;">
{% include figure.liquid loading="eager" path="assets/img/posts/dgx-station/fig-000.png" class="img-fluid rounded z-depth-1" %}

{% include figure.liquid loading="eager" path="assets/img/posts/dgx-station/fig-001.png" class="img-fluid rounded z-depth-1" %}
</div>
<div class="caption">
    Note: Qwen3-4B on 4×H100 used a small training batch and rollout size due to memory constraints. Hence the lower wall clock time than on DGX Station.
</div>

Two findings stood out for me. First, the DGX Station used substantially fewer GPU-hours overall, roughly **3× fewer for the Qwen3-0.6B model**. This is because the multi-GPU setup on H100s incurred communication overhead across the four devices, while the DGX Station trained all model parameters in one coherent memory. Second, the DGX Station delivered competitive raw wall clock time against the multi-GPU system, despite being a single device. Note that the 4×H100 wall-clock time for Qwen3-4B looks better only because I trained it with a small batch and rollout size — it isn't an apples-to-apples comparison with the DGX Station run.

### Fast Iteration on Agent Training

Following our published work, I experimented with a substantially harder agentic reasoning setting. Instead of producing an answer in one shot, the LLM is trained as a multi-turn search agent that issues queries, reads retrieved passages, and chains evidence across many turns before answering.

RL fine-tuning is demanding because each gradient step requires a large number of rollouts. In the in-context reasoning setting from our published work, a rollout is just a single forward pass, and the prompt-to-gradient latency is low with standard optimizations like flash attention and vLLM. Agent training breaks this. A rollout is now a sequence of interleaved generation and environment interaction turns, and we still need many rollouts per prompt. The prompt-to-gradient latency balloons. Fast environment interaction is the lever, and it helps a lot when generation and the environment live on the same compute device.

Using the [SkyRL library](https://docs.skyrl.ai/docs) for agent training and the DGX Station's coherent memory, I was able to significantly reduce the prompt-to-gradient latency by co-locating the policy, the retriever, and the rollout buffer on the device. Rollout generation was fast enough that I had the full pipeline — multiple base models, environment variants, and evaluation benchmarks — working end-to-end within a week of project kickoff. That headroom let us test moonshot ideas cheaply, ruling out bad hypotheses in a day of training rather than a week. Once the recipe stabilized, we transitioned the longer scaled runs to our B200 university cluster, but the deskside DGX Station carried the iteration-heavy early phase of this work.

## Rethinking RAG with Diffusion Language Models

**[Paul Jünger](https://www.linkedin.com/in/pauljuenger)**

Many retrieval-augmented generation (RAG) systems retrieve documents once from the input and keep them fixed while generating a response. But this assumes the question already contains everything needed to find the right evidence, which often isn't true for multi-step reasoning.

In our work published at ICML 2026 <d-cite key="juenger2026self"></d-cite>, we explore a different idea: using the intermediate states of a diffusion language model to refine retrieval as the answer takes shape. Diffusion models generate text by iteratively denoising a masked sequence, and these partially formed responses often surface key entities early. By turning intermediate hypotheses into retrieval queries, the model can fetch new evidence exactly when it becomes useful, rather than relying on a single static search.

Retrieval is also structurally well-suited to diffusion models. In RAG, much of the response is grounded directly in retrieved documents, copying or paraphrasing directly from context. Once strong evidence is available, many output tokens become conditionally independent given that evidence, making them naturally amenable to parallel decoding. In other words, RAG reduces the need for strict left-to-right coordination: precisely where diffusion models shine.

{% include figure.liquid loading="eager" path="assets/img/posts/dgx-station/sardi-main.png" class="img-fluid rounded" %}
<div class="caption" style="margin-top: -1rem; text-align: left; line-height: 1.3;">
    Overview of Self-Augmenting Retrieval for Diffusion Language Models. At step $t$, the diffusion language model first denoises the partially masked response. Tokens with confidence $c_i \ge \tau_q$ then form a query to refresh the retrieved evidence, while only the more confident tokens ($c_i \ge \tau_c$) are committed to the next response $x^{t-1}$. Speculative tokens can thus inform retrieval before they are stable enough to commit.
</div>

Access to NVIDIA DGX Station with a GB300 Superchip was a key enabler for this work. Compared to our previous setup on a shared university cluster with a B200 GPU, deskside access to large-memory, high-performance compute let us iterate immediately instead of waiting in queue. The GB300's large memory enabled us to double our training batch size while cutting epoch time by ~20%. Even more importantly, it gave us full control to profile and optimize our pipeline with [NVIDIA Nsight](https://developer.nvidia.com/nsight-systems), without worrying about workload interference or privilege issues common on shared infrastructure.

Overall, this work suggests a broader perspective: diffusion language models aren't just an alternative decoding strategy. Their denoising structure opens up new ways to integrate reasoning, retrieval, and uncertainty. And having powerful, on-demand compute locally has made it possible to explore these ideas at a much faster pace.

## Generating Synthetic Data for Language Model Development

**[Christian Belardi](https://christianbelardi.github.io/)**

Synthetic data has become a critical component of language model development. Recent open efforts have dedicated significant computation toward generating synthetic corpora to extend real ones. Interested in exploring synthetic data in our own model development, we used our DGX Station equipped with a GB300 Superchip to generate billions of tokens for our experiments, and took the opportunity to compare its generation performance against other NVIDIA hardware.

The pipeline generates synthetic documents from web-scale data using Qwen3-30B-A3B-Instruct, a mixture-of-experts model we run in BF16 with CUDA 12.8. Using vLLM to optimize inference, we feed the model an instruction along with a source document of up to 2,048 tokens; from this input it generates a new document of similar length. The comparisons below come from running this pipeline on a fixed set of 1,000 documents randomly sampled from FineWeb-Edu.

{% include figure.liquid loading="eager" path="assets/img/posts/dgx-station/fig-002.png" class="img-fluid rounded z-depth-1" %}

We tested five configurations: 1×A100, 4×A100, 1×H100, 4×H100, and the GB300. The figure below shows total inference throughput, measured in tokens per second, for each configuration. The GB300 delivered 5.7× the throughput of a single A100 and 2.6× that of the 4×A100 setup.

{% include figure.liquid loading="eager" path="assets/img/posts/dgx-station/fig-003.png" class="img-fluid rounded z-depth-1" %}

The GB300 really stands out in terms of power efficiency. It is over 2.3× more efficient than a single A100, and well ahead of the next best configuration, 4×H100s. For workloads like ours that involve sustained generation over billions of tokens, this kind of efficiency gap translates directly into lower energy costs and more feasible long-running jobs.

Throughput is measured as total tokens divided by the wall-clock time of the generation run, so it reflects end-to-end throughput rather than per-request decode speed. Power is measured as per-GPU board power via NVML (`nvmlDeviceGetPowerUsage`), sampled
at a fixed interval throughout generation.

## Takeaways

Three researchers, three very different workloads — RL fine-tuning, diffusion language modeling, and large-scale inference. In all three cases, the DGX Station delivered capabilities that our existing multi-GPU cluster couldn't match. The common thread is: 252 GB of HBM3e on a single GPU and 748GB of coherent memory eliminates an entire class of engineering workarounds and lets us focus on the research itself.

A few practical observations for anyone considering the DGX Station:

- **Large memory eliminates overhead.** The ability to fit large models, large batches, and full optimizer states in a single memory space simplifies codebases and eliminates distributed training overhead.
- **Hardware is pushing software to play catch up.** The DGX Station features a GB300 superchip pairing a 72-Core Grace CPU, Blackwell Ultra GPU, and a newer software stack built around [NVIDIA CUDA](https://developer.nvidia.com/cuda) 13. That combination is ahead of the x86/CUDA 12 environment many research codebases still utilize. Even vLLM and Huggingface's TRL libraries are just shipping out compiled wheels as we write this post. Expect some time investment in getting packages built and configured — the NVIDIA performance-optimized and pre-built NGC Docker containers help a lot here. Collectively, we three students spent many days in February configuring the software. The time investment has been worth it.
- **Profiling identifies bottlenecks in research software.** The NVIDIA Nsight Systems profiler helped us identify and fix performance bottlenecks. If you have access to the DGX Station, invest time in profiling early.
- **Deskside access changes the workflow.** Not having to queue for shared cluster time means faster iteration cycles, especially for exploratory research.

We are grateful to NVIDIA for early access and to the NVIDIA team for their hands-on support throughout the process. We're continuing to push the machine with larger models and more complex training pipelines, and we're excited to see what the NVIDIA Blackwell GPU ecosystem enables next.
