# 工作流编排结构

## 架构

该工作流刻意拆成两层：

- 编排层：`legacy-data-modernization-workflow`。
- 节点层：多个聚焦的技能，每个技能产出可审阅工件。

编排技能负责决定当前应运行哪个节点，检查节点产物是否足够，并在兼容性门禁缺失时阻止高风险推进。

## 运行模式

单代理模式适用于小脚本、文件量较少、入口和下游依赖比较明确的场景。主代理按节点链路顺序执行，每个节点产物通过门禁后再进入下一步。

多代理模式适用于仓库较大、调用链分散、下游消费者不明确、测试材料较多或需要独立审查视角的场景。主代理先使用 `subagent-delegation-planner` 生成委派计划，再把适合并行探索的节点交给子代理。

无论是否启用子代理，主代理都保留最终判断权：是否接受破坏性变更、是否上线、是否回滚，以及最终交付口径，不能交给子代理决定。

## 节点链路

| 顺序 | 节点 | 主要产物 | 下游消费方 |
| --- | --- | --- | --- |
| 0 | `subagent-delegation-planner` | 子代理委派计划 | 主代理编排 |
| 1 | `repo-entrypoint-mapper` | 入口和调用路径图 | `consumer-contract-mapper`、`legacy-behavior-characterizer` |
| 2 | `consumer-contract-mapper` | 输出消费者契约表 | `legacy-behavior-characterizer`、`compatibility-reviewer` |
| 3 | `legacy-behavior-characterizer` | 基准文件和特征测试 | `refactor-slicer`、`compatibility-reviewer` |
| 4 | `dirty-data-case-designer` | 脏数据样例和预期行为 | `refactor-slicer`、测试设计 |
| 5 | `refactor-slicer` | 有序安全实现切片 | 实现与审查 |
| 6 | `compatibility-reviewer` | 兼容性问题和必要缓解措施 | 发布准备 |
| 7 | `gray-release-planner` | 灰度发布和回滚计划 | 交付 |

## 子代理委派边界

强委派节点：

- `repo-entrypoint-mapper`
- `consumer-contract-mapper`
- `legacy-behavior-characterizer`
- `compatibility-reviewer`
- `gray-release-planner`

受控委派节点：

- `dirty-data-case-designer`
- `refactor-slicer`

不可委派事项：

- 总控编排。
- 上线、回滚和破坏性变更审批。
- 涉及真实生产写入、删除、覆盖旧输出或发布的动作。

子代理产物必须区分已确认事实、推断结论和待确认问题。主代理负责合并冲突、提升兼容性风险优先级，并将结果映射回工作流门禁。

## 子代理配置文件

插件根目录的 `agents/subagents.yaml` 定义了可复用子代理角色：

- 入口探索子代理。
- 下游契约子代理。
- 旧行为基线子代理。
- 脏数据样例子代理。
- 重构切片子代理。
- 兼容性审查子代理。
- 灰度发布子代理。

这些配置用于生成子代理任务书，而不是绕过主代理或自动执行高风险操作。真实运行时是否创建子代理，仍由主代理根据当前任务、上下文规模和用户授权决定。

## 工作流门禁规则

- 在入口和输出消费者完成梳理，或被明确标记为未知前，不开始功能实现。
- 在旧行为被基准文件捕获，或风险例外被记录前，不重构兼容性关键逻辑。
- 不直接修改旧输出结构契约；优先新增输出或使用兼容适配层。
- 在下游兼容验证和回滚条件定义完成前，不进入发布。

## 调用示例

```text
使用 $legacy-data-modernization-workflow，为一个 Python CSV 股票移动平均线脚本设计改造方案。新需求包括批量处理多只股票、行业级输出、处理停牌缺失交易日，同时保留下游报表依赖的旧单股票输出格式。
```

单独调用某个节点：

```text
使用 $compatibility-reviewer，对这个输出结构契约变更做下游兼容性审查。
```

规划子代理协作：

```text
使用 $subagent-delegation-planner，判断这个遗留数据脚本改造任务中哪些节点适合交给子代理，并为每个子代理生成任务书。
```
