# 遗留数据脚本治理工作流插件

本仓库包含一个 Codex 插件，用于辅助改造遗留数据处理脚本。

插件采用“工作流编排层 + 可复用技能节点”的结构。工作流负责协调改造过程，每个技能负责一个聚焦、可审阅、可复用的单点操作。

插件支持单代理和多代理两种使用方式。小型任务可以由主代理顺序执行；仓库较大、上下文较多或需要独立审查时，可以先使用子代理委派规划节点，将入口梳理、下游契约、旧行为基线、兼容性审查等任务交给子代理并行产出工件，再由主代理统一收敛。

## 插件位置

插件路径：

```text
plugins/legacy-data-modernization-workflow
```

插件市场路径：

```text
.agents/plugins/marketplace.json
```

子代理角色配置：

```text
plugins/legacy-data-modernization-workflow/agents/subagents.yaml
```

每个技能都包含自己的展示和默认调用配置：

```text
plugins/legacy-data-modernization-workflow/skills/<技能标识>/agents/openai.yaml
```

## 技能节点

- `legacy-data-modernization-workflow`：编排完整端到端流程。
- `subagent-delegation-planner`：判断哪些节点适合交给子代理，并生成任务书和回收规则。
- `repo-entrypoint-mapper`：梳理脚本入口、运行参数、配置、调度任务和调用路径。
- `consumer-contract-mapper`：识别下游消费者和输出契约。
- `legacy-behavior-characterizer`：捕获基准输出和特征测试。
- `dirty-data-case-designer`：设计覆盖缺失交易日、重复行、空字段、多股票输入和行业聚合的最小 CSV 样例。
- `refactor-slicer`：把功能扩展和重构拆成安全的实现切片。
- `compatibility-reviewer`：检查结构契约、文件、格式、排序、精度和运行行为的兼容风险。
- `gray-release-planner`：规划影子运行、双写、灰度、监控和回滚。

## 使用场景

当需要扩展遗留数据脚本，同时不能破坏存量用户或下游报表时，使用完整工作流。

当只需要某个单点能力时，可以单独调用对应技能。例如：只审查一次输出结构契约变更，或只设计一组脏数据测试样例。

当需要处理大仓库或复杂上下文时，可以先调用 `subagent-delegation-planner`。子代理只产出可审阅工件，不负责最终上线、回滚或破坏性变更决策。

`agents/subagents.yaml` 记录了建议的子代理角色、适用场景、允许动作、禁止动作和必须返回的工件。主代理可以据此生成具体子代理任务书。

## 上传到仓库

当前目录已经按 Codex 插件市场源的形式组织，可以提交并推送到个人 GitHub 仓库。作为可复用插件源发布时，需要保留：

- `.agents/plugins/marketplace.json`
- `plugins/legacy-data-modernization-workflow`

插件市场条目使用相对插件路径，因此仓库被克隆到本地后，可以作为插件市场源安装。
