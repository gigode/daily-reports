---
name: daily-reports
description: 更新全套日报网站。当用户说"更新网站"、"更新日报"、"更新报告"、"发布今天的报告"、"刷新日报网站"、"update reports"、"publish reports"时触发。自动拉取 AI/游戏/足球/芯片/股权投资/全球宏观六领域最新资讯，生成 6 份 HTML 报告，更新 GitHub Pages 网站，保留历史报告。
---

# Daily Reports Skill

全自动日报工作流：拉取数据 → 生成 6 份 HTML 报告 → 更新 gigode.github.io → 提交推送。一条命令完成全部操作。

## 工作流总览

```
用户说"更新网站"
    ↓
Phase 1: 并行拉取 6 领域数据
    ↓
Phase 1.5: 数据可靠性分级与源数据清单 ← CRITICAL（2026-07-10 重写）
    ↓
Phase 2: 生成 6 份 HTML 报告 → C:\works\  [源数据锚定 · 3x放大上限 · 足球/游戏摘要模式]
    ↓
Phase 2.5: 写作后自审清单 ← CRITICAL（2026-07-10 新增）— 逐卡5项检查
    ↓
Phase 3: 复制到 gigode.github.io 仓库 + 添加 Back to Reports 导航 + 更新 hub 页面
    ↓
Phase 4: Git commit & push → 线上生效
```

---

## Phase 1: 拉取数据

### Python 环境说明

- **路径:** `C:\Users\zm_ji\scoop\apps\python\current\python.exe` (Python 3.14.5)
- **不要用** `python3` 或 `python`（Windows Store stub，exit 49）
- **在用 Bash 时使用 herdoc 传 Python:**
  ```bash
  /c/Users/zm_ji/scoop/apps/python/current/python.exe << 'PYEOF'
  ...code...
  PYEOF
  ```

### 1a. AI 数据（aihot API）

```bash
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36 aihot-skill/0.2.0"
SINCE=$(date -u -d '36 hours ago' +%Y-%m-%dT%H:%M:%SZ)
curl -sH "User-Agent: $UA" "https://aihot.virxact.com/api/public/items?mode=selected&since=$SINCE&take=100" -o "/c/works/.tmp-aihot-YYYY-MM-DD.json"
```

用 node 解析 JSON（python3 不可用）：
```bash
node -e "
const d = JSON.parse(require('fs').readFileSync('C:/works/.tmp-aihot-2026-06-30.json','utf8'));
const items = d.items || [];
console.log('total:', items.length);
// group by category
"
```

结果通常 20-30 条，5 个 category：`ai-models` / `ai-products` / `industry` / `paper` / `tip`。

### 1b. 游戏数据（WebFetch — 不用 WebSearch）

**⚠️ WebFetch 幻觉高风险领域。** Steam 搜索页面通过 WebFetch 返回的摘要不可靠——会编造不存在的游戏名称、DLC 和发售日期。必须使用编辑内容为主的网站（Kotaku / LoL Esports / PC Gamer）。

**并行 WebFetch 以下页面（必做）：**

| 序号 | 抓取源 | URL | 提取要求 |
|------|--------|-----|---------|
| G1 | Kotaku 首页 | `https://kotaku.com/` | 15+ 条当日游戏新闻头条，标注每条的 URL |
| G2 | LoL Esports | `https://lolesports.com/news` | 8 条电竞新闻（MSI/Worlds/LCS/LCK），含具体赛事和日期 |
| G3 | PC Gamer | `https://www.pcgamer.com/news/` | 8 条 PC 游戏新闻/评测 |

**补充源（按需）：**
- `https://www.gamespot.com/articles/` — 游戏评测
- `https://www.eurogamer.net/archive` — 欧洲游戏新闻

**提取纪律：**
- 每条必须保留原始 URL
- **只提取 WebFetch 返回中明确出现的内容**，不推测、不补充
- 如果 WebFetch 只返回标题没有正文 → 标记为"仅标题，详情待确认"
- Kotaku 通常返回 20-40 条当日文章，优先提取评论数多/置顶的

**目标：** 行业动态 6-8 + 新游评测 5-7 + 电竞 4-5 + 硬件/社区 3-4，最多 20 条（允许更少）。

### 1c. 足球数据（WebFetch）

| 序号 | 抓取源 | URL | 提取要求 |
|------|--------|-----|---------|
| F1 | Sky Sports 足球 | `https://www.skysports.com/football/news` | 8 条比赛+转会+分析头条 |
| F2 | SI 足球转会 | `https://www.si.com/soccer/transfers` | 8 条转会新闻，标注 ✅官宣 / 🔹传闻 |
| F3 | ESPN 足球 | `https://www.espn.com/soccer/` | 8 条联赛+球员新闻 |

约 8 赛事战报 + 8 转会 + 5 联赛 + 5 球员 + 4 分析 ≈ 30 条。

### 1d. 芯片数据（WebFetch）

| 序号 | 抓取源 | URL | 提取要求 |
|------|--------|-----|---------|
| C1 | Investing 半导体 | `https://www.investing.com/news/stock-market-news` + 搜"chip/semiconductor" | 8 条芯片/半导体企业动态 |
| C2 | TechCrunch 硬件 | `https://techcrunch.com/category/startups/` + 搜"chip/Nvidia/semiconductor" | 8 条融资/产品/企业新闻 |

参考此前报道 + WebFetch 补充。约 6 新产品 + 7 企业 + 6 观点 + 6 政策 ≈ 25 条。

### 1e. 股权数据（WebFetch）

抓取源同芯片模式：TechCrunch Startups + Investing Stock Market News + Yahoo Finance 组。

| 序号 | 抓取源 | URL |
|------|--------|-----|
| E1 | TechCrunch Startups | `https://techcrunch.com/category/startups/` |
| E2 | Investing Markets | `https://www.investing.com/news/stock-market-news` |

约 8 投资 + 6 基金 + 7 政策 + 5 观点 ≈ 26 条。

### 1f. 宏观数据（yfinance + FRED + WebFetch 三源）

**步骤 1 — yfinance 拉 22 个市场指标（必做）：**

```bash
/c/Users/zm_ji/scoop/apps/python/current/python.exe << 'PYEOF'
import yfinance as yf, json, os
all_tickers = ['SPY','QQQ','DIA','IWM','^GSPC','^IXIC','^DJI','^FTSE','^N225','^HSI',
               'GC=F','CL=F','SI=F','HG=F','EURUSD=X','USDJPY=X','USDCNY=X','GBPUSD=X',
               'DX-Y.NYB','^TNX','^FVX','^VIX']
data = yf.download(all_tickers, start='2026-06-20', end='2026-06-30', progress=False, auto_adjust=True)
closes = data['Close']
results = {}
for t in all_tickers:
    s = closes[t].dropna()
    latest = float(s.iloc[-1])
    prev = float(s.iloc[-2]) if len(s) >= 2 else latest
    week = float(s.iloc[0]) if len(s) >= 5 else prev
    results[t] = {'latest': latest, 'd_pct': (latest/prev-1)*100, 'w_pct': (latest/week-1)*100}
out = os.path.join(os.environ['TEMP'], 'macro-yf.json')
with open(out,'w') as f: json.dump(results, f, indent=2)
print('yfinance OK:', len(results))
PYEOF
```

**步骤 2 — FRED 拉 14 个经济指标（必做）：**

```bash
/c/Users/zm_ji/scoop/apps/python/current/python.exe << 'PYEOF'
import urllib.request, json, os
FRED_KEY = 'bcfd49c661640e060bdc12f092668068'
codes = {'DFF': 'Fed Funds Rate', 'DGS10': 'US 10Y', 'DGS2': 'US 2Y', 'T10Y2Y': 'Spread',
         'UNRATE': 'Unemployment', 'CPIAUCSL': 'CPI', 'VIXCLS': 'VIX', 'T10YIE': 'Breakeven',
         'M2SL': 'M2', 'BAMLH0A0HYM2': 'HY OAS', 'GDP': 'GDP', 'INDPRO': 'IndProd',
         'RSAFS': 'Retail', 'HOUST': 'Housing'}
results = {}
for code, label in codes.items():
    try:
        url = f'https://api.stlouisfed.org/fred/series/observations?series_id={code}&api_key={FRED_KEY}&file_type=json&sort_order=desc&limit=2'
        req = urllib.request.Request(url, headers={'User-Agent':'curl/8.0'})
        d = json.loads(urllib.request.urlopen(req, timeout=10).read())
        obs = d.get('observations', [])
        if obs:
            latest = obs[0]
            prev = obs[1] if len(obs) > 1 else None
            results[code] = {'value': float(latest['value']), 'date': latest['date'],
               'prev': float(prev['value']) if prev and prev['value']!='.' else None}
    except Exception as e: print(f'{code} err:', e)
out = os.path.join(os.environ['TEMP'], 'macro-fred.json')
with open(out,'w') as f: json.dump(results, f, indent=2)
print(f'FRED OK: {len(results)} series')
PYEOF
```

**步骤 3 — WebFetch 拉 20+ 宏观新闻：**

| 序号 | 抓取源 | URL | 备注 |
|------|--------|-----|------|
| M1 | Investing Market | `https://www.investing.com/news/stock-market-news` | 市场头条 + 宏观经济交叉 |
| M2 | Investing Indicators | `https://www.investing.com/news/economic-indicators` | 经济数据发布 |
| M3 | Yahoo Finance Economy | `https://finance.yahoo.com/topic/economic-news/` | 宏观经济新闻（NBC/Reuters 交叉验证） |
| M4 | Investing Forex | `https://www.investing.com/news/forex-news` | 外汇/商品 |

注意：`investing.com/news/economy-news` 已失效（404），不要使用。
注意：WebFetch 抓取的宏观新闻细节（官员引语、具体数据）必须交叉验证。yfinance/FRED 的结构化数值数据是主要可靠来源，新闻只做背景补充。

约 4 央行 + 4 市场 + 3 经济 + 2 地缘 + 3 商品外汇 ≈ 16 条（允许更少）。

---

## Phase 1.5: 数据可靠性分级与标注准备（CRITICAL — 2026-07-10 重写）

**2026-07-10 审计发现：** Phase 1.5 原来的"交叉验证"方法失败——用 WebFetch 验证 WebFetch 是循环论证。87% 的编造率说明流程需要根本性重建。

### 🔴 核心原则：数据源直接决定标签，不由内容判断

**不要根据"内容看起来可不可信"来判断标签。标签由数据来源的可靠性决定。**

| 数据来源 | 可靠性 | 默认标签 | 规则 |
|---------|--------|---------|------|
| aihot API（AI）| ⭐⭐⭐⭐⭐ | ✅ 结构化源 | 标题/URL/category 直接使用；解读只扩展不编造 |
| yfinance（市场数据）| ⭐⭐⭐⭐⭐ | ✅ 结构化源 | 价格/涨跌幅直接使用；可做趋势分析 |
| FRED（经济数据）| ⭐⭐⭐⭐⭐ | ✅ 结构化源 | 利率/就业/CPI 直接使用；可做经济学解读 |
| **WebFetch 摘要（所有领域）** | ⭐⭐ | 🔹 WebFetch源 | **所有来自 WebFetch 的内容默认不可信** |
| 官方公告/新闻稿 | ⭐⭐⭐⭐ | ✅ 结构化源 | 必须能追溯到具体 URL |

### 🔴 2026-07-10 审计关键发现

**WebFetch → 报告的编造率按领域：**
- 足球：87.5%（5/16 完全编造 + 9/16 大量细节编造）
- 游戏：86.7%（4/15 + 9/15）
- 芯片/股权：~80%（核心标题可信，细节大面积编造）
- 宏观新闻：~67%（标题层可信，解读层编造）
- AI：~12%（aihot API 几乎无编造）

**编造的根本原因不是"不谨慎"，而是 LLM 的统计补全机制：**
当输入 "France 2-0 Morocco QF (Mbappé scores)"（50 bytes），LLM 会自动补全为完整的足球报道（500-2000 bytes）——包括分钟、助攻、控球率、射门数等"看起来合理"但完全不存在的细节。这是 LLM 的本质运作方式，不能靠"小心"解决，必须靠结构性约束。

### 每份报告的源数据清单（Phase 2 写作前必做）

在生成任何报告之前，必须将该领域的源数据整理为纯文本列表（标题+URL，一行一条），作为写作的唯一依据。这个列表就是"数据边界"——卡片内容不得超出此边界。

**示例（足球）：**
```
SOURCE LIST for footballhot-2026-07-10:
- [ESPN] France 2-0 Morocco QF - Mbappé scores (https://www.espn.com/soccer/...)
- [Sky] England vs Norway QF upcoming (https://www.skysports.com/...)
- [SI] Quansah gets 2-match ban after red card (https://www.si.com/...)
- [ESPN] Pulisic microfracture - out for weeks (https://www.espn.com/...)
- [SI] Barcelona bids for Karim Adeyemi (https://www.si.com/...)
- [Sky] Andrey Santos $67M to Manchester United (https://www.skysports.com/...)
- [ESPN] Vinicius $171.5M Saudi offer (https://www.espn.com/...)
- [Sky] Arsenal considering Bruno Guimarães bid (https://www.skysports.com/...)
- [ESPN] Haaland to Real Madrid rumors (https://www.espn.com/...)
- [ESPN] Tchouameni says "happy at Real Madrid" (https://www.espn.com/...)
```

**每张卡片只能基于列表中明确存在的一行或多行。如果某张卡片的核心事实不在列表中 → 整张卡片删除。**

---

## 信息准确性三级体系 v2.0（所有报告共用 — 2026-07-10 重定义）

| Tier | 定义 | 标签 | 适用范围 |
|------|------|------|---------|
| Tier 1 — 结构化源 | 来自 API/数据库的结构化数据，无编造可能 | `<span class="tag-verified">✅ 结构化源</span>` | aihot API / yfinance / FRED / 企业官方公告 |
| Tier 2 — WebFetch源 | 来自 WebFetch AI 摘要，细节可能不准确 | `<span class="tag-unconfirmed">🔹 WebFetch源</span>` | **所有来自 WebFetch 摘要的内容** |
| Tier 3 — 分析推断 | 基于 Tier 1 数据的合理推断（不是事实） | `<span class="tag-analysis">💭 分析</span>` | 趋势分析、市场解读、因果推断 |

**🔴 关键变更：Tier 标签判断标准从"内容可信度"改为"数据来源类型"。**
- 旧规则：看内容是否可信 → 容易误判，因为编造的内容"看起来可信"
- 新规则：看数据来自哪里 → API=✅, WebFetch=🔹, 不管内容"看起来"多可信

**CSS 标签样式（新增 💭 分析）：**
```css
.tag-verified { display:inline-block;padding:2px 8px;border-radius:4px;background:#16a34a33;color:#4ade80;font-size:0.7rem;margin-left:6px;font-weight:700; }
.tag-unconfirmed { display:inline-block;padding:2px 8px;border-radius:4px;background:#d9770633;color:#fbbf24;font-size:0.7rem;margin-left:6px;font-weight:700; }
.tag-analysis { display:inline-block;padding:2px 8px;border-radius:4px;background:#7c3aed33;color:#c4b5fd;font-size:0.7rem;margin-left:6px;font-weight:700; }
```

---

## 报告生成核心规则（Phase 2 前置必读 — 2026-07-10 新增）

### 规则一：源数据锚定（Source Anchoring）

**每张卡片写作前，必须确认源数据中存在对应的标题行。卡片中的每个关键 claim 必须能追溯到源数据列表中的一行。**

实施方式：在写卡片之前，在心里问自己"源数据列表的哪一行支持我写这句话？"如果答案是"没有" → 不写那句话。

### 规则二：三倍放大上限（3x Amplification Cap）

**WebFetch 来源的卡片，解读文字量不超过对应源数据文字量的 3 倍。**

| 源数据量 | 最大解读量 |
|---------|-----------|
| 仅标题（10-30 字） | ~90 字 |
| 标题+一句话摘要（50-80 字） | ~240 字 |
| 完整段落（200+ 字） | ~600 字 |

**例外：** yfinance/FRED/aihot API 数据可视需要充分展开分析（结构化数据可靠）。

### 规则三：足球/游戏降级为"摘要模式"

由于这两个领域 100% 依赖 WebFetch（无结构化 API），编造率 >85%：
- 默认使用简短"新闻摘要"格式（每条 80-200 字）
- **严禁**详细的比赛战报（分钟、助攻、控球率、xG）、数据分析和战术解读
- 标签统一使用 🔹 WebFetch源（除非有官方公告确认）
- Footer 区域增加提示：`⚠️ 本报告基于 AI 网页摘要，赛事细节未经独立验证，仅供话题参考`

### 规则四：具体数字必须有源可查

以下类型的数字**必须**在源数据中找到对应，否则删除：
- 比分、进球时间
- 股价涨跌幅（yfinance 数据除外）
- 融资金额、估值
- 裁员人数、百分比
- 收视率、用户数
- 分析师目标价、评级

### 规则五：不编造源数据中不存在的实体和事件

- 不编造"分析师说/专家认为/官员表示"类型的引语和观点
- 不编造"历史对比"（如"这是自 1998 年以来首次..."）
- 不编造"比赛战报"级别的赛事细节
- 不编造"公司财报细节"（如"上赛季 42 场首发，传球成功率 92%"）

---

## Phase 2: 生成报告

生成 6 份独立的 HTML 报告，保存到 `C:\works\`：

| 领域 | 文件名 | 条数（上限，允许更少） |
|------|--------|------|
| AI | `aihot-report-YYYY-MM-DD.html` | ≤ 26（aihot API 去重后） |
| 游戏 | `gamehot-report-YYYY-MM-DD.html` | ≤ 20（WebFetch 高风险，宁少勿假） |
| 足球 | `footballhot-report-YYYY-MM-DD.html` | ≤ 20（WebFetch 高风险，宁少勿假） |
| 芯片 | `chiphot-report-YYYY-MM-DD.html` | ≤ 20（WebFetch 中高风险） |
| 股权投资 | `equityhot-report-YYYY-MM-DD.html` | ≤ 20（WebFetch 中高风险） |
| 全球宏观 | `macro-report-YYYY-MM-DD.html` | ≤ 15 条 + 34 指标（yfinance/FRED 可靠） |

日期取当前北京时间（`date +%Y-%m-%d`）。

---

### 2a. 通用 HTML 结构（所有 6 份报告共用）

#### CSS 变量 & 暗色主题风格

所有报告使用暗色背景 + 半透明玻璃卡片风格（非白色亮色主题）：

```css
body {
  background: linear-gradient(145deg, ...); /* 各领域不同 */
  color: #e2e8f0;
  min-height: 100vh; padding: 0 16px 60px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC", "Microsoft YaHei", sans-serif;
}
.container { max-width: 900px; margin: 0 auto; }
```

#### 封面

```html
<div class="cover">
<h1>领域标题</h1>
<div class="sub">领域副标题</div>
<div class="badge">YYYY-MM-DD · 共 N 条</div>
</div>
```

#### 目录导航（toc）

```html
<nav class="toc">
<div class="toc-title">目录</div>
<ul class="toc-list">
<li><a href="#s-section1">📦 板块名 · N</a></li>
...
</ul>
</nav>
```

#### 板块节

```html
<div class="section" id="s-section1" data-category="xxx">
<div class="section-header" style="background:#xxxx2222;border-left:4px solid #xxxxxx;">
<span class="icon">📦</span>
<h2 style="color:#xxxxxx;">板块标题</h2>
<span class="count">N 条</span>
</div>
<!-- 卡片列表 -->
</div>
```

#### 🔴 卡片 HTML 模板（CRITICAL）

每张卡片必须严格按照以下结构，**不容变体**：

```html
<div class="card active">          <!-- 首条 card active，后续 card -->
<div class="card-top">             <!-- 点击区 -->
<span class="card-number">1</span>        <!-- 全局编号 -->
<div class="card-content">               <!-- 标题+元数据 -->
<h3>条目标题文案<span class="tag-verified">✅ 已验证</span></h3>  <!-- Tier 1/2 必须带标签 -->
<div class="card-meta">来源名 · 6月29日 21:22</div>
</div>
<div class="card-arrow">▼</div>          <!-- ▸ 展开箭头，必须存在 -->
</div>                                    <!-- .card-top 结束 -->
<div class="card-detail">                <!-- 展开区，direct child of .card -->
<p>深度解读：Tier 1(300-500字) / Tier 2(100-200字) / 仅标题(80-150字)</p>
<a class="source-link" href="原URL" target="_blank">🔗 查看原文</a>
</div>
</div>
```

**四条铁律（卡片结构）：**
1. `<div class="card-arrow">▼</div>` 必须在 `.card-top` 内的末尾（`.card-content` 之后），**不能省略**
2. `</div>` 关闭 `.card-top` 后才打开 `.card-detail` — card-detail **不能嵌套在 card-top 内**
3. 首条卡片用 `class="card active"`，后续用 `class="card"`
4. **每条标题必须含 Tier 标签：** `<span class="tag-verified">✅ 已验证</span>` 或 `<span class="tag-unconfirmed">🔹 传闻</span>`（Tier 3 纯分析除外）

#### CSS 关键规则

```css
.card {
  background: #1e293bcc; border: 1px solid #334155;
  border-radius: 10px; margin-bottom: 10px; overflow: hidden;
  backdrop-filter: blur(4px); transition: border-color 0.25s;
}
.card:hover { border-color: #475569; }
.card.active { border-color: #XXXXXX66; }  /* 领域 accent */
.card-top {
  display: flex; align-items: flex-start; gap: 12px;
  padding: 14px 16px; cursor: pointer; position: relative;
}
.card-number {
  flex-shrink: 0; width: 28px; height: 28px;
  border-radius: 50%; display: flex; align-items: center; justify-content: center;
  font-size: 0.75rem; font-weight: 700;
  background: #334155; color: #94a3b8; margin-top: 2px;
}
.card-content { flex: 1; min-width: 0; }
.card-content h3 {
  font-size: 0.95rem; font-weight: 600; line-height: 1.4; color: #f1f5f9;
  display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden;
}
.card-meta { font-size: 0.75rem; color: #64748b; margin-top: 4px; }
.card-arrow {
  flex-shrink: 0; width: 24px; height: 24px;
  display: flex; align-items: center; justify-content: center;
  font-size: 0.7rem; color: #475569;
  transition: transform 0.3s ease; margin-top: 4px;
}
.card.active .card-arrow { transform: rotate(180deg); color: #XX; }
.card-detail { display: none; border-top: 1px solid #33415555; margin: 0 16px; padding-top: 12px; }
.card.active .card-detail { display: block; }
.card-detail p { font-size: 0.88rem; line-height: 1.7; color: #cbd5e1; margin-bottom: 10px; }
.source-link {
  display: inline-block; font-size: 0.8rem; color: #XX; text-decoration: none;
  padding: 4px 12px; border-radius: 6px;
  background: #XXXXXX22; border: 1px solid #XXXXXX44;
}
.footer {
  text-align: center; padding: 32px 0 16px;
  border-top: 1px solid #334155; margin-top: 40px;
  font-size: 0.8rem; color: #475569;
}

/* 响应式 */
@media (max-width: 640px) {
  .cover h1 { font-size: 1.8rem; }
  .card-top { padding: 12px; gap: 10px; }
  .card-content h3 { font-size: 0.88rem; }
}
```

#### JS 展开/折叠 + 目录跳转

```html
<script>
document.addEventListener('DOMContentLoaded', function() {
  var cards = document.querySelectorAll('.card');
  cards.forEach(function(card) {
    var top = card.querySelector('.card-top');
    if (top) {
      top.addEventListener('click', function() {
        card.classList.toggle('active');
      });
    }
  });
  var tocLinks = document.querySelectorAll('.toc-list a');
  tocLinks.forEach(function(link) {
    link.addEventListener('click', function(e) {
      var href = this.getAttribute('href');
      if (href && href.startsWith('#')) {
        e.preventDefault();
        var target = document.querySelector(href);
        if (target) target.scrollIntoView({ behavior: 'smooth', block: 'start' });
      }
    });
  });
});
</script>
```

---

### 2b. 各领域专属配置

#### 🤖 AI 行业报告

- **封面渐变:** `linear-gradient(145deg, #0f172a 0%, #1e3a5f 50%, #1d4ed8 100%)`
- **Accent color:** `#2563eb`
- **5 板块:**

| Section ID | 标题 | 图标 | 边框色 | 对应 category |
|---|---|---|---|---|
| s-products | 产品发布/更新 | 📦 | #2563eb | ai-products |
| s-models | 模型发布/更新 | 🧠 | #7c3aed | ai-models |
| s-industry | 行业动态 | 🌐 | #059669 | industry |
| s-paper | 论文研究 | 📖 | #d97706 | paper |
| s-tip | 技巧与观点 | 💡 | #dc2626 | tip |

#### 🎮 游戏行业报告

- **封面渐变:** `linear-gradient(145deg, #0f0f23 0%, #1a1a3e 50%, #7c3aed 100%)`
- **Accent color:** `#7c3aed`
- **5 板块:**

| Section ID | 标题 | 图标 | 边框色 |
|---|---|---|---|
| s-releases | 新游发布/更新 | 📦 | #2563eb |
| s-industry | 行业动态 | 🌐 | #059669 |
| s-esports | 电竞资讯 | ⚽ | #d97706 |
| s-hardware | 硬件/技术 | 🖥️ | #7c3aed |
| s-review | 评测与观点 | ⭐ | #dc2626 |

#### ⚽ 足球行业报告

- **封面渐变:** `linear-gradient(145deg, #0c4a2e 0%, #14532d 50%, #16a34a 100%)`
- **Accent color:** `#16a34a`
- **5 板块:**

| Section ID | 标题 | 图标 | 边框色 |
|---|---|---|---|
| s-matches | 赛事战报 | ⚽ | #16a34a |
| s-transfers | 转会动态 | 💼 | #2563eb |
| s-leagues | 联赛动态 | 🏆 | #d97706 |
| s-players | 球员焦点 | 👤 | #dc2626 |
| s-analysis | 深度分析 | 📊 | #7c3aed |

**转会必须标注:** ✅ 官宣 / 🔹 传闻

#### 🔬 芯片行业报告

- **封面渐变:** `linear-gradient(145deg, #064e3b 0%, #065f46 50%, #10b981 100%)`
- **Accent color:** `#10b981`
- **4 板块:**

| Section ID | 标题 | 图标 | 边框色 |
|---|---|---|---|
| s-products | 新产品/技术 | 🔬 | #10b981 |
| s-companies | 企业动态 | 🏢 | #2563eb |
| s-experts | 大咖观点 | 🎤 | #d97706 |
| s-policy | 政策资本 | 📋 | #dc2626 |

#### 💰 股权投资报告

- **封面渐变:** `linear-gradient(145deg, #78350f 0%, #92400e 50%, #f59e0b 100%)`
- **Accent color:** `#f59e0b`
- **4 板块:**

| Section ID | 标题 | 图标 | 边框色 |
|---|---|---|---|
| s-deals | 投资案例 | 💼 | #f59e0b |
| s-funds | 基金设立 | 🏦 | #2563eb |
| s-policy | 政策行情 | 📊 | #059669 |
| s-experts | 大咖观点 | 🎤 | #7c3aed |

#### 🌍 全球宏观报告（独立暗色风格）

- **封面渐变:** `linear-gradient(135deg, #1a0f0a, #3d2517, #b45309)`
- **Special CSS:** 暗黑风格 `background:#0a0a0a`，metric-grid 布局，`.card.active` 切换（不 toggle，切换时关闭其他卡片）
- **5 板块 + Market Data 区：**

| Section ID | 标题 | 边框色 |
|---|---|---|
| s-cenbank | Central Bank 央行政策 | #dc2626 |
| s-market | Market Moves 市场动态 | #2563eb |
| s-global | Global Economy 全球经济 | #059669 |
| s-geo | Geopolitics & Trade 地缘贸易 | #d97706 |
| s-commodity | Commodities & FX 商品外汇 | #7c3aed |

**Market Data metric-grid 必含 34 个指标，7 组，每指标必须有 metric-explain 中文解释：**

```html
<div class="metric-grid">
<div class="metric-group-label">美股ETF · yfinance 实时 <small>标普/纳斯达克/道琼斯/罗素</small></div>
<div class="metric-card"><div class="metric-label">SPY</div><div class="metric-value">$741.00</div><div class="metric-change pos">+1.65% d</div><div class="metric-explain">标普500 ETF，美股大盘风向标</div></div>
<!-- ... 34 cards total ... -->
</div>
```

**指标分组:**
1. 美股ETF (yfinance × 4): SPY/QQQ/DIA/IWM — 含 d% 涨跌
2. 美联储利率 (FRED × 6): DFF/DGS10/DGS2/T10Y2Y/T10YIE/BAMLH0A0HYM2
3. 美国实体经济 (FRED × 7): UNRATE/CPIAUCSL/M2SL/GDP/INDPRO/RSAFS/HOUST
4. VIX (yfinance × 1)
5. 全球指数 (yfinance × 6): ^GSPC/^IXIC/^DJI/^FTSE/^N225/^HSI — 含 d% 涨跌
6. 大宗商品 (yfinance × 4): GC=F/CL=F/SI=F/HG=F — 含 d% 涨跌
7. 主要货币 (yfinance × 5): DX-Y.NYB/EURUSD=X/USDJPY=X/USDCNY=X/GBPUSD=X — 含 d% 涨跌

**宏观卡片 JS 切换逻辑（独立于其他 5 份报告，同时只展开一张卡片）：**

```javascript
top.addEventListener('click', function(e) {
  e.stopPropagation();
  var isActive = card.classList.contains('active');
  cards.forEach(function(c) { c.classList.remove('active') });
  if (!isActive) { card.classList.add('active') }
});
```

---

## Phase 2.5: 写作后自审清单（Post-Write Self-Audit — 2026-07-10 新增）

**这是 2026-07-10 审计后新增的强制步骤。每份报告写完后、复制到仓库前，必须逐卡执行以下 5 项检查。**

### 逐卡自审清单

对每张卡片问以下 5 个问题（必须全部通过）：

1. ☐ **核心事实存在？** 这条新闻的核心事实在源数据列表中存在吗？
   - ❌ 不存在 → **整张卡片删除**（不是修改，是删除）
   - ✅ 存在 → 继续

2. ☐ **具体数字有据？** 卡片中的具体数字（金额、百分比、比分、人数）在源数据中存在吗？
   - ❌ 存在编造的数字 → **删除该数字**或标记 `🔹`
   - ✅ 全部有据或已标记 → 继续

3. ☐ **引语真实？** 卡片中引用的"专家说/分析师认为/官员表示"在源数据中存在吗？
   - ❌ 不存在 → **删除引语**（不能改为"据分析"保留）
   - ✅ 存在或没有引语 → 继续

4. ☐ **历史对比基于数据？** 卡片中的历史对比基于 Tier 1 数据吗？
   - ❌ 基于"常识"或"记忆"的历史断言 → **删除该对比**
   - ✅ 基于 yfinance/FRED 数据或标注了 `💭 分析` → 继续

5. ☐ **Tier 标签正确？** 该卡的标签匹配数据来源吗？
   - WebFetch 源但标了 ✅ → **改为 🔹 WebFetch源**
   - 标签正确 → ✅ 通过

### 领域专项检查

**足球/游戏报告（WebFetch 100%）：额外检查：**
- ☐ 没有任何"分钟+进球者+助攻者"级别的赛事描述
- ☐ 没有任何 xG、控球率、射门数等高级统计
- ☐ 没有任何"这是自 XXXX 年以来首次..."的历史断言
- ☐ Footer 包含 WebFetch 警告提示

**芯片/股权报告（WebFetch 主导，yfinance 辅助）：额外检查：**
- ☐ 股票涨跌幅来自 yfinance（不来自 WebFetch 摘要中的数字）
- ☐ 融资轮的金额/估值/领投方在源数据中存在
- ☐ 没有编造"Morgan Stanley/Citi/高盛 上调目标价"类分析师报告

**宏观报告（yfinance/FRED + WebFetch 混合）：额外检查：**
- ☐ 所有 34 个市场指标数值来自 yfinance/FRED JSON 文件
- ☐ 新闻解读中的官员引语在 WebFetch 源数据中存在
- ☐ 没有编造的经济预测数字

### 未通过的处理

- **任何卡片未通过检查 →** 修正后再进入 Phase 3，**不允许带着已知问题部署**
- **如果某领域超过 50% 卡片未通过 →** 该领域整份报告重写
- **如果无法修正（源数据不足）→** 缩短为简短摘要，标注 🔹 WebFetch源

---

## Phase 3: 更新网站

### 3a. Git 操作全部用 `git -C /c/works/gigode.github.io`

因为工作目录可能不在 repo 中，必须使用 `-C` 参数。

```bash
git -C /c/works/gigode.github.io pull origin main
```

### 3b. 复制 6 份报告到仓库

```bash
DATE=2026-06-30
mkdir -p "/c/works/gigode.github.io/reports/$DATE"
cp "/c/works/aihot-report-$DATE.html"       "/c/works/gigode.github.io/reports/$DATE/ai.html"
cp "/c/works/gamehot-report-$DATE.html"      "/c/works/gigode.github.io/reports/$DATE/game.html"
cp "/c/works/footballhot-report-$DATE.html"  "/c/works/gigode.github.io/reports/$DATE/football.html"
cp "/c/works/chiphot-report-$DATE.html"      "/c/works/gigode.github.io/reports/$DATE/chip.html"
cp "/c/works/equityhot-report-$DATE.html"    "/c/works/gigode.github.io/reports/$DATE/equity.html"
cp "/c/works/macro-report-$DATE.html"        "/c/works/gigode.github.io/reports/$DATE/macro.html"
```

### 3c. 为每份报告添加 Back to Reports 导航

用 node（python3 不可用）：

```bash
node -e "
const fs = require('fs');
const dir = 'C:/works/gigode.github.io/reports/2026-06-30';
const files = ['ai.html','game.html','football.html','chip.html','equity.html','macro.html'];
const back = '<a href=\"/reports/\" style=\"display:block;padding:12px 20px;color:#6b7280;text-decoration:none;font-size:13px;font-family:PingFang SC,Noto Sans SC,Microsoft YaHei,sans-serif;\">← Back to Reports</a>\n';
files.forEach(f => {
  let s = fs.readFileSync(dir + '/' + f, 'utf8');
  if (s.includes('Back to Reports')) return;
  s = s.replace('<body>', '<body>\n' + back);
  fs.writeFileSync(dir + '/' + f, s, 'utf8');
  console.log(f + ' OK');
});
"
```

### 3d. 更新 hub 页面 reports/index.html

**🔴 CRITICAL: 新日期必须插入到页面最顶部（倒序排列），不是底部！**

在 `</section>`（hero 区块）关闭标签**之后**、第一个 `<div class="date-block">` **之前**插入新日期块。

**具体方法：** 找到 hero 区块结束和第一个已有日期之间的位置（即 `</section>` 和 `<div class="date-block">` 之间），在那里插入新的日期块。这样最新的日期始终在最上面。

**不要**把新日期插入到页面底部的 `<!-- More dates will be added here... -->` 注释前面——那样会导致新日期出现在列表最低端，页面上看不到。

```html
    <div class="date-block">
      <div class="date-head">
        <h2>2026年6月30日</h2>
        <time>周二</time>
      </div>
      <div class="report-links">
        <a class="report-pill rp-ai"    href="/reports/2026-06-30/ai.html">🤖 AI 行业报告 · 22 条</a>
        <a class="report-pill rp-game"  href="/reports/2026-06-30/game.html">🎮 游戏行业报告 · 30 条</a>
        <a class="report-pill rp-foot"  href="/reports/2026-06-30/football.html">⚽ 足球行业报告 · 29 条</a>
        <a class="report-pill rp-chip"  href="/reports/2026-06-30/chip.html">🔬 芯片行业报告 · 26 条</a>
        <a class="report-pill rp-equity" href="/reports/2026-06-30/equity.html">💰 股权投资报告 · 26 条</a>
        <a class="report-pill rp-macro" href="/reports/2026-06-30/macro.html">🌍 全球宏观日报 · 22 条</a>
      </div>
    </div>
```

**Pill 样式（index.html 中已存在）：**
```css
.rp-ai   { background:#dbeafe; color:#1e40af; }
.rp-game { background:#ede9fe; color:#5b21b6; }
.rp-foot { background:#d1fae5; color:#065f46; }
.rp-chip { background:#d4f5e9; color:#0d6b3d; }
.rp-equity { background:#fef9e7; color:#8b6914; }
.rp-macro { background:#fef3c7; color:#92400e; }
```

**6 个 pill 全部列出**，不能遗漏任何一个领域。日期用中文格式（2026年6月30日），周几用中文（周一...周日）。

---

### 3e. （已移至 Phase 1.5 — 数据验证在报告生成之前执行，而非之后）

---

## Phase 4: 提交与推送

```bash
git -C /c/works/gigode.github.io add .
git -C /c/works/gigode.github.io commit -m "Update daily reports for YYYY-MM-DD

- AI report: N items
- Gaming report: N items
- Football report: N items
- Chip report: N items
- Equity report: N items
- Macro report: N items

Co-Authored-By: Claude <noreply@anthropic.com>"
git -C /c/works/gigode.github.io push origin main
```

---

## 输出格式

```
✅ 日报网站已更新！https://gigode.de/reports/YYYY-MM-DD/

| 报告 | 条目 | 地址 |
|---|---|---|
| 🤖 AI | N 条 | /reports/YYYY-MM-DD/ai.html |
| 🎮 游戏 | N 条 | /reports/YYYY-MM-DD/game.html |
| ⚽ 足球 | N 条 | /reports/YYYY-MM-DD/football.html |
| 🔬 芯片 | N 条 | /reports/YYYY-MM-DD/chip.html |
| 💰 股权投资 | N 条 | /reports/YYYY-MM-DD/equity.html |
| 🌍 全球宏观 | N 条 | /reports/YYYY-MM-DD/macro.html |

历史报告已保留，可在 https://gigode.de/reports/ 查看全部。
```

**今日亮点** 部分列出每个领域最值得关注的 1-2 条头条新闻。

---

## 边缘情况

- **当天已发布过报告：** 跳过，提示用户；除非用户明确说"重新生成"
- **API 返回空 / WebFetch 无结果：** 对应报告封面标注"今日暂无数据"，仍生成占位报告
- **Git push 冲突：** `git -C /c/works/gigode.github.io pull --rebase` 再 push
- **日期跨越（0:00-8:00）：** 数据偏少，报告注明时间窗
- **yfinance 下载失败：** 回退到 WebFetch 抓 investing.com indices 页面
- **FRED API 返回空：** 单条卡片标注"数据暂缺"，不阻塞整体生成

## 铁律（不可违反）

### 环境铁律
1. **全文 0 个 python3 命令** — 用 `/c/Users/zm_ji/scoop/apps/python/current/python.exe`
2. **全文 0 个 WebSearch** — 全部用 WebFetch
3. **全文 0 个 python** — 只能用 scoop Python 完整路径
4. **git 命令全部用 `-C`** — `git -C /c/works/gigode.github.io`
5. **node 解析 JSON** — python3 不可用时用 node

### 内容铁律
6. **6 领域全覆盖** — 不能只更新部分
7. **字数弹性化，宁短勿假** — WebFetch 源默认简短（80-200 字），绝不编造细节凑字
8. **每条标题必须含数据源标签** — `✅ 结构化源`（API）或 `🔹 WebFetch源`（WebFetch）或 `💭 分析`（Tier 3）。
   - **标签由数据来源决定，不由内容"看起来可不可信"决定。**
   - **所有 WebFetch 来源的内容默认使用 🔹 标签，无论内容看起来多可信。**
9. **每条必须有来源 URL** — 不遗漏。URL 必须指向包含该信息的实际页面。
10. **源数据锚定规则（2026-07-10 新增）** — 每张卡片的核心事实必须在源数据列表中存在对应行。找不到 → 整卡删除。
11. **三倍放大上限（2026-07-10 新增）** — WebFetch 来源卡片，解读文字 ≤ 源数据的 3 倍。
12. **具体数字必须有源可查（2026-07-10 新增）** — 比分、金额、百分比、人数等具体数字必须在源数据或 yfinance/FRED JSON 中能找到。找不到 → 删除该数字。
13. **不编造引语、历史断言、赛事细节（2026-07-10 新增）** — 禁止编造"专家说/分析师认为"型引语、"自 XXXX 年以来首次"型历史断言、和比赛战报级别细节。

### Phase 2.5 自审铁律（2026-07-10 新增）
14. **Phase 2.5 不可跳过** — 每份报告写完后必须逐卡执行 5 项自审清单
15. **未通过自审 → 修正后再部署** — 不允许带着已知问题 push
16. **足球/游戏报告 Footer 必须含 WebFetch 警告** — `⚠️ 本报告基于 AI 网页摘要，赛事细节未经独立验证，仅供话题参考`

### 结构铁律
17. **card-arrow 必须存在** — 每张卡片缺一不可
18. **card-detail 必须是 card 直接子元素** — 不嵌套在 card-top 内
19. **不删除历史条目** — index.html 只追加不删除
20. **新日期插入 TOP** — index.html 新日期块必须插入最顶部

### 去重铁律
21. **去重前两日报道** — 生成报告前，必须对比前两日报告，剔除标题/事件已出现在前两日中的条目，只保留新资讯。

## 相关资源

- 目标网站: https://gigode.de/reports/
- 仓库: git@github.com:gigode/gigode.github.io.git (本地: C:/works/gigode.github.io)
- 本 Skill 仓库: https://github.com/gigode/daily-reports
- FRED API Key: 见 memory `fred-api-key`
- 卡片结构: 见 memory `daily-reports-card-structure-fix`
