<!--
Sync Impact Report
==================
Version change: (template, unversioned) → 1.0.0
Bump rationale: MAJOR — initial ratification; six principles newly defined.

Derived from, but NOT copied from, mapf-router's constitution v1.0.0. Transferred:
adversarial validation, ladder-based performance, comprehensible traceability,
reproducibility, evidence tiers. Newly added for this project: safety invariants
must be verified by property, not by example (principle II) — the "zero double
booking" acceptance criterion cannot be demonstrated by fixed test cases.

Principle ordering differs from mapf-router deliberately: there the ranking risk was
performance, here it is a correctness invariant under concurrency.

Templates requiring updates:
  ✅ updated  .specify/templates/plan-template.md
  ⚠ pending  .specify/templates/spec-template.md   — 待首個 feature 走 specify 時確認
  ⚠ pending  .specify/templates/tasks-template.md

Deferred TODOs:
  TODO(PERF_LADDER): 原則 III 的階數尚未推導，需 M0/M2 取得場景參數後填入。
-->

# Distributed Slot Allocator Constitution

本專案為**能力建構導向**（非求職導向）：以無人機領域的職缺描述作為能力規格。
本專案為該 JD 所涵蓋的三個專案之一（另兩個為 `mapf-router`、`assertion-engine`），
各專案獨立開設 GitHub repo。

學習路徑為**從實作反推理論**：先做出可運作的系統，再回頭理解背後的理論。
本憲章多處條款（原則 IV 的白話橋接、視覺化驗收）皆為此路徑的支撐設施。

**本專案的核心風險與 `mapf-router` 不同。** 那裡是效能（能不能算得夠快）；
這裡是**併發下的正確性**——一個競態條件可能數千次測試都不出現，出現一次就是雙重配置。
原則順序依此排定。

## Core Principles

### I. 對抗式驗證 (Adversarial Validation) — NON-NEGOTIABLE

Chaos Injector 與一致性稽核器 MUST 獨立於被測系統之外實作：

- Chaos Injector MUST 置於獨立模組（`chaos/`），MUST NOT 匯入 `src/consensus/`
  或 `src/gateway/` 的內部狀態存取路徑
- 「零遺失、零重複」的稽核 MUST 由外部重建配置紀錄後比對，
  **MUST NOT 詢問系統自身是否一致**——系統自我回報的一致性不構成證據
- MUST NOT 依賴被測系統的健康檢查端點來判定它是否已恢復
- CI MUST 以相依性檢查強制上述隔離

**理由**：分散式系統最危險的失效模式，是系統認為自己一致但實際不然。
若稽核工具透過系統自己的視角觀察，這種失效就永遠看不見。

### II. 安全性不變量須以性質驗證，非以範例驗證 — NON-NEGOTIABLE

「零雙重配置」是**安全性不變量（safety property）**，不是效能指標：

- MUST NOT 僅以固定測資的範例測試作為證據——通過 100 次不代表第 101 次不會競態
- MUST 採用 property-based testing：隨機生成併發請求序列，驗證不變量在所有序列下成立
- 併發正確性 MUST 以線性一致性（linearizability）或等價的可序列化性檢查驗證，
  且該檢查器適用原則 I 的獨立性要求
- 所有反例 MUST 縮小（shrink）後納入回歸測試集，並記錄觸發它的種子

**理由**：競態條件的本質是罕見且時序相依。範例測試對它幾乎沒有偵測力，
卻會製造「已測試」的錯覺——這比沒有測試更危險。

### III. 效能以階梯與尾延遲衡量

效能 MUST 以「階數 + 場景」表述，且 MUST 報告尾延遲：

- MUST NOT 以平均延遲宣稱達標。分散式系統的問題幾乎總是出現在 p95 / p99 / max，
  平均值會把它們稀釋掉
- 效能報告 MUST 至少含 p50 / p95 / p99 / max 四個百分位
- 故障恢復時間亦 MUST 以分佈而非單一數值報告

**TODO(PERF_LADDER)**：具體階數（各階的場景、併發數、延遲與恢復時間預算）尚未推導。
MUST 於 M0 定案技術棧、M2 建立量測基礎設施後，依「載具等待槽位可容忍多久」
（受電量餘裕約束）反推填入。在此之前，效能宣稱一律標記為未定錨。

**理由**：沿用 mapf-router 的階梯設計——單一門檻對不熟悉的領域是一堵牆，
過不去就沒有中間狀態。但此處的階數必須由本專案的場景推導，**不得沿用該專案的數值**。

### IV. 文獻可追溯且可理解 (Traceable AND Comprehensible)

每個演算法模組 MUST 標註文獻來源，**且 MUST 附帶白話橋接說明**：

- MUST 記錄論文出處與對應的虛擬碼／圖表編號（Raft 尤其適用：
  任期、log matching、commit 規則各自對應論文哪一節）
- MUST 附白話說明：這個機制在防止什麼壞事發生、關鍵直覺、
  論文符號與本專案資料結構的對應關係
- 白話說明 MUST 足以讓未讀過該論文的人理解實作在做什麼；
  「請參閱原論文」MUST NOT 作為說明
- 與已發表演算法的任何簡化 MUST 記錄簡化內容、理由，
  **以及該簡化放棄了哪些保證**

**理由**：本專案採「從實作反推理論」的學習路徑，不預設實作者具備直接閱讀論文的能力。
只有引用而無橋接的追溯，對學習者等於沒有追溯。Raft 的「簡化版」尤其危險——
不寫明放棄了什麼保證，就會誤以為自己實作了完整的 Raft。

### V. 混沌測試與壓測可重現 (Reproducible Chaos)

- 所有故障注入 MUST 接受顯式亂數種子；失敗情境 MUST 可原樣重播
- 壓測結果 MUST 記錄：併發數與請求模式、節點數、commit SHA、硬體、種子
- 缺乏上述來源資訊的數據 MUST NOT 出現在報告或 README 中

**理由**：混沌測試找到的 bug 若無法重播，等於沒找到。

### VI. 證據等級制 (Evidence Tiers)

所有效能與正確性宣稱 MUST 標註證據等級：**E1 實測**／**E2 推論**／**E3 判斷**。
任何 Gate 判定 MUST 僅接受 E1。技術決策文件中若關鍵準則僅有 E3，MUST 標記為暫定。

**理由**：推論一旦未標註，會隨時間被當作既成事實累積。

## 技術棧與待決策事項

`kickoff.md` 留下兩個分岔，MUST 於 M0 的 SDD plan 階段定案並留下決策記錄：

- **技術棧**：Python（asyncio + gRPC）vs. C++（Asio + Thread Pool）
- **共識層**：自實作簡化版 Raft vs. 整合現有輕量函式庫（或兩者並行做對比）

⚠️ **`mapf-router` 的技術棧結論與 spike 數據 MUST NOT 直接套用於本專案。**
該專案為 CPU-bound 圖搜尋，其 spike 量測的是每次節點展開成本，在此無對應物。
本專案為 I/O-bound 併發，決定性因素是併發模型與尾延遲。若需 spike，MUST 重新設計。

## 開發流程與品質門檻

- 每個 feature MUST 走 SDD 流程：specify → clarify → plan → tasks → implement
- 併發與共識核心 MUST 測試先行；測試 MUST 先失敗再實作
- **驗收輸出 SHOULD 包含視覺化佐證**（leader 切換時序、日誌複製進度、
  分區期間的叢集狀態演變）。視覺化是理解分散式行為最直接的管道，
  屬學習基礎設施而非 M5 才處理的裝飾
- 理論教學環節與實作交付**刻意解耦**：教學產出不列入 feature 驗收標準，
  亦 MUST NOT 因實作進度而被省略
- 重大技術決策 MUST 於 `docs/decisions/` 留下決策記錄，含重新評估的觸發條件

## Governance

本憲章的效力高於其他開發慣例。衝突時以本憲章為準，或先修訂憲章再行動。

**修訂程序**：MUST 以文件形式提出，載明變更內容、理由與對既有產出的影響，
並更新本檔的 Sync Impact Report 與版本號。

**版本政策**（語意化版本）：MAJOR 移除或不相容地重定義原則；
MINOR 新增原則或章節、實質擴充指引；PATCH 措辭釐清。

**合規審查**：每個 feature 的 plan 階段 MUST 執行 Constitution Check。
違反原則的設計 MUST 記錄理由或改採合規做法。複雜度 MUST 有正當理由——
「之後可能會用到」不構成理由。

**Version**: 1.0.0 | **Ratified**: 2026-07-20 | **Last Amended**: 2026-07-20
