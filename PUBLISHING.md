# 发布到个人 GitHub 仓库

本仓库已经按 Codex 插件市场源的结构组织，可以推送到个人 GitHub 仓库。

## 必须保留的文件

仓库中需要保留以下路径：

```text
.agents/plugins/marketplace.json
plugins/legacy-data-modernization-workflow/.codex-plugin/plugin.json
plugins/legacy-data-modernization-workflow/skills/*/SKILL.md
```

## 发布前校验

校验单个技能：

```powershell
python -X utf8 C:\Users\Lenovo\.codex\skills\.system\skill-creator\scripts\quick_validate.py plugins\legacy-data-modernization-workflow\skills\legacy-data-modernization-workflow
```

校验插件整体：

```powershell
python -X utf8 C:\Users\Lenovo\.codex\skills\.system\plugin-creator\scripts\validate_plugin.py plugins\legacy-data-modernization-workflow
```

## 推送到 GitHub

创建一个空的 GitHub 仓库后，将本目录作为仓库根目录推送。

示例：

```powershell
git init
git add .
git commit -m "feat: 添加遗留数据脚本治理工作流插件"
git branch -M main
git remote add origin https://github.com/<owner>/<repo>.git
git push -u origin main
```

## 作为 Codex 插件市场安装

仓库克隆到本地后，将仓库中的插件市场文件添加为 Codex 插件市场源：

```powershell
codex plugin marketplace add <克隆后的仓库路径>\.agents\plugins\marketplace.json
```

然后在 Codex 中从该插件市场安装 `legacy-data-modernization-workflow`。

## 推荐仓库描述

面向遗留数据处理脚本安全改造的 AI 辅助 Codex 插件，覆盖入口识别、旧行为刻画、兼容性审查和灰度发布规划。
