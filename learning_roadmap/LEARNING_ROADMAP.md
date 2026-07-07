# VeighNa (vnpy 4.4) 学习路线图（详细版）

> 本文档面向**有扎实工程背景、Python 熟练、懂基础金融概念但缺少实盘/量化实操经验**的学习者。
> 学习风格：**hands-on 优先**——每一步先跑起来看到结果，再回头读源码理解原理。
>
> **三大目标（按优先级）**
> 1. 快速上手开发「股市趋势跟踪策略（CTA）」与「多因子选股策略」并完成回测；
> 2. 吃透一条策略从「信号 → 委托 → 撮合/路由 → 成交 → 持仓 → 绩效」的完整链路及背后代码组件；
> 3. 把 VeighNa 当作 **harness 组件**，实现 AI 驱动的策略自动开发（程序化生成 → 回测 → 评测闭环）。
>
> **怎么用这份文档**：每个阶段都有「动手任务」和「自测清单」。建议把每个 `- [ ]` 当作可勾选的进度项，做完一项打勾一项。不要跳过动手任务——读懂 ≠ 会用。

---

## 目录

- [阅读前必须建立的 3 个认知](#阅读前必须建立的-3-个认知)
- [源码地图（速查表）](#源码地图速查表)
- [学习路线总览](#学习路线总览)
- [阶段 0 · 环境搭建 & 先跑通第一个回测](#阶段-0--环境搭建--先跑通第一个回测)
- [阶段 1 · 读懂框架的「数据语言」](#阶段-1--读懂框架的数据语言)
- [阶段 2 · 动手写第一个 CTA 趋势跟踪策略](#阶段-2--动手写第一个-cta-趋势跟踪策略)
- [阶段 3 · 吃透事件驱动内核](#阶段-3--吃透事件驱动内核)
- [阶段 4 · 一条策略的完整生命周期与代码组件](#阶段-4--一条策略的完整生命周期与代码组件)
- [阶段 5 · 多因子选股 / AI 量化（vnpy.alpha）](#阶段-5--多因子选股--ai-量化vnpyalpha)
- [阶段 6 · 把 VeighNa 当作 harness 组件](#阶段-6--把-veighna-当作-harness-组件)
- [阶段 7 · （可选）扩展与生产化](#阶段-7--可选扩展与生产化)
- [三大目标主线索引](#三大目标主线索引)
- [学习方法与工程实践](#学习方法与工程实践)
- [附录 A · 术语速查表](#附录-a--术语速查表)
- [附录 B · 常见坑合集](#附录-b--常见坑合集)

---

## 阅读前必须建立的 3 个认知

### 认知一：本仓库 = 核心框架，策略 App 大多是独立 pip 包

你 clone 的这个 repo 只包含**内核 + 通用组件 + AI 量化 alpha 模块**。CTA、组合、价差、期权等**策略应用是独立发布的 pip 包**，需要单独安装。`examples/` 里的示例会 `import vnpy_ctastrategy` 这类外部包。

| 位置 | 内容 | 是否在本仓库 |
|---|---|---|
| `vnpy/event` | 事件驱动引擎（全框架心脏） | ✅ |
| `vnpy/trader` | 交易内核：数据对象、主引擎、Gateway 抽象、数据库/数据源适配、Qt 界面 | ✅ |
| `vnpy/chart` | 高性能 K 线图表 | ✅ |
| `vnpy/rpc` | 跨进程 RPC（分布式） | ✅ |
| `vnpy/alpha` | **AI 多因子量化（自带独立回测引擎）** | ✅ |
| `vnpy_ctastrategy` | CTA 策略模板 `CtaTemplate` + CTA 回测引擎 + 示例策略 | ❌ pip 包 |
| `vnpy_portfoliostrategy` | 组合策略（多合约时序策略） | ❌ pip 包 |
| `vnpy_ctp` / `vnpy_ib` ... | 交易接口 Gateway | ❌ pip 包 |
| `vnpy_sqlite` / `vnpy_rqdata` ... | 数据库 / 数据服务适配 | ❌ pip 包 |

> **关键结论**：如果你找不到 `CtaTemplate`，不是你的错——它在 `vnpy_ctastrategy` 里，不在本仓库。而多因子（alpha）的一切都在本仓库 `vnpy/alpha`。

### 认知二：事件驱动是贯穿一切的主线

整个框架的运行时本质是一个**生产者-消费者事件循环**：

```
行情源/交易接口 (Gateway)  ──put(Event)──►  EventEngine 队列  ──dispatch──►  各引擎/策略的 handler
        ▲                                                                          │
        └──────────────────  send_order / subscribe  ◄────────────────────────────┘
```

- Gateway 收到行情或回报 → 构造 `Event` → `event_engine.put()`；
- `EventEngine` 后台线程从队列取事件 → 按 `event.type` 分发给注册的 handler；
- 策略/OMS/UI 都是 handler 的注册者。

理解了这张图，就理解了 vnpy 80% 的架构。

### 认知三：框架里有**两套**回测引擎，别混淆

| | CTA 回测引擎 | Alpha 回测引擎 |
|---|---|---|
| 所在 | `vnpy_ctastrategy.backtesting`（pip 包） | `vnpy/alpha/strategy/backtesting.py`（本仓库） |
| 面向 | **单标的时序**策略（择时/趋势） | **多标的截面**策略（选股/组合） |
| 策略基类 | `CtaTemplate`（`on_bar` 单根 K 线） | `AlphaStrategy`（`on_bars` 一个截面 dict） |
| 下单范式 | 手动 `buy/sell/short/cover` | **目标持仓** `set_target` + `execute_trading` 再平衡 |
| 信号来源 | 策略内部用指标计算 | 外部预先算好的 `signal_df`（模型预测） |
| 绩效 | 净值/回撤/夏普 | 额外有 `show_performance` 超额收益（Alpha）分析 |

目标 ① 的「趋势跟踪」走 CTA 引擎，「多因子选股」走 Alpha 引擎。

---

## 源码地图（速查表）

| 模块（文件） | 关键类 / 函数 | 作用 | 深入阶段 |
|---|---|---|---|
| `vnpy/event/engine.py` | `Event`, `EventEngine` | 事件对象 + 事件循环分发 | 3 |
| `vnpy/trader/object.py` | `TickData`/`BarData`/`OrderData`/`TradeData`/`PositionData`/`AccountData`/`ContractData` + `*Request` | 全框架数据结构 | 1 |
| `vnpy/trader/constant.py` | `Direction`/`Offset`/`Status`/`Exchange`/`Interval`/`OrderType`/`Product` | 枚举常量 | 1 |
| `vnpy/trader/engine.py` | `MainEngine`/`BaseEngine`/`OmsEngine`/`LogEngine` | 主引擎 + 功能引擎 | 3–4 |
| `vnpy/trader/gateway.py` | `BaseGateway` | 交易接口统一抽象 | 3–4 |
| `vnpy/trader/utility.py` | `BarGenerator`/`ArrayManager`/`round_to` | K 线合成 + 技术指标 | 2 |
| `vnpy/trader/converter.py` | `OffsetConverter`/`PositionHolding` | 开平仓/平今平昨换算 | 4 |
| `vnpy/trader/database.py` | `BaseDatabase`/`get_database` | 数据库适配器抽象 | 4 |
| `vnpy/trader/datafeed.py` | `BaseDatafeed`/`get_datafeed` | 数据服务适配器抽象 | 4 |
| `vnpy/trader/optimize.py` | `OptimizationSetting`/`run_bf_optimization`/`run_ga_optimization` | 穷举 & 遗传算法参数优化 | 2 |
| `vnpy/trader/setting.py` | `SETTINGS` | 全局配置 | 0 |
| `vnpy/alpha/lab.py` | `AlphaLab` | 投研流程与数据/模型/信号持久化中枢 | 5–6 |
| `vnpy/alpha/dataset/template.py` | `AlphaDataset` | 因子特征工程 + 数据切分 | 5 |
| `vnpy/alpha/dataset/datasets/` | `Alpha158`/`Alpha101` | 内置因子集（源自 Qlib） | 5 |
| `vnpy/alpha/model/` | `AlphaModel` + `lasso/lgb/mlp` | ML 模型模板与实现 | 5 |
| `vnpy/alpha/strategy/` | `AlphaStrategy`/`BacktestingEngine` | 截面策略模板 + 回测 | 5 |
| `vnpy/rpc/` | `RpcServer`/`RpcClient` | 跨进程分布式通讯 | 6 |
| `vnpy/chart/` | `ChartWidget` | 高性能 K 线绘图 | 7 |

---

## 学习路线总览

```
阶段0 环境+先跑通 ─► 阶段1 数据语言 ─┬─► 阶段2 CTA 趋势策略【目标①-择时】
                                     │
                                     └─► 阶段5 多因子选股【目标①-选股 + AI基础】
                                              │
阶段3 事件内核 ──► 阶段4 完整链路【目标②】────┤
                                              ▼
                              阶段6 harness 化：程序化/无界面/分布式【目标③】
                                              │
                                              ▼
                                     阶段7（可选）扩展与生产化
```

| 阶段 | 主题 | 预计时间 | 命中目标 |
|---|---|---|---|
| 0 | 环境搭建 & 先跑通回测 | 0.5–1 天 | 热身 |
| 1 | 数据对象与常量 | 1 天 | ①②地基 |
| 2 | CTA 趋势跟踪策略 | 2–3 天 | ① 择时 |
| 3 | 事件驱动内核 | 2 天 | ② 地基 |
| 4 | 策略完整生命周期 | 2–3 天 | ② |
| 5 | 多因子 / AI 量化 alpha | 3–5 天 | ① 选股 + AI |
| 6 | harness 化 / 程序化 / 分布式 | 3–5 天 | ③ |
| 7 | 扩展与生产化（可选） | 按需 | 进阶 |

> **最快摸到目标①的路径**：0 → 1 → 2（约 4–5 天出一个可回测的趋势策略）；选股线 0 → 1 → 5。
> **目标②必经**：3 → 4。**目标③落点**：6（地基是 4 + 5）。

---

## 阶段 0 · 环境搭建 & 先跑通第一个回测

**一句话目标**：把环境装好，跑出人生第一条策略净值曲线，建立整体手感。
**预计**：0.5–1 天 · **前置**：无

### 为什么先做这一步
hands-on 学习的第一原则是「先有正反馈」。在理解任何原理之前，先让一个完整的回测在你机器上跑出图来，你会立刻对「策略 / 回测 / 绩效」有直观认知，后面读源码时所有抽象都有了落点。

### 动手任务
- [ ] 安装底层依赖：`talib` 需要 C 库，先 `brew install ta-lib`（macOS）。
- [ ] 安装核心框架：仓库根目录执行 `bash install_osx.sh`（macOS）/ `bash install.sh`（Ubuntu）。脚本会用 `python3 -m pip` 安装 vnpy 及依赖（numpy、polars、talib 等）。
- [ ] 安装 CTA 应用包（示例依赖）：`python3 -m pip install vnpy_ctastrategy`。
- [ ] 打开并逐格运行 `examples/cta_backtesting/backtesting_demo.ipynb`。

它的核心骨架（记住这 6 步，后面反复出现）：

```python
from datetime import datetime
from vnpy.trader.optimize import OptimizationSetting
from vnpy_ctastrategy.backtesting import BacktestingEngine
from vnpy_ctastrategy.strategies.atr_rsi_strategy import AtrRsiStrategy

engine = BacktestingEngine()
engine.set_parameters(
    vt_symbol="IF888.CFFEX",   # 沪深300股指期货主力连续
    interval="1m",
    start=datetime(2019, 1, 1),
    end=datetime(2019, 4, 30),
    rate=0.3/10000,            # 手续费率
    slippage=0.2,              # 滑点
    size=300,                  # 合约乘数
    pricetick=0.2,             # 最小变动价位
    capital=1_000_000,
)
engine.add_strategy(AtrRsiStrategy, {})
engine.load_data()             # ① 载入历史数据
engine.run_backtesting()       # ② 逐根 K 线回放
df = engine.calculate_result() # ③ 逐日盯市盈亏
engine.calculate_statistics()  # ④ 计算夏普/回撤等
engine.show_chart()            # ⑤ 画净值/回撤/每日盈亏图
```

参数优化（跑完基础回测后再试）：

```python
setting = OptimizationSetting()
setting.set_target("sharpe_ratio")          # 优化目标
setting.add_parameter("atr_length", 25, 27, 1)
setting.add_parameter("atr_ma_length", 10, 30, 10)
engine.run_ga_optimization(setting)         # 遗传算法优化
```

### 概念补给站（轻量）
- **回测**：用历史数据模拟策略下单，评估其历史表现。
- **K 线 OHLCV**：一段时间的 开/高/低/收 价格 + 成交量。
- **vt_symbol = `代码.交易所`**：如 `IF888.CFFEX`。这是 vnpy 全局唯一的合约标识。
- **CTA**：Commodity Trading Advisor，这里泛指**单标的择时/趋势**类策略。
- **rate/slippage/size/pricetick**：手续费率 / 滑点 / 合约乘数 / 最小变动价位——回测真实性的关键，别用默认值糊弄。

### 自测清单（验收标准）
- [ ] 能解释 `calculate_statistics()` 打印的：总收益率、年化收益、最大回撤、夏普比率分别是什么。
- [ ] 能在 `show_chart()` 的图上认出净值曲线、回撤区间、每日盈亏分布。
- [ ] 能说出把这个策略从「股指期货」换到「某只股票」需要改哪些 `set_parameters` 参数。

### 常见坑
- `talib` 安装失败：几乎都是缺 C 库，先装 `ta-lib` 再 `pip install`。
- 示例回测报「找不到数据」：CTA 回测默认从数据库读数据。demo 若无内置数据，需要先通过数据服务下载（见阶段 1 的数据部分）或换用 `examples/download_bars` 准备数据。
- 记住本仓库 `python3` 环境可能没装全 `polars/lightgbm/vnpy_ctastrategy`，按阶段所需逐步安装即可。

---

## 阶段 1 · 读懂框架的「数据语言」

**一句话目标**：掌握全框架的「名词表」——所有数据对象与枚举常量。
**预计**：1 天 · **前置**：阶段 0

### 为什么学这一步
vnpy 里所有东西——行情、委托、成交、持仓、账户——都是 `vnpy/trader/object.py` 里的 `@dataclass`。后面每一行策略代码、每一个引擎方法，参数和返回值都是这些对象。先把「名词」认全，读代码才不会卡壳。

### 核心概念（含金融补给）
委托的两个正交维度（**理解这个就理解了下单**）：

| | `Offset.OPEN`（开仓） | `Offset.CLOSE`（平仓） |
|---|---|---|
| `Direction.LONG`（买） | 买入开仓（做多） | 买入平仓（平掉空头）= `cover` |
| `Direction.SHORT`（卖） | 卖出开仓（做空） | 卖出平仓（平掉多头）= `sell` |

- 期货有完整的「开/平」和「平今/平昨」；A 股通常只有买/卖（T+1、不能裸卖空）。
- `CtaTemplate` 的 4 个下单函数正是这 4 个组合：`buy`(多开) / `sell`(多平) / `short`(空开) / `cover`(空平)。

### 源码精读（`vnpy/trader/object.py` + `constant.py`）

**BaseData 的公共约定**：所有数据对象继承 `BaseData`，都带 `gateway_name`（数据来源接口）和 `extra`（扩展字段）。关键的**复合 ID 在 `__post_init__` 里自动拼接**：
- `vt_symbol = f"{symbol}.{exchange.value}"`
- `vt_orderid = f"{gateway_name}.{orderid}"`（委托全局唯一）
- `vt_tradeid`、`vt_positionid = gateway.vt_symbol.direction`、`vt_accountid` 同理。

**必须熟记的对象**：
- `TickData`：逐笔/切片行情。含 `last_price`、`volume`、**5 档盘口** `bid_price_1..5` / `ask_price_1..5` 及对应 volume、`limit_up`/`limit_down`（涨跌停）。
- `BarData`：K 线。`open/high/low/close_price`、`volume`、`turnover`、`open_interest`、`interval`。
- `OrderData`：委托状态跟踪。`status`（`Status` 枚举）、`traded`（已成量）、`is_active()`（是否在 `{SUBMITTING, NOTTRADED, PARTTRADED}`）、`create_cancel_request()`。
- `TradeData`：单笔成交（一个委托可有多笔成交）。
- `PositionData`：持仓。`volume`、`frozen`、`price`（均价）、`pnl`、`yd_volume`（昨仓）。
- `AccountData`：`balance`、`frozen`、`available = balance - frozen`。
- `ContractData`：合约静态信息。`size`（乘数）、`pricetick`、`min_volume`、`net_position`（是否净持仓模式）、`history_data`（是否支持历史数据）、期权字段 `option_strike/type/expiry` 等。

**请求对象（策略/引擎发给 Gateway）**：`OrderRequest`（含 `create_order_data()`）、`CancelRequest`、`SubscribeRequest`、`HistoryRequest`、`QuoteRequest`。

**枚举（`constant.py`）**：`Direction(LONG/SHORT/NET)`、`Offset(NONE/OPEN/CLOSE/CLOSETODAY/CLOSEYESTERDAY)`、`Status(SUBMITTING/NOTTRADED/PARTTRADED/ALLTRADED/CANCELLED/REJECTED)`、`OrderType(LIMIT/MARKET/STOP/FAK/FOK/RFQ)`、`Exchange`（CFFEX/SHFE/DCE/CZCE/INE/GFEX/SSE/SZSE… 及海外）、`Interval(1m/1h/d/w/tick)`、`Product(EQUITY/FUTURES/OPTION/INDEX/ETF/…)`。

### 动手任务
- [ ] 用 `examples/download_bars/download_bars.ipynb` 或 `examples/alpha_research/download_data_*.ipynb` 下载一段 K 线，`print` 出 `BarData` 逐字段观察。
- [ ] 在 Python REPL 里手动构造一个 `OrderRequest`，调用 `.create_order_data("test_id", "MYGW")`，打印它的 `vt_orderid`、`is_active()`。
- [ ] 画一张对象关系图：一次「买入开仓」如何从 `OrderRequest` → `OrderData`（Status 演变）→ `TradeData` → `PositionData`。

### 自测清单
- [ ] 不看代码，说出 `vt_symbol` / `vt_orderid` / `vt_tradeid` 的拼接规则。
- [ ] 解释 `OrderData.is_active()` 在什么状态下返回 `True`，以及它在撤单/OMS 里的用途。
- [ ] 说清 `ContractData.size` 和 `pricetick` 为什么对回测盈亏计算至关重要。

### 常见坑
- 把 `symbol` 当成 `vt_symbol`：几乎所有引擎查询用的都是 `vt_symbol`（带交易所后缀）。
- 忽略 `size`（合约乘数）：期货 1 手 = `size` 份标的，盈亏 = 价差 × 手数 × size，算错会离谱。

---

## 阶段 2 · 动手写第一个 CTA 趋势跟踪策略

**一句话目标**：独立写出并回测一个趋势跟踪策略，跑顺「策略开发」这件事。
**预计**：2–3 天 · **前置**：阶段 0、1 · **命中目标 ①（择时）**

### 为什么学这一步
这是你最想要的「快速上手 CTA」。CTA 策略是理解 vnpy 策略范式最简单的入口：单标的、时序、回调驱动。掌握它，你就能把「一个想法」变成「可回测的策略」。

### 核心概念（含金融补给）
- **趋势跟踪**：假设价格有惯性，用指标（均线、突破、ATR 通道）判断趋势方向并顺势持仓。
- **技术指标**：均线（SMA/EMA）、RSI、ATR（波动率）、布林带（BOLL）等——都由 `ArrayManager` 封装的 talib 提供。
- **过拟合**：参数优化跑出的「最优参数」往往只是对历史曲线的拟合，样本外可能完全失效。**这是量化最大的陷阱**。

### 源码精读

**CtaTemplate（在 `vnpy_ctastrategy`）——策略生命周期回调**：
- `on_init()`：策略初始化，通常在这里 `load_bar()` 预热指标；
- `on_start()` / `on_stop()`：启停；
- `on_tick(tick)` / `on_bar(bar)`：行情推送（tick 级 / K 线级）；
- `on_trade(trade)` / `on_order(order)` / `on_stop_order(so)`：成交/委托/停止单回报；
- 下单：`buy/sell/short/cover`、`cancel_all()`；
- 类属性 `parameters`（可优化参数名列表）、`variables`（运行时变量，会被界面/日志跟踪）。

**两个必备工具（本仓库 `vnpy/trader/utility.py`）**：

1. `BarGenerator(on_bar, window, on_window_bar, interval, daily_end)`
   - `update_tick(tick)`：把 tick 聚合成 1 分钟 `BarData`（用 `last_price` 更新 OHLC，用 `volume` 增量累加成交量），跨分钟时回调 `on_bar`；
   - `update_bar(bar)`：把 1 分钟 bar 合成 x 分钟 / x 小时 / 日线窗口 bar，完成时回调 `on_window_bar`；
   - 约束：**x 分钟的 x 必须能整除 60**（2/3/5/6/10/15/20/30）；小时线 x 任意；日线必须传 `daily_end`（收盘时间）。

2. `ArrayManager(size=100)`
   - 内部维护 OHLCV 的滚动 `numpy` 数组，`update_bar(bar)` 逐根推入，`inited` 表示已填满；
   - 直接调用 talib 指标：`am.sma(n)`、`am.ema(n)`、`am.atr(n)`、`am.rsi(n)`、`am.boll(n, dev)`、`am.macd(...)`、`am.cci(n)` 等，返回最新值或整段序列（`array=True`）。

**价格对齐工具**：`round_to(value, pricetick)` 把价格取整到最小变动价位（下单价必须对齐，否则被交易所拒单）。

### 动手任务
- [ ] 精读 `vnpy_ctastrategy/strategies/` 下的 `atr_rsi_strategy.py` 和一个双均线示例，看懂 `on_init/on_bar` 里 `BarGenerator` + `ArrayManager` 的配合。
- [ ] **自己写一个双均线趋势策略**（骨架如下），在阶段 0 的回测引擎里跑通：

```python
from vnpy_ctastrategy import CtaTemplate, BarData
from vnpy.trader.utility import BarGenerator, ArrayManager

class DoubleMaStrategy(CtaTemplate):
    fast_window = 10
    slow_window = 20
    fixed_size = 1

    parameters = ["fast_window", "slow_window", "fixed_size"]
    variables = ["fast_ma0", "slow_ma0"]

    def __init__(self, cta_engine, strategy_name, vt_symbol, setting):
        super().__init__(cta_engine, strategy_name, vt_symbol, setting)
        self.bg = BarGenerator(self.on_bar)
        self.am = ArrayManager()
        self.fast_ma0 = 0.0
        self.slow_ma0 = 0.0

    def on_init(self):
        self.load_bar(10)          # 预热：加载 10 天历史 K 线喂给 on_bar

    def on_bar(self, bar: BarData):
        self.am.update_bar(bar)
        if not self.am.inited:
            return

        self.fast_ma0 = self.am.sma(self.fast_window)
        self.slow_ma0 = self.am.sma(self.slow_window)

        cross_over = self.fast_ma0 > self.slow_ma0
        cross_below = self.fast_ma0 < self.slow_ma0

        if cross_over and self.pos == 0:
            self.buy(bar.close_price, self.fixed_size)
        elif cross_below and self.pos > 0:
            self.sell(bar.close_price, abs(self.pos))

        self.put_event()           # 通知界面/日志更新 variables
```

- [ ] 跑一次 GA 参数优化（`OptimizationSetting` + `run_ga_optimization`），对比优化前后夏普，**并特意观察过拟合**：把优化选出的最优参数拿到另一段时间回测，看是否失效。
- [ ] 把标的换成一只**股票**（如 `600000.SSE`）跑一遍，感受股票 T+1、不能做空对 `sell/short` 的约束。

### 源码精读（参数优化 `vnpy/trader/optimize.py`）
- `OptimizationSetting.add_parameter(name, start, end, step)` 生成参数网格；`set_target(name)` 指定优化目标；`generate_settings()` 用笛卡尔积展开所有组合。
- `run_bf_optimization`：穷举，`ProcessPoolExecutor`（`spawn`）多进程并行。
- `run_ga_optimization`：遗传算法，基于 `deap` 的 `eaMuPlusLambda` + NSGA2 选择，带结果缓存避免重复评估。
- 两者都接受 `evaluate_func(setting) -> dict` 和 `key_func(result) -> float`——**这个「参数 → 指标」的函数式接口，正是阶段 6 做 AI harness 的关键复用点**。

### 自测清单
- [ ] 不看示例，独立写出一个可回测的趋势策略，解释每个回调何时被触发。
- [ ] 解释 `BarGenerator` 为什么需要、`ArrayManager.inited` 为什么要判断。
- [ ] 用自己的话说明「样本内最优 ≠ 样本外有效」，以及如何用样本外测试/滚动回测缓解过拟合。

### 常见坑
- `on_bar` 里指标还没 `inited` 就下单 → 用 `if not self.am.inited: return` 守卫。
- 下单价没 `round_to(pricetick)` → 实盘会被拒单。
- 把 `self.pos`（当前净持仓）判断漏掉，导致重复开仓。

---

## 阶段 3 · 吃透事件驱动内核

**一句话目标**：你已经会「用」了，回头彻底搞懂「引擎怎么转」。
**预计**：2 天 · **前置**：阶段 2 · **命中目标 ②（地基）**

### 为什么学这一步
CTA 策略的 `on_bar` 是「被谁、怎么调用的」？答案在事件引擎。这是整个框架的枢纽，理解它才能读懂后面所有引擎、Gateway、App 的协作方式，也是目标 ② 的地基。

### 源码精读

**`vnpy/event/engine.py`（全文仅 146 行，建议逐行读）**：
- `Event(type: str, data)`：类型字符串 + 任意数据；
- `EventEngine`：
  - `_queue`（`queue.Queue`）、`_thread`（处理线程 `_run`）、`_timer`（每 `interval` 秒 put 一个 `EVENT_TIMER`）；
  - `_handlers`（`defaultdict(list)`，按 type 索引）、`_general_handlers`（监听所有事件）；
  - `register(type, handler)` / `unregister` / `register_general` / `put(event)` / `start()` / `stop()`；
  - 核心分发 `_process(event)`：先分发给 `_handlers[event.type]`，再分发给 `_general_handlers`。

**`vnpy/trader/gateway.py`（`BaseGateway`）——事件的生产者**：
- 抽象方法（子类必须实现）：`connect / close / subscribe / send_order / cancel_order / query_account / query_position`；可选 `query_history / send_quote`。
- 回报回调 `on_tick/on_order/on_trade/on_position/on_account/on_contract/on_quote`——**每个都推送两个事件**：
  - 通用事件（如 `EVENT_TICK`）→ 给 OMS、界面等全局监听者；
  - **带 vt_symbol/vt_orderid 后缀的精准事件**（如 `EVENT_TICK + tick.vt_symbol`）→ 让某个策略只收自己关心的合约。
  - 这就是「策略只订阅自己合约」的实现机制。

**`vnpy/trader/engine.py`（`MainEngine` + 功能引擎）**：
- `MainEngine.__init__` 创建/启动 `EventEngine`，持有 `gateways / engines / apps / exchanges`；
- `init_engines()` 默认装载：`LogEngine`、`OmsEngine`、`EmailEngine`、`WechatEngine`；
- `add_gateway(cls)` / `add_app(cls)`（App = `BaseApp` + 其 `engine_class`）；
- `MainEngine` 把 `connect/subscribe/send_order/cancel_order/query_history` 转发给对应 Gateway；把 `get_tick/get_order/.../get_all_*` 转发给 `OmsEngine`。

### 动手任务
- [ ] **写一个最小事件循环**，亲手验证分发机制：

```python
from time import sleep
from vnpy.event import EventEngine, Event, EVENT_TIMER

def on_timer(event: Event):
    print("tick:", event.data)

ee = EventEngine()
ee.register(EVENT_TIMER, on_timer)
ee.start()
sleep(3)          # 每秒打印一次
ee.stop()
```

- [ ] 再注册一个自定义事件类型，手动 `put` 一个带 data 的 `Event`，观察 handler 收到的内容。
- [ ] 用调试器在阶段 2 策略的 `on_bar` 打断点，回溯调用栈，**画出**「Gateway 收到行情 → `put(EVENT_TICK+vt_symbol)` → EventEngine 分发 → 策略引擎 handler → 策略 `on_tick/on_bar`」的完整时序图。

### 自测清单
- [ ] 手绘 vnpy 事件流转架构图，标出生产者、队列、分发、消费者。
- [ ] 解释「通用事件」与「带后缀的精准事件」各自的用途。
- [ ] 说清 `MainEngine`、`EventEngine`、`OmsEngine`、`Gateway` 四者的关系与数据流向。

### 常见坑
- 以为事件是同步调用：其实是**异步**——`put` 只是入队，真正处理在后台线程。策略回调里做耗时操作会阻塞整个事件循环。
- handler 里抛异常：会影响同类型后续 handler，注意异常隔离。

---

## 阶段 4 · 一条策略的完整生命周期与代码组件

**一句话目标**：把「数据 → 信号 → 委托 → 撮合/路由 → 成交 → 持仓 → 绩效」在回测与实盘两种模式下都对着代码打通。
**预计**：2–3 天 · **前置**：阶段 3 · **命中目标 ②**

### 为什么学这一步
这是目标 ② 的正题。理解了这条链路，你就能回答「每一个策略产出（一笔委托/一笔盈亏）到底经过了哪些代码组件」——这是把框架当 harness、做二次开发的前提。

### 源码精读 —— 回测链路（以 CTA `BacktestingEngine` 为例，同时对照 alpha 版）

以本仓库的 `vnpy/alpha/strategy/backtesting.py` 为可读样本（CTA 版逻辑同构）：
- `run_backtesting()`：先 `strategy.on_init()`，然后把所有 `datetime` 排序后逐个 `new_bars(dt)`；
- `new_bars(dt)`：组装当前截面 `bars`（缺失合约用上一根收盘价 fill）→ 先 `cross_order()` 撮合 → 再 `strategy.on_bars(bars)` → `update_daily_close()`；
- `cross_order()`（**撮合规则，务必看懂**）：
  - 买单成交条件：`order.price >= bar.low_price`；卖单：`order.price <= bar.high_price`；
  - 还要求不是全天涨停/跌停（`limit_up/limit_down` 用前收 ±10% 估算）；
  - 成交价取更有利的一侧（买单 `min(order.price, open)`，卖单 `max(...)`）；
  - 生成 `TradeData`，更新 `cash`、手续费（`turnover * rate`）。
- `calculate_result()`：把成交按天归集到 `PortfolioDailyResult` / `ContractDailyResult`，逐日算 `holding_pnl`（持仓浮盈）+ `trading_pnl`（当日交易盈亏）- `commission`（手续费）= `net_pnl`；
- `calculate_statistics()`：由每日 `net_pnl` 累计出 `balance`（净值），再算最大回撤、年化、**夏普 = (日均收益 − 日无风险) / 收益标准差 × √年交易日**、收益回撤比等。

### 源码精读 —— 实盘链路

- 下单：策略 → `MainEngine.send_order(req, gateway_name)` → `Gateway.send_order()` → 交易所；Gateway 通过 `on_order/on_trade` 把回报**异步**推回事件引擎。
- **OMS（`OmsEngine`，`vnpy/trader/engine.py`）——实盘的中央缓存**：
  - 注册所有回报事件，维护 `ticks/orders/trades/positions/accounts/contracts/quotes` 字典 + `active_orders/active_quotes`；
  - `process_order_event` 会把不再活动的委托从 `active_orders` 移除；
  - **每个 gateway 首次收到 `contract` 事件时，自动创建一个 `OffsetConverter`**，后续用成交/持仓事件更新它。
- **持仓换算（`vnpy/trader/converter.py`）——期货平今平昨的关键**：
  - `PositionHolding` 分别跟踪多空的今仓/昨仓（`long_td/long_yd/short_td/short_yd`）及冻结量；
  - `convert_order_request(req, lock, net)` 把一个「平仓」请求拆成交易所要求的 `CLOSETODAY/CLOSEYESTERDAY`：
    - **上期所/能源所（SHFE/INE）**必须显式区分平今/平昨，`convert_order_request_shfe/net` 会自动拆单；
    - `lock` 锁仓模式：不平反向仓而是反向开仓；
    - `is_convert_required` 判断：只有**非净持仓**（`net_position=False`）合约才需要换算。
- **数据层适配器（策略无关的可替换后端）**：
  - `vnpy/trader/database.py`：`BaseDatabase` 抽象 + `get_database()` 工厂，默认 `vnpy_sqlite`，可换 MySQL/MongoDB/DolphinDB 等；
  - `vnpy/trader/datafeed.py`：`BaseDatafeed` 抽象 + `get_datafeed()`，接 RQData/迅投研/TuShare 等历史数据源。

### 动手任务
- [ ] 给阶段 2 的策略加详细 `write_log`，在回测里 **trace 一笔订单的完整生命周期**：`SUBMITTING → NOTTRADED → ALLTRADED`，打印每一步的价格与撮合判定。
- [ ] 阅读 `cross_order()`，用纸笔推演：某根 bar `low=99, high=101, open=100`，一个 `price=99.5` 的买单会不会成交？成交价多少？
- [ ] 写一段文字对比「回测撮合」与「实盘委托」在以下环节的差异：成交判定、成交价、回报时序（同步 vs 异步）、滑点来源。
- [ ]（可选）读 `converter.py` 的 `convert_order_request_shfe`，理解为什么在上期所交易必须区分平今平昨（涉及手续费差异）。

### 自测清单
- [ ] 对照源码，画出回测链路与实盘链路两张数据流图，标注各自经过的类。
- [ ] 解释 `OmsEngine` 在实盘中的角色，以及策略如何通过 `MainEngine.get_position/get_account` 查询状态。
- [ ] 说清 `net_pnl = holding_pnl + trading_pnl − commission` 每一项的含义。
- [ ] 解释 `OffsetConverter` 存在的必要性（如果没有它，期货平仓会出什么问题）。

### 常见坑
- 用回测的「理想撮合」直接推断实盘表现：实盘有排队、部分成交、滑点、拒单，务必留安全边际。
- 期货实盘不处理平今平昨 → 手续费暴增甚至无法平仓。

---

## 阶段 5 · 多因子选股 / AI 量化（vnpy.alpha）

**一句话目标**：掌握 4.0 旗舰的 AI 量化投研闭环，做出一个多因子选股策略。
**预计**：3–5 天 · **前置**：阶段 1、4 · **命中目标 ①（选股）+ AI 基础**

### 为什么学这一步
这是目标 ①「多因子选股」的正题，也是目标 ③「AI 驱动」的桥梁。`vnpy.alpha` 的设计理念源自微软 Qlib，把「数据 → 因子 → 模型 → 信号 → 组合回测」做成了标准化流水线。

### 核心概念（含金融补给）
- **截面 vs 时序**：CTA 是「单标的、按时间」；多因子是「某一天、横向比较一堆股票」，选出预测收益高的建仓。
- **因子（Feature）**：对每只股票每天算一个数值（如动量、波动率、估值），用来预测未来收益。
- **标签（Label）**：要预测的目标，通常是未来 N 日收益率。
- **IC（信息系数）**：因子值与未来收益的相关性，衡量因子有效性。
- **目标持仓再平衡**：不手动 buy/sell，而是设定每只票的「目标仓位」，引擎自动算差额下单。

### 源码精读 —— 投研五步流水线

**中枢：`AlphaLab`（`vnpy/alpha/lab.py`）** —— 一个基于目录的持久化工作台，`__init__(lab_path)` 会创建 `daily/minute/component/dataset/model/signal` 子目录 + `contract.json`：
- 行情：`save_bar_data` / `load_bar_data`（parquet）；`load_bar_df`（**关键**：把价格按首日收盘价归一化、算 `vwap`、把停牌日置为 NaN，产出建因子用的宽表）；
- 成分股：`save_component_data` / `load_component_symbols` / `load_component_filters`（跟踪指数成分随时间的变化，避免幸存者偏差）；
- 合约配置：`add_contract_setting(vt_symbol, long_rate, short_rate, size, pricetick)`（回测撮合要用）；
- 三类产物的存取：`save/load/list_*` for `dataset`（pickle）、`model`（pickle）、`signal`（parquet）。

**第 1 步 · 造因子：`AlphaDataset`（`vnpy/alpha/dataset/template.py`）**
- 构造：`AlphaDataset(df, train_period, valid_period, test_period, process_type)`——一次性传入训练/验证/测试三段时间；
- `add_feature(name, expression)`：支持**字符串表达式因子引擎**（Qlib 风格，如 `"$close / Ref($close, 5) - 1"`）或 polars 表达式；也可 `add_feature(name, result=df)` 直接塞算好的因子；
- `set_label(expression)`：设定预测标签；
- `add_processor("infer"/"learn", processor)`：数据预处理（缺失值填充、时序/截面标准化、去极值等，见 `vnpy/alpha/dataset/processor.py`）；
- `prepare_data()`：**多进程并行**计算所有表达式因子；`process_data()`：应用 processors；
- `fetch_learn(Segment.TRAIN)` / `fetch_infer(Segment.TEST)`：按 `Segment` 取对应切分的数据；
- `show_feature_performance(name)`：调用 `alphalens` 生成 IC/分组收益 tear sheet，直接评估单因子有效性；
- 内置因子集：`vnpy/alpha/dataset/datasets/` 的 `Alpha158`、`Alpha101`。

**第 2 步 · 训模型：`AlphaModel`（`vnpy/alpha/model/`）**
- 统一模板 `AlphaModel`（`fit` / `predict`），实现类：`lasso_model`（L1 线性）、`lgb_model`（LightGBM）、`mlp_model`（神经网络，你机器已装 `torch`）；
- 统一 API 意味着**换算法只改一行**，便于对比。

**第 3 步 · 生成信号**：`model.predict(dataset)` 产出每票每日的预测值 → 组装成含 `datetime, vt_symbol, signal` 的 `signal_df` → `lab.save_signal()`。

**第 4 步 · 组合回测：`AlphaStrategy` + `BacktestingEngine`（`vnpy/alpha/strategy/`）**
- `AlphaStrategy`（截面策略模板）：
  - `on_bars(bars: dict[str, BarData])`：每个截面回调一次；`get_signal()` 拿到当前 datetime 的模型预测；
  - **目标持仓机制**：`set_target(vt_symbol, target)` 设目标 → `execute_trading(bars, price_add)` 自动算 `diff = target − pos`，并智能拆成 `cover+buy`（转多）或 `sell+short`（转空）下单；
  - `get_pos/get_target/get_cash_available/get_holding_value/get_portfolio_value` 组合查询。
- `BacktestingEngine`（`vnpy/alpha/strategy/backtesting.py`）：
  - `set_parameters(vt_symbols, interval, start, end, capital, risk_free, annual_days)`；
  - `add_strategy(strategy_class, setting, signal_df)`——**信号作为入参注入**；
  - `run_backtesting()` → `calculate_result()` → `calculate_statistics()`；
  - `show_performance(benchmark_symbol)`：画**超额收益（Alpha）**、换手率、超额回撤（含成本）——选股策略的核心评估视角。

**第 5 步 · 迭代**：用 `show_feature_performance` / `show_signal_performance` 定位「因子有效性」还是「组合构建」的问题，回到对应步骤优化。

### 动手任务
- [ ] 安装依赖：`python3 -m pip install polars lightgbm scikit-learn alphalens-reloaded plotly`（`torch` 你已有）。
- [ ] 跑通 `examples/alpha_research/download_data_*.ipynb` 下数据入 `AlphaLab`。
- [ ] 跑通 `examples/alpha_research/research_workflow_lgb.ipynb`（或 `mlp`）完整五步，产出净值与超额收益图。
- [ ] **改一个自定义因子**：`add_feature("my_mom", "$close / Ref($close, 20) - 1")`，用 `show_feature_performance` 看它的 IC。
- [ ] 换模型对比：把 `lgb_model` 换成 `lasso_model`，对比夏普和超额收益。

### 自测清单
- [ ] 独立完成一遍「数据 → 因子 → 模型 → 信号 → 回测」闭环。
- [ ] 说清 `AlphaLab` 每一步的产物存在哪个目录、是什么格式。
- [ ] 解释「目标持仓再平衡」相比手动 buy/sell 的优势（组合场景）。
- [ ] 解释 `Segment` 三段切分为什么能缓解过拟合。

### 常见坑
- 用了未来数据（look-ahead bias）：标签用未来收益是对的，但**因子里绝不能混入未来信息**。
- 忽略成分股变化 → 幸存者偏差；用 `load_component_filters` 只在成分股在册期间持有。
- 停牌/涨跌停股无法成交却计入信号：注意数据清洗（`load_bar_df` 已把停牌置 NaN）。

---

## 阶段 6 · 把 VeighNa 当作 harness 组件

**一句话目标**：脱离 GUI，用纯代码编排框架，搭出「AI 驱动策略开发」的评测闭环骨架。
**预计**：3–5 天 · **前置**：阶段 4、5 · **命中目标 ③**

### 为什么学这一步
目标 ③ 的落点。AI 驱动策略开发的本质是一个循环：**（LLM/搜索）生成策略或参数 → 程序化回测 → 拿到结构化绩效 → 反馈给上层 → 再生成**。要实现它，必须能把 vnpy 的回测/交易当作**可编程、无副作用、可批量**的函数来调用。

### 源码精读 / 参考

**无界面（headless）运行**：
- `examples/no_ui/run.py`：经典的**父/子进程守护**模式——父进程按交易时段拉起子进程，子进程 `MainEngine + Gateway + CtaEngine`，`init_all_strategies()` / `start_all_strategies()` 后进入循环，非交易时段自动退出。这是实盘无人值守的标准骨架。
- `examples/veighna_trader/demo_script.py`：脚本化操作主引擎。
- `vnpy_scripttrader`：REPL / 脚本化下单（面向多标的、命令行交易）。

**程序化回测封装**（核心复用点）——把阶段 2 / 5 的回测各封装成纯函数：

```python
# CTA：一个 "参数 -> 绩效" 的纯函数，天然适配超参搜索 / AI 生成
from datetime import datetime
from vnpy_ctastrategy.backtesting import BacktestingEngine

def run_cta_backtest(strategy_cls, setting: dict, vt_symbol: str) -> dict:
    engine = BacktestingEngine()
    engine.set_parameters(
        vt_symbol=vt_symbol, interval="1m",
        start=datetime(2019, 1, 1), end=datetime(2020, 1, 1),
        rate=0.3/10000, slippage=0.2, size=300, pricetick=0.2,
        capital=1_000_000,
    )
    engine.add_strategy(strategy_cls, setting)
    engine.load_data()
    engine.run_backtesting()
    engine.calculate_result()
    return engine.calculate_statistics()   # -> dict(sharpe_ratio, max_ddpercent, ...)
```

```python
# 把它变成 AI/搜索循环的 evaluate 接口
def evaluate(setting: dict) -> float:
    stats = run_cta_backtest(DoubleMaStrategy, setting, "IF888.CFFEX")
    return stats["sharpe_ratio"]

# 上层可以是：网格/遗传（复用 vnpy.trader.optimize 的 evaluate_func/key_func 契约）、
# 贝叶斯优化，或 LLM 生成 setting 甚至生成整个策略类源码后动态加载再 evaluate
```

- **复用点提醒**：`vnpy/trader/optimize.py` 的 `run_bf_optimization/run_ga_optimization` 就是 `evaluate_func(setting)->dict` + `key_func(result)->float` 的契约。你的 AI harness 可以直接套这个契约，把「LLM 生成的参数/策略」接进去。
- Alpha 侧同理：把「造数据集 → 训模型 → 生成 signal_df → `BacktestingEngine.add_strategy(..., signal_df)` → `calculate_statistics()`」封装成一个函数，返回绩效 dict，供上层编排。

**分布式（`vnpy/rpc/`）**：
- `RpcServer` / `RpcClient`（基于 ZeroMQ 的 REQ/REP + PUB/SUB），把某个进程的行情/交易能力暴露为远程服务；
- 参考 `examples/simple_rpc/`（`test_server.py` / `test_client.py`）和 `examples/client_server/`；
- 用途：AI harness 并行跑大量回测时，可把「回测 worker」做成 RPC 服务横向扩展。

**官方 AI Agent 视角**：`docs/fusion/agent/`（`fusion_agent_intro.md` / `fusion_agent_workflow.md` / `fusion_agent_config.md`）——直接对标目标 ③，看官方如何把框架接入 AI agent 工作流。

### 动手任务
- [ ] 把阶段 2 的 CTA 回测和阶段 5 的 alpha 回测**各封装成一个返回绩效 dict 的纯函数**（无 GUI、无全局副作用）。
- [ ] 写一个最小 harness：外层给定若干组参数（先用随机/网格模拟「AI 生成」）→ 批量调用回测函数 → 收集绩效 → 按夏普排序输出 Top-K。
- [ ]（进阶）让「上层」真正生成策略：用模板字符串拼出一个策略类源码 → 动态 `exec`/importlib 加载 → 丢进回测函数评测，跑通「生成 → 回测 → 打分」闭环。
- [ ]（进阶）用 `examples/simple_rpc` 把回测 worker 做成 RPC 服务，验证多 client 并行提交回测任务。
- [ ] 读 `docs/fusion/agent/fusion_agent_workflow.md`，记录官方 agent 闭环与你的设计的异同。

### 自测清单
- [ ] 完全不开 GUI，用一个脚本驱动「（模拟）生成策略/参数 → 回测 → 拿到结构化绩效」的闭环。
- [ ] 你的回测函数是**幂等且无副作用**的吗？能安全并行吗？（注意：GA 优化用的是 `spawn` 多进程，留意全局状态）
- [ ] 解释你的 harness 如何水平扩展（多进程 / RPC）。

### 常见坑
- 回测函数里残留全局状态（如复用同一个 engine 实例）导致并行结果串味——每次评测新建 engine。
- 多进程下不可 pickle 的对象（如某些 lambda、闭包）会报错；GA/进程池注意可序列化。
- 动态加载 AI 生成的代码要有**沙箱/超时/资源限制**，避免恶意或死循环代码拖垮 harness。

---

## 阶段 7 · （可选）扩展与生产化

**一句话目标**：从「会用」到「能改、能上生产」。
**预计**：按需 · **前置**：阶段 4+

### 方向清单
- **自定义 Gateway**：实现 `BaseGateway` 的抽象方法，对接新交易接口（参考任一 `vnpy_*` gateway 包）。
- **自定义 Datafeed / Database**：实现 `BaseDatafeed` / `BaseDatabase`，接入自有数据源或时序库。
- **风控与组合**：`vnpy_riskmanager`（交易流控/下单限制）、`vnpy_portfoliomanager`（子账户盈亏跟踪）。
- **高性能绘图**：`vnpy/chart/`（`ChartWidget`，大数据量 K 线 + 实时更新）。
- **工程规范**（README 明确要求）：提交前 `ruff check .` + `mypy vnpy`；基于 `dev` 分支开发并走 PR 流程。
- **生产运维**：结合你的习惯用 **tmux 长跑** + 本地日志落盘（`SETTINGS["log.file"]=True`）；用 `vnpy_datarecorder` 录制实时行情用于回测/实盘初始化。

---

## 三大目标主线索引

**目标 ① 快速上手写策略**
- 趋势跟踪（择时）：阶段 0 → 1 → 2
- 多因子（选股）：阶段 1 → 5

**目标 ② 吃透策略完整链路与代码组件**
- 阶段 3（事件内核）→ 阶段 4（回测链路 + 实盘链路 + OMS + OffsetConverter + 数据层）

**目标 ③ 把 VeighNa 当 harness 做 AI 驱动开发**
- 地基：阶段 4 + 5 → 落点：阶段 6（程序化回测封装 + evaluate 契约 + RPC + `docs/fusion` agent）

---

## 学习方法与工程实践

1. **先跑 example，再读源码**：每阶段先获得可运行的正反馈，再回头理解原理。
2. **用调试器跟事件流**：在回调打断点看调用栈，比读文档更快理解异步事件驱动。
3. **每阶段画一张图**：数据对象关系图（1）→ 事件流转图（3）→ 策略链路图（4）→ alpha 流水线图（5）。画得出来才算真懂。
4. **长任务上 tmux + 日志落盘**：阶段 5/6 的模型训练与批量回测耗时长，避免网络波动中断丢结果。
5. **警惕过拟合贯穿始终**：任何「好看的回测」都先问一句——样本外还成立吗？有没有用到未来信息？
6. **文档三档对照**：`docs/community`（社区版，主力参考）、`docs/elite`（精英版策略）、`docs/fusion`（AI agent，服务目标 ③）。
7. **始终用 `python3`**。

---

## 附录 A · 术语速查表

| 术语 | 含义 |
|---|---|
| vt_symbol | 全局唯一合约标识 `代码.交易所`，如 `IF888.CFFEX` |
| vt_orderid | 全局唯一委托号 `gateway.orderid` |
| Tick / Bar | 逐笔切片行情 / K 线 |
| Direction / Offset | 买卖方向（多/空/净）/ 开平（开/平/平今/平昨） |
| OMS | 订单管理系统，`OmsEngine`，实盘中央数据缓存 |
| Gateway | 交易/行情接口适配层，事件生产者 |
| CTA | 单标的时序择时/趋势策略 |
| 截面 / 时序 | 某时点横向比较多标的 / 单标的沿时间 |
| 因子 / 标签 / IC | 预测特征 / 预测目标 / 因子与收益相关性 |
| 目标持仓 | 设定每标的目标仓位，引擎自动算差额下单 |
| 夏普比率 | (超额日均收益 / 收益标准差) × √年交易日 |
| 最大回撤 | 净值从峰值到谷值的最大跌幅 |

## 附录 B · 常见坑合集

- 找不到 `CtaTemplate` → 在 `vnpy_ctastrategy`（pip 包），不在本仓库。
- 混淆两套回测引擎 → CTA（`on_bar` 单标的）vs Alpha（`on_bars` 截面）。
- `talib` 装不上 → 先装底层 C 库 `ta-lib`。
- 指标未 `inited` 就下单 / 下单价未 `round_to(pricetick)` → 加守卫、对齐价格。
- 期货实盘不处理平今平昨 → 手续费暴增，依赖 `OffsetConverter`。
- 回测理想撮合直接当实盘 → 忽略排队/部分成交/滑点/拒单。
- 多因子用到未来信息 / 幸存者偏差 → 因子不含未来数据，用成分股时间过滤。
- AI harness 回测函数带全局副作用 → 无法安全并行，每次新建 engine。

---

> 本路线图基于对 VeighNa 4.4 源码的实际走读整理。随框架版本演进，个别文件/接口可能变化，以仓库最新代码为准。
