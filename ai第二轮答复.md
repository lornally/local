## 1. 两台 Mac，一台专职跑模型：能多给内存，而且这是最划算的用法

系统层面的可调整空间比我上次说的更大：

- macOS 默认给 GPU 的 wired memory 上限在 32GB 机器上只有约 **2/3（~21GB）**，所以默认能用得很少。
- 专用机可以调高到 **26GB 稳妥、28GB（28672）激进**。28GB 时只剩 ~3.5GB 给系统，社区明确说这只适合"无显示器、最小化 GUI 的 headless 推理机"，再高就有内核 panic 风险。

具体做法：

```bash
# 1. 提高 Metal 显存上限（重启失效，可用 LaunchDaemon 持久化）
sudo sysctl iogpu.wired_limit_mb=28672

# 2. 精简系统
sudo mdutil -a -i off                 # 关 Spotlight 索引
# 关掉 iCloud 同步、Siri、通知、浏览器；用一个干净的专用账户
caffeinate -s &                       # 或 pmset 设置不休眠

# 3. 把 Ollama 作为内网服务跑（Ollama 没有鉴权，只暴露在内网）
OLLAMA_HOST=0.0.0.0:11434 \
OLLAMA_CONTEXT_LENGTH=65536 \
OLLAMA_KEEP_ALIVE=-1 \
ollama serve
```

另一台机器作为客户端：

```bash
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_BASE_URL=http://192.168.x.x:11434
claude --model qwen3.8:27b
```

要点：

- 千兆有线完全够——token 流量很小，慢的是服务器端的 prefill，不在网络。
- **两台 32GB 不能合并成一个 64GB 显存池**。exo / MLX distributed 理论上能跨机切分模型，但 1GbE + M1 Max 的组合不划算，别折腾。
- 顺手把模型目录排除出 Spotlight，关掉自动更新，避免跑着跑着系统抢内存。

## 2. Ollama 不"重"，问题是它默认不追求速度

Ollama 的二进制很小、常驻开销也低，说它"重"不准确。真正的差距在引擎和默认值：

- 2026 年 Ollama 已接入 MLX 后端（`-mlx` 标签），但对同一模型的独立测试里，**原生 MLX（LM Studio / oMLX）比 Ollama 的 GGUF 路径快 1.4–2.4×**，TTFT 也更好；一篇 M2 Ultra 上的学术对比同样显示 Ollama 在吞吐和 TTFT 上垫底。
- Ollama 的短板：默认上下文小、缺少 MTP 草稿头 / DFlash 推测解码 / 分页 KV 缓存，工具调用历史上出过 bug。
- 它的 `-mlx` 标签本质是 NVFP4 量化模型，所以"MLX 引擎"和"量化格式"的收益混在一起，实测波动大。

**结论**：用 Ollama 验证"这个模型能不能用"最省事；长期跑 Claude Code，建议换 **oMLX** 或 llama.cpp/MLX，长会话的首轮等待差距是几十秒 vs 几秒。

## 3. 打不过 DeepSeek V4.1 Flash，而且 32GB 上不存在同级别本地方案

先对齐名字：你说的应该是 **DeepSeek-V4.1-Flash-1017**。它是 **763B 稀疏 MoE、1M 上下文、MIT 许可、权重开源**的模型。官方数据：DeepSWE v1.1 77.0、Terminal-Bench 4.0 42.3。

两个层面回答：

- **能不能本地跑**：不能。763B 参数即使 Q4 也要 **~380–400GB**，需要 512GB 级的 M5 Ultra 才谈得上。32GB 没有任何办法。
- **能力谁强**：它和本地模型用的不是同一套 benchmark（DeepSWE v1.1 / Terminal-Bench 4.0 vs SWE-bench Pro / Terminal-Bench 2.1），不能直接对分数。但从 763B vs 27B 的体量差距和 agent 定位看，**日常编码和长任务明显更强**，本地 27–30B 打不过。

所以：**32GB 上最强的 Qwen3.8-27B 只相当于"初级工程师"，DeepSeek Flash 4.1 是另一个级别**。想在本机跑到那个级别，硬件门槛是 64GB 起步、更现实是 256–512GB 统一内存——这也是为什么大多数人还是把它当 API 用。

## 4. 对，decode 主要卡内存带宽；升级有帮助，但 Ultra 才是质变

Decode（逐 token 生成）对稠密模型基本是**内存带宽瓶颈**：每生成一个 token 要把权重读一遍，理论上限 ≈ 带宽 ÷ 权重大小。Prefill（读你的 20K 系统提示）则是**算力瓶颈**，看 GPU 核心数。

| 芯片 | 带宽 | 相对 M1 Max |
|---|---|---|
| M1 Max | 400 GB/s | 1× |
| M4 Max | 546 GB/s | 1.37× |
| M5 Max | 460–614 GB/s | 1.15–1.5× |
| M3 Ultra | 819 GB/s | 2× |
| **M5 Ultra** | **1.2 TB/s** | **3×** |

- Qwen3.8-27B Q4（~17GB）在 M1 Max 的理论上限 ~23 tok/s，实测 10–19；M5 Max 大约提升 1.2–1.5×，M5 Ultra 理论 3×。
- **Prefill 的改善比 decode 更明显**：M5 的 GPU Neural Accelerators 和更多核心，能把 Claude Code 首轮 2–4 分钟的等待压下来。
- 但**容量比速度更关键**：32→96GB 才能跑 100B 级 MoE；只换 M5 Max 36GB 意义不大。2026 款 Mac Studio 是 M5 Max（36–128GB，$2,499 起）和 M5 Ultra（96–512GB，$5,499 起），没有 M4 Ultra。
- 购买建议：只为本地模型，**M5 Max 64GB 是性价比点；认真玩就 M5 Ultra 96GB**。另外 MTP / DFlash 推测解码能白嫖 1.5–1.9×，不换硬件也值得开。

## 5. 三个编码 benchmark 的含义

| Benchmark | 考什么 | 分数含义 |
|---|---|---|
| **SWE-bench Pro** | 给一个真实 GitHub issue 和整个仓库，让模型自己找文件、改代码、跑测试 | 61.7 = 解决 61.7% 的任务；Pro 是 Verified 的更难、抗污染版本，多语言、长任务 |
| **Terminal-Bench 2.1** | 在真实终端里完成多步任务：装环境、调试、改配置、跑命令 | 73.0 = 完成率；**最接近 Claude Code 的实际用法**（agent + 工具调用） |
| **LiveCodeBench v6** | 竞赛编程题（LeetCode 风格），定期换题防污染 | 90.3 = 通过率；只考"写出一段正确代码"，不考 agent 能力 |

注意三点：这些都是**厂商自测**，脚手架和版本不同不能直接横比；分数高不等于在 Claude Code 里好用（工具调用稳定性、指令遵循、长上下文更关键）；对 Claude Code 来说 **Terminal-Bench 和 SWE-bench 的参考价值远高于 LiveCodeBench**。

## 6. gpt-oss:20b 和 oMLX 是什么

**gpt-oss:20b** —— OpenAI 在 2025-08 开源的模型（Apache 2.0）：

- MoE 架构，21B 总参 / **3.6B 激活**，MXFP4 量化约 16GB，128K 上下文。
- 原生支持函数调用、MCP、结构化输出、可调推理档位，工具调用很稳。
- 32GB 上跑得很轻松、速度快；缺点是知识截止 2024-06、纯文本、编码能力弱于 2026 年的 27B 模型。定位是"要流畅、工具调用可靠"的日常 agent 模型。

**oMLX** —— 开源的 macOS 原生 MLX 推理服务器（GitHub `jundot/omlx`，Apache 2.0）：

- 核心是**分页 KV 缓存**：热层放内存、冷层放 SSD，前缀可跨请求甚至跨重启复用，不用重算。
- 专门给 Claude Code 做了适配：Anthropic 兼容的 `/v1/messages`、上下文缩放、SSE keep-alive。
- 官方宣称长会话 TTFT 从 30–90 秒降到 5 秒内（厂商自测）；支持工具调用、MCP、多模型。
- 需要 macOS 15+ / Python 3.10+，16GB 能跑、64GB+ 推荐。在 32GB 上跑 Qwen3.8-27B 没问题，是 Ollama 的性能替代品。

**给你的最终组合**：一台 M1 Max 做专用服务器（wired limit 28GB + oMLX + Qwen3.8-27B），另一台跑 Claude Code 指过去；Ollama 留作快速试模型。要升级硬件就直接看 M5 Ultra 96GB，M5 Max 只是小步改进。

**Sources:**
- [Mac Studio M5 Max / M5 Ultra 技术规格](https://support.apple.com/zh-cn/128107) / [MacRumors 2025 vs 2026 对比](https://www.macrumors.com/guide/2025-vs-2026-mac-studio/)
- [iogpu.wired_limit_mb 调优指南](https://modelpiper.com/blog/iogpu-wired-limit-mb-mac) / [Hardware Corner: 提高 Mac 显存](https://www.hardware-corner.net/increase-mac-vram/)
- [DeepSeek-V4.1-Flash-1017 官方说明](https://api-docs.deepseek.com/news/news261017/) / [Unsloth GGUF](https://huggingface.co/unsloth/DeepSeek-V4.1-Flash-1017-GGUF)
- [oMLX 官网](https://omlx.ai/) / [GitHub jundot/omlx](https://github.com/jundot/omlx)
- [OpenAI gpt-oss-20b 文档](https://developers.openai.com/api/docs/models/gpt-oss-20b)
- [Ollama MLX 实测（DevelopersIO, M2 Max）](https://dev.classmethod.jp/en/articles/apple-mlx-ollama-deep-dive/) / [MetalRT Apple Silicon 引擎对比](https://www.runanywhere.ai/blog/metalrt-fastest-llm-decode-engine-apple-silicon) / [MLX vs Ollama vs llama.cpp 论文](https://huggingface.co/papers/2511.05502)