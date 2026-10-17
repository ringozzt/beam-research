# Beam 调研报告

> Reflection AI 的 501B 稀疏 MoE 开放权重模型（激活 23B），Apache-2.0，2026-10-16 权重上架 Hugging Face。
> 调研日期 2026-10-17 · 数据截至当日 · 来源按一手（模型卡/官方博客/HF API）与二手（媒体）分级标注

一句话结论：**Beam 是"西方开放权重"阵营的第一款能打的大模型**——501B 参数只激活 23B，推理账单按 23B 算；Apache-2.0 连蒸馏都不禁，是真正能"拥有"的模型；官方把它定位成 coding / 推理 / agent 三件套，第一卖点不是跑分最高，而是**同等质量下推理 token 少约 3 倍**——agent loop 的成本杀手。

## 思维导图

```mermaid
mindmap
root((Beam<br/>Reflection AI))
是什么
501B 稀疏 MoE，激活 23.4B
Apache-2.0，真开源
纯文本，1M 上下文
2026-10-16 权重上架 HF
公司
2024 年成立，布鲁克林
创始人 Misha Laskin / Ioannis Antonoglou（前 DeepMind）
Nvidia 领投 $800M，估值 $25B
FT：Nvidia 谈加码或收购（未证实）
架构
52 层（3 dense + 49 MoE）
256 experts，152k 词表
Gated GQA + Sliding Window Attention
Modified Harmony chat template
BF16 / FP8 / NVFP4 三种精度
训练（公司口径）
23.8T tokens 预训练
10,500 × GB300 四周 RL
1 亿+ rollouts
技术报告尚未发布
关键能力
推理省 token（同等质量约 3x）
AIME 2026 98.0%
SWE-bench Verified 81.6%
MCP Atlas 80.8%
Terminal Bench v2.1 80.1%
横向对照
GLM-5.2：推理相当，算力约 1/3–1/4
Qwen 3.8-Max：coding/agent 接近
Kimi K3：raw 能力仍领先
DeepSeek V4.1 Flash：Terminal Bench 更高
k2 视角
23B active = 推理账单小
不禁蒸馏 = 可拥有的底座
为 agentic coding 而生
未核实
技术报告 "coming"
基准均为官方自测
DFlash2 变体未解释
```

交互版：[mindmap.html](./mindmap.html) · 图文版：[report.html](./report.html)

## 1. 是什么

- **模型**：Beam，Reflection AI 的首个开放权重模型。稀疏 MoE：总参数 501.3B，每 token 激活 23.4B。纯文本输入输出，1M token 上下文，知识截止 2026-06-30。发布形态为 instruction-tuned，BF16 / FP8 / NVFP4 三种精度（另有一个 DFlash2 变体仓库，模型卡未解释——未核实）。
- **许可**：Apache-2.0（HF API tags 实测 `license:apache-2.0`）。模型卡明确列出**不禁止**：蒸馏、用输出训练其他模型、微调、量化、合并——比 Llama 系社区许可证干净得多。
- **仓库**：`reflection/Beam-1.0-501B-A23B`（BF16 主仓库）及 `-FP8` / `-NVFP4` / `-DFlash2` 三个变体；另有社区 GGUF（DevQuasar，非官方）。上架首日下载量数百，likes 破百。
- **时间线**：2026-10-05 官方博客官宣（当时只有 waitlist，权重未放）；2026-10-16 权重上架 HF、模型卡上线；技术报告标注 "coming"（截至调研日未出）；安全报告见 reflection.ai/beam-safety-report。

## 2. 公司：Reflection AI

- **成立**：2024 年，布鲁克林。创始人 Misha Laskin（CEO，前 DeepMind，主导过 Gemini 的 reward modeling）与 Ioannis Antonoglou（AlphaGo 联创）。
- **融资**：Nvidia 领投 $800M（属 $2B Series B 一部分，2025-10，估值 $8B）；2026-04 新一轮融资，pre-money 估值 $25B，半年翻三倍；参投方含 Eric Schmidt、Citigroup、1789 Capital、Lightspeed、Sequoia。
- **Nvidia 关系**：除领投外，FT 于 2026-10-10/11 爆料 Nvidia 正谈加码投资或直接收购（含 acqui-hire 选项以规避反垄断审查），早期阶段、双方未评论——**未证实**。另据一家金融信息站口径，Nvidia 于 2026-09 以 $13B 收购 Hugging Face——**第三方单源，谨慎引用**。
- **定位**：做"西方开放权重前沿"（Western open-weight frontier），直接对标 DeepSeek / Qwen / GLM / Kimi——OpenAI、Anthropic、Google 闭源，Meta 从前沿开放发布退后之后，西方阵营在开放权重榜单上长期缺席，Beam 是来补这个空位的。

## 3. 架构（模型卡一手）

| 属性 | Beam 501B A23B MoE |
| --- | --- |
| 总参数 / 激活参数 | 501.3B / 23.4B |
| 层数 | 52（3 dense + 49 MoE） |
| Expert 数 | 256 |
| 注意力 | Gated GQA + Sliding Window Attention |
| 上下文 | 1M tokens |
| 词表 | 152k |
| Chat 模板 | Modified Harmony format |
| 精度 | BF16 / FP8 / NVFP4 |
| 模态 | 纯文本 |

- **训练数据**：公开来源 + 商业合作协议数据；mid-training 用真实世界策展样本（代码库、长程任务、长文档）练推理、工具使用与长上下文；RL 环境覆盖 coding / agent / STEM，主要走合成数据管线，辅以供应商数据与开源数据。
- **训练规模（公司口径，未独立验证）**：23.8T tokens 预训练；RL 阶段 10,500 × GB300 跑四周，生成 1 亿+ rollouts，覆盖约 100 万个合成 coding / agent / STEM 环境；自称 92.3% goodput。技术报告未出之前，这些数字按官宣口径引用。
- **部署账本**：推理算力跟 23B 走，显存跟 501B 走——FP8 约 500GB，NVFP4 约 250GB。这是多 GPU 服务器模型，不是工作站模型；等社区量化/蒸馏版本出来，门槛才会下来。

## 4. 能力：为 agent 而生的三个卖点

官方把 Beam 的能力归纳为四块（token-efficient reasoning / software engineering / agentic tool use / math & science），落到数字上最值得看的是这三组：

- **推理省 token**：同等解题质量下，推理阶段 token 用量约为领先开放模型的 **1/3**。在 agent 场景里，multi-step 任务的推理 token 是成本大头——这不是跑分，是账单。
- **Agentic coding**：SWE-bench Verified 81.6%、SWE Bench Pro v2-Hard 76.5–78.6%、Terminal Bench v2.1 80.1%、DeepSWE v1.1 43.3%。注意 DeepSWE 一项明显落后于 GLM-5.3（61.0）与 Kimi K3（68.0）——**长程软件工程仍是短板**。
- **工具与搜索**：MCP Atlas 80.8–81.5%（MCP 工具集成是官方卖点之一）、BrowseComp 76.8%（带上下文管理）、DeepSearchQA 80.5%、HLE（带工具）51.1% vs（不带工具）36.0%——工具把分数抬了 15 个点，说明这模型是"带工具才完全体"的设计。
- **纯推理**：AIME 2026 97.7–98.0%、GPQA Diamond 90.6%、LiveCodeBench 88.3%。

完整基准表见 [report.html](./report.html) §4（含 BF16 / FP8 / NVFP4 三列——量化几乎不掉点，NVFP4 的 SWE-bench Verified 还有 80.7%）。

## 5. 横向对照：打的不是第一，是"性价比第一"

| 对手 | 官方口径 | 解读 |
| --- | --- | --- |
| GLM-5.2（Z.ai） | 推理相当，推理算力约 1/3–1/4 | 公司宣称的估算值，非实测美元成本；发布前先打"效率"这张牌 |
| Qwen 3.8-Max | coding / agent 接近 | 阿里系大 MoE，Beam 以小激活对大激活 |
| Kimi K3 | raw 能力仍领先 | 月之暗面，Beam 没 claim 超过它 |
| DeepSeek V4.1 Flash | Terminal Bench 90.6% > Beam 80.1% | 单项仍被拉开 |
| Nemotron 3 Ultra | SWE-bench Verified 70.7% < Beam 80.9% | 西方阵营内部，Beam 已领先 |

大判断：**Beam 的赌注是"效率"而非"峰值"**——不跟 Kimi K3 拼 raw 分，而是把"同等智能、更少算力"做成可验证的成本优势。西方开放权重长期被中系模型统治榜单，Beam 是第一款有资格上桌的 Apache-2.0 大模型。

## 6. 安全与对齐

- 三层评估：Open Safety Framework、自家 Iris Safety Evaluations（ISE，**将开源**的评测套件，覆盖单轮/多轮/越狱/agent）、外部第三方 assurance。
- ISE 数字：有害请求拒绝/转向率 88.9–89.7%，无害请求合规率 81.5–82.1%。详细方法与结果见安全报告 reflection.ai/beam-safety-report。
- 上报通道：reports@reflection.ai。

## 7. k2 视角：为什么这模型值得一个做 agent runtime 的人盯着

1. **23B active = 推理账单小**：MoE 稀疏激活，per-token 算力只跟 23B 走。自建 agent 集群时，推理成本是按激活参数算的，不是按 501B。
2. **Apache-2.0 + 不禁蒸馏 = 可拥有的底座**：能微调、能量化、能蒸馏、能合并——这是"把模型变成自家 plumbing"的前提。DeepSeek 时代是"便宜的 API"，Beam 给的是"便宜的自有"。
3. **为 agent loop 优化**：省 3x 推理 token + MCP 原生 + terminal 强——multi-step 任务里，这三项直接决定单任务成本。
4. **部署现实**：权重 250–500GB，仍是服务器级；真正的个人/小团队可用，要等社区蒸馏与量化版本。先拿 API/云上版本跑评测，再决定要不要自建。

## 8. 使用前需了解的事实

- 权重可公开下载（HF 四个官方仓库），Apache-2.0，商用与研究均可。
- 技术报告尚未发布——训练规模数字（23.8T tokens、92.3% goodput、1 亿 rollouts）为公司口径。
- 全部基准为官方自测（temp=0.7, top_p=0.99），独立第三方评测 pending。
- "3–4x 更少推理算力"是公司估算，非实测美元成本；发布前先复现再信。
- Nvidia 加码/收购为 FT 爆料的早期谈判，双方未评论。
- DFlash2 变体仓库在 HF 上存在，但模型卡未解释其含义。

## 未核实 / 开放问题

- 技术报告何时发布；训练细节（数据配比、RL 算法）以报告为准。
- 独立评测（LMArena、人工 blind test）中的真实排位。
- DFlash2 变体的具体含义与推荐用途。
- Nvidia 收购/加码谈判的后续。
- 中文能力：官方称"主要为英文开发"，中文基准未披露。
- 蒸馏条款：模型卡称不禁止，但"用输出训练其他模型"的法律边界仍以律师意见为准。

---

*独立第三方调研，与 Reflection AI 无关。数字为 2026-10-17 快照，持续变化中。*
