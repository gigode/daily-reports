# daily-reports

Claude Code skill — 全自动日报工作流，一条命令更新全套网站。

## 功能

说"**更新网站**"自动完成：

1. 并行拉取 **6 大领域**最新资讯
2. 生成 **6 份 HTML 深度报告**（每条约 300-500 字解读，可展开卡片式布局）
3. 发布到 **GitHub Pages**，历史报告自动归档

## 覆盖领域

| 领域 | 数据源 | 报告条数 |
|------|--------|---------|
| 🤖 AI | [aihot.virxact.com](https://aihot.virxact.com) API | ~22 条 |
| 🎮 游戏 | Steam / LoLPedia / Sky Sports / PC Gamer | ~30 条 |
| ⚽ 足球 | Sky Sports / SI / ESPN | ~29 条 |
| 🔬 芯片 | Investing.com / TechCrunch / Tom's Hardware | ~26 条 |
| 💰 股权投资 | TechCrunch / Investing / Yahoo Finance | ~26 条 |
| 🌍 全球宏观 | yfinance + FRED API + Investing.com | ~22 条 |

## 安装

```bash
mkdir -p ~/.claude/skills/daily-reports
cp SKILL.md ~/.claude/skills/daily-reports/
```

重启 Claude Code 即可生效。

## 前置要求

- **Claude Code**（支持 Skill + WebFetch + Bash）
- **Python** (yfinance) — 拉取宏观市场数据（ETF/指数/商品/外汇）
- **FRED API Key** — 从 [fred.stlouisfed.org](https://fred.stlouisfed.org/docs/api/api_key.html) 免费获取，设为环境变量 `FRED_API_KEY`
- **Node.js** — JSON 解析和文件处理
- **GitHub CLI**（`gh`）已登录
- 目标网站仓库 `gigode.github.io` 已 clone 到本地

## 触发词

"更新网站" / "更新日报" / "更新报告" / "发布今天的报告" / "刷新日报网站" / "update reports" / "publish reports"

## 报告示例

访问 [gigode.de/reports](https://gigode.de/reports/) 查看历史报告。

## 每次运行成本

约 8-15 万 token，按 DeepSeek 当前价格 ≈ ¥1-2/次。

## 相关技能

| Skill | 功能 |
|-------|------|
| [aihot](https://github.com/gigode/aihot-skill) | AI 资讯快速简报（Markdown） |
| [ai-report](https://github.com/gigode/ai-report-skill) | AI 行业 HTML 报告 |
| [gamehot](https://github.com/gigode/gamehot-skill) | 游戏资讯简报 |
| [footballhot](https://github.com/gigode/footballhot-skill) | 足球资讯简报 |
| [chip-report](https://github.com/gigode/chip-report-skill) | 芯片行业日报 |
| [equity-report](https://github.com/gigode/equity-report-skill) | 股权投资日报 |
| **daily-reports** | **6 领域全套日报 + 网站发布** |

## License

MIT
