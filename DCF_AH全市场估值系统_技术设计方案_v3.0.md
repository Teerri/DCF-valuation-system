# AH 全市场 DCF 估值系统 — 技术设计方案

> 日期：2026-05-04  
> 面向对象：设计与开发团队

---

## 1. 项目概述

### 1.1 系统目标

本系统面向全市场 AH 股的 DCF 估值，覆盖 A 股（沪深北三所，约 5,400 家）和港股（约 2,600 家）全部上市公司。系统支持数据拉取、筛选、质检与批量估值计算，最终输出每家可计算公司的内在价值、市价偏差与行业内估值信号，为投资决策提供投资参考。

### 1.2 技术栈总览

| 类别 | 技术选型 | 说明 |
|------|----------|------|
| 语言 | Python 3.10+ | 主要开发语言 |
| 数据源 | Wind Python API（WindPy） | 历史财务报表、股价、汇率、行业分类等 |
| 数据源 | 朝阳永续 API（SuntimePy） | 分析师一致预期数据（盈利增长率） |
| 并行 | `concurrent.futures.ProcessPoolExecutor` | 多进程并行计算引擎 |
| 结构化数据库 | MySQL 8.0+ | 财务原始数据、DCF 结果、任务状态持久化 |
| 时序数据存储 | Parquet | 历史股价、历史汇率 |
| Excel 输出 | openpyxl | 筛选、质检、估值结果、估值信号写入 Excel |
| 内存缓存 | Python `dict`（进程内） | 单次运行期间缓存当日股价、汇率 |
| 日志 | Python `logging` + JSON 格式化 | 结构化日志 |
| 测试 | pytest | 单元测试 |
| 配置管理 | YAML + python-dotenv | 参数与敏感信息管理 |
| 开发环境 | VSCode | 本地开发与调试 |

---

## 2. 系统架构

### 2.1 整体模块结构

```
dcf_ah_system/
│
├── universe/
│   └── universe.py                  # 股票池管理：全量标的维护、分类、AH 关联
│
├── data/
│   ├── wind_client.py               # Wind API 封装，负责全量历史财务数据拉取
│   ├── suntime_client.py           # 朝阳永续 API 封装，负责一致预期盈利增长率
│   ├── financial_fields.py          # 中国市场财务字段映射与原始科目计算规则
│   └── fx_handler.py                # 汇率获取与 AH 溢价计算
│
├── screening/
│   ├── sector_filter.py             # 金融类公司识别与剔除
│   ├── profitability_filter.py      # 亏损公司（EBIT≤0）筛选
│   └── data_quality.py             # 财务数据异常检测（计算前质检）
│
├── modeling/
│   ├── dcf.py                       # DCF 核心计算逻辑
│   ├── wacc.py                      # 动态 WACC 计算
│   └── terminal_value.py            # 终值计算
│
├── engine/
│   ├── runner.py                    # 多进程并行计算引擎
│   └── postprocess.py              # 后处理层：估值结果质检 + 行业信号生成
│
├── storage/
│   ├── db.py                        # MySQL 读写封装
│   ├── parquet_store.py             # Parquet 文件读写封装（历史股价、汇率）
│   └── schema.sql                   # MySQL 表结构定义
│
├── output/
│   └── excel_exporter.py            # Excel 输出模块（四个 Sheet）
│
├── logging/
│   └── logger.py                    # 结构化日志模块
│
├── config/
│   ├── settings.yaml                # 全局参数配置（增长率、WACC 分量、质检阈值等）
│   └── .env                         # 敏感信息（账密、数据库连接串）
│
└── main.py                          # 系统入口
```

### 2.2 系统架构

系统采用"数据拉取 → 筛选质检 → 批量计算 → 后处理输出"四阶段架构。


```
══════════════════════════════════════════════════════════════
  触发方式：运行 main.py
══════════════════════════════════════════════════════════════

  python main.py --mode full          # 全量运行
  python main.py --mode resume        # 断点续算
  python main.py --ticker 600519.SH   # 单股调试

══════════════════════════════════════════════════════════════
  第一阶段：数据拉取（ETL）
══════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │                      股票池同步                           │
  │  universe.py                                             │
  │  · 从 Wind 拉取 AH 全市场最新股票列表                   │
  │  · 更新新股/退市/停牌状态                                │
  │  · 写入 MySQL → companies 表                             │
  └──────────────────────────┬───────────────────────────────┘
                             │
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │                    全量财务数据拉取                        │
  │                                                          │
  │  wind_client.py（历史财务数据）                          │
  │  · 利润表原始科目 / 现金流量表原始科目 / 资产负债表      │
  │  · 企业价值数据（股本、债务、现金）                      │
  │  · 行业分类 / 当日股价                                   │
  │  · 结构化数据 → MySQL（financial_statements 等表）        │
  │  · 历史股价序列 → Parquet 文件（用于 Beta 计算）         │
  │  · 历史汇率序列 → Parquet 文件（用于历史回测）           │
  │                                                          │
  │  suntime_client.py（一致预期数据）                      │
  │  · 拉取各公司分析师一致预期盈利增长率                    │
  │  · 写入 MySQL → consensus_growth 表                      │
  │  · 无覆盖标的记录缺失，计算时自动回退                    │
  └──────────────────────────────────────────────────────────┘

══════════════════════════════════════════════════════════════
  第二阶段：筛选与质检
══════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │  从 MySQL 读取财务数据                                   │
  └──────────────────────────┬───────────────────────────────┘
                             │
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │                       筛选层                             │
  │  sector_filter.py        → 剔除金融类公司                │
  │  profitability_filter.py → 剔除 EBIT ≤ 0 公司           │
  │  → 筛选结果暂存内存，最终写入 Excel Sheet 1             │
  └──────────────────────────┬───────────────────────────────┘
                             │ 通过筛选的标的
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │                     数据质检层（计算前）                  │
  │  data_quality.py                                         │
  │  · 多维异常规则检测（关键字段缺失、比率极端异常等）      │
  │  · 未通过标记为 quality_fail                             │
  │  → 质检结果暂存内存，最终写入 Excel Sheet 2             │
  └──────────────────────────────────────────────────────────┘

══════════════════════════════════════════════════════════════
  第三阶段：多进程并行 DCF 计算
══════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │              并行计算引擎 runner.py                       │
  │  · ProcessPoolExecutor，多进程并行       │
  │  · 每个进程处理一批 ticker                               │
  │  · 单进程失败不影响整体（错误隔离）                      │
  │  · 任务状态实时写入 MySQL → task_status 表（支持断点续算）│
  └──────────────┬───────────────────────────────────────────┘
                 │ 每个进程内部
                 ▼
  ┌──────────────────────────────────────────────────────────┐
  │                     DCF 计算层                           │
  │  financial_fields.py → 原始科目 → DCF 变量重构          │
  │  wacc.py             → 动态 WACC（含 Beta 回归）         │
  │  dcf.py              → FCFF 复利预测 + NPV(FCF) + TV    │
  │  terminal_value.py   → 终值计算        │
  │  fx_handler.py       → H 股汇率换算 + AH 溢价           │
  │  → 计算结果实时写入 MySQL → dcf_results 表              │
  └──────────────────────────────────────────────────────────┘

══════════════════════════════════════════════════════════════
  第四阶段：后处理与输出
══════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │              后处理层 postprocess.py                     │
  │  · 估值结果质检（计算后）：PE implied、EV/EBIT、极端折溢价│
  │  · 行业内 Z-score 标准化                                 │
  │  · 行业内百分位数计算                                    │
  │  · 生成估值信号标签（深度低估/低估/中性/高估/极度高估）  │
  └──────────────────────────┬───────────────────────────────┘
                             │
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │              excel_exporter.py                           │
  │  按日期生成文件：dcf_YYYYMMDD.xlsx                       │
  │  ├── Sheet 1「筛选结果」  被剔除标的及原因               │
  │  ├── Sheet 2「质检结果」  未通过质检的标的及规则         │
  │  ├── Sheet 3「DCF估值」   全量估值结果                   │
  │  └── Sheet 4「估值信号」  行业内标准化信号与标签         │
  └──────────────────────────────────────────────────────────┘

══════════════════════════════════════════════════════════════
  贯穿全程：日志层
══════════════════════════════════════════════════════════════

  logger.py：每个阶段的成功/跳过/失败均产生结构化日志
  写入本地日志文件，同时关键记录写入 MySQL → run_log 表
```

### 2.3 断点续算机制

任务状态存储在 MySQL 的 `task_status` 表。使用者运行 `python main.py --mode resume` 时，系统自动查询当日 `pending` 或 `failed` 状态的标的，优先处理，跳过已为 `success` 的标的，无需从头运行。

---

## 3. 各模块详细设计

### 3.1 数据拉取层

#### 3.1.1 WindPy 核心函数

| 函数 | 支持多公司 | 支持多字段 | 适用场景 |
|------|-----------|-----------|----------|
| `w.wset` | — | — | 获取板块/指数成分列表（股票池） |
| `w.wsd` | 支持多公司，但同时只能取一个字段 | 支持多字段，但同时只能取一家公司 | 历史股价时间序列；汇率历史序列 |
| `w.wss` | **支持多公司** | **支持多字段** | **财务截面数据批量拉取** |

**关键约束**：`wss` 单次返回数据量上限为 **8,000 个单元格**（ticker 数 × 字段数乘积上限）。

#### 3.1.2 财务截面数据

DCF 所需从 Wind 拉取的原始财务科目共约 14 个字段（详见 3.1.3），单批次最大 ticker 数 = 8,000 ÷ 14 ≈ **571 家**。（*具体分批请求数量可按实际情况调整*）

```
A 股 5,400 家 ÷ 571 家/批 ≈ 10 批
港 股 2,600 家 ÷ 571 家/批 ≈  5 批
合计约 15 次 wss 调用完成全市场财务数据拉取
```

每批返回宽表 DataFrame（行为 ticker，列为字段名），入库时转为窄表写入 `financial_statements` 表。

#### 3.1.3 从 Wind 拉取原始数据

**利润表原始科目（wss 批量拉取）**

| Wind 字段名 | 中文科目 | 用途 |
|------------|---------|------|
| `profit_oper` | 营业利润 | 计算 EBIT 的基础 |
| `fin_exp_is` | 财务费用（利润表科目） | EBIT 计算备用（附注未披露时） |
| `int_exp_is` | 利息支出（附注明细） | EBIT 计算优先使用 |
| `int_inc_is` | 利息收入（附注明细） | EBIT 计算优先使用 |
| `inc_tax` | 所得税费用 | 计算有效税率 |
| `profit_tot` | 利润总额（EBT） | 计算有效税率分母 |

**现金流量表原始科目（wss 批量拉取）**

| Wind 字段名 | 中文科目 | 用途 |
|------------|---------|------|
| `depr_fa_coga_dpba` | 固定资产折旧、油气资产折耗、生产性生物资产折旧 | D&A 组成部分 |
| `amort_intang_assets` | 无形资产摊销 | D&A 组成部分 |
| `amort_lt_deferred_exp` | 长期待摊费用摊销 | D&A 组成部分 |
| `pay_acq_const_fiolta` | 购建固定资产、无形资产和其他长期资产支付的现金 | CapEx 来源 |

**资产负债表原始科目（wss 批量拉取，需拉取两期用于计算 ΔNWC）**

| Wind 字段名 | 中文科目 | 用途 |
|------------|---------|------|
| `tot_cur_assets` | 流动资产合计 | NWC 计算 |
| `tot_cur_liab` | 流动负债合计 | NWC 计算 |
| `tot_assets` | 总资产 | 数据质检、WACC |
| `tot_liab` | 总负债 | 数据质检 |
| `tot_non_cur_assets` | 非流动资产合计 | 备用 |
| `monetary_cap` | 货币资金（现金及等价物） | 权益价值计算 |
| `tot_debt` | 有息负债合计（含短期+长期借款） | 权益价值计算 |
| `float_a_shares` | 流通 A 股股本（A 股用） | 每股价值计算 |
| `tot_shrhdr_eqy` | 股东权益合计 | 备用 |

**港股补充说明**：港股利润表无"营业利润"科目，使用 `profit_before_tax`（除税前溢利）+ 利息支出还原 EBIT。字段名通过 `financial_fields.py` 中的映射表处理，计算逻辑见 3.5.1。

#### 3.1.4 各类数据的接口选型

| 数据类型 | 接口 | 存储位置 | 说明 |
|----------|------|----------|------|
| 全市场股票列表 | `w.wset` | MySQL `companies` | 单次获取 |
| 行业分类 | `w.wss` | MySQL `companies` | 按批次 |
| 利润表 / 现金流量表 / 资产负债表原始科目 | `w.wss` | MySQL `financial_statements` | 每批 ≤571 家 |
| 历史股价序列（Beta 计算用） | `w.wsd` | Parquet 文件 | 前复权，周频，3年 |
| 当日收盘价（估值比较用） | `w.wss` | 进程内字典缓存 | 单字段，当日 |
| CNY/HKD 历史汇率 | `w.wsd` | Parquet 文件 | 日频序列 |
| 当日汇率 | `w.wsd` | 进程内字典缓存 | 单次 |
| 一致预期盈利增长率 | 朝阳永续 API | MySQL `consensus_growth` | 批量拉取 |

#### 3.1.5 时序数据：Parquet 存储结构

历史股价和汇率数据为高频时间序列，采用列式 Parquet 文件存储，支持 pandas / polars 向量化读取。

```
data_lake/
├── market_prices/
│   ├── 600519.SH.parquet     # 每只股票独立一个文件
│   ├── 000001.SZ.parquet
│   ├── 00700.HK.parquet
│   └── ...
└── fx/
    └── CNY_HKD.parquet       # 人民币/港元历史汇率
```

每个股价 Parquet 文件包含字段：`date`、`close_adj`（前复权收盘价）、`is_trading`（是否正常交易，停牌日标记为 False）。

读取时通过 `parquet_store.py` 封装，对外统一接口，屏蔽文件路径细节。

#### 3.1.6 API 限速与重试

每批 `wss` 请求之间建议间隔 0.5～1 秒，避免触发 Wind 服务端限速。失败自动重试最多 3 次，退避间隔为 1s / 2s / 4s；3 次均失败则记录 `API_ERROR`，将本批次所有 ticker 标记为 `fetch_failed`，不中断整体拉取。

---

### 3.2 朝阳永续数据接入

朝阳永续是国内分析师一致预期数据的行业标准，通过其独立 API 接入，用途是为 DCF 计算层提供**各公司预期盈利增长率**，替代固定假设值。

**拉取内容**

| 数据项 | 说明 | 写入表 |
|--------|------|--------|
| 未来 1～3 年净利润一致预期增速 | 卖方分析师加权平均，按机构影响力和发布时间双维度加权 | `consensus_growth` |
| 覆盖分析师数量 | 用于评估数据可信度 | `consensus_growth` |

**覆盖率与回退策略**

| 情形 | 处理方式 | 日志标记 |
|------|----------|----------|
| 有朝阳永续数据（沪深 300 约 90%，中证全指约 80%） | 直接使用，取 3 年预期均值 | — |
| 无覆盖的 A 股 | 采用申万二级行业中位数增速 | `growth_industry_fallback` |
| 港股（非港股通标的） | 采用 `settings.yaml` 默认值 | `growth_default_fallback` |

---

### 3.3 股票池层

从 Wind 维护 AH 全市场全量股票列表，对每只标的标记完整分类信息。

| 字段 | 取值示例 | 用途 |
|------|----------|------|
| `market` | `SH` / `SZ` / `BJ` / `HK` | 区分交易所 |
| `board` | `主板` / `科创板` / `创业板` / `北交所` / `港股主板` / `GEM` | 板块分组 |
| `sw_industry_l1` | 申万一级行业 | 行业筛选依据 |
| `sw_industry_l2` | 申万二级行业 | Beta / 增长率行业中位数回退 |
| `currency` | `CNY` / `HKD` | 汇率处理 |
| `is_ah` | `True` / `False` | 是否需要 AH 溢价计算 |
| `ah_pair_ticker` | `600519.SH` ↔ `00700.HK` | 两地代码关联 |
| `status` | `active` / `suspended` / `delisted` | 排除停牌/退市标的 |

每次运行时同步最新状态：新上市公司入库、退市公司更新为 `delisted`、停牌公司更新为 `suspended`。

---

### 3.4 筛选层

#### 3.4.1 金融类公司剔除

申万一级行业为"银行"或"非银金融"的全部剔除；对混合型集团，检查金融业务营收占比，超过 50% 则剔除。被剔除标的状态记录为 `skipped_financial_sector`，写入 Excel Sheet 1。

#### 3.4.2 亏损公司筛选

| 情形 | 标记状态 | 处理 |
|------|----------|------|
| 当期 EBIT ≤ 0，上期为正 | `ebit_negative_single_year` | 跳过，提示人工复核 |
| 连续 1 年 EBIT ≤ 0 | `unprofitable` | 跳过 |
| 连续 2 年及以上 EBIT ≤ 0 | `persistent_loss` | 跳过，额外标记 |

---

### 3.5 DCF 计算层

#### 3.5.1 财务指标本地计算规则

**EBIT（息税前利润）**

A 股和 H 股的利润表结构不同，需分别处理：

```
【A 股】
EBIT = 营业利润 + 利息费用净额
其中：
  利息费用净额（优先）= 利息支出（附注明细）- 利息收入（附注明细）
  利息费用净额（备用）= 财务费用（利润表科目，含汇兑损益等，近似值）
  → 年报/半年报已披露附注时用优先值；季报用备用值

【H 股】
EBIT = 除税前溢利 + 利息费用净额
其中：利息费用净额同上取法
```

**EBT（税前利润）**

```
A 股：EBT = profit_tot（利润总额）
H 股：EBT = profit_before_tax（除税前溢利）
```

**有效税率**

```
税率 = inc_tax / profit_tot
约束：若税率 < 0 或 > 100%，触发数据质检规则，标记异常
```

**D&A（折旧与摊销）**



```
D&A = 固定资产折旧及油气资产折耗（depr_fa_coga_dpba）
    + 无形资产摊销（amort_intang_assets）
    + 长期待摊费用摊销（amort_lt_deferred_exp）
```

**CapEx（资本支出）**

从现金流量表投资活动直接取用，Wind 返回值为正数，计算时取负号（现金流出）：

```
CapEx = -pay_acq_const_fiolta
       （购建固定资产、无形资产和其他长期资产支付的现金）
```

**NWC 变化（营运资本变化）**

需要两期资产负债表数据（当期与上期）：

```
NWC_t   = tot_cur_assets_t   - tot_cur_liab_t
NWC_t-1 = tot_cur_assets_t-1 - tot_cur_liab_t-1
ΔNWC    = NWC_t - NWC_t-1
```

预测期 NWC 变化使用历史 3～5 年均值。

#### 3.5.2 核心 DCF 公式

**无杠杆自由现金流（FCFF）：**

```
FCFF = EBIT × (1 - 税率) + D&A + ΔNWC + CapEx
```

**EBIT 逐年预测（复利增长）：**

```
EBITₜ = EBIT₀ × (1 + g_earnings)^t
```

**盈利增长率（g_earnings）分年使用规则：**

```
第 1～3 年：使用朝阳永续 year_1 / year_2 / year_3 各年独立增速
第 4～5 年：使用三年均值，平滑过渡至永续增长率
```

**企业价值（EV）：**

```
EV = Σ [FCFFₜ / (1 + WACC)^t]  (t = 1 to n)
   + TV / (1 + WACC)^(n+1)

终值 TV = FCFFₙ × (1 + g) / (WACC - g)
```

**权益价值与每股内在价值：**

```
权益价值 = EV - 有息负债 + 货币资金
每股内在价值 = 权益价值 / 流通股数
```

#### 3.5.3 永续增长率 g 约束

永续增长率 g 必须满足以下强制约束，违反时自动修正并记录日志：

```
约束规则：
  g ≤ WACC - 1%          （防止终值趋向无穷大）
  g ≤ 4%                 （接近长期 GDP 增速上限）
  g ≥ 0%                 （不假设公司永久负增长）

执行顺序：先应用 g ≤ WACC - 1%，再截断至 [0%, 4%]
```

#### 3.5.4 WACC 计算

| 分量 | 数据来源 | 说明 |
|------|----------|------|
| 无风险利率（Rf） | Wind | A 股：10 年期中债到期收益率；H 股：10 年期港债 |
| 股权风险溢价（ERP） | `settings.yaml` 可配置 | 初版默认：A 股 7%，H 股 6% |
| Beta | 从 Parquet 历史股价本地计算 | 详见 3.5.5 |
| 债务成本（Kd） | 利息支出 / 平均有息负债（财务报表计算） | — |
| 资本结构权重 | 基于市值与账面债务计算 | — |

```
权益成本 Ke = Rf + Beta × ERP
WACC = Ke × (E / (E+D)) + Kd × (1 - 税率) × (D / (E+D))
```

当个别公司 WACC 计算异常时（如 Ke < 0 或 WACC > 30%），采用申万二级行业中位数 WACC，并记录 `wacc_industry_fallback`。

#### 3.5.5 Beta 计算规范

| 规范项 | 要求 |
|--------|------|
| 价格类型 | 前复权收盘价（从 Parquet 文件读取 `close_adj` 字段） |
| 收益率频率 | 周频收益率 |
| 计算窗口 | 过去 3 年（约 156 个交易周） |
| 最小样本数 | ≥ 80 个有效周频数据点（停牌周剔除） |
| 停牌处理 | `is_trading = False` 的周次不纳入回归 |
| 基准指数 | A 股：中证全指；H 股：恒生综合指数 |
| 回退规则 | 样本数不足（< 80）时，采用申万二级行业中位数 Beta，记录 `beta_industry_fallback` |

---

### 3.6 AH 汇率

**汇率处理**：H 股财务数据以人民币列报，股价以港元计价。DCF 计算在人民币口径下完成，最终每股内在价值按当日 CNY/HKD 汇率换算为港元，与港股市价比较。当日汇率缓存在进程内字典；历史汇率从 Parquet 文件读取。

**AH 溢价计算**：对 `is_ah = True` 的标的，额外输出：

```
AH 溢价率 = (A 股价格 / (H 股价格 × CNY/HKD汇率) - 1) × 100%
```

输出字段：A 股每股内在价值（CNY）、H 股每股内在价值（HKD 和 CNY 两套）、当前 AH 溢价率、该溢价率在历史数据中的分位数（从 `dcf_results` 表历史记录计算）。

---

### 3.7 并行计算引擎

#### 3.7.1 并行架构

使用 `ProcessPoolExecutor` 实现多进程并行：

- 以单个 ticker 为最小任务单元，分配至不同进程
- 进程数在 `settings.yaml` 中配置
- 每个进程独立捕获异常，单进程失败不影响整体
- 进程间不共享内存，当日股价、汇率在各进程内独立从缓存/Parquet 读取
- 任务状态（running / success / failed）实时写入 MySQL `task_status` 表

#### 3.7.2 计算结果写入

每个任务完成后立即将结果写入 `dcf_results` 表，不等待全量完成再统一写入，保证断点续算时已完成的结果不丢失。

---

### 3.8 后处理层

第三阶段完成后，从 `dcf_results` 表读取本次全量结果，依次执行以下操作：

#### 3.8.1 估值结果质检（计算后）

在 DCF 计算完成后对结果本身进行合理性检验，防止参数问题导致看似合理的极端估值进入最终输出：

| 检查规则 | 阈值 | 处理 |
|----------|------|------|
| 隐含 PE（每股内在价值 / EPS）> 100 倍 | 过高，可能增长率假设过于乐观 | 标记 `valuation_outlier` |
| EV / EBIT > 50 倍 | 极端倍数 | 标记 `valuation_outlier` |
| 折溢价率绝对值 > 200% | 极端偏差 | 标记 `valuation_outlier` |
| 权益价值为负 | 企业价值低于净债务 | 标记 `negative_equity_value` |

被标记的标的在 Excel Sheet 3 中保留但以特殊颜色标注，不纳入 Sheet 4 的信号生成。

#### 3.8.2 行业内估值信号生成

对通过结果质检的标的，在申万一级行业内进行横截面标准化，生成可比较的估值信号：

**Z-score 标准化**（行业内）：

```
valuation_zscore = (折溢价率_i - 行业均值) / 行业标准差
```

**百分位数**（行业内排序）：

```
valuation_percentile = 该标的在行业内折溢价率的升序百分位
```

**估值信号标签**：

| 行业内百分位 | 信号标签 |
|-------------|---------|
| < 10% | 深度低估 |
| 10% ～ 30% | 低估 |
| 30% ～ 70% | 中性 |
| 70% ～ 90% | 高估 |
| > 90% | 极度高估 |

---

### 3.9 数据存储设计

#### 3.9.1 存储分工总览

| 数据类型 | 存储位置 | 原因 |
|----------|----------|------|
| 股票基础信息 | MySQL `companies` | 结构化，每次运行更新 |
| 财务报表原始科目 | MySQL `financial_statements` | 结构化，计算基础，长期复用 |
| 一致预期盈利增长率 | MySQL `consensus_growth` | 结构化，每次运行更新 |
| DCF 计算结果 | MySQL `dcf_results` | 结构化，支持历史回溯与 AH 溢价历史分位数查询 |
| 任务运行状态 | MySQL `task_status` | 结构化，断点续算依赖 |
| 历史股价序列 | Parquet `data_lake/market_prices/` | 时序，列式存储，Beta 回归向量化读取 |
| 历史汇率序列 | Parquet `data_lake/fx/` | 时序，列式存储 |
| 当日股价 / 汇率 | 进程内字典缓存 | 仅当次运行使用，无需持久化 |
| 筛选结果 | Excel Sheet 1 | 面向人工查阅 |
| 质检结果 | Excel Sheet 2 | 面向人工查阅 |
| DCF 估值结果 | Excel Sheet 3 | 面向人工查阅与投资分析 |
| 行业估值信号 | Excel Sheet 4 | 面向投资分析 |

#### 3.9.2 MySQL 表结构

**companies 表**

| 字段 | 类型 | 说明 |
|------|------|------|
| ticker | VARCHAR(20) PK | 股票代码 |
| name | VARCHAR(100) | 公司名称 |
| market | VARCHAR(10) | SH / SZ / BJ / HK |
| board | VARCHAR(20) | 板块 |
| sw_industry_l1 | VARCHAR(30) | 申万一级行业 |
| sw_industry_l2 | VARCHAR(30) | 申万二级行业 |
| currency | VARCHAR(10) | CNY / HKD |
| is_ah | TINYINT(1) | 是否 A+H 两地上市 |
| ah_pair_ticker | VARCHAR(20) | 对应另一市场的代码 |
| status | VARCHAR(20) | active / suspended / delisted |
| updated_at | DATETIME | 最后更新时间 |

**financial_statements 表**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT AUTO_INCREMENT PK | 主键 |
| ticker | VARCHAR(20) | 股票代码 |
| report_date | DATE | 报告期 |
| period_type | VARCHAR(10) | annual / semi / quarter |
| profit_oper | DECIMAL(20,2) | 营业利润（原始科目） |
| int_exp_is | DECIMAL(20,2) | 利息支出（附注明细） |
| int_inc_is | DECIMAL(20,2) | 利息收入（附注明细） |
| fin_exp_is | DECIMAL(20,2) | 财务费用（利润表，备用） |
| inc_tax | DECIMAL(20,2) | 所得税费用 |
| profit_tot | DECIMAL(20,2) | 利润总额（EBT） |
| depr_fa_coga_dpba | DECIMAL(20,2) | 固定资产折旧及折耗 |
| amort_intang_assets | DECIMAL(20,2) | 无形资产摊销 |
| amort_lt_deferred_exp | DECIMAL(20,2) | 长期待摊费用摊销 |
| pay_acq_const_fiolta | DECIMAL(20,2) | 购建长期资产支付现金（CapEx） |
| tot_cur_assets | DECIMAL(20,2) | 流动资产合计 |
| tot_cur_liab | DECIMAL(20,2) | 流动负债合计 |
| tot_assets | DECIMAL(20,2) | 总资产 |
| tot_liab | DECIMAL(20,2) | 总负债 |
| monetary_cap | DECIMAL(20,2) | 货币资金 |
| tot_debt | DECIMAL(20,2) | 有息负债合计 |
| float_a_shares | DECIMAL(20,2) | 流通股数 |
| created_at | DATETIME | 入库时间 |

**consensus_growth 表**

| 字段 | 类型 | 说明 |
|------|------|------|
| ticker | VARCHAR(20) | 股票代码 |
| fetch_date | DATE | 拉取日期 |
| year_1_growth | DECIMAL(8,4) | 未来第 1 年净利润预期增速 |
| year_2_growth | DECIMAL(8,4) | 未来第 2 年净利润预期增速 |
| year_3_growth | DECIMAL(8,4) | 未来第 3 年净利润预期增速 |
| analyst_count | INT | 覆盖分析师数量 |
| data_source | VARCHAR(20) | suntime / industry_median / default |

**dcf_results 表**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT AUTO_INCREMENT PK | 主键 |
| ticker | VARCHAR(20) | 股票代码 |
| run_date | DATE | 计算日期 |
| report_date | DATE | 使用的财报期 |
| ebit_calculated | DECIMAL(20,2) | 本地计算的 EBIT 值 |
| tax_rate | DECIMAL(8,6) | 有效税率 |
| da | DECIMAL(20,2) | D&A 合计 |
| capex | DECIMAL(20,2) | CapEx |
| enterprise_value | DECIMAL(24,2) | 企业价值（元） |
| equity_value | DECIMAL(24,2) | 权益价值（元） |
| intrinsic_per_share | DECIMAL(12,4) | 每股内在价值（原币） |
| intrinsic_per_share_cny | DECIMAL(12,4) | 每股内在价值（CNY，H 股换算） |
| market_price | DECIMAL(12,4) | 当日市价 |
| valuation_gap | DECIMAL(8,4) | 折溢价率（内在价值/市价 - 1） |
| wacc | DECIMAL(8,6) | 使用的 WACC |
| beta | DECIMAL(8,4) | 使用的 Beta |
| earnings_growth_rate | DECIMAL(8,6) | 盈利增长率（均值） |
| capex_growth_rate | DECIMAL(8,6) | 资本支出增长率 |
| perpetual_growth_rate | DECIMAL(8,6) | 永续增长率（g，已应用约束） |
| forecast_years | INT | 预测期年数 |
| ah_premium | DECIMAL(8,4) | AH 溢价率（仅 A+H 股） |
| growth_data_source | VARCHAR(20) | 增长率数据来源 |
| beta_source | VARCHAR(20) | beta 来源（calculated / industry_fallback） |
| valuation_zscore | DECIMAL(8,4) | 行业内 Z-score |
| valuation_percentile | DECIMAL(8,4) | 行业内百分位数 |
| signal_label | VARCHAR(20) | 深度低估/低估/中性/高估/极度高估 |
| calc_status | VARCHAR(30) | success / failed / valuation_outlier / negative_equity_value |
| created_at | DATETIME | 入库时间 |

**task_status 表**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT AUTO_INCREMENT PK | 主键 |
| run_date | DATE | 运行日期 |
| ticker | VARCHAR(20) | 股票代码 |
| stage | VARCHAR(20) | fetch / screen / calc / postprocess |
| status | VARCHAR(30) | pending / running / success / failed / skipped_* / quality_fail |
| error_msg | TEXT | 失败时的错误信息 |
| updated_at | DATETIME | 最后更新时间 |

---

### 3.10 Excel 输出模块

每次全流程运行结束后，生成一份以日期命名的 Excel 文件：

```
output/
└── dcf_YYYYMMDD.xlsx
```

**Sheet 1：筛选结果**

| 列名 | 说明 |
|------|------|
| 股票代码 | ticker |
| 公司名称 | name |
| 市场 | SH / SZ / BJ / HK |
| 申万一级行业 | sw_industry_l1 |
| 剔除原因 | 金融类公司 / EBIT 为负（单年）/ EBIT 为负（连续）等 |
| 备注 | 具体数值或行业标签 |

**Sheet 2：质检结果**

| 列名 | 说明 |
|------|------|
| 股票代码 | ticker |
| 公司名称 | name |
| 报告期 | 使用的财报期 |
| 触发规则 | 如"EBIT/总资产超阈值"、"关键字段缺失"等 |
| 异常字段 | 触发规则的具体字段名 |
| 实际值 | 该字段的实际数值 |
| 阈值 | 对应规则的阈值说明 |
| 处理结果 | skip / manual_review |

**Sheet 3：DCF 估值结果**

| 列名 | 说明 |
|------|------|
| 股票代码 / 公司名称 / 市场 / 申万行业 | 基础信息 |
| 计算日期 / 使用财报期 | 计算信息 |
| EBIT（本地计算）/ 有效税率 / D&A / CapEx | 关键中间量，便于核验 |
| 企业价值 / 权益价值 / 每股内在价值 | 估值结果（含 H 股 CNY 换算） |
| 当日市价 / 折溢价率 | 与市价对比 |
| WACC / Beta / 盈利增长率 / 永续增长率 | 使用的假设参数 |
| 增长率数据来源 / Beta 来源 | 数据溯源 |
| AH 溢价率 | 仅 A+H 股填写 |
| 结果状态 | success / valuation_outlier / negative_equity_value |

**Sheet 4：估值信号**

| 列名 | 说明 |
|------|------|
| 股票代码 / 公司名称 / 市场 / 申万行业 | 基础信息 |
| 折溢价率 | 原始偏差 |
| 行业内 Z-score | valuation_zscore |
| 行业内百分位数 | valuation_percentile |
| 估值信号标签 | 深度低估 / 低估 / 中性 / 高估 / 极度高估 |

仅包含 `calc_status = 'success'` 的标的，`valuation_outlier` 不纳入此 Sheet。

---

### 3.11 数据质检层
计算前对从 MySQL 读取的财务数据执行以下检验。任一规则触发则标记 `quality_fail`，跳过计算，写入 Excel Sheet 2。阈值统一配置在 `settings.yaml`。

| 检查规则 | 阈值 | 说明 |
|----------|------|------|
| EBIT（本地计算值）/ 总资产 | 绝对值 > 50% | 正常企业通常在 -10%～30% 区间 |
| 营收同比变化 | 单年变化 > ±500% | 可能为重大重组或数据错误 |
| 资产负债率 | > 99% | 技术性资不抵债，数据存疑 |
| 关键原始科目缺失 | profit_oper / depr_fa_coga_dpba / pay_acq_const_fiolta 任一为 NULL | 无法完成 FCFF 计算 |
| 总资产异常 | 总资产 ≤ 0 | 数据错误 |
| 流通股数异常 | float_a_shares ≤ 0 | 无法计算每股价值 |
| 有效税率异常 | 税率 < 0 或 > 100% | 税费科目数据有误 |
| 现金流与利润严重背离 | 经营现金流 / 净利润 < -3 且连续 2 年 | 盈利质量极差信号 |

---

### 3.12 日志与错误隔离

| 级别 | 场景 | 示例 |
|------|------|------|
| INFO | 正常完成 | `[SUCCESS] 600519.SH 每股内在价值 ¥1,823.4，折价 12.3%，信号：低估` |
| INFO | 阶段汇总 | `[SUMMARY] 计算完成 3,241 家，跳过 1,832 家，失败 27 家` |
| WARNING | 跳过/降级 | `[SKIP] 601318.SH 行业=非银金融，已跳过` |
| WARNING | 参数回退 | `[FALLBACK] 00700.HK Beta 使用行业中位数替代` |
| WARNING | 质检失败 | `[QUALITY_FAIL] 000001.SZ EBIT/总资产=62%，超过阈值50%` |
| WARNING | g 约束修正 | `[G_CLAMP] 000858.SZ g 从 5.2% 修正至 WACC-1%=4.1%` |
| WARNING | 估值结果异常 | `[VALUATION_OUTLIER] 300750.SZ 隐含PE=143倍，标记异常` |
| ERROR | 计算报错 | `[ERROR] 02318.HK ZeroDivisionError at dcf.py line 87` |
| ERROR | API 失败 | `[API_ERROR] Wind 超时，ticker=600036.SH，重试 3/3 均失败` |

每个进程的计算在独立 try/except 块内执行，捕获异常后记录 ERROR 日志、更新 `task_status` 为 `failed`，继续处理下一个 ticker，不中断整体任务。

---

## 4. 补充设计建议

**财报更新追踪**：建立 `report_calendar` 表记录各公司预计财报披露日期，财报季期间建议更频繁地手动触发运行，确保估值与最新财务数据保持同步。

**参数配置集中管理**：WACC 分量（ERP 默认值）、增长率默认值、质检阈值、g 约束边界、信号分位数边界等全部配置在 `settings.yaml`。

**估值结果分布监控**：每次运行完成后，自动输出汇总统计——各市场/行业平均折溢价率分布、与上次运行结果偏差超过 30% 的标的列表，提示人工复核是否为数据或参数异常。

**基础单元测试**：覆盖 EBIT / D&A / CapEx / NWC 的本地计算逻辑、g 约束执行、Beta 计算、质检规则触发，使用 pytest 框架。

---

## 5. 快速启动

```bash
# 1. 安装依赖
pip install -r requirements.txt

# 2. 配置环境变量
cp .env.example .env
# 填写 WIND_USER, WIND_PASSWORD, MYSQL_URL, SUNTIME_API_KEY 等

# 3. 初始化数据库
mysql -u root -p < storage/schema.sql

# 4. 全量运行（第一次或需要更新全部数据）
python main.py --mode full

# 5. 断点续算（上次运行中断后继续）
python main.py --mode resume

# 6. 单股调试
python main.py --ticker 600519.SH

# 7. 仅重跑计算（数据已拉取，只重算 DCF）
python main.py --mode calc-only
```

---
## 6. 参考资料
[1] https://quant.go-goal.cn/#/help

[2] http://people.stern.nyu.edu/adamodar/pdfiles/eqnotes/dcfcf.pdf

[3] http://people.stern.nyu.edu/adamodar/pdfiles/basics.pdf

[4] https://www.oreilly.com/library/view/valuation-techniques-discounted/9781118417607/xhtml/sec30.html

[5] https://www.cchwebsites.com/content/calculators/BusinessValuation.html
