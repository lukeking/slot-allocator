# 專案二：分散式動態資源分配與鎖定引擎 (Distributed Slot Allocator)

## 專案目標
模擬高併發情境下，多台載具同時搶佔有限資源（充電樁 / 降落窗口），設計一套能防止雙重配置（double booking）、具備節點容錯能力的分散式狀態機。

## 問題定義
給定 M 個資源槽位（各有時間區間限制）與 N 個並發請求者，設計配置演算法在滿足時間區間不重疊約束下，公平且無競態地分配資源；並在單一協調節點失效時，系統仍能持續運作（透過共識機制達成狀態一致）。

## 驗證方式（無需硬體）
- 所有「載具」皆以模擬 client（thread / process / async task）表示，發出併發請求。
- 節點失效、網路分區以程式化方式注入（kill 某個模擬節點、人為延遲/丟包），屬於分散式系統的 chaos testing，不需真實充電站或機隊。
- 驗收指標為「零雙重配置」與「故障恢復時間」，皆可由測試框架自動量測。

## 技術棧
- 主要：Python (asyncio + gRPC) 或 C++ (Asio + Thread Pool)
- 共識層：自實作簡化版 Raft（leader election + log replication），或整合現有輕量函式庫做對比
- 壓力測試：Locust / 自製並發 client 產生器

## 系統架構
```
┌───────────────┐     ┌───────────────────┐     ┌─────────────────┐
│ Client         │ --> │ Slot Allocation     │ --> │ Interval Tree /  │
│ Simulator (N)  │     │ Gateway (Leader)    │     │ Graph Coloring   │
│                │     │                     │     │ Core             │
└───────────────┘     └───────────────────┘     └─────────────────┘
                                │
                                v
                       ┌───────────────────┐
                       │ Raft Consensus      │
                       │ (Leader + Followers)│
                       └───────────────────┘
                                │
                                v
                       ┌───────────────────┐
                       │ Chaos Injector      │
                       │ (node kill/delay)   │
                       └───────────────────┘
```

### 核心元件
1. **Client Simulator**：可設定併發數與請求模式（突發、穩定、尖峰）的模擬載具請求產生器。
2. **Slot Allocation Core**：以 Interval Tree 或 Graph Coloring 演算法判斷時間區間是否衝突並完成配置。
3. **分散式狀態機**：以 Raft（或簡化版）在多節點間複製配置日誌，確保 leader 失效後 follower 能接手且狀態一致。
4. **Chaos Injector**：獨立測試工具，模擬節點崩潰、網路延遲/分區，驗證系統的容錯與恢復能力。
5. **Metrics Dashboard**：記錄請求延遲、衝突拒絕率、故障恢復時間等指標。

## 開發里程碑
| 里程碑 | 內容 | 驗收標準 |
|---|---|---|
| M0 | 專案骨架、CI、基礎通訊層 | client-server 可完成基本 RPC 呼叫 |
| M1 | 單節點 Slot Allocation（Interval Tree） | 單節點下併發請求無雙重配置 |
| M2 | 併發壓力測試框架 | 可模擬 100+ 併發 client，記錄延遲分佈 |
| M3 | Raft 分散式狀態機整合 | 3-5 節點叢集下 leader 選舉與日誌複製正確運作 |
| M4 | Chaos Testing（節點失效/網路分區） | 注入故障後系統於可量測時間內恢復，且配置紀錄零遺失/零重複 |
| M5 | Dashboard + README + Demo | 提供可重現的壓力測試腳本與結果報告 |

## 非目標（Non-goals）
- 不涉及真實充電站硬體協定或無人機降落感測器。
- 不處理跨資料中心級別的地理分散式部署（範圍限定在單一模擬叢集內）。

## Repo 結構建議
```
slot-allocator/
├── README.md
├── src/
│   ├── core/            # Interval Tree / Graph Coloring
│   ├── consensus/        # Raft 實作
│   └── gateway/           # RPC / API 層
├── chaos/                 # 故障注入工具
├── loadtest/               # 併發壓力測試腳本
└── tests/
```
