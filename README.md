# my-skills

個人用的 Claude Code Plugin Marketplace，用途是把常用的 Skill 集中管理、跨專案共用，不用每個專案各自複製一份 `.claude/skills`。

Marketplace 名稱：`my-skills`
Repo：`JiangShuuu/my-skills`

---

## 1. 共享 Skill 說明

目前包含以下 plugin（每個都是獨立 plugin，可以單獨安裝）：

| Plugin | 說明 |
|---|---|
| `tanstack-vue-data-table` | shadcn-vue DataTable 完整指南，包含元件生成、useVueTable 寫法、columns 定義、selection state 等規範 |
| `tanstack-vue-query` | TanStack Vue Query 規範，包含 useQuery/useMutation/useInfiniteQuery、queryKeys 工廠函式、Optimistic Update 等慣例 |
| `react-coding-standards` | React coding standards，撰寫元件、管理 state、處理 effect 或 review React 程式碼時使用 |
| `python-fastapi` | 2 個 skill：通用 FastAPI 3-tier 架構指南，以及從 stock-lint-bot 專案整理出的 FastAPI 程式碼規範（type hints、async、DI、錯誤處理） |
| `flutter` | 18 個 Flutter 開發 skill：layout、state management、architecture、animation、performance、testing、theming、routing、databases、caching、concurrency、native interop、plugins、platform views、localization、accessibility、app size、HTTP/JSON、各平台環境設定 |
| `ui-ux-pro-max` | UI/UX 設計指南（第三方，MIT license，原作者 nextlevelbuilder）：50+ 種 style、21 種色票、50 種字體搭配、20 種圖表、9 種技術棧的可搜尋設計資料庫 |

完整清單與描述見 [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json)。

Skill 本身是「model-invoked」——Claude 會依 `SKILL.md` 的 `description` 自動判斷何時該用，一般情況不需要在專案的 `CLAUDE.md` 裡額外重複寫觸發規則。但如果是「特定情境要同時觸發多個 skill」這類複合需求，見第 5 節。

---

## 2. 安裝方法（以專案為單位，建議寫進專案設定、team 共用）

這個 marketplace 是給「專案」用的，建議做法是把註冊資訊直接寫進**該專案**的 `.claude/settings.json` 並 commit 進 repo，
team 每個人 clone 下來就有一致的設定，不用各自在自己機器上重複註冊一次全域設定。

> 若專案的 `.claude/settings.json` 已經有 `extraKnownMarketplaces` 註冊過 `my-skills`（例如之前已經裝過其他 plugin），代表 Step 1 做過了，可以直接跳到下面的 Step 2。

### Step 1：在專案內註冊 marketplace

在專案根目錄執行（會寫入該專案的 `.claude/settings.json`）：

```bash
cd /path/to/your-project
claude plugin marketplace add JiangShuuu/my-skills --scope project
```

執行後 `.claude/settings.json` 會多出這段：

```json
{
  "extraKnownMarketplaces": {
    "my-skills": {
      "source": {
        "source": "github",
        "repo": "JiangShuuu/my-skills"
      }
    }
  }
}
```

### Step 2：安裝需要的 plugin（同樣用 `--scope project`）

```bash
claude plugin install <plugin-name>@my-skills --scope project
```

例如：

```bash
claude plugin install tanstack-vue-data-table@my-skills --scope project
claude plugin install tanstack-vue-query@my-skills --scope project
claude plugin install react-coding-standards@my-skills --scope project
claude plugin install python-fastapi@my-skills --scope project
claude plugin install flutter@my-skills --scope project
claude plugin install ui-ux-pro-max@my-skills --scope project
```

這會在同一份 `.claude/settings.json` 補上 `enabledPlugins`。

### Step 3：commit `.claude/settings.json`

把 Step 1、2 寫入的 `.claude/settings.json` commit 進專案 repo。之後 team 成員 clone 這個專案：

- **marketplace 註冊會自動完成**：開啟專案時 Claude Code 跳出「trust this folder」提示，同意後就會自動讀取 `extraKnownMarketplaces`、註冊好 marketplace，**不需要**自己手動下 `claude plugin marketplace add`。
- **但 plugin 安裝這步不會自動發生**——`enabledPlugins` 只是聲明「這個專案要用哪些 plugin」，外部來源（GitHub repo）的 plugin 不會自動下載/載入，每個人 clone 後仍需自己手動跑一次：

```bash
claude plugin install <plugin-name>@my-skills
```

安裝完後啟動 `claude`，可以：
- 直接打 `/tanstack-vue-data-table:tanstack-vue-data-table`、`/react-coding-standards:react-coding-standards`、`/ui-ux-pro-max:ui-ux-pro-max`、`/flutter:<skill-name>` 手動叫用
- 或什麼都不打，讓 Claude 依 description 自動觸發（例如「幫我設計這個頁面的架構」「檢查這段 React code」）
- 用 `/plugin` 或 `/context` 確認 plugin 是否已載入

---

## 3. `--scope` 說明（補充：不寫進專案、只在自己機器用的情境）

上面 Step 1/2 用的是 `project` scope；如果不想動到專案設定（例如只是想自己先測試看看），還有其他兩種 scope：

| `--scope` | 寫入位置 | 適用情境 |
|---|---|---|
| `project`（推薦，見第 2 節） | 該專案的 `.claude/settings.json`，會被 commit、team 共用 | 確定要讓整個團隊都用這個 plugin |
| `local` | 該專案的 `.claude/settings.local.json`，通常有 gitignore，只影響你自己這台機器、這個專案 | 個人測試、還沒確定要不要團隊共用 |
| `user`（預設） | 全域使用者設定，不寫入任何專案 | 這個 plugin 你想在所有專案都能用，跟這個 repo 的 team 共用設定無關 |

只想自己本機測試、不動專案設定的範例：

```bash
cd /path/to/your-project
claude plugin install flutter@my-skills --scope local
```

只想全域裝好、不限定哪個專案：

```bash
claude plugin marketplace add JiangShuuu/my-skills
claude plugin install flutter@my-skills
```

---

## 4. 更新 Skill 方法及維護

### 使用端（其他專案）：拉取最新版本

```bash
# 更新本機的 marketplace 目錄快照
claude plugin marketplace update my-skills

# 更新已安裝的 plugin 到最新版
claude plugin update flutter@my-skills
```

> 若 plugin 沒有設 `version`（或 version 沒變），可能不會觸發更新，維護時記得在 `plugin.json` bump `version`。
   - ex. `flutter/.claude-plugin/plugin.json` 的 `version`

### 維護端（這個 repo）：新增／修改 skill

1. 在對應的 plugin 資料夾下新增／修改 `SKILL.md`（多個 skill 的 plugin 放在 `skills/<name>/SKILL.md`，單一 skill 的 plugin 可以直接放 `SKILL.md` 在 plugin 根目錄）
2. 若是新 plugin，記得：
   - 建立 `<plugin>/.claude-plugin/plugin.json`（`name`、`description`、`version`、`author`）
   - 到根目錄 [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json) 的 `plugins` 陣列補上一筆（`name`、`source: "./<plugin>"`、`description`）
3. 改完先本機驗證，確定沒壞掉再 push：

```bash
claude plugin validate ./<plugin-name>
```

4. `git add` / `git commit` / `git push` 到 `main`，其他人下次 `claude plugin marketplace update my-skills` 就會拿到最新內容。

### 注意事項

- 每次改動 plugin 的實質內容，建議同步 bump `plugin.json` 裡的 `version`，使用端才會判斷有更新可拉。
- `ui-ux-pro-max` 內容來自第三方（nextlevelbuilder，MIT license），只搬了實際 skill 內容（`SKILL.md` + `data/` + `scripts/`），沒有帶入原 repo 的 git history／CLI 工具；`python-fastapi` 裡的 `fastapi-coding-standards` 是特定專案（stock-lint-bot）的慣例，非通用規範，套用到其他專案前請自行評估是否適用。

---

## 5. 進階：用 CLAUDE.md 精準觸發特定情境的 Skill

單一 skill 靠自己的 `description` 自動觸發通常就夠了。但遇到以下情況，光靠 description 比對不夠可靠，建議直接在專案的 `CLAUDE.md` 寫明確指令：

- **同一情境要同時觸發兩個以上的 skill**：例如一句「幫我 review 這個 PR」，你希望同時叫用 Claude Code 內建的 `/code-review`，以及自己裝的公司內部規範 skill。這是兩個獨立 skill 各自判斷是否相關（機率性），不保證兩個都會被觸發。
- **自訂 skill 跟內建功能語意重疊**：例如你自己也做了一個叫 `code-review` 的 skill，容易跟內建的 `/code-review` 搶自動觸發、或讓 Claude 難以判斷該叫哪一個。

### 為什麼寫進 CLAUDE.md 比較精準

`CLAUDE.md` 每次 session 都會被載入 context，Claude 會把裡面的指示當作既定規則遵守，等同於一條「複合式行為指令」，不用再賭兩個 skill 的 description 是否都剛好比對命中。遵從率遠高於單純依賴各自 description 被動匹配。

### 寫法範例（以 code review 為例）

```markdown
## Code Review 規範

當使用者要求 review PR、review 這段程式碼，或貼上 PR 連結時，請「同時」呼叫以下兩個 skill：
1. `/code-review`（Claude Code 內建，找 correctness bug 與簡化空間）
2. `/company-tools:company-review-skill`（公司內部規範檢查）
```

要點：
- **一定要寫出完整、可辨識的呼叫名稱**。內建的第一方 skill（如 `/code-review`）沒有 plugin namespace；你自己裝的 plugin skill 要寫成 `/plugin-name:skill-name` 完整形式（例如 `/company-tools:company-review-skill`），Claude 才能準確對應到該呼叫哪一個，不會猜錯。
- **描述觸發條件要具體**：把會出現的措辭都列出來（review PR、review code、貼 PR 連結），比只寫一句「code review 時」更容易命中。
- 這仍然是「指令遵從」而非「強制執行」——CLAUDE.md 是很強的訊號，但本質上 Claude 是理解後照做，不是被 harness 強制綁定，遵從率高但不是 100% 保證。

### 想要更硬的保證（合規等級需求）才需要考慮的做法

如果連 CLAUDE.md 的指令遵從都不夠、需要「一定會觸發」的技術保證，可以改用 **hook**（例如 `UserPromptSubmit`）：寫一個 shell script 判斷使用者輸入是否符合特定 pattern（如偵測 PR 連結或「code review」關鍵字），命中時由 harness 在該次請求注入強制指示。這是 harness 執行的邏輯判斷，觸發時機比純指令遵從更確定，但設定較複雜，一般情境不需要，只有合規/稽核等級需求才建議上。
