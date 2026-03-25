
从wireshark分析工具来看，感觉很多trading里面的工具都会不一样。
这让我想起去AWS做network engineer的感觉。虽然我在外面有很多年network经验，但是进去还是需要一段时间了解里面特别的环境，举例，我们在AWS里面通常都不能直接SSH到设备。这和外面就很不一样。

From what I’ve seen with Wireshark and other analysis tools, it feels like a lot of the tooling inside trading firms will be quite different from what I’m used to.
It actually reminds me of when I joined AWS as a network engineer. Even though I already had many years of networking experience outside, I still needed time to get used to their very special environment. For example, at AWS we usually couldn’t just SSH directly into devices, which is very different from how things work in most other places.

* design the capture strategy, build the parsers, and connect TCP‑level observations to kernel scheduling, NIC behaviour, and exchange microstructure.”


简单直观版理解 
“Exchange microstructure”（市场 / 交易所微结构）就是：一笔订单从你系统里发出去，到在交易所排队、撮合、成交、反馈回来的整个“内部机械结构和细节规则”。
它不关心“这个股票值不值这个价”（那是宏观/基本面/量化选股），而是关心：
* 订单是怎么在撮合机里排队的？
* 谁先谁后？谁排在你前面？
* 交易所什么时候丢你包、限流、拒单？
* 不同订单类型（limit, market, IOC, FOK, hidden, pegged…）的真实行为和坑是什么？
* 一开盘、一收盘、大行情时，微观上发生了什么变化？
你可以把它想象成： “不是研究股价走势，而是研究‘交易所匹配引擎 + 网络 + 协议’这一整套系统的机械细节，看看怎么在这个系统里排得更靠前、成交得更快、更稳定。”
微结构具体包括哪些东西？ 
典型会关心的维度大概有：
1. 订单簿结构（Limit Order Book）
    * 每一档 bid/ask 的挂单量分布、更新频率。
    * 价差（spread）、深度（depth）、冲击成本（price impact）。
    * 在什么情况下某档价格瞬间被“扫空”。
2. 排队与撮合规则
    * 是纯粹 price‑time priority（价优时间优）吗？
    * 有没有 size‑time、pro‑rata、hybrid 之类的队列规则？
    * 修改订单会不会丢掉排队位置？（某些交易所 amend 相当于 cancel+new）
    * hidden / iceberg / pegged 等特殊订单在队列中的真实排序规则。
3. 订单类型与行为差异
    * Limit、Market、IOC、FOK、Pegged、Post‑only…
    * 不同类型在高波动时的风险：滑点、被拒单、被拉穿盘口。
    * 哪些 order type 实际上会触发特殊 throttling 或风控。
4. 时间结构与事件驱动
    * 开盘集合竞价、收盘拍卖、盘中 auction / volatility halt。
    * 大经济数据、期货到期、rebalance、指数调整等事件对 order book 的即时影响。
    * 这些时刻下，队列长度、cancel/replace 速率、fill 速率的统计特征。
5. 技术层面的延迟与路径
    * 从你的机房到交易所：线路、switch、网卡、驱动、内核、应用，每一段的延迟与抖动。
    * 交易所侧网关/撮合机的处理路径，什么时候会排队、什么时候会丢包或限流。
    * 不同 colocation / POP / gateway 之间微妙的 latency 差异如何反映到 queue position 上。
6. 风控与限流机制（micro 级别）
    * per‑session / per‑IP / per‑port 的节流规则。
    * 撮合机或网关的保护逻辑（过载、价格保护、self‑trade prevention）。
    * 这些逻辑在 pcaps / feed / ack pattern 里会表现成什么“特征”。
为什么 HFT / 交易基础设施特别在乎 microstructure？ 
对高频或低延迟交易来说，同样的策略，如果你的微结构理解更深、系统更贴合交易所实际行为，就能：
* 更准确预估自己在队列中的位置（queue position）。
* 知道什么时间用什么 order type 最划算。
* 避免踩到“被交易所限流/保护/暗规则打脸”的坑。
* 在 open/close 这种“系统最极限”的场景下仍然保持稳定和快速。
换句话说： Microstructure 是让你从“会下单”进阶到“懂得跟交易所这台机器打配合”。
你现在看 IMC 这种 “Quantitative Microstructure Analyst” 的 JD，其实就是让你：
* 把网络 / Linux / 协议这块的硬功夫，
* 加上对交易所撮合规则、order book 行为、延迟统计的理解，
* 用数据（pcap、feed、logs）把这些微结构“量化”出来，
* 然后反馈到系统优化和策略执行里。
这就是 exchange microstructure 的核心：研究交易所在“微观时间尺度”和“微观队列细节”上的行为模式，并把它变成可以用来赚钱/降风险的工程和模型。


the detailed mechanics of how your orders travel from your system into the exchange, line up in the order book, get matched, turn into trades, and then come back as confirmations.
It focuses on questions like:
* How exactly are orders queued and matched inside the matching engine?
* Who gets priority in the queue, and why is someone ahead of you?
* When and why does the exchange throttle, reject, or drop your orders?
* How do different order types (limit, market, IOC, FOK, hidden, pegged, etc.) really behave in practice, including all the quirks?
* What happens at a micro level during opens, closes, auctions, and volatile events?

exchange microstructure means combining:
* low‑latency networking and Linux/kernel knowledge,
* detailed understanding of the exchange’s matching rules and order book behavior,
* and large‑scale analysis of pcaps, market data feeds, and logs,

