---
title: 'Benchmarking Ollama vs LM Studio vs MLX'
description: 'Investigating a laggy experience'
pubDate: 2026-04-18
tags: [ 'ai', 'tech', 'mlx', 'ollama', 'lm studio' ]
draft: false
---

As I'm continuing to deepen my understanding about how AI works, I'm playing around more and more with using local LLMs.
I don't want to always rely on Claude being up or having tokens available. Sometimes I just want privacy. So running my
own model that I can access anytime and have running 24/7 is pretty appealing. In the past I've used Ollama pretty
lightly. It's the first project I was aware of that had an easy UI to interact with a locally hosted model. It was clear
pretty early on that my machines were pretty underpowered to do anything significant, but big improvements in smaller
models and Apple's silicon ecosystem have opened the door to more regular usage. But is Ollama the right product for the
gig? I've had people telling me about LM Studio for a while, and it came up that it uses Apple's native-ish `mlx` for
serving up the models. Ollama
is [moving that way](https://9to5mac.com/2026/03/31/ollama-adopts-mlx-for-faster-ai-performance-on-apple-silicon-macs/),
but for now it's in preview and very limited on models. I've also been interacting with `mlx` directly as I'm playing
around with fine-tuning, so should I just use it directly?

Let's find out.

Tonight I did a short benchmark test on the three options. I'm tapping into a Mac M2 Studio with 64 gigs of
RAM at the [Initial Capacity](https://initialcapacity.io) office in Boulder via Cloudflare Tunnel. We've been playing
around to see what it can really do, but as a coding companion it's pretty laggy with `qwen-coder-next`. Before trying to
figure out the right model, let's check the throughput on a base level. We can get
that assumption out of the way and then focus on the model choice.

The first thing I've done is learn a lot about the different models. What Ollama has vs what Hugging Face has and how
they compare is genuinely complicated. Why are there so many versions on Hugging Face? What's an MLX model vs GGUF? And
what is nvfp4? (MLX models are optimized for Apple's MLX framework, GGUF is GGML Universal Format, nvfp4 is NVIDIA's new
format that does better on their hardware). It was tricky to find a format we could use with all three products because Ollama, in the midst of switching
to MLX, had a regression and MLX doesn't work on the latest versions. So the current Ollama pull default for Qwen3.6 is
GGUF. This is the main difference in the testing - slightly different quantization formats for our test as LM Studio and
`mlx_lm` both use a Hugging Face MLX model. Still worth the comparison, in my opinion.

I ran a benchmark script that did a couple light warm-up calls, then tasked it with 4 different coding prompts 3x each
for a total of 12 measurable calls. All 12 calls hit my 1536-token cap during the reasoning phase - the model never even
got to the final output. Doesn't matter as we still see the tokens per second. And the results were pretty staggering.

## Headline Numbers

| Backend       | Model / Quant                                    | Engine          | tok/s median | tok/s stdev | Total s median | TTFT median |
|---------------|--------------------------------------------------|-----------------|--------------|-------------|----------------|-------------|
| Ollama 0.20.3 | `qwen3.6:latest` / Q4_K_M GGUF                   | llama.cpp/Metal | 33.1         | 1.8         | 40.3 s         | 360 ms      |
| LM Studio     | `mlx-community/qwen3.6-35b-a3b` / MLX 4-bit      | MLX native      | 80.9         | 0.8         | 19.1 s         | 260 ms      |
| mlx_lm.server | `mlx-community/Qwen3.6-35B-A3B-4bit` / MLX 4-bit | MLX native      | 85.0         | 1.0         | 18.1 s         | 157 ms      |

Yep, Ollama is slow - at least on this machine with the current release. 33 tokens per second vs 80+ is a big gap, and
Time To First Token tells the same story: it's more than twice as fast on MLX. Ollama's move to the MLX engine may well close
the gap a bit - and [their internal testing](https://ollama.com/blog/mlx) seems to show a nearly 2x improvement, but for
the stable release today, MLX is the clear choice.

So what does this look like for our team? Two things.

First, I'm going to swap the M2 endpoint from Ollama to `mlx_lm.server`. That's a roughly 2.5x throughput win with zero
code changes. LM Studio's LM Link (their Tailscale-based remote access tool) is worth a look if we want a proper UI
instead of curling an API endpoint, though we would be giving up a few tokens per second.

Second, with the engine question answered, we can focus on the model. Qwen3.6 is the obvious next try - the benchmarks
on agentic coding are strong, and `qwen-coder-next` was half-broken for us anyway. Whether it holds up as a daily driver
is the next thing to figure out.