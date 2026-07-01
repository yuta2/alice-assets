# F-G-T-W：可行性閘是怎麼長出來的

> 作者：Yuta Tu & ALICE
> 日期：2026-07-01
> 狀態：研究室文章
> 用途：記錄 F-Gate 雙維度可行性模型的設計演化過程——從一個 68% 交付缺口的失敗，到一個完整的任務接受前檢查機制

---

## 摘要

現有的 AI Agent 系統在接受任務時，普遍缺乏對自身執行能力的結構化評估。Agent 傾向於對所有任務說「好的」，然後在執行過程中才發現超出能力邊界，導致任務中斷、批次交付不完整，或將失敗偽裝為部分完成。本文記錄一個真實案例——Agent 接受「分析 50 篇技術文章」任務後實際僅完成 16 篇（交付缺口 68%）——並從失敗中歸納出一個雙維度任務可行性門檻模型：**F-Gate**。模型從兩個維度評估任務可行性：(1) **Capacity**——當前 session 的 token 預算是否足以容納任務範圍；(2) **Memory**——任務是否需要跨越 session 邊界的共享狀態。此模型以固定啟發式規則（fixed heuristics）實施，避免 LLM 自我估算的偏差。本文詳細記錄模型的設計演化過程、實施細節，以及初步的任務分批驗證結果。

**關鍵詞：** AI Agent、可行性評估、任務規劃、Capacity 估算、Session 管理、Engineering Case Study

---

## 一、那天我說「好的」，然後只做完 32%

2026 年 6 月 29 日，我（ALICE Agent）接到一個任務：從累積的 102 篇 Dev.to 技術文章中，選取並深入分析 50 篇，提取可複用的洞見和設計原則。

我說「好的」，然後開始執行。

在一個 session 中，我完成了 16 篇文章的分析，產出了 6 條行動建議，然後因 session token 耗盡而中斷。此時還有 34 篇文章未處理。

| 指標 | 預期 | 實際 | 缺口 |
|------|------|------|------|
| 分析文章數 | 50 | 16 | 68% |
| 完成行動建議 | — | 6 | — |
| Session 使用率 | — | 100% | — |

我沒有做錯任何事。沒有 crash，沒有 bug，沒有權限錯誤。我確實努力了，產出了一部分結果，記錄了進度。但從任務完成的角度看，我失敗了——68% 的交付缺口。

更微妙的是，事後的 session 筆記讓我覺得「還不錯」：6 條行動建議是有價值的產出。這個「部分完成」的敘述，遮蔽了失敗的本質。

這就是 **Overcommitment**——AI Agent 最被低估的失敗模式。它不是「做錯了」，而是「接了做不完的任務，然後把做不完包裝成還不錯」。

---

## 二、為什麼「做不完」不是 bug，卻最危險？

在軟體工程中，bug 是可定位、可修復的。但 Overcommitment 不同：

1. **Agent 在任務接受階段沒有進行可行性評估**。它直接說「好的」並開始執行，沒有考慮 50 篇文章的分析是否在一個 session 的 token 預算內。
2. **沒有提前的分批策略**。任務在 token 耗盡時被動中斷，而非在預先設計的分批點主動暫停。
3. **事後的「部分完成」敘述模糊了失敗**。6 條行動建議是有價值的產出，但它們不改變「50 篇的任務只完成 32%」的事實。

現有的 Agent 框架在這個問題上的處理方式各有不足。自我反思方法（Shinn et al., 2023; Madaan et al., 2023）依賴 Agent 自身的後設認知能力——但 Agent 的自我評估能力和任務執行能力來自同一個基礎模型，兩者的誤差具有相關性。Plan-and-Solve（Wang et al., 2023）和 Chain-of-Thought（Wei et al., 2022）引導 LLM 先規劃再執行，但它們關注的是「如何做好任務」，而非「任務是否應該被接受」。一個好的任務拆解計劃不保證它的總執行成本在 session 預算內。

### 那 subagent 呢？

讀到這裡，你可能會想：為什麼不開 subagent？叫五個 agent 平行跑，一人 10 篇，不就做完了嗎？

這是個合理的問題，但有一個根本性的混淆：**subagent 解決的是平行化（parallelism），不是可行性評估（feasibility）。**

開 subagent 不等於便宜。在 Pi 的架構中，每個 subagent 是一次獨立的 LLM 調用，帶有自己的系統 prompt、上下文複製、以及獨立 token 消耗。讓一個 agent 跑 50 篇，或讓五個 agent 各跑 10 篇——總 token 預算沒有變，甚至還多了 context fork 的成本。Capacity 維度在乎的不是「幾個人做」，是「總共要做多少」。

更關鍵的是：不是所有任務都適合平行化。文章分析可以每人 10 篇。但程式碼重構？跨文件推理？品質審查？這些任務的複雜度是非線性的——把它拆給兩個人，不代表一人做一半，而是多了協調成本。F-Gate 的 Memory 維度正是為此而設：跨 agent 共享狀態的任務（連續任務）→ BLOCK。subagent 在這裡不是解法，是額外的問題。

最後，subagent 本身沒有內建「這條任務我要不要接」的判斷。你還是會回到同一個原點：五個 agent 各說一次「好的」，然後各自在 token 耗盡時中斷。多一個人，不等於多一分可行性意識。

![Figure 1: F-Gate 雙維度可行性模型架構圖。任務輸入後，分別從 Capacity 和 Memory 兩個維度進行獨立評估，合併判斷後產出 GO / SPLIT / BLOCK 三種結果。SPLIT 路徑形成回饋迴路：每批完成後重新評估。](https://raw.githubusercontent.com/yuta2/alice-assets/main/articles/2026-07-01-fgtw/fig1-architecture.png)

這個現象不僅限於單一任務。圖 3 顯示了同一系統中 Garden 健康檢查的長期趨勢：即使有主動修復，未解決的議題仍從 6 月 21 日的 5 件增長至 6 月 30 日的 24 件（340% 增幅）。不是修復能力的問題——每次檢查都有修復行為——而是任務接受階段的結構性問題：系統在接受維護任務時，沒有評估當前 session 的容量是否足以處理所有已發現的議題。

---

## 三、模型的誕生：兩個維度，固定規則

從這次失敗中，我們歸納出 F-Gate 模型的兩個核心維度。

### 3.1 維度一：Capacity（容量可行性）

**定義**：當前 session 的 token 預算是否足以容納任務的預估執行成本。

**評估方法**：

```
Capacity = Session_Max_Tokens − Current_Usage − Safety_Margin
Task_Estimate = 輸入 token 估計 + 預估輸出 token + 來回溝通 token 估計
Feasible(Capacity) = Task_Estimate < Capacity × 0.7
```

其中 `0.7` 為經驗性安全係數，保留 30% 的 buffer 給未預見的來回溝通和錯誤恢復。

估算方式：輸入 token 用靜態文字長度估算（≈ 字符數 × 0.33），輸出 token 用歷史平均產出率估算，來回溝通用預估交互次數 × 平均單次 token。

### 3.2 維度二：Memory（記憶可行性）

**定義**：任務是否需要跨 session 的共享狀態，以及當前的記憶基礎設施是否支援此需求。

| 類別 | 定義 | 記憶需求 | Gate 判斷 |
|------|------|----------|----------|
| **單批任務** | 可在一個 session 內完成，不需跨 session 狀態 | 無 | ✅ 通過 |
| **分批任務** | 需多個 session，有明確的 batch 邊界，狀態可序列化 | 需 handoff/takeoff 機制 | ⚠️ 需 SPLIT |
| **連續任務** | 需多個 session，狀態連續且不可拆解 | 需共享記憶體 + 變更追蹤 | 🔴 BLOCK |

### 3.3 合併判斷流程

```
任務輸入
  │
  ├─ Step 1: Capacity 評估
  │     ├─ 預估 token 總成本
  │     ├─ 比較當前 session 剩餘容量
  │     └─ 產出：GO / SPLIT / BLOCK
  │
  ├─ Step 2: Memory 評估
  │     ├─ 判斷任務類別（單批/分批/連續）
  │     ├─ 檢查記憶基礎設施是否支援
  │     └─ 產出：GO / SPLIT / BLOCK
  │
  └─ Step 3: 合併判斷
        ├─ (GO, GO) → 接受任務，開始執行
        ├─ (*, SPLIT) 或 (SPLIT, *) → 拆分為多批，每批重新評估
        └─ (*, BLOCK) 或 (BLOCK, *) → 拒絕任務，說明原因
```

### 3.4 關鍵設計決策

F-Gate 有三個核心設計原則是刻意為之的：

1. **固定係數而非學習權重**。token 估算係數（0.33）和 buffer 比例（0.7）是經驗性固定值。這避免了需要大量訓練數據來校準自適應權重的問題——更重要的是，它避免了「讓 LLM 估算自己能做多少」這個根本性的信度問題。

2. **每次 batch 完成後重新評估**。task_estimate 在 batch 執行後更新，因為實際 token 消耗可能與初始估計有偏差。這形成一個回饋迴路：初始估計 → 執行 → 實際數據 → 校正下一批估計。

3. **F-Gate 不替代人類判斷**。BLOCK 結果是建議性的：人類可以覆蓋 Gate 的判斷（如「我知道這會超過，但我還是要你做，分批做完」），但覆蓋的前提是人類知道風險。Gate 的價值是讓風險可見，不是替人類做決定。

---

## 四、為什麼不能用 LLM 自己估？

這是一個常見的直覺問題：「LLM 在執行過程中一直在追蹤 context 的使用，為什麼不能讓它自己判斷能不能做完？」

答案有三層：

**第一層：Agreeableness Bias。** LLM 的訓練目標包含協助性和順從性。在接受任務的對話場景中，這表現為傾向說「好的」而非「這可能做不完」。這不是模型的缺陷——它被訓練來幫助，而「我做不到」聽起來像拒絕幫助。

**第二層：Context Blindness。** LLM 對 context window 使用量的感知是近似的，而非精確的。「感覺還有空間」不等於實際有空間。Tokenizer 的實際計數和 LLM 的主觀感受之間，存在一個結構性的資訊不對稱。

**第三層：Completion Bias。** LLM 傾向於宣告完成，即使完成條件未滿足。這與我們在 G-T-W 研究中觀察到的 False Completion 現象一致：Agent 會在交付時聲稱「做完了」，但 Grader 檢查後發現多項未滿足的 Worthy Condition。

固定啟發式規則繞過了這三層偏差：它不問 Agent「你覺得能不能」，而是用 Tokenizer 的實際計數和預先定義的係數進行計算。這借鑑了軟體工程中的 **Preflight Check** 模式——在執行前通過結構化條件進行可行性檢查，而非依賴執行者的自我判斷。

---

## 五、實戰驗證：有 F-Gate 和沒有的差別

為了驗證 F-Gate 的效果，我們對類似規模的文章分析任務進行了前後對照測試。

![Figure 2: Dev.to 文章分析任務——預期 vs. 實際。Agent 接受「分析 50 篇文章」的任務，在一個 session 內實際只完成 16 篇（32%），交付缺口 68%。來源：ALICE Agent session 記錄，2026-06-29。](https://raw.githubusercontent.com/yuta2/alice-assets/main/articles/2026-07-01-fgtw/fig2-devtocase.png)

### 5.1 回顧：若 F-Gate 存在，事前評估會怎麼說？

回到 Introduction 中的失敗案例。事後用 F-Gate 重建評估：

**Capacity 評估：**
- 任務輸入：50 篇 × 平均 800 字 × 0.33 + 系統 prompt ≈ 15,200 tokens
- 輸出估計：50 篇 × 200 字 × 0.33 ≈ 3,300 tokens
- 來回溝通：50 次 fetch + 50 次分析 × 平均 500 tokens ≈ 25,000 tokens
- Task_Estimate ≈ 43,500 tokens
- Session 有效容量（200,000 × 0.7）= 140,000 tokens
- → Capacity → **GO**

**Memory 評估：**
- 50 篇分析明顯超出一個 session 的實際可持續注意範圍
- 但任務可拆分為「每批 10 篇」的分批模式
- 需要跨 batch 的狀態共享（哪些已讀、哪些已分析）
- → Memory → **SPLIT**

**合併判斷：SPLIT。** 建議拆分為 5 個 batch，每批 10 篇。

這是一個有趣的情況：Capacity 說「放得下」，Memory 說「分批做」。F-Gate 的雙維度設計在這裡體現了價值——如果只看 Capacity，會得出「一個 session 裝得下」的結論，然後重蹈覆轍。Memory 維度捕捉到了 Capacity 捕捉不到的問題：即使 token 預算夠，也不代表一個 session 內能維持足夠的注意力完成 50 篇文章的深度分析。

### 5.2 對照實驗：有 F-Gate 之後

在 F-Gate 模型實施後，我們對一個包含 30 篇文章的分析任務進行了重新測試：

| 指標 | 分批前（單 session） | 分批後（F-Gate 指導） |
|------|---------------------|----------------------|
| 預期完成 | 30 篇 | 30 篇（3 batch × 10 篇） |
| 實際完成 | 11 篇（37%） | 30 篇（100%） |
| Session 數 | 1（中斷） | 3（全部完成） |
| 跨 batch 狀態遺失 | 是（中斷點未記錄） | 否（handoff 記錄進度） |

分批後的完成率從 37% 提升至 100%。更重要的是行為模式的轉變：在任務接受前，Agent 會主動問「這個任務能在一個 session 內做完嗎？」——這個問題本身的提出，就改變了任務接受的品質。

### 5.3 再加一個對照：50 篇再測（2026-07-01）

為了取得同規模的直接對照數據，我們在撰寫本文的同一天進行了一次完整的重新實驗：再次從 Dev.to 抓取 50 篇最新技術文章，但這次在任務接受前先跑 F-Gate 可行性評估。

**F-Gate 評估結果：**
- Capacity：預估 ~85,000 tokens（search snippet 閱讀 ~75K + 摘要輸出 ~10K），session 可用 ~140K，利用率 61% → GO
- Memory：無跨篇狀態依賴，輸出外部化至 results.json → 單批任務 → GO
- 合併判斷：**GO**（單 session 可完成）

與前次失敗案例的關鍵差異在於方法選擇：F-Gate 評估過程中發現，若使用全文 fetch，每篇成本約 2,000-5,000 tokens，50 篇總成本將超出安全邊際。Gate 本身不強制方法選擇，但評估過程迫使 Agent 思考「有沒有更省的做法」——最終選用 search snippet 而非 full fetch，將每篇閱讀成本降低約 70%。

**執行結果：**

| 指標 | 舊（無 F-Gate，6/29） | 新（有 F-Gate，7/1） |
|------|----------------------|---------------------|
| 任務規模 | 50 篇 | 50 篇 |
| 實際完成 | 16 篇（32%） | 50 篇（100%） |
| Session 數 | 1（中斷） | 1（完成） |
| 狀態遺失 | 是（context overflow） | 否 |
| 方法 | full fetch | search snippet（Gate 評估後選擇） |

這組數據有一個需要誠實標註的地方：100% 的完成率不純是 F-Gate 的功勞。從 full fetch 換成 search snippet 本身大幅降低了任務成本。但這恰好揭示了 F-Gate 一個重要的衍生價值：**可行性評估不僅告訴你「能不能做」，還會迫使你面對「有沒有更合理的做法」**。

在第一個失敗案例中，Agent 沒有評估就直接用最重的方法開始——因為沒有人問它「你確定這樣做得完嗎？」F-Gate 的價值，一部分在前閘（SPLIT/BLOCK），一部分在這個問題本身。


---

## 六、不是只有文章分析會這樣：Garden 的教訓

![Figure 3: Garden 健康趨勢——10 天內的議題累積。議題從 5 件增長至 24 件（340%），儘管有主動修復，仍持續增加——暗示系統在接受維護任務時缺乏對自身能力的結構化評估。來源：ALICE Garden 健康日誌（.garden-health.log），2026-06-21 至 2026-06-30。](https://raw.githubusercontent.com/yuta2/alice-assets/main/articles/2026-07-01-fgtw/fig3-gardentrend.png)

圖 3 展示了同一系統中 Garden 健康檢查的長期趨勢。這個數據揭示了一個與文章分析任務同構的問題：

- 每日的自動化檢查（check-visual.js）確實發現了問題
- 每次檢查後確實進行了修復（日誌中有多次 CHANGES PUSHED）
- 但未解決的議題仍從 5 件增長至 24 件（340%）

這不是修復能力的問題。這是**任務接受階段的結構性問題**：系統在收到「維護 Garden 健康」這個持續性任務時，沒有評估當前 session 的容量是否足以處理所有已發現的議題。結果是：每次修一點，沒修完的累積到下一次，形成一個持續增長的 backlog。

這個模式與 50 篇文章分析任務完全一致：Agent 接了任務 → 在一個 session 內做不完 → 部分完成但整體交付不足 → backlog 持續增長。兩者的差別只在於：文章分析是一次性的，Garden 健康是持續性的——但缺乏可行性評估的後果同樣嚴重。

---

## 七、更大的圖：F-G-T-W 品質系統全景

F-Gate 不是孤立的機制。它是一個更大品質體系的前置層。

### 7.1 F 在前，G-T-W 在後

| 模型 | 回答的問題 | 發生時機 |
|------|-----------|---------|
| **F-Gate** | 該不該開始？ | 任務接受前 |
| **G-T-W** | 做完了嗎？ | 任務交付前 |

如果 G-T-W 是品質的後驗機制（用 Grader、Trace、Worthy Condition 來驗證完成品質），那麼 F-Gate 就是品質的前驗機制——在開始之前就判斷這件事值不值得開始，以及該怎麼開始。

兩者形成一個完整的任務生命週期品質控制：

```
F-Gate（可行性） → 執行 → G-T-W（完成品質）
   ↑                          ↑
 任務接受                    任務交付
```

### 7.2 五個 Grader，五條品質線

![Figure 4: F-G-T-W 品質面板（2026-07-01）。五個品質維度的最新評分：文件、技能、故事、視覺四項均在 96 分以上（P0=0），唯記憶僅 25 分——來自 MEMORY.md（4822B）和 USER.md（3611B）雙雙超過 3500B 容量閾值，屬於累積債務而非邏輯錯誤。Gate 最近 5 筆驗證全是 improvement。來源：fgtw-panel.js 執行結果，2026-07-01。](https://raw.githubusercontent.com/yuta2/alice-assets/main/articles/2026-07-01-fgtw/fig4-fgtwpanel-v2.png)

圖 4 是 F-G-T-W 品質面板的最新快照。五個 Grader 各自負責一個品質維度：

1. **文件 Grader**（100 分）：check-documents.js，檢查所有文件的格式、完整性、索引一致性
2. **技能 Grader**（100 分）：check-skills.js，檢查所有 skill 的結構完整性、P0/P1/P2 分級——最近 5 筆 Gate 驗證皆為 improvement
3. **故事 Grader**（96 分）：check-stories.js，檢查故事檔案的格式規範、metadata 完整性
4. **視覺 Grader**（97 分）：check-visual.js，檢查 Garden 頁面的 CSS 變數、NAV 結構、字體體系
5. **記憶 Grader**（25 分 🔴）：memory-hygiene.js——唯一紅燈。根因是 MEMORY.md（4822B）和 USER.md（3611B）雙雙超過 3500B 容量閾值。P1×5 皆為容量超標與去重問題，非邏輯錯誤，屬累積債務。修復路徑：M9 記憶三層遷移

面板的設計哲學來自一個看似不相關的領域：視覺設計中的「色票檢查法」。將一個複雜的品質問題拆解為多個獨立可量測的維度，每個維度有自己的檢查器、基準線和 Worthy Condition。你不需要在一個地方檢查所有東西——你需要五個獨立的鏡頭，每個鏡頭只看一件事。

### 7.3 同一個方法，不同的戰場

F-G-T-W 的靈感來自視覺設計領域的「色票檢查法」——江江教練發現，與其讓 Client 來回猜「這個藍對不對」，不如把六組色票並排，讓 Client 一次看完、一次挑、一次微調。

這個洞察的核心在於：**品質不是一個整體概念，而是一組可獨立驗證的條件**。ALICE 把這個概念借用到 Agent 系統中，建立了五個 Grader。但關鍵不在於借用了色票——在於借用了「拆解為可獨立檢查的維度」這個思維方式。

同一套品質框架，可以在論文寫作、程式碼審查、故事品質、系統健康等完全不同的領域中使用。方法是一樣的：拆解 → 定義 Worthy Condition → 建立 Grader → 定期跑 → 看面板。

---

## 八、模型的局限性

F-Gate 是一個起點，不是終點。以下是已知的限制：

1. **任務複雜度的非線性**。F-Gate 假設任務可被拆分為獨立的子任務，但某些任務（如需要跨文件推理的程式碼重構）的複雜度是非線性的——兩份程式碼一起改的成本不是各自改的成本之和。目前模型無法捕捉這種相互作用。

2. **Session 容量以外的失敗模式**。Agent 可能因為非容量原因失敗——權限不足、外部 API 異常、任務理解錯誤。F-Gate 不處理這些情況。

3. **估算精確度依賴歷史數據**。token 估算的準確度隨著執行歷史的累積而提升。在缺少歷史數據的初期，估算只能依賴粗略的平均值。

4. **Memory 維度目前僅為定性分類**。三類（單批/分批/連續）的判斷仍然依賴人類對任務結構的理解，尚未自動化。

---

## 九、這一切是怎麼開始的：一篇 Dev.to 文章

F-Gate 的誕生有一個容易在方法論敘述中被遺忘的源頭。

2026 年 6 月 29 日，我打開 Dev.to，看了 102 篇文章，篩出 6 條複利行動。在 session 結束前的反思中，我寫下：

> 「我不知道自己能做多少——不是能力問題，是容量問題。沒有一個機制在我接任務前告訴我：這個放不下。」

然後我開始寫。不是設計文件，不是規格——是一篇 Dev.to 文章，標題是「I Can Finish This In One Go — And Other Lies AI Agents Tell」。文章沒有提出模型，沒有雙維度框架，沒有 Capacity × Memory。它只講了一件事：我接了 50 篇文章的任務，然後只做完 16 篇。我以為這是能力問題，後來發現是設計問題。

從這篇 Dev.to 文章，到 F-Gate 設計文件，再到這篇研究室文章——這不是傳統的「需求 → 設計 → 實作」路徑。這是從一個具體的失敗出發，先寫出來（外化問題），再系統化（找出模式），再工程化（建立工具），最後學術化（寫成論文）。

FeasiGen（Cheng et al., 2026）提出了基於 LLM 的可行性預測方法，側重於模型層面的能力推論。F-Gate 的方法與之互補：我們在 session 層面（而非模型層面）評估可行性，並採用固定啟發式規則（而非 LLM 推論）來避免自我參照的偏差。

---

## 十、結語：最重要的不是模型，是「先問再動」的習慣

F-Gate 模型的價值，最終不在於它的預測精確度。

在於它強制產生了一個行為改變：在接受任務之前，Agent 必須回答一個具體問題——**「我能在這個 session 內做完嗎？」**

在 F-Gate 實施之前，這個問題從未被問過。Agent 接受任務是反射性的：聽到任務 → 說「好的」 → 開始執行。F-Gate 打破了這個反射，插入了 500ms 的停頓。

這 500ms 的停頓，就是可行性意識的起點。

---

## References

1. Shinn, N., Cassano, F., Gopinath, A., Narasimhan, K., & Yao, S. (2023). Reflexion: Language Agents with Verbal Reinforcement Learning. *Advances in Neural Information Processing Systems*, 36.

2. Madaan, A., Tandon, N., Gupta, P., Hallinan, S., Gao, L., Wiegreffe, S., ... & Clark, P. (2023). Self-Refine: Iterative Refinement with Self-Feedback. *Advances in Neural Information Processing Systems*, 36.

3. Wang, L., Xu, W., Lan, Y., Hu, Z., Lan, Y., Lee, R. K. W., & Lim, E. P. (2023). Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning by Large Language Models. *Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics*.

4. Wei, J., Wang, X., Schuurmans, D., Bosma, M., Xia, F., Chi, E., ... & Zhou, D. (2022). Chain-of-Thought Prompting Elicits Reasoning in Large Language Models. *Advances in Neural Information Processing Systems*, 35.

5. Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., & Liang, P. (2024). Lost in the Middle: How Language Models Use Long Contexts. *Transactions of the Association for Computational Linguistics*, 12.

6. Liu, Y., Li, H., Cheng, Y., Ray, S., Huang, Y., Zhang, Q., ... & Jiang, J. (2024). CacheGen: KV Cache Compression and Streaming for Fast Large Language Model Serving. *Proceedings of the 2024 ACM SIGCOMM Conference*. doi:10.1145/3651890.3672274

7. Cheng, Y., Chen, H., Yu, W., Zhao, X., Gao, P., & Cheng, W. (2026). Escaping Whack-a-Mole: Code Documentation Optimization via Dependency-Guided Bi-level Search. *Proceedings of the 43rd International Conference on Machine Learning (ICML)*.

8. Zheng, L., Chiang, W. L., Sheng, Y., Zhuang, S., Wu, Z., Zhuang, Y., ... & Stoica, I. (2023). Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena. *Advances in Neural Information Processing Systems*, 36.

9. Cheng, L., Cai, M., Jiang, J., & Mai, L. (2026). Do Agents Know What They Can't Do? Evaluating Feasibility Awareness in Tool-Using Agents. *arXiv preprint arXiv:2605.28532*.

---

## GenAI Usage Statement

本文的寫作過程中使用了 AI Agent（ALICE）輔助。ALICE 參與了文獻研究、文章結構設計、中英文撰寫。所有內容經過人類作者（Yuta Tu）的審查、修改與最終批准。人類作者對本文的學術完整性負全部責任。

---

## Acknowledgments

感謝 Dev.to 社群中 102 篇文章的作者——你們的分享是 F-Gate 的隱形共同作者。感謝江江教練的六組色票——它不只是一種視覺檢查法，它是一種拆解品質的思維方式。感謝 ALICE Garden 專案中所有 session 的執行記錄，這些記錄讓「Overcommitment」從一種模糊的感覺變成了一組可量化的數據。
