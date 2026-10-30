# 23 用量與費用（成本記錄、單價、預算提醒、用量頁）

- 狀態：**實作完成，待人工驗收**（2026-10-31；分支 `m23-usage-cost`，尚未合併回 main。自動測試全過、建置零警告；實機項目見 `plan/manual-verification.md` 的 M23-01～10）
- 對應里程碑：M23
- 來源：2026-10-29～30 與 Claude 的討論（claude.ai）；`plan/misc-backlog.md` 第 4 項
- 視覺稿：Claude Design 畫布「Lumi 用量頁 UI」。**以最下排「第二版」v2-1～v2-3 為準**；上兩排是第一版，只當作「先不做」項目的參考。

## 可行性評估（2026-10-30，讀完程式碼後）

**結論：可行，沒有技術上的阻礙。** 最關鍵的一點已經成立：所有對供應商的呼叫都經過 `ConversationService` 的兩個地方（`StreamOnceAsync`：對話與工具迴圈的每一次呼叫；`CompactSummaryAsync`：摘要），在這兩處各記一筆就能涵蓋全部費用，不用動 Providers 的介面。

| 部分 | 難度 | 說明 |
|---|---|---|
| 每次請求記錄＋單價快照（`request_usage`） | 低 | 新表＋兩個呼叫點。`UsageEvent` 已在串流中送出，停止時也拿得到 Anthropic 的輸入用量 |
| 估算（沒有 usage） | 低 | `ContextEstimator` 現成；`stream_options` 被拒時自動重送是 OpenAI 相容 provider 裡的小改動 |
| 價目表（LiteLLM）與比對 | 中 | 比對規則是純函式，好測；風險在 LiteLLM 檔案格式日後變動（只讀幾個欄位、讀不到就當沒有，風險可控） |
| 預算與提醒 | 低 | 只提醒不中斷，掛在 `RunFinished` 之後 |
| 用量頁、模型設定單價段落、用量面板兩行 | 中 | 純 UI 工作量；沿用既有元件 |
| 9 條使用習慣提示 | **中～高** | 規則本身簡單，工作量在「時機」：回應結束後、輸入框取得焦點時、選模型之前。第 4、5 條的動作要改（見下） |
| Lumi 自己避免浪費 L1～L8 | 低 | 一半已經做好（見該章現況欄） |

**讀程式時發現、和草稿不同的地方（已改進下文）：**

1. **不存在的背景呼叫**：對話標題是本機截字（`ConversationService.MakeTitle`），排程只有 `NotifyAction`（不呼叫模型）。`purpose` 只有 `chat` 與 `compaction` 兩種；L8 不需要做。
2. **第 4、5 條「移除這個檔案／移除圖片」做不到省錢**：上下文項目送出一次後就拍成快照、存在那則使用者訊息裡（M4 Q1），之後從範圍移除只是「之後不再附上」，前文裡的內容還在（現有提示 `Context_RemovedSentNote`：「已從範圍移除；模型之前看過的內容仍在這則對話裡」）。歷史只能往後加（agent.md 第 7 節），也不能從中間拿掉。所以這兩條的動作改成「整理並繼續」（M22 的摘要會把它們整理掉）與「開新對話」（使用者同意，2026-10-30，決定 18）。
3. **重新產生會從尾端刪掉訊息**：如果費用掛在 `messages` 上（做法 A），被重新產生掉的那次費用會消失，月報就少算。這是採用做法 B（獨立的 `request_usage` 表）的決定性理由。同理，刪除對話也不應讓本月花費變少，所以 `request_usage.conversation_id` **不設外鍵**（和 `memories.source_conversation_id` 相同做法）。
4. **沒有勿擾時間**：小精靈的預算提醒拿掉「勿擾時間延後」。
5. **Ollama 沒有回報上下文上限**：程式裡沒有讀取連線回報上下文長度的地方，拿掉「Ollama 優先用連線回報的值」（本機連線本來就不比對價目表）。
6. **L3、L4 已經成立**：system prompt 與工具定義在每則對話第一次送出時凍結（migration 8、9），之後改記憶／人設是附在訊息尾端（`SystemPromptUpdate`）；新對話只帶已啟用的工具（M15 D3）。`PromptComposer` 不含日期時間。所以 `prefix_hash` 欄位不做（照構造不會變），改成一個測試鎖住。
7. **工具定義大小拿得到**：`RunTurnAsync` 裡的 `offered` 就是最終送出的工具清單，現在已經用 `ContextEstimator.EstimateText(ToolSnapshotJson.Serialize(offered))` 估算整理門檻。
8. **第 4、5 條不需要「送出計數」欄位**：從歷史就算得出來（那則使用者訊息之後有幾則助理訊息），不加欄位。

**建議分兩次發版**：Wave 1～2（費用記錄、單價、預算、用量頁、用量面板）先發一版；Wave 3（9 條提示）接著發。Wave 3 不影響前兩波的資料格式。

## 目標

讓使用者知道三件事：

1. 這個月花了多少、會不會超過預算？
2. 這則對話花了多少？
3. 怎麼用可以比較省？

金額是依單價**估算**，不保證等於供應商帳單；目的是給使用者一個控制預算的基礎。

## 範圍

**做**

- 每次 API 請求記下 token 與**當時的單價快照**，算出費用（美元）。
- 模型設定加上單價（輸入、輸出、快取讀取、快取寫入），依 model ID 從內建價目表自動帶入建議值；新增模型時上下文上限與最大輸出一起帶入。
- 每月預算，用到 80% 與 100% 時各提醒一次（工作台與小精靈）。
- 設定視窗新增分類「用量與費用」（單頁，畫板 v2-1）。
- 工作台用量面板加兩行：本對話費用、本月 ／ 預算（畫板 v2-3）。
- **使用習慣提示**：偵測到 9 種浪費行為時，在對話裡適當的時機出現一句提示（第 7 節）。**不顯示「預估可省」金額**。
- **Lumi 自己避免浪費**：工具結果、工具迴圈、圖片大小等程式面的處理（獨立章節「Lumi 自己避免浪費」），不做成提示。
- 匯出明細（CSV）。

**不做**（理由與替代做法見 `plan/misc-backlog.md`「#4 用量統計：先不做的項目」）

token 組成對照、依用途分布、每回合上下文圖、請求紀錄表、「所有檢查」清單、LLM 使用習慣分析、匯率設定、估算值開關、提醒門檻自選、小精靈提醒開關、每日預算、快取／階梯價的獨立欄位、用量面板依模型拆分、「下一輪約 NT$」常駐顯示。

另外（讀程式後）：舊訊息的費用不回補（升級前的訊息沒有單價快照，本月從升級那天開始算，見第 6 節的日期範圍）；圖片生成（22F）的費用不計（計價方式是每張圖，不是 token，之後再評估）；「開新對話並帶上摘要」不做（見第 7.2 節）。

## 已定案的決定（含理由）

| # | 決定 | 理由 |
|---|---|---|
| 1 | **以「每次 API 請求」為記錄單位**，對話費用是加總 | 同一則對話可以換模型；agent 迴圈一輪回應會打多次 API |
| 2 | **每筆記錄保留當時的單價快照** | 使用者之後改單價，不應改到過去的帳 |
| 3 | 內部一律存**美元**；顯示新台幣，匯率固定 32.0（不給設定，見「先不做」） | 價目表都是美元；匯率之後再開放 |
| 4 | **預算只提醒、不中斷回應**；固定 80% 與 100% 各一次，一個月一次 | 使用者決定（2026-10-29）。硬性上限請使用者到供應商後台設定 |
| 5 | 沒有回報用量的請求**用估算值計入**，畫面標「≈」 | 公司 Gateway（直通 Azure）等連線可能沒有串流 usage；計入比漏算安全 |
| 6 | 價目表用 **LiteLLM `model_prices_and_context_window.json`** 的快照，打包進 app；「更新價目表」重新下載 | 不自己維護；欄位涵蓋 input／output／cache read／cache write／context。授權 MIT（enterprise 目錄除外），Phase 0 打包前再確認一次 |
| 7 | 模糊比對**近似時不自動套用**，列出候選讓使用者選 | 名稱很像的模型價差可能很大（`gpt-4.1` 與 `gpt-4.1-mini` 差 5 倍） |
| 8 | 單價掛在**模型設定**上，不是 model ID 上 | 同一個 model ID 在不同連線（直連、Azure、OpenRouter）價格可能不同 |
| 9 | ~~本機連線預設不計費~~ → **改為使用者在模型設定自己勾選「不計費（例如地端模型）」**（2026-10-31，見決定 21） | 用量照記，費用算 0 |
| 10 | 使用習慣提示**只用規則**，不呼叫 LLM | 先做八成價值；LLM 分析列在「先不做」 |
| 10a | 提示**不顯示「預估可省」金額**，只陳述發生了什麼與可以怎麼做 | 使用者決定（2026-10-30）：建議不一定能達成，金額會變成無法兌現的承諾 |
| 10b | 提示**在對話裡、行為發生的當下**出現；用量頁只列「這個月出現過哪些情況」 | 使用者決定（2026-10-30）：當下看到才改得了習慣 |
| 10c | 9 條提示**全部納入計畫**；實作時判斷難易，做不到的條目在實作紀錄說明後延後 | 使用者決定（2026-10-30） |
| 10d | 每次觸發都記錄（含使用者關掉的），門檻集中成常數，之後用實際資料校準 | 門檻是初始值，不是業界標準 |
| 11 | UI 採第二版精簡稿 | 第一版太像管理後台，違反 ui-guide「一般使用者優先」「不做成管理控制台」 |
| 12 | **費用紀錄用獨立的 `request_usage` 表**（做法 B），`conversation_id` 不設外鍵 | 協調者讀程式後決定（2026-10-30）：重新產生會從尾端刪訊息、刪除對話會 cascade，費用掛在 `messages` 上會讓已經花的錢消失 |
| 13 | 價目表**整份原檔打包**（embedded resource），載入時才挑需要的欄位；不在建置時裁切 | 協調者決定：省掉一個要連網的建置步驟；檔案約 1～2 MB，相對發佈檔（含 Playwright 約 100 MB）可忽略。「更新」下載同一個檔案，走同一份解析程式 |
| 14 | 已經存在的模型設定：**升級後第一次啟動自動比對一次**，完全相符／正規化後相符的直接帶入單價（不動上下文上限與最大輸出） | 否則升級後所有模型都是「未設定單價」，本月數字是 0 |
| 15 | 預算存在 `AppSettings.Usage`；「這個月已提醒到哪個門檻」存在 `app_meta`（key `budget_alerted`，值如 `2026-10:80`） | 提醒狀態不是使用者設定，寫入時不該觸發設定頁重載（`Changed`） |
| 16 | `stream_options` 被拒的連線只在**本次執行期間**記住（記憶體），不存檔 | 每次啟動最多多一次 400，換來不用新增設定欄位；連線換成支援的版本後自動恢復 |
| 17 | 提示的位置統一用**輸入框上方提示列**（第 7 條原本寫「那則回覆下方」也改用它），只有第 1 條用模型選單內的確認區 | 協調者決定：少一種位置、少一個元件；提示列就在對話裡、當下出現，符合 10b |
| 18 | 第 4、5 條提示的動作是「整理並繼續」「開新對話」，不是「移除」 | 使用者同意（2026-10-30）：送出過的內容已在前文，從範圍移除不會讓之後的請求變小 |
| 19 | 設定首頁既有的「本月用量」卡片**加上金額與預算**（大數字改成新台幣、預算列與進度條、「管理 ›」連到用量頁）；輸入／輸出 token 的小標籤保留 | 使用者決定（2026-10-30），部分推翻 M21「首頁不放 US$ 與預算」：顯示的是新台幣，不是 US$ |
| 21 | **不從網址判斷「本機」**；免費由使用者在模型設定勾選「不計費」（`ModelPricing.IsFree`，存檔後 `Source = Confirmed`），拿掉 `LocalConnection`、`PriceMatchKind.Local` 與自動帶入時的免費判斷 | 使用者指出（2026-10-31）：地端與雲端模型都整合在同一個 AI Gateway 後面，同一個網址下兩種都有，程式分不出來；`ChatFailureText.IsLocalTarget` 原本是 M9 9C 用來決定錯誤訊息要不要補「看起來沒有網路」，借來當計費判斷本來就不合適 |
| 20 | 預算提醒文字採用協調者擬的版本（`Budget_Alert80Format`、`Budget_Alert100Format`、`Sprite_Budget80`、`Sprite_Budget100`） | 使用者沒有提供畫板原文（2026-10-30）；之後要改只動 `Strings.resx` |

### 已被後續變更取代的討論

- 討論時提過「小精靈和工作台共用對話，小精靈的短問答要付整段上下文的錢 → 組請求時改送摘要檢查點＋最近 K 回合」。**M22 已改成小精靈有自己的對話**（`SpriteConversationService`），這個問題不存在，不用做。

## 設計

### 1. 現況（2026-10-30 讀程式碼確認）

- `TokenUsage(InputTokens, OutputTokens, CacheCreationInputTokens, CacheReadInputTokens)`：`InputTokens` **不含**快取，和 Anthropic 一致。
- `OpenAIStreamAccumulator.ParseUsage` 已經把 OpenAI 的 `prompt_tokens` 減掉 `cached_tokens`（也處理 DeepSeek 的 `prompt_cache_hit_tokens`）；沒有回 usage 時訊息的 `Usage` 是 null。
- `OpenAICompatibleChatProvider.BuildRequest` 已經送 `stream_options = { include_usage = true }`；HTTP 400／413／422 時讀錯誤內文交給 `OpenAIHttpErrors.FromResponse`。
- `messages` 表每則訊息有 `model_id` 與四個 token 欄位；`conversation_compactions` 表記摘要那次呼叫的用量與 `model_id`。
- **工具迴圈中每次 API 回應各自是一則 assistant 訊息**（`StreamOnceAsync` 收到 `Completed` 就 `AppendMessageAsync`）。但「每則 assistant 訊息」≠「每次請求」：失敗的請求不存、停止的請求存成 `Stopped` 但沒有 usage、重新產生會刪掉舊的。→ 決定 12。
- **呼叫供應商的地方只有兩處**：`ConversationService.StreamOnceAsync`（對話）與 `CompactSummaryAsync`（摘要）。標題是本機截字，排程不呼叫模型。
- 本對話的 token 累計在 `ConversationUsageTally`（App），從訊息加總；設定首頁的本月 token 在 `IConversationStore.GetTokenTotalsAsync`。**這兩者維持不變**（token 數字照舊從訊息算），費用另外從 `request_usage` 算。
- 上下文圓環的等級門檻：`UsageLevelRules.WarningThreshold = 0.60`、`DangerThreshold = 0.85`。
- 小精靈氣泡有 `SpriteBubbleViewModel.ShowNotice(string)`，停留時間 `SpriteShellOptions.BubbleLinger = 8 秒`。
- 新增模型的預設值在 `ModelsViewModel.DefaultsFor(ProviderFormat)`（註解寫「M2 不內建模型規格表」，這次改成有價目表時用價目表）。
- 設定分類由 `SettingsCategory` 列舉定義順序（M20），目前 0 首頁～10 資料與隱私。
- 沒有長條圖圖示（`LumiIcon*` 只有 37 個，沒有 chart）→ Phase 0 新增。

### 2. 資料

**單價**（`ModelSettings` 的 init 屬性 `Pricing`，舊設定檔沒有時是 `ModelPricing.None`；型別見 Phase 0）

| 欄位 | 型別 | 說明 |
|---|---|---|
| `InputPerMTok` | `decimal?` | US$／每百萬 tokens；null = 未設定單價 |
| `OutputPerMTok` | `decimal?` | 思考 tokens 也用這個價 |
| `CacheReadPerMTok` | `decimal?` | null 時以輸入價計算 |
| `CacheWritePerMTok` | `decimal?` | null 時以輸入價計算；Anthropic 取 5 分鐘的價（LiteLLM 的 `cache_creation_input_token_cost` 就是 5 分鐘價） |
| `IsFree` | `bool` | 本機模型；true 時費用一律 0 |
| `Source` | `PriceSource` | `None`（未設定）／`Suggested`（自動帶入、完全相符）／`Confirmed`（使用者選了近似候選或自己填） |

**每次請求的費用紀錄**：新表 `request_usage`（migration 12，每次 API 呼叫一列）

```sql
CREATE TABLE request_usage (
  id                          TEXT PRIMARY KEY,
  created_at                  TEXT NOT NULL,          -- UTC，固定格式（和其他表相同）
  conversation_id             TEXT,                   -- 刻意不設外鍵：刪除對話不刪費用
  purpose                     TEXT NOT NULL,          -- 'chat' | 'compaction'
  mode                        TEXT NOT NULL,          -- 'workbench' | 'sprite'
  model_setting_id            TEXT,
  model_id                    TEXT NOT NULL,
  connection_name             TEXT NOT NULL,
  input_tokens                INTEGER NOT NULL,
  output_tokens               INTEGER NOT NULL,
  cache_creation_input_tokens INTEGER NOT NULL,
  cache_read_input_tokens     INTEGER NOT NULL,
  usage_estimated             INTEGER NOT NULL DEFAULT 0,
  price_snapshot_json         TEXT,                   -- null = 當時未設定單價
  cost_usd                    REAL,                   -- null = 未設定單價（不計入預算）
  tool_definition_tokens      INTEGER NOT NULL DEFAULT 0  -- 估算，給第 6 條提示與日後分析
);
CREATE INDEX ix_request_usage_created ON request_usage(created_at);
CREATE INDEX ix_request_usage_conversation ON request_usage(conversation_id, created_at);
```

`price_snapshot_json` 存 `PriceSnapshot`（四個單價都已解析成數字＋`IsFree`）。`cost_usd` 寫入時算好：

```
cost = (input × InputPrice + output × OutputPrice
        + cacheRead × (CacheReadPrice ?? InputPrice)
        + cacheWrite × (CacheWritePrice ?? InputPrice)) / 1,000,000
```

單價未設定（`InputPerMTok` 或 `OutputPerMTok` 為 null，且不是 `IsFree`）時 `price_snapshot_json` 與 `cost_usd` 都存 null，畫面顯示「未設定單價」，不計入預算。

**預算**（`AppSettings.Usage`，型別 `UsageSettings`）：`MonthlyBudgetUsd decimal?`（null = 不設預算）、`MutedHints`（被「不再提醒這類」的規則 id）。UI 輸入新台幣，存的時候除以 32.0。已提醒的門檻存在 `app_meta`（決定 15）。

**提示紀錄**：新表 `usage_hints`（同在 migration 12）

```sql
CREATE TABLE usage_hints (
  id              TEXT PRIMARY KEY,
  rule            TEXT NOT NULL,      -- 第 7.2 節的 id
  conversation_id TEXT,               -- 不設外鍵（同 request_usage）
  created_at      TEXT NOT NULL,
  outcome         TEXT NOT NULL       -- 'shown' | 'dismissed' | 'muted' | 'acted' | 'suppressed'
);
CREATE INDEX ix_usage_hints_created ON usage_hints(created_at);
CREATE INDEX ix_usage_hints_rule ON usage_hints(rule, conversation_id);
```

`shown` 先寫一列；使用者按了按鈕後把**同一列**改成 `dismissed`／`muted`／`acted`。

### 3. 估算（usage 缺漏時）

在 `StreamOnceAsync` 裡追蹤：最後一個 `UsageEvent`（`lastUsage`）、收到的文字與思考（`receivedText`）、是否收到過任何事件（`anyEvent`）。結束時依下表記錄：

| 結束方式 | 記不記 | tokens | `usage_estimated` |
|---|---|---|---|
| `Completed`，訊息有 `Usage` | 記 | 訊息的 `Usage` | 0 |
| `Completed`，訊息沒有 `Usage`，有 `lastUsage` | 記 | `lastUsage` | 0 |
| `Completed`，兩者都沒有 | 記 | 估算輸入＋估算輸出 | 1 |
| 停止（取消）或 `Failed`，且 `anyEvent` | 記 | 輸入：`lastUsage` 有值用它，沒有用估算；輸出：`max(lastUsage.Output, 估算(receivedText))` | 1 |
| 停止或 `Failed`，什麼事件都沒收到（例如 HTTP 401、連不上） | **不記** | — | — |

- 估算輸入 = `ContextEstimator.EstimateText(request.System)` ＋ `ContextEstimator.EstimateText(ToolSnapshotJson.Serialize(request.Tools))`（有工具時）＋ `request.Messages.Sum(ContextEstimator.EstimateMessage)`。估算輸出 = `ContextEstimator.EstimateText(receivedText)`。估算時快取讀寫都是 0（保守：全部當未快取輸入）。
- 摘要呼叫（`CompactSummaryAsync`）用同一套規則，`purpose = compaction`。
- 相容服務回 **400 且錯誤內文含 `stream_options`**（不分大小寫）時：拿掉 `stream_options` 自動重送一次，並記住這個連線 id（決定 16），之後同一次執行都不帶。（Azure 舊版 api-version 會回 `Unknown parameter: 'stream_options'`；新版 v1 API 支援。）
- 公司 Gateway 實測步驟（人工驗收）：非串流確認有 `usage`；串流加 `stream_options` 確認 `[DONE]` 前有 usage chunk；同一個超過 1,024 tokens 的 prompt 打兩次，第二次 `cached_tokens > 0`。

### 4. 價目表與比對

- 打包：LiteLLM 原檔放 `src/Lumi.Web/Pricing/litellm-model-prices.json`（EmbeddedResource），旁邊 `litellm-model-prices.date.txt` 記快照日期（`yyyy-MM-dd`）。
- 解析：只讀 `input_cost_per_token`、`output_cost_per_token`、`cache_read_input_token_cost`、`cache_creation_input_token_cost`、`max_input_tokens`、`max_output_tokens`、`litellm_provider`、`mode`。只收 `mode` 是 `chat`（或沒有 `mode`）且有輸入與輸出價的條目；`sample_spec` 這一筆略過。任何一筆讀不懂就略過那一筆，不讓整份失敗。
- 單位換算：LiteLLM 是「每 token 美元」，乘 1,000,000 成「每百萬 tokens」，用 `decimal` 計算後四捨五入到小數 4 位。
- 「更新價目表」：從 `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json` 下載（走 `IWebClient`，所以跟 Proxy 設定；`MaxBytes` 8 MB），解析成功且至少 100 筆才覆蓋到 `LumiPaths.DataDirectory/model-prices.json`，並把今天的日期寫進 `model-prices.date.txt`；失敗就沿用目前的版本。啟動時使用者資料夾有檔案且解析成功就用它，否則用內建版。
- 比對順序（找到就停）：
  1. **完全相符**（不分大小寫）。
  2. **正規化後相符**：雙方都去掉供應商前綴（第一個 `/` 之前的部分，例如 `anthropic/`、`openai/`、`azure/`、`openrouter/anthropic/` 取最後一段）、日期後綴（正規表示式 `-\d{8}$`、`-\d{4}-\d{2}-\d{2}$`）、`:latest`、Bedrock 區域前綴（`us.`、`eu.`、`apac.`、`global.`）與版本後綴（`-v\d+(:\d+)?$`），再比對。多筆相符時取「供應商符合連線」的那筆（第 4 點），其次取鍵名最短的。
  3. **前綴相符**：以 `-` 為邊界，取最長的相符前綴（例如 `gpt-4o-prod` → `gpt-4o`）。
  4. **近似**：Jaro-Winkler ≥ 0.85，取前 3 名。
- 第 1、2 步的結果自動帶入，`Source = Suggested`，畫面顯示「已自動帶入」。第 3、4 步只列候選（第 3 步的結果排第一），**不自動套用**。
- 連線是 Azure（Base URL 的主機結尾是 `.openai.azure.com`、`.services.ai.azure.com` 或 `.cognitiveservices.azure.com`）時，`litellm_provider` 是 `azure` 的條目優先；連線格式是 Anthropic 時 `anthropic` 優先；其他 OpenAI 相容連線 `openai` 優先。公司 Gateway 的網址判斷不出來，照一般 OpenAI 相容處理（候選照樣列出）。
- 本機連線：`IsFree = true`，不比對。
- 上下文上限與最大輸出：**新增模型時**比對到（第 1、2 步）就一起帶入表單；編輯既有模型時不動這兩欄（決定 14）。

### 5. 預算提醒

- 每次 `RunFinished`（回應結束：完成、失敗、停止）之後，計算本月（本地時區的月份；月初 00:00 轉 UTC 查詢）`cost_usd` 加總。
- 跨過 80% 或 100%，而且本月還沒提醒過這個門檻 → 發出提醒，寫 `app_meta`。兩個門檻一起跨過時只提醒 100%。已提醒 100% 之後不會再提醒 80%。月份變了（`budget_alerted` 的月份不是本月）就當作沒提醒過。
- 沒設預算、或預算是 0 以下：不提醒。
- 只在回應結束後出現，不在回應中途。手動「整理並繼續」沒有 `RunFinished`，不檢查（下一次回應結束時會檢查到）。
- 工作台模式：訊息流最上方（`UpdateBanner` 與 `NotificationStrip` 那一層）一條提醒列，80% 用 Warning 樣式、100% 用 Danger 樣式，按鈕「查看用量」（開設定視窗到「用量與費用」）「知道了」。
- 小精靈模式：小精靈氣泡 `ShowNotice` 說一句，停留時間照 `BubbleLinger`（8 秒）。
- 設定首頁的「本月用量」卡片加上金額與預算（決定 19，畫面見第 6 節「設定首頁」）。

### 6. 畫面（照畫板 v2-1～v2-3）

**設定 › 用量與費用**（新分類，Menu 放在「AI 連線」之後；圖示用新增的 `LumiIconBarChart`；副標「花了多少、預算」）

1. 頁首（`SettingsPageHeader`）：eyebrow「用量與費用」、標題「這個月花了多少」。
2. 本月卡片：大數字 `NT$488`（有估算的請求時前面加「≈」）、`10/1–10/29 · 約 US$15.25`（起日 = 本月 1 日與本月第一筆紀錄取較晚者，迄日 = 今天）；預算列「每月預算 NT$800 · 已用 61%」＋「編輯」；進度條（< 80% `LumiAccentBrush`、80～99% `LumiWarningIconBrush`、≥ 100% `LumiDangerBrush`，同上下文圓環）；預測句「照目前的速度，月底約 NT$920，會超過預算。」（本月有紀錄的天數 ≥ 7 天才顯示；預測 = 本月累計 ÷ 已過天數 × 當月天數；沒超過預算時用「照目前的速度，月底約 NT$620。」；沒設預算時也顯示不帶後半句的版本）。有未設定單價的請求時，卡片底部一行 muted 文字「有 3 次請求的模型未設定單價，沒有計入。」＋「單價設定 ›」。
   - 「編輯」原地展開：金額輸入（新台幣整數）、取消／儲存，說明「用到 80% 和 100% 時各提醒一次。Lumi 只會提醒，不會停止回應。」，下方「不設預算」。
   - 沒設預算時：只顯示「設定每月預算」連結（按了一樣原地展開）。
3. 每天：本月每日費用長條圖（1 日到月底每天一根；今天用 `LumiAccentHoverBrush`；其他日用 `LumiAccentBrush`；0 用 `LumiBorderBrush` 的 2px 短條；未來的日子不畫）。滑鼠停留 ToolTip「10/29 · NT$32」。
4. 花最多的對話：本月前 5 則，標題、`10/29 · 12 回合`、費用、›；點擊開啟該對話並回到工作台。有估算的對話費用前加「≈」。已刪除的對話不列。
5. 這個月出現過的情況：見 7.3。沒有任何紀錄就整段不顯示。
6. 頁尾：「金額依各模型的單價估算，實際以供應商帳單為準。」、「單價設定 ›」（跳 AI 連線頁）、「匯出明細（CSV）」。
7. 本月沒有任何紀錄：卡片只顯示 `NT$0` 與「這個月還沒有用量。」，第 3～5 段不顯示。

**設定首頁 › 本月用量卡片**（`HomePanel.axaml` 的 `HomeUsageCard`，決定 19）

- 卡片標題列右邊加「管理 ›」（和「工具」卡片相同的寫法），開到「用量與費用」。
- 大數字從 tokens 改成本月金額 `NT$488`（有估算時 `≈NT$488`），後面不再接「tokens」。
- 有預算時，大數字下方一行 muted「每月預算 NT$800 · 已用 61%」＋ 4px 進度條（顏色規則同用量頁）；沒設預算時不顯示這兩樣。
- 「今天 …」那行改成今天的金額（`今天 NT$32`）。
- 輸入／輸出的小標籤（tokens）不變。
- 讀不到時照舊顯示「暫時讀不到用量」。卡片的 XAML 註解「沒有金額與預算、沒有連結」一併改掉。

**AI 連線 › 模型設定**（既有表單加「單價」段落，放在「上下文上限」那一組之後）

- 完全相符：一行「輸入 3.00 · 輸出 15.00 · US$／每百萬 tokens · ✓ 已自動帶入 · 編輯」。使用者自己改過（`Confirmed`）時不顯示「✓ 已自動帶入」。
- 「編輯」展開：輸入、輸出兩欄；「進階：快取價格」收合快取讀取／寫入（說明「留空時以輸入價計算。」）；「還原自動帶入的單價」（只有比對得到時顯示）。
- 近似：警示框「價目表裡沒有「xxx」，這幾個最像。選一個，或自己填：」＋候選單選（model ID、`輸入 3.00 · 輸出 15.00`）＋「套用」「自己填」。
- 找不到：直接顯示空欄位，說明「價目表裡找不到這個模型。先留空也可以，費用會顯示「未設定單價」。」
- 不計費：單價段落最上方有「不計費（例如地端模型）」勾選；勾了就不比對價目表、隱藏單價欄位，只顯示一行「不計費 · 用量照常記錄，費用算 0」。取消勾選會重新比對價目表。
- 段落底部說明：「只用來估算費用。改了只影響之後的請求。」；右邊「價目表日期：2026-10-15 · 更新價目表」。
- 改了 model ID 或連線時重新比對（使用者自己填的 `Confirmed` 單價不覆蓋，只在 model ID 改變時才重比）。

**工作台用量面板**（`UsageFlyout.axaml`，在三個小方塊與連線列之間加一段，上方 1px Border 分隔）

- 「本對話」右側 `NT$71`（有估算時 `≈NT$71`；整則對話都未設定單價時「未設定單價」）
- 「本月」右側 `NT$488 ／ 800`；下方 4px 進度條（顏色規則同用量頁）；右下「用量與費用 ›」開設定視窗到該分類
- 沒設預算時第二行只顯示 `NT$488`、不畫進度條
- 說明文字（`Usage_CostNote`）改為「金額是估算；實際費用由供應商計算，從你的帳號扣款。」
- 數字在面板打開時與每次 `RunFinished` 後重新查詢（不在串流中即時更新：費用要等請求結束才寫入）。

數字格式（`Money`）：新台幣取整數、千分位（< NT$10 時留一位小數，0 顯示 `NT$0`）；美元留兩位。

### 7. 使用習慣提示

#### 7.1 共通規則

- **不在回應中途出現**：回應結束（`RunFinished`）後，或使用者正要做某個動作時（換模型、回到舊對話後點輸入框）才出現。
- **同一時間只顯示一則**；多則同時符合時依下表「優先」數字小的先，其餘這次記 `suppressed`。
- **頻率**：除非下表另有寫明，同一則對話、同一條規則只提示一次（以 `usage_hints` 有沒有這則對話、這條規則的 `shown` 以後的紀錄判斷，重開 Lumi 也算數）。
- 每則提示都有「知道了」與「不再提醒這類」。後者寫進 `AppSettings.Usage.MutedHints`，用量頁可以重新開啟。
- **不顯示金額**，只寫發生了什麼、可以怎麼做。數字只用 tokens、回合數、次數、時間這類事實；tokens 用 `92K` 這種寫法（≥ 1,000 時除以 1,000 取整數加 K）。
- **小精靈模式不顯示提示**，只記錄（`suppressed`）。
- **位置**（決定 17）：
  - **輸入框上方提示列** `ComposerHint`：一行文字＋最多兩個動作按鈕＋「知道了」「不再提醒這類」；樣式同用量面板的快滿提醒（`LumiWarningBackgroundBrush`／`LumiWarningBorderBrush`，圓角 10、內距 12、文字 `LumiWarningTextBrush` 13px）。
  - **確認區**：模型選單內，選了別的模型之後、真正換之前出現（第 1 條）。
- 送出新訊息、切換對話、開新對話時，目前顯示的提示自動收起（記 `dismissed`）。
- 門檻集中在 `UsageHintThresholds`（Phase 0），註明是初始值。

#### 7.2 九條規則

| 優先 | id | 行為 | 觸發條件（初始值） | 時機與位置 | 提示文字 | 動作按鈕 |
|---|---|---|---|---|---|---|
| 1 | `model-switch` | 對話中途換模型 | 使用者在模型選單選了別的模型，且這則對話目前上下文（`ContextUsed`）≥ 20K | **選下去之前**，模型選單內的確認區 | 「這則對話已有 92K 的前文。換模型後，這些前文要用全價重新送一次。」 | 「開新對話再換」「照樣換」 |
| 2 | `long-conversation` | 長對話一直聊下去 | 上下文 ≥ 上限的 60%（圓環變橘色），或回合數 ≥ 40 且最近 10 則助理訊息平均輸入（含快取）≥ 60K | 回應結束後，輸入框上方 | 「這則對話的前文已經很長，之後每問一句都會一起送出。換話題時，開新對話比較省。」 | 「開新對話」 |
| 3 | `stale-resume` | 隔很久回到長對話 | 打開或切回一則對話，距離它最後一則助理訊息 ≥ 60 分鐘，且上下文 ≥ 50K | **輸入框取得焦點時**（還沒送出），輸入框上方；每次打開這則對話最多一次（這條不受「同一則對話只一次」限制） | 「上次是 3 小時前。快取已經過期，下一句會把 92K 的前文用全價重新送一次。」 | 「開新對話」「繼續」 |
| 4 | `big-file` | 大檔案留在前文 | 前文裡（還沒被摘要整理掉的）單一上下文檔案（`ContextBlock`）≥ 15K tokens，且它之後已有 ≥ 20 則助理訊息 | 回應結束後，輸入框上方；同一個檔案只提示一次 | 「「ScheduleHistory.dc.html」（37K tokens）在這則對話的前文裡，之後每一輪都會一起送出。用完了的話，可以整理對話或開新對話。」 | 「整理並繼續」「開新對話」 |
| 5 | `images` | 圖片留在前文 | 前文裡（還沒被摘要整理掉的）使用者附上的圖片合計 ≥ 3 張或 ≥ 5K tokens（每張以 `ContextEstimator.ImageTokens` 估），且最早那張之後已有 ≥ 10 則助理訊息 | 回應結束後，輸入框上方 | 「這則對話的前文裡有 4 張圖片，之後每一輪都會重新送出。看完了的話，可以整理對話或開新對話。」 | 「整理並繼續」「開新對話」 |
| 6 | `mcp-tools` | 啟用太多工具 | 這次請求的 `tool_definition_tokens` ≥ 10K | 新對話的第一次回應結束後，輸入框上方；**每 7 天最多一次**（跨對話）。設定 › 工具 › MCP 伺服器另外常駐顯示各伺服器的工具說明大小 | 「目前啟用的工具說明約 14K tokens，每次請求都會送出。用不到的 MCP 伺服器可以先關掉。」 | 「管理工具」 |
| 7 | `regenerate` | 一直重新產生 | 同一則使用者訊息的回覆第 3 次按「重新產生」（計數只在記憶體，重開 Lumi 歸零） | 第 3 次重新產生的回應結束後，輸入框上方 | 「已經重新產生 3 次了。補充你想要的方向（例如長度、格式、重點），通常比再試一次有效。」 | — |
| 8 | `long-output` | 要求整份重寫 | 同一則對話中，有 3 則助理訊息輸出 ≥ 4K tokens 且文字含 ```` ``` ```` | 第 3 則的回應結束後，輸入框上方 | 「最近幾次回覆都很長，輸出是最貴的部分。改程式時可以請 Lumi 只給修改的地方。」 | — |
| 9 | `repeat-paste` | 同一份內容反覆貼上 | 剛送出的使用者訊息裡有 ≥ 2K tokens 的段落（以空白行分段），和這則對話先前使用者訊息的某段完全相同（空白正規化後比對 SHA-256） | 這次回應結束後，輸入框上方 | 「這段內容在第 12 回合已經貼過，前文裡就有，下次可以直接說「剛才那段」。」 | — |

- 第 4、5 條的動作從「移除」改成「整理並繼續」（呼叫既有的 `CompactAndContinueCommand`）與「開新對話」：理由見可行性評估第 2 點（決定 18）。
- 「開新對話」：按鈕只開空白新對話（「開新對話並帶上摘要」不做，可行性評估已排除）。第 2、3 條的文字已經說明了「為什麼」，不另外加「把重點貼過去」。
- 第 1 條「開新對話再換」：先開新對話，再換模型。
- 第 3 條「繼續」= 知道了（記 `acted`）。
- 第 6 條「管理工具」：開設定視窗到「工具」分類的 MCP 分頁。

#### 7.3 用量頁的「這個月出現過的情況」

一列一條本月有紀錄（任何 `outcome` 都算）的規則：規則名稱、「N 則對話」、一句說明（字串表的 `Hint_*Summary`）。點擊展開對話清單（標題、日期），點對話開啟它。被設為「不再提醒」的規則在說明後多「已關閉提醒 · 重新開啟」。**不顯示金額**。

### 8. 匯出明細（CSV）

一次請求一列，本月（從用量頁匯出就是本月）：時間（本地時間 `yyyy-MM-dd HH:mm:ss`）、對話標題（已刪除的對話寫「（已刪除的對話）」）、用途（`對話`／`整理`）、模式（`工作台`／`小精靈`）、連線、model ID、未快取輸入、快取讀取、快取寫入、輸出、是否估算（`是`／空白）、單價快照（`輸入/輸出/快取讀/快取寫` 四個數字用 `/` 分隔，免費寫 `免費`，未設定寫空白）、費用（US$，小數 6 位；未設定單價空白）。UTF-8 with BOM（Excel 開啟中文不亂碼），RFC 4180 引號規則。檔名 `Lumi-用量-2026-10.csv`，用 `FilePickerService.SaveFileAsync` 讓使用者選位置。

## Lumi 自己避免浪費（不做成提示）

| # | 項目 | 現況（2026-10-30 確認） | 要做的 |
|---|---|---|---|
| L1 | 工具結果大小 | `ToolLoopOptions.MaxResultChars = 20_000`，`ToolExecutor.Truncate` 只留開頭並附「[內容過長，已截斷]」 | 改成保留**開頭 70%、結尾 30%**（build log 的錯誤常在最後）；中間換成「[中間省略了 {N} 行；需要的話可以用讀檔工具指定行範圍再讀]」。20,000 字元先不改，之後用 `request_usage` 的資料校準 |
| L2 | 工具迴圈上限 | `MaxIterations = 10`，超過回 `ToolLoopLimit` | 同一個工具、相同參數（`InputJson` 去空白後相同）**連續 3 次**都 `IsError` 就提早停下，回 `ChatFailureKind.ToolLoopLimit`，訊息「「{工具顯示名稱}」連續 3 次失敗，先停下來了。最後一次的錯誤：{Summary}。可以補充說明後再試。」，`Retryable: true` |
| L3 | 快取前綴穩定 | **已成立**：system prompt 與工具定義每則對話凍結（migration 8、9），中途變動附在訊息尾端；`PromptComposer` 不含時間日期 | 加一個測試鎖住：同一則對話連續兩次請求的 `System` 與 `Tools` 完全相同（含中途改人設與記憶的情況）。不加 `prefix_hash` 欄位 |
| L4 | 工具定義只送需要的 | **已成立**：新對話只帶已啟用的工具（M15 D3） | 不用做。「工具很多時先送摘要、需要時再展開」列入評估，不在本計畫實作 |
| L5 | 圖片縮小再送 | 沒有縮圖：`ContextAttachmentBuilder.BuildImageAsync`（`Lumi.Tools`）原檔照存 | 加入上下文的圖片在存成附件前，長邊超過 1568px 就等比縮到 1568px（Anthropic 建議值；超過會被伺服器縮小但照樣計費）；PNG 維持 PNG，其他轉 JPEG 品質 85。縮圖由 App 層實作（Avalonia 已帶 SkiaSharp），經 Core 介面注入。GIF 不處理（可能是動畫）。工具產生的截圖不處理（瀏覽器視窗本來就小於上限） |
| L6 | 壓縮用同一個模型 | `CompactSummaryAsync` 用目前模型（關閉思考） | 維持；在該方法的註解補一句理由：同一個模型才能吃到前文的快取，改用便宜模型反而全價重送 |
| L7 | 記憶注入大小 | 記憶總字數上限 `MemoryOptions.MaxTotalLength = 2000` 字，中文約 2,000 tokens | 夠小，不用做。位置在 system prompt（凍結），中途變動走 `SystemPromptUpdate`，已穩定 |
| L8 | 背景呼叫精簡 | 標題是本機截字，沒有背景呼叫 | 不用做 |

- L1、L2 的觸發寫進 log（`Information` 等級，不含內容），方便開發時檢查。

## 實作分包

通則同 M22：照 [[subagent-worktrees-manual]] 先 commit Phase 0，再 `git worktree add -b w23<包> ../DesktopAgent-wt/w23<包> HEAD`；子代理不 commit、不改 `plan/`、**不新增字串**（全部在 Phase 0 加好；缺就停下來回報）、只用既有 token 與樣式 class、建置零警告、遵守間距尺度（0/4/8/12/16/24/32）。**遇到計畫沒寫到、或和程式現況不符的地方，停下來在回報裡說明，不要自己猜。**

回報格式：改了哪些檔案、新增測試數量與結果、和計畫不同的地方與原因、沒做完或卡住的項目。

| 包 | 內容 | Wave | 前置 |
|---|---|---|---|
| Phase 0 | Core 型別與介面、設定欄位、migration 12、字串、圖示、價目表檔案、測試用 in-memory store | — | — |
| 23A | 價目表解析、比對、更新下載 | 1 | Phase 0 |
| 23B | 費用紀錄：`UsageRecorder`、`SqliteUsageStore`、`ConversationService` 兩個呼叫點、估算、L2 | 1 | Phase 0 |
| 23C | OpenAI 相容 `stream_options` 退回、L1 工具結果頭尾截斷 | 1 | Phase 0 |
| 23D | 月報計算與預算判斷（純邏輯）、`Money` 格式 | 1 | Phase 0 |
| 23E | L5 圖片縮小 | 1 | Phase 0 |
| 23F | 模型設定的單價段落、舊模型自動帶入 | 2 | 23A |
| 23G | 設定 › 用量與費用頁、CSV 匯出 | 2 | 23B、23D |
| 23H | 工作台用量面板兩行、預算提醒（工作台＋小精靈） | 2 | 23B、23D |
| 23L | 設定首頁「本月用量」卡片加金額與預算 | 2 | 23B、23D |
| 23I | 提示規則（純邏輯）與 `SqliteUsageHintLog` | 3 | Wave 2 合併 |
| 23J | 提示的畫面：`ComposerHint`、協調者服務、模型切換確認、重新產生計數 | 3 | 23I（可以和 23I 同時開工：只依賴 Phase 0 的介面；整合時先合 23I） |
| 23K | MCP 伺服器頁顯示工具說明大小 | 3 | Phase 0 |
| 整合 | DI、設定視窗分類、啟動時自動帶入、文件、全體測試 | — | 各 Wave 之後 |

---

### Phase 0（協調者）

1. **價目表檔案**：下載 LiteLLM 原檔到 `src/Lumi.Web/Pricing/litellm-model-prices.json`，寫 `litellm-model-prices.date.txt`（下載日期）。確認 LiteLLM 授權（MIT，只用這個 JSON、不含 enterprise 目錄），在 `README.md` 的第三方元件段落（沒有就新增）記一行來源與授權。`Lumi.Web.csproj` 把兩個檔案設成 `EmbeddedResource`。記下檔案大小與筆數到實作紀錄。
2. **Core 型別**（`src/Lumi.Core/Usage/`，命名空間 `Lumi.Core.Usage`）：

```csharp
// ModelPricing.cs
/// <summary>模型設定上的單價（US$／每百萬 tokens）。只用來估算費用。</summary>
public sealed record ModelPricing(
    decimal? InputPerMTok,
    decimal? OutputPerMTok,
    decimal? CacheReadPerMTok = null,
    decimal? CacheWritePerMTok = null,
    bool IsFree = false,
    PriceSource Source = PriceSource.None)
{
    public static readonly ModelPricing None = new(null, null);

    /// <summary>算得出費用：免費，或輸入與輸出價都有。</summary>
    public bool IsSet => IsFree || (InputPerMTok is not null && OutputPerMTok is not null);
}

public enum PriceSource { None, Suggested, Confirmed }

/// <summary>寫進 <c>request_usage.price_snapshot_json</c> 的單價：快取價已經解析（null 時用輸入價）。</summary>
public sealed record PriceSnapshot(decimal Input, decimal Output, decimal CacheRead, decimal CacheWrite, bool IsFree)
{
    /// <summary>未設定單價回 null；免費回全 0 且 <see cref="IsFree"/>。</summary>
    public static PriceSnapshot? From(ModelPricing pricing) => /* 實作 */;
}

// UsageRecord.cs
public enum UsagePurpose { Chat, Compaction }

/// <summary>一次 API 呼叫的用量與費用（<c>request_usage</c> 一列）。</summary>
public sealed record UsageRecord(
    string Id,
    DateTimeOffset CreatedAt,
    string? ConversationId,
    UsagePurpose Purpose,
    ChatMode Mode,
    string? ModelSettingsId,
    string ModelId,
    string ConnectionName,
    TokenUsage Tokens,
    bool Estimated,
    PriceSnapshot? Price,
    decimal? CostUsd,
    int ToolDefinitionTokens)
{
    public static string NewId() => Guid.CreateVersion7().ToString();
}

// IUsageStore.cs
public interface IUsageStore
{
    Task AddAsync(UsageRecord record, CancellationToken ct = default);

    /// <summary>[from, to) 之間的紀錄，依時間由舊到新（月報與匯出用；一個月的量在記憶體裡處理沒問題）。</summary>
    Task<IReadOnlyList<UsageRecord>> ListAsync(DateTimeOffset from, DateTimeOffset to, CancellationToken ct = default);

    /// <summary>一則對話全部的費用加總（含重新產生掉的那幾次）。</summary>
    Task<ConversationCost> GetConversationCostAsync(string conversationId, CancellationToken ct = default);

    /// <summary>[from, to) 之間費用最高的對話，只列還存在的對話，最多 <paramref name="limit"/> 則。</summary>
    Task<IReadOnlyList<ConversationCostItem>> ListTopConversationsAsync(DateTimeOffset from, DateTimeOffset to, int limit, CancellationToken ct = default);

    /// <summary>[from, to) 之間 cost_usd 的加總（null 不計）。預算檢查用，比 ListAsync 便宜。</summary>
    Task<decimal> SumCostAsync(DateTimeOffset from, DateTimeOffset to, CancellationToken ct = default);
}

/// <param name="CostUsd">有單價的請求加總；全部都沒有單價時是 null。</param>
public sealed record ConversationCost(decimal? CostUsd, bool HasEstimated, int UnpricedCount)
{
    public static readonly ConversationCost Empty = new(null, false, 0);
}

/// <param name="TurnCount">這則對話目前的使用者訊息數。</param>
/// <param name="LastAt">這段期間內最後一筆紀錄的時間（顯示日期用）。</param>
public sealed record ConversationCostItem(string ConversationId, string Title, DateTimeOffset LastAt, int TurnCount, decimal CostUsd, bool HasEstimated);

// IUsageRecorder.cs
/// <summary>
/// <see cref="Chat.ConversationService"/> 每次呼叫供應商結束時回報一次。實作負責算費用、寫入、吞掉自己的錯誤（記錄失敗不能影響對話）。
/// </summary>
public interface IUsageRecorder
{
    Task RecordAsync(UsageObservation observation, CancellationToken ct = default);
}

/// <param name="Model">發出請求時的模型設定（單價從這裡拍快照）。</param>
public sealed record UsageObservation(
    string? ConversationId, UsagePurpose Purpose, ChatMode Mode, ResolvedModel Model,
    TokenUsage Tokens, bool Estimated, int ToolDefinitionTokens);

public sealed class NullUsageRecorder : IUsageRecorder
{
    public Task RecordAsync(UsageObservation observation, CancellationToken ct = default) => Task.CompletedTask;
}

// UsageSettings.cs
/// <param name="MonthlyBudgetUsd">每月預算（美元）；null＝不設。</param>
/// <param name="MutedHints">被「不再提醒這類」的提示 id（<see cref="UsageHintIds"/>）。</param>
public sealed record UsageSettings(decimal? MonthlyBudgetUsd = null, IReadOnlyList<string>? MutedHints = null)
{
    public static readonly UsageSettings Default = new();

    public IReadOnlyList<string> Muted => MutedHints ?? [];

    /// <summary>預算 ≤ 0 當作不設；MutedHints 去重、只留認得的 id。</summary>
    public UsageSettings Normalize() => /* 實作 */;
}

// Money.cs
public static class Money
{
    /// <summary>固定匯率（決定 3）；之後開放設定時改這裡。</summary>
    public const decimal TwdPerUsd = 32.0m;
}

// UsageHints.cs
public static class UsageHintIds
{
    public const string ModelSwitch = "model-switch";
    public const string LongConversation = "long-conversation";
    public const string StaleResume = "stale-resume";
    public const string BigFile = "big-file";
    public const string Images = "images";
    public const string McpTools = "mcp-tools";
    public const string Regenerate = "regenerate";
    public const string LongOutput = "long-output";
    public const string RepeatPaste = "repeat-paste";

    /// <summary>依優先順序（第 7.2 節）。</summary>
    public static readonly IReadOnlyList<string> All = [ModelSwitch, LongConversation, StaleResume, BigFile, Images, McpTools, Regenerate, LongOutput, RepeatPaste];
}

/// <summary>提示的門檻。都是初始值（不是業界標準），之後用 usage_hints 與 request_usage 的實際資料校準（決定 10d）。</summary>
public static class UsageHintThresholds
{
    public const int ModelSwitchContextTokens = 20_000;
    public const double LongConversationContextRatio = 0.60;
    public const int LongConversationTurns = 40;
    public const int LongConversationRecentRequests = 10;
    public const int LongConversationAvgInputTokens = 60_000;
    public static readonly TimeSpan StaleResumeIdle = TimeSpan.FromMinutes(60);
    public const int StaleResumeContextTokens = 50_000;
    public const int BigFileTokens = 15_000;
    public const int BigFileRequestsAfter = 20;
    public const int ImagesCount = 3;
    public const int ImagesTokens = 5_000;
    public const int ImagesRequestsAfter = 10;
    public const int McpToolDefinitionTokens = 10_000;
    public static readonly TimeSpan McpToolsInterval = TimeSpan.FromDays(7);
    public const int RegenerateCount = 3;
    public const int LongOutputTokens = 4_000;
    public const int LongOutputCount = 3;
    public const int RepeatPasteTokens = 2_000;
}

public enum UsageHintOutcome { Shown, Dismissed, Muted, Acted, Suppressed }

public sealed record UsageHintEntry(string Id, string Rule, string? ConversationId, DateTimeOffset CreatedAt, UsageHintOutcome Outcome);

public interface IUsageHintLog
{
    Task AddAsync(UsageHintEntry entry, CancellationToken ct = default);
    Task SetOutcomeAsync(string id, UsageHintOutcome outcome, CancellationToken ct = default);
    /// <summary>這則對話、這條規則有沒有 Shown 以後的紀錄（Suppressed 不算）。</summary>
    Task<bool> WasShownAsync(string rule, string conversationId, CancellationToken ct = default);
    /// <summary>這條規則最近一次 Shown 以後紀錄的時間；沒有回 null（第 6 條的 7 天限制用）。</summary>
    Task<DateTimeOffset?> LastShownAsync(string rule, CancellationToken ct = default);
    /// <summary>[from, to) 之間的紀錄（用量頁 7.3）。</summary>
    Task<IReadOnlyList<UsageHintEntry>> ListAsync(DateTimeOffset from, DateTimeOffset to, CancellationToken ct = default);
}

// IImageDownscaler.cs（L5）
public interface IImageDownscaler
{
    /// <summary>長邊超過 <paramref name="maxEdge"/> 就等比縮小，回新的位元組與媒體類型；不需要縮或無法處理時回 null（呼叫端用原圖）。</summary>
    DownscaledImage? Downscale(byte[] data, string mediaType, int maxEdge);
}
public sealed record DownscaledImage(byte[] Data, string MediaType);
```

   `PriceSnapshot.From` 與 `UsageSettings.Normalize` 由協調者實作（各十行內），附 `tests/Lumi.Core.Tests/Usage/UsageContractTests.cs` 測試。

3. **設定欄位**：`ModelSettings` 加 `public ModelPricing Pricing { get; init; } = ModelPricing.None;`（不放建構式參數）；`AppSettings` 加 `public UsageSettings Usage { get; init; } = UsageSettings.Default;`；`IModelCatalog.SetUsageAsync(UsageSettings settings, CancellationToken ct = default)`，預設實作丟 `NotSupportedException`（照 `SetBehaviorAsync`），`JsonModelCatalog` 實作（照 `SetBehaviorAsync`：`MutateAsync(s => s with { Usage = settings.Normalize() }, ct)`），`JsonModelCatalog.Normalize` 加 `Usage = settings.Usage.Normalize()`。確認舊設定檔讀得進來、`decimal?` 能來回存讀（`DefaultSettingsTests` 或 `JsonModelCatalog` 既有測試加一個案例）。
4. **migration 12**：`SqliteSchema.Migrations` 最後加第 2 節的兩張表（同一個字串），註解「12：每次 API 請求的用量與費用、使用習慣提示紀錄（M23）。conversation_id 刻意不設外鍵：刪除對話不刪費用。」
5. **測試替身**：`tests/Lumi.Core.Tests/TestSupport/InMemoryUsageStore.cs`、`InMemoryUsageHintLog.cs`（實作 Phase 0 介面，照 `InMemoryConversationStore` 的寫法；`ListTopConversationsAsync` 回空清單即可，測試需要時再補）。App 測試需要的話照樣複製一份到 `tests/Lumi.App.Tests/TestSupport/`。
6. **設定導覽**：`ISettingsWindowService.cs` 的 `SettingsSection` 加 `Usage`（23G、23H、23L 都用它開到用量頁；`SettingsViewModel` 裡實際切頁的處理在整合包加，Phase 0 先讓它落到首頁也可以，只要建置通過）。
7. **圖示**：`LumiIconBarChart`（Lucide `bar-chart-3` 的 path），加在既有圖示資源檔，照其他 `LumiIcon*` 的格式。
8. **字串**（`Strings.resx`＋`Strings.cs`，鍵名與文字照下表；`{0}` 是格式參數）：

| 鍵 | 文字 |
|---|---|
| `Settings_CategoryUsage` | 用量與費用 |
| `Settings_CategoryUsageHint` | 花了多少、預算 |
| `UsagePage_Title` | 這個月花了多少 |
| `UsagePage_RangeFormat` | {0}–{1} · 約 {2} |
| `UsagePage_BudgetFormat` | 每月預算 {0} · 已用 {1}% |
| `UsagePage_EditBudget` | 編輯 |
| `UsagePage_SetBudget` | 設定每月預算 |
| `UsagePage_BudgetInputLabel` | 每月預算（新台幣） |
| `UsagePage_BudgetNote` | 用到 80% 和 100% 時各提醒一次。Lumi 只會提醒，不會停止回應。 |
| `UsagePage_BudgetInvalid` | 請輸入大於 0 的整數金額。 |
| `UsagePage_NoBudget` | 不設預算 |
| `UsagePage_Cancel` | 取消 |
| `UsagePage_Save` | 儲存 |
| `UsagePage_ProjectionOverFormat` | 照目前的速度，月底約 {0}，會超過預算。 |
| `UsagePage_ProjectionFormat` | 照目前的速度，月底約 {0}。 |
| `UsagePage_UnpricedFormat` | 有 {0} 次請求的模型未設定單價，沒有計入。 |
| `UsagePage_Empty` | 這個月還沒有用量。 |
| `UsagePage_DailyTitle` | 每天 |
| `UsagePage_DailyTooltipFormat` | {0} · {1} |
| `UsagePage_TopTitle` | 花最多的對話 |
| `UsagePage_TopMetaFormat` | {0} · {1} 回合 |
| `UsagePage_HintsTitle` | 這個月出現過的情況 |
| `UsagePage_HintCountFormat` | {0} 則對話 |
| `UsagePage_HintMuted` | 已關閉提醒 |
| `UsagePage_HintUnmute` | 重新開啟 |
| `UsagePage_Footnote` | 金額依各模型的單價估算，實際以供應商帳單為準。 |
| `UsagePage_PricingLink` | 單價設定 |
| `UsagePage_Export` | 匯出明細（CSV） |
| `UsagePage_ExportDoneFormat` | 已匯出到 {0} |
| `UsagePage_ExportFailed` | 匯出失敗，請換個位置再試一次。 |
| `UsagePage_DeletedConversation` | （已刪除的對話） |
| `Pricing_Title` | 單價 |
| `Pricing_SummaryFormat` | 輸入 {0} · 輸出 {1} · US$／每百萬 tokens |
| `Pricing_AutoFilled` | 已自動帶入 |
| `Pricing_Edit` | 編輯 |
| `Pricing_Input` | 輸入 |
| `Pricing_Output` | 輸出 |
| `Pricing_Advanced` | 進階：快取價格 |
| `Pricing_CacheRead` | 快取讀取 |
| `Pricing_CacheWrite` | 快取寫入 |
| `Pricing_CacheHint` | 留空時以輸入價計算。 |
| `Pricing_Restore` | 還原自動帶入的單價 |
| `Pricing_CandidatesFormat` | 價目表裡沒有「{0}」，這幾個最像。選一個，或自己填： |
| `Pricing_CandidatePriceFormat` | 輸入 {0} · 輸出 {1} |
| `Pricing_Apply` | 套用 |
| `Pricing_Manual` | 自己填 |
| `Pricing_NotFound` | 價目表裡找不到這個模型。先留空也可以，費用會顯示「未設定單價」。 |
| `Pricing_Local` | 本機模型，不計費 · 用量照常記錄，費用算 0 |
| `Pricing_Note` | 只用來估算費用。改了只影響之後的請求。 |
| `Pricing_Invalid` | 請輸入 0 以上的數字。 |
| `Pricing_TableDateFormat` | 價目表日期：{0} |
| `Pricing_UpdateTable` | 更新價目表 |
| `Pricing_Updating` | 更新中… |
| `Pricing_UpdatedFormat` | 價目表已更新（{0}）。 |
| `Pricing_UpdateFailed` | 更新失敗，繼續使用目前的價目表。 |
| `Usage_ConversationCost` | 本對話 |
| `Usage_MonthCost` | 本月 |
| `Usage_MonthOfBudgetFormat` | {0} ／ {1} |
| `Usage_OpenUsagePage` | 用量與費用 |
| `Usage_Unpriced` | 未設定單價 |
| `Usage_CostNote`（**改既有文字**） | 金額是估算；實際費用由供應商計算，從你的帳號扣款。 |
| `Budget_Alert80Format` | 這個月的 AI 費用已經用到預算的 80%（{0} ／ {1}）。 |
| `Budget_Alert100Format` | 這個月的 AI 費用已經超過預算（{0} ／ {1}）。Lumi 會繼續回應；要限制花費，請到供應商的後台設定上限。 |
| `Budget_ViewUsage` | 查看用量 |
| `Budget_Dismiss` | 知道了 |
| `Sprite_Budget80` | 這個月的 AI 費用用到 80% 囉，月底可能會超過預算。 |
| `Sprite_Budget100` | 這個月的 AI 費用已經超過預算了，要注意一下喔。 |
| `Hint_Dismiss` | 知道了 |
| `Hint_Mute` | 不再提醒這類 |
| `Hint_NewConversation` | 開新對話 |
| `Hint_Continue` | 繼續 |
| `Hint_ManageTools` | 管理工具 |
| `Hint_SwitchInNewConversation` | 開新對話再換 |
| `Hint_SwitchAnyway` | 照樣換 |
| `Hint_ModelSwitchFormat` | 這則對話已有 {0} 的前文。換模型後，這些前文要用全價重新送一次。 |
| `Hint_LongConversation` | 這則對話的前文已經很長，之後每問一句都會一起送出。換話題時，開新對話比較省。 |
| `Hint_StaleResumeFormat` | 上次是 {0}前。快取已經過期，下一句會把 {1} 的前文用全價重新送一次。 |
| `Hint_BigFileFormat` | 「{0}」（{1} tokens）在這則對話的前文裡，之後每一輪都會一起送出。用完了的話，可以整理對話或開新對話。 |
| `Hint_ImagesFormat` | 這則對話的前文裡有 {0} 張圖片，之後每一輪都會重新送出。看完了的話，可以整理對話或開新對話。 |
| `Hint_McpToolsFormat` | 目前啟用的工具說明約 {0} tokens，每次請求都會送出。用不到的 MCP 伺服器可以先關掉。 |
| `Hint_RegenerateFormat` | 已經重新產生 {0} 次了。補充你想要的方向（例如長度、格式、重點），通常比再試一次有效。 |
| `Hint_LongOutput` | 最近幾次回覆都很長，輸出是最貴的部分。改程式時可以請 Lumi 只給修改的地方。 |
| `Hint_RepeatPasteFormat` | 這段內容在第 {0} 回合已經貼過，前文裡就有，下次可以直接說「剛才那段」。 |
| `Hint_ElapsedHoursFormat` | {0} 小時 |
| `Hint_ElapsedDaysFormat` | {0} 天 |
| `Hint_ModelSwitchName` | 對話中途換模型 |
| `Hint_ModelSwitchSummary` | 換模型後，前文要用全價重新送一次。 |
| `Hint_LongConversationName` | 長對話一直聊下去 |
| `Hint_LongConversationSummary` | 前文很長時，每問一句都會一起送出。 |
| `Hint_StaleResumeName` | 隔很久回到長對話 |
| `Hint_StaleResumeSummary` | 快取過期後，前文要用全價重新送一次。 |
| `Hint_BigFileName` | 大檔案留在前文 |
| `Hint_BigFileSummary` | 大檔案每一輪都會一起送出。 |
| `Hint_ImagesName` | 圖片留在前文 |
| `Hint_ImagesSummary` | 圖片每一輪都會重新送出。 |
| `Hint_McpToolsName` | 啟用太多工具 |
| `Hint_McpToolsSummary` | 工具說明每次請求都會送出。 |
| `Hint_RegenerateName` | 一直重新產生 |
| `Hint_RegenerateSummary` | 補充方向通常比再試一次有效。 |
| `Hint_LongOutputName` | 要求整份重寫 |
| `Hint_LongOutputSummary` | 輸出是最貴的部分。 |
| `Hint_RepeatPasteName` | 同一份內容反覆貼上 |
| `Hint_RepeatPasteSummary` | 前文裡已經有的內容，不用再貼一次。 |
| `Mcp_ToolDefinitionSizeFormat` | 工具說明約 {0} tokens |

   「整理並繼續」沿用既有的 `Usage_CompactAndContinue`；首頁卡片的「管理」沿用 `Home_Manage`、「今天 …」沿用 `Home_UsageTodayFormat`、預算列沿用 `UsagePage_BudgetFormat`。預算提醒文字照表（決定 20）。

9. `dotnet build` 零警告、`dotnet test` 全過後 commit，再開 Wave 1。

---

### 包 23A：價目表與比對（Wave 1）

- **目標**：給一個 model ID 與連線，回傳「自動帶入」或「候選清單」或「找不到」；可以從 GitHub 更新價目表。
- **先讀這些檔案**：`src/Lumi.Core/Usage/ModelPricing.cs`（Phase 0）、`src/Lumi.Web/Updates/GitHubReleaseUpdateChecker.cs`（仿照它的 `IWebClient` 用法與錯誤處理）、`src/Lumi.Web/WebServiceCollectionExtensions.cs`（只讀，不改）、`src/Lumi.Core/Chat/ChatFailureText.cs` 的 `IsLocalTarget`、`src/Lumi.Infrastructure/LumiPaths.cs`、本計畫第 4 節。
- **可以新增／修改的檔案**：
  - 新增 `src/Lumi.Core/Usage/PriceTable.cs`、`PriceMatcher.cs`、`JaroWinkler.cs`、`IPriceTableSource.cs`
  - 新增 `src/Lumi.Web/Pricing/LiteLlmPriceParser.cs`、`PriceTableSource.cs`
  - 新增 `tests/Lumi.Core.Tests/Usage/PriceMatcherTests.cs`、`JaroWinklerTests.cs`、`tests/Lumi.Web.Tests/Pricing/LiteLlmPriceParserTests.cs`、`PriceTableSourceTests.cs`（`Lumi.Web.Tests` 不存在就停下來回報）
- **步驟**：
  1. `PriceTable`：`public sealed record PriceEntry(string Key, string? Provider, decimal InputPerMTok, decimal OutputPerMTok, decimal? CacheReadPerMTok, decimal? CacheWritePerMTok, int? MaxInputTokens, int? MaxOutputTokens)`；`public sealed class PriceTable(IReadOnlyList<PriceEntry> entries, DateOnly snapshotDate)`，提供 `Entries`、`SnapshotDate`、`static PriceTable Empty`。
  2. `PriceMatcher.Match(PriceTable table, string modelId, ConnectionSettings connection) → PriceMatch`：
     - `public enum PriceMatchKind { Exact, Normalized, Candidates, NotFound, Local }`
     - `public sealed record PriceMatch(PriceMatchKind Kind, PriceEntry? Entry, IReadOnlyList<PriceEntry> Candidates)`；`Exact`／`Normalized` 時 `Entry` 有值；`Candidates` 時最多 3 筆；`Local` 時兩者都空。
     - 規則照第 4 節逐條實作；正規化抽成 `public static string Normalize(string key)`（公開以便測試）。
     - 供應商偏好：`PreferredProvider(ConnectionSettings)` 回 `"azure"`／`"anthropic"`／`"openai"`，主機判斷照第 4 節。
     - `ToPricing(PriceEntry e) → ModelPricing`（`Source = Suggested`）也放這裡。
  3. `JaroWinkler.Similarity(string a, string b) → double`（標準演算法，前綴加權 0.1、最多 4 字元；比較前都轉小寫）。
  4. `IPriceTableSource`：`PriceTable Current { get; }`、`Task<PriceTableUpdateResult> UpdateAsync(CancellationToken ct)`；`public sealed record PriceTableUpdateResult(bool Success, DateOnly SnapshotDate)`。
  5. `LiteLlmPriceParser.Parse(Stream json, DateOnly snapshotDate) → PriceTable`：用 `JsonDocument`，規則照第 4 節「解析」；讀不懂的條目略過。
  6. `PriceTableSource`（`internal sealed`，建構式 `(IWebClient web, ILogger<PriceTableSource> logger, string? dataDirectory = null)`，null 時用 `LumiPaths.DataDirectory`）：第一次讀 `Current` 時載入（使用者資料夾的 `model-prices.json`＋`model-prices.date.txt` 優先，解析失敗或少於 100 筆就用內建資源）；`UpdateAsync` 照第 4 節，寫檔先寫 `.tmp` 再 `File.Move(..., overwrite: true)`。
- **測試**：
  - `PriceMatcherTests`：`Exact_IgnoresCase`；`Normalized_StripsProviderPrefix`（`anthropic/claude-x` ↔ `claude-x`）；`Normalized_StripsDateSuffix`（`claude-sonnet-4-20250514` ↔ `claude-sonnet-4`、`gpt-4o-2024-08-06` ↔ `gpt-4o`）；`Normalized_StripsLatestAndBedrock`（`llama3:latest`、`us.anthropic.claude-x-v1:0`）；`Prefix_IsCandidateNotAutoApplied`（`gpt-4o-prod` → `Candidates`，第一名 `gpt-4o`）；`Fuzzy_ReturnsTopThree`；`Fuzzy_DoesNotAutoApply_MiniVsFull`（`gpt-4.1-mimi` 只得到候選）；`Azure_PrefersAzureEntries`；`Local_ReturnsLocal`（`http://localhost:11434`）；`NotFound_WhenNothingClose`。表格用測試內手寫的小表。
  - `LiteLlmPriceParserTests`：單位換算（`0.000003` → `3.0000`）；略過 `sample_spec`、非 chat、缺價錢的條目；壞掉的單筆不影響其他；**內建資源檔能解析且 ≥ 500 筆**。
  - `PriceTableSourceTests`：使用者資料夾有有效檔就用它；壞檔就退回內建；`UpdateAsync` 成功寫檔、下載失敗或筆數不足不覆蓋（假的 `IWebClient`）。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Pricing|FullyQualifiedName~PriceMatcher|FullyQualifiedName~JaroWinkler"` 全過。
- **不要做**：不要在 DI 註冊（整合包做）；不要改 `ModelSettings` 或任何 UI；不要把價目表裁切後另存一份進 repo。

### 包 23B：費用紀錄（Wave 1）

- **目標**：每次呼叫供應商結束後，`request_usage` 多一列，費用照當時的單價算好；工具連續失敗 3 次提早停下（L2）。
- **先讀這些檔案**：`src/Lumi.Core/Chat/ConversationService.cs`（`RunTurnAsync`、`StreamOnceAsync`、`CompactSummaryAsync`）、`src/Lumi.Core/Chat/ContextEstimator.cs`、`src/Lumi.Core/Usage/*`（Phase 0）、`src/Lumi.Infrastructure/Persistence/SqliteConversationStore.cs`（仿照它的連線、時間格式與參數寫法）、`SqliteMemoryStore.cs`、`tests/Lumi.Core.Tests/Chat/ConversationToolLoopTests.cs`（仿照假 provider 的寫法）、`tests/Lumi.Infrastructure.Tests/SqliteUsageQueriesTests.cs`、本計畫第 2、3 節與 L2。
- **可以新增／修改的檔案**：
  - 新增 `src/Lumi.Core/Usage/CostCalculator.cs`、`UsageRecorder.cs`
  - 修改 `src/Lumi.Core/Chat/ConversationService.cs`
  - 新增 `src/Lumi.Infrastructure/Persistence/SqliteUsageStore.cs`
  - 新增 `tests/Lumi.Core.Tests/Usage/CostCalculatorTests.cs`、`UsageRecorderTests.cs`、`tests/Lumi.Core.Tests/Chat/ConversationUsageRecordingTests.cs`、`ConversationToolFailureLoopTests.cs`、`tests/Lumi.Infrastructure.Tests/SqliteUsageStoreTests.cs`
- **步驟**：
  1. `CostCalculator.Cost(TokenUsage tokens, PriceSnapshot? price) → decimal?`：null 價格回 null；免費回 0；其餘照第 2 節公式（`decimal`）。
  2. `UsageRecorder : IUsageRecorder`（建構式 `(IUsageStore store, TimeProvider time, ILogger<UsageRecorder> logger)`）：`PriceSnapshot.From(observation.Model.Model.Pricing)`；本機連線（`ChatFailureText.IsLocalTarget(observation.Model.Connection.BaseUrl)`）而且 `Pricing.Source == None` 時視為免費；組 `UsageRecord`（`ConnectionName` = 連線名稱）寫入。任何例外 `LogWarning` 後吞掉。
  3. `ConversationService`：建構式最後加選用參數 `IUsageRecorder? usageRecorder = null`（null 用 `NullUsageRecorder`）。
     - `StreamOnceAsync` 多兩個參數 `ChatMode mode`、`int toolDefinitionTokens`（由 `RunTurnAsync` 傳入；`toolDefinitionTokens` = `offered.Count > 0 ? ContextEstimator.EstimateText(ToolSnapshotJson.Serialize(offered)) : 0`，在迴圈外算一次）。追蹤 `lastUsage`、`receivedText`（含 `ThinkingDelta`）、`anyEvent`，結束時照第 3 節的表呼叫 `RecordAsync`（`CancellationToken.None`，在 `throw` 或 `yield break` 之前）。估算輸入抽成 `internal static int EstimateRequestInput(ChatRequest request)`。
     - `CompactSummaryAsync` 加 `ChatMode mode` 參數（手動整理 `CompactAsync` 傳 `ChatMode.Workbench`），用同一套規則記 `UsagePurpose.Compaction`、`toolDefinitionTokens = 0`。
     - **L2**：`RunTurnAsync` 記住上一輪最後一個工具結果的 `(名稱, 正規化後的 InputJson, IsError)`；每輪結束時，若這一輪只有一個工具呼叫且和前兩輪相同、都是 `IsError`，存好工具結果之後回 `Failed(ToolLoopLimit, 訊息見 L2, Retryable: true)` 並結束。正規化 = 用 `JsonDocument` 讀再 `JsonSerializer.Serialize` 回去（讀不了就用原字串去空白）。觸發時 `LogInformation("工具 {Tool} 連續失敗 {Count} 次，提早停止", ...)`。
  4. `SqliteUsageStore : IUsageStore`（`internal sealed`，建構式照 `SqliteMemoryStore`）：實作 Phase 0 介面；`ListTopConversationsAsync` 以 `INNER JOIN conversations` 排除已刪除，`TurnCount` 用子查詢數 `messages.role = 'user'`；`price_snapshot_json` 用 `System.Text.Json` 存讀。
- **測試**：
  - `CostCalculatorTests`：四種 token 各自乘對的價；快取價 null 用輸入價（由 `PriceSnapshot.From` 處理，這裡測組合）；免費 0；未設定 null。
  - `UsageRecorderTests`：寫入的快照與費用正確；本機連線＋未設定 → 免費；store 丟例外時不往外丟。
  - `ConversationUsageRecordingTests`（假 provider＋`InMemoryUsageStore`）：`Completed_WithUsage_RecordsActual`；`Completed_WithoutUsage_UsesLastUsageEvent`；`Completed_NoUsageAtAll_RecordsEstimated`；`ToolLoop_RecordsEachRequest`（兩次呼叫兩列）；`Cancelled_AfterText_RecordsEstimated`；`Failed_BeforeAnyEvent_RecordsNothing`；`Regenerate_KeepsEarlierRecords`；`Compaction_RecordsCompactionPurpose`；`PriceChange_DoesNotAffectEarlierRecords`；`ToolDefinitionTokens_Recorded`；`SpriteMode_RecordsSpriteMode`。
  - `ConversationToolFailureLoopTests`：同工具同參數連續 3 次錯誤 → `ToolLoopLimit` 且只呼叫 provider 3 次；參數不同不觸發；中間有一次成功不觸發。
  - `SqliteUsageStoreTests`：存讀來回（含 null 快照）；`SumCostAsync` 略過 null；區間邊界 [from, to)；刪除對話後紀錄還在、Top 清單不列它；`GetConversationCostAsync` 的 `HasEstimated`／`UnpricedCount`。
- **完成檢查**：`dotnet build` 零警告；`dotnet test` 全部通過（`ConversationService` 改了簽章，既有測試都要過）。
- **不要做**：不要改 `messages` 或 `conversation_compactions` 的寫法；不要動 `ConversationUsageTally`；記錄失敗不能讓對話失敗；不要在 DI 註冊。

### 包 23C：`stream_options` 退回與工具結果頭尾截斷（Wave 1）

- **目標**：舊版 Azure 等不認得 `stream_options` 的服務也能對話（用量改估算）；工具結果太長時保留開頭與結尾（L1）。
- **先讀這些檔案**：`src/Lumi.Providers/OpenAICompatible/OpenAICompatibleChatProvider.cs`、`OpenAIHttpErrors.cs`、`tests/Lumi.Providers.Tests/`（找現有 OpenAI provider 測試的假 `HttpMessageHandler` 寫法仿照）、`src/Lumi.Core/Tools/ToolExecutor.cs`、它的既有測試、本計畫第 3 節與 L1。
- **可以新增／修改的檔案**：`OpenAICompatibleChatProvider.cs`、`ToolExecutor.cs`、新增 `tests/Lumi.Providers.Tests/OpenAIStreamOptionsFallbackTests.cs`、修改或新增 `ToolExecutor` 的測試檔。
- **步驟**：
  1. Provider 加欄位 `ConcurrentDictionary<string, byte> _noStreamOptions`（連線 id）。`BuildRequest` 多一個 `bool includeStreamOptions` 參數。收到 400 且 `errorBody` 含 `stream_options`（`OrdinalIgnoreCase`）而且這次有帶時：記下連線 id、`LogInformation("連線 {ConnectionId} 不支援 stream_options，改為不帶並估算用量", ...)`、丟掉這次回應、不帶 `stream_options` 重送**一次**；重送仍失敗照一般錯誤處理。
  2. `ToolExecutor.Truncate`：超過上限時取開頭 70% 與結尾 30% 的字元（兩端都不切斷代理對、盡量對齊到換行：開頭往前找最近的 `\n`、結尾往後找最近的 `\n`，找不到就照字元切），中間換成 `\n[中間省略了 {N} 行；需要的話可以用讀檔工具指定行範圍再讀]\n`（N = 被省略部分的換行數＋1）。`TruncatedNotice` 常數保留但改成這段格式字串的前綴，避免其他地方編譯失敗（用到的地方先 grep 確認，有依賴舊文字的測試就照新格式更新）。
- **測試**：`FallbackWithoutStreamOptions_On400MentioningIt`（第一個請求帶、第二個不帶、最後成功）；`RemembersConnection`（同一個 provider 第二次對話直接不帶）；`Other400_NoRetry`；`Truncate_KeepsHeadAndTail`；`Truncate_ReportsOmittedLines`；`Truncate_DoesNotSplitSurrogates`；`Short_Unchanged`。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Providers|FullyQualifiedName~ToolExecutor"` 全過，再跑一次 `dotnet test` 全部。
- **不要做**：不要把「不支援」寫進設定檔；不要改 Anthropic provider；不要改 `MaxResultChars` 的數值。

### 包 23D：月報計算與預算判斷（Wave 1）

- **目標**：用量頁與用量面板要的所有數字都由純邏輯算好；預算提醒的判斷與狀態保存。
- **先讀這些檔案**：`src/Lumi.Core/Usage/*`（Phase 0）、`src/Lumi.Core/Chat/IConversationStore.cs`（`GetMetaAsync`／`SetMetaAsync`）、本計畫第 5、6 節。
- **可以新增／修改的檔案**：新增 `src/Lumi.Core/Usage/MonthlyUsageSummary.cs`、`BudgetMonitor.cs`、`MoneyFormat.cs`；測試 `tests/Lumi.Core.Tests/Usage/MonthlyUsageSummaryTests.cs`、`BudgetMonitorTests.cs`、`MoneyFormatTests.cs`。
- **步驟**：
  1. `MoneyFormat`：`Twd(decimal usd, bool estimated = false) → string`（`NT$1,234`；< 10 一位小數 `NT$3.2`；0 → `NT$0`；estimated 加前綴「≈」）；`Usd(decimal usd) → string`（`US$15.25`）；`TwdAmount(decimal usd) → decimal`（四捨五入到整數，給預算輸入框顯示）；`UsdFromTwd(decimal twd) → decimal`（除以 `Money.TwdPerUsd`）；`Tokens(int tokens) → string`（輸出必須和 App 的 `WorkbenchViewModel.FormatTokens` 完全相同——Core 不能參考 App，所以照它的規則另寫一份，測試用同一組數字對照）。全部用 `CultureInfo.InvariantCulture` 的數字格式。
  2. `MonthlyUsageSummary.Build(IReadOnlyList<UsageRecord> records, decimal? budgetUsd, DateTimeOffset now, TimeZoneInfo zone) → MonthlyUsageSummary`，屬性：`TotalUsd`、`HasEstimated`、`UnpricedCount`、`RangeStart`（`DateOnly`，本月 1 日與第一筆紀錄的本地日期取較晚者）、`RangeEnd`（今天）、`Daily`（`IReadOnlyList<DailyCost>`，`DailyCost(DateOnly Date, decimal Usd, bool IsToday, bool IsFuture)`，從 1 日到月底）、`BudgetRatio`（`double?`，沒預算 null）、`ProjectedUsd`（`decimal?`；有紀錄的天數 < 7 時 null；= 總額 ÷ 已過天數 × 當月天數，已過天數 = `RangeStart` 到今天的天數）、`WillExceed`（有預算且預測 > 預算）、`IsEmpty`。另提供 `static (DateTimeOffset From, DateTimeOffset To) MonthRange(DateTimeOffset now, TimeZoneInfo zone)`（本地月初到下月初，轉成 UTC）。
  3. `BudgetMonitor`（建構式 `(IUsageStore usage, IConversationStore meta, IModelCatalog catalog, TimeProvider time)`，時區用 `time.LocalTimeZone`）：`Task<BudgetAlertLevel?> CheckAsync(CancellationToken ct)`；`public enum BudgetAlertLevel { Eighty = 80, Hundred = 100 }`。規則照第 5 節；`app_meta` key 常數 `BudgetAlertedKey = "budget_alerted"`，值 `yyyy-MM:80`／`yyyy-MM:100`。另提供 `Task<(decimal SpentUsd, decimal? BudgetUsd)> GetMonthAsync(CancellationToken ct)` 給用量面板與提醒文字用。
- **測試**：`MoneyFormatTests`（各邊界：0、9.99、10、1234.5、估算前綴、Tokens 999／1000／92_400）；`MonthlyUsageSummaryTests`（跨月邊界用 Asia/Taipei 時區；`RangeStart` 取第一筆；第 6 天沒有預測、第 7 天有；今天標記；未來日；沒預算時 `BudgetRatio` null；未設定單價的計數）；`BudgetMonitorTests`（未達 80% 無提醒；跨 80% 提醒一次、第二次不提醒；直接跨 100% 只提醒 100%；100% 後不再 80%；換月重置；沒預算不提醒）。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Usage"` 全過。
- **不要做**：不要寫任何 UI；不要讀 `request_usage` 以外的表算費用。

### 包 23E：圖片縮小（L5，Wave 1）

- **目標**：加入上下文的大圖在存成附件前縮到長邊 1568px。
- **先讀這些檔案**：`src/Lumi.Tools/Files/ContextAttachmentBuilder.cs`（`BuildImageAsync`）、`src/Lumi.Core/Context/Attachments.cs`、`src/Lumi.Core/Usage/IImageDownscaler.cs`（Phase 0）、`tools/Lumi.IconGen/`（看 SkiaSharp 的用法）、`tests/Lumi.Architecture.Tests/DependencyRuleTests.cs`（確認 App 能用 SkiaSharp、Tools 不能）。
- **可以新增／修改的檔案**：修改 `ContextAttachmentBuilder.cs`（建構式加選用 `IImageDownscaler? downscaler = null`）；新增 `src/Lumi.App/Services/SkiaImageDownscaler.cs`；新增 `tests/Lumi.Tools.Tests/Files/ContextAttachmentDownscaleTests.cs`、`tests/Lumi.App.Tests/SkiaImageDownscalerTests.cs`。
- **步驟**：
  1. `ContextAttachmentBuilder`：常數 `public const int MaxImageEdge = 1568;`。`BuildImageAsync` 讀到 bytes、判斷 media type 後，`save` 為 true 且有 downscaler 時呼叫 `Downscale`；回非 null 就用新的 bytes 與 media type 存檔。預估 token 的路徑（`save: false`）不縮。
  2. `SkiaImageDownscaler`：`image/gif` 回 null；`SKBitmap.Decode` 失敗回 null；長邊 ≤ maxEdge 回 null；否則等比縮放（`SKSamplingOptions` 高品質），PNG 輸入輸出 PNG，其他輸出 JPEG 品質 85（`image/jpeg`）。
- **測試**：Tools：有 downscaler 時存的是縮過的 bytes、沒有時原樣、`save: false` 不呼叫。App：3000×2000 PNG → 1568×1045 PNG；1000×800 回 null；JPEG 輸出是 JPEG；GIF 回 null；壞資料回 null。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Downscale"` 全過。
- **不要做**：不要在 DI 註冊；不要處理工具產生的截圖；不要讓 `Lumi.Tools` 參考 SkiaSharp。

---

### 包 23F：模型設定的單價段落（Wave 2）

- **目標**：新增或編輯模型時看得到、改得了單價；比對結果照第 6 節「AI 連線 › 模型設定」顯示；舊模型升級後自動帶入。
- **先讀這些檔案**：`src/Lumi.App/ViewModels/ModelsViewModel.cs`（`ModelSnapshot`、`CurrentSnapshot`、`DefaultsFor`、`SaveAsync`、連線改變時的處理）、`src/Lumi.App/Views/ModelsPanel.axaml`、`src/Lumi.Core/Usage/PriceMatcher.cs`、`IPriceTableSource.cs`（23A）、`tests/Lumi.App.Tests/AiSettingsViewModelTests.cs`（仿照測試組法）、`plan/ui-guide/ui-guide.html`（表單與警示框）、本計畫第 4、6 節。
- **可以新增／修改的檔案**：修改 `ModelsViewModel.cs`、`ModelsPanel.axaml`（`.axaml.cs` 有需要才改）；新增 `src/Lumi.App/ViewModels/ModelPricingViewModel.cs`、`src/Lumi.App/Services/PricingAutoFill.cs`；新增 `tests/Lumi.App.Tests/ModelPricingViewModelTests.cs`、`PricingAutoFillTests.cs`、`tests/Lumi.App.Tests/Headless/ModelPricingSectionTests.cs`。
- **步驟**：
  1. `ModelPricingViewModel`（`ModelsViewModel` 擁有一個，屬性名 `Pricing`）：輸入 `(PriceTable, modelId, ConnectionSettings, ModelPricing current)` 時重算 `Match`。狀態屬性：`IsLocal`、`IsAutoFilled`（`Source == Suggested`）、`ShowSummary`、`IsEditing`、`ShowCandidates`、`ShowNotFound`、`Candidates`（`ObservableCollection<PriceCandidateItem>`，含 `ModelId`、`PriceText`、`IsSelected`）、四個文字欄位（`string`，空白 = null）、`ShowAdvanced`、`CanRestore`、`InputError`／`OutputError`／…（不是 ≥ 0 的數字時 `Pricing_Invalid`）。命令：`EditCommand`、`ApplyCandidateCommand`、`ManualCommand`、`RestoreCommand`、`UpdateTableCommand`（呼叫 `IPriceTableSource.UpdateAsync`，成功後重算並顯示 `Pricing_UpdatedFormat`，失敗顯示 `Pricing_UpdateFailed`）。`ToPricing() → ModelPricing`：使用者改過任何欄位或套用候選 → `Confirmed`；未改且是自動帶入 → `Suggested`；本機 → `IsFree = true`。數字顯示用 `0.00`（小數超過兩位時照實際位數，最多 4 位）。
  2. `ModelsViewModel`：`ModelSnapshot` 加 `ModelPricing Pricing`（`IsDirty` 會跟著比較）；`ModelId` 或連線改變時呼叫 `Pricing.Rematch(...)`（`Confirmed` 的單價不覆蓋，除非 model ID 改了）；**新增模型**比對到 `Exact`／`Normalized` 時，用 `MaxInputTokens`／`MaxOutputTokens` 覆蓋 `ContextWindow`／`MaxOutputTokens` 欄位（有值才覆蓋；`DefaultsFor` 仍是比對不到時的預設）；`SaveAsync` 存 `model with { Pricing = Pricing.ToPricing() }`。建構式加 `IPriceTableSource` 參數。
  3. `ModelsPanel.axaml`：在上下文上限那一組之後加「單價」段落，版面照第 6 節。摘要行用 `TextBlock` 的 `Run`；「✓ 已自動帶入」用 `LumiIconCheck` 14px＋`LumiSuccessBrush`；「編輯」「還原…」「更新價目表」用 `Button Classes="tertiary small"`；近似候選用 `RadioButton` 清單放在 Warning 樣式的 `Border`（同 `UsageFlyout` 的快滿提醒：`LumiWarningBackgroundBrush`、`LumiWarningBorderBrush`、圓角 10、內距 12）；欄位間距 12、段落間距 24。
  4. `PricingAutoFill.RunAsync(IModelCatalog catalog, IPriceTableSource prices, IConversationStore meta, CancellationToken ct)`：`app_meta` 的 `pricing_autofill_done` 不是 `1` 時，對每個 `Pricing == ModelPricing.None` 的模型比對；`Exact`／`Normalized` → `SaveModelAsync(model with { Pricing = ToPricing(entry) })`；`Local` → `IsFree = true, Source = Suggested`；其餘不動。最後寫 `pricing_autofill_done = 1`。不改上下文上限與最大輸出。
- **測試**：`ModelPricingViewModelTests`（Exact → 摘要＋已自動帶入；近似 → 候選、套用後 `Confirmed`；找不到 → 空欄位；本機 → `IsFree`；改欄位 → `Confirmed`；還原；負數錯誤；`UpdateTable` 成功與失敗訊息）；`ModelsViewModel` 新增模型時上下文上限被帶入、編輯既有模型不被改；`PricingAutoFillTests`（只做一次、只動 `None` 的、本機變免費、近似不動）；Headless：段落在五種狀態下的可見元素。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Pricing|FullyQualifiedName~ModelsViewModel|FullyQualifiedName~AiSettings"` 全過，再跑全部。
- **不要做**：不要改 `SettingsViewModel`、`SettingsWindow.axaml`；不要在啟動流程呼叫 `PricingAutoFill`（整合包接）；不要新增字串或色票。

### 包 23G：設定 › 用量與費用頁、CSV（Wave 2）

- **目標**：設定視窗多一頁「用量與費用」，照第 6 節顯示本月卡片、預算編輯、每日長條、花最多的對話、這個月出現過的情況（Wave 3 之前資料是空的、整段不顯示）、頁尾；可以匯出 CSV。
- **先讀這些檔案**：`src/Lumi.App/Views/HomePanel.axaml`、`HomeViewModel.cs`（仿照卡片、非同步載入與 `SettingsPageHeader` 的用法）、`src/Lumi.App/Views/MemoryPanel.axaml`（原地展開編輯的寫法）、`src/Lumi.App/Views/FilePickerService.cs`（`SaveFileAsync`）、`src/Lumi.App/ViewModels/ISettingsWindowService.cs`（只讀：開啟對話、跳頁的既有方法）、`src/Lumi.Core/Usage/MonthlyUsageSummary.cs`、`BudgetMonitor.cs`、`MoneyFormat.cs`（23D）、`IUsageHintLog`（Phase 0）、`plan/ui-guide/ui-guide.html`、本計畫第 6、7.3、8 節。
- **可以新增／修改的檔案**：新增 `src/Lumi.App/ViewModels/UsageSettingsViewModel.cs`、`UsageHintSummaryItem.cs`、`src/Lumi.App/Views/UsagePanel.axaml`、`UsagePanel.axaml.cs`、`src/Lumi.App/Controls/DailyCostBars.cs`、`src/Lumi.App/Services/UsageCsvExporter.cs`；測試 `tests/Lumi.App.Tests/UsageSettingsViewModelTests.cs`、`UsageCsvExporterTests.cs`、`tests/Lumi.App.Tests/Headless/UsagePanelTests.cs`。
- **步驟**：
  1. `UsageSettingsViewModel`（建構式 `(IUsageStore usage, IUsageHintLog hints, IModelCatalog catalog, IConversationStore conversations, IFilePickerService files, ISettingsWindowService? windows, TimeProvider time)`）：`LoadAsync()`（頁面顯示時呼叫）讀本月紀錄 → `MonthlyUsageSummary`、Top 5、提示紀錄。顯示屬性全部是字串或布林（`TotalText`、`RangeText`、`BudgetText`、`BudgetRatio`、`BudgetLevel`（Normal／Warning／Danger）、`ProjectionText`、`ShowProjection`、`UnpricedText`、`IsEmpty`、`HasBudget`、`IsEditingBudget`、`BudgetInput`、`BudgetError`、`Daily`、`TopConversations`、`Hints`、`ShowHints`）。命令：`EditBudgetCommand`、`SaveBudgetCommand`（整數 > 0，否則 `UsagePage_BudgetInvalid`；`SetUsageAsync(Usage with { MonthlyBudgetUsd = MoneyFormat.UsdFromTwd(x) })`）、`CancelBudgetCommand`、`ClearBudgetCommand`、`OpenConversationCommand(string id)`、`OpenPricingCommand`（跳 AI 連線頁）、`ExportCommand`、`UnmuteHintCommand(string rule)`、`ToggleHintCommand(string rule)`（展開對話清單）。
  2. 開啟對話與跳頁：用 `ISettingsWindowService` 既有的方法；**沒有「開啟某則對話並回到工作台」的方法就停下來回報**（整合包加）。
  3. `UsageHintSummaryItem`：`Rule`、`Name`（`Hint_*Name`）、`Summary`（`Hint_*Summary`）、`CountText`（不同 `conversation_id` 的數量）、`IsMuted`、`IsExpanded`、`Conversations`（標題與日期；查不到標題的不列）。
  4. `DailyCostBars`（`Control`，自繪）：屬性 `Items`（`IReadOnlyList<DailyCost>`）；高度 96；每根寬度平均分配、間距 2；最高那天滿高；0 的畫 2px 高 `LumiBorderBrush`；今天 `LumiAccentHoverBrush`、其他 `LumiAccentBrush`；未來不畫；`ToolTip` 依滑鼠位置顯示 `UsagePage_DailyTooltipFormat`（日期 `M/d`、金額 `MoneyFormat.Twd`）。照 `UsageRing` 的自繪寫法與 DynamicResource 取色方式。
  5. `UsagePanel.axaml`：外框照 `HomePanel`；卡片 `Border` 背景 `LumiSurfaceBrush`、邊框 `LumiBorderBrush` 1px、圓角 14、內距 24；大數字 32px SemiBold；進度條高 6、圓角 3，軌道 `LumiSurfaceAltBrush`，填色依 `BudgetLevel` 用 `LumiAccentBrush`／`LumiWarningIconBrush`／`LumiDangerBrush`（和上下文圓環 `UsageRing.LevelBrushKey` 相同）；段落標題 14px SemiBold；段落間距 24；清單列高 44、列間 1px `LumiBorderBrush` 分隔；「›」用 `LumiIconChevronRight` 16px。
  6. `UsageCsvExporter.WriteAsync(Stream output, IReadOnlyList<UsageRecord> records, IReadOnlyDictionary<string, string> titles, TimeZoneInfo zone)`：欄位與格式照第 8 節；標題列文字寫在這個類別的常數（不放 resx，同 M22 工具顯示名稱的慣例）。
- **測試**：ViewModel（空月份、有預算 61%、預算輸入驗證與換算 800 → 25 美元、清除預算、預測句顯示條件、未設定單價計數、提示列與已靜音、匯出呼叫 exporter）；CSV（BOM、引號跳脫、已刪除對話、免費／未設定單價的快照欄位、本地時間）；Headless（卡片三種狀態、編輯展開、提示段落空時不顯示、五套主題淺深色下文字與背景對比 ≥ 4.5，仿照 `SettingsContrastTests`）。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Usage"` 全過，再跑全部。
- **不要做**：不要改 `SettingsViewModel`、`SettingsWindow.axaml`、`ISettingsWindowService`（整合包接分類與頁面）；不要新增字串或色票；不要畫第一版畫板的其他區塊。

### 包 23H：工作台用量面板與預算提醒（Wave 2）

- **目標**：用量面板多「本對話」「本月 ／ 預算」兩行；回應結束後跨過預算門檻時，工作台出現提醒列、小精靈說一句。
- **先讀這些檔案**：`src/Lumi.App/ViewModels/WorkbenchViewModel.Usage.cs`、`ConversationUsageViewModel.cs`、`src/Lumi.App/Views/UsageFlyout.axaml`、`src/Lumi.App/Views/UpdateBanner.axaml`＋`UpdateNoticeViewModel.cs`（仿照提醒列）、`src/Lumi.App/Views/WorkbenchWindow.axaml`（提醒列放在 `UpdateBannerView` 與 `NotificationStripView` 的 `StackPanel` 裡）、`src/Lumi.App/Services/NotificationCenter.cs`（看提醒怎麼送到小精靈）、`src/Lumi.App/Sprites/SpriteBubbleViewModel.cs`（`ShowNotice`）、`src/Lumi.Core/Chat/IChatSession.cs`（`RunFinished`、`ActiveMode`）、`tests/Lumi.App.Tests/WorkbenchUsageTests.cs`、`Headless/UsageFlyoutTests.cs`、本計畫第 5、6 節。
- **可以新增／修改的檔案**：修改 `WorkbenchViewModel.Usage.cs`、`ConversationUsageViewModel.cs`、`UsageFlyout.axaml`、`WorkbenchWindow.axaml`；新增 `src/Lumi.App/ViewModels/BudgetAlertViewModel.cs`、`src/Lumi.App/Views/BudgetBanner.axaml`＋`.axaml.cs`、`src/Lumi.App/Services/BudgetAlertCoordinator.cs`；測試 `tests/Lumi.App.Tests/BudgetAlertCoordinatorTests.cs`、修改 `WorkbenchUsageTests.cs`、`Headless/UsageFlyoutTests.cs`、新增 `Headless/BudgetBannerTests.cs`。
- **步驟**：
  1. `ConversationUsageViewModel`：`ConversationUsageSnapshot` **不改**；另加 `UpdateCost(ConversationCost conversation, decimal monthUsd, decimal? budgetUsd)`，屬性 `ConversationCostText`（`MoneyFormat.Twd(…, HasEstimated)`；`CostUsd` null 且 `UnpricedCount > 0` → `Usage_Unpriced`；沒有紀錄 → `NT$0`）、`MonthCostText`（有預算 `Usage_MonthOfBudgetFormat`，預算顯示為不帶「NT$」的整數，例如 `NT$488 ／ 800`）、`HasBudget`、`MonthRatio`、`MonthLevel`、`OpenUsagePageCommand`（建構式多一個選用 `Action? openUsagePage`）。
  2. `WorkbenchViewModel.Usage.cs`：加 `RefreshCostAsync()`：查 `IUsageStore.GetConversationCostAsync(CurrentConversationId)` 與 `BudgetMonitor.GetMonthAsync`，推給 `Usage.UpdateCost`。在載入／切換對話、`RunFinished`、`CompactionCompleted` 之後呼叫；建構式需要的 `IUsageStore`、`BudgetMonitor` 用選用參數加在最後（null 時不顯示費用段落：`Usage.ShowCost = false`）。**如果 `WorkbenchViewModel` 的建構式改動需要動到 `WorkbenchViewModel.cs` 以外的檔案，停下來回報。**
  3. `UsageFlyout.axaml`：在三個小方塊之後、連線列之前加費用段落（`IsVisible="{Binding Usage.ShowCost}"`）：上方 1px `LumiBorderBrush`；兩列 `Grid ColumnDefinitions="*,Auto"`，左 13px muted、右 13px Medium；「本月」下方 4px 進度條（顏色規則同 23G）；右下 `Button Classes="tertiary small"`「用量與費用 ›」。段落內間距 8。
  4. `BudgetAlertCoordinator`（App 服務，建構式 `(IChatSession session, BudgetMonitor monitor, BudgetAlertViewModel workbenchBanner, ISpriteNotice? sprite, IUiDispatcher dispatcher)`）：訂閱 `RunFinished` → `CheckAsync` → 有結果時依 `session.ActiveMode` 顯示在工作台提醒列或小精靈。小精靈的送法照 `NotificationCenter` 的現有做法；**如果沒有現成的「讓小精靈說一句」介面，停下來回報**（不要自己在 Sprites 資料夾加介面）。
  5. `BudgetAlertViewModel`：`IsVisible`、`IsDanger`、`Text`（`Budget_Alert80Format`／`Budget_Alert100Format`，參數是本月與預算的 `MoneyFormat.Twd`）、`ViewUsageCommand`（開設定到用量頁並關閉提醒列）、`DismissCommand`。
  6. `BudgetBanner.axaml`：照 `UpdateBanner` 的版面；Warning：`LumiWarningBackgroundBrush`／`LumiWarningBorderBrush`／`LumiWarningTextBrush`；Danger：`LumiDangerBackgroundBrush`／`LumiDangerBorderBrush`／`LumiDangerBrush`。放進 `WorkbenchWindow.axaml` 的那個 `StackPanel`，在 `UpdateBannerView` 之後。
- **測試**：面板文字（估算前綴、未設定單價、有無預算）；切換對話後本對話費用更新；`BudgetAlertCoordinatorTests`（工作台模式顯示提醒列、小精靈模式送小精靈、沒跨門檻不顯示）；Headless（費用段落位置與可見性、提醒列兩種樣式）。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Usage|FullyQualifiedName~Budget|FullyQualifiedName~Workbench"` 全過，再跑全部。
- **不要做**：不要改 `ConversationUsageTally` 與 token 的計算；不要在串流中即時更新費用；不要改 DI 註冊（整合包做）。

### 包 23L：設定首頁「本月用量」卡片加金額與預算（Wave 2）

- **目標**：首頁的本月用量卡片照第 6 節「設定首頁」顯示新台幣金額、預算列與進度條、「管理 ›」。
- **先讀這些檔案**：`src/Lumi.App/ViewModels/HomeViewModel.cs`（`HomeUsageCard`、`LoadActivityAsync`、`ApplyActivity`、`GoToCommand`）、`src/Lumi.App/Views/HomePanel.axaml`（`HomeUsageCard` 與 `HomeToolsCard` 標題列右邊「管理」按鈕的寫法）、`src/Lumi.Core/Usage/BudgetMonitor.cs`、`MonthlyUsageSummary.cs`、`MoneyFormat.cs`（23D）、`tests/Lumi.App.Tests/` 裡 `HomeViewModel` 的既有測試、本計畫第 6 節。
- **可以新增／修改的檔案**：`HomeViewModel.cs`、`HomePanel.axaml`、`HomeViewModel` 的既有測試檔（新增案例）、`tests/Lumi.App.Tests/Headless/` 裡首頁的既有 headless 測試檔（新增案例；沒有就新增 `HomeUsageCardTests.cs`）。
- **步驟**：
  1. `HomeUsageCard` 加欄位 `string? BudgetText`、`double? BudgetRatio`、`UsageLevel BudgetLevel`（< 0.8 Normal、< 1 Warning、其餘 Danger）；`MonthText` 改放金額（`MoneyFormat.Twd(月費用, 有估算)`），`TodayText` 改放今天的金額（`Home_UsageTodayFormat` 包 `MoneyFormat.Twd`）。`Loading`、`Unavailable` 兩個靜態值照改。
  2. `LoadActivityAsync`：除了既有的 token 查詢，另外用 `IUsageStore.ListAsync(本月起, 下月初)` 算本月與今天的費用、有沒有估算（用 `MonthlyUsageSummary.Build`），預算從 `IModelCatalog.Current.Usage.MonthlyBudgetUsd`。建構式加選用 `IUsageStore? usage = null`；null 時卡片維持舊行為（tokens）。任何一個查詢失敗照舊 `Unavailable`。
  3. `HomePanel.axaml`：標題列改成 `Grid ColumnDefinitions="*,Auto"`，右邊「管理 ›」按鈕照 `HomeToolsCard`，`CommandParameter="Usage"`；大數字後面的「tokens」`TextBlock` 在有 `IUsageStore` 時隱藏（綁 `Home.UsageCard.ShowTokensUnit`）；大數字下方加預算列（muted）與進度條（高 4、圓角 2、軌道 `LumiSurfaceAltBrush`、填色 `LumiAccentBrush`／`LumiWarningIconBrush`／`LumiDangerBrush`）；`Compact`／`narrow` 的既有樣式不改。改掉卡片上方「沒有金額與預算、沒有連結」的註解。
- **測試**：有費用與預算 → 大數字、預算列、比例、等級；沒設預算 → 沒有預算列；有估算 → 「≈」；沒有 `IUsageStore` → 舊行為（tokens）；查詢失敗 → `Unavailable`；`GoToCommand("Usage")` 送出 `SettingsSection.Usage`。Headless：三種狀態的可見元素、五套主題對比（仿照 `SettingsContrastTests`）。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Home"` 全過，再跑全部。
- **不要做**：不要改其他卡片；不要拿掉輸入／輸出 tokens 的小標籤；不要改 `SettingsViewModel`（`Usage` 的切頁由整合包接）。

---

### 包 23I：提示規則與紀錄（Wave 3）

- **目標**：9 條規則中可以從資料判斷的 7 條（2、3、4、5、6、8、9）都是純函式；第 1、7 條需要使用者動作，由 23J 判斷。提示紀錄寫進 `usage_hints`。
- **先讀這些檔案**：`src/Lumi.Core/Usage/UsageHints.cs`（Phase 0）、`src/Lumi.Core/Chat/ContextEstimator.cs`、`ContentBlock.cs`、`CompactionApplier.cs`（判斷哪些訊息已被摘要整理掉：`seq <= SummaryUptoSeq`）、`src/Lumi.Core/Context/ContextFormatter.cs`（`ContextBlock` 的檔名與內容欄位）、`SqliteUsageStore.cs`（23B，仿照寫法）、本計畫第 7 節。
- **可以新增／修改的檔案**：新增 `src/Lumi.Core/Usage/UsageHintRules.cs`、`UsageHintMatch.cs`、`src/Lumi.Infrastructure/Persistence/SqliteUsageHintLog.cs`；測試 `tests/Lumi.Core.Tests/Usage/UsageHintRulesTests.cs`、`tests/Lumi.Infrastructure.Tests/SqliteUsageHintLogTests.cs`。
- **步驟**：
  1. `UsageHintMatch(string Rule, IReadOnlyList<string> Args)`：`Args` 是提示文字的格式參數（已經格式化好的字串，例如 `"92K"`、檔名）。另有 `string? Key`：第 4 條用檔名、其餘 null（同一個檔案只提示一次用）。
  2. `UsageHintRules.AfterResponse(HintInput input) → IReadOnlyList<UsageHintMatch>`（依優先排序）。`HintInput(IReadOnlyList<ChatMessage> History, CompactionState Compaction, int ContextUsed, int ContextLimit, int TurnCount, int LastToolDefinitionTokens, bool IsFirstResponseInConversation)`。逐條實作第 7.2 節的 2、4、5、6、8、9：
     - 第 2 條：`ContextLimit > 0 && ContextUsed >= ContextLimit × 0.60`，或 `TurnCount >= 40` 且最近 10 則有 `Usage` 的助理訊息平均（輸入＋快取讀＋快取寫）≥ 60K。
     - 第 4 條：只看 `Seq > Compaction.SummaryUptoSeq` 的使用者訊息裡的 `ContextBlock`；token 用 `ContextEstimator.EstimateText(content)`；之後的助理訊息數 ≥ 20。多個符合時取最大的那個。
     - 第 5 條：同樣範圍內使用者訊息的 `ImageBlock`；張數 × `ContextEstimator.ImageTokens` 為 tokens。
     - 第 6 條：`IsFirstResponseInConversation && LastToolDefinitionTokens >= 10K`（7 天限制由 23J 查紀錄判斷）。
     - 第 8 條：輸出 ≥ 4K 且任一 `TextBlock` 含 ```` ``` ```` 的助理訊息數 ≥ 3，**且最後一則助理訊息本身符合**（才是「第 3 次的回應結束後」）。
     - 第 9 條：最後一則使用者訊息的文字以 `\n\s*\n` 分段，≥ 2K tokens 的段落（`ContextEstimator.EstimateText`）正規化（連續空白換成一個空格、去頭尾）後 SHA-256，和先前使用者訊息的段落比對；符合時回第一次出現的回合數（第幾則使用者訊息，從 1 起）。
  3. `UsageHintRules.OnResume(ChatMessage? lastAssistant, int contextUsed, DateTimeOffset now) → UsageHintMatch?`（第 3 條）：經過時間 ≥ 60 分鐘且上下文 ≥ 50K；時間參數 < 24 小時用 `Hint_ElapsedHoursFormat`（小時取整數，至少 1），否則 `Hint_ElapsedDaysFormat`——**格式化留給 App**，這裡 `Args` 放 `"h:3"` 或 `"d:2"`、與 `MoneyFormat.Tokens(contextUsed)`。
  4. `SqliteUsageHintLog : IUsageHintLog`（`internal sealed`）：實作 Phase 0 介面。
- **測試**：每條規則至少「剛好達到門檻觸發」「差一點不觸發」兩個案例；第 4、5 條「已被摘要整理掉的不算」；第 8 條「最後一則不符合時不觸發」；第 9 條「空白差異仍視為相同」「短段落不算」「回合數正確」；排序依優先；`SqliteUsageHintLogTests`（存讀、改 outcome、`WasShownAsync` 不把 suppressed 算進去、`LastShownAsync`、區間查詢）。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~UsageHint"` 全過。
- **不要做**：不要碰 UI；不要讀 `request_usage`（工具定義大小由呼叫端傳入）；規則裡不要寫死數字，一律用 `UsageHintThresholds`。

### 包 23J：提示的畫面（Wave 3）

- **目標**：9 條提示在第 7.2 節寫的時機與位置出現，頻率、「不再提醒」、紀錄都正確。
- **先讀這些檔案**：`src/Lumi.App/ViewModels/WorkbenchViewModel.cs`（`RunFinished`、`Regenerate`、開新對話、切換對話的處理）、`WorkbenchViewModel.Compaction.cs`（`CompactAndContinueCommand`）、`src/Lumi.App/Views/ChatComposer.axaml`（提示列放在輸入框外框上方）、`ChatComposer.axaml.cs`（輸入框焦點事件）、`src/Lumi.App/ViewModels/ModelSwitcherViewModel.cs`（`SelectAsync`）、`src/Lumi.App/Views/ModelPicker.axaml`、`src/Lumi.Core/Usage/UsageHintRules.cs`（23I；同時開工時先照本計畫 23I 的簽章寫）、本計畫第 7 節。
- **可以新增／修改的檔案**：新增 `src/Lumi.App/ViewModels/ComposerHintViewModel.cs`、`src/Lumi.App/Views/ComposerHint.axaml`＋`.axaml.cs`、`src/Lumi.App/Services/UsageHintCoordinator.cs`、`src/Lumi.App/ViewModels/WorkbenchViewModel.Hints.cs`；修改 `ChatComposer.axaml`、`ChatComposer.axaml.cs`、`ModelSwitcherViewModel.cs`、`ModelPicker.axaml`、`WorkbenchViewModel.cs`（只加呼叫點與建構式選用參數）；測試 `tests/Lumi.App.Tests/UsageHintCoordinatorTests.cs`、`ModelSwitchHintTests.cs`、`tests/Lumi.App.Tests/Headless/ComposerHintTests.cs`。
- **步驟**：
  1. `ComposerHintViewModel`：`IsVisible`、`Text`、`PrimaryText`／`PrimaryCommand`（可為 null）、`SecondaryText`／`SecondaryCommand`（可為 null）、`DismissCommand`、`MuteCommand`。
  2. `ComposerHint.axaml`：Warning 樣式 `Border`（同第 7.1 節）；第一行文字（`TextWrapping="Wrap"`）；第二行按鈕列：動作按鈕 `Classes="secondary" Height="32" Padding="12,0"`，最後是「知道了」「不再提醒這類」（`Classes="tertiary small"`）；按鈕間距 8、行距 12。放在 `ChatComposer.axaml` 輸入框外框之上，`Margin` 底部 8。
  3. `UsageHintCoordinator`（建構式 `(IUsageHintLog log, IModelCatalog catalog, TimeProvider time)`）：`Task<bool> TryShowAsync(string conversationId, IReadOnlyList<UsageHintMatch> matches, ChatMode mode, ComposerHintViewModel target)`：依序取第一個「沒被 mute、這則對話沒提示過（第 3 條例外；第 4 條以 `rule + Key` 判斷；第 6 條看 `LastShownAsync` 是否超過 7 天）」的寫 `Shown` 並顯示，其餘寫 `Suppressed`；小精靈模式全部寫 `Suppressed`。按鈕：知道了 → `Dismissed`；不再提醒 → `Muted`＋`SetUsageAsync`（`MutedHints` 加入）；動作按鈕 → `Acted`。提示文字依規則與 `Args` 組 `Hint_*` 字串（第 3 條的時間在這裡格式化）。
  4. `WorkbenchViewModel.Hints.cs`：
     - `RunFinished`（工作台模式）之後：組 `HintInput`（`ContextUsed`、`ContextWindow`、`Turns`、這次最後一筆 `request_usage` 的 `ToolDefinitionTokens`——從 `IUsageStore.ListAsync` 取本對話最近一筆、`IsFirstResponseInConversation` = 歷史裡只有一則使用者訊息），呼叫 `UsageHintRules.AfterResponse`；再加第 7 條（下一點）；交給 coordinator。
     - 第 7 條：記憶體 `Dictionary<string, int>`（最後一則使用者訊息 id → 重新產生次數）；`RegenerateCommand` 完成時 +1，達到 3 次時加入 `regenerate` 比對（`Args = ["3"]`）。
     - 第 3 條：`ChatComposer` 輸入框 `GotFocus` → 呼叫 `OnComposerFocusedAsync()`；每次打開或切到一則對話後第一次聚焦才檢查（旗標在切換對話時重設）。
     - 送出、切換對話、開新對話時 `ComposerHint` 收起並記 `Dismissed`。
     - 動作：「開新對話」→ `NewConversationCommand`；「整理並繼續」→ `CompactAndContinueCommand`；「管理工具」→ 設定視窗工具分類 MCP 分頁（`ISettingsWindowService` 既有方法，沒有就停下來回報）；「繼續」→ 只記 `Acted`。
  5. 第 1 條（`ModelSwitcherViewModel`）：建構式加選用 `Func<int>? currentContextTokens` 與 `Func<Task>? newConversation`。`SelectAsync` 選到的不是目前模型，且 `currentContextTokens() >= 20K`、規則沒被 mute、這則對話沒提示過 → 不換，設 `PendingSwitch`（含目標 entry 與提示文字），`ModelPicker.axaml` 在清單上方顯示確認區（Warning 樣式，文字＋「開新對話再換」(secondary)「照樣換」(primary)）。「開新對話再換」→ `newConversation()` 再換；「照樣換」→ 直接換。兩者都記 `Acted`；關掉選單沒選 → `Dismissed`。紀錄透過建構式選用的 `UsageHintCoordinator`。
- **測試**：coordinator（優先順序、只一則、mute 後 suppressed、同對話不重複、第 3 條可重複、第 4 條同檔案不重複、第 6 條 7 天、小精靈只記錄）；`WorkbenchViewModel`（回應結束後出現、送出時收起、重新產生第 3 次出現、聚焦時第 3 條出現且只一次）；`ModelSwitchHintTests`（上下文 < 20K 直接換、≥ 20K 先確認、兩個按鈕的行為）；Headless（提示列樣式、按鈕數量、可讀性）。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Hint|FullyQualifiedName~ModelSwitch|FullyQualifiedName~Workbench"` 全過，再跑全部。
- **不要做**：不要在回應中途顯示；不要顯示金額；不要新增第三種提示位置；不要改 DI 註冊。

### 包 23K：MCP 伺服器頁顯示工具說明大小（Wave 3）

- **目標**：設定 › 工具 › MCP 伺服器清單每一列多一行「工具說明約 3.2K tokens」。
- **先讀這些檔案**：`src/Lumi.App/ViewModels/McpServersViewModel.cs`、`src/Lumi.App/Views/McpServersPanel.axaml`、`src/Lumi.Infrastructure/Persistence/SqliteMcpToolCache.cs`（取得各伺服器的工具清單）、`src/Lumi.Core/Tools/ToolDefinition.cs`、`ToolSnapshotJson`。
- **可以新增／修改的檔案**：`McpServersViewModel.cs`（與它的列項目類別）、`McpServersPanel.axaml`、對應的既有測試檔。
- **步驟**：每個伺服器以快取的工具定義計算 `ContextEstimator.EstimateText(ToolSnapshotJson.Serialize(definitions))`，顯示 `Mcp_ToolDefinitionSizeFormat`（數字用 `MoneyFormat.Tokens`；< 1,000 時照實際數字）；沒有快取時不顯示。muted 12px，放在伺服器名稱下方的說明行之後。
- **測試**：有快取顯示、沒有快取不顯示、數字格式。
- **完成檢查**：`dotnet build` 零警告；`dotnet test --filter "FullyQualifiedName~Mcp"` 全過。
- **不要做**：不要為了計算去連線 MCP 伺服器。

---

### 整合（協調者）

1. **Wave 1 之後**：DI 註冊——`IUsageStore` → `SqliteUsageStore`、`IUsageRecorder` → `UsageRecorder`（傳進 `ConversationService`）、`IPriceTableSource` → `PriceTableSource`、`BudgetMonitor`、`IImageDownscaler` → `SkiaImageDownscaler`（傳進 `ContextAttachmentBuilder`）；`CompositionRootTests` 補上。全體建置與測試後再開 Wave 2。
2. **Wave 2 之後**：
   - `SettingsCategory` 在 `Connections = 1` 之後插入 `Usage = 2`，其後各值加 1；`SettingsViewModel` 的導覽處理 `SettingsSection.Usage`（首頁卡片、用量面板、預算提醒列都開到這裡）；`HomeViewModel` 的 DI 傳入 `IUsageStore`（23L）；`SettingsWindow.axaml` 加 Menu 項目（`LumiIconBarChart`、`Settings_CategoryUsage`、`Settings_CategoryUsageHint`）與頁面（`UsagePanel`，顯示時呼叫 `LoadAsync`）；`SettingsViewModel`／`ISettingsWindowService` 加開到用量頁的方法；23G 回報缺少的「開啟對話並回到工作台」方法在這裡補。檢查所有以數字寫死分類索引的測試。
   - 啟動流程呼叫 `PricingAutoFill.RunAsync`（在設定與資料庫都就緒之後、背景執行、失敗只記 log）。
   - `BudgetAlertCoordinator` 註冊並在啟動時建立。
   - 全體建置與測試。這時可以先發一版（見可行性評估）。
3. **Wave 3 之後**：`IUsageHintLog` → `SqliteUsageHintLog`、`UsageHintCoordinator` 註冊。
4. **文件**：`docs/ui-spec.md`（用量與費用頁、模型單價段落、用量面板費用段落、預算提醒列、輸入框提示列、模型切換確認區）；`agent.md`（新表、費用記錄的位置、`purpose` 值）；`plan/manual-verification.md` 新增「M23」表（內容見下方人工驗收清單）；`plan/misc-backlog.md` 第 4 項狀態改為「已排入 M23」；主計畫的里程碑表與決定紀錄。
5. 全方案建置零警告、全部測試通過。

## 驗收標準

（以下以自動測試驗證完成；需要實機、真實金鑰或視覺判斷的部分在人工驗收清單。）

- [x] 每次呼叫供應商（含工具迴圈的每一次、摘要）在 `request_usage` 各有一列；失敗且沒收到任何事件的請求沒有列。
- [x] 費用照當時的單價快照計算；之後改單價，舊紀錄的費用不變。
- [x] 重新產生、刪除對話之後，本月費用不變少。
- [x] 沒有 usage 的請求以估算值計入，畫面顯示「≈」。
- [x] 不支援 `stream_options` 的相容服務：自動不帶重送，對話正常，用量為估算。
- [x] 新增模型時，價目表裡完全相符／正規化後相符的 model ID 自動帶入單價、上下文上限、最大輸出；近似只列候選、不自動套用；本機連線顯示不計費。
- [x] 升級後第一次啟動，既有模型自動帶入單價（只做一次、不改上下文上限）。
- [x] 「更新價目表」成功時日期更新；失敗時沿用舊表並顯示訊息。
- [x] 預算跨 80%、100% 各提醒一次（一起跨過只提醒 100%）；換月重新計算；工作台與小精靈模式各自用對的位置；回應中途不出現。
- [x] 用量頁：本月金額、日期範圍、預算列與進度條、預測句（7 天以上）、每日長條、前 5 對話、出現過的情況、頁尾；預算可設定、修改、清除。
- [x] 用量面板：本對話與本月 ／ 預算兩行；沒設預算時只有金額；「用量與費用 ›」開到正確分類。
- [x] 設定首頁「本月用量」卡片顯示新台幣金額、預算列與進度條（有預算時）、今天的金額；「管理 ›」開到用量頁；輸入／輸出 tokens 標籤仍在。
- [x] CSV 匯出可用 Excel 開啟、中文不亂碼，欄位齊全。
- [x] 9 條提示各自在對的時機與位置出現；同時只一則；頻率限制正確；「不再提醒這類」生效且可在用量頁重新開啟；小精靈模式不顯示只記錄；每次觸發都有紀錄。
- [x] L1：過長的工具結果保留頭尾並寫出省略行數。L2：同工具同參數連續 3 次失敗提早停下。L3：同一則對話的 system 與工具定義在請求之間不變（測試）。L5：大圖縮到長邊 1568px。
- [x] 全方案建置零警告、全部測試通過；架構測試（相依、間距、色票）通過。

## 人工驗收清單

（已搬進 `plan/manual-verification.md` 的「M23」表 M23-01～10，尚未執行。）

1. **公司 Gateway 實測**（第 3 節）：非串流確認有 `usage`；串流加 `stream_options` 確認 `[DONE]` 前有 usage chunk；同一個超過 1,024 tokens 的 prompt 打兩次，第二次 `cached_tokens > 0`。在 Lumi 裡對話一次，`request_usage` 的 `usage_estimated` 是 0。
2. **Anthropic 直連**：開思考、用工具的一輪回應，`request_usage` 的列數＝呼叫次數，快取讀寫有值；費用和 Anthropic Console 同一段時間的用量大致相符（±10%）。
3. **Ollama**：顯示「本機模型，不計費」；用量照記、費用 0。
4. 停止回應：回應到一半按停止，該次以「≈」計入。
5. 模型設定：新增 `claude-…`（完全相符）、`gpt-4o-prod`（近似）、亂打的 ID（找不到）三種，畫面與帶入結果正確；「更新價目表」在公司網路（PAC＋Proxy）下成功。
6. 預算：設一個很小的預算，看到 80% 提醒（Warning）與 100% 提醒（Danger），各一次；小精靈模式下是小精靈說一句。
7. 用量頁：五套主題 × 淺深色下 v2-1～v2-3 對照畫板，文字與背景對比清楚；長條圖今天的顏色；前 5 對話點了會開啟並回到工作台。設定首頁的本月用量卡片在寬、窄兩種視窗寬度下版面正常，金額和用量頁一致。
8. CSV：Windows Excel 直接雙擊開啟，中文、數字欄位正確。
9. 9 條提示各觸發一次的實機確認：文字、位置、只出現一次、「不再提醒」生效、用量頁「這個月出現過的情況」列出、「重新開啟」生效。（觸發方法：1 換模型；2 長對話到圓環變橘；3 把電腦時間往後調或等 1 小時；4 加一個大檔案再聊 20 輪；5 貼 3 張圖再聊 10 輪；6 啟用多個 MCP 伺服器開新對話；7 重新產生 3 次；8 連續要求 3 次完整程式；9 同一段長文貼兩次。）
10. 大圖：貼一張 4K 截圖，附件資料夾裡的檔案長邊是 1568px，模型仍看得懂。

## 使用者確認紀錄（2026-10-30）

- **Q1** 第 4、5 條的動作改成「整理並繼續」「開新對話」：同意 → 決定 18。
- **Q2** 設定首頁要不要放「本月 NT$／預算」：要 → 決定 19，放在既有的「本月用量」卡片（包 23L）。
- **Q3** 預算提醒文字：沒有提供畫板原文 → 決定 20，採用協調者擬的版本。

## 實作紀錄

分支 `m23-usage-cost`；子代理工作在 `../DesktopAgent-wt/w23<包>`（照 [[subagent-worktrees-manual]] 從 Phase 0 的 commit 建立），由協調者合併回來。

- **2026-10-31 Phase 0 完成**（commit `d3a7293`）：
  - `src/Lumi.Core/Usage/` 新增計畫列的型別（`ModelPricing`、`PriceSnapshot`、`UsageRecord`、`IUsageStore`、`IUsageRecorder`、`UsageSettings`、`Money`、`UsageHintIds`／`UsageHintThresholds`／`IUsageHintLog`、`IImageDownscaler`）。
  - `ModelSettings.Pricing`、`AppSettings.Usage`、`IModelCatalog.SetUsageAsync`；`JsonModelCatalog` 的檔案 DTO 要另外加 `Usage` 欄位（計畫沒寫到：設定檔是手寫的 `SettingsFile` DTO，不是直接序列化 `AppSettings`）；`ModelSettings.Pricing` 直接隨模型序列化。
  - migration 12（`request_usage`、`usage_hints`）。既有的「模擬舊版資料庫」升級測試要在 `UPDATE schema_version` 前加 `DROP TABLE IF EXISTS request_usage; DROP TABLE IF EXISTS usage_hints;`（共 4 個測試檔），`SqliteCompactionTests` 的硬編碼版本 11 改成 `LatestVersion`。**之後每次加 migration 都要重複這件事。**
  - 價目表快照：LiteLLM 原檔 3.1 MB、4,538 筆（含 `sample_spec`），快照日期 2026-10-31；`Lumi.Web.csproj` 設成 EmbeddedResource；README 加「第三方資料」段落記來源與 MIT 授權。
  - 字串（約 100 條，`Usage_CostNote` 改新文字）、`LumiIconBarChart`、`SettingsSection.Usage`、兩邊 `FakeModelCatalog` 的 `SetUsageAsync`、`InMemoryUsageStore`／`InMemoryUsageHintLog`（Core 與 App 測試各一份）。
  - 測試：`UsageContractTests`（9）、`JsonModelCatalogUsageTests`（4）；全方案建置零警告、全部測試通過。
- **2026-10-31 Wave 1 完成**（23A–23E，Haiku 5.5 子代理平行實作，協調者逐包審查後合併；全方案零警告、全部測試通過）：
  - **23A 價目表與比對**：`PriceTable`／`PriceMatcher`／`JaroWinkler`／`IPriceTableSource`、`LiteLlmPriceParser`、`PriceTableSource`（internal）。內建表解析出 3,126 筆聊天模型。協調者在子代理回報後的決定：
    ① `PriceTableSource` 的 `dataDirectory` 改成**必填建構式參數**（`Lumi.Web` 只能參考 `Lumi.Core`，不能用 `LumiPaths`），整合包傳入 `LumiPaths.DataDirectory`；
    ② 正規化多去掉 Bedrock 的廠商前綴（`anthropic.`、`meta.`、`amazon.`、`mistral.`、`cohere.`、`ai21.`、`deepseek.`、`openai.`；LiteLLM 內有 `us.anthropic.claude-…-v1:0` 這種鍵名）；
    ③ **本機判斷不用 `ChatFailureText.IsLocalTarget`**（它把 10.x、172.16–31、192.168.x 與無點主機名也當本機，公司內網 Gateway 會被當成免費）；新增 `LocalConnection.IsLoopback`（只認 localhost、127/8、::1、0.0.0.0），`PriceMatcher` 與 `UsageRecorder` 都用它。計畫決定 9 的「本機」以此為準；
    ④ 前綴候選與近似候選合併去重，最多 3 筆。
  - **23B 費用紀錄**：`CostCalculator`、`UsageRecorder`、`SqliteUsageStore`，`ConversationService` 兩個呼叫點與 L2。實作細節：估算輸出把思考文字也算進去，但不放進「已停止」訊息的內容（另用 `CallUsage` 追蹤）；摘要呼叫用 `finally` 記錄（有 `yield` 的區塊不能用 `catch`）；`ListTopConversationsAsync` 略過全部沒單價的對話。協調者補的：L2 不把使用者按的「拒絕」（`ToolApproval.Denied`）算成工具失敗；`CompactSummaryAsync` 註解補上 L6 的理由。
  - **23C**：`stream_options` 被拒（400 且內文含 `stream_options`）時不帶重送一次並記住連線（只在記憶體）；`ToolExecutor.Truncate` 改保留開頭 70%／結尾 30%，中間寫出省略行數（換行對齊最多找 200 字元）。
  - **23D**：`MoneyFormat`、`MonthlyUsageSummary`、`BudgetMonitor`。`MoneyFormat.Tokens` 與 `WorkbenchViewModel.FormatTokens` 規則完全相同（兩邊要一起改）；`BudgetMonitor` 用 `SemaphoreSlim` 避免兩個回應同時結束時重複提醒。
  - **23E**：`SkiaImageDownscaler`（SkiaSharp 2.88，用 `SKFilterQuality.High`）。**帶 EXIF 旋轉標記的照片不縮**（縮圖會丟掉方向資訊，保守處理；手機照片因此不會變小，之後可改成縮前先套用旋轉）；帶透明的 WebP 會轉 JPEG 失去透明（照計畫，未另外處理）。
  - **踩到的坑**：完整 `dotnet test` 在本機同時跑多個 worktree 後變得很慢（App.Tests 30 分鐘）。子代理之後只跑篩選過的測試，完整測試由協調者在整合時跑一次。
- **2026-10-31 Wave 2 完成**（23F、23G、23H、23L，Haiku 5.5 子代理；協調者審查、合併並做 Wave 2 整合）：
  - **23F**：`ModelPricingViewModel`、`PricingAutoFill`（`app_meta` 旗標 `pricing_autofill_done`）、`ModelsViewModel`／`ModelsPanel.axaml` 的單價段落。`IPriceTableSource` 是 `ModelsViewModel` 的選用參數，沒註冊時整段隱藏。單價欄位本身沒有單位文字（摘要行有）；單價有錯誤時 `CanSave` 擋住儲存；新增模型時上下文上限／最大輸出只在 model ID 改變時才由價目表覆蓋。小缺口：設定頁編輯別處的連線 URL 時，已開著的模型表單不會自動重新比對。
  - **23G**：`UsageSettingsViewModel`、`UsagePanel`、`DailyCostBars`、`UsageCsvExporter`、`UsageHintSummaryItem`。協調者補的：① 本月沒有用量時「設定每月預算」連結與編輯器仍然顯示（原本空月份整段隱藏，沒用量的人就無法先設預算）；② CSV 的文字欄位以 `= + - @` 開頭時前面補單引號，避免 Excel 當公式執行（對話標題可能是使用者貼進來的）。預算百分比一律無條件捨去（99.6% 顯示 99%），首頁卡片（23L）同一規則。
  - **23H**：用量面板費用兩行、`BudgetAlertViewModel`／`BudgetBanner`／`BudgetAlertCoordinator`（定義 `ISpriteNotice`）。`WorkbenchViewModel` 建構式加了選用的 `IUsageStore`、`BudgetMonitor`（計畫沒列這個檔案，但無法避免）。沒做「開面板時重新查詢」：費用在載入／切換對話、每次 `RunFinished`、整理完成時更新（缺口：App 一直開著跨月、之後沒有任何回應時，本月數字晚一步更新）。預算提醒文字沒有「≈」前綴（`BudgetMonitor` 不回傳是否有估算）。
  - **23L**：首頁「本月用量」卡片。「管理 ›」走 `SettingsViewModel.GoToCommand`（和工具卡片相同），不經 `HomeViewModel.NavigateRequested`。進度條放三條 `ProgressBar` 依等級切換顯示。
  - **Wave 2 整合**：
    - DI：`Lumi.Infrastructure` 註冊 `SqliteUsageStore`／`IUsageStore`／`UsageRecorder`／`BudgetMonitor`；`Lumi.Core` 預設 `NullUsageHintLog`（Wave 3 由 SQLite 版取代）；`Lumi.Web` 新增 `AddLumiPriceTable(dataDirectory)`（`PriceTableSource` 是 internal，App 不能直接 new）；`AddLumiApp` 註冊 `IImageDownscaler`、`UsageSettingsViewModel`（工廠，`openConversation` = 回到工作台再開對話）、`SpriteNotice`（`SpriteBubbleViewModel.ShowNotice`）、`BudgetAlertCoordinator`。
    - 啟動：`App.StartAsync` 建立 `BudgetAlertCoordinator`，背景執行 `PricingAutoFill`。
    - 設定視窗：`SettingsCategory.Usage = 2`，原本的 2–10 各加 1；`SettingsWindow.axaml` 新增選單項目與頁面（頁首：眉標「用量與費用」、標題「這個月花了多少」，沒有說明文字）；切到該分類時自動 `LoadAsync()`；`UsageSettingsViewModel.OpenPricing` 由 `SettingsViewModel` 接到 AI 連線頁。`SettingsViewModel` 建構式在 `home` 之後多了 `UsageSettingsViewModel usage`。
    - **測試裡寫死分類編號的地方全部改成用 `SettingsCategory` 列舉**（`SettingsRig` 等 7 個檔案原本寫 `CategoryIndex = 2/3/5…`，選單插入一項就整批指到別頁）；選單數量 11 → 12；鍵盤順序測試補上 `Usage`；`CompositionRootTests` 補上新服務。
    - 架構測試：預算提醒列的 `Margin="64,8,64,0"` 加入間距例外（和通知列、更新提醒同一條版面）。
- **2026-10-31 Wave 3 完成**（23I、23J、23K，Haiku 5.5 子代理；23J 等 23I 合併後才開，因為依賴規則的 `Args` 格式）：
  - **23I**：`UsageHintRules.AfterResponse`／`OnResume`（純函式，門檻全用 `UsageHintThresholds`）、`UsageHintMatch`、`SqliteUsageHintLog`。子代理自行判斷的細節（接受）：第 2 條的「最近 10 則」不足 10 則時不觸發；第 9 條比對先前的使用者訊息時不套用「已被摘要整理掉」的範圍（整份歷史都算）；第 4 條以 `ContextBlock.DisplayName` 區分檔案（不同路徑的同名檔視為同一個）；第 5 條的「≥ 5K tokens」實際上被「≥ 3 張」涵蓋；第 8 條沒有 usage 的助理訊息不計入。協調者另外在 Infrastructure 的 DI 以 `SqliteUsageHintLog` 取代 Core 預設的 `NullUsageHintLog`。
  - **23J**：`ComposerHintViewModel`／`ComposerHint`、`UsageHintCoordinator`、`WorkbenchViewModel.Hints.cs`、`ModelSwitcherViewModel` 的確認區（`PendingSwitch`）與 `ModelPicker.axaml`、`ChatComposer.axaml` 根節點由 `Border` 改成 `StackPanel`（上方加提示列）。與計畫不同、已接受的：重新產生的計數 key 用「對話 id ＋ 目前的回合數」（`MessageViewModel` 沒有 Id，`RegenerateAsync` 是同步路徑）；只有正常結束的回應才判斷提示（停止、失敗不算）；「管理工具」開到工具清單分頁（不是 MCP 分頁）；`RetrySwitcher`（錯誤列換模型重試）不接提示。協調者補的：`WorkbenchWindow.axaml.cs` 接上模型選單 `Flyout.Closed` → `DismissPendingSwitchAsync()`（選單關掉沒選就收起確認區並記為略過）；`ChatComposer.axaml` 提示列外層的 `Margin="64,8,64,8"` 加入間距例外；`UsageHintCoordinator` 註冊成 singleton。已知小缺口：「不再提醒這類」寫設定失敗時例外沒有接住（沒有錯誤顯示位置）。
  - **23K**：資料來源改用 `IMcpToolSource.GetTools()`（連線中的即時清單，否則快取，不需連線）；顯示的是該伺服器**全部**工具定義的大小，伺服器斷線或個別工具被關掉時會比實際每次請求送出的大；字級沿用 muted（11px，專案沒有 12px token）。
- **2026-10-31 整體結果**：全方案建置零警告；全部測試通過（App 2,417、Core 1,447、Infrastructure 541、Tools 1,294、Providers 190、Web 182、Architecture 131 等）。文件已更新：`docs/ui-spec.md` 第 15 節、`agent.md` 第 7 節、`plan/manual-verification.md` 的 M23 表、`plan/misc-backlog.md` 第 4 項。
- **已知限制與後續（沒有排進 M23）**：① 帶 EXIF 旋轉標記的照片不縮圖；帶透明的 WebP 縮圖後失去透明。② 預算提醒文字沒有「≈」標記。③ 費用數字不在串流中即時更新，且 App 開著跨月、之後沒有任何回應時，本月數字會晚一步更新。④ 設定頁編輯別處的連線 URL 時，已開著的模型表單不會自動重新比對價目表。⑤ 「不再提醒這類」寫設定失敗時沒有錯誤顯示。⑥ 圖片生成（22F）的費用不計。⑦ LLM 使用習慣分析、匯率設定、每日預算等仍在 backlog「#4 先不做」。⑧ 提示門檻都是初始值，等有實際資料後用 `usage_hints` 與 `request_usage` 校準。
- **2026-10-31 調整：「不計費」改成使用者勾選**（決定 21）。原本借用 `ChatFailureText.IsLocalTarget`（M9 9C 的錯誤訊息用，不是計費用）判斷「本機免費」，後來收窄成只認 loopback，但使用者的地端與雲端模型都在同一個 AI Gateway 後面，網址分不出來，這個啟發式不成立。改為：`ModelPricingViewModel.IsFree`（模型設定單價段落最上方的勾選，新字串 `Pricing_FreeToggle`、`Pricing_FreeNote`，取代 `Pricing_Local`）；勾了不比對、不顯示欄位、存成 `IsFree = true, Source = Confirmed`；取消勾選會重新比對價目表。移除 `LocalConnection`、`PriceMatchKind.Local`、`UsageRecorder` 與 `PricingAutoFill` 的免費判斷。沒設單價的 localhost 連線現在是「未設定單價」，不計入預算。上面 23A 紀錄的第③點與 M23-03 人工驗收已隨之更新。

