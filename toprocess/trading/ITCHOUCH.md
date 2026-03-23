# ASX ITCH / OUCH 文档里的关键技术点总结

https://www.asxonline.com/content/dam/asxonline/public/documents/asx-trade-refresh-manuals/asx-trade-itch-ouch-application-conformance-process.pdf

下面只抓“技术考点”，尽量可直接在面试里用来展示对交易接入与市场数据的理解。

## 1. Conformance / 交易接入流程思路

- 所有直连 ASX 的交易 / 市场数据应用，**必须通过 Conformance 测试**才能上 Production。
- 何时必须重做 conformance：
  - 影响连接或报文的功能改动；
  - 加新 ASX-facing 功能；
  - 换 OS 重编译；
  - 交易所生产环境升级（标记为 mandatory）；
  - 软件长期未连网后重连；
  - 生产发现异常 / 扰乱市场行为；
  - 交易所特别要求。
- 开发、测试环境：
  - 使用 ASX 提供的 FTE/ETE 测试平台，**模拟生产环境**，做连接与报文验证。

面试可讲成：交易所通过强制的 conformance 流程，确保买方 / 卖方 / ISV 的交易网关在连接管理、报文格式、风险行为上都满足规则，减少“erroneous application behaviour”。

---

## 2. OUCH：Order Entry 协议核心点

### 2.1 连接与会话管理

- OUCH 是 ASX 的低延迟订单入口协议，关键流程：
  - Logon：发送 “L” Packet 建立会话 + 后续保持心跳；
  - Logout：发送 “O” Packet **优雅退出**；
  - 要求会话稳定，不能频繁掉线。
- 账户禁用场景（Account Disable）：
  - ASX 主动断开并改密码；
  - 客户端在收到 “Account disconnected” 后可以尝试重新登录，但：
    - 无效密码的尝试次数 < 3 次；
    - 重试间隔 > 5 秒；
    - 因为 3 次失败账号会锁定。
- OUCH Recovery：
  - 断线前有订单已被接受；
  - 非正常断线（进程崩溃或被交易所踢掉）；
  - 重连时带上 Requested Sequence Number = 1，“从头恢复”；
  - 客户端必须正确处理**缺失消息重放**和**重复消息丢弃**。

这些是典型面试点：**会话管理、心跳、断线重连、幂等与重复消息处理**。

---

### 2.2 Equity Order Management 基本能力

- 必须支持的基本操作：
  - Enter Order（Y：普通 Limit 单，Day 有效）；
  - Replace Order（用原 OUCH Order Token 修改价格或数量，收到 “Order Replaced”）；
  - Cancel Order（通过 Token 取消，收到 “Order Cancelled”）。
- 订单状态管理：
  - 配合 ASX Assisted Testing 中的 Order Status Verification：
    - 正确接收部分成交 / 全部成交消息；
    - 更新本地订单簿（剩余数量、状态）；
    - 处理撤单确认。

面试可用来讲：客户端如何维护**本地订单簿**，对应外部事件（Accepted / Executed / Canceled）来做状态机。

---

## 3. OUCH：Centre Point / 高级订单类型

文档中列出了 ASX 特有的一系列 “Centre Point” 暗池 / 智能路由单类型，关键在于你能说清“类型 + 行为”：

- Mid-point only（“N”）  
  在买卖盘中点价格执行的暗池订单。
- Dark Limit（“D”）  
  有显式限价，但在暗池中执行，不显示到公开订单簿。
- Sweep / Limit Sweep（“S” / “T”）  
  带“扫单”特性，先在 Centre Point 执行，然后可能向 lit book 扫流动性。
- Dual-posted Sweep（“P”）  
  同时在不同场所挂单并带 sweep 行为。
- Block w/ MAQ / Dark Limit w/ MAQ / Any Price Block w/ MAQ（“B” / “F” / “E”）  
  带 **Minimum Acceptable Quantity (MAQ)** 的大额块交易暗池订单。
- Any Price Block（“C”）  
  价格可在一定范围内灵活匹配的大宗订单。

可以总结成：ASX 提供一套围绕暗池、块交易、扫单、双重挂单、最小成交量控制的**复杂订单路由与执行逻辑**。面试时可以用这部分展示对“暗池 + MAQ + sweep 逻辑”的认知。

---

## 4. OUCH：辅助功能与风险控制点

### 4.1 Unintentional Crossing Protection (UCP)

- Enter Order 时携带 UCP key；
- 交易所确认后返回同样的 UCP key；
- 用于防止**自成交（self-cross）/ 账户之间非预期对敲**。

可以抽象讲成：在高频交易场景中，用 UCP 作为业务层的“自成交保护”机制，避免同一策略在不同会话上互相打单。

---

### 4.2 Time-in-Force: FaK / FoK

- FaK (Fill and Kill, TIF=3)：
  - 能成交的部分立即成交，剩余部分取消；
- FoK (Fill or Kill, TIF=4)：
  - 要么一次全部成交，否则全部取消。

这是常见交易面试考点：解释 **Day / IOC / FaK / FoK** 等 TIF 的区别。这里文档明确把 FaK / FoK 定义为 OUCH “Supported Functionality”。

---

### 4.3 Short Selling 参数

- Enter Order 时可带 **short sell quantity > 0**；
- Side 可以为 “T” 或 “C”（ASX 定义的方向编码）；
- 交易所会以“适当的 short sell quantity”将订单放入市场。

可以延伸讲：卖空在实盘中通常还会结合 locate、可借券、flagging 等监管需求。

---

### 4.4 Cancel by Order ID

- 与通过 OUCH Order Token 取消不同，这里是 **使用 Firm Order ID** 取消；
- 流程：
  - 先下一个订单，Order Accepted 中返回一个 Order ID；
  - 再以 Order ID 作为参数发送 Cancel Order；
  - 收到取消确认，订单从市场与本地订单簿移除。

这可以说明你理解“网关内部生成的 token”与“交易所 firm order id”两层标识的区别。

---

## 5. ITCH：市场数据与恢复机制

ITCH 是 ASX 的市场数据协议，文档覆盖了两个核心恢复机制：

### 5.1 Rewinder Gap Request（UDP 重传）

- 应用检测到序列号出现 gap 时：
  - 向 Rewinder server 发送 Request Packet；
  - 指定“起始 sequence number + 请求重传的消息数量”；
  - Rewinder 通过 UDP 返回对应的历史消息。
- UDP payload 有最大长度限制，所以可能需要多次请求才能补全。

面试可以讲为：**市场数据流通常是 UDP 多播**，因此客户端必须自己做序列号检测 + gap 补齐，交易所提供 rewinder 服务辅助重建完整消息流。

---

### 5.2 Glimpse Snapshot（TCP 快照）

- 客户端通过 TCP 连接到 Glimpse server；
- 接收一个当前市场数据快照 + “end of snapshot” 消息；
- 快照与实时 ITCH 增量消息组合，用于：
  - 冷启动；
  - 毁损状态恢复；
  - 快速重建本地 order book。

这一块能够体现你对“**快照 + 增量**”架构的理解，以及为何用 TCP 做 snapshot、UDP 做实时 multicast。

---

## 6. ITCH：订单 / 交易与 Session State 校验

在 Assisted Testing 中，ASX 会：

- 主动在某合约上挂单、成交，让你：
  - 确认价格、数量；
  - 计算剩余挂单量；
  - 统计合约总成交量；
- 修改某合约的 **Trading Session State**，例如：
  - Pre-open / Open / Halt / Close 等；
  - 客户端需要正确理解并展示新的交易阶段。

技术要点在于：客户端必须从 ITCH 消息中解析 **session state / instrument status**，影响前端是否允许下单、是否展示为暂停、竞价等。

---

## 7. 可以在面试里如何使用这些点

你可以把这篇文档抽象成几条“框架级认知”，在 trading / market data / low-latency 相关面试中用来输出：

1. 交易所级别的 **Conformance 与风险控制**：  
   - 必须的 logon/logoff、心跳、账号锁定规则、非正常断线恢复、重复消息处理。
2. OUCH 订单协议：  
   - 基本 CRUD（Enter / Replace / Cancel）；  
   - 高级订单类型（暗池、块交易、MAQ、sweep、dual-posted）；  
   - 时间有效性（FaK / FoK）、Short Sell、UCP 自成交保护。
3. ITCH 市场数据体系：  
   - UDP 多播 + 序列号 + Rewinder 补包；  
   - TCP Glimpse 快照 + 增量消息重建本地 order book；  
   - Session State / Instrument Status 驱动前端与风控行为。

这些都是“非纯理论、直接来自真实交易所规范”的点，在面试里讲出来会显得非常实战。

