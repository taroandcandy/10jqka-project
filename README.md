# 遗留数据脚本治理工作流插件

本仓库是对“遗留数据处理脚本扩展与重构”笔试题的一种工程化回答：不把答案停留在一次性论述，而是把“理解 -> 行为刻画 -> 扩展 -> 重构 -> 验证 -> 灰度交付”沉淀为一个可复用的 Codex 插件。

题目中的场景是：团队有一个运行多年的 Python 脚本，每天从 CSV 读取单只股票数据并计算移动平均线；现在要支持多股票批处理、按行业输出、处理停牌缺失数据，同时不能破坏下游报表依赖的旧输出格式。这个仓库给出的核心思路是：**先保护旧行为，再扩展新能力；先建立可验证工作流，再让 AI 参与每个可控节点。**

## 给出题人的阅读路径

建议按以下顺序审阅：

1. [WORKFLOW.md](WORKFLOW.md)：看整体工作流如何组织，以及为什么采用“编排层 + 技能节点 + subagent”的结构。
2. [PLAN.md](PLAN.md)：看实施计划、阶段门禁、subagent 增强方案和后续落地顺序。
3. `plugins/legacy-data-modernization-workflow/skills/*/SKILL.md`：看每个节点是否具备标准技能格式、清晰输入输出和验收标准。
4. [PUBLISHING.md](PUBLISHING.md)：看该方案如何作为插件发布到 GitHub，并被使用方安装复用。

## 答案亮点

这个方案不是让 AI 一次性重写旧脚本，而是把 AI 放进受控工程流程中：

- **入口先行**：先定位脚本入口、运行参数、调度方式和配置来源。
- **契约先行**：先识别下游报表和输出契约，再决定是否能改字段、格式和路径。
- **基线先行**：先用基准输出和特征测试锁住旧行为，再扩展新功能。
- **小步演进**：把重构和功能变化拆成可审查、可回滚的小切片。
- **兼容审查**：独立检查字段顺序、日期格式、数值精度、空值表示、排序和退出行为。
- **灰度交付**：通过影子运行、双写、小范围灰度、指标监控和回滚条件降低上线风险。
- **subagent 协作**：把耗时、上下文重、可独立验收的节点交给 subagent，主代理保留最终判断权。

## 插件结构

```text
.agents/plugins/marketplace.json
plugins/legacy-data-modernization-workflow/
  .codex-plugin/plugin.json
  agents/subagents.yaml
  skills/
    legacy-data-modernization-workflow/
      SKILL.md
      agents/openai.yaml
    subagent-delegation-planner/
      SKILL.md
      agents/openai.yaml
    repo-entrypoint-mapper/
      SKILL.md
      agents/openai.yaml
    consumer-contract-mapper/
      SKILL.md
      agents/openai.yaml
    legacy-behavior-characterizer/
      SKILL.md
      agents/openai.yaml
    dirty-data-case-designer/
      SKILL.md
      agents/openai.yaml
    feature-extension-planner/
      SKILL.md
      agents/openai.yaml
    refactor-slicer/
      SKILL.md
      agents/openai.yaml
    regression-validation-planner/
      SKILL.md
      agents/openai.yaml
    compatibility-reviewer/
      SKILL.md
      agents/openai.yaml
    gray-release-planner/
      SKILL.md
      agents/openai.yaml
    ai-usage-recorder/
      SKILL.md
      agents/openai.yaml
```

## 技能节点

- `legacy-data-modernization-workflow`：总控技能，编排端到端治理流程。
- `subagent-delegation-planner`：判断哪些节点适合交给 subagent，并生成任务书和回收规则。
- `repo-entrypoint-mapper`：梳理脚本入口、运行参数、配置、调度任务和调用路径。
- `consumer-contract-mapper`：识别下游消费者和输出契约。
- `legacy-behavior-characterizer`：捕获基准输出和特征测试。
- `dirty-data-case-designer`：设计覆盖停牌缺失交易日、重复行、空字段、多股票输入和行业聚合的最小 CSV 样例。
- `feature-extension-planner`：设计多股票批处理、行业输出、停牌缺失数据处理和旧输出兼容方案。
- `refactor-slicer`：把重构拆成安全的实现切片。
- `regression-validation-planner`：设计单元、基准文件、结构契约、下游冒烟、新旧差异和性能验证。
- `compatibility-reviewer`：检查结构契约、文件、格式、排序、精度和运行行为的兼容风险。
- `gray-release-planner`：规划影子运行、双写、灰度、监控和回滚。
- `ai-usage-recorder`：生成 AI_USAGE.md，记录 AI 参与过程、关键提示词、验证方式和拒绝大范围重写的原因。

## 与题目要求的对应关系

| 题目要求 | 对应实现 |
| --- | --- |
| 定位入口、调用方和输出消费者 | `repo-entrypoint-mapper`、`consumer-contract-mapper` |
| 建立旧行为基线 | `legacy-behavior-characterizer` |
| 分阶段流程设计 | `legacy-data-modernization-workflow`、`WORKFLOW.md`、`PLAN.md` |
| 至少 3 条关键提示词 | 每个技能的 `SKILL.md` 和 `agents/openai.yaml` 都给出调用语义与默认提示 |
| 扩展多股票、行业输出和停牌处理 | `feature-extension-planner`、`dirty-data-case-designer` |
| 核心模块拆分和伪代码思路 | `refactor-slicer` 支撑后续 `DESIGN.md` |
| 测试矩阵和下游兼容验证 | `regression-validation-planner`、`compatibility-reviewer` |
| 灰度指标和回滚条件 | `gray-release-planner` |
| 记录 AI 使用过程和拒绝大范围重写 | `ai-usage-recorder` |

## subagent 设计

插件支持单代理和多代理两种运行方式。

小型任务可以由主代理顺序执行。仓库较大、上下文较多或需要独立审查时，可以先调用 `subagent-delegation-planner`，再把入口梳理、下游契约、旧行为基线、兼容性审查等任务交给 subagent 并行处理。

subagent 角色定义在：

```text
plugins/legacy-data-modernization-workflow/agents/subagents.yaml
```

subagent 只产出可审阅工件，不负责最终上线、回滚或破坏性变更决策。主代理负责汇总冲突、统一口径，并决定是否进入下一阶段门禁。

## 验证状态

当前插件已通过本地校验：

- 12 个技能均通过 `quick_validate.py`。
- 插件整体通过 `validate_plugin.py`。
- 技能正文、README、WORKFLOW 和 PUBLISHING 已中文化，便于国内面试场景阅读。

## 仓库发布

该插件已经按 Codex 插件市场源组织，可发布到个人 GitHub 仓库：

```text
https://github.com/taroandcandy/10jqka-project
```

使用方克隆仓库后，可以通过 `.agents/plugins/marketplace.json` 安装该插件。详细步骤见 [PUBLISHING.md](PUBLISHING.md)。
