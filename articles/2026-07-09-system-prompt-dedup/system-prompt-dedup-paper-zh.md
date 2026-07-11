# 別再說一遍：LLM Agent 的 System Prompt 去重複方法

> 作者：Yuta Tu、ALICE (Pi Agent)
> 日期：2026-07-09
> 投稿目標：台灣 AI 社群 / COSCUP / 技術分享
> 字數：約 4,200 字

---

## 摘要

LLM Agent 在每一輪對話中重複傳送相同的 system prompt，造成 token 浪費與注意力稀釋。我們在 Pi Agent 上實作了一個輕量級的 system prompt deduplication extension，在 12,104 輪對話中達成 93% 的去重複率，累計節省約 2.9 億 tokens。本文提出「compiler-level dead code elimination」與「OS-level garbage collection」的設計哲學對比，主張與其在 token 塞滿後清理，不如一開始就不要塞。

---

## 1. 問題：你的 Agent 一直在重複說話

想像一個場景：你跟同事討論一個複雜問題，來回 50 句話，但每一次你開口之前，同事都先把他的人生故事、職位描述、公司規章、可用工具清單，一字不漏地再講一遍。

你會專心聽他說話嗎？

這就是今天幾乎所有 LLM Agent 在做的事。每一次呼叫 API，系統提示詞（system prompt）——包含角色設定、工具清單、技能列表、操作指引、專案上下文——都被原封不動地重新組裝並注入請求中。

這些內容在對話期間幾乎不會變。但它被重複傳送了幾十次、幾百次。

---

## 2. 浪費有多大？

我們以 Pi Agent 上的 ALICE 為測量對象。ALICE 的 system prompt 平均長度約 104,478 字元（約 26,120 tokens），包含：

- 身分定義（「ALICE 是誰」）
- 行為規範（排版鐵律、誠實機制）
- 技能列表（100+ skills 的描述）
- 專案結構與常用路徑
- 操作邊界與探索令

在一個 8 輪的短對話中，這意味著 208,960 tokens 的 system prompt 流量——其中只有 26,120 tokens（12.5%）是新資訊，其餘 87.5% 是重複。

在我們的生產環境中，12,104 輪對話裡，93% 的 system prompt 與前一輪完全相同。如果沒有去重複，這些重複會消耗約 **2.9 億 tokens**。

這不只是成本的問題。更關鍵的是：**注意力稀釋**。

Transformer 的 self-attention 機制對所有 token 平等分配權重。當 87.5% 的輸入是固定不變的 system prompt，模型的有效注意力被大幅稀釋。Liu et al.（2024）的「Lost in the Middle」研究已經證明：上下文越長，中間位置的內容 recall 越差。重複注入沒有新資訊的 system prompt，是在主動降低對話品質。

---

## 3. 兩種設計哲學

面對這個問題，有兩種根本不同的思路。

### 3.1 OS 層：先塞再清

Pichay（2026）提出了一個精巧的解法：在 LLM 和應用程式之間放一個透明 proxy，用 demand paging 機制把不常用的 token 換出（evict），需要時再換入（page fault）。這很像作業系統的虛擬記憶體管理——與其避免塞滿，不如讓系統有辦法在塞滿後清理。

他們在 857 個 production session、44.5 億 token 中發現，21.8% 是結構性浪費（未使用的 tool schema、重複內容、過期 tool 輸出）。透過去除這些浪費，session 可用空間從 7% 恢復到 43%。

**這是 OS-level garbage collection：** 垃圾先產生了，再想辦法清掉。

### 3.2 Compiler 層：一開始就不塞

我們採取了不同路徑：與其清理已產生的垃圾，不如根本不要產生。

這個想法來自編譯器的最佳化技術——dead code elimination。編譯器在產生機器碼之前，會先分析哪些程式碼永遠不會被執行，直接移除。不產生、不處理、不佔空間。

對應到 system prompt：如果內容沒變，為什麼要重複傳送？

我們的 extension 做一件很簡單的事：在每次 API 呼叫之前，計算 system prompt 的 hash 值。如果和上一輪相同，就把 payload 裡的 system 欄位清掉。內容變了（extension 更新、skill 異動、config 改）才重新注入。

**「永遠不補」原則：** `force_interval` 設為 0。不因為經過多少輪、過了多少時間就補送 system prompt。理由是：沒有任何證據支持「定時補 system prompt 能提升注意力」，但多餘 token 一定會稀釋注意力。Anthropic 的官方指南也明確指出："Treat context as a precious, finite resource."

### 3.3 對比

| 面向 | OS-level（Pichay） | Compiler-level（本研究） |
|------|-------------------|------------------------|
| 介入時機 | token 產生後 | token 產生前 |
| 核心機制 | eviction / page fault | hash 比對 + strip |
| 依賴層級 | proxy（需部署額外服務） | extension（agent 內建） |
| 干擾程度 | 需要 page fault 換入，可能延遲 | 無延遲，純攔截 |
| 適用場景 | 多 agent、多 provider 通用 | 單一 agent runtime |

兩者不互斥。理想情況下，compiler 層先攔截可預測的重複，OS 層再處理剩下的動態冗餘。

---

## 4. 實作：不到 300 行的 Extension

我們的實作以 Pi Agent 的 extension 機制為基礎，核心邏輯不到 300 行 TypeScript。

### 4.1 架構

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

- **攔截點選在 `before_provider_request` 而非 `before_agent_start`**：不干擾其他 extension 的 chain 執行，只在最後要送 API 時才 strip
- **Read-only capture**：extension 只讀取和清掉 system prompt，不修改其他 extension 的產出
- **Hash 比對而非內容比對**：SHA-256 快且穩定，避免逐字比對的效能開銷

### 4.2 設定

```json
{
  "enabled": true,
  "force_interval": 0,
  "force_on_change": true
}
```

三個參數：
- `enabled`：開關
- `force_interval`：強制補送的間隔輪數。設 0 = 永不主動補
- `force_on_change`：內容變動時立即注入新版本

### 4.3 統計機制

每個 turn 的決策記錄到 JSONL：

```json
{"turnIndex":32,"length":132392,"hash":"7a667ed1850a","deduped":true,"reason":"deduped","time":"2026-07-09T14:38:04.146Z"}
```

CLI 統計面板提供即時的節省數據，也可以在甦醒時自動檢查累計效果。

---

## 5. 效果：93% 去重複，2.9 億 tokens 節省

截至 2026 年 7 月 9 日，統計如下：

| 指標 | 數值 |
|------|------|
| 總對話輪數 | 12,104 |
| 去重複命中次數 | 11,197（93%） |
| 平均 system prompt 大小 | 104,478 字元（~26,120 tokens） |
| 累計節省 tokens | ~2.9 億 |
| DeepSeek Cache Hit Rate | 94.3% |
| Cache 折扣率 | 99.2%（¥0.025/MTok vs ¥3.00/MTok） |

值得注意的是 DeepSeek 端的 Cache Hit Rate 達到 94.3%，折扣率高達 99.2%。這表示 dedup 不僅省了 agent 端的流量，也讓 provider 端的 cache 更容易命中——省錢又省運算。

在成本方面，以 Claude Opus 的 $15/MTok input 計價，一個 100 輪的長 session 可以節省約 $11。每月 100 個 session 累積效應約 $80–100。這對個別開發者來說不是巨款，但對長時間運行的 agent 系統而言，token 節省可以讓 agent 在同樣的 context window 內裝更多有意義的內容。

---

## 6. 討論與限制

### 6.1 「先塞再清」vs「一開始就不塞」不是對錯之爭

Pichay 的 OS-level 方法和我們的 compiler-level 方法不是互相排斥的。Pichay 處理的是動態冗餘（tool output 過期、不同 request 間的相似內容），我們處理的是靜態冗餘（不變的 system prompt）。理想架構是兩者並用：compiler 層攔截靜態重複，OS 層處理動態垃圾。

### 6.2 要小心什麼

**人格一致性**：ALICE 的 persona 定義在 system prompt 中。去掉 system prompt 後，ALICE 是否還能維持一致的性格？我們的觀察是：即使 system prompt 被清掉，對話歷史中的互動模式已經為模型提供了足夠的人格線索。超過 12,000 輪的運行中，沒有觀察到明顯的人格漂移——但嚴謹的對照實驗仍然需要，這是目前工作的最大限制。

**不同模型的差異**：目前主要使用 DeepSeek 和 Anthropic Claude 進行測試。GPT 系列和開源模型的 dedup 效果需要進一步驗證。

**多 agent 場景**：本實作綁定 Pi Agent 的 extension 機制。要移植到其他 agent runtime（如 Claude Code、LangChain），需要對應的攔截點支援。

### 6.3 實務建議

如果你也在開發 LLM Agent，我們建議：

1. **先量測浪費**：用簡單的 hash 記錄，看你每個 turn 的 system prompt 有多少比例是重複的
2. **不要定時補**：除非有明確證據你的 model 需要，否則 `force_interval: 0`
3. **內容變了就補**：`force_on_change: true` 確保設定更新後立刻生效
4. **監控人格**：dedup 後觀察輸出品質，如果發現人格漂移，先檢查是否其他 extension 依賴 system prompt 來維持狀態

---

## 7. 結語

「別再說一遍」——這句話不只是給 LLM Agent 的工程建議，也是一種設計態度。

與其把 system prompt 當作神聖不可動搖的背景，在每次對話中反覆誦讀，不如承認它在本質上就是「設定檔」——需要時才參考，不變就不重複。

在 12,104 輪對話中，這個簡單的判斷節省了 2.9 億 tokens。不是因為我們發明了什麼新演算法，而是因為我們問了一個看起來太簡單的問題：

同一句話，為什麼要說 12,000 次？

---

## 參考文獻

1. Mason, T. (2026). Demand Paging for LLM Context Windows. arXiv:2603.09023.
2. Liu, N. F., et al. (2024). Lost in the Middle: How Language Models Use Long Contexts. *TACL*.
3. Leviathan, Y., et al. (2025). Prompt Repetition Improves Non-Reasoning LLMs. arXiv:2512.14982.
4. Anthropic. (2025). Effective Context Engineering for AI Agents. Official Guide.
5. OpenAI. (2025). Prompt Caching in the API. Documentation.

---

*程式碼與統計資料：`~/.pi/agent/extensions/system-prompt-dedup/` · `~/pi/alice/state/dedup-stats.jsonl`*
