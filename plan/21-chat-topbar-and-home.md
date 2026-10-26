# M21 對話視窗精簡化（用量圖示）與設定首頁儀表板（1A）

分支 `m21-chat-topbar-home`。依據：[ui-usage-and-topbar.md](ui-usage-and-topbar.md)（對話視窗，**以本文「與已確認稿的差異」為準**）、[方案 C · 整合版@1x.png](方案%20C%20·%20整合版@1x.png)（設定首頁，只做 1A）、[2 · 點開用量面板@1x.png](2%20·%20點開用量面板@1x.png)。

使用者 2026-10-27 決定：

| 項目 | 決定 |
|---|---|
| 訊息下方 token 數 | **保留**「設定 › 外觀與操作 › 進階」開關（預設關閉），不移除 |
| 模型膠囊文字 | **不變**：仍是「供應商 · model ID」，窄版仍有供應商字首方塊 |
| 按鈕尺寸 | **不變**：維持 32，不挑戰 ui-guide.html |
| 用量「正常」顏色 | **跟著主題走**（`LumiAccentBrush`） |
| 範圍 | 做「2 對話視窗」與「1A 設定首頁」；**1B 不做** |
| 首頁記憶卡開關 | 不做（首頁維持只讀） |
| 頂部「N/9 項正常」與「重新檢查」 | 不做（「9 項」沒有定義） |

## 不做的項目（1B 與其他，實作者不要補）

設定搜尋框（⌘K）、「小精靈可以主動說話／勿擾」橫幅、US$ 費用與每月預算、工作台／小精靈用量比例、提醒「近 7 天觸發」圖與「本週已觸發」、「明細 ›」「查看全部紀錄 ›」連結（沒有對應頁面）、頂部健康度。

## 與已確認稿（ui-usage-and-topbar.md）的差異

| 已確認稿 | 本計畫 |
|---|---|
| 第二列按鈕 36、圓角 10、Surface 底、高 56；標題列 macOS 44 | **全部維持現況**：標題列高 40、工具列 `Padding="12,8"`、Chrome 底、`toolbar` 按鈕高 32 |
| 用量按鈕 36、圓環 22／半徑 8／線寬 3 | 按鈕 **32×32**，小圓環 **直徑 20、線寬 2.5**；面板大圓環照稿（直徑 104、線寬 10） |
| 模型膠囊顯示名稱、不顯示連線名稱 | 不變（「供應商 · model ID」） |
| §7 移除訊息下方 token 數 | 保留（進階開關） |
| 新增 7 個色彩 token（`UsageNormal` 等） | **不新增色票**，用既有 token：Normal＝`LumiAccentBrush`、Warning＝`LumiWarningIconBrush`、Danger＝`LumiDangerBrush`、紅點＝`LumiDangerDotBrush`、開啟中底＝`LumiAccentSoftBrush`、軌道＝`LumiBorderBrush`、大圓環軌道＝`LumiBorderBrush`、警示底／框＝`LumiWarningBackgroundBrush`／`LumiWarningBorderBrush` |
| 輸入方塊在「供應商沒回報快取」時改畫箭頭 | 簡化為：**本對話累計快取讀取 = 0 時一律畫輸入箭頭圖示、寫「tokens」**（不新增「有回報」旗標、不改資料庫）；> 0 才畫迷你圓環與「快取 N%」 |
| 「⋯」匯出／刪除在空對話 | 空對話（沒有任何訊息）時三項全部停用 |

其餘（兩列分層、齒輪移到第一列、「⋯」只放對話動作、用量圖示與面板內容、四種等級與門檻、數字格式、累計定義、快滿提醒、模型選單精簡、回應中行為、驗收）照 ui-usage-and-topbar.md。

## 工作拆分

| 包 | 內容 | Wave | 負責 |
|---|---|---|---|
| Phase 0 | 字串、圖示、介面簽章、ViewModel 骨架、計畫文件更新 | — | 協調者 |
| 21A | 用量資料層：DeepSeek 快取解析、本月／今天 token 加總查詢、最近工具動作查詢 | 1 | 子代理 |
| 21B | `UsageRing` 控制項（含四種等級、動畫、虛線空狀態）與 `UsageLevel` | 1 | 子代理 |
| 21C | 第一列齒輪、第二列「⋯」改成對話動作（重新命名、匯出 Markdown、刪除）、匯出器 | 1 | 子代理 |
| 21D | 首頁資料：`HomeViewModel` 改寫（待處理、各卡資料、最近動作） | 1 | 子代理 |
| 21E | 用量按鈕＋面板、`ConversationUsageViewModel`、累計用量、模型選單精簡 | 2（21A、21B 合併後） | 子代理 |
| 21F | `HomePanel.axaml` 重寫、側欄橘點 | 2（21D 合併後） | 子代理 |
| 整合 | DI、`SettingsHost` 測試輔助、文件、手動驗證清單、整體測試 | — | 協調者 |

通則：照 [[subagent-worktrees-manual]] 的做法——先 commit Phase 0，再 `git worktree add -b w21<包> ../DesktopAgent-wt/w21<包> HEAD`，每個子代理在自己的 worktree 工作，不 commit、不改 `plan/`、不碰清單以外的檔案；**不新增字串**（缺字串就停下來回報）；只用既有 token 與樣式 class；建置零警告。`WorkbenchViewModel` 是 `partial class`：各包把新增的成員放進**自己的新檔案**（`WorkbenchViewModel.ConversationActions.cs`、`WorkbenchViewModel.Usage.cs`），不要大改原檔，避免合併衝突。遇到計畫沒寫到或與程式現況不符就停下來回報，不要猜。

回報格式：改了哪些檔案、新增測試數量與結果、和計畫不同的地方與原因、沒做完的項目。

---

## Phase 0（協調者）

1. **字串**（`Strings.resx`＋`Strings.cs`）。命名前綴：`Usage_`（用量面板與圓環）、`Workbench_Rename／Export／DeleteConversation…`、`Home_`（儀表板）。至少包含：
   - 用量：本對話用量、上下文、空間充足、已用超過一半、快滿了建議開新對話、還可以再用約 {0}、輸入／輸出／回合、快取 {0}%、tokens／來回、已連線 · {0}／尚未確認／連線失敗 · 重試、費用說明（沿用 `Workbench_CostNote`）、快滿提醒文字與「開新對話」「繼續這則」、圓環 Tooltip（`上下文 {0} · 點一下看詳情`、`尚無用量資料`）、無障礙名稱（`本對話用量：上下文 {0}`）、「<1%」。
   - 「⋯」：對話選項（AutomationProperties.Name）、重新命名對話、匯出為 Markdown、刪除這則對話、重新命名對話框的標題與按鈕、匯出完成／失敗訊息。
   - 首頁：頁首（設定首頁／Lumi 設定總覽）、待處理 · {0}、重點設定、最近的動作、目前模型（切換 ›）、記憶（管理 ›、記住的事 {0}／{1} 則、已用字數 {0}／{1}）、本月用量、今天 {0}、輸入／輸出、工具（內建工具、MCP 伺服器、{0} 已啟用、{0} 已連線、{0} 個工具、未連線、重新連線）、提醒（待觸發提醒、執行中計時器、下一個提醒：…）、動作狀態（已允許、你已拒絕、失敗）、相對時間（剛剛、{0} 分鐘前、{0} 小時前、昨天）、空狀態（目前沒有需要處理的項目、還沒有動作紀錄）、待處理各項標題與說明（AI 連線尚未測試、AI 連線失敗、沒有金鑰、沒有模型、語音辨識尚未設定）與按鈕（測試連線、重新測試、前往設定）。
   - 舊的 `Home_*` 卡片字串（`Home_AppearanceTitle` 等）不再使用的可以刪（`StringsTests` 只擋「有屬性沒資源」）。
2. **圖示**（`Icons.axaml`，同步 `IconTests` 的清單）：新增 `ArrowDown`（輸出）、`ArrowRight`（輸入，備援）、`Repeat`（回合）、`Download`（匯出，選用）。既有可用：`Settings`（齒輪）、`Check`、`X`、`Ban`、`Refresh`、`AlertTriangle`、`Pencil`、`Trash`。
3. **介面簽章**：
   - `IFilePickerService.SaveFileAsync(string suggestedName, string extension)` → `Task<string?>`（取消回 null）；`NullFilePickerService` 與測試用的 Fake 同步補上；真正實作用 Avalonia `StorageProvider.SaveFilePickerAsync`。
   - `IConversationStore`（用 default 實作，仿檔內其他預設方法，Fake 不用改）：
     `Task<TokenTotals> GetTokenTotalsAsync(DateTimeOffset since, CancellationToken ct = default)`（回 `TokenTotals(long Input, long CacheRead, long Output)`，預設全 0）；
     `Task<IReadOnlyList<RecentToolAction>> ListRecentToolActionsAsync(int limit, CancellationToken ct = default)`（`RecentToolAction(DateTimeOffset At, string ConversationId, ToolResultBlock Result)`，預設空清單）。
   - `UsageLevel` 列舉（`Empty／Normal／Warning／Danger`）與純函式 `UsageLevel.From(bool hasData, double ratio)`（放 `Lumi.App.ViewModels`，21B 實作內容）。
4. **骨架**：`ConversationUsageViewModel`（屬性清單見 ui-usage-and-topbar.md 第 4 節，內容 21E 實作）；`HomeViewModel` 新增的公開成員簽章（見 21D）。
5. **文件**：在 [ui-usage-and-topbar.md](ui-usage-and-topbar.md) 開頭加一行「與本稿不同處以 plan/21-chat-topbar-and-home.md 為準」；`docs/ui-spec.md` 2.1、2.3、第 3 節、設定首頁段落由整合階段一次改完。

---

## 包 21A：用量資料層（Wave 1）

- **先讀**：`src/Lumi.Providers/OpenAICompatible/OpenAIStreamAccumulator.cs`（`ParseUsage`）、`src/Lumi.Infrastructure/Persistence/SqliteConversationStore.cs`（讀寫 `input_tokens…`、`created_at` 的格式與既有比較寫法）、`src/Lumi.Core/Chat/IConversationStore.cs`、`TokenUsage.cs`、`ContentBlock.cs`（`ToolResultBlock`）、`tests/Lumi.Providers.Tests`、`tests/Lumi.Infrastructure.Tests` 裡的對話 store 測試。
- **可修改**：上列 src 檔、對應測試檔與測試用 Fake store（若有）。
- **步驟**：
  1. `ParseUsage`：快取 = `prompt_tokens_details.cached_tokens`；沒有時改讀 DeepSeek 的 `prompt_cache_hit_tokens`；兩者都沒有是 0。`InputTokens` 維持「`prompt_tokens` 減快取」。
  2. `SqliteConversationStore.GetTokenTotalsAsync(since)`：`SUM` 各 token 欄位（`COALESCE(...,0)`），條件 `created_at >= since` 且 `input_tokens IS NOT NULL`；`since` 的格式照該檔存 `created_at` 的格式轉換。`Input` 要含快取（`input + cache_creation + cache_read`），`CacheRead` 另外回。
  3. `ListRecentToolActionsAsync(limit)`：查 `role = 'tool'` 的訊息依 `created_at` 由新到舊（一則工具訊息可能有多個 `tool_result` 區塊，要展開後再取 `limit` 筆，依區塊所在訊息時間排序）；解析 `content_json` 的 `ToolResultBlock`。不新增 migration、不改 schema。
- **測試**：DeepSeek 兩種欄位、只有 `cached_tokens`、都沒有；`GetTokenTotalsAsync` 跨月邊界、沒有 usage 的訊息不算、cache 欄位 NULL；`ListRecentToolActionsAsync` 順序、`limit`、一則訊息多個結果、損毀 JSON 略過不丟例外。
- **不要做**：不改 `TokenUsage` 的欄位、不加 migration、不動 `ConversationService`。

## 包 21B：`UsageRing` 控制項（Wave 1）

- **先讀**：[ui-usage-and-topbar.md](ui-usage-and-topbar.md) 2.1、2.3；`src/Lumi.App/Controls/LevelWaveform.cs`（自訂繪製控制項的寫法）、`Themes/Tokens.axaml`、`tests/Lumi.App.Tests/Theming/PaletteContrastTests.cs`。
- **新增**：`src/Lumi.App/Controls/UsageRing.cs`、`UsageLevel` 的實作（`src/Lumi.App/ViewModels/UsageLevel.cs`）、測試。
- **步驟**：
  1. `UsageRing : Control`，StyledProperty：`Ratio`（0–1，夾住）、`Level`（`UsageLevel`）、`Diameter`、`Thickness`、`DisplayRatio`（動畫用）。覆寫 `Render`：軌道整圈（`LumiBorderBrush`）、進度弧從 12 點鐘順時針、圓頭線端。`Ratio` 變化時用 `DoubleTransition`（200ms、ease-out）補間 `DisplayRatio`。
  2. 規則：`Ratio == 0` 不畫進度弧；`0 < Ratio < 1%` 畫最小 2°；`Empty` 只畫虛線軌道（線寬 2、dash 2.5／3.2、色 `LumiTextMutedBrush` 降透明度）。
  3. 顏色依 `Level`：Normal＝`LumiAccentBrush`、Warning＝`LumiWarningIconBrush`、Danger＝`LumiDangerBrush`（用 DynamicResource 取，主題與深淺色切換時跟著換）。
  4. `UsageLevel.From(hasData, ratio)`：沒資料 → `Empty`；< 0.60 → `Normal`；0.60 ≤ ratio < 0.85 → `Warning`；≥ 0.85 → `Danger`。
- **測試**：`UsageLevel` 在 0、0.59、0.60、0.84、0.85、1、無資料；Headless：四種等級各 render 一次不丟例外、`Ratio` 夾在 0–1、Diameter 20 時量測大小為 20×20。
- **不要做**：不做按鈕與面板（21E）。

## 包 21C：齒輪、「⋯」對話動作、匯出（Wave 1）

- **先讀**：`Views/CustomTitleBar.axaml(.cs)`、`TitleBarLayout.cs`、`Views/ChatToolbar.axaml(.cs)`、`ViewModels/HistoryViewModel.cs`（重命名規則第 279–345 行、刪除第 350–390 行）、`Views/ConfirmDialog.axaml`＋`ConfirmService.cs`、`ChatToolbarTests.cs`、`ResponsiveWorkbenchTests.cs`、`tests/Lumi.App.Tests/Headless` 裡標題列相關測試。
- **可修改／新增**：`CustomTitleBar.axaml(.cs)`、`ChatToolbar.axaml(.cs)`、`WorkbenchViewModel.ConversationActions.cs`（新）、`src/Lumi.Core/Chat/ConversationMarkdownExporter.cs`（新）、`Views/RenameDialog.axaml(.cs)`（新，仿 `ConfirmDialog`）、上列測試檔。
- **步驟**：
  1. **齒輪**：`CustomTitleBar` 在小精靈按鈕右邊加 32×32 圖示按鈕（無框、透明底、`LumiIconSettings`、hover `LumiSurfaceAltBrush`），Command 綁 `OpenSettingsCommand`（已存在）、Tooltip 與 `AutomationProperties.Name` ＝ `Strings.Workbench_Settings`。與小精靈間距 8；仍在分隔線與系統按鈕左邊。設定視窗共用此標題列（`ShowSpriteButton=False`、`DataContext=null`）：齒輪要一併隱藏（新增 `ShowSettingsButton` 或併入同一開關）。回應中齒輪**可用**。
  2. **「⋯」選單**：移除「設定」「管理連線與模型」兩項（管理仍在模型選單底部），改為「重新命名對話」「匯出為 Markdown」「刪除這則對話」（Header 文字用 `LumiDangerBrush`）。`AutomationProperties.Name` 改 `對話選項`。回應中：重新命名可用、匯出與刪除停用；空對話（沒有訊息）三項全停用。
  3. **重新命名**：開 `RenameDialog`（TextBox 預填目前標題、儲存／取消、Enter 儲存、Esc 取消）；標題的清理與長度上限重用 `HistoryViewModel` 的同一份規則（抽成共用靜態函式，不要複製）；成功後更新工作台標題與歷史清單。
  4. **刪除**：重用 `ConversationDeletionService` 與 `HistoryViewModel` 的確認文字與「刪除後切到下一則」流程（`ConversationDeletedAsync`）；不得另寫一套。
  5. **匯出**：`ConversationMarkdownExporter.Export(Conversation, IReadOnlyList<ChatMessage>)` 純函式 → Markdown：`# 標題`、匯出時間；使用者訊息 `## 你`、助理訊息 `## Lumi`，文字區塊原樣；工具呼叫一行引用 `> 工具：{DisplayName} {Target}（成功／失敗）`；`ThinkingBlock`／`RedactedThinkingBlock`／`ProviderBlock` 略過；圖片 `[圖片：{名稱}]`；`ContextBlock` `[附檔：{名稱}]`。檔名預設「標題.md」（去除檔名不允許的字元）。流程：`IFilePickerService.SaveFileAsync` → 寫檔 → 成功／失敗訊息（用既有的 Notice／Error 呈現方式，先看 `WorkbenchViewModel` 現有通知機制）。
- **測試**：標題列在 Windows／macOS 配置下齒輪都在系統按鈕左邊、設定視窗沒有齒輪；「⋯」三項的啟用／停用條件（回應中、空對話）；匯出器各區塊（含空訊息、只有工具）；重新命名空白、過長、取消；刪除流程確認與取消。更新既有 `ChatToolbarTests`／`ResponsiveWorkbenchTests` 裡對 `SettingsMenuItem`、`ManageModelsMenuItem` 的斷言。
- **不要做**：不動模型膠囊、不加用量按鈕（21E）、不改 `HistoryDrawer` 的內聯重新命名。

## 包 21D：首頁資料（Wave 1）

- **先讀**：`ViewModels/HomeViewModel.cs`、`HomeViewModelTests.cs`、`MemoryViewModel.cs`（記憶筆數與總字數怎麼算、上限來自 `MemoryOptions.MaxItems／MaxTotalLength`）、`ToolsViewModel.cs`（`Groups`、`mcp:` 群組前綴）、`McpServersViewModel.cs`、`Lumi.Core/Mcp/IMcpToolSource.cs`（`GetStatuses`、`RestartAsync`）、`RemindersViewModel.cs`、`Lumi.Core/Reminders/ScheduledTypes.cs`（`ScheduledKind`）、`ConnectionStatusCache.cs`、`Lumi.Core/Models/IConnectionTester.cs`、`SpeechSettings.cs`。
- **可修改**：`HomeViewModel.cs` 與其測試、`tests/Lumi.App.Tests/TestSupport` 的 `SettingsHost`（調整建構）。**不碰** `HomePanel.axaml`、`SettingsWindow.axaml`。
- **新的 `HomeViewModel` 公開成員**（View 只綁這些）：
  - `PendingItems`（`ObservableCollection<HomePendingItem>`：`Id`、`IconKey`、`Title`、`Description`、`ActionText`、`IsPrimary`、`ActionCommand`）、`PendingTitle`（`待處理 · N`）、`HasPending`、`NoPendingText`。
  - `ModelCard`：顯示名稱、`供應商 · model ID`、狀態文字與是否需注意、「工作台與小精靈共用這個模型」的可見性（**先查實際行為**：`AppSettings` 有 `WorkbenchModelId` 與 `SpriteModelId` 兩個欄位，兩者不同時不顯示這句，不要顯示不實的說明）。
  - `MemoryCard`：開／關文字（**純文字，沒有開關控制項**）、筆數與上限、字數與上限、兩個進度值（0–1）。
  - `UsageCard`：本月總 tokens（輸入＋輸出）、今天總 tokens、本月輸入、本月輸出（用 `WorkbenchViewModel.FormatTokens`），載入中與失敗狀態；資料來自 `IConversationStore.GetTokenTotalsAsync`（本月 1 日 00:00 與今天 00:00 取**本地時間**再轉 UTC）。
  - `ToolsCard`：內建工具「已啟用／總數」（群組不是 `mcp:` 開頭的項目）、MCP 已連線／總數、MCP 伺服器列（名稱、狀態點、`{n} 個工具`／未連線、失敗時 `重新連線` 命令 → `IMcpToolSource.RestartAsync`；超過 4 個只列前 4 個並顯示「還有 N 個」）。
  - `RemindersCard`：待觸發提醒數、執行中計時器數（`ScheduledKind` 區分；計時器不算進「待觸發提醒」）、下一個提醒（標題＋時間，沒有時為提示句）。
  - `RecentActions`（`ObservableCollection<HomeActionItem>`：相對時間文字、狀態（成功／失敗／已拒絕）、`DisplayName`、`Target`、`Summary`、核准徽章文字）；來自 `ListRecentToolActionsAsync(5)`。核准徽章：`AllowedOnce／AllowedForConversation／AllowedAlways` →「已允許」，`Denied` →「你已拒絕」，`NotRequired` → 不顯示。失敗（`IsError`）狀態文字用 `Summary`。
  - `AttentionConnections`、`AttentionSpeech`（bool，給側欄橘點）。
  - `RefreshAsync()`（保留）、`LoadActivityAsync()`（載入用量與最近動作，頁面顯示時呼叫，不在 UI 執行緒上做查詢）。
- **待處理規則**（依序，同一類只出現一條）：沒有模型 → 前往設定（`Models`）；沒有金鑰 → 前往設定（`Connections`，帶 `FocusApiKey`）；連線失敗 → 重新測試（就地執行 `IConnectionTester`，結果寫進 `ConnectionStatusCache`）；連線狀態 Unknown → 測試連線（主要按鈕，就地執行）；語音端點為空 → 前往設定（`Speech`）。沒有任何項目時整段不顯示，改顯示一行 `NoPendingText`。
- **建構子**：新增注入 `IConversationStore`、`IMcpToolSource`、`IConnectionTester`（及測試連線需要的 secret 取得方式，照 `ConnectionsViewModel` 既有做法）、`TimeProvider`；`Attach` 簽章視需要擴充，一併改 `SettingsViewModel` 的呼叫與 DI 註冊。
- **測試**：改寫 `HomeViewModelTests`（原 6 卡斷言換成上述各卡）；待處理各條件與順序；就地測試連線成功／失敗後清單更新；用量卡的本月／今天邊界（用 `TimeProvider` 假時鐘）；工具卡內建與 MCP 的拆分；提醒卡計時器與提醒分開計數；最近動作的徽章對應。

## 包 21E：用量按鈕、面板與模型選單精簡（Wave 2）

- **先讀**：[ui-usage-and-topbar.md](ui-usage-and-topbar.md) 第 2、3、4、5 節；21A、21B 的成果；`WorkbenchViewModel.cs`（`LastUsage`、`Turns`、`SyncSessionUsage`、載入與串流的 `UsageEvent` 處理約 480、794–809 行）、`ModelSwitcherViewModel.cs`（`UpdateSession` 等）、`ModelPicker.axaml`、`ModelSwitcherViewModelTests.cs`、`ChatToolbar.axaml`。
- **可修改／新增**：`ConversationUsageViewModel.cs`、`Views/UsageFlyout.axaml(.cs)`（新）、`WorkbenchViewModel.Usage.cs`（新）、`ChatToolbar.axaml`（只加用量按鈕，位置在膠囊與「⋯」之間）、`ModelPicker.axaml`、`ModelSwitcherViewModel.cs` 及測試。
- **步驟**：
  1. **累計**：`ConversationUsageTally`（純類別，好測）：`Committed`＝本對話已存訊息的 usage 加總（載入時對所有助理訊息加總）、`InFlight`＝進行中請求最新的 `UsageEvent`（它是該次請求的累計值，不要重複相加）；顯示值＝`Committed + InFlight`；收到 `Completed` 把該訊息 usage 併入 `Committed`、清掉 `InFlight`；失敗／中止以資料庫實際存下來的為準。`ContextUsed`＝最近一次請求（輸入含快取＋輸出，沿用現有算法）。`TurnCount`＝使用者送出次數（沿用 `Turns`）。切換／新對話整組換掉。
  2. **`ConversationUsageViewModel`**：依 ui-usage-and-topbar.md 第 4 節屬性，加上 `Level`（`UsageLevel.From`）、`Ratio`、各顯示字串（格式照 3.2：`FormatTokens`，<1% 顯示「<1%」，「還可以再用約」下限 0）、快取命中率＝累計快取讀取 ÷ 累計輸入（含快取）；累計快取 = 0 時 `ShowCacheRing=false`（見差異表）；`ContextWindow` 為 0 或未知時：不顯示百分比與圓環進度，圓環用 `Empty`，面板只顯示累計數字。
  3. **按鈕**：32×32、`Classes="tertiary toolbar iconOnly"` 的外觀，內含 `UsageRing`（直徑 20、線寬 2.5）；面板開啟中底色 `LumiAccentSoftBrush`、框 `LumiAccentBrush`；`Danger` 時右上 10px 紅點（`LumiDangerDotBrush`，2px `LumiSurfaceBrush` 外框）；Tooltip 與 `AutomationProperties.Name` 照 2.1；`Button.Flyout`（`Placement="BottomEdgeAlignedRight"`、`VerticalOffset=6`）；Tab 可聚焦，Enter／Space 開、Esc 關並把焦點還給按鈕。回應中可用（唯讀）。
  4. **面板**（`UsageFlyout`，寬 320）：照第 3 節——標題列、上下文區（大圓環直徑 104、線寬 10＋三行文字）、三個小方塊（輸入／輸出／回合）、連線列（Ok／Unknown／Failed；Failed 的「重試」呼叫現有的連線測試流程，找 `ModelSwitcherViewModel`／`ConnectionStatusCache` 既有接口，沒有就回報）、費用說明。`Danger` 時插入快滿提醒（「開新對話」回應中停用、「繼續這則」只關閉面板）。顏色只用既有 token（見差異表）。
  5. **模型選單精簡**：移除 `ModelPicker` 的 `SessionSection`、`ModelSwitcherViewModel` 的 `UpdateSession／HasSession／UsageText／ContextUsed／ContextWindow／ContextLabel`，以及 `WorkbenchViewModel` 裡只為它存在的 `SyncSessionUsage`、`StatusText`（若仍有其他引用就保留）。寬度維持 320。模型選單的搜尋、分組清單、「管理連線與模型…」不變；膠囊 Tooltip 維持現狀。
- **測試**：累計（載入加總、串流中不重複計、完成後併入、切換對話歸零、失敗中止）；`Level` 與顯示字串在 0%、<1%、59%、60%、84%、85%、100%；快取命中率三種供應商（Anthropic 的 `input + creation + read`、OpenAI 相容與 DeepSeek 的 `prompt − cached`）；累計快取 0 時不顯示迷你圓環；`ContextWindow = 0`；Headless：Tab 到按鈕、Enter 開面板、Esc 關閉且焦點回到按鈕；快滿提醒出現與按鈕行為；更新 `ModelSwitcherViewModelTests`（刪除用量相關 14 個中不再適用者）。

## 包 21F：設定首頁畫面（Wave 2）

- **先讀**：設計稿 [方案 C · 整合版@1x.png](方案%20C%20·%20整合版@1x.png)（只做 1A 範圍）、`Views/HomePanel.axaml(.cs)`、[RWD原則.md](ui-guide/RWD原則.md)、[ui-guide.html](ui-guide/ui-guide.html)、`Views/SettingsWindow.axaml`（側欄）、`Controls/FractionWidthLimiter.cs`、`HomePanelTests.cs`、21D 的 `HomeViewModel`。
- **可修改**：`HomePanel.axaml(.cs)`、`SettingsWindow.axaml` 側欄的橘點（只動 `CategoryConnections` 與 `CategorySpeech` 兩個項目，其他不碰）、`HomePanelTests.cs`。
- **版面**（由上而下，內容最大寬度約 920）：
  1. 頁首：小字「設定首頁」（`AccentText`）、標題「Lumi 設定總覽」（20／700，`section-title`）。沒有健康度膠囊、沒有搜尋框。
  2. 待處理（`HasPending` 才顯示）：標題「待處理 · N」（`WarningText`）；一個 `Warning` 淡底圓角容器，列內：圖示方塊、標題（14／600）、說明（`muted`）、右側按鈕（`IsPrimary` 用 `primary`，否則 `secondary`）；列與列之間 1px 分隔。沒有待處理時顯示一行 `NoPendingText`（`muted`，前面加 `Check` 圖示）。
  3. 重點設定：寬視窗第一排三卡（目前模型、記憶、本月用量）、第二排兩卡（工具、提醒排程，寬度約 1.1 : 1）；中等視窗 2 欄；窄視窗 1 欄。斷點用既有 `Window.compact`／`Window.narrow` class，以樣式 Setter 改 `Grid.Column／Grid.ColumnSpan／Grid.Row`（不要用 code-behind 量寬度）；設定視窗最小寬 720（扣側欄後內容約 480），這個寬度下必須是 1 欄且沒有水平捲動。
     - **目前模型**：標題「目前模型」＋右上「切換 ›」；中間一個 `LumiCodeBackground` 圓角塊，內 `LumiMonoFont` 顯示 model ID，下方「供應商 · 格式」與狀態文字（需注意時用 Warning 色文字加圖示）。
     - **記憶**：標題＋「管理 ›」；「已開啟／已關閉」文字（**不是開關**）；兩條進度（`ProgressBar` 或 Border 寬度，用既有色 `LumiAccentBrush`）：記住的事、已用字數。
     - **本月用量**：標題；大字本月 tokens（20／700）加 `tokens` 單位；下方「今天 {0}」；底部兩個 `SurfaceAlt` 小標籤「輸入 {0}」「輸出 {0}」。載入中顯示 `—`，失敗顯示 muted 說明。沒有連結。
     - **工具**：標題＋「管理 ›」；左右兩區「內建工具 45 ／ 45 已啟用」「MCP 伺服器 2 ／ 3 已連線」；下面 MCP 列（狀態點、等寬名稱、「N 個工具」或「未連線」＋「重新連線」按鈕）；沒有 MCP 伺服器時整段隱藏。底部「寫入與網路先詢問，刪除類每次詢問」這句**先對照工具權限的實際預設**再決定是否保留，不符就刪掉。
     - **提醒排程**：標題＋「管理 ›」；兩個數字方塊（待觸發提醒、執行中計時器）；下面一行「下一個提醒：…」。沒有 7 天圖。
  4. 最近的動作：標題「最近的動作」（沒有「查看全部紀錄」）；列表容器同待處理的圓角邊框；每列：相對時間（`muted`）、狀態圖示（`Check`／`X`／`Ban`，不只靠顏色）、動作名稱＋目標（等寬小標籤）＋摘要、右側核准徽章（`Border.tag` 樣式）。空狀態一行 muted 文字。
- 卡片本身**不是**整張可點的 Button（卡內有各自的連結）：「切換 ›」「管理 ›」用 `tertiary` 文字按鈕，Command 用 `GoToCommand`（`Models`／`Memory`／`Tools`／`Reminders`）。每個按鈕與可互動控制項都要有 `AutomationProperties.Name`；各區給 `x:Name`（`HomePendingList`、`HomeModelCard`、`HomeMemoryCard`、`HomeUsageCard`、`HomeToolsCard`、`HomeRemindersCard`、`HomeRecentActions`）。
- **側欄橘點**：`CategoryConnections`、`CategorySpeech` 各加一個 8px `LumiWarningDotBrush` 圓點，`IsVisible` 綁 `Home.AttentionConnections`／`Home.AttentionSpeech`；`AutomationProperties.HelpText` 寫「需要處理」（不只靠顏色）。
- `HomePanel.axaml.cs`：頁面顯示（`AttachedToVisualTree`）時呼叫 `LoadActivityAsync()`。
- **測試**：Headless——各區塊 `x:Name` 存在；沒有待處理時改顯示 `NoPendingText`；待處理按鈕有 Command 與無障礙名稱；記憶卡**沒有** `ToggleSwitch`／`CheckBox`；720 寬時 1 欄；1040 寬時第一排三卡同一橫排；側欄橘點隨 `Attention*` 顯示；淺色與深色各 render 一次。重寫原本 6 張卡片的 5 個測試。

---

## 整合（協調者）

- DI：`AppServiceCollectionExtensions` 補 `HomeViewModel` 新依賴、`IFilePickerService` 真實作；`SettingsHost`／`WorkbenchHost` 測試輔助調整。
- 文件：`docs/ui-spec.md`（2.1 頂端區、2.3 本對話、第 3 節訊息下方 token〔保留進階開關的說明〕、設定首頁）、`plan/manual-verification.md`（加入以下手動項目）。
- 手動驗證（mac 與 Windows 各一次，Windows 無法在本機測的項目標明）：Windows 11 標題列齒輪與關閉鈕間有分隔線且可拖曳；用量圖示在 0%／59%／60%／84%／85%／100% 與新對話虛線；五個色票主題與深淺色下圓環顏色正確、文字對比 ≥ 4.5:1；1920×1080 在 100%／125% 縮放下小圓環清晰；鍵盤 Tab／Enter／Space／Esc；DeepSeek、Anthropic、OpenAI 相容、Ollama 的面板數字與供應商回傳一致；匯出的 `.md` 開啟檢查；設定首頁在 720／1040／1400 寬度。
- 合併順序：Wave 1 四包 → 跑全測試 → Wave 2 兩包 → 整合。每包合併後 `dotnet build Lumi.sln` 零警告、`dotnet test` 全過再進下一步。

## 待確認的小事（不阻擋開工，實作者遇到時照下面預設）

1. 工作台標題列右上齒輪在 narrow 寬度：維持圖示（本來就只有圖示）。
2. 首頁「語音辨識尚未設定」是否對所有人都算「待處理」：先照稿放，之後可改成只在使用者曾開過語音時才提示。
3. 最近的動作顯示 5 筆。
