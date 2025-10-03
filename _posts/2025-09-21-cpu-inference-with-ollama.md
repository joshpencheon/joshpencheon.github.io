---
layout: post
title: CPU inference with Ollama
tags: ai ruby
---

I've been using [Ollama](https://ollama.com/) for the past year or so to experiment with open-source large language models, and have most recently been trying out OpenAI's [gpt-oss](https://ollama.com/library/gpt-oss) 20 billion parameter model.

## The hardware

Not having access to a GPU from the last decade, I've instead been running these models via Ollama's CPU-only support. It has been interesting to see how recent library improvements have made `gpt-oss` much more useable when operating in this way.

Here's the hardware I'm using:

```
$ neofetch kernel cpu memory
kernel: 6.16.4-arch1-1
cpu: Intel Xeon E5-2698 v4 (40) @ 3.600GHz
memory: 15181MiB / 64208MiB
```

This is a Broadwell-EP CPU from early 2016, paired with quad-channel DDR4-2400. Memory bandwidth is king for LLM inference, and this hardware has a *theoretical* maximum of  a little under 80GB/s.

## The test

I used the following prompt against the `lib/` source of [a small Ruby gem](https://github.com/joshpencheon/hobble), which resulted in 1.01k input tokens being fed to the model:

```bash
MODEL="gpt-oss:20b"
GEM_CODE=$(
  git ls-files -- lib \
  | xargs -I {} bash -c 'echo -e "{} contains:\n$(cat {})"'
)
echo -e "offer a refactoring suggestion for this ruby gem:\n\n$(GEM_CODE)" \
| ollama run $MODEL --verbose -
```

## The results

A combination of Ollama enabling flash attention for CPU-only prompt processing, as well as efficiency improvements in the handling of the way `gpt-oss` model weights are stored resulted in a big performance improvement between 0.11.4 and 0.11.8:

<div class="overflow-auto" markdown="1">

  | Model              | Ollama Version | Prompt Eval Rate (tokens/s) | Generation Rate (tokens/s) |
  |--------------------|----------------|-----------------------------|----------------------------|
  | gpt-oss:20b        | 0.11.4         | 10.17                       | 6.16                       |
  | gpt-oss:20b        | 0.11.8         | 57.50                       | 10.16                      |
  | qwen3-coder:latest | 0.11.8         | 78.19                       | 13.55                      |
  | qwen3:0.6b         | 0.11.8         | 404.92                      | 42.81                      |

</div>

I've included a couple of extra models in there for comparison, but the `gpt-oss` improvements were ~50% token generation, and 5x prompt evaluation.
