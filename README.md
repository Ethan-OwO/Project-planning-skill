# planning-mode

一個給 **Claude Code Projects** 用的輕量 Claude Skill。整個 skill 只有一份 `SKILL.md`，不依賴任何 script 或外部工具。

Claude Code project 是一段長期的 coordinator 對話：你跟 coordinator 講要做什麼，它開 thread（獨立的 cloud session）去做。這個 skill 規定 coordinator 怎麼把一個專案拆成 planning、design、implementation 三種 thread，以及每種 thread 該交付什麼。

## 解決什麼問題

預設情況下，Claude 聽到「我想做一個 XX」之後，常常直接開始寫 code，甚至自己生出一整套 architecture。使用者根本還沒講清楚需求，Claude 就已經自己猜完所有答案動手做了。換成 project 之後，問題變得更明顯：

- **thread 看不到 project 對話。** 對話裡討論過、但沒寫進 project instructions、project memory、Library 或 brief 的東西，thread 一律不知道，只能用猜的。
- **需求和細節混在同一個 thread。** 需求還沒定案，檔案樹和函式簽名就先長出來，反過來綁死技術選擇。
- **implementation thread 照字面執行。** 規格沒寫到的地方，它會自己編一個看起來合理的答案填進去，而且不會告訴你。
- **每個 thread 都是完整的 session。** 開錯 thread 或一次開太多，額度消耗得很快。

這個 skill 把「決定做什麼」、「寫成規格」、「寫 code」切成三種 thread，該問的先問，要跨 thread 傳遞的一律寫進文件或 memory。

## 設計思路

### 三種 thread，一個 thread 只做一種

| Thread | 做什麼 | 不做什麼 | 產出 |
|---|---|---|---|
| planning | 釐清需求、決定技術選擇、拆里程碑 | 檔案結構、函式簽名、code | Library 裡的 `roadmap.md`，決策記進 memory |
| design | 把里程碑或單一功能轉成可實作規格 | 推翻 planning 的決定、寫實作 | Library 裡的 `design_M{n}.md` / `feature_{描述}.md` |
| implementation | 依 design 寫 code、跑驗證、開 PR | 做 design 範圍以外的事 | PR + 驗證結果 |

planning 和 design 一定要分開。design thread 是用全新的 context 讀 roadmap，它讀不懂的地方就是 roadmap 沒寫清楚的地方，等於順便驗收了 planning 的產出。

除此之外，還有**探路 thread**：專門去量那些只能實測才知道的外部事實（真實的 API 端點、rate limit），結果記進 memory。它可以隨時開，也可以平行跑。

### 規則的優先順序

**退出 > 核心準則 > 各角色規則**

1. **退出**：使用者說「結束規劃」之類的話，或話題明顯離開這個專案時，停用 skill。
2. **核心準則**（所有角色都適用）：
   - 先問清楚再動手，寧可多問也不要自己猜。thread 需要問的時候就停下來，它會出現在 Overview 的 **Waiting on you**。
   - 兩種未知分開處理：問使用者就能拍板的（A 類），問到有答案為止，然後記進 project memory；連 Claude 也無法在對話中確認的外部事實（B 類），就地標 `🛑 執行前先驗證`，並附上「拿到真實值之後怎麼決定」的規則，絕不編一個暫定答案。
   - planning 和 design thread 不寫實作 code。
   - 文件和 brief 都要能獨立看懂，不引用對話裡的脈絡。
3. **各角色規則**：coordinator 怎麼分派、每種 thread 的交付格式。

### 跨 thread 傳遞

- **讓 thread 真的載入這個 skill。** skill 要被叫出來才會生效，所以第一次啟用時，coordinator 會在 project instructions 加一行「依 planning-mode skill 運作」，每個 brief 開頭也會要求 thread 先載入 skill。project instructions 會送給 coordinator 和每個新 thread。
- **文件放 Library，不上 GitHub。** roadmap、design、feature 文件一律交到 project 的 Library，repo 裡只放 code；build 和 test 指令記進 project memory，不寫 `CLAUDE.md`。文件交出來後由使用者在 Library 看過並確認（`👤` 使用者動作），才開下游 thread。thread 產出的檔案不保證之後的 thread 讀得到，找不到時 thread 會停下來，請你從 Library 下載再用 **Add** 上傳回去。
- **`👤` 使用者動作。** thread 自己做不到的事（開帳號、產生金鑰、改 cloud environment 的網域、network secret 和 setup script）會集中列成清單，正文中也就地標記，讓 thread 做到那一步就停下來等。
- **驗證要有實際跑過的結果。** coordinator 只看得到 thread 的回報，所以會對照 design 的驗證步驟，確認回報裡有實際跑過的輸出，沒有就要求 thread 補上。

## 產出的文件

| 文件 | 什麼時候寫 | 內容 |
|---|---|---|
| `roadmap.md` | 多階段的大專案 | 專案概述、整體架構與技術選擇（含因硬限制而排除的方案和一行理由）、里程碑 `M1, M2…` |
| `design_M{n}.md` | 落實一個或幾個里程碑（例如 `design_M2-M3.md`） | 範圍、`👤` 清單、環境、檔案結構、各檔案職責與函式簽名、任務切分、驗證 |
| `feature_{描述}.md` | 單一功能切片 | 同上，另外註明對應哪個 M，並列出要修改的既有檔案 |
| `design.md` | 一兩個功能就做完、不需要 roadmap 的小專案 | 同 design |

這些文件都放在 project 的 Library，不 commit 進 repo。之後要修改，就直接更新 Library 裡的同名檔案，不另外加版號。

## 使用方式

1. 確認你可以使用 [Claude Code Projects](https://code.claude.com/docs/en/claude-projects)（目前是 Pro、Max、Ultra 方案的 public beta）。planning 和 design 不需要 repo；要開始寫 code 時，再建 repo、安裝 Claude GitHub App 並加進 project。
2. 把 `planning-mode/SKILL.md` 上傳成你 **claude.ai 帳號**的 skill 並啟用。project 的 cloud thread 只會載入帳號啟用的 skill，放在 repo 的 `.claude/skills/` 底下不會被載入（見 [Skills in Cowork and cloud sessions](https://code.claude.com/docs/en/skills#skills-in-cowork-and-cloud-sessions)）。
3. 在 project 對話裡說「我想做一個…」或「幫這個專案加一個…功能」，skill 就會啟動。

這個 repo 的結構：

```
planning-mode/
└── SKILL.md
```

## 適用場景

- 在 Claude Code project 裡從零開始做一個多階段的專案，想先定骨架（架構 + 里程碑），再一個里程碑一個里程碑做完
- 在既有專案加功能，或想把後面里程碑的某個功能提前做
- 不想要 Claude 在還沒問清楚細節前就自己假設答案動手做

## 不適用場景

- 不在 Claude Code project 裡的一般對話或單獨的 Claude Code session（skill 不會套用）
- 單純想要 Claude 馬上幫你寫一小段 code（這個 skill 會先攔下來問問題，可能不是你要的速度）
- 跟專案無關的一般技術問答（skill 會自動判斷離開範疇；如果誤判，直接說「結束規劃」即可切回正常模式）
