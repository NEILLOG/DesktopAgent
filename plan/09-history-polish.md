# 09 歷史對話與完成度

- 狀態：9A～9C 程式完成，待人工驗收（2026-10-23 使用者指示直接做，Q1～Q3 皆採建議）；9D、9E、9F 未開始
- 對應里程碑：M9
- 前置：M2（對話儲存、最簡歷史抽屜）、M12（設計系統與歷史抽屜樣式）都已程式完成。
- 檔名說明：`plan/09-convert-from-old.md` 是**舊版 WPF 資料格式規格**（不是 M9 的細部計畫），所以這份用不同的檔名。它與 M9 的關係見 Q3。
- 設計依據：`plan/ui-guide/ui-guide.html`（統一設計原則）；介面規格寫回 `docs/ui-spec.md`。

## 目標

對話累積之後還找得到、管得動（搜尋、分組、改名、刪除）；出錯時使用者知道發生什麼事、能怎麼辦；關掉再開，工作台回到上次的位置與大小；Windows 上的 emoji（特別是國旗）有定案。

## 現況盤點（動工前先確認，免得重做）

| 項目 | 現況 | M9 要補的 |
|---|---|---|
| 歷史抽屜 | `HistoryViewModel`：最近 100 則、只有標題加時間（今天 `HH:mm`、其他 `M/d`）、點了切換 | 搜尋、依日期分組、改名、刪除、載入更多 |
| 儲存 | `IConversationStore` 已有 `ListAsync`、`RenameAsync`；**沒有刪除、沒有搜尋**；schema 目前 3 版；外鍵 `ON DELETE CASCADE` 已設（訊息、上下文項目會跟著刪） | `DeleteAsync`、`SearchAsync`、分頁 |
| 附件 | `AttachmentCleaner` 只在啟動時跑一次（M4 紀錄：刪除對話後要同樣呼叫） | 刪除後呼叫清理 |
| 標題 | 第一則訊息送出時自動設成前 30 字（`ConversationService.TitleMaxLength`）；之後不會再自動改 | 手動改名不會被蓋掉（現況已成立，補測試） |
| 錯誤列 | 已有：依 `ChatFailureKind` 分類、「重試」「換模型重試」、小精靈錯誤氣泡、金鑰問題帶「開啟設定」 | 離線／逾時的文字更具體、`RetryAfter` 顯示（見 9C） |
| 啟動失敗 | `App.StartAsync` 只寫 log 然後 `Shutdown(1)`——**使用者看到的是程式突然消失** | 顯示說明對話框（9C） |
| 視窗位置 | 小精靈位置已記（`SpriteSettings.Placement`）；工作台每次 1040×760 置中 | 工作台位置、大小、最大化（9D） |
| Emoji | `EmojiText` 以字形叢集切段，指定 Apple Color Emoji／Segoe UI Emoji；沒有打包 emoji 字型 | Windows 國旗驗證與決定（9E） |

## 範圍

做：
- 歷史抽屜：搜尋、依日期分組、改名、刪除（含確認）、載入更多
- `IConversationStore`：`DeleteAsync`、`SearchAsync`、分頁；SQLite migration 4
- 刪除對話後的附件清理、權限記憶清除、目前對話被刪時的導向
- 錯誤處理補強：離線／逾時提示、`RetryAfter`、啟動失敗對話框、資料庫無法開啟的說明、未處理例外落地到 log
- 工作台視窗位置、大小、最大化記憶（含螢幕變動的保護）
- Emoji 字型：先在 Windows 實機驗證，再決定要不要附 Twemoji
- 文件同步：`docs/ui-spec.md`（歷史抽屜、錯誤、視窗）、`agent.md`（第 7 節補「整個對話可以刪除」）、README、主計畫

不做：
- 匯入舊版 WPF 的歷史對話（Q3，建議另開里程碑）
- 搜尋結果點開後捲到命中的那則訊息、命中字高亮到訊息內文（只在清單裡高亮）
- 匯出對話、釘選／封存、資料夾或標籤、多選批次刪除、「清除全部」
- 垃圾桶／復原刪除（Q2）
- 自動依內容重新產生標題
- 設定視窗的位置與大小記憶（維持每次置中、預設 1040×760，是 M12 的決定）
- 把語音轉寫存成「語音訊息」型別（06 留給 M9 評估：**評估結果是不做**——訊息只存文字，沒有任何功能需要區分輸入方式；真的需要時再加欄位）
- 自動重送（背景自動重試 429／5xx）：重試維持使用者手動按，理由見「直接定的事」

---

## 開工前要確認的事

影響結構或行為的決定，各給建議；**確認前不動工**（agent.md 第 3 節）。

| # | 問題 | 建議 | 影響 |
|---|---|---|---|
| Q1 | 搜尋要搜什麼、怎麼搜？ | **搜標題加訊息文字內容**（只看使用者與助理的文字區塊，不含工具結果、檔案快照、thinking）。作法：migration 4 在 `messages` 加 `search_text` 欄（寫入訊息時從 `TextBlock` 取純文字），啟動時背景把舊訊息補上一次；查詢用 `LIKE`（不分大小寫，跳脫 `%`／`_`）。理由：這是個人用的本機資料量，`LIKE` 掃幾萬則訊息仍是毫秒級；SQLite 內建的 FTS5 預設分詞器不會切中文，trigram 分詞器要求查詢至少 3 個字元，1～2 字的中文詞（「報告」）會查不到，反而要另外補退路。不直接搜 `content_json`：System.Text.Json 預設會把中文轉成 `\uXXXX`，`LIKE` 比不到。替代：只搜標題（最簡單，但「我記得聊過 X」找不到）；或 FTS5 trigram 加 `LIKE` 退路（資料量大到 `LIKE` 慢時再換，介面不用改）。 | 9A |
| Q2 | 刪除要不要能復原？ | **直接刪除，不做垃圾桶**；刪之前一定跳確認對話框，說明「這則對話的訊息、加入的檔案清單與圖片附件會一併刪除，無法復原」。按鈕用 `danger`，預設焦點在「取消」。理由：垃圾桶要另外的狀態欄位、還原 UI 與過期清理，個人桌面工具的確認對話框已足夠；`agent.md` 第 7 節「不能刪除中間的訊息」是針對單則訊息（模型會檢查歷史一致性），整個對話刪除不受影響，我會把這點寫進該節。回應中的對話（含小精靈正在回答的）不能刪，按鈕停用並寫出原因。 | 9A、9B |
| Q3 | 要不要把「匯入舊版歷史對話」放進 M9？`plan/09-convert-from-old.md` 已整理好舊版格式，主計畫 M9 沒有列這項。 | **不放，另開一個里程碑（暫稱 M13「匯入舊版資料」）**。理由：舊版是 Windows DPAPI 加密，只能在 Windows 解、需要 Windows 實機驗證；範圍不只對話（記憶、排程、人格、設定都在規格裡）；而且 M9 的清單與搜尋要先穩定，匯入的資料才有地方放。M9 會讓 `conversations` 的欄位足夠承接匯入（有標題、建立與更新時間），不為它預留額外欄位。替代：只把「歷史對話」那一類塞進 M9——可行，但要多一個 Windows 驗證關卡，會拖慢 M9 其他項目。 | 範圍 |

其他我直接定的事（有意見再指出；數字集中在 `HistoryOptions`／`WindowMemoryOptions`）：

- **依日期分組**：今天、昨天、過去 7 天、過去 30 天、更早（依本地時區，用 `TimeProvider`，與 `FormatTime` 一致）；沒有項目的群組不顯示。搜尋中不分組，改列「N 筆結果」。
- **分頁**：一次載 50 則，抽屜捲到底顯示「載入更多」按鈕（不做無限捲動，避免鍵盤與螢幕閱讀器的焦點跳動）；游標用 `(updated_at, id)`。搜尋最多 100 筆，超過寫「只顯示前 100 筆，請縮小搜尋條件」。
- **搜尋互動**：抽屜頂端搜尋框，輸入後 250 ms 才查（`System.Reactive` 的 Throttle 只用在這裡，agent.md 第 4 節允許局部使用）；Esc 清除搜尋；結果顯示標題（命中字高亮）加一行命中片段（前後各約 20 字）；查詢字串去頭尾空白，空白不搜。搜尋是資料庫查詢，取消舊查詢（`CancellationToken`）避免舊結果蓋過新的。
- **改名**：項目的「…」選單或 F2 進入行內編輯；Enter 確認、Esc 取消；去頭尾空白，空字串視為取消，上限 60 字（自動標題 30 字不變）。回應中也可以改名（只動標題，不影響歷史）。
- **刪除目前開著的對話**：導向清單中的下一則；沒有就開新對話（沿用 `NewConversationAsync` 的延遲建立）。刪除不經過 `IChatSession` 的送出流程，但要通知它目前對話已不存在。
- **權限記憶**（`ToolPermissionMemory`，依對話分開、只在記憶體）在刪除時一併清掉。
- **離線偵測採被動**：不主動偵測也不阻擋送出。`NetworkInterface.GetIsNetworkAvailable()` 在 VPN、內網端點、本機模型（localhost）下不可靠，會誤擋合法的使用情境。失敗後的 `Network` 類錯誤才判斷：若系統回報沒有任何可用網路介面，訊息補一句「看起來沒有網路連線」；連線目標是 localhost／內網主機時改說「確認服務是否已啟動」（OpenAI 相容那邊已有）。
- **自動重試維持不做**：串流中途重送會讓使用者看到文字重複、也可能重複計費；429／529 的 `RetryAfter` 只顯示成「約 N 秒後再試」的說明，由使用者按「重試」。
- **視窗記憶**：存在使用者資料夾的 `settings.json`（和 `SpriteSettings` 同一個檔案，`IModelCatalog` 旁邊加窗口設定），欄位是位置（螢幕像素）、大小（邏輯像素）、是否最大化、所在螢幕的縮放比例。還原時檢查：寬高不小於 `MinWidth`／`MinHeight`、不大於目前所有螢幕聯集；至少有 100×40 的標題列區域落在某個螢幕內，否則退回置中的預設大小。最大化時記的是「還原後」的位置與大小，不是最大化的尺寸。移動或縮放時節流 500 ms 才寫檔，關閉視窗時再寫一次。
- **Emoji 決定規則**（9E）：Windows 實機上若國旗顯示成兩個字母（Segoe UI Emoji 沒有國旗字形，是已知行為），就附 Twemoji COLR 並只用於國旗與缺字的叢集；其他 emoji 仍用系統字型。這個決定要等實機驗證，見 9E 的關卡。

---

## 設計

### 資料層（Core／Infrastructure）

`IConversationStore` 新增（不改既有成員的語意）：

```csharp
/// 刪除整個對話：訊息、上下文項目隨外鍵一併刪除。不存在時回 false。
Task<bool> DeleteAsync(string conversationId, CancellationToken ct = default);

/// 有訊息的對話，依 updated_at 由新到舊；before 是上一頁最後一則（游標），null 表示第一頁。
Task<IReadOnlyList<Conversation>> ListAsync(int limit, ConversationCursor? before, CancellationToken ct = default);

/// 比對標題與訊息文字（不分大小寫）；每筆附命中片段。
Task<IReadOnlyList<ConversationSearchHit>> SearchAsync(string query, int limit, CancellationToken ct = default);
```

- `ConversationCursor(DateTimeOffset UpdatedAt, string Id)`、`ConversationSearchHit(Conversation Conversation, string Snippet)` 放 `Lumi.Core/Chat`。原本的 `ListAsync(limit)` 保留（呼叫端不用全改），內部呼叫新的重載。
- SQLite migration 4：`messages` 加 `search_text TEXT NOT NULL DEFAULT ''`；舊訊息由啟動後的背景作業補（`SearchTextBackfill`，分批、可中斷、重跑安全，進度不影響使用——補完之前舊訊息只能靠標題搜到，設定頁不特別提示）。既有 migration 不改。
- `AppendMessageAsync` 寫入時一併填 `search_text`（`SearchTextExtractor`，Core 純函式：串接 `TextBlock`，上限 20,000 字元，截斷不報錯）。
- 附件清理：`AttachmentCleaner` 目前只有「啟動時跑一次」。新增 `ConversationDeletionService`（Core）：`store.DeleteAsync` → `memory.Clear(conversationId)` → `cleaner.CleanAsync()`；工作台的刪除命令只呼叫它。
- 資料庫是較新 schema 時的行為不變（拒絕開啟），但錯誤要能被 9C 的啟動失敗對話框顯示。

### 歷史抽屜（App）

- `HistoryViewModel` 擴充：`SearchText`、`Groups`（`ObservableCollection<HistoryGroup>`，群組標題加項目）、`HasMore`、`LoadMoreCommand`、`RenameCommand`、`DeleteCommand`、`IsSearching`。切換對話、刪除、改名的邏輯各自可單元測試（不開視窗）。
- 項目列：標題＋時間；游標停留或鍵盤聚焦時右側出現「…」純圖示按鈕（Tooltip＋`AutomationProperties.Name`），選單項目「改名」「刪除」。右鍵與 Shift+F10 也開同一個選單。
- 搜尋框用 M12 的 `TextBox` 樣式與 Focus 光暈，前面放 Lucide 搜尋圖示；有輸入時右側顯示清除鈕。無結果時顯示空狀態文字「沒有符合『…』的對話」。
- 刪除確認沿用 `ConfirmDialog`（M12 已 token 化）；Narrow 寬度下抽屜是 Overlay，確認對話框要蓋在它上面。
- 圖示若缺（搜尋、更多、鉛筆、垃圾桶）補進 `Icons.axaml`，來源是 Lucide，授權檔已在 `Assets/LICENSES`。
- 所有顏色、間距、字級只用 M12 的 token；一律不加新的色碼（架構測試守住）。

### 錯誤處理與離線（9C）

不重做錯誤列；M9 只補以下缺口：

1. **啟動失敗**：`App.StartAsync` 的 `catch` 先顯示一個簡單的 Avalonia 視窗（不依賴 DI——啟動失敗時 DI 可能就是壞的），內容：一句繁中原因分類（資料庫無法開啟／資料庫由較新版本建立／設定檔無法讀取／其他）、log 資料夾位置、「複製詳細資訊」「結束」兩個按鈕，詳細資訊不含金鑰與訊息內容。log 用 `LogCritical` 照舊。
2. **資料庫無法開啟**的分類：檔案被其他程序鎖住（常見：第二個實例）、權限不足、磁碟空間不足、schema 較新。第二個實例的情形：偵測到時用說明取代直接消失（是否做單一實例鎖屬於另一件事，**不在 M9**，只做說明）。
3. **全域未處理例外**：`AppDomain.UnhandledException`、`TaskScheduler.UnobservedTaskException`、Avalonia UI 執行緒例外寫進 log（不彈窗，除非是啟動階段）。
4. **聊天錯誤文字**：`Network` 加上述的離線判斷；`RateLimited`／`Overloaded` 帶 `RetryAfter` 時顯示「約 N 秒後再試」；錯誤列與小精靈氣泡共用同一段文字（已共用 `ChatFailure.Message`）。Anthropic SDK 例外目前讀不到 `RetryAfter`（`AnthropicErrorMapper` 註解），先不處理，維持 null。
5. **串流中途中斷**：確認既有行為——已收到的文字存成助理訊息、`status` 標示未完成，「重試」不會刪掉它（`RegenerateAsync` 才會）。若實際行為不符，**列入實作紀錄並回報，不在 M9 順手改歷史規則**（agent.md 第 7 節）。

### 視窗記憶（9D）

- `WindowSettings(int X, int Y, double Width, double Height, bool Maximized, double Scaling)` 放 `Lumi.Core/Models`；`IModelCatalog` 旁邊加 `GetWindowAsync`／`SetWindowAsync`（或獨立 `IWindowSettingsStore`——實作時依 `settings.json` 的寫入方式擇一，不新增檔案）。
- `WindowPlacementRules`（App，純函式，仿 `SpritePlacementRules`）：輸入儲存值與目前螢幕清單（含工作區、縮放），輸出是否可用與修正後的位置大小。多螢幕、螢幕拔掉、縮放比例改變都用單元測試守住。
- 範圍僅工作台視窗。小精靈位置已有。設定視窗不記。
- 寫檔節流用 `System.Reactive`（局部）或自己的 `DispatcherTimer`，擇一，不新增依賴。

### Emoji（9E）

先驗證、後決定，順序固定：

1. 準備一份驗證字串（國旗 🇹🇼🇯🇵🇺🇸、膚色 👍🏽、ZWJ 序列 👨‍👩‍👧、keycap 1️⃣、Unicode 15+ 新字），放在 `poc` 或 `samples` 的小頁面，**在 Windows 11 實機**跑工作台訊息、程式碼區塊、小精靈氣泡三處。
2. 結果記進這份計畫的實作紀錄。國旗正常 → 不附字型，結案。
3. 國旗顯示成字母 → 評估 Twemoji COLR：確認 Avalonia 11.3 的 Skia 後端在 Windows 上能繪製 COLR v0 的內嵌字型（這是最大的未知數，所以要先 spike）；可行才打包，並在 `Assets/Fonts/LICENSES` 與關於頁標示授權（字型檔案的授權條款需在加入前核對）；只讓 `EmojiText` 對區域指示符叢集（國旗）或系統缺字的叢集使用它，其他不變。
4. 不可行 → 保持現狀，把「Windows 國旗顯示為字母」記為已知限制，寫進 README，不再花時間。

我在 macOS 上無法做第 1 步，需要使用者在 Windows 上跑（或回報結果）。**9E 不擋其他項目**，放在最後做。

---

## 工作項目

### 9A 資料層

- [x] `ConversationCursor`、`ConversationSearchHit`、`IConversationStore` 新成員（Core）
- [x] SQLite migration 4（`search_text`）＋ `SearchTextExtractor` ＋ `SearchTextBackfill`
- [x] `SqliteConversationStore`：`DeleteAsync`、分頁 `ListAsync`、`SearchAsync`（`LIKE` 跳脫、片段擷取）
- [x] `ConversationDeletionService`（刪除 → 清權限記憶 → 清附件）
- [x] 測試：刪除連動（訊息、上下文項目）、分頁邊界（相同 `updated_at`）、搜尋（中英文、1～2 字中文、`%`／`_`、大小寫、跳脫）、片段擷取、backfill 重跑安全、migration 3→4

### 9B 歷史抽屜介面

- [x] `HistoryViewModel` 擴充（搜尋、分組、載入更多、改名、刪除）＋ `HistoryOptions`
- [x] 抽屜 View：搜尋框、群組標題、項目「…」選單、行內改名、載入更多、空狀態
- [x] 刪除確認（`ConfirmDialog`），目前對話被刪的導向
- [x] 鍵盤與螢幕閱讀器：F2、Delete、Esc、Shift+F10；純圖示都有 Tooltip 與 `AutomationProperties.Name`。「…」刻意不在 Tab 鏈上（見實作紀錄 9B）
- [ ] Wide／Compact／Narrow 三段版面檢查（Narrow Overlay 不蓋住確認對話框）：**未做**。headless 測試只驗證抽屜的結構與互動，沒有實機或截圖比對；Narrow 的覆蓋行為要人工看（見實作紀錄 9B）
- [x] 測試：分組邊界（午夜、時區）、搜尋節流與取消、改名驗證、刪除後選取項目、回應中停用

### 9C 錯誤處理與離線

- [x] 啟動失敗視窗（不依賴 DI）＋ 失敗分類（實機顯示未驗證，見實作紀錄 9C）
- [x] 全域未處理例外寫 log
- [x] `Network` 離線判斷、`RetryAfter` 文字
- [x] 確認串流中斷後的存檔行為（結果寫進實作紀錄）
- [x] 測試：失敗分類純函式、文字內容不含金鑰與路徑

### 9D 視窗位置與大小記憶

- [ ] `WindowSettings` ＋ 讀寫（`settings.json`，損毀時退回預設並沿用既有的一次性提示）
- [ ] `WindowPlacementRules` 與測試（多螢幕、螢幕消失、縮放變動、最小尺寸）
- [ ] 工作台視窗接線（開啟時還原、移動／縮放節流寫入、關閉時寫入、最大化處理）

### 9E Emoji

- [ ] 驗證頁與 Windows 實機結果（使用者執行）
- [ ] 依結果結案，或打包 Twemoji COLR 並標示授權

### 9F 文件

- [ ] `docs/ui-spec.md`：歷史抽屜（搜尋、分組、改名、刪除）、啟動失敗視窗、視窗記憶
- [ ] `agent.md` 第 7 節補「整個對話可以刪除，單則訊息仍不可」，第 12 節若有新增規則一併
- [ ] README、`plan/00-master-plan.md`（M9 狀態、決策紀錄）

---

## 驗收標準

自動化（每一段完成都要跑）：
- 全部既有測試不退化；架構測試通過（Core 不引用 Avalonia、色碼只在 `Themes/`、P/Invoke 只在 Platform）
- 新增的純函式與資料層都有單元測試（見各段）

人工驗收：
- 建 150 則以上的對話（可用測試資料腳本）：抽屜依日期分組、載入更多正常、搜尋中文 1～2 字能命中、命中片段合理
- 改名後重開程式還在；刪除後重開程式不會回來；`attachments/` 內該對話的圖片檔案已清掉
- 刪除目前開著的對話：導向下一則或新對話，輸入框不殘留舊草稿與上下文列
- 回應中：刪除與切換行為與 M2 一致（刪除停用並有原因）
- 拔掉網路送訊息：看到具體的錯誤說明，「重試」可用；接回網路按重試成功
- 把資料庫檔案改成被鎖住或換成較新 schema：啟動時看到說明視窗而不是程式消失
- 工作台移到第二螢幕、縮放、最大化後關閉重開，回到相同位置；拔掉第二螢幕後重開，視窗出現在可見範圍
- Windows 實機的 emoji 驗證結果已記錄

## 風險

- **`LIKE` 掃描效能**：個人資料量不是問題；若日後資料量大（數十萬則訊息），換 FTS5 trigram 加退路，`SearchAsync` 介面不變。
- **migration 4 的 backfill** 若被中斷：重跑安全（以 `search_text = ''` 且有文字區塊為條件）；但純圖片或純工具訊息本來就沒有文字，條件要避免每次啟動都重掃，用 `schema_version` 旁的旗標記錄「已補完」。
- **COLR 字型在 Avalonia/Skia 是否可用**不確定，所以 9E 先 spike。
- **刪除是不可逆的**：確認對話框＋預設焦點在取消；不做批次刪除。
- **視窗還原的螢幕座標**在 macOS（點）與 Windows（像素）單位不同：儲存時帶縮放比例，還原時重新換算並用 `WindowPlacementRules` 的測試守住。

## 待人工確認（9A～9C，全部做完後一次確認）

- 9A：補寫舊訊息與送出訊息同時寫 SQLite 的鎖競爭沒實機測過；LIKE 只對 ASCII 不分大小寫。
- 9B：Narrow 時抽屜 Overlay 與刪除確認框的層級；Compact／Wide 抽屜內搜尋框、清除鈕、群組標題對齊；「…」游標移入才出現、點它不誤切換對話；從非目前列點「…」→改名時焦點與選取；F2／Delete／Shift+F10／右鍵開同一選單；回應中刪除的 Tooltip；中文 1～2 字搜尋、命中字（只加粗）辨識度、100 筆上限提示；刪除確認預設焦點在取消（設定頁刪連線、清金鑰也同樣）；刪目前對話後導向與草稿清空；深色與五套主題的選取底可讀性。
- 9B 決定待確認：「…」不在 Tab 鏈上（改用 F2／Delete／Shift+F10／選單鍵）；回應中一律不能刪任何對話。
- 9C：啟動失敗視窗實機顯示（鎖住資料庫、schema 改 99、`chmod 000 settings.json` 三種重現方式見 9C 實作紀錄）；鎖住時視窗約晚 30 秒出現（SQLite 預設 busy timeout）；詳細資訊含經遮蔽的例外訊息（要不要更保守只留型別與 HResult）；拔網路／VPN 的離線文字、429 的「約 N 秒後再試」；UI 例外寫 log 且仍照舊結束。
- 9C 結論待決定：串流中途失敗時已收到的部分文字不保留（既有 Q15 規則，只有使用者取消才存 Stopped 訊息）。與本計畫預期不符；要保留需改歷史規則，M9 沒動。

## 實作紀錄（偏離原計畫的地方與原因）

### 9A

9A：有偏離，都是計畫沒寫到、為了讓介面與測試能接上的小決定：

- `ListAsync(int, CancellationToken)` 保留為 `IConversationStore` 的預設實作（呼叫新重載並傳 `null`），`SqliteConversationStore` 與兩個測試用假實作各自有對應的 overload 委派，呼叫端不用全改。
- 分頁的排序與游標 tie-break 用 `c.id`（不是 `rowid`）：分頁查詢是 `ORDER BY updated_at DESC, id DESC`，和游標條件 `(updated_at, id)` 一致。`GetLatestAsync` 仍用 `rowid`，沒有改。
- 搜尋只列有訊息的對話（與 `ListAsync` 一致）；空對話即使標題命中也不列。
- 搜尋片段取「第一則命中的訊息」；換行與連續空白收成一個空格；前後**不加省略號**（9B 畫面自己加）；片段不標記命中字（計畫不做高亮到內文）。
- 補寫旗標用 migration 4 一起建的 `app_meta(key, value)` 表（計畫的「擇一」）。
- `ConversationDeletionService.DeleteAsync` 在對話不存在（回 false）時仍會清權限記憶與附件；兩者都沒有副作用，所以不短路。資料庫丟例外時才完全不往後做。
- `ConversationDeletionService` 註冊在 `AddLumiCore`，但依賴的 `AttachmentCleaner` 由 `AddLumiInfrastructure` 註冊（與既有的 `AttachmentCleaner` 註冊位置一致）。App 必須同時註冊兩者。
- 既有測試 `UpgradeFromVersion1_…`、`UpgradeFromVersion2_…` 模擬舊資料庫時，改為同時移除 `search_text` 與 `app_meta`（它們是 migration 4 加入的）；否則 migration 4 會重複加欄位而失敗。測試的驗證內容沒有弱化。
- 補寫作業（`SearchTextBackfillService`）在 `StopAsync` 取消；沒有做完就不記旗標，下次啟動接著做。
- 新增 `ToolPermissionMemory.Clear(conversationId)`（計畫允許）。

9A 需要人工確認的事（實作上沒有處理）：

- `LIKE` 只對 ASCII 不分大小寫；非 ASCII 的大小寫折疊（例如希臘字母、帶變音符號的拉丁字母）不做。中文不受影響。
- 補寫與送出訊息同時寫 SQLite 時的鎖競爭沒有實機測過（每批 200 則、一批一個交易，應該很短）。
- 搜尋仍是 `LIKE` 全表掃描，符合計畫的風險說明；資料量大時再換 FTS5。

### 9B

9B：有偏離，都是計畫沒寫到或需要為了安全、可用性做的決定（採最保守的做法）：

- **回應中不能刪除任何對話**（計畫只寫「回應中的對話不能刪」）。原因：`AttachmentCleaner` 只能在沒有送出進行中時跑（它的註解寫明：圖片先存檔、訊息後存，中間的檔案看起來像孤兒）。刪除另一則對話也會觸發清理，所以回應中一律停用，「…」選單的刪除項目 Tooltip 寫出「回應中，暫時無法刪除」。
- **「…」不在 Tab 鏈上**（`IsTabStop=False`）。每列一個 Tab 停止點，清單很長時 Tab 會很難用。鍵盤操作改用 F2（改名）、Delete（刪除）、Shift+F10 與選單鍵（開選單）；Tab 進到清單後用方向鍵移動。清除鈕是 Tab 停止點（只在有輸入時出現）。這和計畫的「Tab 順序」描述有出入，請人工確認是否接受。
- **群組標題列是 `ListBox` 裡 `IsEnabled=false` 的列**（在 `ContainerPrepared` 設定），不是獨立的控制項。這樣方向鍵會跳過標題，而且標題和對話在同一個捲動區裡。
- **抽屜開啟時重新載入第一頁**。既有程式在啟動後不會載入歷史清單，要等送出或切換對話才出現（M2 的漏洞，本次順手修正，範圍只限抽屜）。
- **刪除順序**：先從資料庫刪（`ConversationDeletionService`：訊息、權限記憶、附件清理），刪除失敗就不切換；成功後才切到下一則。切換用既有的 `OpenConversationAsync`，沒有「沒有對話」的特殊路徑：沒有下一則時用 `NewConversationAsync` 同一段的建立流程。
- **`IChatSession` 沒有新增成員**：工作台切換到別的對話後，session 就不再指向被刪的對話（既有的切換流程會更新 `ConversationId`），不需要額外通知。
- **改名：行內輸入框失去焦點視為取消**（計畫沒寫）。不會在點到別處時自動儲存，避免誤存。
- **改名字數**：輸入框 `MaxLength` 以 UTF-16 單位計，送出時以字元（code point）計截斷。含 emoji 的標題可能在輸入時就被 `MaxLength` 擋住，比送出時截斷得更早，兩者不完全一致。
- **搜尋狀態**：`IsSearching` 指「節流中或等待結果」；搜尋模式下 `HasMore` 一律為 false（「載入更多」隱藏）；清除搜尋後才重新載入第一頁。
- **命中字只加粗**，沒有改色（選取列的文字是 `TextOnSoft`，避免另外的對比問題）。
- **確認對話框的預設焦點**：`ConfirmDialog` 在 `destructive` 時把焦點放在「取消」。這個改動會套用到所有破壞性確認（設定裡的刪除連線、清除金鑰等），不只是歷史抽屜。
- **命中片段長度**：實際的擷取長度在 Core 的 `SearchSnippet.Context`（9A 已決定），`HistoryOptions.SnippetContext` 只是參照，App 端改它沒有效果。
- **圖示**：新增 `LumiIconSearch`、`LumiIconPencil`、`LumiIconTrash`（Lucide 0.468.0，路徑取自官方 SVG）；「更多」與「X」已存在，直接沿用。
- **Narrow 的確認對話框**：`ConfirmDialog` 是獨立視窗（以作用中的視窗為 owner），天然蓋在 Overlay 抽屜上面。程式上沒有額外處理，**但沒有實機確認**。

9B 未驗證（需人工確認，見回報）：Narrow／Compact 的抽屜與確認框的實際畫面、搜尋命中字的視覺（只加粗）、「…」在滑鼠游標移入時的出現與位置、改名輸入框的焦點框、中文標題換行、Windows 上的字型與高度。

### 9C

9C：有偏離（計畫沒寫到、與現況不符，或為了讓它能運作）：

- **沒有檔案記錄器**：計畫假設有 log 資料夾，但現況只有 console 與 debug 輸出，打包後的程式沒有地方可看。新增 `LumiFileLoggerProvider`（`Lumi.Infrastructure/Logging`，沒有新套件）：寫進 `<使用者資料夾>/logs/lumi-yyyy-MM-dd.log`，Information 以上，`Microsoft.*` 只記 Warning 以上，寫檔失敗時吞掉。說明視窗的「記錄檔資料夾」指向這裡。
- **分類器放在 `Lumi.Infrastructure`**（計畫寫 App 或 Core）：Core 不能引用 SQLite，App 不能引用 `Microsoft.Data.Sqlite`（`SqliteRuleTests` 守住）。純函式，只看型別與錯誤碼，不讀訊息文字。
- **新增兩個例外型別**：`DatabaseTooNewException : InvalidOperationException`（既有呼叫端與測試相容，訊息原樣）；`SettingsFileUnreadableException : IOException`。計畫沒寫「設定檔無法讀取」怎麼辨識，所以 `JsonModelCatalog` 讀檔時的 `IOException`／`UnauthorizedAccessException` 改包成這個例外。**損毀的 JSON 行為不變**（改名保留、用預設值），讀不進來不改名、不覆寫。
- **分類多一項**：SQLITE_READONLY、SQLITE_PERM 歸為「權限不足」（計畫沒列）。磁碟空間不足的 IOException 比對 HResult 低 16 位（0x70、0x27），HRESULT 形式（0x8007xxxx）也接受；macOS 的 ENOSPC（errno 28）沒有納入，磁碟滿時 SQLite 會回 SQLITE_FULL，那條已處理。
- **離線補句放在 `ConversationService`（Core）一處**，不是 `ChatSession`：離線判斷需要連線目標（Base URL），`ChatSession` 沒有這個資訊。`ChatSession`、工作台、小精靈都只讀 `ChatFailure.Message`，所以三邊拿到同一段文字（`ConversationFailureTests` 鎖定）。`ConversationService` 沒有注入 `INetworkStatus` 時預設 `AlwaysOnlineNetworkStatus`（既有測試不受影響）。
- **「內網」的判斷**（`ChatFailureText.IsLocalOrPrivateHost`）：localhost、loopback、私有 IPv4（10/8、172.16/12、192.168/16、169.254/16）、IPv6 loopback／link-local／unique-local、沒有點的主機名稱、`.local`／`.lan`／`.internal`／`.home.arpa`。其他一律當外部主機。公司的一般網域名稱會被當外部：只有在「沒有任何可用網路介面」時才補句，所以誤判的代價低。
- **可用網路介面**（`NetworkAvailability`）：必須已啟動、不是 loopback，名稱或描述不含 virtual、vmware、vmnet、vbox、hyper-v、vethernet、docker、veth、bridge、wsl、loopback、pseudo、awdl、llw、anpi。VPN 隧道（utun、Tunnel）算可用。列舉失敗時當作有網路。
- **`RetryAfter`**：0 或負值不顯示；無條件進位到秒；超過 60 秒改成分鐘（61 秒 → 「約 2 分鐘」）。只替換「請稍後再試」這個詞，其他文字不動。
- **啟動失敗的範圍**：`Program.Main` 改為回傳 `int`，讓 `Shutdown(1)` 的結束碼真的成為程序結束碼。`host.Start()` 之前失敗時沒有 DI，說明視窗只用 `LumiFileLoggerProvider` 直接寫 log。全域未處理例外的掛鉤在主機建好後才掛（`App.StartNormally`）；UI 例外掛在 `Dispatcher.UIThread.UnhandledException`。
- **Avalonia 本身起不來不在涵蓋範圍**：例如平台渲染器初始化失敗，發生時還沒有任何 Lumi 程式碼執行，程式照舊崩潰。這不是 DI 或資料庫問題，計畫也沒有要求。
- **詳細資訊含例外訊息（經遮蔽）**：計畫寫「詳細資訊不含金鑰與訊息內容」。我把「訊息內容」理解為聊天內文，例外訊息仍放進去（經過 `SecretRedactor`：`sk-…`、`Bearer …`、`api_key=…`、32 字元以上英數字串換成 `[遮蔽]`，使用者資料夾換成 `~`）。若要更保守，見下方人工確認 2。

**串流中斷的存檔行為（第 5 點，現況鎖定，沒有改歷史規則）**：與計畫的預期**不符**。

- 串流中途 provider 回 `Failed`（StreamError、Overloaded 等）：**已收到的文字不存**，也**沒有**狀態為未完成（`Stopped`）的助理訊息。整則回應都不存（Q15；`ConversationService.StreamOnceAsync` 只把使用者取消（`OperationCanceledException`）時已收到的文字存成 `Stopped`）。UI 會移除正在串流的氣泡，只留錯誤列。
- 「重試」（`RetryAsync`）不新增、不刪除任何訊息，同一段歷史重送一次（新增 `StreamFailure_AfterSomeText_SavesNoAssistantMessage_AndRetryResendsTheSameQuestion`）。
- 「重新產生」（`RegenerateAsync`）才會移除尾端的助理訊息，前面的訊息不動（新增 `Regenerate_RemovesTheStoredAnswerOnly_FromTheTail`）。
- 使用者停止時，已收到的文字存為 `Stopped`（既有 `Cancel_AfterText_SavesStoppedMessage_AndRethrows`）。
- 既有 `Send_Failed_DoesNotSaveAssistant` 也鎖定「失敗不存助理訊息」。

9C 需要人工確認的事（實作上沒有處理）：

1. **串流中途失敗時部分文字不保留**（與計畫第 5 點的預期不符）。若要保留，等於改 Q15 的歷史規則，我沒有動。
2. **詳細資訊是否要連例外訊息都不放**（目前放遮蔽後的訊息）。
3. **鎖住資料庫時，說明視窗會晚約 30 秒才出現**：`Microsoft.Data.Sqlite` 預設的 busy timeout 是 30 秒，連線字串沒有另外設定。第二個實例只在資料庫真的被鎖住時會遇到，平常不影響。要縮短需要改 `SqliteConversationStore` 的連線設定。
4. **說明視窗沒有實機顯示過**（見下方「未實機驗證」）。

未實機驗證：在這個工作環境中，Avalonia 的原生渲染計時器初始化失敗（`Avalonia.Native was not able to start the RenderTimer`，錯誤碼 -6661），程式在任何 Lumi 程式碼執行之前就結束，所以無法看到說明視窗。說明視窗、按鈕、複製、結束都只用 headless 測試覆蓋（`StartupFailureWindowTests`）。資料庫被鎖住的情境用真的 SQLite 檔案測過（`RealDatabaseFailureTests`：另一條連線以 `locking_mode=EXCLUSIVE` 獨占，分類為 DatabaseLocked；較新 schema 分類為 DatabaseTooNew）。

人工確認步驟（在 macOS 上做，建議用測試用的資料夾，不要動正式資料）：

- 鎖住資料庫：`(echo "PRAGMA locking_mode=EXCLUSIVE; BEGIN EXCLUSIVE; SELECT 1;"; sleep 120) | sqlite3 "$HOME/Library/Application Support/Lumi/lumi.db"`，然後啟動 Lumi。約 30 秒後應出現「Lumi 無法啟動」，原因為「資料庫正被其他程序使用中，Lumi 可能已經在執行……」；按「結束」後程序結束（從終端機執行時 `echo $?` 應為 1）；`logs/lumi-*.log` 有 `[Critical] Lumi.Startup` 或「啟動失敗」的紀錄。
- 較新 schema：`sqlite3 lumi.db "UPDATE schema_version SET version = 99;"` 後啟動，應顯示「資料庫是較新版本的 Lumi 建立的」。
- 設定檔讀不進來：`chmod 000 settings.json` 後啟動，應顯示「設定檔無法讀取」（測試完記得改回權限）。
- 離線：關閉網路（或停用 Wi-Fi）後送訊息，錯誤列應含「看起來沒有網路連線」；接回網路後按「重試」成功。
- 限流：送到會回 429 且帶 `retry-after` 的連線（例如 OpenAI 相容的 Gateway），錯誤列應顯示「約 N 秒後再試」。

9D～9F：未動。
