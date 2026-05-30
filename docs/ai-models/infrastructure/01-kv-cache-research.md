# KV Cache 调研：从 Transformer 推理优化到 Agent Runtime 基础设施

调研日期：2026-05-30

## 这篇文档回答什么

这是一份围绕 **KV Cache** 的横向调研。资料范围包括论文、博客、技术报告、官方文档、benchmark / eval、开源项目、开发者社区讨论、工程案例、招聘市场、事故复盘和历史类比。

本文重点不是逐篇综述所有论文，而是回答几个工程问题：

1. KV Cache 到底缓存了什么，为什么它会成为 LLM 推理的核心瓶颈？
2. 论文、开源框架和云 API 分别怎么解决 KV Cache 的内存、吞吐、延迟和隔离问题？
3. 对 long-context、RAG、coding agent、multi-agent、agent runtime 和 harness engineering 有什么影响？
4. 如何区分已经稳定的事实、论文或厂商观点，以及基于资料做出的推断？

为避免混淆，文中使用三类标签：

- **[事实]**：来自论文、官方文档、开源项目文档、公开 issue、招聘 JD 或可复现系统机制。
- **[观点]**：论文作者、厂商、社区或本文作者对重要性的判断。
- **[推断]**：基于资料做出的工程趋势或设计建议，需要在具体 workload 上验证。

## 核心结论

**[事实]** KV Cache 是 decoder-only Transformer 自回归推理的基础优化。模型生成第 `t` 个 token 时，需要让当前 token 的 query 访问前面所有 token 的 key 和 value；缓存前缀 token 在每层 attention 里的 K/V 张量，可以避免每一步重新计算整个前缀。

**[事实]** KV Cache 的代价是显存。容量近似按下面公式线性增长：

```text
KV cache bytes
≈ batch_size
  × sequence_length
  × num_layers
  × 2                    # K 和 V
  × num_kv_heads
  × head_dim
  × bytes_per_element
```

长上下文、并发 batch、多轮 agent、parallel sampling、beam search、multi-agent 和大量常驻会话都会把 KV Cache 变成主要资源。

**[事实]** 行业里的 KV Cache 优化已经分成几个层次：

| 层次 | 代表技术 | 主要解决什么 |
|---|---|---|
| 模型结构 | MQA、GQA、sliding window attention、hybrid attention | 从源头减少 KV heads 或可见窗口 |
| 单机推理引擎 | PagedAttention、RadixAttention、static/offloaded/quantized cache | 减少碎片、提升 batch、复用前缀 |
| 压缩和裁剪 | KIVI、H2O、SnapKV、CacheGen | 降低 cache 体积或传输成本 |
| 分布式服务 | prefill/decode disaggregation、KV-aware routing、KV offload | 把 KV 作为跨节点状态管理 |
| API 产品 | OpenAI prompt caching、Anthropic prompt caching、Gemini context caching、Bedrock prompt caching | 把底层 KV 复用包装成成本和延迟功能 |
| 安全隔离 | cache salt、workspace / org isolation、selective sharing、timing side-channel 防护 | 避免跨租户缓存泄漏 |

**[观点]** KV Cache 是 2024-2026 年 LLM serving 的“内存管理问题”。模型参数、矩阵乘和 kernel 优化仍然重要，但在长上下文与 agentic workload 下，真正决定成本和延迟的常常是：能不能把前缀缓存命中、能不能把 cache 放得下、能不能在正确的 worker 上复用、能不能安全地跨请求共享。

**[推断]** 对 agent runtime 来说，KV Cache 会变成类似数据库 buffer pool、OS page cache、CDN edge cache 的基础设施对象。未来 agent harness 不只要管理文本上下文，还要管理 cache locality、prefix stability、tool schema 稳定性、cache TTL、cache hit 观测、敏感数据隔离和上下文压缩对缓存命中的影响。

## 先区分几个容易混淆的缓存

**[事实]** “KV Cache”在不同语境里经常被混用，需要先拆开：

| 名称 | 缓存对象 | 是否直接复用旧输出 | 常见场景 |
|---|---|---:|---|
| KV Cache | 每层 attention 的 key/value 张量 | 否 | 单次生成内部、流式 decode |
| Prefix / Prompt Cache | 重复 prompt prefix 对应的 KV state | 否 | 多请求共享系统提示词、长文档、多轮 agent |
| Response Cache | 完整输入到完整输出的结果 | 是 | FAQ、语义缓存、普通应用缓存 |
| Embedding / RAG Cache | 向量、检索结果、rerank 结果 | 否 | RAG pipeline |
| Context / Conversation Store | 原始消息、tool result、摘要 | 否 | agent state management |

**[观点]** 工程讨论里最常见的误解是把 prompt caching 理解成 response caching。Prefix cache 只是跳过重复前缀的 prefill compute，后续 token 仍然会重新生成。

**[推断]** 在 agent 系统里，最好把“文本状态”和“推理状态”分开建模：

```text
文本状态：messages / tool results / summaries / files / retrieved docs
推理状态：KV cache blocks / prefix hashes / cache TTL / worker affinity / offload tiers
```

前者决定模型看到什么，后者决定成本和延迟。

## KV Cache 的基本机制

**[事实]** Transformer attention 来自 2017 年论文 [Attention Is All You Need](https://arxiv.org/abs/1706.03762)。在每一层里，token hidden state 会被投影成 query、key、value。对 decoder-only causal LM 来说，当前位置只能 attend 到过去和当前 token。

没有 KV Cache 时，自回归生成第 `t` 个 token 往往要重新处理整个 prefix：

```text
step 1: 处理 x1
step 2: 重新处理 x1, x2
step 3: 重新处理 x1, x2, x3
...
```

有 KV Cache 后，前面 token 的 K/V 被保存下来：

```text
prefill: 处理完整 prompt，保存每层 K/V
decode step t:
  只计算新 token 的 Q/K/V
  用新 Q attend 到 cached K/V + 新 K/V
  追加新 token 的 K/V 到 cache
```

**[事实]** KV Cache 对 prefill 和 decode 的影响不同：

| 阶段 | 输入形态 | 主要瓶颈 | KV Cache 的作用 |
|---|---|---|---|
| Prefill | 一次处理整个 prompt | 计算量大、TTFT 高 | 产生 prefix 的 K/V |
| Decode | 每次处理一个或少量 token | memory bandwidth、batch 调度 | 读取历史 K/V，追加新 K/V |

**[事实]** Prefix caching 进一步把“本请求内部复用”扩展成“跨请求复用”。如果两个请求有完全相同的前缀，服务端可以复用之前算好的 K/V，只对新增 suffix 做 prefill。

例如：

```text
请求 A:
  [system prompt][tool schemas][repo docs][user question A]

请求 B:
  [system prompt][tool schemas][repo docs][user question B]
```

如果前三段完全相同，请求 B 可以复用前三段的 KV state。

## 显存数量级：为什么长上下文贵

**[事实]** KV Cache 随 `sequence_length` 和 `batch_size` 线性增长。下面是近似量级，不代表具体厂商实现。

假设一个普通 multi-head attention 模型：

```text
num_layers = 32
num_kv_heads = 32
head_dim = 128
dtype = fp16 / bf16 = 2 bytes
```

单 token KV Cache 约为：

```text
32 × 2 × 32 × 128 × 2 = 524,288 bytes ≈ 512 KiB/token
```

一个 32k token 上下文约为：

```text
512 KiB × 32,768 ≈ 16 GiB
```

如果同时服务 8 个这样的请求，仅 KV Cache 就可能接近 128 GiB。

**[事实]** MQA / GQA 会显著改变这个数量级。假设一个大模型使用 80 层、8 个 KV heads、head dim 128、bf16：

```text
80 × 2 × 8 × 128 × 2 = 327,680 bytes ≈ 320 KiB/token
```

128k token 上下文约为 40 GiB。即便用了 GQA，超长上下文的单会话 KV Cache 仍然可以大到足以支配整张 GPU 的可用显存。

**[观点]** “支持 1M context”不等于“便宜地支持很多 1M context 会话并发”。长上下文 capability 是模型能力、推理 kernel、KV Cache 管理、调度、offload 和价格策略共同作用的产品结果。

**[推断]** 对 coding agent 而言，真正值得优化的不是“永远把全部 repo 塞进 1M context”，而是：

- 稳定复用系统提示词、工具定义、项目约定和常用文档。
- 把动态 tool result 和用户临时输入放在靠后位置。
- 用 retrieval、summary 和 file-level context 控制 prefix churn。
- 用 cache hit rate、TTFT 和 cost per task 评估上下文策略，而不是只看 context window 大小。

## 论文脉络

### Transformer 到 MQA / GQA

**[事实]** [Fast Transformer Decoding: One Write-Head is All You Need](https://arxiv.org/abs/1911.02150) 提出 Multi-Query Attention（MQA）：多个 query heads 共享同一组 key/value heads，从而减少 incremental decoding 时 K/V 张量大小和 memory bandwidth。

**[事实]** [GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints](https://arxiv.org/abs/2305.13245) 提出 Grouped-Query Attention（GQA）：key/value heads 数量介于 MHA 和 MQA 之间，并给出把已有 MHA checkpoint uptrain 成 MQA/GQA 的 recipe。论文摘要提到用原预训练 compute 的 5% 做 uptraining。

**[观点]** MQA / GQA 是“从模型结构上给 KV Cache 减负”。这比服务端事后压缩更根本，因为它减少了每一步必须读写的 K/V state。

**[推断]** 新模型如果面向长上下文和高并发推理，几乎一定会在 attention 结构上为 KV Cache 做取舍：更多 query heads 保留表达能力，更少 KV heads 降低推理成本。

### PagedAttention / vLLM

**[事实]** [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180) 把 OS virtual memory / paging 的思想引入 KV Cache 管理。PagedAttention 把每个序列的 KV Cache 拆成固定大小 blocks，不要求逻辑相邻的 tokens 在物理显存中连续，从而减少碎片和预留浪费。

**[事实]** PagedAttention 论文报告 vLLM 在相同 latency 水平下，相比 FasterTransformer 和 Orca 等系统提升约 2-4 倍吞吐，且长序列、大模型和复杂 decoding 算法下收益更明显。

**[观点]** PagedAttention 的重要性不只是“一个更快的 attention kernel”，而是把 LLM serving 的核心问题重新表述成内存分配、引用计数、copy-on-write、block table 和调度问题。

**[推断]** KV Cache 管理会越来越像 OS 内核问题：页表、碎片、换出、预取、租户隔离、优先级和回收策略都会成为推理系统的常规概念。

### RadixAttention / SGLang

**[事实]** [SGLang](https://github.com/sgl-project/sglang) 的论文 [SGLang: Efficient Execution of Structured Language Model Programs](https://papers.nips.cc/paper_files/paper/2024/file/724be4472168f31ba1c9ac630f15dec8-Paper-Conference.pdf) 把 RadixAttention 描述为一种 tree-based LRU KV cache，用 prefix tree 管理跨请求的 KV reuse，并结合 cache-aware scheduling。

**[观点]** vLLM 更像 block/page 管理，SGLang 的 RadixAttention 更强调 program / prompt structure 下的前缀树复用。二者目标相近，但数据结构和调度取向不同。

**[推断]** Agent workload 比普通 chat 更适合 tree / DAG 形态的 cache 管理，因为任务常常共享 system prompt、tool schema、repo context 和阶段性中间状态，但 suffix 分叉很多。

### StreamingLLM、H2O、SnapKV：不是所有 token 都同等重要

**[事实]** [H2O: Heavy-Hitter Oracle for Efficient Generative Inference of Large Language Models](https://arxiv.org/abs/2306.14048) 观察到少量 tokens 对 attention 贡献很大，提出基于 heavy hitters 的 KV Cache eviction 方法。

**[事实]** [Efficient Streaming Language Models with Attention Sinks](https://arxiv.org/abs/2309.17453) 发现保留初始 tokens 的 KV 可以显著稳定 window attention，因此提出 StreamingLLM：保留 attention sinks 加最近窗口，用有限 cache 支撑 streaming use。

**[事实]** [SnapKV](https://arxiv.org/abs/2404.14469) 提出根据 prompt 末尾 observation window 识别重要 KV positions，论文报告在 16K 输入上达到 3.6 倍 generation speed 和 8.2 倍 memory efficiency，同时在多组长序列数据集上保持接近 baseline 的表现。

**[观点]** 这条线说明 KV Cache 不必总是“完整保留”。但一旦开始裁剪，问题就从系统优化变成模型行为近似：不同任务、不同模型、不同位置的信息价值不一样。

**[推断]** 对高风险任务，KV eviction / compression 应该被纳入 eval，而不能只看吞吐。代码审查、法律、医疗、财务、长文档问答这类任务中，丢失中间证据可能表现为“看似流畅但依据断裂”。

### KIVI、Prompt Cache、CacheGen：压缩、量化和传输

**[事实]** [KIVI](https://arxiv.org/abs/2402.02750) 是 tuning-free asymmetric 2-bit KV Cache quantization。论文报告 key cache 适合 per-channel quantization，value cache 适合 per-token quantization；在 Llama、Falcon、Mistral 上接近原质量，同时减少 peak memory，并提高 batch / throughput。

**[事实]** [Prompt Cache: Modular Attention Reuse for Low-Latency Inference](https://arxiv.org/abs/2311.04934) 研究 modular attention state reuse，目标是降低长 prompt 场景下 time-to-first-token。

**[事实]** [CacheGen](https://cs.stanford.edu/~keithw/sigcomm2024/sigcomm24-final1571-acmpaginated.pdf) 研究 KV Cache compression and streaming，用于更快地加载上下文状态，避免每次都重新 prefill 长上下文。

**[观点]** KV Cache 压缩比权重量化更敏感。权重是固定的，KV 是请求相关、位置相关、层相关、会持续追加的运行时状态。

**[推断]** 未来推理服务会同时使用多种精度和多级存储：热点 KV 在 HBM，较冷 KV 在 CPU DRAM，长期可复用 prefix 在 SSD / remote store，跨节点通过 RDMA / NVLink / NIXL 移动。

### DistServe、Mooncake、LMCache：KV Cache 变成分布式系统状态

**[事实]** [DistServe](https://arxiv.org/abs/2401.09670) 把 prefill 和 decode 分配到不同 GPU，减少两阶段互相干扰。论文报告在 TTFT / TPOT 约束下，可服务请求率或 SLO 紧度相对 baseline 有显著提升。

**[事实]** [Mooncake](https://arxiv.org/abs/2407.00079) 是 KVCache-centric disaggregated serving 架构，分离 prefill 和 decoding clusters，并使用 CPU DRAM / SSD 等资源做 disaggregated KV cache。

**[事实]** [LMCache](https://arxiv.org/abs/2510.09665) 把自己定位为 enterprise-scale LLM inference 的 KV cache layer，支持 cache offloading、prefix reuse 和 prefill-decode disaggregation 的跨引擎 cache transfer。论文摘要报告：结合 vLLM 在 multi-round QA 和 document analysis 等 workload 上最高 15 倍吞吐提升，并指出 context truncation 会显著降低 prefix cache hit ratio。

**[观点]** 这条路线说明 KV Cache 已经从“单模型内部 tensor”上升为“集群级共享状态”。它不再只是 kernel 优化，而是 routing、storage、network、admission control 和 SLO 管理。

**[推断]** 大规模 agent 平台的架构会越来越像：

```text
API Gateway
  -> cache-aware router
  -> prefill pool
  -> KV transfer / offload layer
  -> decode pool
  -> trace / billing / safety layer
```

在这种架构中，调度器要同时看负载、cache overlap、用户隔离、SLO、成本和 GPU 拓扑。

## 官方文档和产品化现状

### Hugging Face Transformers

**[事实]** Hugging Face Transformers 的 [KV cache strategies](https://huggingface.co/docs/transformers/v4.53.0/en/kv_cache) 文档列出多种 cache class：`DynamicCache`、`StaticCache`、`OffloadedCache`、`QuantizedCache`、`SlidingWindowCache` 等。文档明确说明 KV cache 可以避免重复计算，并且不同 cache 类型在 memory efficiency、`torch.compile()` 支持、latency 和 long context generation 上取舍不同。

**[事实]** `DynamicCache` 是多数模型默认方式，随生成增长；`StaticCache` 预分配固定大小，更适合 JIT / CUDA graph 类优化；offloaded / quantized cache 用内存或延迟换更长上下文。

**[推断]** 本地实验和小规模服务可以先用 HF cache APIs 理解机制；生产 serving 通常需要 vLLM、SGLang、TensorRT-LLM、Dynamo、LMCache 等更完整的调度与内存管理。

### vLLM

**[事实]** vLLM 的 [Automatic Prefix Caching](https://docs.vllm.ai/en/stable/design/prefix_caching/) 文档说明：prefix caching 会缓存已处理请求的 KV blocks，新请求如果共享相同 prefix，就可以复用这些 blocks。vLLM v1 使用 hash-based approach，每个 KV block 的 hash 与 block tokens 和之前 prefix 相关。

**[事实]** vLLM 文档还提到 `cache_salt` 可注入第一个 block 的 hash，使只有同 salt 请求能复用 cached KV blocks。文档对非加密 hash 在多租户场景里的 collision / privacy 风险有明确警告。

**[观点]** vLLM 的文档已经把 cache 安全写进设计说明，说明 prefix caching 不再只是性能开关，而是多租户系统边界。

### TensorRT-LLM

**[事实]** NVIDIA TensorRT-LLM 的 [KV Cache System](https://nvidia.github.io/TensorRT-LLM/features/kvcache.html) 文档说明：KV cache 是 blocks pool，支持跨请求复用，并通过 offloading、prioritized eviction 等工具增加 reuse。它还支持 variable attention window sizes 以及 MQA / GQA 等 attention 优化。

**[事实]** TensorRT-LLM 的旧版 [KV cache reuse](https://github.com/NVIDIA/TensorRT-LLM/blob/main/docs/source/legacy/advanced/kv-cache-reuse.md) 文档提醒：如果仅按 input ids 区分请求，某些场景可能导致错误复用。这类说明很重要，因为 KV state 的正确性不仅取决于 token ids，还取决于模型版本、position encoding、adapter、prompt schema、multimodal state 等上下文。

**[推断]** 生产系统里的 KV cache identity 不能只理解成“token 序列 hash”。安全的 cache identity 至少要包含模型、tokenizer、prompt template、tools/schema、adapter/LoRA、position policy、cache dtype、tenant/salt 等维度。

### NVIDIA Dynamo

**[事实]** NVIDIA Dynamo 文档把性能核心概括为 disaggregated serving、KV cache-aware routing 和 KV cache offloading。其 [Disaggregated Serving](https://docs.dynamo.nvidia.com/dynamo/design-docs/disaggregated-serving) 文档说明：prefill engine 计算 KV cache，然后把 KV cache 转给 decode engine；Dynamo 使用 NIXL 做 VRAM 到 VRAM 的非阻塞 KV transfer。

**[事实]** Dynamo 的 [KV cache-aware routing](https://docs.nvidia.com/dynamo/user-guides/kv-cache-aware-routing) 文档说明：router 根据 worker 的 cached blocks 与负载做路由，目标是在 cache 命中和 load balance 之间取舍。

**[观点]** 这类系统说明 LLM serving 正在从“启动一个模型 server”演进为“管理一个 KV stateful cluster”。

### OpenAI Prompt Caching

**[事实]** OpenAI 的 [Prompt caching](https://platform.openai.com/docs/guides/prompt-caching) 文档说明：prompt caching 可降低 latency 和 input token cost；缓存对 1024 tokens 以上 prompt 自动启用；cache hits 需要 exact prefix match；建议把静态内容放在 prompt 开头，把动态内容放在末尾；响应 usage 里可通过 `usage.prompt_tokens_details.cached_tokens` 观察缓存命中。

**[事实]** OpenAI 文档还说明 `prompt_cache_key` 可影响 routing，提高相同 prefix 请求落到同一推理机器并复用 cached state 的概率；同一 prefix + key 组合请求率约超过 15 RPM 时，部分请求可能溢出到更多机器，降低 cache effectiveness。OpenAI 文档还说明 prompt caches 不在组织之间共享，extended retention 下 key/value tensors 最大保留 24 小时。

**[观点]** OpenAI 把 KV state locality 抽象成 API 参数，这是一个重要信号：应用开发者不需要直接操作 KV tensor，但需要懂 cache-aware prompt layout 和 routing key。

### Anthropic Prompt Caching

**[事实]** Anthropic 的 [Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) 文档支持 automatic caching 和显式 `cache_control` breakpoint。文档说明 cache hit 要求被标记之前的 prompt segment 100% identical，并且 prompt caching 不影响 output token generation。

**[事实]** Anthropic 默认 cache lifetime 是 5 分钟，也支持额外成本的 1 小时 TTL。Anthropic 文档在 2026-02-05 后说明：Claude API、Claude Platform on AWS、Microsoft Foundry beta 使用 workspace-level isolation；Bedrock 和 Vertex AI 维持 organization-level cache isolation。

**[推断]** 对 Claude Code / coding agent 这类长任务，5 分钟 TTL 对“人类停下来读代码、agent 等待外部命令、subagent 慢任务”可能不够；1 小时 TTL 更适合长 session，但价格、可用性和平台支持需要按当前文档确认。

### Google Gemini Context Caching

**[事实]** Gemini API 的 [Context caching](https://ai.google.dev/gemini-api/docs/caching) 文档提供 implicit caching 和 explicit caching。Gemini 2.5 及更新模型默认启用 implicit caching；explicit caching 允许开发者先把内容缓存为 cached content，后续请求引用 cached tokens。缓存有 TTL，默认 1 小时。

**[观点]** Gemini 的 context caching 更接近显式资源对象，OpenAI 更偏自动 prefix caching + routing key，Anthropic 更偏 breakpoint / automatic cache 控制。三者暴露的控制面不同，但底层目标都是减少重复 prefill。

### Amazon Bedrock Prompt Caching

**[事实]** Amazon Bedrock 的 [Prompt caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html) 文档通过 `cachePoint` 定义 cache checkpoint。文档说明：prompt prefixes 应该在请求之间保持静态；不同模型有最小 cache checkpoint token 数；cache 有 TTL，命中时 TTL 会刷新；响应会包含 cache read / write token 指标。

**[推断]** 如果一个团队通过 Bedrock 使用多个模型供应商，cache 行为会被 Bedrock 抽象统一一部分，但每个底层模型的最小 token、TTL、计费和 checkpoint 限制仍然需要单独测。

## Benchmark / Eval 应该怎么读

**[事实]** KV Cache 优化论文和系统通常报告这些指标：

| 指标 | 含义 | 注意点 |
|---|---|---|
| TTFT | time to first token，首 token 延迟 | prefix caching 主要改善这里 |
| TPOT / ITL | 每个输出 token 延迟 | decode 阶段、memory bandwidth、batch 调度相关 |
| Throughput | tokens/s 或 requests/s | 需同时看 latency SLO |
| Cache hit rate | prefix 或 block 命中比例 | 高命中不一定等于低成本，要看写入成本和 eviction |
| GPU KV usage | KV cache 显存占用 | 指标可能有 bug，需要压测验证 |
| Eviction rate | cache blocks 被回收比例 | 反映容量和调度压力 |
| Recompute tokens | 由于 cache miss 重新 prefill 的 tokens | 更贴近成本 |
| Correctness delta | 优化前后质量差异 | 对 quantization / eviction 特别重要 |
| Cost per successful task | 完成一次任务的总成本 | agent eval 比单次 tokens/s 更重要 |

**[事实]** vLLM / PagedAttention 论文报告吞吐提升，DistServe / Mooncake 报告在 SLO 约束下的 serving capacity，KIVI / SnapKV 报告 memory / throughput 与质量变化，OpenAI / Anthropic / Bedrock / Gemini 文档报告 cached tokens 与成本/延迟指标。

**[观点]** KV Cache eval 不能只看 `tokens/s`。交互式 chat 更关心 TTFT 和 ITL，batch offline 更关心吞吐和成本，coding agent 更关心 cost per resolved task、cache miss 后是否重算大量上下文、长任务是否因 TTL 过期变慢。

**[推断]** 面向 coding agent 的 benchmark 可以这样设计：

```text
固定项目上下文 + 20 个连续任务
对比策略：
  A. 每次发送完整上下文，不做 prefix layout
  B. 稳定 system/tools/repo docs 前缀
  C. 稳定前缀 + prompt_cache_key / cache_control / cachedContent / cachePoint
  D. 稳定前缀 + dynamic tool result 放末尾
观察：
  cached_tokens
  TTFT
  wall-clock task time
  total input cost
  resolved rate
  cache misses caused by context compaction / tool schema change / timestamp
```

## 开源项目地图

**[事实]** 下面这些项目与 KV Cache 直接相关：

| 项目 | 角色 | KV Cache 相关能力 |
|---|---|---|
| [vLLM](https://github.com/vllm-project/vllm) | 高吞吐 LLM serving engine | PagedAttention、continuous batching、prefix caching、KV connector |
| [SGLang](https://github.com/sgl-project/sglang) | structured LM program runtime / serving | RadixAttention、cache-aware scheduling、prefix reuse |
| [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM) | NVIDIA 推理优化栈 | paged KV cache、reuse、offload、FP8、GQA/MQA 支持 |
| [Dynamo](https://docs.dynamo.nvidia.com/) | NVIDIA 分布式推理框架 | KV-aware routing、disaggregated serving、KV offload |
| [LMCache](https://github.com/LMCache/LMCache) | KV cache layer | offload、reuse、跨 engine KV transfer |
| [KIVI](https://github.com/jy-yuan/KIVI) | KV quantization | 2-bit asymmetric KV cache quantization |
| [llama.cpp](https://github.com/ggml-org/llama.cpp) | 本地推理 | KV cache、context shift、KV quantization / offload 相关能力 |
| [Hugging Face Transformers](https://github.com/huggingface/transformers) | 模型库 | cache classes、prefill cache、offload / quantized cache |

**[观点]** 如果目标是学习机制，最好的路线是先用 Hugging Face / tiny model 手写一次 `past_key_values`；如果目标是生产 serving，应该直接研究 vLLM / SGLang / TensorRT-LLM / Dynamo 的调度和内存管理，而不是自己从零写 server。

**[推断]** 对个人和小团队，KV Cache 的选型优先级大致是：

```text
单机开源模型 serving：vLLM 或 SGLang
NVIDIA 生产优化：TensorRT-LLM / Dynamo
长文档或重复上下文：prefix caching + LMCache 类 offload
本地开发和实验：Transformers / llama.cpp
API 调用：OpenAI / Anthropic / Gemini / Bedrock 的 prompt caching 控制面
```

## 开发者社区讨论

**[事实]** Reddit、GitHub discussions、vLLM forum 等社区里，KV Cache 高频讨论点包括：

- 本地显存不够时，KV cache quantization 是否值得。
- 长上下文是否应该靠 RAG、summary 还是 persistent KV cache。
- vLLM prefix caching 命中率为什么低。
- OpenAI / Anthropic / Gemini / Bedrock prompt caching 的 TTL、命中条件和计费行为。
- 多 agent 是否能直接传 KV cache，而不是传文本。
- KV cache offload 到 CPU / SSD 的延迟是否抵消收益。
- FP8 KV cache、prefix cache、disaggregated transfer 出错时为什么会产生静默错误。

**[事实]** vLLM forum 讨论 [How should I set kv cache in vllm?](https://discuss.vllm.ai/t/how-should-i-set-kv-cache-in-vllm/1947) 指出：vLLM 可缓存共享 prefix 的 KV cache，但客户端仍要负责保存和发送完整对话历史；vLLM 不替应用管理用户对话历史。

**[观点]** 社区讨论里最有价值的不是具体参数，而是反复出现的痛点：大家都希望缓存长文档、工具 schema、repo context 和 agent session，但真实系统会被 TTL、prefix 精确匹配、模板微小变化、缓存容量、worker routing 和 API 不透明性限制。

**[推断]** “persistent KV cache 替代 RAG”只适合窄场景：文档集合小、更新慢、用户明确在同一知识域内提问、模型 context 足够、服务端可控制 KV state 生命周期。对开放域问答或大规模知识库，RAG 仍然是更稳的主路径。

## 工程案例和事故复盘

公开、正式的 KV Cache 事故复盘不多，但 GitHub issue 已经暴露出几类真实风险。

### FP8 KV cache scale 错误导致静默输出腐化

**[事实]** vLLM issue [#37554](https://github.com/vllm-project/vllm/issues/37554) 报告：在 Qwen3.5 hybrid GDN+Attention 模型上使用 `--calculate-kv-scales` + FP8 KV cache 会导致输出腐化。issue 描述 root cause 是 profiling 阶段用 dummy forward 计算 per-layer FP8 quantization scales，而 hybrid recurrent state 未初始化，导致错误 scales 被固定并用于后续推理。

**[观点]** 这类 bug 的危险点在于它不是 crash，而是“模型继续回答，但答案变成 hallucinated inputs、topic loops、gibberish”。这比显式错误更难监控。

**[推断]** KV cache dtype / quantization 变更必须跑 correctness regression，不能只跑吞吐 benchmark。至少要覆盖数学、事实、多轮、代码、长上下文和目标业务样本。

### Prefill/decode disaggregation 下 KV transfer 可能腐化

**[事实]** SGLang issue [#23020](https://github.com/sgl-project/sglang/issues/23020) 报告：PD disaggregation 模式下约 100 个请求后模型输出逐渐退化成重复循环和乱码。issue 中列出怀疑点包括 KV page data transfer 写入地址、旧 page 复用、D-side page reuse race、Mooncake session 累积状态等。

**[观点]** 分离 prefill/decode 后，KV Cache 成了网络传输对象。正确性不只取决于 attention kernel，还取决于 page ownership、transfer completion、metadata consistency 和 scheduler race。

**[推断]** 生产 PD 架构应该把 KV transfer 当成可靠数据通道来观测：校验 block metadata、生命周期、transfer completion、reuse race、page poisoning、fallback recompute 和 per-request checksum / canary。

### Host / secondary cache 错误检索导致答错上下文

**[事实]** TensorRT-LLM issue [#8274](https://github.com/NVIDIA/TensorRT-LLM/issues/8274) 报告：使用 host / secondary cache 后，在高并发请求后 benchmark accuracy 下降，人工检查发现像是 prefix caching 检索了错误上下文。

**[观点]** 多层 cache 越复杂，越要重视 cache identity 和 invalidation。错误命中比 cache miss 更糟，因为它会把别的上下文注入当前推理。

**[推断]** KV cache 系统应优先保证“miss 后重算”而不是“错误 hit”。当 cache connector load 失败或校验失败时，安全降级策略应该是丢弃相关 blocks 并从最长可信 prefix 重算。

### 指标错误也会造成运维误判

**[事实]** vLLM issue [#20302](https://github.com/vllm-project/vllm/issues/20302) 报告 `kv_cache_usage_perc` / `gpu_cache_usage_perc` 指标异常。issue 里同时出现 prefix cache hit rate 与延迟观测不完全一致的情况。

**[观点]** KV Cache 是强状态系统，metrics 如果不可信，调参会变成猜测。hit rate、usage、eviction、TTFT 要一起看。

**[推断]** KV Cache 观测最少需要四类指标：

- 命中：prefix/block hit rate、cached tokens、cache read tokens。
- 容量：GPU/CPU/disk KV usage、fragmentation、eviction。
- 延迟：TTFT、TPOT/ITL、cache lookup time、transfer time。
- 正确性：fallback recompute count、cache validation failures、silent corruption canaries。

## 安全：Prompt Caching 带来的 side channel

**[事实]** [Auditing Prompt Caching in Language Model APIs](https://arxiv.org/abs/2502.07776) 指出：prompt caching 会带来数据相关的 timing differences，cached prompt 比 non-cached prompt 更快；如果 cache 在用户之间共享，攻击者可能通过响应时间判断某些 prompt prefix 是否被别人使用过。论文称在多个真实 API provider 中检测到 global cache sharing。

**[事实]** NVIDIA 技术博客 [Structuring Applications to Secure the KV Cache](https://developer.nvidia.com/blog/structuring-applications-to-secure-the-kv-cache/) 讨论了 prefix caching 的 timing side-channel 风险，并建议在 prompt 早期加入用户或会话标识，以减少跨用户 prefix collision。vLLM 文档通过 `cache_salt` 提供类似隔离手段；Anthropic 和 OpenAI 文档也明确了 org / workspace 级隔离。

**[事实]** 后续研究如 [Selective KV-Cache Sharing to Mitigate Timing Side-Channels in LLM Inference](https://arxiv.org/abs/2508.08438) 和 [CachePrune](https://arxiv.org/abs/2605.23640) 尝试在共享效率和隐私隔离之间做细粒度折中。

**[观点]** KV Cache 安全的难点是性能目标和隐私目标天然冲突：共享越多，命中越好，泄漏面也越大；隔离越强，性能收益越少。

**[推断]** 企业 agent 平台应该默认：

- 不跨组织共享敏感 prefix。
- 对包含 secrets、PII、客户文档、代码仓库的 prefix 使用 tenant salt 或禁用跨租户共享。
- 不把 cache hit/miss 细节暴露给不可信用户。
- 对 prompt caching 文档和 DPA / security review 做专门说明。
- 把 prompt cache retention 与数据保留政策分开审查。

## 招聘市场信号

**[事实]** 2026 年公开 JD 中，KV Cache 已经从研究词汇变成 inference engineer 的常见技能项。调研中看到的岗位包括：

- NVIDIA [Principal Software Engineer - Large-Scale LLM Memory and Storage Systems](https://jobs.nvidia.com/careers/job/893392559275)：JD 提到 KV-cache offload、reuse、remote sharing、disaggregated prefill、peer-to-peer KV-cache sharing、多层 KV-cache storage、RDMA / NVLink 等。
- Periodic Labs [LLM Inference Engineer](https://jobs.ashbyhq.com/periodic-labs/ad93b9c5-e5e5-4840-a250-e6c332c8fb53)：JD 提到 TensorRT-LLM、vLLM、SGLang，以及 tensor/expert/pipeline parallelism、speculative decoding、KV cache management。
- Modular [Cloud Inference Engineer](https://builtin.com/job/cloud-inference-engineer/9190165)：JD 提到 disaggregated inference、multi-node deployment、high performance networking、distributed kv-cache management。

**[观点]** KV Cache 能力正在从“懂 Transformer”升级为“懂 GPU 内存、分布式系统、网络、调度和产品成本”的岗位要求。

**[推断]** 如果一个工程师想进入 LLM inference / agent infrastructure 方向，KV Cache 是很好的切入口。可展示的项目包括：

- 手写一个 tiny Transformer incremental decoding KV cache。
- 给 Hugging Face model 实现 prefilled cache reuse demo。
- 用 vLLM 压测 prefix caching hit/miss、TTFT 和吞吐。
- 复现 KIVI / quantized cache 的质量与显存 trade-off。
- 做一个 cache-aware agent prompt layout benchmark。
- 写一个 mini KV block allocator 或 radix prefix cache。

## 历史类比

**[观点]** KV Cache 不是一个全新的工程问题，它更像多个旧问题在 LLM serving 里的重组。

### OS Page Cache / Virtual Memory

**[事实]** PagedAttention 明确借鉴 OS virtual memory 和 paging。KV blocks 对应 page，block table 对应 page table，eviction / allocation / fragmentation 对应内存管理问题。

**[推断]** 未来 KV Cache 会出现更多类似 OS 的策略：working set estimation、page replacement、copy-on-write、prefetch、huge page、NUMA affinity、page coloring、安全隔离。

### CDN / HTTP Cache

**[观点]** Prefix caching 类似 CDN：静态前缀越稳定，命中越高；动态内容越早出现，缓存越容易失效；TTL、cache key、purge、tenant isolation 都重要。

**[推断]** Prompt engineering 会部分变成 cache key engineering：字段顺序、时间戳、tool schema、JSON serialization、空白字符、系统提示词版本都可能影响命中。

### Database Buffer Pool

**[观点]** KV Cache 是推理系统的 hot state。它和数据库 buffer pool 一样，需要在有限内存里保留最可能复用、最昂贵、最影响延迟的 blocks。

**[推断]** Agent runtime 可以借鉴数据库的观测指标：hit ratio、dirty / clean pages、eviction cause、working set、query plan；只是这里的 query plan 变成 prompt / tool / context plan。

### CPU Cache / Branch Prediction

**[观点]** 多 agent 和 branching generation 会产生分叉路径，KV block sharing 和 copy-on-write 类似 CPU cache line / speculative execution 的资源管理问题。

**[推断]** 对 self-consistency、beam search、parallel tool exploration，cache sharing 会成为降低 branching 成本的关键。

## 对 Agent / Harness Engineering 的影响

**[事实]** Agent workload 有几个典型特征，都会放大 KV Cache 重要性：

- 系统提示词长。
- 工具定义多。
- 多轮上下文不断增长。
- tool result 大且动态。
- coding agent 会反复读取同一 repo / docs。
- subagent 共享相同任务背景但探索不同分支。
- 长任务中人类或外部工具会造成时间间隔。

**[观点]** Agent harness 的 context engineering 不应只问“模型能不能看到这些信息”，还要问“这些信息是否稳定、是否可缓存、是否值得缓存、是否会破坏前缀命中”。

**[推断]** 一个 cache-aware agent prompt layout 可以这样组织：

```text
稳定高复用前缀：
  system policy
  developer instructions
  tool schemas
  stable project rules
  stable repo map / docs index

中等复用内容：
  current task plan
  selected files / excerpts
  reusable examples

动态后缀：
  latest user message
  latest tool outputs
  timestamps
  command stdout/stderr
  volatile retrieved chunks
```

**[推断]** 对 coding agent，尤其要避免把这些内容放到 prompt 前部导致 cache bust：

- 当前时间戳。
- 随机 request id。
- 每轮变化的 tool result。
- 不稳定排序的 JSON object。
- 每次都重新生成的文件列表。
- context compaction 后格式大幅变化的摘要。
- 可选工具 schema 顺序变化。

## 实践清单

### 使用 API prompt caching

**[推断]** 如果使用 OpenAI / Anthropic / Gemini / Bedrock，优先做这些事：

1. 把稳定内容放在请求开头。
2. 把动态内容放在请求末尾。
3. 固定 tool schema、JSON key 顺序和 prompt template 版本。
4. 记录 `cached_tokens` / cache read / cache write 指标。
5. 以 20-100 个真实多轮任务做 hit rate 和 TTFT 测试。
6. 根据供应商文档选择 `prompt_cache_key`、`cache_control`、`cachedContent` 或 `cachePoint`。
7. 用租户、项目、会话或 repo 维度设计 cache key，不要盲目全局共享。
8. 对 TTL 过期导致的 re-warm 成本单独建模。
9. 对敏感文档和代码仓库做 cache retention / isolation 审查。

### 自托管 serving

**[推断]** 如果自托管 vLLM / SGLang / TensorRT-LLM：

1. 先定义 workload：chat、RAG、coding agent、batch eval、long document QA。
2. 分别测 cold prompt、warm prefix、长输出、短输出。
3. 看 TTFT、TPOT、吞吐、显存、cache hit、eviction，而不是只看 tokens/s。
4. 开启 prefix caching 前后做 correctness diff。
5. 如果启用 FP8 / quantized KV cache，必须跑业务样本回归。
6. 如果启用 offload / secondary cache，检查错误命中和 stale cache 风险。
7. 如果启用 disaggregated prefill/decode，检查 KV transfer race、fallback recompute、page lifecycle。
8. 对多租户场景使用 salt / namespace / org isolation。
9. 把 cache metrics 接到 tracing：每个 request 记录 prefix length、matched blocks、recomputed tokens、worker id。

### 设计 agent eval

**[推断]** Agent eval 应该把 KV Cache 纳入指标：

| 维度 | 指标 |
|---|---|
| 成本 | input tokens、cached input tokens、cache writes、cache reads、cost per task |
| 延迟 | TTFT、step latency、tool wait 后 cache rewarm |
| 可靠性 | cache miss 后是否正确恢复、压缩后 resolved rate |
| 状态 | prompt prefix diff、tool schema diff、compaction diff |
| 安全 | tenant isolation、sensitive prefix cache policy |

## 反模式

**[观点]** 下面这些做法会让 KV Cache 收益大幅下降：

- 每轮都在 system prompt 顶部插入当前时间。
- 每轮动态生成 tool schema 或随机排序 tools。
- 把最新 tool output 放到所有稳定说明之前。
- 在多轮 agent 里频繁重写完整 system prompt。
- 为了“节省 tokens”频繁 compaction，但导致稳定 prefix 变化。
- 只看供应商宣传的缓存折扣，不记录自己的 cached tokens。
- 把 cache hit 当成一定正确，忽略错误命中和静默腐化。
- 在多租户系统里全局共享 prefix cache，却没有 salt 和 timing side-channel 评估。

## 未解决问题

**[事实]** KV Cache 仍有大量开放问题：

- 长上下文下哪些 token 应该保留、压缩或丢弃。
- KV Cache quantization 的任务级质量影响。
- 非 prefix reuse 如何可靠实现。
- 多模态输入的 cache identity 如何定义。
- LoRA / adapter / tool schema / prompt template 变化如何 invalidation。
- 分布式 KV transfer 的一致性与校验。
- 跨租户共享的 privacy / timing side-channel。
- Cache-aware routing 与 load balancing 的最优取舍。
- Agent context compaction 与 prefix caching 的协同。

**[推断]** 未来两年，KV Cache 的研究和工程会继续沿四条线推进：

1. **更小**：GQA/MQA、quantization、eviction、compression。
2. **更远**：CPU/SSD/remote memory offload，跨节点 KV transfer。
3. **更准**：semantic-aware eviction、task-aware cache policy、agent-aware scheduling。
4. **更安全**：tenant isolation、selective sharing、side-channel mitigation、cache audit。

## 参考资料

### 基础和模型结构

- [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- [Fast Transformer Decoding: One Write-Head is All You Need](https://arxiv.org/abs/1911.02150)
- [GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints](https://arxiv.org/abs/2305.13245)

### KV Cache 管理和 serving 系统

- [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180)
- [vLLM Automatic Prefix Caching](https://docs.vllm.ai/en/stable/design/prefix_caching/)
- [SGLang: Efficient Execution of Structured Language Model Programs](https://papers.nips.cc/paper_files/paper/2024/file/724be4472168f31ba1c9ac630f15dec8-Paper-Conference.pdf)
- [TensorRT-LLM KV Cache System](https://nvidia.github.io/TensorRT-LLM/features/kvcache.html)
- [NVIDIA Dynamo Disaggregated Serving](https://docs.dynamo.nvidia.com/dynamo/design-docs/disaggregated-serving)
- [NVIDIA Dynamo KV Cache-Aware Routing](https://docs.nvidia.com/dynamo/user-guides/kv-cache-aware-routing)
- [Hugging Face Transformers KV cache strategies](https://huggingface.co/docs/transformers/v4.53.0/en/kv_cache)

### 压缩、裁剪和长上下文

- [H2O: Heavy-Hitter Oracle for Efficient Generative Inference of Large Language Models](https://arxiv.org/abs/2306.14048)
- [Efficient Streaming Language Models with Attention Sinks](https://arxiv.org/abs/2309.17453)
- [Prompt Cache: Modular Attention Reuse for Low-Latency Inference](https://arxiv.org/abs/2311.04934)
- [KIVI: A Tuning-Free Asymmetric 2bit Quantization for KV Cache](https://arxiv.org/abs/2402.02750)
- [SnapKV: LLM Knows What You are Looking for Before Generation](https://arxiv.org/abs/2404.14469)
- [CacheGen: KV Cache Compression and Streaming for Fast Large Language Model Serving](https://cs.stanford.edu/~keithw/sigcomm2024/sigcomm24-final1571-acmpaginated.pdf)

### 分布式和 cache layer

- [DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving](https://arxiv.org/abs/2401.09670)
- [Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving](https://arxiv.org/abs/2407.00079)
- [LMCache: An Efficient KV Cache Layer for Enterprise-Scale LLM Inference](https://arxiv.org/abs/2510.09665)

### API Prompt / Context Caching

- [OpenAI Prompt caching](https://platform.openai.com/docs/guides/prompt-caching)
- [Anthropic Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini API Context caching](https://ai.google.dev/gemini-api/docs/caching)
- [Amazon Bedrock Prompt caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html)

### 安全和隐私

- [Auditing Prompt Caching in Language Model APIs](https://arxiv.org/abs/2502.07776)
- [Structuring Applications to Secure the KV Cache](https://developer.nvidia.com/blog/structuring-applications-to-secure-the-kv-cache/)
- [Selective KV-Cache Sharing to Mitigate Timing Side-Channels in LLM Inference](https://arxiv.org/abs/2508.08438)
- [CachePrune: Privacy-Aware and Fine-Grained KV Cache Sharing for Efficient LLM Inference](https://arxiv.org/abs/2605.23640)

### 工程案例 / Issue

- [vLLM issue #37554: FP8 KV cache scale corruption](https://github.com/vllm-project/vllm/issues/37554)
- [SGLang issue #23020: PD disaggregation KV cache corruption](https://github.com/sgl-project/sglang/issues/23020)
- [TensorRT-LLM issue #8274: incorrect KV cache retrieval](https://github.com/NVIDIA/TensorRT-LLM/issues/8274)
- [vLLM issue #20302: kv_cache_usage_perc metric](https://github.com/vllm-project/vllm/issues/20302)

### 开发者社区和招聘市场

- [vLLM forum: How should I set kv cache in vllm?](https://discuss.vllm.ai/t/how-should-i-set-kv-cache-in-vllm/1947)
- [NVIDIA Principal Software Engineer - Large-Scale LLM Memory and Storage Systems](https://jobs.nvidia.com/careers/job/893392559275)
- [Periodic Labs LLM Inference Engineer](https://jobs.ashbyhq.com/periodic-labs/ad93b9c5-e5e5-4840-a250-e6c332c8fb53)
- [Modular Cloud Inference Engineer](https://builtin.com/job/cloud-inference-engineer/9190165)
