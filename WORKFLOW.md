# 工作流编排结构

## 架构

该工作流刻意拆成两层：

- 编排层：`legacy-data-modernization-workflow`。
- 节点层：多个聚焦的技能，每个技能产出可审阅工件。

编排技能负责决定当前应运行哪个节点，检查节点产物是否足够，并在兼容性门禁缺失时阻止高风险推进。

## 节点链路

| 顺序 | 节点 | 主要产物 | 下游消费方 |
| --- | --- | --- | --- |
| 1 | `repo-entrypoint-mapper` | 入口和调用路径图 | `consumer-contract-mapper`、`legacy-behavior-characterizer` |
| 2 | `consumer-contract-mapper` | 输出消费者契约表 | `legacy-behavior-characterizer`、`compatibility-reviewer` |
| 3 | `legacy-behavior-characterizer` | 基准文件和特征测试 | `refactor-slicer`、`compatibility-reviewer` |
| 4 | `dirty-data-case-designer` | 脏数据样例和预期行为 | `refactor-slicer`、测试设计 |
| 5 | `refactor-slicer` | 有序安全实现切片 | 实现与审查 |
| 6 | `compatibility-reviewer` | 兼容性问题和必要缓解措施 | 发布准备 |
| 7 | `gray-release-planner` | 灰度发布和回滚计划 | 交付 |

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
