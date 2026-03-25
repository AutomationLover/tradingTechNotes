MoldUDP64 可以把它理解成：在 UDP 上面加了一层「会话 + 序列号 + 批量消息封装」的小协议，专门为行情/订单簿这种市场数据做**有序分发和丢包检测**，常见用法是：UDP + MoldUDP64 + ITCH 消息。

下面按几个维度来讲，方便你直接抄到笔记里。

## 1. MoldUDP64 主要解决什么问题？

UDP 本身的问题你很熟：

- 不保证顺序（乱序、丢包都可能发生）
- 不保证可靠（不会自动重传）
- 每个 UDP 报文大小有限，上层自己处理拆包/粘包
- 客户端断线重连时，没办法说「从第 N 条开始给我」

而交易所行情的硬需求是：

- 所有 market event 有严格的顺序，用来重建 order book
- 客户端要能检测「是不是少了一段消息」
- 客户端断线重连要能从某个序列号开始 replay
- 经常用多播 UDP，要求延迟低、开销小

所以 MoldUDP64 的核心目标可以一句话概括：

给每条（或每批）行情消息，打上一个 **Session + 64 位递增序列号**，然后打包进 UDP。这样：
- 客户端能检测 seq 是否连续，发现 gap 就知道丢包
- 客户端可以用 seq 范围向交易所请求 replay
- 方便断线重连：从某个 seq 继续往后拉

MoldUDP64 自己**不做重传，也不做拥塞控制**，这些由「replay 通道 / snapshot 服务 + 客户端逻辑」来完成。

---

## 2. MoldUDP64 报文结构（简化版）

一个标准的 MoldUDP64 UDP 报文，大致长这样（结构示意，非精确格式）：

Session Header:
  Session Name      固定长度字符串（比如 10 字节），标识这条行情流
  Sequence Number   64 位无符号整数，表示本报文中第一条消息的序号
  Message Count     16 位整数，表示本报文中有多少条消息

Messages（重复 Message Count 次）:
  Message Length    16 位整数，表示这一条消息长度
  Message Payload   实际消息内容（通常是 ITCH 消息）

可以抽象成：

[Session Name][Seq Number][Msg Count]
  [Msg Len][Msg #1 Payload]
  [Msg Len][Msg #2 Payload]
  ...
  [Msg Len][Msg #N Payload]

几点要点：

- Session Name 用来区分不同的 feed（比如不同行情通道）
- Sequence Number 是「这一批消息的起始序列号」，后面的消息按顺序递增
- 每条消息前有一个长度字段，解决了「UDP 里多条消息拆分」的问题
- UDP 层完全不知道这些含义，对它来说这只是 payload

---

## 3. 客户端正常接收时的工作流程

从客户端视角，典型流程是这样（伪逻辑）：

1. 初始化
   - 订阅某个 Session（比如某组多播地址）
   - 设置 expected_seq = 1 或交易所给定的起始值

2. 循环收 UDP 包并解析 MoldUDP64 头：
   - 读 Session Name，检查是不是自己订阅的那个
   - 读 Sequence Number（seq）和 Message Count（count）

3. 检查序列号：
   - 如果 seq == expected_seq：
       当前这个包刚好是自己期望的起点，正常处理
   - 如果 seq > expected_seq：
       中间缺了一段（expected_seq ~ seq-1），说明丢包或晚到
       记录 gap，触发 replay 逻辑
   - 如果 seq < expected_seq：
       可能是重传包或旧包，按策略处理（例如丢弃或只用来填 gap）

4. 按顺序读包内的消息：
   - 循环 count 次：
       读 Message Length
       读 Message Payload（ITCH 消息）
       交给行情处理逻辑（重建 order book 等）
       expected_seq += 1

用口语说就是：**expected_seq 是「我接下来想要的那条消息序号」，MoldUDP64 header 里的 seq 告诉你「这个 UDP 包从哪一条消息开始」**。

---

## 4. 丢包检测与 replay / snapshot 恢复

关键点是：MoldUDP64 只负责「让你发现丢包」，恢复是上层机制。

1. 丢包检测
   - 假设 expected_seq = 1001
   - 某个包的 seq = 1005，Message Count = 3
   - 说明本包消息覆盖 1005~1007，而 1001~1004 没出现
   - 这时就知道 1001~1004 这一段缺失

2. 重传请求（replay）
   - 客户端通过交易所提供的 replay 通道发送请求：
     例如：请从 seq=1001 重传到 1004
   - replay 流可能是 TCP，也可能是另一条 UDP 流

3. 从快照恢复
   - 如果缺失太多，或者你中途才开始连上 feed：
     - 先从 snapshot 服务拉一个「全量 order book snapshot」，上4. 从快照恢复（补充完整）
   - 如果缺失的序列号太多，或者你是「中途加入」行情流：
     - 先从交易所的 snapshot 服务获取一个「订单簿/市场状态快照」
       这个快照通常带有一个对应的最新 seq，例如 snapshot_seq
     - 然后从 snapshot_seq+1 开始，通过 MoldUDP64 feed 接收增量消息
     - 把 snapshot + 之后的增量按序 replay，一步步把本地 order book 补齐到最新

整体恢复思路是：
  小范围丢包 → 用 replay 通道补齐缺失 seq
  大范围缺失 / 新启动 → 用 snapshot + 之后的增量 feed 重建状态

---

## 5. 和 QUIC/TCP 的类比（帮助你脑中定位）

为了跟你之前看的 QUIC / HTTP3 对上号，可以这么类比（只是帮助记忆，不是严格一一对应）：

- TCP / QUIC：
  - 在传输层做「自动重传 + 拥塞控制」，上层看到的是一条可靠的 byte stream
  - 丢包由 sender 根据 ACK 自动判断并重传

- MoldUDP64：
  - 不在传输层做重传，只在 UDP 上加「Session + 64 位 seq + 批量消息」
  - 用来：
    - 让 client 发现自己**少收了哪一段消息**
    - 让 client 知道「从哪个 seq 以后需要 replay/snapshot」
  - 真正的重传/恢复是在**应用级/会话级**完成（replay 通道 + snapshot 服务）

一句话区别：
  QUIC：保证「数据传输」可靠
  MoldUDP64：保证你能恢复「市场事件序列」

---

## 6. 一个更完整的伪代码示例（纯文本，无代码块语法）

下面是一个极简的 MoldUDP64 接收伪代码（逻辑思路），方便你直接放到笔记里：

main:
  expected_seq = START_SEQ
  while true:
    pkt = recv_from_udp()

    session     = read_session_name(pkt)
    seq         = read_u64(pkt)        # 本包第一条消息的序列号
    msg_count   = read_u16(pkt)

    if session != TARGET_SESSION:
      continue  # 过滤掉别的 feed

    if seq > expected_seq:
      # 发现 gap: [expected_seq, seq-1] 这一段缺失
      record_gap(expected_seq, seq - 1)
      trigger_replay(expected_seq, seq - 1)

    # 如果 seq < expected_seq，要看你的策略：
    #   - 如果这是专门的 replay 流：可能用来填补之前记录的 gap
    #   - 如果是普通 feed：说明是旧包，通常丢弃

    current_seq = seq
    for i from 1 to msg_count:
      msg_len = read_u16(pkt)
      msg     = read_bytes(pkt, msg_len)

      # 处理 ITCH 消息（更新本地 order book / trade log 等）
      handle_itch_message(msg, current_seq)

      current_seq  += 1
      if current_seq > expected_seq:
        expected_seq = current_seq

---

## 7. 总结一句话定义（方便你以后快速回忆）

MoldUDP64 =

「在 UDP 之上，为市场数据流增加 Session 标识 + 64 位单调递增的消息序列号 + 批量消息封装的轻量协议，用于：
  1）让客户端检测丢包（通过 seq gap），
  2）驱动 replay/snapshot 恢复，
  3）保证 ITCH 等行情消息可以按正确的时间顺序重放。」

你可以把它当作「专门为行情 feed 定制的、基于 UDP 的有序消息分发壳子」，而不是一个像 TCP/QUIC 那样的通用传输协议。

