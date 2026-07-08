# 阶段 0 · 答疑记录（QA）

> 本文汇总学习阶段 0 过程中的问答，配合 `01_金融基础概念.md` 与 `02_股票交易对象流转.md` 阅读。
> 所有代码引用均指向本仓库真实源码（文件 + 行号）。

---

## 目录

- [Q1. status 和 OrderType 对应的金融概念是什么？](#q1-status-和-ordertype-对应的金融概念是什么)
- [Q2. 委托/成交/合约/行情具体指代什么？](#q2-委托成交合约行情具体指代什么)
- [Q3. Gateway 是什么？](#q3-gateway-是什么)
- [Q4. Tick 和 Bar 背后的完整交易流程是什么样的？](#q4-tick-和-bar-背后的完整交易流程是什么样的)
- [Q5. OMS 是什么对象？它存在的意义是什么？](#q5-oms-是什么对象它存在的意义是什么)
- [Q6. OMS 的 process_*_event() 结果如何通知策略端 / MainEngine？](#q6-oms-的-process_event-结果如何通知策略端--mainengine)
- [Q7. 为什么 on_order / on_trade 需要一次推两个事件？](#q7-为什么-on_order--on_trade-需要一次推两个事件)

---

## Q1. status 和 OrderType 对应的金融概念是什么？

### OrderType（订单类型）——「我这笔单子想以什么方式进入市场撮合」

市场的核心结构是**订单簿（Order Book）**：所有买卖报价按价格排成两列队伍，撮合机器在买方出价 ≥ 卖方要价时自动成交。买卖之间的缝隙叫**价差（spread）**。你下单有两种基本方式：

- **限价单 `LIMIT`**：指定价格，够不到就排队等。价格可控，但可能不成交。**量化最常用**。
- **市价单 `MARKET`**：不指定价格，立即成交。几乎一定成交，但价格不可控，会有**滑点（slippage）**——买多了会把卖一、卖二越吃越高，实际均价比看到的差。

其余类型 = 限价单 + 附加规则：

| `OrderType` | 中文 | 金融含义 |
|---|---|---|
| `LIMIT` | 限价 | 指定价格，排队等（最常用） |
| `MARKET` | 市价 | 立即成交，有滑点 |
| `STOP` | 停止单 | 价格突破 X 才触发下单（止损/追涨） |
| `FAK` | Fill and Kill | 立即成交能成的部分，剩余立刻撤销 |
| `FOK` | Fill or Kill | 要么全部立即成交，要么全部撤销 |
| `RFQ` | 询价 | 冷门品种没人挂单时，主动喊报价 |

### Status（委托状态）——「我这张委托现在走到人生哪一步了」

这是一台描述委托从生到死的**状态机**：

```
① SUBMITTING（提交中）  指令飞向交易所途中，还没确认
        ▼
② NOTTRADED（未成交/已挂单）  在订单簿里排队，还没成交
        ▼
③ PARTTRADED（部分成交）  成交了一部分，还剩一部分挂着
        ▼
④ ALLTRADED（全部成交）  ← 正常终点
```

两个非正常终点：
- **CANCELLED（已撤销）**：主动撤回未成交部分。
- **REJECTED（拒单）**：交易所直接拒绝，没进订单簿（价格没对齐、钱不够、超涨跌停等）。

**代码落点**：`OrderData.is_active()` 在状态属于 `{SUBMITTING, NOTTRADED, PARTTRADED}` 时返回 `True`（`ACTIVE_STATUSES`，见 `object.py:14`），金融含义是「这张委托还活着、还可能成交、因此还能撤销」。

> 详见 `01_金融基础概念.md` 第 2、3 节。

---

## Q2. 委托/成交/合约/行情具体指代什么？

它们是**同一件事的不同侧面**，用一次买入串起来：

| 名词 | 代码类 | 一句话 | 生活类比 |
|---|---|---|---|
| **委托 Order** | `OrderRequest` → `OrderData` | 你的下单意向 + 状态跟踪 | 你点的那道菜（订单） |
| **成交 Trade** | `TradeData` | 真实配对的一次交割（不可撤销） | 真端上桌的那盘菜 |
| **持仓 Position** | `PositionData` | 成交累积后你现在攥着多少 | 已吃进肚子的所有菜 |
| **账户 Account** | `AccountData` | 你的钱包（可用 = 总 − 冻结） | 钱包余额 |
| **合约 Contract** | `ContractData` | 标的物的规格说明书（静态） | 菜的份量/起订规格 |
| **行情 Market Data** | `TickData` / `BarData` | 市场实时/汇总的价格信息（输入） | 市场当前的报价牌 |

关键点：

- **委托 ↔ 成交是一对多**：一张委托可分多笔成交（大单被分批撮合）。成交不可撤，委托可撤。
- **合约为什么关键**：`pricetick`（最小变动价位）决定下单价必须对齐；`size`（合约乘数）决定期货 1 手 = 多少份标的，盈亏 = 价差 × 手数 × size，漏乘会离谱错。
- **行情两种粒度**：`TickData`（含 5 档盘口 + 涨跌停）是最细的实时快照；`BarData`（OHLCV）是一段时间的汇总 K 线。

### 委托的两个正交维度（理解这个 = 理解下单）

| | `Offset.OPEN`（开仓/建仓） | `Offset.CLOSE`（平仓/了结） |
|---|---|---|
| `Direction.LONG`（买入） | 买入开仓 = 做多 | 买入平仓 = 平空头 → `cover` |
| `Direction.SHORT`（卖出） | 卖出开仓 = 做空 | 卖出平仓 = 平多头 → `sell` |

- **Direction**：买还是卖；**Offset**：建仓还是了结。
- 对应 CTA 的 4 个下单函数：`buy`(多开)/`sell`(多平)/`short`(空开)/`cover`(空平)。
- A 股通常只有买/卖（`Offset` 填 `NONE`），T+1、不能裸卖空；期货有完整开/平，甚至平今/平昨。

> 详见 `01_金融基础概念.md` 第 4 节。

---

## Q3. Gateway 是什么？

**Gateway（网关/接口）= 你的程序和某一家券商/交易所之间的「翻译官 + 快递员」。**

**为什么需要它**：全世界券商、交易所的通信协议各不相同（CTP、XTP、IB API……），数据格式、函数名都不一样。若策略直接对接每一家，换券商就得重写策略。

**VeighNa 的解法**：定义统一抽象接口 `BaseGateway`（`vnpy/trader/gateway.py`），规定任何接口都必须实现标准动作：

- **对外发指令**：`connect` / `subscribe` / `send_order` / `cancel_order` / `query_account` / `query_position` / `query_history`。
- **收回报**：`on_tick` / `on_order` / `on_trade` / `on_position` / `on_account` / `on_contract`。

每接入一家券商写一个具体子类（如 `vnpy_ctp`），负责把「券商私有格式」翻译成「vnpy 标准对象」。

> 一句话：Gateway 把「五花八门的券商协议」翻译成「vnpy 统一的数据对象和标准动作」。它既是行情的**生产者**（推 `TickData`），也是指令的**执行者**（发 `OrderRequest`）。
>
> 回测里没有真实券商，是 `BacktestingEngine` 用历史数据模拟了 Gateway 的角色。

---

## Q4. Tick 和 Bar 背后的完整交易流程是什么样的？

### Tick（逐笔/切片行情）——「市场的每一次心跳」

市场状态变化时（有新成交、盘口变化），交易所推送一张实时快照，内含 `last_price`、`volume`、5 档盘口、涨跌停价。中国股票/期货通常每 0.5 秒一张。

### Bar（K 线）——「把一段时间的 tick 打包成一根蜡烛」

一段时间内所有 tick 汇总成 OHLCV：开盘价（第一笔）、最高价、最低价、收盘价（最后一笔）、总成交量。真实市场只推 tick，vnpy 用 `BarGenerator` 在程序里把 tick 实时"攒"成 bar。

### 一个 Tick 背后的完整链路（把所有名词用上）

```
1. 交易所推 TickData → Gateway.on_tick() → 推进事件系统
2. 策略在 on_tick(tick) 里决策 → 生成 OrderRequest(LONG, OPEN)
3. MainEngine.send_order → Gateway.send_order → 交易所；账户资金被 frozen
4. OrderData 状态机启动：SUBMITTING → NOTTRADED（Gateway.on_order 推回）
5. 撮合成交：可能分批 PARTTRADED → ALLTRADED；每笔成交 Gateway.on_trade 推回 TradeData
6. 持仓/资金更新：PositionData（volume、均价），AccountData（frozen 释放）
7. 下一个 Tick 到来，循环回到第 1 步
```

**Tick 级 vs Bar 级策略**：本质一样，只是决策节拍不同。Tick 级每 0.5 秒跑一次 `on_tick()`（高频）；Bar 级攒够一根 K 线才跑 `on_bar()`（趋势跟踪、多数中低频，如双均线）。触发后下单→成交→持仓流程完全一致。

> 详见 `01_金融基础概念.md` 第 6 节；实盘对象流转详见 `02_股票交易对象流转.md`。

---

## Q5. OMS 是什么对象？它存在的意义是什么？

### 它是什么

OMS = **Order Management System（订单管理系统）**，代码里就是 `OmsEngine`，是挂在 `MainEngine` 下的功能引擎（继承 `BaseEngine`），`MainEngine` 启动时默认装载。它内部的本质就是**一堆字典**：

```369:381:vnpy/trader/engine.py
        self.ticks: dict[str, TickData] = {}
        self.orders: dict[str, OrderData] = {}
        self.trades: dict[str, TradeData] = {}
        self.positions: dict[str, PositionData] = {}
        self.accounts: dict[str, AccountData] = {}
        self.contracts: dict[str, ContractData] = {}
        self.quotes: dict[str, QuoteData] = {}

        self.active_orders: dict[str, OrderData] = {}
        self.active_quotes: dict[str, QuoteData] = {}

        self.offset_converters: dict[str, OffsetConverter] = {}
```

### 它存在的意义

事件的本质是**用完即弃**：Gateway 推出的每个 `Event` 被分发给 handler 后就不再保存。于是「我现在有哪些活动委托？」「600000 最新持仓多少？」这类**当前状态**无处可查。

OMS 就是解决这个问题的**中央状态缓存 / 全局快照仓库**：它订阅所有回报事件，把飞驰而过的一次性事件，沉淀成"当前最新状态"的可查询字典。价值三点：

1. **状态沉淀**：把易逝的事件流变成随时可查的最新快照。
2. **单一数据源**：避免每个策略/界面各自维护重复且易不一致的缓存，集中维护一份共享。
3. **派生加工数据**：用 `is_active()` 单独维护 `active_orders`（还挂在市场上的委托），为每个 gateway 维护 `OffsetConverter`（持仓换算）。

> 一句话：OMS 是把「异步事件流」翻译成「同步可查状态」的那层缓存。

---

## Q6. OMS 的 process_*_event() 结果如何通知策略端 / MainEngine？

**关键认知：OMS 根本不"通知"任何人。它只把数据存进自己的字典，仅此而已。**

看 `process_order_event`，全是"写入自己的字典"，没有任何一行往外发消息：

```399:414:vnpy/trader/engine.py
    def process_order_event(self, event: Event) -> None:
        """"""
        order: OrderData = event.data
        self.orders[order.vt_orderid] = order

        # If order is active, then update data in dict.
        if order.is_active():
            self.active_orders[order.vt_orderid] = order
        # Otherwise, pop inactive order from in dict
        elif order.vt_orderid in self.active_orders:
            self.active_orders.pop(order.vt_orderid)

        # Update to offset converter
        converter: OffsetConverter | None = self.offset_converters.get(order.gateway_name, None)
        if converter:
            converter.update_order(order)
```

策略和 MainEngine 拿到结果靠**两种完全不同的机制**：

### MainEngine：主动拉取（pull）——方法委托

MainEngine 初始化时把 OMS 的 `get_*` 方法**直接绑定成自己的方法**：

```144:163:vnpy/trader/engine.py
        oms_engine: OmsEngine = self.add_engine(OmsEngine)
        self.get_tick: Callable[[str], TickData | None] = oms_engine.get_tick
        self.get_order: Callable[[str], OrderData | None] = oms_engine.get_order
        self.get_trade: Callable[[str], TradeData | None] = oms_engine.get_trade
        self.get_position: Callable[[str], PositionData | None] = oms_engine.get_position
        self.get_account: Callable[[str], AccountData | None] = oms_engine.get_account
        ...
```

所以 `main_engine.get_position(...)` 实际就是读 `oms.positions` 字典。**谁需要状态，谁主动调 `get_*` 去查。**

### 策略端：被动推送（push）——但绕过 OMS，直接订阅 EventEngine

**策略不从 OMS 获取实时回报。策略和 OMS 是"平级的兄弟"，两者都直接向同一个 `EventEngine` 注册 handler。**

事件引擎的分发核心：一个事件来了，把**所有**注册了该 type 的 handler 用列表推导**依次全部调用**：

```66:78:vnpy/event/engine.py
    def _process(self, event: Event) -> None:
        """
        First distribute event to those handlers registered listening
        to this type.

        Then distribute event to those general handlers which listens
        to all types.
        """
        if event.type in self._handlers:
            [handler(event) for handler in self._handlers[event.type]]

        if self._general_handlers:
            [handler(event) for handler in self._general_handlers]
```

所以一个 `EVENT_ORDER` 到来时，OMS 的 `process_order_event` 和策略引擎的处理器是**并列被调用的**，OMS 不在策略上游。

```
                Gateway.on_order → EventEngine.put(EVENT_ORDER)
                                        │
                        EventEngine._process 分发给所有 handler
                                        │
         ┌──────────────────────────────┼──────────────────────────────┐
         ▼                               ▼                              ▼
 OmsEngine.process_order_event    策略引擎 handler              UI 委托监控表 handler
 （写进 orders 字典）              （推给策略 on_order）          （刷新界面）
       ↑ 三者平级，互不通知，各自独立处理同一个事件
```

### 两种模式对比

| | MainEngine / 上层查询 | 策略端实时回报 |
|---|---|---|
| 机制 | **拉（pull）** | **推（push）** |
| 数据来源 | 调 `get_*` 读 OMS 字典 | 直接订阅 EventEngine 事件 |
| 经过 OMS 吗？ | 是（查 OMS 缓存） | 否（和 OMS 平级，各收各的） |
| 典型场景 | "现在总持仓多少？" 主动查 | 委托状态一变立刻回调 `on_order` |

> 准确表述：`process_*_event` 的结果不主动流向任何人；它只把状态存好等 MainEngine 来查（拉）；策略的实时回报是它自己直接从 EventEngine 订阅来的（推），跟 OMS 没有上下游关系。

---

## Q7. 为什么 on_order / on_trade 需要一次推两个事件？

每个回报都推两个：一个通用的，一个带后缀的：

```109:115:vnpy/trader/gateway.py
    def on_order(self, order: OrderData) -> None:
        """
        Order event push.
        Order event of a specific vt_orderid is also pushed.
        """
        self.on_event(EVENT_ORDER, order)
        self.on_event(EVENT_ORDER + order.vt_orderid, order)
```

原因是这两个事件面向**两类完全不同的订阅者**：

### 通用事件 `EVENT_ORDER`（`"eOrder."`）——面向「要看全部」的人

订阅者是关心全市场所有委托的组件：**OMS**（缓存所有，注册在通用事件上，见 `engine.py:387`）、**UI 委托监控表**（显示所有）、**全局风控**。

### 精准事件 `EVENT_ORDER + vt_orderid`——面向「只关心自己那一笔」的策略

策略只注册 `EVENT_ORDER + 自己的vt_orderid`（成交/行情用 `vt_symbol` 后缀）。EventEngine 的 `_handlers` 字典就把事件**精准投递**给对应策略，别人收不到。

### 为什么这么设计？——性能与解耦

事件是**按 type 字符串路由**的（`_handlers[event.type]`）。假设跑了 100 个策略：

- **只有通用事件**：每来一条回报要推给全部 100 个策略，每个策略还得在 `on_order` 里写 `if 是我的单 else return`。100 条 × 100 策略 = 一万次调用，99% 无效——浪费 CPU + 重复过滤代码。
- **有精准事件**：策略 A 只注册 `EVENT_ORDER + A的vt_orderid`，只有 A 的回报会唤醒 A。无需过滤，无无效调用，策略间彻底解耦。

| 事件 | 频道粒度 | 典型订阅者 | 用途 |
|---|---|---|---|
| `EVENT_ORDER` | 全局广播 | OMS、UI 监控表、风控 | "我要所有委托" |
| `EVENT_ORDER + vt_orderid` | 精准点播 | 具体某个策略 | "我只要我自己那一笔" |

> 一句话：通用事件服务"全局收集者"，精准事件服务"个体订阅者"，靠 event type 字符串做路由，避免了全量广播 + 手动过滤的性能与耦合灾难。

---

> 相关源码（本仓库）：
> `vnpy/trader/object.py`、`vnpy/trader/constant.py`、`vnpy/trader/engine.py`、`vnpy/trader/gateway.py`、`vnpy/trader/converter.py`、`vnpy/trader/event.py`、`vnpy/event/engine.py`
