# 交付与发布说明

本文说明这个笔试答案如何以 GitHub 仓库和 Codex 插件的形式交付、复现和安装。

## 交付目标

本项目不是普通脚本代码提交，而是一套可复用的“遗留数据脚本治理工作流”插件。它面向题目中描述的真实工程场景：

- 旧 Python 脚本长期运行。
- 文档不足、测试不足。
- 下游报表依赖旧输出。
- 新需求要求批量股票、行业输出和停牌缺失数据处理。
- 改造过程必须可验证、可灰度、可回滚。

因此，本仓库交付的是一套可以被使用方安装、调用和扩展的工作流插件，而不是一次性说明文。

## GitHub 仓库

目标仓库：

```text
https://github.com/taroandcandy/10jqka-project
```

仓库根目录需要保留以下文件：

```text
README.md
WORKFLOW.md
PLAN.md
PUBLISHING.md
.agents/plugins/marketplace.json
plugins/legacy-data-modernization-workflow/.codex-plugin/plugin.json
plugins/legacy-data-modernization-workflow/agents/subagents.yaml
plugins/legacy-data-modernization-workflow/skills/*/SKILL.md
plugins/legacy-data-modernization-workflow/skills/*/agents/openai.yaml
```

其中：

- `README.md` 面向出题人解释方案价值、题目对应关系和插件结构。
- `WORKFLOW.md` 描述完整工作流、门禁、单代理和多代理模式。
- `PLAN.md` 记录实施计划和后续扩展步骤。
- `.agents/plugins/marketplace.json` 让仓库可以作为 Codex 插件市场源。
- `plugin.json` 是插件清单。
- `skills/*/SKILL.md` 是每个节点的标准技能定义。
- `agents/subagents.yaml` 是 subagent 角色配置。
- `skills/*/agents/openai.yaml` 是每个技能的中文展示和默认调用配置。

## 发布前校验

在仓库根目录执行以下命令。

校验所有技能：

```powershell
Get-ChildItem -LiteralPath 'plugins\legacy-data-modernization-workflow\skills' -Directory | ForEach-Object {
  python -X utf8 'C:\Users\Lenovo\.codex\skills\.system\skill-creator\scripts\quick_validate.py' $_.FullName
}
```

校验插件整体：

```powershell
python -X utf8 'C:\Users\Lenovo\.codex\skills\.system\plugin-creator\scripts\validate_plugin.py' 'plugins\legacy-data-modernization-workflow'
```

说明：

- `-X utf8` 用于确保 Windows 环境能正确读取中文 `SKILL.md`。
- 校验通过表示技能 frontmatter、目录结构和插件清单满足 Codex 插件基本要求。

## 推送到 GitHub

首次发布到空仓库：

```powershell
git init
git add .
git commit -m "feat: 添加遗留数据脚本治理工作流插件"
git branch -M main
git remote add origin git@github.com:taroandcandy/10jqka-project.git
git push -u origin main
```

后续版本更新：

```powershell
git add .
git commit -m "docs: 优化交付说明"
git push
```

如果当前终端没有 GitHub SSH 权限，需要先配置 SSH key，或在已经能访问该仓库的终端里执行推送。

## 安装为 Codex 插件

使用方克隆仓库后，将仓库内的插件市场文件添加到 Codex：

```powershell
codex plugin marketplace add <克隆后的仓库路径>\.agents\plugins\marketplace.json
```

然后在 Codex 中安装：

```text
legacy-data-modernization-workflow
```

插件安装后，可以直接调用总控技能：

```text
使用 $legacy-data-modernization-workflow，为一个遗留 Python CSV 股票移动平均线脚本设计安全改造工作流。
```

也可以单独调用某个节点：

```text
使用 $compatibility-reviewer，审查这个输出结构契约变更是否会影响下游报表。
```

## 面向出题人的复现方式

评审不需要真的接入生产股票数据，也不需要准备一个完整旧系统。可以通过以下方式判断方案是否完整：

1. 查看 `README.md` 是否解释清楚题目背景和方案主线。
2. 查看 `WORKFLOW.md` 是否覆盖理解、行为刻画、扩展、重构、验证、灰度和回滚。
3. 查看各 `SKILL.md` 是否都有输入、执行步骤、输出格式和验收标准。
4. 查看 `agents/subagents.yaml` 是否把耗时、上下文重的节点配置成可委派 subagent。
5. 运行发布前校验命令，确认插件结构有效。

## 当前版本说明

当前版本已经具备：

- 1 个总控工作流技能。
- 1 个 subagent 委派规划技能。
- 7 个工作流节点技能。
- 7 个 subagent 角色配置。
- 每个技能的中文展示配置。
- 中文 README、WORKFLOW、PLAN 和 PUBLISHING。

这说明答案不只是“我会怎么做”的文字描述，而是一套可以被安装、复用、迭代的 AI 辅助工作流资产。
