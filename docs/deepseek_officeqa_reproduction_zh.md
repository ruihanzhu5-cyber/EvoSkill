# EvoSkill × DeepSeek：OfficeQA 低预算复现与架构优化方案

> 本文整理自本次讨论，目标是在约 100 元预算内，用 DeepSeek 复现 EvoSkill 的核心流程，并以此定位和优化架构问题，而不是追求论文全部实验数值的严格复现。

## 1. 实验目标与边界

原论文的 OfficeQA 和 SealQA 实验使用 Claude Code + Claude Opus 4.5。替换为 DeepSeek 后，实验应称为：

> 在 DeepSeek 上复现 EvoSkill 方法，并研究其架构优化与跨模型适用性。

它不是论文结果的严格复现，因此不应直接宣称复现了论文中的 OfficeQA 67.9% 或 SealQA 38.7%。本实验更适合回答：

- EvoSkill 能否让 DeepSeek 自动生成有效技能？
- 架构修改能否提高测试准确率？
- 能否以更少 Token、费用和时间获得相同或更好的效果？
- EvoSkill 对 Claude 是否存在隐含依赖？

## 2. 推荐数据集

预算有限时优先选择 **OfficeQA**：

- 可以使用本地确定性评分，不需要额外的 LLM 裁判 API。
- 文档和答案相对固定，比依赖实时网络搜索的 SealQA 更容易控制变量。
- 失败可清楚分为文档检索、表格提取、计算推理和技能触发等类型，适合做架构分析。

完整实验需要 OfficeQA 数据集及 Treasury Bulletin 文档。仓库目前提供的是 10 题示例，适合调试但不适合得出可靠研究结论。

## 3. API、GPU 与软件需求

推荐运行链路：

```text
EvoSkill → OpenCode → DeepSeek API
```

最简方案只需要：

```bash
export DEEPSEEK_API_KEY="你的密钥"
```

不需要：

- Anthropic API
- OpenAI API
- OpenRouter API
- Google API
- Daytona API（只有使用 Daytona 云沙箱时才需要）
- GPU

OpenCode 本身开源免费，费用来自 DeepSeek API。EvoSkill 通过远程 API 调用模型，本地只负责 Python 调度、工具执行、数据处理和 Git 管理，所以普通 CPU 机器即可。Harbor 容器任务可能需要 Docker，但仍不要求 GPU。

密钥应放在环境变量或本机 `~/.bashrc` 中，不要写入源码、`.evoskill/config.toml` 或提交到 Git。

## 4. DeepSeek 与 OpenCode 配置

安装 OpenCode：

```bash
./install.sh --agents opencode
opencode --version
```

运行以下命令创建项目配置：

```bash
uv run evoskill init
```

主要配置文件是：

```text
.evoskill/config.toml
```

推荐先使用 V4 Flash 调试：

```toml
[harness]
name = "opencode"
model = "deepseek/deepseek-v4-flash"
timeout_seconds = 600
max_retries = 1

[evolution]
mode = "skill_only"
iterations = 2
frontier_size = 2
concurrency = 1
no_improvement_limit = 2
failure_samples = 2

[scorer]
type = "multi_tolerance"
```

正式复验可改为：

```toml
model = "deepseek/deepseek-v4-pro"
```

应在 OpenCode 中运行 `/models`，以实际显示的模型 ID 为准。旧模型名 `deepseek-chat` 和 `deepseek-reasoner` 将于北京时间 2026-07-24 23:59 弃用，不建议用于新实验。

仓库鉴权层已经支持 `DEEPSEEK_API_KEY`，通过 OpenCode 执行、使用本地 `multi_tolerance` 评分时通常不需要修改源码。

### LLM 评分器限制

当前通用 `call_llm()` 评分路径尚未直接实现 DeepSeek provider。若配置：

```toml
[scorer]
type = "llm"
provider = "deepseek"
```

则需要补充 `src/cli/shared.py` 中的 provider 推断、模型名前缀处理和 OpenAI 兼容客户端。OfficeQA 推荐直接使用 `multi_tolerance`，可避免这项改动和额外 API 成本。

## 5. 专业名词简释

- **API**：调用远程 AI 模型的网络入口。
- **API Key**：访问 API 的密钥，同时用于身份和计费，不能公开。
- **Token**：模型处理文本的计费单位；问题、提示词、文档、工具结果、历史消息和模型输出都会消耗 Token。
- **输入 Token**：发送给模型阅读的所有内容。
- **输出 Token**：模型生成的答案、思考内容和工具调用指令。
- **缓存命中**：请求与过去请求存在可复用的完整前缀，重复输入按更低价格计费。
- **上下文**：模型单次请求能够看到的全部内容。
- **思考模式**：模型先分析、调用工具和检查，再给最终答案；效果可能更好，但输出 Token 更多。
- **Tool Calls**：模型调用文件读取、搜索、Shell、Python 或技能等工具。
- **Harness**：EvoSkill 与具体 Agent 运行时之间的适配层；本方案使用 OpenCode。
- **迭代**：一轮“抽题 → 找失败 → 分析原因 → 生成技能 → 验证技能”。
- **Epoch**：训练样本平均被完整使用一遍。
- **训练集**：用于发现失败并生成技能。
- **验证集**：用于决定候选技能是否保留。
- **测试集**：只在最后评估泛化效果，不能用于选择技能。
- **Proposer**：分析失败并提出技能创建或修改建议的 Agent。
- **Skill Builder**：把建议实现成 `SKILL.md` 和辅助脚本的 Agent。
- **Frontier**：当前得分最高的若干技能方案集合。
- **Scorer**：判断答案是否正确的评分器。

## 6. Token 与费用估算

根据 2026-07-15 的 DeepSeek 官方价格：

| 模型 | 输入缓存命中 | 输入缓存未命中 | 输出 |
|---|---:|---:|---:|
| V4 Flash | ¥0.02/百万 Token | ¥1/百万 Token | ¥2/百万 Token |
| V4 Pro | ¥0.025/百万 Token | ¥3/百万 Token | ¥6/百万 Token |

一次论文风格的 OfficeQA 10% 训练配置，粗略估计约有 429 个 Agent 会话：基线验证、训练题执行、Proposer、Skill Builder、候选验证及最终测试。由于每题读取的文档和工具轨迹不同，只能给出区间：

- 输入约 1000 万～5000 万 Token
- 输出及思考约 100 万～500 万 Token

全部按缓存未命中估算：

| 模型 | 一次 10% 演化实验估算 |
|---|---:|
| V4 Flash | ¥12～¥60 |
| V4 Pro | ¥36～¥180 |

缓存会降低输入成本，但缓存是尽力而为，预算不应假设 100% 命中。实际消耗必须以 API 返回的以下字段为准：

- `prompt_cache_hit_tokens`
- `prompt_cache_miss_tokens`
- 输出 Token

## 7. 100 元预算的推荐复现方案

不要直接复现论文全部 5%、10%、15% 配置。建议固定选择 40～60 道 OfficeQA；以 50 道为例：

- 训练集：15 道
- 验证集：15 道
- 测试集：20 道

测试集必须固定，并且不能参与技能生成。

预算分配建议：

| 阶段 | 模型 | 预算 |
|---|---|---:|
| 环境与接口调试 | V4 Flash | ¥5 |
| 原始架构基线 | V4 Flash | ¥15 |
| 2～3 个架构改进实验 | V4 Flash | ¥45 |
| 最佳方案复验 | V4 Pro | ¥25 |
| 重试和异常备用 | — | ¥10 |

实验矩阵：

| 实验 | 内容 | 模型 | 迭代 |
|---|---|---|---:|
| A | 无技能基线 | Flash | 0 |
| B | 原始 EvoSkill | Flash | 4 |
| C | 修改后的 EvoSkill | Flash | 4 |
| D | 最优方案复验 | Pro | 2～4 |

A、B、C 必须使用相同模型、数据划分、评分规则和迭代预算。每次只修改一个架构变量，否则无法判断是哪项改动带来了效果。

优先优化方向：

1. 压缩无效上下文和失败 trace。
2. 改善失败样本选择，减少重复或低价值提议。
3. 在候选技能进入昂贵验证前，增加本地格式、泄漏和重复检查。

## 8. 完成后能够得到什么

合理的成果不是“复现论文 67.9%”，而是：

- 一套可运行的 DeepSeek 版 EvoSkill 流程。
- 一组自动生成、可以检查和复用的 OfficeQA 技能。
- 无技能、原始 EvoSkill 和优化架构之间的公平对照结果。
- 准确率、Token、费用、耗时、工具调用和失败率数据。
- EvoSkill 在 DeepSeek 上的主要失败模式。
- 一个有证据支持的架构优化方向。
- 固定的数据划分、配置和日志，可在预算增加后扩展到完整数据集。

即使准确率没有提高，也可以形成有价值的结论，例如：某些技能没有被触发、结构化输出不稳定、上下文增长过快，或者原架构对 Claude 的行为存在隐含依赖。

## 9. 失败模式与监测方法

建议给 EvoSkill 增加本地“行车记录仪”，保存为 JSONL/CSV，不调用额外 LLM：

```text
.evoskill/metrics/
├── run-001-events.jsonl
├── run-001-questions.csv
├── run-001-skills.csv
└── run-001-summary.json
```

每道题至少记录：

- `run_id`、`iteration`、`split`、`question_id`、`category`
- 预测答案、正确答案和得分
- 输入、缓存命中、缓存未命中和输出 Token
- 费用、耗时、重试次数和解析是否成功
- 工具、搜索词、读取文件和提取的证据片段
- 可用技能、实际触发技能
- 人工或规则标注的错误类型

主要失败模式及信号：

| 失败模式 | 监测信号 |
|---|---|
| 搜索错误文档 | 正确来源已知，但 Agent 从未读取该文件 |
| 读错表格单元格 | 读取了正确文档，但提取值或最终采用值错误 |
| Proposer 生成重复技能 | 新旧技能名称、描述或正文相似度过高 |
| 技能过于针对单题 | 技能含训练题独有答案、数字、长句、年份或文件名 |
| 技能存在但没有触发 | 相关题上技能可用，但实际调用次数为零或很少 |
| 验证集错误保留技能 | 验证集上升但最终测试集下降，泛化差距过大 |
| 反馈历史膨胀 | Proposer 输入 Token 连续增长，反馈历史占比过高 |
| 技能收益低于成本 | Proposer、Builder 和验证花费很高，但测试分数无提升 |
| JSON 解析失败 | `invalid_json`、缺字段、枚举错误、截断、空响应或超时 |

第一版只需监测六项：

1. 每道题得分。
2. 输入、输出与缓存 Token。
3. 费用和耗时。
4. JSON 是否首次解析成功。
5. 技能是否被调用。
6. 候选技能带来的验证分数变化。

对固定的 20 道测试题，可人工标注：

```text
retrieval       搜错或没找到文档
extraction      找到文档但取错值
reasoning       数据正确但计算错误
skill_not_used  有相关技能但没触发
skill_bad       触发技能但仍失败
parse_error     输出解析失败
```

## 10. 架构优化评估指标

实验结束后至少生成以下对照表：

| 指标 | 原始 EvoSkill | 优化后 |
|---|---:|---:|
| 测试准确率 |  |  |
| 总输入 Token |  |  |
| 总输出 Token |  |  |
| 总费用 |  |  |
| 平均耗时 |  |  |
| 搜索错误率 |  |  |
| 单元格提取错误率 |  |  |
| 重复技能率 |  |  |
| 技能触发率 |  |  |
| JSON 首次解析失败率 |  |  |
| 验证—测试泛化差距 |  |  |

推荐增加两个成本指标：

```text
单位提升成本 = 总费用 / 测试准确率提升百分点
单位提升 Token = 总 Token / 测试准确率提升百分点
```

这样可以回答：哪种架构能以更低成本产生更有效的技能？

## 11. 推荐执行顺序

1. 准备 DeepSeek Key、OpenCode、OfficeQA 数据和文档。
2. 用 3～5 道题、1 次迭代确认接口、工具和评分器全部正常。
3. 固定 40～60 道题及训练/验证/测试划分。
4. 运行无技能基线并记录完整指标。
5. 用 Flash 运行 4 次原始 EvoSkill 迭代。
6. 根据监测结果选择一个失败模式，只修改一个架构变量。
7. 使用相同条件运行优化架构。
8. 用固定测试集统一比较，避免每轮窥视测试集。
9. 仅对最优方案使用 V4 Pro 复验。
10. 汇总准确率、成本、Token、失败类型和技能质量。

最终理想结论是：

> 在固定 OfficeQA 子集上，EvoSkill 能够为 DeepSeek 自动产生技能；所提出的架构改进在相同模型、数据和预算下，提高了测试效果，或在效果相当的情况下显著降低了 Token 与费用。
