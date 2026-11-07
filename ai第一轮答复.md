## 结论先说

这 4 个都能在 M1 Max 32GB 上跑，也都能接 Claude Code。但名单里有两个该换掉：

- **Qwen 2.5 Coder 32B** 是 2024 年的模型，2026 年已被同尺寸新模型全面拉开差距，不必再考虑。
- **Qwen 3.5 27B** 本身不错，但已经被 **Qwen3.6-27B**（2026-04）和 **Qwen3.8-27B**（2026-08）连续超越，直接上 3.8。
- **Gemma 4 26B** 和 **Muse Glimmer 30B** 都是 2026 年的新模型，推荐没问题。

**对你这台机器的首选：Qwen3.8-27B**（编码最强）。想要更顺的 agent 体验选 **Muse Glimmer 30B**；想要速度选 **Gemma 4 26B-A4B**。三个都装也就 50GB 左右，可以都试试。

## 先说硬件账本

M1 Max 32GB 的统一内存，系统占用后 GPU 实际可用大约 **23–24GB**（默认 wired limit 约 75%，可用 `sudo sysctl iogpu.wired_limit_mb=...` 略微上调，但有风险）。内存带宽 400GB/s，是速度上限的根源。

关键约束不是模型本身，而是 **Claude Code**：它的系统提示 + 工具定义每轮就有约 **20K tokens**，实际需要 32K–64K 上下文。所以模型权重 + KV cache 必须控制在 ~22GB 内，否则会开始走 CPU/swap，速度断崖。

## 四个推荐逐个点评

| 模型 | 实际尺寸 (Q4) | 32GB 上表现 | 评价 |
|---|---|---|---|
| Qwen2.5 Coder 32B | ~20GB 稠密 | ~8–12 tok/s，上下文余量很小 | 太老，放弃 |
| Qwen3.5 27B | ~17GB | 可跑，但已被 3.6/3.8 取代 | 换成 Qwen3.8-27B |
| Gemma 4 26B-A4B | 权重 ~15.4GB，MoE 仅 3.8B 激活 | ~33 tok/s，但 8K KV 就接近内存上限（安全上下文约 14K） | 快，但长上下文吃紧、编码偏弱 |
| Muse Glimmer 30B | ~17–18GB 稠密 | 官方称 32GB 为"舒适档"，估计十几到 25 tok/s | agent/MCP 定向优化，和 Claude Code 很搭 |

几个具体数据点：

- **Gemma 4 26B-A4B** 是 MoE（25.2B 总参 / 3.8B 激活），速度是四个里最快的，官方测评 LiveCodeBench 80.5；但在 32GB 上开 32K 上下文会非常紧张，建议用 IQ4_XS（~13GB）而不是 Q4_K_M。
- **Muse Glimmer 30B** 是 Meta 2026-08 重新开源的稠密模型（Apache 2.0），SWE-bench Verified 76.0、MCP Atlas 75.5，明显是冲着"本地 agent"设计的；32GB 是它官方标称的安全档。
- 注意 **Qwen3.5/3.6 35B-A3B** 这类 MoE 虽然快，但 Q4 实测要 ~25.6GB，超过你 23GB 的可用额度——**32GB 上不推荐**，会掉到 6–7 tok/s。

## 更好的选择

**Qwen3.8-27B（2026-08，Apache 2.0）** 是当前 32GB 档位的最优解：

- 稠密 27B，Q4 约 14–17GB，混合注意力（48 层线性注意力 + 16 层全注意力）让长上下文的 KV cache 很小，正好补上 32GB 的短板。
- 编码：SWE-bench Pro 61.7、Terminal-Bench 2.1 73.0、LiveCodeBench v6 90.3——同尺寸里最强，Terminal-Bench 尤其贴合 Claude Code 的用法。
- M1 Max 实测：llama.cpp GGUF ~12 tok/s，MLX 或带 MTP 草稿头 ~19 tok/s；比 3.6 慢一些（Metal kernel 还没吃透新架构），但质量提升明显。
- 注意：模型卡数据多为厂商自测，且 2026 年的榜单版本混杂，建议自己跑一圈再定。

其他备选：**gpt-oss:20b**（~14GB、速度快、工具调用成熟）适合追求流畅；**oMLX** 这类带 SSD KV 分页缓存的运行时能把 Claude Code 首轮 TTFT 从 90 秒压到 1 秒级，值得折腾。

## 能装吗？能

最简单是 Ollama（建议最新版，Apple Silicon 已有 MLX 后端）：

```bash
ollama pull qwen3.8:27b        # 或 muse-glimmer:30b / gemma4:26b
```

LM Studio 有图形界面，适合先聊天试模型；跑 Claude Code 还是 Ollama 最省事。

## 能配 Claude Code 吗？能

Ollama 从 **v0.14（2026-01）** 起原生兼容 Anthropic Messages API，不需要任何代理：

```bash
# 先以 64K 上下文启动（默认可能只有 4K–32K，agent 会跑飞）
OLLAMA_CONTEXT_LENGTH=65536 ollama serve

export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_BASE_URL=http://localhost:11434   # 不要加 /v1
export API_TIMEOUT_MS=3000000                      # 首轮 prefill 很慢
claude --model qwen3.8:27b
```

也可以直接 `ollama launch claude` 自动配置。几点现实：

1. **首轮很慢**：20K tokens 的系统提示在 M1 Max 上约 90–100 tok/s 的 prefill，意味着第一次要等 2–4 分钟；后续轮有前缀缓存会好很多（oMLX 的分页 KV 缓存对此改善最大）。
2. **别用默认上下文**：Ollama 在 <24GB 显存时默认只有 4K，必须显式设 32K–64K。
3. **质量是"初级工程师"水平**：简单编辑、样板代码、解释代码没问题；大型仓库的重构还是留给云端 Claude。Ollama 的 Anthropic 兼容层也不支持 `tool_choice` 强制、托管搜索等少数特性。
4. 跑的时候关掉其他吃内存的 App，否则 32GB 会被挤压到 swap。

## 我的建议路线

先 `ollama pull qwen3.8:27b` 跑 Claude Code 试一天；如果嫌首轮慢，换 `gemma4:26b` 的 IQ4_XS 版或 `gpt-oss:20b` 做日常小任务；Muse Glimmer 30B 作为第二选择保留。Qwen2.5 Coder 32B 和 Qwen3.5 27B 就不用下载了。

**Sources:**
- [Ollama Anthropic compatibility](https://docs.ollama.com/api/anthropic-compatibility) / [Ollama blog: Claude Code with Anthropic API compatibility](https://ollama.com/blog/claude)
- [Alibaba unveils Qwen3.8-27B](https://www.alibabacloud.com/blog/alibaba-unveils-qwen3-8-27b-and-releases-weights-of-qwen3-8-flagship-model_603463)
- [Qwen3.5-27B vs Qwen3.6-27B benchmarks](https://llm-stats.com/models/compare/qwen3.5-27b-vs-qwen3.6-27b)
- [Gemma 4 release notes](https://ai.google.dev/gemma/docs/releases) / [Gemma 4 26B-A4B on M1 Max 32GB](https://willitrunai.com/can-run/gemma-4-26b-a4b-on-m1-max-32gb)
- [Meta Muse Glimmer 30B (The Register)](https://www.theregister.com/ai-and-ml/2026/08/10/zuck-rekindles-open-weights-llama-drama-with-muse-glimmer/5285666) / [Muse Glimmer on Mac](https://ai-on-mac.com/articles/meta-muse-glimmer-30b-mac-en/)
- [Qwen3.8-27B on M1 Max 32GB benchmarks](https://bishal.io/blog/six-local-models-m1-max-llama-cpp-vs-mlx.md)
- [Qwen3.5 35B-A3B on M1 Max 32GB (memory analysis)](https://willitrunai.com/can-run/hf-unsloth--qwen3-5-35b-a3b-gguf-on-m1-max-32gb)
- [Why local LLMs are slow even when they fit](https://openclawdc.com/blog/why-local-llms-are-slow-even-when-they-fit/)