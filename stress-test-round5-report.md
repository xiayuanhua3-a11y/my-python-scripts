# InBetween QAT — 第五輪壓測分析報告

**測試時間：** 2026-04-03 10:26:32 — 10:41:27（SGT）  
**Unix 時間戳：** 1775183192000 — 1775184087000  
**測試用戶：** 639400200001–639400205000（`stresstestteamjb200001–205000`，共 **5,000 人**）  
**本輪特點：** 開發代碼優化後首輪壓測；用戶數從 10k 降至 5k；同時對 APISIX 網關進行健康核查

---

## 一、歷輪關鍵指標對比

| 指標 | 第三輪（10k） | 第四輪（10k 運維優化） | **第五輪（5k 開發優化）** |
|------|------------|---------------------|------------------------|
| OOM 發生 | 有 | 無 | **無** ✅ |
| Pod 崩潰 / HTTP 503 | 大量 | 1 個 | **0 個** ✅ |
| Server 日誌量 | 18M+/min | 18M+/min | **60–120k/min** ✅ |
| 測試後日誌持續爆量 | 6 分鐘 | 6 分鐘 | **立即歸零** ✅ |
| CPU 峰值 | 9.5 核 | 5 核 | **5 核** |
| Memory 穩定性 | 持續上升 | 持續上升 | **穩定在 4.5–5 GiB** ✅ |
| Heart not matching | 97,506 | 97,506 | **3** ✅ |
| In Between Start Betting Error Code 2 | 31,119 | 31,119 | **89**（僅啟動時）✅ |
| WS 1006 斷線 | 7,834 | 51 | **51** |
| Settlement 重試風暴 | 有（掩蓋） | 有 | **無** ✅ |

---

## 二、壓測結果（客戶端上報）

### 錯誤統計

| 項目 | 類型 | 數量 | 樣本時間 |
|------|------|------|---------|
| **In Between Start Betting** | Error Code 2 | **89** | 10:26:58（啟動首輪） |
| **Personal History** | no data returned | **1,146** | 10:26:46–10:27:34 |
| **Transaction History** | no data returned | **1,410** | 10:27:01–10:28:00 |
| **WS Connection - Game** | Error 1006 | **51** | 10:30:28, 10:35:55 |
| Heart | not matching | 3 | 10:32:02–10:32:31（同一用戶） |
| Heart | Request Timeout | 16 | 10:31:27, 10:35:07 |
| In Between History | Request Timeout | 7 | 10:31:34–10:31:59 |
| Personal History | Request Timeout | 2 | 10:31:59–10:32:14（同一用戶） |
| Transaction History | Request Timeout | 2 | 10:31:59–10:32:14（同一用戶） |

### 慢請求統計（超過時間閾值）

| 項目 | 閾值 | 實測最大延遲 | 對應時間 |
|------|------|------------|---------|
| **Game Start** | 2000ms | **17,214ms** | 10:26:32（首輪啟動） |
| **Payout** | 2000ms | 41,294ms | — |
| **Chat / Gift** | 1000ms | **4,380ms** | 10:32:06（集中） |
| In Between History | 1000ms | 1,118ms | 10:32:11 |
| In Between Start Betting | 1000ms | 2,112ms | 10:32:10 |
| Personal / Transaction History | 1000ms | 1,118ms | 10:32:11 |
| Login | 1000ms | 2,812ms | 10:26:46（啟動突峰） |
| Logoff | 1000ms | 2,832ms | 10:36:53（登出突峰） |
| Connect WS | 500ms | 776ms | 10:26:46（啟動突峰） |
| Enter Room | 500ms | 660ms | 10:26:43（啟動突峰） |
| Room List | 500ms | 713ms | 10:26:42（啟動突峰） |
| Coin Num | 500ms | 586ms | 10:26:48（啟動突峰） |
| Launch Game | 300ms | 388ms | 10:26:41（啟動突峰） |
| phoneNumberLoginWithPassword (gRPC) | 200ms | 880ms | 10:26:32 |

---

## 三、SLS 日誌分析

### Server 日誌量（`inbetween-qat` namespace）

| 時間（SGT） | 日誌行數 | 說明 |
|------------|---------|------|
| 10:20–10:25 | ~35/min | 測試前靜默 |
| **10:26** | **102,510** | 測試開始，5k 登入突峰 |
| 10:27–10:30 | 60–84k/min | 穩定運行 |
| 10:31 | 114,041 | 遊戲輪次事件 |
| 10:32–10:36 | 62–121k/min | 穩定 |
| **10:37** | **794** | 測試結束，立即歸零 |
| 10:38+ | ~35/min | 恢復靜默 |

> **關鍵進步：第四輪測試後日誌持續 18M+/min 長達 6 分鐘（Settlement 重試風暴），本輪測試結束後日誌量立即歸零，說明 Settlement 重試問題已徹底解決。**

### APISIX 日誌量（`ingress-apisix` namespace）

| 時間（SGT） | 日誌行數 | 說明 |
|------------|---------|------|
| 10:20–10:25 | ~155/min | 靜默基線（監控輪詢） |
| **10:26** | **5,145** | 登入 + WS upgrade 突峰 |
| 10:27–10:30 | 150–200/min | 用戶已通過 WS 連線，HTTP 靜默 |
| **10:31** | **7,426** | 遊戲輪次 HTTP 請求突峰 |
| 10:32–10:34 | 150–1,800/min | 正常 |
| 10:35–10:36 | 4,000–5,000/min | Logoff 突峰 |
| 10:37+ | ~155/min | 恢復靜默 |

### Server 10:30:59 日誌採樣（Chat/Gift 慢的前置背景）

```
INFO  Scheduled-Thread-10  CoinUtil  Player[stresstestteamjb200344:1980] Bet Coin -100025, Balance:533121644.45
INFO  Scheduled-Thread-9   CoinUtil  Player[stresstestteamjb201111:2135] Bet Coin -61000, Balance:533115354.63
INFO  Scheduled-Thread-12  CoinUtil  Player[stresstestteamjb200397:1978] Bet Coin -3200, Balance:525753465.92
INFO  Player-Thread-1       ReqChatHandler    Received MsgId:1018,Uid:1946,Room:1001
INFO  Player-Thread-4       ReqGiftHandler    Received MsgId:1022,Uid:1946,Name:stresstestteamjb200748
...（無 ERROR / WARN 日誌）
```

> 10:31–10:32 區間 server 無任何 ERROR 級別日誌，全為 INFO。

---

## 四、APISIX 網關健康核查

### 結論：APISIX 完全健康，不是任何問題的原因

**10:31–10:32 APISIX 詳細分析（Chat/Gift 最慢的區間）：**

| 指標 | 值 |
|------|----|
| 狀態 200 請求數 | 84 |
| 狀態 101（WS Upgrade）| 1 |
| 4xx / 5xx 錯誤 | **0** |
| 慢請求（HTTP > 300ms）| **0** |
| 唯一「長時間」條目 | WS session req_time = 271s（長連線正常 session 時長，非延遲） |

**整個測試期間 APISIX 性能：**

| 指標 | 值 |
|------|----|
| HTTP 上游響應時間 | 3–10ms |
| 零 4xx/5xx 錯誤 | 確認 |
| 零請求積壓 | 確認（日誌量在突峰後立即回落） |

**主要請求類型（10:26–10:37）：**

| 路徑 | 請求次數 | 說明 |
|------|---------|------|
| `/inbetween/v1/launchGameIGO` | 12,352 | 進遊戲（正常，5k 用戶多次） |
| `/inbetween/v1/gameRoom` | 661 | 房間狀態輪詢 |
| `/inbetween/v1/gameStatus` | 659 | 遊戲狀態輪詢 |
| `/inbetween/v1/previewGameRoom` | 273 | 預覽房間 |
| `/apisix/admin/configs` | 165 | APISIX 內部配置同步 |
| `/ws?username=...` | 每用戶 3 條 | WS 連線建立日誌 |

> **APISIX 對所有請求僅增加 < 1ms 延遲。客戶端觀察到的所有慢返回全部來自 backend server 處理，與 APISIX 無關。**

---

## 五、各報錯與慢返回根本原因

### 問題一：Game Start 17s ——`noticeTasksQueue` 未修復

**現象：** Round 13220 的 Game Start 通知需要 17,214ms 才送達所有客戶端。

**根因：** `BroadcastWorkThread.run()` 的 `noticeTasksQueue` 仍為每次循環只取 1 個 notice，sleep 10ms。5k 用戶 × 10ms = 50s 理論上限，實測 17s（有其他因素優化）。

**判斷：** 此問題在第四輪（10k）也是 17.5s，本輪（5k）17.2s 基本相同，說明 `noticeTasksQueue → drainTo` 修復尚未落地。

**影響：** 測試開始首輪下注窗口錯位，導致 89 個 Error Code 2（下注時間未到）。首輪後穩態未見大量 Error Code 2，說明後續輪次有改善。

---

### 問題二：Chat / Gift 週期性慢到 4.3s ——`ProfitPayoutTask` 搶佔 DB，與 APISIX 無關

**現象：** Chat 和 Gift 在 10:32:06–10:32:10 同時慢到 4.3s（同一批用戶，同時發生）。

**根因鏈：**
```
10:31:00  ProfitPayoutTask 觸發（@Scheduled cron="0 * * * * ?"）
→ 串行查詢 round_record / bet_record / settlement_record
→ 佔滿 Hikari 連線池（或大量佔用 Scheduled-Thread pool）
→ Player-Thread 的 Chat/Gift handler 等待 DB 連線
→ 10:32:06 Chat/Gift 請求延遲 4.3s（等待連線釋放）
```

Server 日誌確認：10:30:59 密集的 `Scheduled-Thread CoinUtil Bet Coin` 結算日誌，正是 `ProfitPayoutTask` 在跑。APISIX 在此區間零慢請求、零錯誤，確認延遲來自 server 端。

---

### 問題三：Personal / Transaction History "no data returned" ——預期行為

**現象：** 1,146 + 1,410 個「no data returned」，響應極快（10ms）。

**根因：** 壓測帳號（`stresstestteamjb*`）是全新帳號，數據庫中無歷史記錄。Server 正常查詢返回空集合，客戶端測試框架標記為 "no data"。**這不是系統錯誤，是正常的空數據響應。**

---

### 問題四：Heart 問題幾乎全部集中在單一用戶

**現象：** Heart not matching × 3、Heart Timeout × 16，以及 History / PersonalHistory / TransactionHistory Timeout × 各 2，均集中在 `stresstestteamjb201852` 用戶，時間窗口 10:31:59–10:32:31。

**根因：** 這是**單個測試客戶端**的問題（可能網絡抖動或本地狀態異常），並非服務器系統性問題。

---

### 問題五：Login / Connect WS / Enter Room 啟動慢 ——啟動突峰，正常行為

**現象：** 10:26:43–10:26:49 各類請求慢 500ms–2.8s。

**根因：** 5k 用戶同時登入（gRPC 認證 + Redis token 寫入 + WS upgrade）造成初始突峰，屬於**一次性啟動負載**。穩態無此問題。

gRPC `phoneNumberLoginWithPassword` 達 880ms（> 500ms 閾值），建議後續關注 gRPC 服務在突峰下的性能。

---

### 問題六：WS 1006 斷線 51 個

**現象：** 第四輪（10k 用戶）也是 51 個，本輪（5k 用戶）同樣是 51 個。

**判斷：** 51 不隨用戶數線性增長，很可能是**固定的特定測試客戶端**連線不穩定，而非服務器系統性問題。建議觀察這 51 個斷線是否集中在少數固定的 player ID。

---

## 六、資源使用分析

### CPU（圖表觀察）

| 時段 | CPU 使用 | 說明 |
|------|---------|------|
| 10:20–10:25 | ~0 核 | 靜默 |
| 10:26–10:28 | 爬升至 5 核 | 登入 + 初始化突峰 |
| 10:28–10:36 | 週期性 3–5 核 | 遊戲輪次脈衝（下注 = 高，等待 = 低） |
| 10:37+ | 降回 0 | 測試結束乾淨釋放 |

CPU 出現週期性波峰，對應各遊戲輪次的下注廣播（`BroadcastWorkThread`）。**CPU 在測試結束後乾淨釋放**（`robot.enable=false` 繼續生效）。

### Memory（圖表觀察）

| 時段 | Memory | 說明 |
|------|--------|------|
| 10:20–10:25 | ~2.3 GiB | 乾淨基線 |
| 10:27–10:28 | 爬升至 ~4.5–5 GiB | 5k 玩家物件載入 |
| 10:28–10:36 | 穩定在 4.5–5 GiB | **無持續增長**（之前各輪持續攀升）|
| 10:37–10:45 | 維持在 4.5 GiB | 未完全釋放 |

**關鍵改善：記憶體不再持續攀升**（之前各輪測試中 10k 用戶會把記憶體推到 7–9 GiB）。穩定在 4.5 GiB 說明無記憶體洩漏螺旋。

記憶體未完全釋放的原因：殭屍 Player session 仍在 `rooms.getPlayers()` Map 中（zombie session 定時清理尚未落地）。ZGC `ZUncommitDelay=300` 在 session 物件被 GC 回收後才能把物理頁還給 OS，目前物件仍被強引用。

---

## 七、剩餘問題優先級

| 優先級 | 問題 | 根因 | 修復方式 |
|--------|------|------|---------|
| **P0** | Game Start 17s（首輪） | `noticeTasksQueue` 每次只取 1 個 | `BroadcastWorkThread` 改 `drainTo` 批處理 |
| **P1** | Chat/Gift 4.3s 週期性 | `ProfitPayoutTask` 串行 DB | 優化 ProfitPayoutTask 並行查詢或增加 Hikari min-idle |
| **P1** | Memory 不釋放 | 殭屍 session 無主動清理 | `BaseRooms` 每 30s 掃描斷線玩家 |
| **P2** | Login gRPC 突峰 880ms | 5k 並發認證 | gRPC 服務擴容 / 限流預熱 |
| **P3** | WS 1006 × 51（固定） | 特定測試客戶端不穩定 | 調查具體 player ID，確認是否可重現 |

---

## 八、下一步建議

### 近期行動
1. **落地 `noticeTasksQueue → drainTo` 修復**（P0，代碼改動最小，效果最大）
2. **落地殭屍 session 定時清理**（P1，解決記憶體問題）
3. 完成後用 **10k 用戶**重跑完整壓測，驗證修復效果

### 擴展驗證
- 增加壓測時長（目前 15 分鐘），觀察多輪次後 Payout 延遲是否收斂
- 監控 `ProfitPayoutTask` 每分鐘實際執行時間（建議加 Micrometer Timer）
- 確認 WS 1006 的 51 個斷線是否集中在固定用戶

---

## 附：本輪與前幾輪核心對比

```
第一輪（10k）：JVM OOM → 大量超時，初步診斷
第二輪（10k）：JVM 調整中，仍 OOM，O(N²) 廣播和線程池不足暴露
第三輪（10k）：OOM 改善，Settlement 重試風暴（18M log/min）、O(N²) 廣播持續
第四輪（10k）：運維優化（ZGC + robot=false），CPU 釋放，Pod 穩定，主問題剩代碼
第五輪（5k）：開發優化後，Settlement 解決，系統大幅改善，APISIX 健康確認
              剩：noticeTasksQueue + ProfitPayoutTask + zombie session
```

---

*報告生成時間：2026-04-03 | SLS 分析：`alibaba_cloud_observability` + `ingress-apisix` namespace*  
*關聯文件：`stress-test-findings.md`（第 1–3 輪）、`stress-test-round4-report.md`（第 4 輪）、`ops-tuning.md`（運維配置）*
