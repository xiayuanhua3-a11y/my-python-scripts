# BroadcastWorkThread 深度解析

**来源**: 代码审查 + Round 6 压测 OOM 根因分析  
**文件**: `Gaming-MQ-DB-Service/.../between/game/thread/BroadcastWorkThread.java`

---

## 一、整体架构

`BroadcastWorkThread` 是一个**单线程事件循环**（继承 `Thread`），命名为 `InBetween-Broadcast-Thread`，维护 4 条独立队列，每 10ms 轮询一次：

```
run() loop（每 10ms 唤醒一次）
 │
 ├── 1. broadcastBetTaskBuffer    ← 下注广播（纯同步，在本线程执行，O(N×B)）
 ├── 2. broadcastResultTaskBuffer ← 结算广播（同步遍历，再塞入 noticeTasksQueue）
 ├── 3. broadcastTaskQueue        ← 全体广播（Game Start 等，异步分发到线程池）
 └── 4. noticeTasksQueue          ← 个性化通知（异步分发到线程池）
```

---

## 二、`broadcastBetTaskBuffer` — 下注广播（O(N×B) 同步阻塞）

### 逻辑流程

每次有玩家下注，调用方执行：
```java
broadcastPlayerBetTask(betBean);   // 把 BetBean 塞入 broadcastBetTaskBuffer
```

Broadcast 线程每 10ms 检测到队列非空，执行：

```java
var tempQueue = new ConcurrentLinkedQueue<>(broadcastBetTaskBuffer);  // ① 复制 bet 队列
broadcastBetTaskBuffer.clear();
var players = rooms.getPlayers();
players.forEach((uid, player) -> {                                     // ② 遍历所有玩家（10k）
    var t = new ConcurrentLinkedQueue<>(tempQueue);                    // ③ 每人再复制一次！
    var list = new ArrayList<>();
    t.stream().filter(betBean -> !betBean.getUid().equals(uid))        // ④ 过滤自己
              .forEach(betBean -> { /* build protobuf */ list.add(...); });
    if (!list.isEmpty()) {
        MessageUtil.sendMsg(player.getSession(), ...);                 // ⑤ 同步发送
    }
});
```

### 代价分析

| 步骤 | 代价 |
|------|------|
| ① 复制 bet 队列 | O(B)，B = 本次 10ms 窗口内积累的 bet 数 |
| ② 遍历玩家 | O(N)，N = 在线玩家数 |
| ③ **每个玩家再深拷贝队列** | O(N × B) 额外内存分配 |
| ④ stream + filter | O(N × B) CPU 消耗 |
| ⑤ sendMsg（**同步**） | N 次网络写操作，全部阻塞 Broadcast 单线程 |

**总复杂度：O(N × B)**

### 什么时候接近 1 亿操作？

| 场景 | B（每 10ms 积累的 bet 数） | 总操作数 |
|------|--------------------------|----------|
| 平时低流量 | 1–5 | 10,000–50,000 |
| 压测下注高峰 | 50–200 | 500k–2M |
| 10k 人同时疯狂下注 | 接近 10,000 | **接近 1 亿** |

> 严格来说不是固定的 N² = 10k×10k，而是 N×B。但在高并发下注时 B 随队列积压不断增大，会逐渐逼近 N²。

### 关键缺陷

这条路径**没有提交到线程池**，全部同步阻塞在 `InBetween-Broadcast-Thread` 上。  
即使 B=1，10,000 次同步 sendMsg 也可能需要几百毫秒，导致下一个 10ms 窗口的 bet 继续积压 → B 越来越大 → 雪球效应。

---

## 三、`broadcastTaskQueue` → `doAction(BroadcastTask)` — OOM 爆炸点（Round 6 直接根因）

### 逻辑流程

Game Start 等全体广播触发时，调用方执行：
```java
broadcastTask(msgId, bytes);   // 塞入 broadcastTaskQueue
```

Broadcast 线程处理：
```java
batch.parallelStream().forEach(task -> {
    getExecutor().execute(() -> doAction(task));   // 提交给线程池执行
});
```

`doAction(BroadcastTask)` 内部：
```java
private void doAction(BroadcastTask task) {
    var players = rooms.getPlayers();
    players.forEach((uid, player) -> {
        getExecutor().execute(() -> {              // ← 嵌套！每个玩家再提交一个任务
            if (MessageUtil.checkSessionNetStatus(player.getSession())) {
                MessageUtil.sendMsg(player.getSession(), task.msgId, task.bytes);
            } else {
                rooms.onPlayerExitRoom(uid);
            }
        });
    });
}
```

### 双层嵌套导致的任务爆炸链

```
Game Start → 1 个 BroadcastTask 进入 broadcastTaskQueue
  → doAction() 被 executor 执行（占用 1 个 Broadcast-Thread）
    → players.forEach 遍历 13,617 个玩家
      → 每个玩家调用 getExecutor().execute(...)
        → 前 10,000 个进入 PLAYER_BROADCAST_ASYNC_POOL 的 queue（容量 10,000）
        → 后 3,617 个触发 CallerRunsPolicy
          → 阻塞 InBetween-Broadcast-Thread 本身！其他队列被卡死
        → 最多 40 个 Broadcast-Thread 同时运行
          → 每个线程 sendMsg 分配 1 个 4MB Netty ByteBuf（Direct Memory）
          → 40 × 4MB = 160MB 瞬间分配
```

### PLAYER_BROADCAST_ASYNC_POOL 配置

```java
// MBSConfigure.java
@Bean(MBSConstant.PLAYER_BROADCAST_ASYNC_POOL)
public ThreadPoolTaskExecutor asyncThreadPoolBroadcastExecutor(){
    // core=10, max=40, queue=10000, CallerRunsPolicy
    return getThreadPoolTaskExecutor("Broadcast-Thread-", CpuCores, CpuCores * 4, 10000, 30, 60);
}
```

---

## 四、为什么 10k 玩家时 Direct Memory 直接被榨干

### 内存压力叠加图

```
组件                                        Direct Memory 占用
───────────────────────────────────────────────────────────────
10,000 条 WebSocket 持久连接
  Netty Socket read/write 缓冲区
  ≈ 100KB/连接 × 10,000                  → ~1,000 MB（占 98.3%）

Game Start 广播触发：
  40 个线程同时执行 sendMsg
  每次 sendMsg 分配 1 个 BinaryWebSocketFrame ByteBuf
  ≈ 4MB/次 × 40 线程并发               → +160 MB

1,000 + 160 = 1,160 MB > 1,024 MB（MaxDirectMemorySize=1g）
                                        → java.lang.OutOfMemoryError
                                        → Broadcast-Thread-6, -41, -47... 全部崩溃
```

### Round 6 精确 OOM 日志（16:07:56）

```
Exception in thread "Broadcast-Thread-6"
java.lang.OutOfMemoryError: Cannot reserve 4194304 bytes of direct buffer memory
  (allocated: 1070249628, limit: 1073741824)
```

| 项目 | 数值 |
|------|------|
| `limit` | 1,073,741,824 bytes = **1 GiB** |
| `allocated` | 1,070,249,628 bytes = **~1023 MB（99.9% 满）** |
| 尝试分配 | 4,194,304 bytes = **4 MB**（一个 ByteBuf） |
| 结果 | OOM → 线程死亡 |

> **关键**：Direct Memory 不是 JVM Heap，GC 不直接回收。Netty ByteBuf 依赖引用计数（`release()`）手动归还，WS 连接存活期间缓冲区不释放。10k 连接把 1GB 配额几乎耗尽，广播操作没有余量。

---

## 五、`noticeTasksQueue` — 结算时的大批量积压

`broadcastResultTaskBuffer` 处理结算时，把每个玩家的个人通知塞入 `noticeTasksQueue`：

```java
// 一次性往队列塞 10k 个任务
players.forEach((uid, player) -> {
    noticeTasksQueue.offer(notice);
});
```

下一个 loop tick，一次性提交 10k 个任务到线程池：

```java
batch.parallelStream().forEach(task -> {
    getExecutor().execute(() -> doAction(task));   // 10k 个任务瞬间提交
});
```

线程池 queue 容量 = 10,000，恰好打满 → 后续任务触发 CallerRunsPolicy → Broadcast 线程被阻塞执行任务 → 其他队列无法处理。

---

## 六、两条路径问题性质对比

| | `broadcastBetTaskBuffer` | `broadcastTaskQueue` → `doAction` |
|---|---|---|
| 执行方式 | **同步**，在 Broadcast 单线程 | **异步**，提交到线程池 |
| 复杂度 | O(N × B)，N=玩家数，B=bet数 | O(N) 任务提交，但嵌套 executor |
| 问题类型 | CPU/延迟，单线程阻塞 | **Direct Memory OOM，线程死亡** |
| 10k 时症状 | 下注广播越来越慢 | **Broadcast 线程全部崩溃，服务雪崩** |
| 严重程度 | P1 性能 | **P0 可用性** |

---

## 七、修复方向

### P0 — 立即（无需改代码）：扩大 MaxDirectMemorySize

```yaml
# server.yaml JAVA_TOOL_OPTIONS
-XX:MaxDirectMemorySize=3g
```

理由：10k WS 连接需要 ~1GB，广播 ByteBuf 需要额外 1-2GB，3g 是安全下限。

### P1 — 高优：`doAction(BroadcastTask)` 改为 Batch 提交

```java
private void doAction(BroadcastTask task) {
    var playerList = new ArrayList<>(rooms.getPlayers().values());
    int batchSize = 200;
    for (int i = 0; i < playerList.size(); i += batchSize) {
        List<Player> batch = playerList.subList(i, Math.min(i + batchSize, playerList.size()));
        getExecutor().execute(() -> {
            for (Player player : batch) {
                Channel ch = player.getSession().getChannel();
                if (ch.isActive() && ch.isWritable()) {
                    MessageUtil.sendMsg(player.getSession(), task.getMsgId(), task.getBytes());
                } else if (!ch.isActive()) {
                    rooms.onPlayerExitRoom(player.getUid());
                }
            }
        });
    }
}
```

效果：13,617 个 executor 任务 → **68 个**，峰值并发 Direct Memory 分配从 40×4MB 变为可控。

### P1 — 高优：`broadcastBetTaskBuffer` 改为异步 + 去掉每人复制队列

将同步的 `sendMsg` 改为提交到线程池，并只复制一次 protobuf bytes 供所有玩家复用，而不是每个玩家都重新构建。

### P2 — 中优：`isWritable()` 背压检查

在所有 sendMsg 前加：
```java
if (ch.isActive() && ch.isWritable()) { ... }
```

防止写缓冲区满时继续堆积 ByteBuf。

### P3 — 中优：Session 清理

确保 `channelInactive()` 时强制从 `players` Map 移除断线玩家，避免旧连接的 Direct Memory 缓冲区长期不释放。
