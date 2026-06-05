---
name: daily-reports
description: 更新全套日报网站。当用户说"更新网站"、"更新日报"、"更新报告"、"发布今天的报告"、"刷新日报网站"、"update reports"、"publish reports"时触发。自动拉取 AI/游戏/足球三领域最新资讯，生成 HTML 报告，更新 GitHub Pages 网站，保留历史报告。
---

# Daily Reports Skill

全自动日报工作流：拉取数据 → 生成三份 HTML 报告 → 更新 gigode.github.io → 提交推送。一条命令完成全部操作。

## 工作流总览

```
用户说"更新网站"
    ↓
Phase 1: 并行拉取 AI + 游戏 + 足球 数据
    ↓
Phase 2: 生成三份 HTML 报告 → C:\works\
    ↓
Phase 3: 复制到 gigode.github.io 仓库 + 更新 hub 页面
    ↓
Phase 4: Git commit & push → 线上生效
```

## Phase 1: 拉取数据

### 1a. AI 数据（aihot API）

```bash
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36 aihot-skill/0.2.0"
SINCE=$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)
curl -sH "User-Agent: $UA" "https://aihot.virxact.com/api/public/items?mode=selected&since=$SINCE&take=100"
```

结果通常 25-50 条，5 个 category：`ai-models` / `ai-products` / `industry` / `paper` / `tip`。

### 1b. 游戏数据（WebSearch）

并行搜索 5 个类别（中英文双语）：

| 类别 | 搜索词 |
|---|---|
| new-releases | "new game releases today June 2026 Steam latest games" + "新游戏发布 Steam 新游 2026年6月" |
| industry | "gaming industry news June 2026 game company acquisition" + "游戏行业动态 游戏公司 收购 2026" |
| esports | "esports news June 2026 LPL LCK tournament" + "电竞资讯 LPL 赛事 2026年6月" |
| hardware | "gaming hardware news June 2026 GPU console Switch 2" + "游戏硬件 显卡 主机 Switch 2 2026" |
| reviews | "game review recommendation June 2026 best games" + "游戏评测 游戏推荐 2026年6月" |

每类取 top 5-8 条，合并为约 30 条。必须保留每条 URL。

### 1c. 足球数据（WebSearch）

并行搜索 5 个类别（中英文双语）：

| 类别 | 搜索词 |
|---|---|
| matches | "football match results today June 2026 scores" + "足球比赛结果 比分 2026年6月" |
| transfers | "football transfer news June 2026 latest rumors" + "足球转会 最新转会 2026年6月" |
| leagues | "football league news June 2026 Premier League Champions League" + "五大联赛 欧冠 英超 动态 2026" |
| players | "football player news June 2026 injuries awards" + "足球球星 伤病 动态 2026" |
| analysis | "football analysis tactical review June 2026" + "足球深度分析 战术 2026" |

每类取 top 5-8 条，合并为约 30 条。必须保留每条 URL。

### 搜索注意事项

- **日期动态化**：搜索词中的"June 2026"应替换为当前月份和年份
- **去重**：同一事件被多个来源报道的，选最权威的一条
- **时间标注**：搜索结果中能提取到时间的转为北京时间（UTC+8），没有的诚实标注"时间不详"
- **传闻标注**：转会类必须区分"官宣"(✅)和"传闻"(🔹)
- 执行时 WebSearch 可以并行发起（10 个搜索词可一次调用全部发出）

## Phase 2: 生成报告

生成三份独立的 HTML 报告，保存到 `C:\works\`：

- `aihot-report-YYYY-MM-DD.html`
- `gamehot-report-YYYY-MM-DD.html`
- `footballhot-report-YYYY-MM-DD.html`

日期取当前日期（北京时间）。

### 报告 HTML 规范

每份报告必须遵循以下规范：

#### 核心约束
1. **每条独立成卡** — 绝不合并条目到群体卡片
2. **编号贯穿全文** — 全局编号 1-N，不在板块内重新计数
3. **每卡可展开** — 点击标题栏切换 .open 类，展开区显示深度解读
4. **深度解读** — 每条 300-500 字分析（非简单的摘要翻译）
5. **五个板块固定顺序** — 按各自领域 category 分组
6. **保留原始链接** — 每条卡片底部放来源 URL
7. **首条默认展开** — 通过 JS 设置
8. **目录导航** — 板块可点击跳转，显示各板块条数
9. **封面** — 深色渐变背景、总条数/板块数统计
10. **时间转换** — 所有时间显示为北京时间 + 人话格式

#### 设计规范
```css
:root {
  --bg: #f7f8fa; --card-bg: #fff; --text: #111827; --muted: #6b7280;
  --accent: #2563eb; --border: #e5e7eb;
  --radius: 10px;
}
```
- 字体：PingFang SC, Noto Sans SC, Microsoft YaHei, sans-serif
- 卡片白色背景、圆角 10px、悬停阴影加深
- 类别标签 5 种配色对应 5 个板块
- 封面渐变色按领域区分：
  - AI: `#0f172a → #1e3a5f → #1d4ed8`（深蓝）
  - 游戏: `#0f0f23 → #1a1a3e → #7c3aed`（深紫）
  - 足球: `#0c4a2e → #14532d → #16a34a`（深绿）

#### 每个领域的板块

**AI:**
| category | 板块标题 |
|---|---|
| ai-models | 模型发布/更新 |
| ai-products | 产品发布/更新 |
| industry | 行业动态 |
| paper | 论文研究 |
| tip | 技巧与观点 |

**游戏:**
| 板块 | 涵盖 |
|---|---|
| 新游发布/更新 | 新游戏、DLC、重大更新 |
| 行业动态 | 收购、融资、政策、人事 |
| 电竞资讯 | 赛事结果、战队动态 |
| 硬件/技术 | GPU、主机、引擎、外设 |
| 评测与观点 | 评测、分析、社区热点 |

**足球:**
| 板块 | 涵盖 |
|---|---|
| 赛事战报 | 比分、赛程、杯赛对阵 |
| 转会动态 | 官宣/传闻、租借、续约 |
| 联赛动态 | 积分榜、政策、俱乐部经营 |
| 球员焦点 | 伤病、荣誉、场外新闻 |
| 深度分析 | 战术、评论、数据洞察 |

#### 卡片 HTML 模板
```html
<div class="card open">
<div class="card-top" onclick="this.parentElement.classList.toggle('open')">
<div class="card-num">N</div>
<div class="card-main">
<h3>条目标题</h3>
<div class="card-meta"><span class="src">来源</span><span class="t">北京时间</span><span class="tag tg-xxx">类别</span></div>
<div class="card-brief">一句话摘要</div>
</div>
<div class="card-arrow">▾</div>
</div>
<div class="card-detail">
<h4>深度解读</h4>
<p>300-500 字分析...</p>
<p>来源：<a href="URL" target="_blank">来源名称</a></p>
</div>
</div>
```

## Phase 3: 更新网站

### 3a. 同步仓库
```bash
cd C:/works/gigode.github.io && git pull origin main
```

### 3b. 复制报告到仓库
```bash
DATE=$(date +%Y-%m-%d)
mkdir -p "C:/works/gigode.github.io/reports/$DATE"
cp "C:/works/aihot-report-$DATE.html"   "C:/works/gigode.github.io/reports/$DATE/ai.html"
cp "C:/works/gamehot-report-$DATE.html"  "C:/works/gigode.github.io/reports/$DATE/game.html"
cp "C:/works/footballhot-report-$DATE.html" "C:/works/gigode.github.io/reports/$DATE/football.html"
```

### 3c. 为每份报告添加返回导航

在每个报告的 `<body>` 标签后、`<div class="container">` 之前插入：
```html
<a href="/reports/" style="display:block;padding:12px 20px;color:#6b7280;text-decoration:none;font-size:13px;font-family:PingFang SC,Noto Sans SC,Microsoft YaHei,sans-serif;">← Back to Reports</a>
```

### 3d. 更新 reports/index.html

在 `reports/index.html` 的日期列表中**追加**新日期条目（按日期倒序，最新在前）。

在 `<!-- More dates will be added here in the future -->` 注释**之前**插入新条目块：

```html
    <div class="date-block">
      <div class="date-head">
        <h2>YYYY年M月D日</h2>
        <time>周X</time>
      </div>
      <div class="report-links">
        <a class="report-pill rp-ai"   href="/reports/YYYY-MM-DD/ai.html">🤖 AI 行业报告 · N 条</a>
        <a class="report-pill rp-game" href="/reports/YYYY-MM-DD/game.html">🎮 游戏行业报告 · N 条</a>
        <a class="report-pill rp-foot" href="/reports/YYYY-MM-DD/football.html">⚽ 足球行业报告 · N 条</a>
      </div>
    </div>
```

- 日期用中文格式：`2026年6月5日`
- 周X 用中文：周一/周二/.../周日
- 条数为各报告中实际条目数

**重要：** 不要删除已有的历史日期条目——所有历史报告都应保留。

## Phase 4: 提交与推送

```bash
cd C:/works/gigode.github.io
git add .
git commit -m "Update daily reports for YYYY-MM-DD

- AI report: N items
- Gaming report: N items
- Football report: N items

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
git push origin main
```

## 给用户的输出

完成后给用户一个简洁的总结，格式如下：

```
✅ 日报网站已更新！https://gigode.de/reports/

| 报告 | 条目 | 地址 |
|---|---|---|
| 🤖 AI | N 条 | /reports/YYYY-MM-DD/ai.html |
| 🎮 游戏 | N 条 | /reports/YYYY-MM-DD/game.html |
| ⚽ 足球 | N 条 | /reports/YYYY-MM-DD/football.html |

历史报告已保留，可在 https://gigode.de/reports/ 查看全部。
```

## 边缘情况

- **当天已发布过报告**：跳过当天，提示用户"今天的报告已在 YYYY-MM-DD 发布过"，除非用户明确说"重新生成"
- **API 返回空或搜索无结果**：对应报告标注"今日暂无数据"，仍生成报告但封面注明
- **Git push 冲突**：先 `git pull --rebase` 再 push
- **日期跨越**：如果用户在北京时间 0:00-8:00 之间更新，数据可能偏少（欧洲比赛/美国新闻还没出）。在报告中注明时间窗。

## 不要做

- 不要在用户说"更新"时只更新某一个领域——必须三个领域全覆盖
- 不要删除 `reports/index.html` 中已有的历史日期条目
- 不要修改报告的整体 HTML 结构和 CSS 方案
- 不要在搜索无结果时编造内容
- 不要跳过深度解读——每条必须 300-500 字分析
- 不要遗漏 URL——每条必须有来源链接
