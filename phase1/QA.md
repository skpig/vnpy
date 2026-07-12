# 阶段 1 · 自测批改记录（QA）

> 对阶段 1 自测清单的作答批改与重点讲解。配合 `01_数据语言精讲.md` 阅读。

---

## 批改总览

| 题 | 结果 | 备注 |
|---|---|---|
| 1 | 🔴 需讲解 | 前缀 = 命名空间，防撞号（交易所 / 券商） |
| 2 | ✅ 正确 | 补充：数据流方向 |
| 3 | 🟡 方向对 | 需能说清 orderid/status/traded 各自含义 |
| 4 | ✅ 正确 | — |
| 5 | 🔴 需讲解 | 期货同一合约多空双持仓，direction 防撞号 |
| 6 | ✅ 基本对 | 补充：派生值不该外部传入，防不一致 |
| 7 | ✅ 方向对 | 补全 buy/sell/short/cover 四象限 |

> 7 题中 5 题方向正确。卡住的 1、5 恰好是同一个知识点：**复合 ID 的"防撞号 / 命名空间"设计**。

---

## 第 1 题：为什么 `vt_` 字段要带前缀？

**一句话**：因为要在一个"全局大字典"里当唯一 key，前缀是为了防止"撞号"。**前缀 = 命名空间**（就像"腾讯.001"和"阿里.001"是不同的人）。

OMS 把所有东西存进字典（如 `self.orders: dict[str, OrderData]`），key 是 `vt_orderid`。字典 key 必须唯一，否则后者覆盖前者。前缀分两类：

### 类 A：`vt_symbol` 加交易所后缀 —— 防不同交易所代码撞号

```python
vt_symbol = f"{symbol}.{exchange.value}"   # 600000.SSE
```

不同交易所可能有相同代码：`000001` 在深交所是"平安银行"，在上交所是"上证指数"。
- `000001.SZSE` → 平安银行
- `000001.SSE` → 上证指数

### 类 B：`vt_orderid` 等加 gateway_name 前缀 —— 防不同券商编号撞号

```python
vt_orderid = f"{gateway_name}.{orderid}"   # CTP.123
```

`orderid` 由券商各自分配。同时连 CTP 和 IB 时，两家都可能给 `orderid="123"`：
- 无前缀：两张不同委托都叫 `"123"` → 存字典时互相覆盖。
- 有前缀：`CTP.123` vs `IB.123` → 各自独立。

| 字段 | 前缀 | 防的是什么撞号 |
|---|---|---|
| `vt_symbol` | 交易所 | 不同交易所的相同代码 |
| `vt_orderid` / `vt_tradeid` / `vt_accountid` | gateway_name | 不同券商的相同编号 |

---

## 第 2 题（✅）：数据对象 vs 请求对象的区别

作答："只有数据对象才带 gateway" —— **正确**，这是核心区分点。补充完整：

- **是否带 `gateway_name`**：数据对象继承 `BaseData` 都带（记录来源）；请求对象不带（发给谁在 `send_order(req, gateway_name)` 时指定）。✅
- **数据流方向**：数据对象 = Gateway → 你（回报"事实"）；请求对象 = 你 → Gateway（发出"意图"）。

---

## 第 3 题（🟡）：OrderData 有而 OrderRequest 没有的三字段

作答："请求还没真正传给券商，没发生实质性交易" —— **方向对**（"还没进市场所以没有这些属性"），但要能说清每个字段：

三个字段 = `orderid` / `status` / `traded`，都是**"进入市场后才产生的属性"**：

| 字段 | 含义 | 为什么请求没有 |
|---|---|---|
| `orderid` | 委托的身份证号 | 券商收到后才分配 |
| `status` | 委托的状态 | 还没进市场，无状态 |
| `traded` | 已成交量 | 还没成交 |

经 Gateway 的 `create_order_data()`（分配 orderid、设 status=SUBMITTING）后，意图才变成有身份、有状态的正式委托。

---

## 第 4 题（✅）：volume vs last_volume

作答："一个是过去累计，一个是最近成交" —— **完全正确**。

精确化：`volume` = 今日开盘到当前的累计成交量（单调递增）；`last_volume` = 最新这一笔的成交量。算"这个 tick 内新增量"要用 `本tick.volume − 上tick.volume`。

---

## 第 5 题：为什么 `PositionData.direction` 拼进 `vt_positionid`？

**一句话**：期货里同一合约可"同时"持有多头和空头两笔独立持仓，必须用 `direction` 把它们区分成两个 key。

### 关键：期货可双向持仓

股票不存在"负持仓"——要么持有，要么没有。
但期货可双向：对 `rb2405`，你可能昨天开 5 手**多单**、今天又开 3 手**空单**，同一合约上**同时**有两笔方向相反的持仓，盈亏方向相反，要分开管理。

### 不带 direction 会怎样

```python
vt_positionid = f"{gateway_name}.{vt_symbol}.{direction.value}"
```

- 不带：两笔持仓 key 都是 `GW.rb2405.SHFE` → 存 OMS 字典时后者覆盖前者 → 丢一笔。
- 带：多头 `GW.rb2405.SHFE.多`、空头 `GW.rb2405.SHFE.空` → 各存一份，互不干扰。

**这和第 1 题是同一个道理**——"防撞号 / 命名空间"，只不过区分的是同一合约的多头腿 vs 空头腿。股票只用 `.多`（不能持负股）。

---

## 第 6 题（✅）：available 为什么在 `__post_init__` 里算

作答："避免传参错误导致非法" —— **方向对**。补充本质：

`available = balance − frozen` 是**永远由其他字段推导的派生值**。铁律：**能算出来的，就不该让外部传入**：
1. **消除不一致**：若允许同时传 balance/frozen/available，可能传出自相矛盾的值（available=999 但 balance−frozen=100）。自动计算保证永远一致。
2. **减少调用负担**：只需提供两个源头，派生值框架代劳。

同类用法：`LogData.time = Datetime.now()`、所有 `vt_*` ID 拼接——都是"根据已有字段自动生成新字段"。

---

## 第 7 题（✅）：Direction × Offset 四象限

作答："看多/看空 × 开仓/平仓" —— **两个维度抓对了**。补全四象限与下单函数：

| | `Offset.OPEN`（开仓） | `Offset.CLOSE`（平仓） |
|---|---|---|
| `Direction.LONG`（买入） | 买入开仓 = 做多 → **`buy`** | 买入平仓 = 平空头 → **`cover`** |
| `Direction.SHORT`（卖出） | 卖出开仓 = 做空 → **`short`** | 卖出平仓 = 平多头 → **`sell`** |

**易错点**：
- `buy`(多开) 和 `cover`(空平) 都是 **Direction.LONG（买入）**，区别只在 offset；
- `short`(空开) 和 `sell`(多平) 都是 **Direction.SHORT（卖出）**，区别只在 offset。

即"买入"既可能开多仓也可能平空仓——由 offset 决定。这就是 Direction 和 Offset 是**两个正交维度**的含义。

---

> 相关源码：`vnpy/trader/object.py`、`vnpy/trader/constant.py`。
