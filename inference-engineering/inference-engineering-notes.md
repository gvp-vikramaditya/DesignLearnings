# Inference engineering: simple podcast notes

Inference engineering means making a trained AI model respond faster, serve more
users, and cost less to run while maintaining acceptable answer quality.

These notes cover the concepts discussed in the podcast and the follow-up
explanation of NVMe. Model and hardware specifications were checked on
2026-10-15. The podcast itself was not provided, so its exact model reference
has not been confirmed.

## 1. What happens when a model answers?

A **token** is a small piece of text: a word, part of a word, or punctuation.

| Stage | What happens |
| --- | --- |
| **Prefill** | The model processes the input prompt. It can process many input tokens in parallel. |
| **Decode** | The model generates the answer, normally one token at a time. Each new token depends on the previous ones. |

Three useful performance terms:

- **Time to first token:** How long you wait before the answer starts appearing, including queueing and prompt processing.
- **Time between tokens:** How quickly the answer continues appearing.
- **Throughput:** How many tokens the system produces per second across all users.

A system can serve more users overall without making each individual answer
faster.

## 2. Optimization techniques

| Technique | Simple explanation | Benefit | Main trade-off |
| --- | --- | --- | --- |
| **Speculative decoding** | A small draft model, or a built-in prediction component, proposes several upcoming tokens. The main model checks them together. | Can generate multiple accepted tokens per main-model verification step. | Rejected guesses waste work; speedup depends on how often guesses are accepted and the cost of drafting and verification. |
| **Quantization** | Store and sometimes compute with model numbers using fewer bits, such as 8-bit or 4-bit instead of 16-bit. | Uses less memory and moves less data; can also speed up computation. | It is lossy: rounding changes the numbers and may reduce answer quality. |
| **KV cache** | Keep intermediate attention information for earlier tokens instead of recomputing it repeatedly. KV means keys and values. | Makes generating subsequent tokens much cheaper. | The cache consumes memory and grows with context length and concurrent requests. |

Standard, correctly implemented speculative decoding can preserve the main
model's output distribution. It does not inherently sacrifice quality like
lossy quantization can.

The KV cache is **not a cache of complete answers**. Think of it as retaining
useful working notes about the text the model has already processed.

## 3. The model: GLM-5.2

GLM-5.2 is a Z.ai model. It may be the model mentioned in the podcast, but that
cannot be confirmed without the recording or a link.

| Property | Meaning |
| --- | --- |
| **744 billion total parameters** | The model contains approximately 744 billion learned numbers, commonly called weights. |
| **40 billion active parameters per token** | Only a subset participates in processing each token. |
| **Mixture of Experts (MoE)** | The model routes tokens through selected specialist components rather than using every expert each time. |
| **About 1 million tokens of context** | It supports very long inputs and conversations, although using that capacity requires substantial resources. |

**40 billion active parameters does not mean you only need to store 40 billion
parameters.** Different tokens use different experts. The full set of weights
still needs to be available, typically spread across GPUs or other memory.

GLM-5.2 also includes improvements to its **multi-token prediction (MTP)**
component for speculative decoding.

Sources: [Z.ai model card](https://huggingface.co/zai-org/GLM-5.2) and
[NVIDIA model specifications](https://catalog.ngc.nvidia.com/orgs/nim/teams/zai-org/models/glm-52).

### How much memory do the weights need?

For a 744-billion-parameter model, these are rough **weight-only estimates**,
not actual deployment requirements. GB here uses decimal units.

| Weight precision | Approximate storage |
| --- | ---: |
| 16-bit: 2 bytes per parameter | 1,488 GB |
| 8-bit: 1 byte per parameter | 744 GB |
| 4-bit: half a byte per parameter | 372 GB |

Actual memory requirements also include KV cache, temporary computation
buffers, quantization metadata, and runtime overhead. Some weights may remain
at higher precision.

This explains why keeping a very large model entirely in GPU memory requires
multiple GPUs, even when only some experts are active per token.

## 4. H100 versus B200 GPUs

These are NVIDIA data-center GPUs, not model names. This is a **per-GPU**
comparison using H100 SXM and the B200 configuration in NVIDIA's DGX B200 system.

| Specification | H100 SXM | B200 |
| --- | ---: | ---: |
| Architecture | Hopper | Newer Blackwell |
| GPU memory | 80 GB HBM3 | 180 GB HBM3e |
| Memory bandwidth | 3.35 TB/s | 8 TB/s |
| Peak FP8 Tensor Core compute, dense | About 1,979 TFLOPS | About 4,500 TFLOPS |
| NVLink GPU-to-GPU bandwidth | 900 GB/s | 1,800 GB/s |

B200 per-GPU figures are derived from NVIDIA's eight-GPU DGX B200
specifications. Other product variants can differ. The compute figures above
use dense performance; NVIDIA also publishes higher figures that require
structured sparsity.

- **Memory capacity:** How much the GPU can hold, including weights, KV cache, and working data.
- **Memory bandwidth:** How quickly it can move data between its own GPU memory and processing units.
- **Compute, measured in FLOPS:** How many numerical operations it can perform per second. A TFLOPS is a trillion floating-point operations per second.
- **NVLink bandwidth:** How quickly GPUs can exchange data with one another. This is different from local memory bandwidth.

For this comparison, B200 has approximately 2.25 times the memory, 2.4 times the
memory bandwidth, and 2 times the NVLink bandwidth.

**That does not guarantee an answer arrives 2.4 times faster.** Actual tokens
per second depend on the model, precision, prompt length, batch size, software,
and number of GPUs. Peak compute figures are hardware ceilings, not
chatbot-speed measurements.

Sources: [NVIDIA H100 specifications](https://www.nvidia.com/en-us/data-center/h100.md)
and [NVIDIA DGX B200 specifications](https://www.nvidia.com/en-us/data-center/dgx-b200.md).

## 5. Why batch size matters

**Batching means processing work from multiple requests together.**

Instead of running a separate operation for each of eight users, the GPU can
process their next-token work together. This helps reuse model weights and make
better use of the hardware.

| Smaller batches | Larger batches |
| --- | --- |
| Often better for low per-user latency | Often better for total throughput |
| Less KV-cache memory needed, for comparable request lengths | More KV-cache memory needed, for comparable request lengths |
| May leave GPU capacity unused | Can increase waiting time or slow each generation step |

Modern serving systems often use **continuous batching**: finished requests
leave and new requests join without waiting for every answer in the batch to
finish.

The goal is not the biggest possible batch. It is **the highest throughput that
still meets the desired response time**.

## 6. How parallelism affects latency

**Parallelism means splitting work across GPUs or running multiple pieces of
work simultaneously.**

| Type | Simple explanation | Main purpose |
| --- | --- | --- |
| **Tensor parallelism** | Several GPUs share the calculations within a model layer. | Fit large models and potentially reduce computation time per step. |
| **Pipeline parallelism** | Different GPUs hold different groups of layers. | Fit a large model and overlap work from multiple requests. |
| **Expert parallelism** | Different GPUs hold different MoE experts. | Distribute expert storage and computation. |
| **Data parallelism** | Multiple copies of the model serve different requests. | Increase serving capacity and reduce queues under load. |

More GPUs do not automatically mean lower latency. They also spend time
exchanging data and synchronizing. Beyond a point, communication overhead can
outweigh the extra compute.

Engineers tune batch size and parallelism together: enough parallel work to
use the hardware efficiently, but not so much coordination or queueing that
users wait longer.

## 7. What is NVMe?

**NVMe stands for Non-Volatile Memory Express.** It is a communication protocol
designed for fast SSD storage, usually connected through PCIe.

- **Non-volatile** means stored data survives when power is off.
- **SSD** is the storage device; **NVMe** is how the computer communicates with it.
- Compared with older SATA SSDs, NVMe SSDs typically provide higher speeds and handle more simultaneous requests.

### Where NVMe fits in inference

| Component | What it holds |
| --- | --- |
| **NVMe SSD** | Model weight files saved on disk |
| **CPU RAM** | Data and weights being loaded or managed by the server |
| **GPU memory (HBM)** | Weights, KV cache, and working data used during inference |

A typical loading path is:

```text
NVMe SSD -> CPU RAM -> GPU memory
```

Once loaded, a model normally generates tokens using GPU memory, not by reading
its weights from the SSD for every token. Offloading systems can use CPU memory
or storage when the model does not fit, but moving that data adds overhead.

**NVMe is not NVLink:** NVMe is for accessing storage; NVLink is for high-speed
communication between GPUs.
