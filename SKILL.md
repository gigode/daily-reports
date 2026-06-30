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
Phase 2: 生成 6 份 HTML 报告 → C:\works\
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

### 1b. 游戏数据（WebFetch — 不用 WebSearch，WebSearch 不可用）

**并行 WebFetch 以下页面，每行一个 WebFetch 调用：**

| 序号 | 抓取源 | URL | 提取要求 |
|------|--------|-----|---------|
| G1 | Steam 新游 | `https://store.steampowered.com/search/?sort_by=Released_DESC&os=win` | 8 款 6 月新游，标题+价格+URL |
| G2 | Steam 精选 | `https://store.steampowered.com/explore/new/` | 8 款热门新游，标题+价格+URL |
| G3 | LoLPedia 电竞 | `https://lolesports.com/news` | 8 条电竞头条，MSI/Worlds/LCS/LPL/LCK |
| G4 | Sky Sports 转会 | `https://www.skysports.com/transfer/news` | 8 条转会/签约新闻 |

其他游戏数据源（按需抓取）：
- `https://www.pcgamer.com/news/` — 游戏行业新闻
- `https://www.gamespot.com/articles/?type=review` — 游戏评测

每条必须保留原始 URL。10 条新游 + 5 行业 + 6 电竞 + 5 硬件 + 4 评测 ≈ 30 条。

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
FRED_KEY = '$FRED_API_KEY'
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

| 序号 | 抓取源 | URL |
|------|--------|-----|
| M1 | Investing Economics | `https://www.investing.com/news/economy-news` |
| M2 | Investing Indicators | `https://www.investing.com/news/economic-indicators` |
| M3 | Investing Market | `https://www.investing.com/news/stock-market-news` |
| M4 | Yahoo Finance | `https://finance.yahoo.com/topic/economy/` |
| M5 | Investing Forex | `https://www.investing.com/news/forex-news` |

约 4 央行 + 5 市场 + 4 经济 + 4 地缘 + 5 商品外汇 ≈ 22 条。

---

## 报告生成策略

**每份报告由 Claude 直接 Write 生成，不要用 Agent。**
Agent 生成的质量参差不齐，且经常出现卡片结构错误。直接写出完整 HTML 是最可靠的方式。

**card-arrow 必须存在于每张卡片中**，card-detail 必须是 card 的直接子元素（不嵌套在 card-top 内）。

---

## Phase 2: 生成报告

生成 6 份独立的 HTML 报告，保存到 `C:\works\`：

| 领域 | 文件名 | 条数 |
|------|--------|------|
| AI | `aihot-report-YYYY-MM-DD.html` | ~22 |
| 游戏 | `gamehot-report-YYYY-MM-DD.html` | ~30 |
| 足球 | `footballhot-report-YYYY-MM-DD.html` | ~29 |
| 芯片 | `chiphot-report-YYYY-MM-DD.html` | ~26 |
| 股权投资 | `equityhot-report-YYYY-MM-DD.html` | ~26 |
| 全球宏观 | `macro-report-YYYY-MM-DD.html` | ~22 |

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
<h3>条目标题文案</h3>
<div class="card-meta">来源名 · 6月29日 21:22</div>
</div>
<div class="card-arrow">▼</div>          <!-- ▸ 展开箭头，必须存在 -->
</div>                                    <!-- .card-top 结束 -->
<div class="card-detail">                <!-- 展开区，direct child of .card -->
<p>300-500 字中文深度解读...</p>
<a class="source-link" href="原URL" target="_blank">🔗 查看原文</a>
</div>
</div>
```

**三条铁律：**
1. `<div class="card-arrow">▼</div>` 必须在 `.card-top` 内的末尾（`.card-content` 之后），**不能省略**
2. `</div>` 关闭 `.card-top` 后才打开 `.card-detail` — card-detail **不能嵌套在 card-top 内**
3. 首条卡片用 `class="card active"`，后续用 `class="card"`

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

在 `<!-- More dates will be added here in the future -->` 注释**之前**插入新日期块：

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

1. **全文 0 个 python3 命令** — 用 `/c/Users/zm_ji/scoop/apps/python/current/python.exe`
2. **全文 0 个 WebSearch** — 全部用 WebFetch
3. **全文 0 个 python** — 只能用 scoop Python 完整路径
4. **git 命令全部用 `-C`** — `git -C /c/works/gigode.github.io`
5. **6 领域全覆盖** — 不能只更新部分
6. **card-arrow 必须存在** — 每张卡片缺一不可
7. **每条深度解读 300-500 字** — 不缩水
8. **每条必须有来源 URL** — 不遗漏
9. **不删除历史条目** — index.html 只追加不删除
10. **node 解析 JSON** — python3 不可用时用 node

## 相关资源

- 目标网站: https://gigode.de/reports/
- 仓库: git@github.com:gigode/gigode.github.io.git (本地: C:/works/gigode.github.io)
- 本 Skill 仓库: https://github.com/gigode/daily-reports
- FRED API Key: 见 memory `fred-api-key`
- 卡片结构: 见 memory `daily-reports-card-structure-fix`
