# M22 同事回饋修正與圖片生成

分支 `m22-feedback-fixes`。來源：2026-10-27 同事試用回饋（14 題）與 log（[issue #1](https://github.com/NEILLOG/DesktopAgent/issues/1)）。

使用者 2026-10-27／28 決定：

| 題號 | 項目 | 決定 |
|---|---|---|
| 5 | 對話時 `FileNotFoundException: System.Net.ServerSentEvents` | 修發佈方式＋看得懂的錯誤訊息（根因無法確認，但發佈方式本來就該改） |
| 6、11 | 首頁一直提示未設定；拿掉 Claude 預設 | 做；「沒有金鑰」對所有格式都維持提示（公司 Gateway 也要金鑰） |
| 3 | 小精靈輸入列陰影斷層（Windows） | 做 |
| 8、9 | 模型與連線合併成一頁；詳細區不要內層捲軸 | 一起做，版面照 22D 草圖（已確認） |
| 10 | 小精靈「狀態預覽」 | 預覽區固定，只捲右邊 |
| 12 | 訊息文字無法圈選 | 只做第一階段：段落內可選取 |
| 2 | gpt-image-2 | 假設公司 AI Gateway 有 `/v1/images/generations` 與 `/v1/images/edits`，做「產生圖片」與「編輯圖片」兩個工具；點圖用系統看圖程式開啟 |
| 4 | 小精靈獨立對話 | 小精靈**固定一則自己的對話**（不跟著工作台切換）；所有對話都做**上下文壓縮**（22H–22J） |
| 1 | 新注音組字反白 | **pass**（Avalonia 只給組字文字與游標位置，拿不到選字段落） |
| 7 | Playwright 手動腳本 | **pass** |
| 13、14 | 排程跑 prompt、Skill | **稍後討論**，不在本計畫 |

## 根因摘要（實作者先讀）

- **5**：Windows 發佈設定 `IncludeAllContentForSelfExtract=true`（[win-x64.pubxml](../src/Lumi.App/Properties/PublishProfiles/win-x64.pubxml)）讓所有受控 DLL 啟動時解壓到 `%TEMP%\.net\Lumi.App\<雜湊>\`。`System.Net.ServerSentEvents` 要到第一次 OpenAI 相容對話才載入；推測公司的防毒或暫存清理在執行中刪掉了它。這個設定原本只是為了 Playwright（`.playwright`）與 `appsettings.json`。
- **6、11**：[JsonModelCatalog.CreateInitialSettings](../src/Lumi.Infrastructure/Settings/JsonModelCatalog.cs) 第一次啟動就建「Anthropic 連線（`dev-anthropic`）＋ Claude Opus 5.5」並設成預設模型。同事之後新增公司連線與模型，但預設模型仍是沒金鑰的 Claude，首頁就一直顯示「AI 連線缺少金鑰」。另外：
  - `ConnectionStatusCache` 只在記憶體，每次重開都回到「尚未測試」。
- **3**：`LumiSpriteShadow` 是 `0 8 24`，往下約延伸 32px，但 `SpriteShellOptions.BottomMargin` 只有 12，陰影被透明視窗邊緣切掉。
- **9**：[ConnectionsPanel.axaml](../src/Lumi.App/Views/ConnectionsPanel.axaml) 外面已有整頁的 `ScrollViewer`，詳細卡片裡又包了一層（`Grid RowDefinitions="Auto,*,Auto"` 中間那列），視窗不高時可填寫區很小。`ModelsPanel.axaml` 同樣結構。
- **10**：小精靈頁整頁包在一個 `ScrollViewer`（[SettingsWindow.axaml](../src/Lumi.App/Views/SettingsWindow.axaml) `ShowSprite` 那段），預覽區在左欄，一捲就跑掉。
- **12**：助理回答用 `MarkdownTextBlock : EmojiTextBlock : TextBlock`，使用者訊息與思考過程用 `EmojiTextBlock`，都不能選取。
- **4**：
  - `IChatSession` 是單一實例，工作台與小精靈共用「目前的對話」（`CoreServiceCollectionExtensions`）。
  - `ConversationService.RunTurnAsync` 每次都把整則歷史（`BuildRequestMessages`）送出，沒有任何上下文管理：超過模型上限後每次送出都會失敗。

## 不做的項目（實作者不要補）

跨段落／跨訊息選取、圖片遮罩（mask）與圖片生成在小精靈模式、內建的放大檢視、排程跑 prompt、Skill、Playwright 腳本、新注音反白、直接連 Azure（`api-key` 標頭、deployment、api-version）、壓縮用另一個（較便宜的）模型、每天一則小精靈對話。

## 工作拆分

| 包 | 內容 | Wave | 負責 |
|---|---|---|---|
| Phase 0 | 字串、`ImageGenerationSettings`、`IImageGenerator` 介面、`ToolResult.ImageFiles`、壓縮的 schema／介面／事件、計畫文件 | — | 協調者 |
| 22A | 發佈方式（Playwright 自己解壓）、`appsettings` 改內建預設、缺檔錯誤訊息 | 1 | 子代理 |
| 22B | 拿掉 Claude 預設與舊預設清理、連線狀態存檔、首頁提示規則 | 1 | 子代理 |
| 22C | 小精靈陰影、小精靈頁預覽固定 | 1 | 子代理 |
| 22D | AI 連線頁合併（連線＋模型一個清單）、詳細區改整頁捲動 | 1（草圖確認後） | 子代理 |
| 22E | 訊息文字可選取 | 1 | 子代理 |
| 22F | 圖片：Providers 的 Images API 用戶端（generations＋edits）、`generate_image`／`edit_image` 工具、工具卡片顯示圖片 | 1 | 子代理 |
| 22G | 圖片生成的設定 UI（放進合併後的 AI 連線頁） | 2（22D、22F 合併後） | 子代理 |
| 22H | 上下文壓縮核心：清掉舊工具結果、LLM 摘要、觸發時機、超過上限自動整理重試 | 1 | 子代理 |
| 22I | 壓縮的畫面：「正在整理對話…」、分隔線、用量面板「整理並繼續」 | 2（22H 合併後） | 子代理 |
| 22J | 小精靈專屬對話 | 2（22A、22H 合併後；都動 `ChatSession`） | 子代理 |
| 整合 | DI、文件（ui-spec、manual-verification）、發佈實測（macOS）、整體測試 | — | 協調者 |

通則同 M21：照 [[subagent-worktrees-manual]] 先 commit Phase 0，再 `git worktree add -b w22<包> ../DesktopAgent-wt/w22<包> HEAD`；子代理不 commit、不改 `plan/`、不新增字串（缺就停下來回報）、只用既有 token 與樣式 class、建置零警告、遵守間距尺度（0/4/8/12/16/24/32）。遇到計畫沒寫到或與程式現況不符就停下來回報，不要猜。

回報格式：改了哪些檔案、新增測試數量與結果、和計畫不同的地方與原因、沒做完的項目。

---

## Phase 0（協調者）

1. **字串**（`Strings.resx`＋`Strings.cs`）：
   - 缺檔錯誤：`Chat_ErrorAppFilesMissing`「Lumi 的程式檔案不完整，請重新啟動 Lumi；仍然發生請重新下載安裝。」
   - 首頁：預設模型無法使用的待處理（標題／說明）。
   - 合併頁（22D）：清單分組標題、「新增模型」（連線底下）、「新增連線」、頁首說明更新、預設模型區標題。
   - 圖片生成：兩個工具的顯示名稱與 Summary、未設定時的錯誤、找不到來源圖片的錯誤、設定區（標題、說明、連線、模型 ID、預設尺寸、尺寸選項三個）、工具卡片「開啟圖片」「開啟資料夾」。
   - 壓縮：「正在整理對話…」、分隔線「以上內容已整理成摘要」（含「查看摘要」）、用量面板「整理並繼續」、整理失敗的訊息、超過上限自動整理後重試的說明。
   - 小精靈對話：對話標題「與小精靈的對話」。
2. **Core 型別**：
   - `ImageGenerationSettings(string? ConnectionId, string? ModelId, string Size = "1024x1024")`，掛在 `AppSettings` 的 init 屬性（舊檔沒有時是 `Default`），`IsConfigured` = 兩個 ID 都有。`IModelCatalog.SetImageGenerationAsync`（介面＋`JsonModelCatalog` 實作與讀寫，照 `Notes` 的做法；格式錯誤當作預設）。
   - `IImageGenerator`（`Lumi.Core/Images/`）：
     - `Task<GeneratedImage> GenerateAsync(ImageGenerationRequest request, CancellationToken ct)`；
     - `Task<GeneratedImage> EditAsync(ImageEditRequest request, CancellationToken ct)`；
     - `ImageGenerationRequest(ConnectionSettings Connection, string ModelId, string Prompt, string Size)`；
     - `ImageEditRequest(ConnectionSettings Connection, string ModelId, string Prompt, string Size, IReadOnlyList<SourceImage> Images)`、`SourceImage(byte[] Data, string MediaType, string FileName)`；
     - `GeneratedImage(byte[] Data, string MediaType, string? RevisedPrompt)`；
     - 失敗丟 `ImageGenerationException(ChatFailureKind Kind, string Message)`。
   - `ToolResult.ImageFiles`（`IReadOnlyList<string>`，init，預設空）：**只顯示、不送給模型**的圖片檔路徑。`ToolResultBlock` 加同名選用欄位（JSON 沒有時為空；舊資料相容，不改 schema）。
   - `ConnectionStatus` 存檔用欄位見 22B 第 3 點：Phase 0 先加 `ConnectionSettings.LastTest`（`ConnectionTestRecord(bool Success, DateTimeOffset At)?`，init，舊檔沒有時 null）與讀寫。
   - **壓縮**（22H 用）：
     - schema migration 11：`conversation_compactions(id TEXT PK, conversation_id TEXT FK ON DELETE CASCADE, kind TEXT NOT NULL /* 'tool_results' | 'summary' */, upto_seq INTEGER NOT NULL, summary TEXT, model_id TEXT, created_at TEXT NOT NULL)`，索引 `(conversation_id, created_at)`。
     - `IConversationStore`：`AddCompactionAsync`、`GetCompactionStateAsync(conversationId)` → `CompactionState(int? ToolResultsUptoSeq, int? SummaryUptoSeq, string? Summary)`（各取最新一筆）。介面加預設實作（回空狀態），測試用的 in-memory store 補實作。
     - `ChatMessage` 需要能取得 `Seq`：若現在沒有，就由 store 在 `GetMessagesAsync` 時填進新的選用屬性（不改既有建構式參數）。
     - `ChatEvent`：`CompactionStarted(CompactionKind Kind)`、`CompactionCompleted(CompactionKind Kind, int UptoSeq)`。
     - `ChatFailureKind.ContextTooLong`。
     - `IChatSession.CompactAsync(CancellationToken)`：手動整理（22I 的按鈕用）。

3. 22D 草圖已確認（2026-10-28）。

---

## 包 22A：發佈方式與缺檔錯誤（Wave 1）

1. **發佈設定**（`win-x64.pubxml`、`osx-arm64.pubxml`）：拿掉 `IncludeAllContentForSelfExtract`，保留 `IncludeNativeLibrariesForSelfExtract`（原生函式庫都在啟動時就載入）。受控 DLL 從此直接從執行檔載入，不再解壓到暫存資料夾。
2. **Playwright driver 自己解壓**：
   - 建置時把 `.playwright` 資料夾壓成 zip，當作 `Lumi.Browser`（或 `Lumi.App`）的 `EmbeddedResource`。用 MSBuild 的 `ZipDirectory` 加一個在複製 Playwright 內容之後執行的 target。
   - 發佈時不再把 `.playwright` 放進輸出（避免單檔旁邊多一個資料夾）。一般 `dotnet build`／`dotnet run` 維持現狀，driver 就在輸出資料夾。
   - `PlaywrightDriverLocator.Prepare()`：`BaseDirectory` 有 node 時照舊。否則把內嵌 zip 解到 `%LOCALAPPDATA%\Lumi\playwright\<內嵌 zip 的 SHA-256 前 16 碼>\`（macOS：`~/Library/Application Support/Lumi/playwright/...`）。
     - 先解到同層暫存資料夾，完成後寫一個 `.complete` 標記檔再改名，避免解到一半。
     - 已有標記就直接用。
     - macOS 補 node 執行權限（沿用 `EnsureExecutable`）。
     - 設 `PLAYWRIGHT_DRIVER_SEARCH_PATH`。
     - 舊的雜湊資料夾留著不刪。
   - 原本掃描 `.net/<程式名稱>/*` 的邏輯可以刪；`FindExtractionRoot` 的測試改成新的解壓邏輯測試。
3. **`appsettings.json`**：單檔旁邊可能沒有它。把內容（Logging 等級、`Lumi:Developer` 預設值）改成程式內建的預設（在載入 json 之前 `AddInMemoryCollection`），`appsettings.json` 仍是 optional 覆寫。log 等級必須和現在一樣（`Microsoft` 是 Warning）。
4. **缺檔錯誤訊息**：
   - `ChatSession` 的未預期錯誤：例外鏈裡有 `FileNotFoundException` 或 `FileLoadException` 時，改成回報一個 `ChatFailure`（新 `ChatFailureKind.AppFilesMissing`，`Retryable: false`，訊息用 `Chat_ErrorAppFilesMissing`），不要只顯示一般的「未預期的錯誤」。
   - `ConnectionStatusCache.StatusAfterFailure` 對它回 null（不是連線問題）。
   - 工作台與小精靈都要能顯示這個訊息。
5. **測試**：
   - Locator：已解壓就重用、解到一半（沒標記）會重解、雜湊不同各自一個資料夾。
   - `ChatSession`：`FileNotFoundException` 轉成 `AppFilesMissing`，內層例外也要認得。
   - `StatusAfterFailure`。
6. **驗收**（協調者在 macOS 實測）：`scripts/publish-release.sh --only mac` 產出的 app 在乾淨使用者資料夾能啟動、能對話、能開瀏覽器工具；`~/.net`（或 `$TMPDIR/.net`）裡不再出現受控 DLL。Windows 交給 CI 發佈後手動驗證（列入 manual-verification）。

## 包 22B：拿掉 Claude 預設與首頁提示（Wave 1）

1. **第一次啟動**：`CreateInitialSettings()` 回 `AppSettings.Empty`（沒有連線、沒有模型、預設模型 null），其餘欄位照預設。首頁會出現「沒有模型」的待處理，引導去設定。
2. **舊預設清理**（載入設定檔時一次性、可重複執行無副作用）：連線 `Id == "dev-anthropic"` 而且金鑰不存在時，刪掉這個連線與它底下的模型；預設模型照 `Normalize` 改成剩下的第一個。
   - 有金鑰就不動（開發者自己在用）。
   - 金鑰查詢失敗（`SecretStoreException`）也不動。
   - 記一行 Information log。
3. **連線狀態存檔**：
   - `ConnectionStatusCache.Set`（測試連線）與 `Mark`（對話成功／失敗）之後，把結果寫進 `ConnectionSettings.LastTest`（經 `IModelCatalog`，不觸發「連線設定改了」而清掉狀態）。
   - 啟動時用 `LastTest` 初始化快取（模型清單仍是空的，要列模型清單時照舊重新測試）。
   - 連線的 Base URL／格式／金鑰改了，`LastTest` 清成 null（現在 `Remove` 的時機）。
4. **首頁待處理規則**（`HomeViewModel.ApplyPending`）：
   - 「沒有金鑰」維持現有規則：所有格式都算待處理（使用者 2026-10-28：公司 AI Gateway 也需要金鑰）。
   - 「尚未測試」維持（現在重開不會再出現，因為有存檔）。
   - 預設模型找不到（設定檔裡指向已刪的模型）：照「沒有模型」處理。
5. **ModelsViewModel**：新增模型存檔時，如果目前的預設模型無法使用（找不到，或它的連線沒有金鑰），就把新模型設成預設，並用 `DefaultChangedNotice` 告知。
6. **測試**：
   - 第一次啟動是空的。
   - 清理：沒金鑰刪、有金鑰不刪、金鑰查詢失敗不刪、清完預設模型改成剩下的。
   - `LastTest` 讀寫、舊檔相容、改 URL 會清掉。
   - 首頁：重開後讀得到上次結果、不再顯示「尚未測試」。
   - 新增模型取代無法使用的預設。
   - 既有依賴 Claude 初始設定的測試要改。

## 包 22C：小精靈陰影與預覽固定（Wave 1）

1. **陰影**：`SpriteShellOptions.BottomMargin` 改成 32（陰影 y 位移 8＋模糊 24）。左右也檢查：輸入列寬 300、視窗寬由 `SpriteLayout` 算，左右各至少留 24；不夠就加寬視窗。氣泡（`SpriteBubbleView`）的陰影同樣檢查上方與左右。
   - 不改陰影本身（ui-guide 的 token）。
   - 同步檢查視窗位置的邊界計算（貼螢幕邊時，角色位置不能因為留白變大而跑掉）。
   - 更新 `SpriteLayout` 的測試數字。
2. **小精靈頁**：
   - 外層 `ScrollViewer` 只包右欄（設定內容）；左欄預覽區（`SpritePreviewArea`）固定在上方不捲。
   - 視窗 < 900（stacked，上下排）時：預覽區固定在上方，下方內容捲動。
   - 「狀態預覽」那排按鈕仍在右欄原位置。
   - Headless 測試：捲到底時預覽區仍在可視範圍內（寬版與 stacked 各一）。

## 包 22D：AI 連線頁合併與整頁捲動（Wave 1，草圖確認後）

**草圖**（取代現在的「連線／模型設定」分段切換）：

```
AI 連線                                                    [＋ 新增連線]
供應商與模型
新增自己的 API 金鑰，並選擇每次對話預設使用的模型。

預設模型  [ GPT-6（公司 AI Gateway） ▾ ]       費用說明……

┌ 清單 260 ─────────────────┐  ┌ 詳細（隨選取切換）─────────────────────┐
│ 公司 AI Gateway   ● 已連線 │  │  選了連線 → 連線表單（名稱、格式、     │
│   ◆ GPT-6        預設      │  │            Base URL、金鑰、測試連線）   │
│   ◆ DeepSeek V4 Flash      │  │  選了模型 → 模型表單（連線、model ID、 │
│   ＋ 新增模型               │  │            名稱、上下文、輸出、思考…）  │
│                            │  │                                         │
│ 本機 Ollama       ○ 未測試 │  │  底部：刪除／取消／儲存（跟著內容走，  │
│   ◆ llama4                 │  │        整頁捲動，不再有內層捲軸）       │
│   ＋ 新增模型               │  │                                         │
└────────────────────────────┘  └─────────────────────────────────────────┘
```

1. **清單**：連線一列（ProviderMark、名稱、狀態點），底下縮排列出它的模型（名稱、「預設」標籤），每個連線最後一列是虛線的「＋ 新增模型」（預先選好這個連線）。
   - 頁首主要按鈕固定是「＋ 新增連線」。
   - 沒有任何連線時，清單是空狀態加說明。
2. **詳細**：
   - 依選取顯示現有的連線表單或模型表單（重用 `ConnectionsViewModel`／`ModelsViewModel` 與兩個 panel 的詳細區，不重寫表單邏輯）。
   - 切換時的「放棄變更」確認照舊。
   - 新增連線存檔後，自動選取它並提示「接著新增模型」（用既有的說明列樣式）。
3. **預設模型選單**移到頁首下方（原本在 ModelsPanel 上方）。
4. **整頁捲動（第 9 題）**：
   - 拿掉詳細卡片的內層 `ScrollViewer`，表單照內容長高，整頁一起捲。
   - 底部按鈕列跟著表單走，不固定在視窗底。
   - 清單欄超過高度時跟著整頁捲（不另外捲）。
5. **導覽相容**：`SettingsSection.Connections`／`Models` 兩個值都導到這一頁；帶 `ConnectionId`、`FocusApiKey` 的導覽要能選到該連線並聚焦金鑰。`SettingsViewModel.TabIndex`、分段切換相關的屬性與測試移除。
6. **斷點**：stacked（< 900）時清單在上、詳細在下；compact／narrow 照現有 master-detail 樣式。
7. **測試**：分組清單內容與順序、「＋ 新增模型」預選連線、選連線／選模型切換詳細、未存變更確認、導覽到連線並聚焦金鑰、預設模型選單、沒有內層捲動（詳細區高度等於內容高度）。舊的分段切換測試改寫或刪除。

## 包 22E：訊息文字可選取（Wave 1）

1. `EmojiTextBlock` 改成繼承 `SelectableTextBlock`（`StyleKeyOverride` 改成 `typeof(SelectableTextBlock)`），`MarkdownTextBlock` 跟著變。`EmojiText.Apply` 對 `SelectableTextBlock` 照常運作（它是 `TextBlock` 子類）。
2. 範圍：助理回答（Markdown 每個段落、清單項目、表格儲存格、程式碼區塊）、使用者訊息、思考過程。工具卡片與其他地方用到 `EmojiTextBlock` 的，選取不造成問題就一起開放，有問題（例如按鈕裡的文字）改用一般 `TextBlock` 或加 `IsHitTestVisible`／屬性關掉。
3. **連結**：`MarkdownTextBlock` 的連結點擊與手形游標要保留。按下後有拖曳就是選取，沒拖曳放開就是點連結。
4. **選取外觀**：選取底色用既有的 token（`LumiAccentSoftBrush` 或 Fluent 預設的 selection brush 對應到主題），深色模式也要看得清楚。
5. **複製**：`Ctrl/Cmd+C` 複製選取文字；右鍵選單「複製」（沿用既有字串 `Workbench_Copy` 之類，沒有就停下來回報）。
6. **捲動**：在訊息上拖曳選取時，對話清單不能被拖著捲（Avalonia 的 ScrollViewer 手勢要檢查）。
7. **測試**（headless）：訊息文字可選取（`SelectionStart/End` 變化）、`Ctrl+C` 進剪貼簿、點連結仍觸發 `LinkClicked`、拖曳不觸發連結。

## 包 22F：圖片生成與編輯（Wave 1）

1. **`OpenAIImageGenerator : IImageGenerator`**（`Lumi.Providers/OpenAICompatible/`）：
   - 產生：`POST {base}/images/generations`（`OpenAIUrls` 組網址），JSON body `{ model, prompt, size, n: 1 }`。
   - 編輯：`POST {base}/images/edits`，`multipart/form-data`：`model`、`prompt`、`size`、`n=1`，每張來源圖一個 `image[]` 檔案欄位（帶檔名與媒體類型）。
   - 有金鑰才送 `Authorization: Bearer`；`HttpClient` 用具名 client（照 `OpenAICompatibleChatProvider` 的註冊方式），逾時 180 秒。
   - 兩者回應格式相同：取 `data[0].b64_json`；沒有就取 `data[0].url` 再下載一次（同一個 client）。`revised_prompt` 有就帶回。
   - 媒體類型：看位元組開頭（PNG／JPEG／WebP），不認得就當 PNG。
   - 錯誤：HTTP 狀態碼對應 `ChatFailureKind`（沿用 `OpenAIHttpErrors` 的對應），丟 `ImageGenerationException`。
2. **`generate_image` 工具**（`Lumi.Tools/Images/GenerateImageTool.cs`）：
   - Descriptor：`ToolPermission.Network`、`ToolModes.Workbench`、逾時 240 秒、新群組 `ToolGroups.Images`（含工具清單頁的群組名稱與說明）。
   - Args：`prompt`（必填）、`size`（選填，`1024x1024`／`1536x1024`／`1024x1536`，其他值回錯誤）。
   - 說明要寫清楚：只有使用者要求畫圖或產生圖片時才用；prompt 用英文或使用者的語言都可以；會花費 API 費用。
   - 執行：
     - 設定沒填（`ImageGenerationSettings.IsConfigured` 為 false 或連線不存在）→ 回錯誤，告訴模型請使用者到「設定 › AI 連線 › 圖片生成」設定。
     - 呼叫 `IImageGenerator` 後存檔到「圖片／Lumi」（`Environment.SpecialFolder.MyPictures` 下的 `Lumi`，不存在就建立），檔名 `yyyyMMdd-HHmmss.png`（副檔名依媒體類型，同名加 `-2`）。
     - 結果文字：存檔路徑、尺寸、`revised_prompt`（有的話）。
     - `ImageFiles` = [路徑]；**不放進 `Images`**（不把圖片送回模型）。
   - Preview：顯示 prompt（截斷 120 字）與尺寸。
3. **`edit_image` 工具**（`Lumi.Tools/Images/EditImageTool.cs`）：
   - Descriptor 同 `generate_image`（同一群組）。
   - Args：`prompt`（必填，要怎麼改）、`images`（必填，1–4 個來源）、`size`（選填，同上）。
   - 每個來源可以是：
     - (a) 這則對話裡使用者附上的圖片，用訊息裡顯示的名稱（`ImageBlock.DisplayName`）指定，從附件儲存區讀取；
     - (b) 這則對話裡 Lumi 產生或編輯過的圖片（工具結果 `ImageFiles` 裡的完整路徑）；
     - (c) 使用者加入這則對話範圍內的圖片檔（照 `FileScope` 的檢查）。
   - 其他路徑一律拒絕（回錯誤說明可用的來源），不能讀任意檔案。
   - 只接受 PNG／JPEG／WebP，每張最多 20 MB。
   - 說明要寫清楚：使用者要求修改、合成、換背景等時才用；要指定哪張圖時用上面三種方式。
   - 執行與存檔同 `generate_image`（檔名加 `-edit`），`ImageFiles` = [新檔路徑]。
   - Preview：prompt 與來源圖片的名稱。
4. **工具卡片顯示圖片**（`ToolCardView`／`ToolCardViewModel`）：
   - 有 `ImageFiles` 時，在結果區顯示縮圖（最寬 320、等比，從檔案載入，載入失敗顯示檔名與「檔案已移動或刪除」）。
   - 點縮圖用系統預設的看圖程式開啟（Windows「相片」、macOS「預覽程式」，放大、平移、旋轉都在那裡做），Lumi 不做內建的放大檢視；「開啟資料夾」在檔案總管／Finder 中選取該檔（沿用既有的開檔服務）。
   - 歷史對話重新載入時也要顯示（從 `ToolResultBlock.ImageFiles`）。
5. **DI** 與工具註冊照現有工具的做法；架構測試（工具群組、Providers 不依賴 App）要過。
6. **測試**：
   - Generator：`b64_json`、`url` 兩種回應、HTTP 錯誤對應、沒金鑰不送標頭（假的 `HttpMessageHandler`）；edits 的 multipart 欄位（`model`、`prompt`、多個 `image[]`）。
   - 工具：未設定的錯誤、size 驗證、存檔路徑與重名、結果沒有 `Images` 但有 `ImageFiles`。
   - `edit_image` 來源解析：附件名稱、先前產生的路徑、範圍內的檔案；範圍外路徑被拒、不支援的格式、超過 4 張。
   - `ToolResultBlock.ImageFiles` 的 JSON 舊資料相容。
   - 卡片顯示縮圖與檔案不存在的狀態。

## 包 22G：圖片生成設定 UI（Wave 2）

1. 在合併後的 AI 連線頁，清單最下方加一個固定項目「圖片生成」（圖示＋目前設定摘要或「尚未設定」）。選取後詳細區顯示：
   - 連線（下拉，所有連線）；
   - 模型 ID（可編輯的下拉，選項來自該連線的可用模型清單，可重新整理，照 ModelsViewModel 的做法）；
   - 預設尺寸（三選一）；
   - 說明一行：「對話中要求畫圖時，Lumi 會用這個模型產生圖片，存到『圖片／Lumi』。會產生 API 費用。」
   - 儲存／取消。
2. 存檔走 `IModelCatalog.SetImageGenerationAsync`；連線被刪除時設定的 `ConnectionId` 清成 null（`Normalize`）。
3. **測試**：選取、存檔、連線刪除後變未設定、模型清單載入。

## 包 22H：上下文壓縮核心（Wave 1）

做法分兩層，先做不花錢的那層。整理只影響**送給模型的內容**；資料庫裡的原始訊息、畫面、歷史紀錄、匯出都不變。

1. **第一層：清掉舊的工具結果**（不呼叫模型）：
   - `seq <= ToolResultsUptoSeq` 的工具結果，送出時內容換成一行「（較早的工具結果已省略：{顯示名稱}{ · 目標}）」。
   - 圖片（`ImageBlock`）一併拿掉。`IsError` 與成對結構照舊。
   - 邊界取「倒數第 3 則使用者訊息之前」，最近的工具結果不動。
2. **第二層：LLM 摘要**：
   - 用同一個模型、不帶工具、不開 thinking，`max_tokens` 4000。
   - 送出邊界之前的內容（已套用第一層）＋一則固定的整理指示。指示要求依序列出：使用者的目標與要求、做過的決定與結論、重要事實與數字（原樣保留）、用過的檔案／網址／工具結果重點、尚未完成的事項；用對話的語言；不加入原文沒有的內容。
   - 邊界取「倒數第 2 則使用者訊息之前」，最近兩輪原樣保留。
   - 結果存成 `kind='summary'` 的一筆。摘要失敗就不存，回報失敗，對話照舊。
3. **套用**（`BuildRequestMessages` 之前，抽成可測的純函式 `CompactionApplier.Apply(history, state)`）：
   - `seq <= SummaryUptoSeq` 的訊息全部拿掉，改成第一則使用者訊息前面加一段 `<conversation_summary>…</conversation_summary>`，並說明「這是先前對話的摘要」。
   - 再套第一層。
   - **thinking 區塊**：被保留下來的助理訊息，除了「最後一則使用者訊息之後」（正在進行的工具迴圈）以外，一律拿掉 thinking／redacted_thinking 區塊。Anthropic 允許省略先前輪次的 thinking；整理後前綴變了，留著舊簽章反而可能 400。
4. **觸發**（`RunTurnAsync` 每次呼叫供應商之前）：
   - 估算這次請求的 token：最近一則助理訊息回報的用量（input＋cache＋output）＋之後新增訊息的字元數 ÷ 3。沒有用量資料時，全部用字元估算。
   - 上限 = 模型設定的 `ContextWindow`。超過 **80%**：先做第一層。重新估算仍超過 **70%**：再做第二層。
   - 整理前發 `CompactionStarted`、完成後發 `CompactionCompleted`。
   - 小精靈模式同樣適用。
5. **超過上限的失敗**：
   - Providers 把錯誤訊息含 `context_length_exceeded`、`maximum context length`、`prompt is too long`、`too many tokens` 的 400 對應成 `ChatFailureKind.ContextTooLong`（Anthropic 與 OpenAI 相容都要）。
   - `ConversationService` 收到時：自動做第二層，然後重送一次。仍失敗就回報（訊息請使用者開新對話）。
   - `StatusAfterFailure` 對它回 null。
6. **手動整理**：`IChatSession.CompactAsync` 直接做第二層（不看門檻）；回應中不能執行。
7. **用量**：整理後下一次請求的實際用量會自然變小。`ConversationUsageTally` 的累計（輸入／輸出／回合）不變；摘要那次呼叫的用量**算進累計**（用的是同一把金鑰的錢），用 `UsageEvent` 發出並存進 `conversation_compactions`（schema 欄位不夠就停下來回報）。
8. **測試**（假的 provider）：
   - `CompactionApplier`：摘要取代前段、工具結果省略、成對結構不壞、thinking 只留最後一段、沒有整理時與現在完全相同。
   - 門檻：80% 只做第一層、仍超過 70% 才做第二層、沒有 `ContextWindow` 時不整理。
   - `ContextTooLong` 會整理並重送一次、第二次失敗回報。
   - 手動整理、摘要失敗不存、刪除對話時 cascade。
   - 兩家 provider 的錯誤對應。

## 包 22I：壓縮的畫面（Wave 2）

1. **整理中**：工作台訊息區底部顯示「正在整理對話…」（沿用等待中的樣式），小精靈氣泡顯示同一句。
2. **分隔線**：
   - 對話中 `SummaryUptoSeq` 那則訊息之後插一條分隔線「以上內容已整理成摘要」，右邊「查看摘要」，點開用 Expander 顯示摘要全文（可選取）。
   - 只顯示最新一次的位置。
   - 第一層（工具結果）不顯示分隔線。
   - 歷史重新載入時也要顯示。
3. **用量面板**：快滿提醒（Danger）多一個「整理並繼續」按鈕，放在「開新對話」左邊，primary 改給它；呼叫 `CompactAsync`；回應中停用。
4. **失敗**：整理失敗時，在訊息區顯示一則提示（既有的錯誤提示樣式），不擋使用者繼續送出。
5. **測試**（headless）：分隔線位置與展開、按鈕狀態、整理中提示出現與消失。

## 包 22J：小精靈專屬對話（Wave 2）

1. **專屬對話**：
   - 小精靈對話的 id 存在 `app_meta`（key `sprite_conversation_id`）。
   - 第一次需要時建立，標題「與小精靈的對話」。
   - 對話被刪除（找不到 id）時重新建立一則。
2. **切換規則**（單一 `ChatSession` 不變，切模式時切對話）：
   - 進入小精靈模式：記下工作台目前的對話 id，`OpenConversationAsync(小精靈對話)`。
   - 回到工作台：重新開回記下的那則（被刪了就照 `InitializeAsync` 的規則）。
   - 切換時若正在回應：不停止，等 `RunFinished` 之後才切；這段時間小精靈不顯示那則回應（`SpriteConversationController` 現在就會忽略工作台發起的回應），輸入列照現有的回應中規則。
   - 啟動時若上次是小精靈模式（`SpriteSettings.LastMode`），直接開小精靈對話。
3. **歷史清單**：小精靈對話照常出現在工作台的歷史清單，可以打開來看或繼續聊。在工作台打開它不影響規則（回到小精靈時仍開它）。可以刪除，刪了下次自動重建。
4. **「在工作台顯示完整內容」**（小精靈把長內容交給工作台的既有流程）：先讀現有做法。如果它依賴「兩邊是同一則對話」，改成：工作台開啟小精靈對話並捲到那則訊息。不確定就停下來回報。
5. **測試**：切到小精靈開專屬對話、切回工作台回到原對話、回應中切換延後、專屬對話被刪後重建、啟動時的模式、在工作台顯示完整內容。

## 整合（協調者）

1. 依序合併 Wave 1（22A–22F、22H），再開 Wave 2（22G、22I、22J）。
2. `docs/ui-spec.md`：AI 連線頁（合併版）、小精靈頁、訊息可選取、工具卡片圖片、整理分隔線與「整理並繼續」、小精靈專屬對話。
3. `plan/manual-verification.md` 新增「M22」表：
   - Windows 單檔發佈後對話正常、`%TEMP%\.net` 不再有受控 DLL、瀏覽器工具可用；
   - 乾淨環境第一次啟動沒有 Claude；
   - 舊設定檔的 `dev-anthropic` 被清掉；
   - 重開後首頁不再提示未測試；
   - 小精靈陰影；
   - 小精靈頁捲動；
   - 合併頁新增連線→新增模型流程；
   - 訊息選取與複製（Mac／Windows）；
   - 公司 Gateway 實際產生圖片、用附上的圖片編輯；
   - 長對話自動整理（Anthropic 開 thinking、OpenAI 相容各一）、手動「整理並繼續」；
   - 小精靈與工作台切換時各自的對話。
4. 全方案建置零警告、全部測試通過；macOS 發佈實測（22A 驗收）。

## 待確認的小事（不阻擋開工，實作者遇到時照下面預設）

- 圖片存放位置：預設「圖片／Lumi」，不跟對話的檔案範圍綁在一起。
- `generate_image`／`edit_image` 權限用 `Network`（可選「本次對話都允許」）。
- 壓縮門檻 80%／70%、保留最近 2 輪（摘要）／3 輪（工具結果），先寫成常數，之後依實測調整。
- 圖片不送回模型：模型只知道存檔路徑與 `revised_prompt`，看不到圖。
- `dev-anthropic` 清理只看「金鑰不存在」，不看有沒有對話用過它。
