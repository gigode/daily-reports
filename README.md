# daily-reports

Claude Code Skill — 全自动日报工作流。

> 一条命令：拉取 AI / 游戏 / 足球三领域最新资讯 → 生成 HTML 深度报告 → 发布到 GitHub Pages → 保留历史归档。

## 触发词

| 中文 | 英文 |
|---|---|
| 更新网站 | update reports |
| 更新日报 | publish reports |
| 更新报告 | refresh daily reports |
| 发布今天的报告 | |

## 工作流

```
用户说"更新网站"
    ↓
📡 并行拉取 AI + 游戏 + 足球 数据
    ↓
📝 生成三份 HTML 报告（各约 30 条，每条 300-500 字深度解读）
    ↓
📂 复制到 GitHub Pages 仓库 + 更新归档页
    ↓
🚀 Git commit & push → 线上生效
```

## 数据源

| 领域 | 数据来源 |
|---|---|
| 🤖 AI | [AI HOT](https://aihot.virxact.com) REST API（精选条目） |
| 🎮 游戏 | WebSearch 实时搜索（IGN / Game Developer / KeenGamer 等） |
| ⚽ 足球 | WebSearch 实时搜索（ESPN / BBC Sport / 懂球帝 / 直播吧 等） |

## 安装

将 `SKILL.md` 复制到你的 Claude Code skills 目录：

```bash
mkdir -p ~/.claude/skills/daily-reports
cp SKILL.md ~/.claude/skills/daily-reports/
```

重启 Claude Code 即可生效。

## 前置要求

- Claude Code（支持 Skill + WebSearch + WebFetch）
- GitHub CLI（`gh`）已登录
- 目标 GitHub Pages 仓库已 clone 到工作目录

## 部署的网站

报告发布到 GitHub Pages，例如：

- 🏠 https://gigode.de/
- 📑 https://gigode.de/reports/
- 🤖 https://gigode.de/reports/2026-06-05/ai.html
- 🎮 https://gigode.de/reports/2026-06-05/game.html
- ⚽ https://gigode.de/reports/2026-06-05/football.html

每条历史报告永久保留，逐日累积。

## 费用

每次运行约消耗 8-12 万 token。按 DeepSeek 当前价格约为 **¥0.7-1.5/次**。

## 技能家族

| Skill | 功能 |
|---|---|
| [aihot](https://github.com/gigode/aihot-skill) | AI 资讯快速简报（Markdown） |
| [ai-report](https://github.com/gigode/ai-report-skill) | AI 行业 HTML 报告 |
| [gamehot](https://github.com/gigode/gamehot-skill) | 游戏资讯简报 |
| [footballhot](https://github.com/gigode/footballhot-skill) | 足球资讯简报 |
| **daily-reports** | **三领域全套日报 + 网站发布** |

## License

MIT
