# InBetween QAT — 第六輪壓測報告

**日期**: 2026-04-03  
**時間**: 16:06:33 – 16:18:18  
**用戶範圍**: 639400200001 – 639400210000（**10,000 人**，回到 10k）  
**本輪特點**: 開發應用了廣播並行化優化後的首次 10k 壓測

---

## 一、時序還原

| 時間 | Server 日誌量/min | APISIX 日誌量/min | 事件 |
|------|-----------------|-----------------|------|
| 16:05 | ~40 | ~160 | 正常待機 |
| 16:06 | 134,842 | 8,428 | 壓測開始，10k 用戶湧入 |
| 16:07 | 165,947 | 1,879 | 峰值，Game Start 廣播觸發 |
| **16:07:56** | — | — | **💥 Direct Memory OOM 爆發，Broadcast 線程池崩潰** |
| 16:08 | **4,179** | 57 | Server 廣播能力完全喪失 |
| 16:10 | 2,148 | 10,023 | 舊 WS 連線陸續斷開（存活 ~230s 後關閉） |
| 16:11 | 618 | **169,495** | APISIX 開始返回 503（後端 unhealthy） |
| 16:12–16:16 | ~600 | **~235,000** | 341,629 個 503 — 服務完全雪崩 |
| 16:17+ | 恢復低量 | 回到正常 | 壓測結束 |

---

## 二、壓測結果摘要

### 嚴重錯誤

| 類型 | 數量 | 說明 |
|------|------|------|
| WS 503 | 341,629 | Server 崩潰後 APISIX 返回 503 |
| Connect WS 503 | 341,629 | 同上 |
| Launch Game 503 | 334,344 | 同上 |
| Heart timeout | 122,123 | Server 不響應心跳 |
| Exit Room timeout | 30,073 | Server 不響應 |
| Logoff timeout | 30,073 | Server 不響應 |
| Personal History timeout | 29,204 | Server 不響應 |
| Transaction History timeout | 29,165 | Server 不響應 |
| In Between History timeout | 29,160 | Server 不響應 |
| WS socket hang up | 882 | 連接被強制關閉 |
| WS 502 | 2 | 極少量 |

### 性能超時

| 事項 | 延遲 | 對比 Round 5 |
|------|------|------------|
| Game Start | **53,787ms / 54,912ms** | Round 5: 17s → **惡化 3.2x** |
| Chat | 最高 14,864ms | Round 5: 4s 週期性 |
| Gift | 最高 15,207ms | — |
| In Between Start Betting | 最高 27,336ms | — |
| Launch Game 504 | 3,087 筆 | Server 崩潰期間 |

### CPU / Memory

- CPU: 16:06 開始攀升至 10 cores，**壓測結束後不釋放**（持續滿載）
- Memory: ~2.3 GiB → ~6.5 GiB，**壓測結束後不釋放**

---

## 三、根本原因分析

### 根因 #1：Direct Memory OOM（最高優先級）

**SLS 精確日誌（16:07:56）**：

```
Exception in thread "Broadcast-Thread-6"
Exception in thread "Broadcast-Thread-41"
java.lang.OutOfMemoryError: Cannot reserve 4194304 bytes of direct buffer memory
  (allocated: 1070249628, limit: 1073741824)

Exception in thread "Broadcast-Thread-47"
java.lang.OutOfMemoryError: Cannot reserve 4194304 bytes of direct buffer memory
  (allocated: 1070248593, limit: 1073741824)
...（持續至 Broadcast-Thread-120+）
```

**數據解讀**：

| 項目 | 數值 |
|------|------|
| `limit` | 1,073,741,824 bytes = **1 GiB** (`-XX:MaxDirectMemorySize=1g`) |
| `allocated` | 1,070,249,628 bytes = **~1023 MB（99.9% 滿）** |
| 嘗試分配 | 4,194,304 bytes = **4 MB**（一個 BinaryWebSocketFrame ByteBuf）|
| 結果 | OOM → 線程死亡 |

**直接記憶體消耗來源**：
- Netty 10k WS 連接的 Socket 緩衝區 ≈ **~1020 MB**（幾乎耗盡 1GB 限制）
- Broadcast 線程池同時分配 4MB × 40 線程 = **160 MB** → 完全超限

---

### 根因 #2：開發優化的雙層嵌套並行設計缺陷

**當前 `doAction(BroadcastTask)` 代碼**：

```java
// BroadcastWorkThread.java — 現行實現
private void doAction(BroadcastTask task) {
    var players = rooms.getPlayers();
    if(!players.isEmpty()){
        players.forEach((uid, player)->{
            // 每個玩家提交一個獨立任務 → 一次 Game Start 提交 13,617 個任務
            getExecutor().execute(() -> {
                if (MessageUtil.checkSessionNetStatus(player.getSession())) {
                    MessageUtil.sendMsg(player.getSession(), task.msgId, task.bytes);
                } else {
                    rooms.onPlayerExitRoom(uid);
                }
            });
        });
    }
}
```

**`PLAYER_BROADCAST_ASYNC_POOL` 配置**：

```java
// MBSConfigure.java
@Bean(MBSConstant.PLAYER_BROADCAST_ASYNC_POOL)
public ThreadPoolTaskExecutor asyncThreadPoolBroadcastExecutor(){
    // core=10, max=40, queue=10000, CallerRunsPolicy
    return getThreadPoolTaskExecutor("Broadcast-Thread-", CpuCores, CpuCores * 4, 10000, 30, 60);
}
```

**任務爆炸鏈路**：

```
Game Start 觸發 → broadcastTaskQueue 有 1 個 BroadcastTask
  → doAction(BroadcastTask) 被執行
    → players.forEach 遍歷 13,617 個玩家
      → 每個玩家 getExecutor().execute(...) ← 提交 13,617 個任務
        → 前 10,000 個進 queue
        → 後 3,617 個觸發 CallerRunsPolicy（阻塞 InBetween-Broadcast-Thread）
        → 40 個 Broadcast-Thread 同時運行
          → 每個線程分配 4MB ByteBuf
          → 40 × 4MB = 160MB → 超過 1GB 直接記憶體剩餘空間 → OOM
```

---

### 根因 #3：Game Start 54 秒（比 Round 5 惡化 3.2x）

- Round 5（5k 用戶）：17s  
- Round 6（10k 用戶）：54s（期望 ~34s，實際 54s）

**原因**：13,617 個任務 / 40 個線程 = 每條線程需順序處理 340 個 sendMsg。CPU 被 10k 連接飽和，每個操作耗時更長，積累至 54s。  
**加上** CallerRunsPolicy 導致 `InBetween-Broadcast-Thread` 被阻塞執行任務，無法繼續處理其他 queue。

---

### 根因 #4：SettlementMQListener 修復未上線

SLS 16:07:29 仍在報 Gift 扣費失敗：

```
2026-04-03 16:07:29 ERROR Scheduled-Thread-6 SettlementMQListener
  调用结算接口发送Gift扣费记录失败.uid:5556 GiftOrder:Gift-1774632924
```

本輪 Settlement mock / disable 修復未覆蓋此問題。

---

### 根因 #5：CPU / Memory 壓測後不釋放

| 原因 | 詳情 |
|------|------|
| 10k 玩家 Session 對象滯留 | WS 斷線後 `players` Map 未即時清理 |
| PLAYER_BROADCAST_ASYNC_POOL 積壓 | 10k+ 任務仍在 queue，線程池無法釋放 |
| SettlementMQListener 持續重試 | 每 5s 一次 Gift 失敗重試，CPU 持續活躍 |
| 缺少 ZGC + ZUncommitDelay | JVM Heap 膨脹後不歸還 OS |

---

## 四、APISIX 分析

**結論：APISIX 正常，是受害者而非根因。**

| 觀察 | 解讀 |
|------|------|
| 16:08 APISIX 只有 57 條日誌 | Server 崩潰後，幾乎沒有流量能到達 APISIX |
| 16:10 APISIX 爆增 10,023 條 | 舊 WS 連接（持續 ~230s）正常斷開，APISIX 記錄關閉事件 |
| 16:11–16:16 ~235,000 條/min | APISIX 發現後端 unhealthy，對所有新請求返回 503 |
| 16:17+ 恢復正常 | 壓測結束，後端逐漸可用 |

APISIX 在 16:17 後 log 量立即回落至正常水平（~140/min），說明其本身資源充足，無需調整。

---

## 五、與前幾輪對比

| 指標 | Round 4（10k, Ops 優化）| Round 5（5k, Dev 優化）| Round 6（10k, Dev 優化）|
|------|----------------------|----------------------|----------------------|
| Game Start | 99-109s | **17s** ✅ | 54s ⬆️ |
| OOM 類型 | Heap OOM | 無 OOM ✅ | **Direct Memory OOM** 🆕 |
| 503 爆量 | 75,481 | 無 ✅ | **341,629** 🆕 |
| CPU 釋放 | 不釋放 | **釋放** ✅ | 不釋放 ⬆️ |
| Memory 釋放 | 不釋放 | 不釋放 | 不釋放 |
| SettlementMQ 修復 | 否 | 是 ✅ | **未生效** ⬆️ |
| APISIX 問題 | 無 | 無 | 無 ✅ |

---

## 六、修復建議

### P0 — 立即解決：增大 MaxDirectMemorySize（純 Ops，無需改代碼）

在 `server.yaml` 的 `JAVA_TOOL_OPTIONS` 中加入：

```yaml
- name: JAVA_TOOL_OPTIONS
  value: >-
    -Xms4g -Xmx5g
    -XX:MaxDirectMemorySize=3g
    -XX:+UseZGC -XX:ZUncommitDelay=120
    -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:+ExitOnOutOfMemoryError
```

**理由**：10k WS 連接需要 ~1GB 直接記憶體，廣播 ByteBuf 需要額外 1-2GB。設 3g 是安全下限。

---

### P1 — 高優先：修改 `doAction(BroadcastTask)` 為 Batch 提交

**問題**：一次提交 13,617 個任務給 40 個線程，直接記憶體峰值壓力極大。

**建議改法**：

```java
private void doAction(BroadcastTask task) {
    var playerList = new ArrayList<>(rooms.getPlayers().values());
    int batchSize = 200;  // 每批 200 人，減少並發 ByteBuf 分配
    for (int i = 0; i < playerList.size(); i += batchSize) {
        List<Player> batch = playerList.subList(i, Math.min(i + batchSize, playerList.size()));
        getExecutor().execute(() -> {
            for (Player player : batch) {
                try {
                    Channel ch = player.getSession().getChannel();
                    if (ch.isActive() && ch.isWritable()) {  // isWritable() 背壓檢查
                        MessageUtil.sendMsg(player.getSession(), task.getMsgId(), task.getBytes());
                    } else if (!ch.isActive()) {
                        rooms.onPlayerExitRoom(player.getUid());
                    }
                } catch (Exception ignore) {}
            }
        });
    }
}
```

**效果**：
- 13,617 人 / 200 = **68 個 executor 任務**（原來 13,617 個）
- 峰值並發直接記憶體分配從 40 × 4MB = 160MB → **可控**
- 加入 `isWritable()` 背壓防止 ByteBuf 無限堆積

---

### P2 — 高優先：同步修復 `noticeTasksQueue` 的 isWritable 檢查

```java
private void doAction(NoticeTask task) {
    var player = rooms.getPlayers().get(task.getUid());
    if(player != null){
        Channel ch = player.getSession().getChannel();
        if (ch.isActive() && ch.isWritable()) {
            MessageUtil.sendMsg(player.getSession(), task.msgId, task.bytes);
        } else if (!ch.isActive()) {
            rooms.onPlayerExitRoom(task.getUid());
        }
    }
}
```

---

### P3 — 中優先：限制 PLAYER_BROADCAST_ASYNC_POOL 規模

當前 `max = CpuCores * 4 = 40`，配合 batchSize=200 優化後，可進一步調整：

```java
// MBSConfigure.java
@Bean(MBSConstant.PLAYER_BROADCAST_ASYNC_POOL)
public ThreadPoolTaskExecutor asyncThreadPoolBroadcastExecutor(){
    // max 降至 CpuCores * 2 = 20，減少並發直接記憶體壓力
    return getThreadPoolTaskExecutor("Broadcast-Thread-", CpuCores, CpuCores * 2, 5000, 30, 60);
}
```

---

### P4 — 中優先：確認 Settlement Mock 已上線

確認 QAT 環境中 Gift 結算 mock / disable 生效。SLS 顯示本輪仍有 Gift 扣費失敗 log，說明未生效。

---

### P5 — 低優先：Session 清理機制

在 `BinaryWebSocketFrameHandler.channelInactive()` 中確保強制清理：

```java
@Override
public void channelInactive(ChannelHandlerContext ctx) {
    // 確保 players Map 移除斷線玩家
    if (currentPlayer != null) {
        rooms.onPlayerExitRoom(currentPlayer.getUid());
    }
}
```

---

## 七、下一輪壓測前 Checklist

- [ ] `JAVA_TOOL_OPTIONS` 加入 `-XX:MaxDirectMemorySize=3g -XX:+UseZGC`
- [ ] `doAction(BroadcastTask)` 改為 batchSize=200 批量提交
- [ ] `doAction(NoticeTask/BroadcastTask)` 加入 `ch.isWritable()` 背壓檢查
- [ ] 確認 Settlement Gift mock 在 QAT 已生效（SLS 無 Gift 扣費失敗 log）
- [ ] 建議維持 5k 用戶規模驗證 Direct Memory 修復，穩定後再升至 10k
