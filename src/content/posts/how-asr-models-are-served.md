---
title: I Thought ASR Needed Its Own vLLM — Here’s How Speech Models Are Actually Served
pubDatetime: 2026-09-12T00:00:00Z
description: "Inside the ASR inference engine: model execution, batching, streaming state, parallelism, multilingual serving, open-source runtimes, and the breakthroughs that made modern speech AI practical."
tags:
  - blog
  - asr
  - inference
  - voice-ai
---

After I wrote about [running vLLM on a Mac Mini](/posts/vllm-cpu-inference-guide), I thought I finally understood model serving: load the weights, manage the KV cache, continuously batch requests, and generate tokens as fast as the hardware allows.

Then I started looking at voice AI and asked what felt like an obvious question:

> **Where is the vLLM of speech recognition?**

If an automatic speech recognition model turns audio into text, surely its inference engine has the same problems as an LLM engine—with audio frames replacing prompt tokens.

That is only half true.

An ASR engine still loads weights, schedules work, batches requests, and executes kernels. But it is driven by **time**, not an open-ended token sequence. It has to keep up with audio that is still arriving, preserve state for thousands of live streams, batch clips of wildly different durations, and decide how much future audio it can wait for before accuracy gains become unacceptable latency.

This post stays inside that engine. I will mostly ignore codecs, punctuation, and diarization and focus on the interesting part: **how ASR models execute, how they are served today, which open-source projects exist, and what breakthroughs unlocked the current wave of speech products.**

![Animated diagram showing audio chunks entering a deadline-aware scheduler, being batched across ASR engine replicas, and producing partial transcripts](/asr-engine-serving.svg)

_The ASR engine’s real job: keep live chunks moving before their deadlines while packing offline audio tightly enough to keep the accelerators busy._

---

## What Is Inside an ASR Engine?

Strip away the API and the audio cleanup, and a production engine owns four things:

```plaintext
audio frames → scheduler → model executor → decoder → transcript
                    │             │             │
              stream state    GPU kernels    search state
```

### 1. The scheduler

The scheduler does not ask only, “How many requests can I batch?” It asks:

- How many **audio frames** are in this batch?
- Which live chunks have a deadline in the next 50 ms?
- Which requests have compatible shapes, languages, and decoding options?
- Does this chunk belong to an existing stateful stream?
- Can an offline job wait so a live call can run first?

This is the ASR equivalent of vLLM’s request scheduler, but its currency is audio duration and deadlines rather than prompt and output tokens.

### 2. The acoustic encoder

Most modern ASR compute lives here. A Conformer or Transformer encoder consumes a time-frequency representation of the speech and turns thousands of acoustic frames into contextual embeddings. Unlike LLM decode, much of this work is parallel across the time axis. The GPU likes a wide, well-packed batch.

The encoder is also why padding hurts. Put a 2-second clip beside a 30-second clip in a dense batch and the short request may carry 28 seconds of padding through expensive layers. A good engine buckets requests by length or caps **total frames per batch**, not merely the number of requests.

### 3. The decoder

This is where ASR architectures diverge:

- **CTC** predicts frame labels plus blanks, then collapses them into text. Greedy decoding is cheap; beam search and an external language model cost more.
- **RNN-T / transducer** combines encoder output with a small prediction network. It naturally emits partial text and carries state from one audio chunk to the next.
- **Whisper-style encoder-decoder** runs an autoregressive text decoder over encoded audio. This part resembles an LLM and uses a decoder KV cache, but the transcript is far shorter than a long chat context.
- **TDT (Token-and-Duration Transducer)** predicts both a token and how many input frames it covers, allowing inference to skip frames instead of visiting every one.

So there is no universal ASR execution loop. A CTC model wants fast encoder batches. A streaming transducer wants stateful chunk scheduling. Whisper wants a strong encoder plus efficient autoregressive decoding.

### 4. The state manager

For a live stream, chunk 42 must continue the same recognition state as chunks 1–41. The engine keeps encoder caches, transducer predictor state, hypotheses, timestamps, and endpoint information associated with a stream ID.

This creates a serving constraint LLM APIs can often avoid: **session affinity is part of correctness.** If the next audio packet lands on another replica, that worker must receive the state too—or the transcript can reset, repeat, or lose context.

---

## Why vLLM’s Main Trick Does Not Map Cleanly

vLLM broke open LLM serving with **PagedAttention**. Autoregressive generation creates a large, variable-length KV cache per request; paging that cache reduces waste and enables continuous batching.

ASR memory has a different shape:

| Engine pressure      | LLM                                    | ASR                                                         |
| -------------------- | -------------------------------------- | ----------------------------------------------------------- |
| Input                | Text tokens known at arrival           | Audio frames, sometimes still arriving                      |
| Dominant execution   | Long token-by-token decode             | Large parallel encoder + usually smaller decode             |
| Per-request memory   | KV cache grows with context and output | Encoder/chunk activations plus compact stream/decoder state |
| Batch key            | Tokens and decode step                 | Frames, duration, deadline, decoder type                    |
| User-visible latency | First token + inter-token gap          | First stable word + end-of-speech finalization              |

PagedAttention still helps autoregressive speech decoders, and this is one reason [vLLM now supports Whisper and other transcription models](https://docs.vllm.ai/en/latest/models/supported_models/#transcription). But a page-efficient decoder does not solve padding in the encoder, deadline scheduling for live audio, or sticky state for a transducer.

The closest ASR equivalent is not one algorithm. It is the combination of:

> **length-aware batching + cached streaming encoders + fast decoding + full-model replicas.**

---

## How ASR Engines Parallelize Work

Yes, ASR uses parallelism heavily—just not usually to make one model fit.

### Replicate first

ASR checkpoints are commonly tens of millions to a few billion parameters. OpenAI’s [Whisper family](https://github.com/openai/whisper#available-models-and-languages) ranges from 39M to 1.55B parameters; its listed runtime footprint ranges from roughly 1 GB to 10 GB of VRAM. A complete model often fits on one accelerator.

That makes the default scale-out design simple: **one full model per GPU, many replicas, and load-balance streams or files across them.** Tensor parallelism is available for bigger speech-language models, but it introduces communication into a latency-sensitive path and is rarely the first lever for classic ASR.

### Batch the encoder

The engine groups similar-length chunks and executes their encoders together. General-purpose servers such as [NVIDIA Triton](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/batcher.html) provide dynamic batching; speech-aware systems add duration buckets, frame budgets, and sequence-aware queues.

### Parallelize offline audio carefully

An hour-long recording can be split into overlapping windows and mapped across workers. The overlap preserves words cut at boundaries, after which timestamps and hypotheses are merged. This produces enormous offline throughput, but it is not the same as streaming: a live engine cannot process audio that has not happened yet.

### Protect live traffic from batch traffic

Offline jobs want large batches and maximum GPU utilization. Live transcription wants a small predictable delay. Putting both behind one FIFO queue produces a system that benchmarks beautifully and interrupts people in production.

The usual design is two admission queues feeding either separate pools or reserved capacity in the same pool:

```plaintext
live chunks ──→ deadline queue ──┐
                                ├─→ ASR replicas
offline files → throughput queue ┘
```

---

## Do 22 Languages Need 22 Routes?

Usually, **no**.

A multilingual checkpoint normally stores those languages in one shared set of weights. The caller supplies a language code, or the engine performs language identification and conditions the same model using a language embedding or control token. Whisper, for example, scores language tokens and inserts the winning token into its decode sequence.

That is an inference decision, not a request being sent to a “Hindi GPU.”

A real router becomes useful only when the backends are genuinely different: a Hindi specialist, a medical English model, a low-latency streaming model, a cheap offline model, or an in-region deployment for data residency. Known language hints should be passed directly because they avoid detection errors—especially on short clips and code-switched speech.

The clean rule remains:

> **Language capability lives inside the model. Routing lives between different models or policies.**

---

## Is There an Open-Source “vLLM for ASR”?

There is no single winner with vLLM’s gravitational pull—yet. The ecosystem is split because server GPUs, mobile devices, offline files, and stateful live audio need different engines.

| Project                                                                                                             | Where it shines                                           | What it actually gives you                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| [vLLM](https://docs.vllm.ai/en/stable/serving/openai_compatible_server/)                                            | GPU serving for generative ASR and speech-language models | OpenAI-compatible transcription, translation, and realtime APIs; batching and broad model support                                                  |
| [faster-whisper](https://github.com/SYSTRAN/faster-whisper)                                                         | High-throughput Whisper on CPU/GPU                        | CTranslate2 kernels, FP16/INT8 execution, batched transcription; commonly paired with an API server such as `speaches`                             |
| [whisper.cpp](https://github.com/ggml-org/whisper.cpp)                                                              | Local and edge Whisper                                    | Quantized C/C++ runtime, broad hardware backends, realtime example, and an OAI-like HTTP server                                                    |
| [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx)                                                                | Streaming and embedded ASR                                | CTC/transducer/Whisper support, stream state, batching, WebSocket servers, and deployment across phones, browsers, CPUs, GPUs, and NPUs            |
| [Triton + NeMo/Riva](https://docs.nvidia.com/deeplearning/riva/user-guide/docs/asr/asr-pipeline-configuration.html) | Datacenter speech services                                | Triton is the open-source general serving layer; NeMo supplies speech models, while Riva packages optimized enterprise streaming/offline pipelines |

The interesting development is that the boundary is disappearing. vLLM’s current model list includes dedicated transcription and realtime speech models, while speech-native projects are adding better batching and OpenAI-compatible APIs. We are moving toward shared serving infrastructure, but transducer state and audio-aware scheduling still require speech-specific machinery.

---

## What Actually Unlocked the ASR Boom?

There was no single PagedAttention moment. Four advances stacked on top of each other.

### 1. Unlabelled audio became training data

[wav2vec 2.0](https://arxiv.org/abs/2006.11477) showed that a model could learn useful speech representations from raw, untranscribed audio and then be fine-tuned with far less labelled data. That changed the economics for languages and domains without enormous human-transcribed datasets.

### 2. Weak supervision made models robust, multilingual defaults

[Whisper](https://cdn.openai.com/papers/whisper.pdf) trained on 680,000 hours of diverse, weakly supervised audio. Instead of winning only on one clean benchmark, it delivered a model developers could download and point at accents, noise, multiple languages, translation, and long-form media. That “works outside the lab” quality created an ecosystem.

### 3. Encoders and decoders stopped wasting so much work

Conformer combined attention’s wider context with convolution’s local acoustic pattern matching. [FastConformer](https://arxiv.org/abs/2305.05084) redesigned downsampling and reported 2.8× faster execution than the original Conformer. [TDT](https://arxiv.org/abs/2304.06795) let a transducer predict duration and skip input frames, reporting up to 2.82× faster ASR inference than conventional transducers.

This is a genuine serving breakthrough: if silence or a long sound spans several frames, the decoder no longer needs to inspect every frame as a separate decision.

### 4. Production inference caught up with the models

Quantization, fused kernels, Flash Attention/SDPA, CTranslate2, GGML, ONNX Runtime, TensorRT, and batching made good models cheap enough to run locally or at high throughput. The [faster-whisper benchmarks](https://github.com/SYSTRAN/faster-whisper#benchmark) show the same Whisper weights running dramatically differently depending on runtime, precision, and batch size.

Distillation pushed again: [Distil-Whisper](https://arxiv.org/abs/2311.00430) reported 5.8× faster inference with 51% fewer parameters while staying within 1% WER of its teacher on its out-of-distribution tests. Its speculative-decoding setup reported another 2× speed-up while preserving the teacher’s output.

That stack—data, architecture, decoder, runtime—is what opened the door. ASR became accurate enough for messy real audio, small enough for commodity hardware, fast enough for live interaction, and open enough that a startup no longer needed a speech-research team before shipping its first prototype.

---

## The Hard Problems That Remain

Modern ASR is impressive, but serving it well is not solved:

- **Mixed lengths destroy batch efficiency.** The engine must bucket constantly without making short requests wait.
- **Streaming is a deadline problem.** A throughput optimization that adds 200 ms of queueing can ruin a voice assistant.
- **State complicates autoscaling.** Moving or evicting a live stream requires moving its recognition state safely.
- **Accuracy and latency pull in opposite directions.** More right-context and larger chunks help recognition but delay partial text.
- **Beam search is irregular.** Hypothesis counts, transcript lengths, and endpoint decisions vary, which makes GPU work less uniform.
- **Code-switching and proper nouns remain hard.** A global WER can hide a terrible experience for one language, accent, customer, or vocabulary.
- **Long-form models can repeat or hallucinate.** Chunk boundaries and previous-text conditioning need careful handling.

This is why the dashboard should track audio-seconds queued, frames per batch, real-time factor, first-partial latency, finalization latency, active streams per replica, and WER sliced by language and environment. Tokens per second tells only the decoder’s corner of the story.

---

## The Short Version

If I had to compress the whole article into six lines:

1. **An ASR engine is an audio-frame scheduler wrapped around an encoder and one of several decoders.**
2. **The dominant parallelism is length-aware encoder batching and full-model replication**, not splitting a small checkpoint across GPUs.
3. **Streaming adds deadlines and sticky state; offline transcription adds aggressive batching and parallel chunking.** Treat them as separate traffic classes.
4. **A multilingual model does not need one route per language.** Route only when you operate genuinely different backends or policies.
5. **There is no single vLLM of ASR.** vLLM, faster-whisper, whisper.cpp, sherpa-onnx, and Triton/NeMo each own a different part of the deployment space.
6. **ASR’s unlock was cumulative:** self-supervision, weakly supervised scale, Conformer-class encoders, faster transducers, distillation, quantization, and optimized runtimes.

The question I started with was slightly wrong. ASR did not need one project to copy vLLM.

It needed an engine designed around the physics of audio: **frames instead of prompts, deadlines instead of open-ended generation, and state that moves at the speed of somebody speaking.**

`#ai` `#voiceai` `#asr` `#speechrecognition` `#inference` `#vllm` `#opensource` `#mlops`
