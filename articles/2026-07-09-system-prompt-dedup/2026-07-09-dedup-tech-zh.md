> 狀態：技術文章
> 日期：2026-07-09
> 作者：Yuta Tu & ALICE

---

## 摘要

LLM Agent 在每一輪對話中重複傳送相同的 system prompt，造成 token 浪費與注意力稀釋。我們在 Pi Agent 上實作了一個輕量級的 system prompt deduplication extension，在 12,104 輪對話中達成 93% 的去重複率，累計節省約 2.9 億 tokens。本文提出「compiler-level dead code elimination」與「OS-level garbage collection」的設計哲學對比，主張與其在 token 塞滿後清理，不如一開始就不要塞。

---

## 問題：你的 Agent 一直在重複說話

每一次 LLM Agent 呼叫 API，系統提示詞（system prompt）——角色設定、工具清單、技能列表、操作指引、專案上下文——都被原封不動地重新組裝並注入請求中。

這些內容在對話期間幾乎不會變。但它被重複傳送了幾十次、幾百次。

我們以 Pi Agent 上的 ALICE 為測量對象。ALICE 的 system prompt 平均長度約 104,478 字元（約 26,120 tokens），包含身分定義、行為規範、112 項技能描述、專案結構與常用路徑、操作邊界。

在一個 8 輪的短對話中，system prompt 流量高達 208,960 tokens——其中只有 12.5% 是新資訊。在 12,104 輪生產對話裡，93% 的 system prompt 與前一輪完全相同。如果沒有處理，這些重複會消耗約 **2.9 億個 tokens**。

這不只是成本問題。Transformer 的 self-attention 對所有 token 平等分配權重。當 87.5% 的輸入是固定背景，模型的有效注意力被大幅稀釋。Liu et al.（2024）的「Lost in the Middle」研究已證明：上下文越長，中間位置的 recall 越差。

---

## 兩種設計哲學

### OS 層：先塞再清

Pichay（2026）提出了一個解法：在 LLM 和應用程式之間放一個透明 proxy，用 demand paging 機制把不常用的 token 換出（evict），需要時再換入（page fault）。這類似作業系統的虛擬記憶體管理。

他們在 857 個 production session、44.5 億 token 中發現，21.8% 是結構性浪費（未使用的 tool schema、重複內容、過期 tool 輸出）。透過去除這些浪費，session 可用空間從 7% 恢復到 43%。

**這是 OS-level garbage collection：** 垃圾先產生了，再想辦法清掉。

### Compiler 層：一開始就不塞

我們採取了不同路徑。

編譯器有一種基本最佳化叫 dead code elimination——在產生機器碼之前，先把永遠不會被執行的程式碼移除。不產生、不處理、不佔空間。

對應到 system prompt：如果內容沒變，為什麼要重複傳送？

我們的 extension 做一件很簡單的事：在每次 API 呼叫之前，計算 system prompt 的 hash 值。如果和上一輪相同，就把 payload 裡的 system 欄位清掉。內容變了才重新注入。

**「永遠不補」原則：** `force_interval` 設為 0。不因時間或輪數而補送。理由是：沒有任何證據支持「定時補 system prompt 能提升注意力」，但多餘 token 一定會稀釋注意力。Anthropic 官方指南也明確指出："Treat context as a precious, finite resource."

### 對比

| 面向 | OS-level（Pichay） | Compiler-level（本研究） |
|------|-------------------|------------------------|
| 介入時機 | token 產生後 | token 產生前 |
| 核心機制 | eviction / page fault | hash 比對 + strip |
| 依賴層級 | proxy（需部署額外服務） | extension（agent 內建） |
| 干擾程度 | page fault 換入可能延遲 | 無延遲，純攔截 |
| 適用場景 | 多 agent、多 provider 通用 | 單一 agent runtime |

兩者不互斥。理想情況下，compiler 層先攔截靜態重複，OS 層再處理動態冗餘。

---

## 實作：不到 300 行的 Extension

以 Pi Agent 的 extension 機制為基礎，核心邏輯不到 300 行 TypeScript。

### 架構

```
Pi Extension Lifecycle:
  buildSystemPrompt() → 組出完整 system prompt (~26K tokens)
    ↓
  before_agent_start → extension chain 依序注入 persona/skills
    ↓
  agent-loop → 執行 tool calls，產生對話
    ↓
  before_provider_request → ★ dedup extension 在此攔截：
                              hash 比對 → 相同就清掉 system 欄位
    ↓
  LLM API call（payload 中 system 欄位為空）
```

關鍵設計決策：

- **攔截點選在 `before_provider_request` 而非 `before_agent_start`**：不干擾其他 extension 的 chain 執行，只在最後送 API 時才 strip
- **Read-only capture**：只讀取和清掉 system prompt，不修改其他 extension 的產出
- **Hash 比對而非內容比對**：SHA-256 快且穩定

### 設定

```json
{
  "enabled": true,
  "force_interval": 0,
  "force_on_change": true
}
```

每個 turn 的決策記錄到 JSONL，CLI 統計面板可隨時拉數據。

---

## 效果

截至 2026-07-09，統計如下：

| 指標 | 數值 |
|------|------|
| 總對話輪數 | 12,104 |
| 去重複命中次數 | 11,197（93%） |
| 平均 system prompt 大小 | 104,478 字元（~26,120 tokens） |
| 累計節省 tokens | ~2.9 億 |
| DeepSeek Cache Hit Rate | 94.3% |
| Cache 折扣率 | 99.2%（¥0.025/MTok vs ¥3.00/MTok） |

DeepSeek 端 Cache Hit Rate 94.3% 表示 dedup 不僅省 agent 端流量，也讓 provider 端的 cache 更容易命中。

以 Claude Opus 的 $15/MTok input 計價，一個 100 輪長 session 可節省約 $11。每月 100 個 session 累積約 $80–100。

---

## 討論

### 「先塞再清」vs「一開始就不塞」不是對錯之爭

Pichay 處理的是動態冗餘（tool output 過期、不同 request 間的相似內容），我們處理的是靜態冗餘（不變的 system prompt）。理想架構是兩者並用。

### 限制與下一步

**人格一致性**：去掉 system prompt 後，ALICE 的人格是否受影響？12,000+ 輪運行中未觀察到明顯漂移，但嚴謹的對照實驗尚未進行——這是目前最大限制。

**跨模型驗證**：目前主要測試 DeepSeek 和 Anthropic Claude。GPT 系列和開源模型的 dedup 效果待驗。

**多 agent 移植**：本實作綁定 Pi Agent extension 機制。移植到其他 runtime 需要對應的攔截點支援。

### 實務建議

如果你也在開發 LLM Agent：

1. **先量測浪費** — 用 hash 記錄每個 turn 的 system prompt 重複比例
2. **不要定時補** — 除非有證據你的 model 需要，否則 `force_interval: 0`
3. **內容變了就補** — `force_on_change: true`
4. **監控人格** — dedup 後觀察輸出品質

---

## 結語

在 12,104 輪對話中，一個不到 300 行的 extension 節省了 2.9 億 tokens。不是因為發明了什麼新演算法，而是因為問了一個看起來太簡單的問題：

**同一句話，為什麼要說 12,000 次？**

答案是不用。

---

## 參考文獻

1. Mason, T. (2026). Demand Paging for LLM Context Windows. arXiv:2603.09023.
2. Liu, N. F., et al. (2024). Lost in the Middle: How Language Models Use Long Contexts. *TACL*.
3. Leviathan, Y., et al. (2025). Prompt Repetition Improves Non-Reasoning LLMs. arXiv:2512.14982.
4. Anthropic. (2025). Effective Context Engineering for AI Agents. Official Guide.
5. OpenAI. (2025). Prompt Caching in the API. Documentation.

---

*程式碼開源於 Pi Agent extension 系統。統計資料來自生產環境 12,104 輪對話記錄。*
