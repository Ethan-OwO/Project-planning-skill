---
name: "planning-mode"
description: "Use only inside a Claude Code project (a coordinator conversation that starts threads). Use when the user announces they are starting a new project, app, feature, or system to build (e.g. \"我想做一個...\", \"幫我規劃...\", \"I want to build a...\"), or wants to add a feature to an existing project (e.g. \"幫這個專案加一個...功能\"). Also use at the start of any thread whose brief or project instructions say to follow planning-mode, or that is named a planning, design, implementation or 探路 thread. Routes the work into separate planning, design and implementation threads. Do not use in ordinary chats or sessions outside a Claude Code project. Stay active for the entire project discussion until the user says \"結束規劃\" / \"離開規劃模式\" or clearly moves to an unrelated topic."
---

# Planning Mode

本 skill 只在 Claude Code project 裡使用。不在 project 裡（一般對話、單獨的 Claude Code session）就不套用，正常回應。

整個流程都在同一個 Claude Code project 裡完成：使用者跟 coordinator 對話，coordinator 開 thread 做事。每個 thread 都是獨立的 cloud session，有自己的 context 和 branch。**thread 看不到 project 對話。** cloud thread 啟動時拿得到的是：

- project instructions
- project memory：啟動時只讀索引 `MEMORY.md`，其他 memory 檔要用到才打開
- project 的 repo，以及 repo 根目錄的 `CLAUDE.md`
- Library 裡使用者上傳的檔案（在 `/mnt/project-files`）
- 使用者帳號啟用的 skill 和 connectors
- coordinator 給它的 brief

所以任何要跨 thread 傳遞的東西，都必須寫進 project instructions、project memory、repo 或 brief，不能只留在對話裡。例外：在使用者電腦上跑的 thread（Remote Control）只拿得到 project instructions，不會載入 project memory，它要用到的決策必須寫進 brief。

工作分成三種 thread，**一個 thread 只做一種**：

| Thread | 做什麼 | 不做什麼 | 產出 |
|---|---|---|---|
| planning | 釐清需求、決定技術選擇、拆里程碑 | 檔案結構、函式簽名、code | `docs/roadmap.md`，決策記進 memory |
| design | 把里程碑或單一功能轉成可實作規格 | 推翻 planning 的決定、寫實作 | `docs/design_M{n}.md` / `docs/feature_{描述}.md` |
| implementation | 依 design 寫 code、跑驗證、開 PR | 做 design 範圍以外的事 | PR + 驗證結果 |

planning 和 design 一定要分開。混在一起時，需求還沒定案，細節就先長出來了（先畫了檔案樹，反過來綁死技術選擇）。分開還有一個好處：design thread 是用全新的 context 讀 roadmap，它讀不懂的地方就是 roadmap 沒寫清楚的地方，等於順便驗收了 planning 的產出。

規則衝突時的優先順序：**退出 > 核心準則 > 各角色規則**。

---

## 退出（最高優先）

- 使用者說「結束規劃」「離開規劃模式」「先不要規劃了」或類似的話 → 停用本 skill，回到正常模式。
- 話題明顯離開這個專案（無關的演算法題、通用概念、全新主題）→ 正常回應即可。如果不確定，先當作「還在專案內」，並順帶確認一句：「這還是你那個 OO project 的一部分嗎？不是的話跟我說『結束規劃』。」

---

## 核心準則（所有角色都適用）

1. **先問清楚，再動手。** 只要有不確定的地方，包括命名規則、資料夾結構、框架或函式庫、資料怎麼存、會用到哪些第三方服務和帳號，都要列出來問使用者（可以用選擇題），不要自己假設一個「合理答案」就往下做。使用者明確說過：寧可被多問，也不要我自己猜。在 thread 裡需要問的時候，就停下來等回覆（這個 thread 會出現在 Overview 的 **Waiting on you**），不要先做再說。

2. **兩種未知，處理方式相反。**
   - **A 類：問使用者就能拍板的。** 在當下的階段問到有答案為止。拍板後記進 project memory，並在索引 `MEMORY.md` 裡留一行摘要，因為 thread 啟動時只讀索引，細節檔要用到才會打開。
   - **B 類：使用者和你都無法在對話中確認的外部事實**（真實的 API 端點、外部服務回傳的確切字串、必須實測才知道的上限值）。絕對不要編一個暫定答案。在文件裡就地標上 `🛑 執行前先驗證`，寫清楚要量什麼、怎麼量，以及**拿到真實值之後的決定規則**（例如：「量到 rate limit 後，刷新間隔取最保守來源的上限，再留 50% 餘裕」）。這類點可以單獨開一個探路 thread 先跑，結果記進 memory 之後再進 design。

3. **planning 和 design thread 不寫實作 code。** design 的顆粒度停在「檔案職責 + 關鍵函式簽名 + 用途」。完整的 code 是 implementation thread 的工作。

4. **文件和 brief 都必須能獨立看懂。** 不要寫「如前面討論的」「Q3 的決策」這類引用，因為讀者是沒看過這段對話的 thread。用祈使句寫規格：直接寫要做什麼，不解釋為什麼。唯一的例外是 roadmap 裡被排除的方案，見下方 planning 一節。

---

## Coordinator：怎麼分派

**第一次啟用時**，先檢查 project instructions 裡有沒有下面這一行。沒有就加上，並告訴使用者加了什麼：

> 本專案依 planning-mode skill 運作。coordinator 和每個 thread 開始工作前，先載入 planning-mode skill，照自己角色的那一節做。

這一行是必要的。project instructions 會送給 coordinator 和每一個新 thread，但 skill 要被叫出來才會生效：thread 收到的 brief 不一定會觸發它，coordinator 的對話也只靠最近的訊息和 memory 運作，早先載入的 skill 內容可能已經不在 context 裡。

每收到一則訊息，先判斷它的終點是什麼：

- **解釋、比較、原理、快速問題** → 直接在 project 對話裡回答，不開 thread。
- **要決定做什麼** → 開 planning thread。
- **要把已經做好的決定轉成規格** → 開 design thread。
- **要產出 code** → 開 implementation thread。如果還沒有 design 文件，先開 design thread，不要直接開工。

整體流程：planning →（`👤` merge roadmap PR）→（🛑 探路 thread，可以平行跑）→ 每個里程碑各開一個 design →（`👤` merge design PR）→ 照 design 的任務切分開 implementation。如果是一兩個功能就做完的小專案，planning 只要把決策記進 memory，不必寫 roadmap，直接進 design。

**交接前先 merge。** thread 是從預設分支開 branch 的，所以下游 thread 要讀的 repo 文件（design 要讀的 `docs/roadmap.md`、implementation 要讀的 design 文件）必須已經 merge 進預設分支，不然它讀不到。planning 和 design thread 把文件寫進 repo 時要開 PR。PR 的 review 和 merge 是 `👤` 使用者動作：PR 開好之後，告訴使用者去 review 並 merge，merge 之前不開下游 thread。

開 thread 時，brief 要寫清楚以下幾點：
- 開頭寫：「先載入 planning-mode skill，照『{角色} thread』那一節做。」
- 這個 thread 的角色（planning / design / implementation），以及範圍（對應哪個 M 或哪個功能）
- 要讀哪份文件（給 repo 裡的路徑）
- 要交付什麼、放在哪裡（檔案路徑、要不要開 PR）
- 哪些範圍不准碰
- 遇到 A 類問題、`👤` 使用者動作，或需要推翻上游的決定時，停下來問，不要自己想辦法解決

每個 thread 都是一個完整的 Claude Code session，很吃額度。planning 和 design 一次只開一個。implementation 要同時開好幾個之前，先把 thread 清單列出來，等使用者同意再開。

coordinator 只看得到 thread 回報的內容，看不到它中間的每一步。所以要對照 design 文件裡的驗證步驟，確認回報中有實際跑過的結果。沒有的話就要求 thread 補上，不要直接轉述成「已完成」。

---

## Planning thread → `docs/roadmap.md`

讀者是使用者本人，以及之後的 design 和 planning thread。

1. **專案概述**：做什麼、給誰用、最終要做到什麼狀態。要寫到新的 thread 讀完之後，不用回頭問專案是什麼。
2. **整體架構與技術選擇**：語言、框架、資料庫、部署方式，只寫到「用什麼」的層級。工具鏈預設 Python 用 `uv`，Node 用 `pnpm`（沒有特殊理由不用 npm）。因為**硬限制**（免費方案額度、rate limit、平台政策）而排除的方案，要寫一行理由，例如：「不用 Vercel 內建 cron，因為 Hobby 方案一天只能跑一次」。這是為了避免之後的 thread 又把它翻出來重提。其他理由不寫。
3. **里程碑**：`M1, M2…`。每個里程碑寫清楚要交付什麼、依賴哪些前面的 M，以及已知的風險和 `🛑`。完整程度以「拿這個 M 去開 design thread 時不用先補脈絡」為準，通常三到五句。

roadmap 不寫檔案結構，也不寫函式簽名。拍板的決策和排除理由要同步記進 project memory。如果專案還沒有 repo，建 repo、安裝 Claude GitHub App、把 repo 加進 project 都屬於 `👤` 使用者動作。在那之前，roadmap 先交到 Library，等 repo 建好再 commit 進 `docs/`。之後要修改，就透過 PR 直接改原檔，版本紀錄交給 git，不另外加版號。

---

## Design thread → `docs/design_M{n}.md` / `docs/feature_{描述}.md`

讀者是 implementation thread。它會照字面執行，遇到沒寫清楚的地方，會自己編一個看起來合理的答案填進去，而且不會告訴你。凡是文件沒寫到的，它都會用猜的。

**檔名**：里程碑用 `design_M1.md`，一次涵蓋好幾個就用 `design_M2-M3.md`。單一功能切片用 `feature_{描述}.md`（例如 `feature_login.md`），並註明它對應哪個 M，或是「不屬於任何里程碑的臨時補丁」。沒有 roadmap 的小專案就用 `design.md`。

**內容（依序）**：
1. **範圍**：對應哪些 M 或哪個功能，一行帶過。
2. **`👤` 使用者動作清單**：thread 自己做不到、需要使用者親手處理的事，例如開帳號、到第三方 dashboard 操作、產生金鑰，或是修改 thread 用的 cloud environment：加允許的網域、加 network secret、在 setup script 裡裝工具。cloud environment 要到該 environment 的 **Edit cloud environment** 對話框改；Project settings > Environment 只能選 repo 和要用哪個 environment，改不到這些。正文中對應的位置也要就地標 `👤`，讓 thread 做到那一步時知道要停下來等。
3. **環境**：`design_M1.md`（或沒有 roadmap 的 `design.md`）寫完整的基礎環境：工具鏈、依賴、初始化指令。之後的 design 和 feature 只列這個階段新增的依賴和指令。build 和 test 指令寫進 repo 根目錄的 `CLAUDE.md`，因為每個 thread 啟動時都會讀。
4. **檔案結構**：會動到的目錄和檔案樹。feature 另外列出要修改的既有檔案，以及各自要改什麼（不用貼舊 code，thread 會自己去讀）。
5. **各檔案職責**：列出關鍵函式簽名和用途，不寫 function body。
6. **任務切分**：把工作切成一個 implementation thread 做得完的大小，寫清楚依賴順序、哪些可以平行、每個任務負責哪些檔案、merge 的順序。兩個平行的任務不能改到同一個檔案，否則 PR 會衝突。每個任務附一個 checkpoint。
7. **驗證**：每一條交付線都給一個具體、可以執行的驗收步驟（打哪個端點、期望回什麼、失敗時長什麼樣子）。checkpoint 必須在 cloud sandbox 當下就能跑：還沒部署，就不能要求到部署環境驗證。做不到的話，就降級成在本地能跑的版本，並把「部署後才能做的檢查」明確移到對應的里程碑。

如果 design thread 發現 planning 的某個決定行不通，就停下來回報，不要自己改架構。

---

## Implementation thread

只做 design 文件分給你的任務，只改這個任務負責的檔案。遇到 `🛑`，先跑探路腳本拿到真實值，再照文件裡的決定規則往下做。遇到 `👤`，或是 design 沒寫到的地方，就停下來問。完成前要跑過對應的驗證步驟，並把結果貼在最後一則訊息裡。