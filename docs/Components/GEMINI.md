# InvestmentCareer 深度财报研究库 — Gemini CLI 自动化工作流规范

本配置文件专用于 [`antuotuo2000/investmentcareer`](https://github.com/antuotuo2000/investmentcareer) 代码仓库根目录。当在本地仓库目录启动 [Gemini CLI](https://codelabs.developers.google.com/gemini-cli-hands-on)（参见 [`run-gemini-cli` 配置说明](https://github.com/google-github-actions/run-gemini-cli)）时，助手会自动读取本文件并按以下标准流水线执行财报分析、网页生成、索引更新与 Git 自动推送。

## 1. 极简指令触发规则

当用户输入**单个股票代码**（如 `AMD`、`ASML`、`COST`、`00700.HK`）、**公司中英文名称**（如 `阿斯麦`、`Costco`）或简短指令（如 `分析 PLTR`）时，无需反复追问格式，默认直接执行以下端到端四步流程：

1. **联网核实最新财报与估值数据**
2. **按仓库统一机构研报模板生成纯净版 `<TICKER>.html`**
3. **运行本地增量扫描脚本更新 `index.html` 离线索引（如存在）**
4. **自动执行 `git add`、`git commit` 与 `git push` 推送至 `main` 分支**

## 2. 财报数据核实与真实性红线

所有写入 `<TICKER>.html` 的财务指标、业务增速、管理层指引与估值倍数必须来自可验证的公开披露源，严禁凭记忆编造或估算未披露数值：

| 数据维度 | 优先权威核实渠道 | 必填核心指标要求 |
| :--- | :--- | :--- |
| 最新季度与年度财报 | [SEC EDGAR 官方文件库](https://www.sec.gov/edgar/searchedgar/companysearch)（10-K / 10-Q / 8-K）及公司 IR 新闻稿 | 最新单季与全年营收、同比增速（YoY）、GAAP 与 Non-GAAP 毛利率、营业利润率、稀释 EPS |
| 现金流与资产负债表 | 公司现金流量表与资产负债表附注 | 经营活动现金流（OCF）、资本开支（CapEx）、自由现金流（FCF）、账面现金、总债务与回购分红额 |
| 业绩指引与一致预期 | 最新季报 Business Outlook 章节及主流市场统计 | 下季度营收与 Non-GAAP EPS 指引区间、远期市盈率（Forward P/E）、华尔街共识目标价 |
| 行业份额与技术壁垒 | 官方技术白皮书与第三方行业追踪报告 | 核心产品市占率、代际技术路线图、核心竞争对手横向对比参数 |

正文中引用的关键外部财务数据与行业结论须附带可点击的 `<a href="..." target="_blank" rel="noopener noreferrer">` 原文链接。

## 3. `<TICKER>.html` 页面结构与门户自动解析契约

仓库首页 `index.html` 通过 [GitHub REST Contents API](https://docs.github.com/en/rest/repos/contents) 自动扫描根目录下的 `.html` 文件，并利用内置 DOM 解析器自动提取每篇新财报的元数据与核心财务卡片。因此，新生成的 `<TICKER>.html` 必须严格遵守以下结构契约（可参考同目录下的 `NVDA.html` 或 `STX.html`）：

### 3.1 文件纯净度与前端依赖规范

- **输出文件名**：保存在仓库根目录，统一命名为大写股票代码 `<TICKER>.html`（如 `AMD.html`、`ASML.html`）。
- **严禁脚本包裹字符**：文件必须以 `<!DOCTYPE html>` 开头、以 `</html>` 结尾；严禁在首尾残留 `html_content = """` 等 Python 赋值代码，严禁残留 `[cite: ...]` 文本标记。
- **固定版本前端库**：统一通过 `<script>` 引入 [Tailwind CSS 3.4.17](https://cdn.tailwindcss.com/3.4.17) 与 [Chart.js 4.4.1](https://cdn.jsdelivr.net/npm/chart.js@4.4.1)，内置深浅色主题切换（`dark` class）与打印导出样式（`@media print`）。

### 3.2 门户首页自动抓取的三大必填 DOM 节点

| 门户自动提取字段 | `<TICKER>.html` 内对应的必填 HTML 结构 | 示例格式 |
| :--- | :--- | :--- |
| 股票代码、交易所与公司名 | `<title>` 标签须包含 `中文名 英文名 (交易所: 代码) 深度研究报告：副标题` | `<title>超威半导体 AMD (NASDAQ: AMD) 深度研究报告：MI350 算力放量与数据中心份额跃迁</title>` |
| 卡片三宫格财务快照 (`finMetrics`) | `<aside id="sidebar">` 内的快照区包含 3 组 `<div><span>指标名</span><span>数值</span></div>`（每组正好 2 个直接子 `<span>`） | `<div><span>FQ1 最新营收</span><span>$4.23B (+61% YoY)</span></div>` |
| 四大核心 KPI 卡片 (`kpis`) | 主内容区首屏包含 4 个带 `.report-card` 类名的指标卡片，内含指标标题、核心数值、同比/利润率徽章与一句话解读 | `$4.23B`、`+61.0% YoY`、`Non-GAAP 毛利率 57.8%` |

### 3.3 标准七大深度研究章节与图表配置

每份公司财报分析网页应完整覆盖以下 7 个章节，并配备 3 至 4 张置于固定高度容器（如 `class="relative h-72 w-full"` 且 Chart.js 配置 `maintainAspectRatio: false`）内的交互式图表：

1. **00. 执行摘要与核心指标（`id="sec-exec"`）**：投资评级徽章、核心投资逻辑（Core Thesis）横幅、三栏核心结论与 4 张 `.report-card` KPI 卡片。
2. **01. 业务与收入架构（`id="sec-biz"`）**：支持双视图切换的季度/年度分部营收与毛利率演进柱线复合图，配各业务分部拆解卡片。
3. **02. 行业地位与核心护城河（`id="sec-moat"`）**：五维竞争壁垒雷达图（Radar Chart）及四大护城河深度拆解卡片。
4. **03. 同行竞争格局横向对比（`id="sec-comp"`）**：核心竞争对手关键技术/财务指标对比表 + 同业营业利润率横向条形图。
5. **04. 财务健康度与估值重置体系（`id="sec-fin"`）**：多年度营收、Non-GAAP 净利润与隐含远期 P/E 双轴演进图 + 资产负债表、现金流与股东回报拆解。
6. **05. 投资决策与多空风险配置（`id="sec-inv"`）**：牛市上行催化剂（Bull Case）与核心下行风险/证伪信号（Bear Risks）对照卡片。
7. **06. 季度跟踪 KPI 核查清单（`id="sec-kpi"`）**：6 项可勾选并支持 `localStorage`（包裹 `try/catch`）保存进度的下季财报核心验证清单。

## 4. 本地索引更新与 GitHub 自动提交流程

生成并自检 `<TICKER>.html` 无语法错误（可通过 `node --check` 校验内嵌脚本块）后，依次在本地终端执行（参见 [Gemini CLI Shell 工具说明](https://google-gemini.github.io/gemini-cli/docs/tools/shell.html)）：

```bash
# 1. 若本地存在离线同步脚本，同步更新 index.html 内置列表（可选，线上已支持 GitHub API 自动发现）
if [ -f "自动扫描更新网站.py" ]; then
  python3 自动扫描更新网站.py
fi

# 2. 拉取远端最新提交，避免多人或多端冲突
git pull --rebase origin main

# 3. 暂存新生成的财报网页及更新后的索引并提交推送
git add "<TICKER>.html" index.html
git commit -m "feat(report): add <TICKER> deep-dive financial research report"
git push origin main
```

推送完成后，向用户汇报新生成的本地文件路径、提交的 Commit 摘要，以及线上 [GitHub Pages](https://pages.github.com/) 访问地址：`https://antuotuo2000.github.io/investmentcareer/`。

## 5. 本地电脑首次配置速查（仅首次需要）

如果您尚未在本地电脑配置 [Gemini CLI](https://codelabs.developers.google.com/gemini-cli-hands-on)，只需在终端按顺序执行一次以下命令即可打通全流程：

```bash
# 1. 克隆您的财报仓库到本地并进入目录（确保本 GEMINI.md 位于仓库根目录）
git clone https://github.com/antuotuo2000/investmentcareer.git
cd investmentcareer

# 2. 全局安装 Google 官方 Gemini CLI（需本机已安装 Node.js 20+ 与 Git）
npm install -g @google/gemini-cli

# 3. 在仓库目录内启动 Gemini CLI（首次启动按提示用 Google 账号登录即可）
gemini
```

启动后，您只需在终端直接输入股票代码（例如 `AMD` 或 `ASML`），本地客户端便会自动读取本 `GEMINI.md` 规范，完成财报检索、网页生成与 GitHub 自动推送上线。
