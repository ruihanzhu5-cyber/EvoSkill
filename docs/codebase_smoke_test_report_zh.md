# EvoSkill 原仓库端到端跑通验收报告

## 结论

EvoSkill 原仓库的核心代码可以在 **OpenCode 1.18.1 + DeepSeek V4 Flash** 环境下完成端到端运行。整个验证过程没有修改 `src/` 核心源码，只增加了 DeepSeek smoke 配置和 OpenCode provider 兼容配置。

成功覆盖的完整链路为：

```text
配置与数据加载
→ OpenCode 调用 DeepSeek
→ Agent 读取 Treasury Bulletin 文档
→ 结构化答案解析
→ 本地评分
→ 失败样本收集
→ Proposer 生成技能提案
→ Skill Builder 生成候选技能
→ 候选程序验证
→ Frontier 更新/淘汰
→ 报告与反馈历史落盘
→ 命令正常退出
```

这证明仓库可以作为后续复现和架构优化的 codebase。需要注意：DeepSeek V4 并非开箱即用，必须关闭思考模式，并禁止无人值守运行中的交互式提问。

## 环境

| 项目 | 版本/配置 |
|---|---|
| 操作环境 | WSL 2 / Ubuntu 26.04 |
| 项目 Python | 3.12.13 |
| uv | 0.11.28 |
| OpenCode | 1.18.1 |
| 模型 | `deepseek/deepseek-v4-flash` |
| 数据 | 官方 OfficeQA 10 题样例 + 9 份 Treasury Bulletin 文档 |
| 演化模式 | `skill_only` |
| 最大迭代 | 1 |
| 并发 | 1 |
| 重试 | 1 |
| 评分器 | 本地 `multi_tolerance` |

## 基础验证

### 依赖和 CLI

- `uv sync` 成功解析 504 个包并检查 230 个已安装包。
- `import src` 成功。
- `evoskill --help` 成功加载所有主要命令。

### 单元测试

```text
663 passed, 1 failed, 2 warnings
```

唯一失败是 OfficeQA OpenRouter 示例配置与测试断言不一致：测试期望 `opencode`，实际示例配置为 `openhands`。这不是核心执行逻辑失败。

## 最小评估结果

第一次 DeepSeek `eval` 未产生答案。OpenCode 日志给出明确原因：

```text
Thinking mode does not support this tool_choice
```

DeepSeek V4 默认开启思考模式，而 EvoSkill 的结构化输出需要强制工具选择。通过项目级 `opencode.json` 设置以下选项后，重复评估成功：

```json
{
  "provider": {
    "deepseek": {
      "models": {
        "deepseek-v4-flash": {
          "options": {
            "thinking": {
              "type": "disabled"
            }
          }
        }
      }
    }
  }
}
```

修复后，2 道验证题均返回非空结构化答案并完成本地评分。答案是否正确不作为 smoke test 的唯一标准；关键证据是模型调用、文档工具、结构化解析和评分链路全部完成。

## 完整演化结果

第一次 `run` 在验证阶段遇到 OpenCode 的交互式 `question` 请求。无人值守运行无法回答，因而挂起。增加以下权限后解决：

```json
{
  "permission": {
    "question": "deny"
  }
}
```

一次只抽取 easy 类别时，两道训练题全部通过，EvoSkill 按设计跳过技能生成。为了自然触发失败而不篡改数据，将 `failure_samples` 从 1 调整为 2，使训练批次覆盖 easy 和 hard 两个类别。

最终运行结果：

| 阶段 | 结果 |
|---|---|
| 基线验证 | 2/2，通过，得分 100% |
| 训练抽样 | 3 题（2 easy、1 hard） |
| 失败收集 | 1 题失败 |
| Proposer | 成功生成 `structured-table-query` 提案 |
| Skill Builder | 成功生成候选技能 |
| 候选验证 | 2 道验证题，得分 50% |
| Frontier 决策 | 候选低于基线，被正确淘汰 |
| 最佳程序 | `base`，100% |
| 退出状态 | 正常完成 |

技能没有被保留并不表示运行失败。候选技能被完整生成、验证，并按照 Frontier 规则正确淘汰，证明演化控制流有效。

## 生成的证据

- `program/base` 分支
- `frontier/base` 标签
- `.evoskill/feedback_history.md` 中的完整技能提案和淘汰结果
- `.evoskill/reports/run-2026-07-15-230606.md` 运行报告
- OpenCode 会话和工具调用日志

## Token 与费用

OpenCode 在本次会话期间的累计统计（包含初始连接测试、失败尝试、重跑和最终成功运行）为：

| 指标 | 数值 |
|---|---:|
| Sessions | 23 |
| Messages | 187 |
| Input | 760.1K |
| Output | 32.0K |
| Cache Read | 5.5M |
| OpenCode 估算总成本 | `$0.13` |

EvoSkill 最终报告只记录 `$0.0009`，与 OpenCode 累计统计存在明显差异。这表明当前 EvoSkill 成本追踪没有覆盖所有 OpenCode 子会话或在提前停止路径上漏计费用。正式实验应同时记录 DeepSeek 控制台和 OpenCode `stats`，不能只依赖 EvoSkill 报告。

## 发现的风险与后续优化点

1. **DeepSeek V4 思考模式兼容性**：结构化输出的 `tool_choice` 与默认思考模式冲突。
2. **无人值守交互风险**：OpenCode `question` 工具会让批处理等待到超时。
3. **成本漏记**：EvoSkill 报告成本明显低于 OpenCode 会话统计。
4. **Agent 工作区副作用**：一次运行中 Agent 在数据目录创建了临时 Python 计算脚本；测试清理时已删除。正式实验应监测非技能目录写入。
5. **示例测试漂移**：OpenRouter 示例使用 OpenHands，而单元测试仍期望 OpenCode。
6. **小样本不代表准确率**：2 道验证题得到 100% 不能用于证明模型质量，仅证明控制流程跑通。

## 核心源码完整性

验证结束时执行：

```bash
git diff reproduction-baseline -- src
```

输出为空，说明 `src/` 核心源码相对原始恢复点没有任何修改。本报告所述成功运行来自原仓库代码、运行配置和 provider 兼容配置，而非修改算法实现。

## 最终判断

| 判断项 | 结果 |
|---|---|
| 仓库基础环境可安装 | 通过 |
| 核心测试总体健康 | 通过（663/664） |
| DeepSeek/OpenCode 工具链 | 通过 |
| 最小评估链路 | 通过 |
| 失败收集 | 通过 |
| Proposer | 通过 |
| Skill Builder | 通过 |
| 候选验证 | 通过 |
| Frontier 决策 | 通过 |
| 正常结束与报告 | 通过 |
| 核心源码未修改 | 通过 |

**最终结论：EvoSkill 原仓库可以跑通，并可作为 DeepSeek 复现与后续架构优化的 codebase。**
