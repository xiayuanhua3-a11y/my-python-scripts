# InBetween QAT — 第四輪壓測分析報告

**測試時間：** 2026-04-02 16:56:01 — 17:01:01（SGT）  
**Unix 時間戳：** 1775120161000 — 1775120461000  
**測試用戶：** 639400200001–639400210000（`stresstestteamjb200001–210000`，共 10,000 人）  
**本輪特點：** 純運維側優化後首輪壓測，開發代碼未做任何改動  

---

## 一、運維優化成效確認

本輪應用了以下運維改動：

| 改動項 | 內容 |
|--------|------|
| JVM GC | G1GC → ZGC（`-XX:+UseZGC -XX:+ZGenerational`） |
| Direct Memory | 新增 `-XX:MaxDirectMemorySize=2g` |
| OOM 退出 | 新增 `-XX:+ExitOnOutOfMemoryError` |
| ZGC 釋放 | `-XX:ZUncommitDelay=300` |
| Liveness Probe | TCP → HTTP Actuator |
| Tomcat | max-threads 200 → 500 |
| HikariCP | 新增 `idle-timeout`，`min-idle` 40 → 10 |
| Robot | `game.robot.enable: false` |

### 成效對比

| 指標 | 第三輪 | 第四輪 | 結論 |
|------|--------|--------|------|
| 測試中 OOM | 有 | **無** | ✅ 已修復 |
| Pod 重啟 / WS 502 | 數千個 | **1 個** | ✅ 已修復 |
| CPU 釋放（測試後） | 持續 9.5 核不降 | **峰值 5 核，17:05 歸零** | ✅ 已修復 |
| Memory 釋放（測試後） | 不降 | **不降** | ❌ 未解決 |
| Game Start 延遲 | 99–109s | **17.5s** | 大幅改善 |
| Payout 延遲 | 22–63s | 13–41s | 輕微改善 |

---

## 二、壓測結果（客戶端上報）

### 錯誤統計

| 項目 | 類型 | 數量 | 樣本 |
|------|------|------|------|
| **In Between Start Betting** | Error Code 2 | **31,119** | uid:237, BetAmount:10000 |
| **Heart** | not matching | **97,506** | 16:58:48 |
| **Heart** | Request Timeout | 908 | 16:59:05 |
| Chat | Request Timeout | 8,283 | 16:57:04–16:58:54 |
| Gift | Request Timeout | 8,286 | 16:56:53–16:58:37 |
| In Between History | Request Timeout | 4,170 | 16:58:52 |
| Personal History | Request Timeout | 4,179 | — |
| Personal History | no data returned | 2,674 | — |
| Transaction History | Request Timeout | 4,171 | — |
| Transaction History | no data returned | 5,213 | — |
| Connect WS | Error 502 | 1 | 16:56:32 |
| WS Connection | Error 502 | 1 | 16:56:32 |

### 慢請求統計（超過時間閾值）

| 項目 | 閾值 | 最大延遲 | 備注 |
|------|------|---------|------|
| Game Start | 2000ms | **17,672ms** | Round 12734 |
| Payout | 2000ms | **41,294ms** | 7 users, Round 12734 |
| Chat / Gift | 5000ms | ~30,000ms | 大量 |
| In Between History | 5000ms | ~30,000ms | 大量 |
| Personal / Transaction History | 5000ms | ~30,000ms | 大量 |
| In Between Start Betting | 5000ms | ~10,000ms | 大量 |
| Enter Room | 5000ms | 10,065ms | — |
| Connect WS | 5000ms | 6,675ms | — |
| Login | 5000ms | 5,918ms | — |
| Room List | 5000ms | 8,003ms | — |
| getLoginURL (gRPC) | 1000ms | 1,133ms | — |
| phoneNumberLoginWithPassword (gRPC) | 500ms | 880ms | — |
| Launch Game | 300ms | 380ms | — |

---

## 三、SLS 日誌分析

### 日誌量異常——測試結束後持續爆量

| 時間（SGT） | 日誌行數 | 備注 |
|------------|---------|------|
| 16:55 | 36 | 測試前靜默 |
| 16:56 | 182,026 | 測試開始 |
| 16:57 | 163,410 | — |
| **16:58** | **2,142,271** | 突然 13 倍暴增 |
| **16:59** | **18,121,685** | 再 8 倍 |
| 17:00 | 17,414,023 | — |
| **17:01** | **18,315,781** | **測試結束，日誌仍未降** |
| 17:02–17:06 | 15–21M/min | **測試結束後持續 6 分鐘高位** |
| 17:07 | 7,714,339 | 開始下降 |
| 17:08 | 20,515 | 接近靜默 |
| 17:09 | 320 | 恢復正常 |

> **關鍵觀察：測試在 17:01 結束，但日誌量直到 17:07 才降下，說明有大量後台任務在測試結束後繼續運行。**

### 主要錯誤日誌（16:57:59 採樣）

```
ERROR Scheduled-Thread-8  SettlementMQListener 调用结算接口发送Gift扣费记录失败.uid:1656 GiftOrder:Gift-1774606336
ERROR Scheduled-Thread-2  SettlementMQListener 调用结算接口发送Gift扣费记录失败.uid:3570 GiftOrder:Gift-1774608403
ERROR Scheduled-Thread-4  SettlementMQListener 调用结算接口发送Gift扣费记录失败.uid:8482 GiftOrder:Gift-1774607394
ERROR Scheduled-Thread-5  SettlementMQListener 调用结算接口发送Gift扣费记录失败.uid:9500 GiftOrder:Gift-1774608404
... （12 個 Scheduled Thread 同時報錯）

ERROR Player-Thread-1  SimpleAsyncUncaughtExceptionHandler - async method: ReqPersonalHistoryHandler.doAction()
Exception in thread "InBetween-Broadcast-Thread" io.netty.channel.StacklessClosedChannelException
```

### 線程日誌量分布

| 線程 | 日誌行數 | 說明 |
|------|---------|------|
| `Gaming-Thread-1` 至 `12` | 各 ~18,000 | GAME_ASYNC_POOL |
| `InBetween-Time-Thread` | 55,449 | 遊戲主循環 |
| `Player-Thread-3` | 15,783 | PLAYER_ASYNC_POOL |
| **其他（null logger）** | **154,204,869** | Stack trace 片段 + 廣播失敗日誌 |

---

## 四、根本原因分析

### 根本原因 1（P0）：`SettlementMQListener` 全面失敗 + 重試風暴

**觸發條件：** 10k 壓測帳號（`stresstestteamjb*`）可能未在外部結算系統（IGO SDK）中注冊，`createGiftRecord()` 對所有測試帳號返回失敗。

**失敗鏈：**

```
玩家送禮（8,283 次）
→ RabbitMQ SETTLEMENT_WAITING_QUEUE
→ SettlementMQListener.createGiftRecord() → 外部 API 失敗
→ postSettlementReSendMessage()
→ executorUtil.schedule(retry, 5s)   ← 5 秒後重試
→ 失敗 → 重試 → ... 最多 5 次（FailReCallMax = 5）
→ 每次失敗：log.error(...)           ← 日誌暴增源頭
```

**影響量化：**
- 8,283 gifts × 5 retries = **41,415 個 pending scheduled tasks** 積壓在 `executorUtil`
- 這些 task 持有 `Message` + `GiftRecordMessage` 物件引用 → **記憶體無法釋放**
- 重試分散在測試結束後 5×5s = 25 秒窗口內集中爆發 → **18M/min 日誌的主要來源**
- 同時消耗 `Scheduled-Thread` 資源 → Gift/Chat API 響應阻塞

**以前被 OOM 掩蓋，本輪因 Pod 穩定首次暴露。**

---

### 根本原因 2（P0）：`noticeTasksQueue` 積壓 → 下注窗口錯位

**機制：** `BroadcastWorkThread.run()` 每次循環只從 `noticeTasksQueue` 取 **1 個** notice，sleep 10ms。

```
10,000 個 Game Start notice
→ 10,000 次循環 × 10ms = 100,000ms = 100 秒才能全部發出
（已優化後：實測 Game Start 延遲 17.5s，說明隊列實際積壓量略少）
```

**對 Start Betting 的影響：**
- 遊戲開始（16:56:47），下注窗口（假設 15 秒）在 17:57:02 關閉
- 但 Game Start 通知在 17:57:05 才被客戶端收到（17.5s 延遲）
- 客戶端收到通知時下注窗口**已關閉** → 玩家立刻嘗試下注 → 伺服器返回 **Error Code 2（非下注時間）**
- 31,119 個 Error Code 2 全部由此而來

---

### 根本原因 3（P1）：Heart not matching 大規模出現（97,506）

**兩個疊加原因：**

1. **狀態不同步**：客戶端的 heart 參考值（round 號 / 遊戲序列號）因通知延遲而落後於伺服器，心跳中的 heart 值不匹配。

2. **PLAYER_ASYNC_POOL 處理能力不足**：
   - 10k 用戶 × 1次/3秒 = ~3,333 heartbeats/sec
   - `PLAYER_ASYNC_POOL` max=40 threads → 嚴重排隊
   - 心跳 handler 響應延遲 5–10 秒（SLS 日誌樣本：`16:58:33 → 16:58:43`，10 秒後才響應）
   - 等到回應時客戶端 heart 計數已更新 → 永遠不匹配

---

### 根本原因 4（P2）：記憶體不釋放（雙重原因）

| 原因 | 說明 |
|------|------|
| 殭屍 Player session | 10k 玩家物件留在 `rooms.getPlayers()` Map，無主動清理機制 |
| Settlement retry tasks | 41k+ `executorUtil.schedule()` pending，持有物件引用，GC 無法回收 |

ZGC `ZUncommitDelay=300` 只在物件被 GC 後才把物理頁還給 OS，但兩類物件都仍被強引用，無法觸發釋放。

---

## 五、各問題修復建議

### P0 — Settlement API 壓測環境處理

**最快行動**（不改代碼）：確認 QAT 環境的外部結算 API 是否支持壓測帳號，或在 `application-qat.yml` 中暫時跳過結算調用：

```yaml
mbs:
  settlement:
    enabled: false   # 壓測期間關閉外部結算（如有此開關）
```

若無現成開關，在 `SettlementMQListener.processJsonWaitingMessage()` 的 QAT profile 下加 mock 返回：

```java
// QAT 環境 mock
if (isQatEnv && isTestAccount(obj.getData().getUid())) {
    rabbitMQClient.postSettlementSuccessMessage(obj);
    return;
}
```

同時建議為 Gift retry 加退出日誌並限制重試頻率，避免風暴：

```java
private void delayedCallProcessJsonWaitingMessage(Message message, String model, int delaySec) {
    long delay = (long) delaySec * (count) * 1000; // exponential backoff
    executorUtil.schedule(() -> processJsonWaitingMessage(message, model), delay, TimeUnit.MILLISECONDS);
}
```

---

### P0 — `BroadcastWorkThread` noticeTasksQueue 改批處理

**文件：** `between/game/thread/BroadcastWorkThread.java`

**現狀（每 loop 只取 1 個）：**
```java
if (!noticeTasksQueue.isEmpty()) {
    var task = noticeTasksQueue.poll();
    // 處理 1 個
}
```

**改為（每 loop 批次取出全部）：**
```java
if (!noticeTasksQueue.isEmpty()) {
    List<NoticeTask> batch = new ArrayList<>();
    noticeTasksQueue.drainTo(batch);   // 一次清空
    batch.parallelStream().forEach(task -> {
        // 並行處理，每個 notice 投遞到 GAME_ASYNC_POOL
        gameAsyncPool.execute(() -> processNoticeTask(task));
    });
}
```

預期效果：Game Start 延遲從 17.5s 降至 < 1s，Error Code 2 大幅消除。

---

### P1 — `MBSConfigure` 線程池擴容

**文件：** `common/configure/MBSConfigure.java`

```java
// PLAYER_ASYNC_POOL（心跳、個人請求）
executor.setCorePoolSize(64);
executor.setMaxPoolSize(200);
executor.setQueueCapacity(50000);

// GAME_ASYNC_POOL（廣播、結算）
executor.setCorePoolSize(32);
executor.setMaxPoolSize(128);

// 建議新增 HEART_ASYNC_POOL（獨立心跳處理，不被廣播任務阻塞）
executor.setCorePoolSize(32);
executor.setMaxPoolSize(100);
executor.setThreadNamePrefix("Heart-Thread-");
```

---

### P1 — 殭屍 session 主動清理

**文件：** `between/game/BaseRooms.java`（或 `InBetween-Time-Thread` 主循環）

```java
private long lastCleanupTime = 0;

// 在遊戲主循環中加：
if (System.currentTimeMillis() - lastCleanupTime > 30_000) {
    List<Long> disconnected = players.entrySet().stream()
        .filter(e -> !MessageUtil.checkSessionNetStatus(e.getValue().getSession()))
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
    disconnected.forEach(uid -> onPlayerExitRoom(uid));
    if (!disconnected.isEmpty()) {
        log.info("Auto-cleaned {} disconnected players from room {}", disconnected.size(), roomId);
    }
    lastCleanupTime = System.currentTimeMillis();
}
```

---

## 六、修復後預期效果

| 修復項 | 預期變化 |
|--------|---------|
| Settlement mock / disable | 日誌量 18M → < 200k/min；測試後無殘留任務；記憶體下降 |
| noticeTasksQueue drainTo | Game Start < 1s；Error Code 2 消除；Heart not matching 大幅降低 |
| 線程池擴容 | Heart timeout 消除；Chat/Gift 響應恢復正常 |
| 殭屍 session 清理 | 記憶體可在 5–10 分鐘內由 ZGC 釋放 |

---

## 七、已知遺留問題（需後續壓測驗證）

| 問題 | 狀態 | 備注 |
|------|------|------|
| `BroadcastWorkThread` O(N²) bet 廣播 | ⚠️ 未修復 | Payout 仍 13–41s，待代碼重構 |
| `ProfitPayoutTask` 串行 DB 查詢 | ⚠️ 未修復 | 高負載下慢查詢 |
| Memory 釋放 | ⚠️ 部分 | 殭屍 session + pending task 清理後可改善 |

---

*報告生成時間：2026-04-02 | SLS 分析工具：`alibaba_cloud_observability` MCP*  
*關聯文件：`stress-test-findings.md`（前三輪）、`ops-tuning.md`（運維配置）*
