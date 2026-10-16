# Lumi 開發指引（給 Agent）

這份文件是寫給協助開發的 AI agent 與人類開發者的「不可偏離的方向」。
動手前先讀完；遇到這裡沒寫到、又會影響架構的決定，**停下來問，不要猜**。

---

## 1. 專案是什麼

Lumi 是一個跨平台（Windows / macOS）的桌面 AI 助理，使用者自帶 API 金鑰（BYOK）。

- **工作台模式**：一般的對話視窗。可以加入檔案或資料夾當上下文、切換模型、語音輸入，下方顯示 token 與回合數。
- **小精靈模式**：透明無框的桌面角色（第一個角色是光球）。游標靠近才浮出精簡輸入列，用簡短氣泡回答，並用動畫表現思考、呼叫工具等過程。
- 兩種模式**共用同一則對話**，只是外觀與回答長度不同。

## 2. 開始工作前要讀

1. `README.md`：專案概觀
2. `plan/00-master-plan.md`：里程碑與已定案的決策
3. 目前要執行的那份 `plan/NN-*.md`
4. 動到小精靈時：`docs/sprite-dev-guide.md`
5. 動到工作台或設定視窗的介面時：`docs/ui-spec.md`

## 3. 工作方式

- **只做目前計畫範圍內的事。** 發現計畫外需要改的地方，先提出來，不要順手改。
- 完成一個項目就在計畫檔裡打勾，並在「實作紀錄」寫下偏離原計畫的地方與原因。
- 「主計畫的決策紀錄」裡已定案的事項，不要重新討論或替換。
- 沒有規格的地方（例如 Whisper 的回應格式）不要自己發明格式，留 `TODO` 並回報。

## 4. 技術棧（固定，不要替換）

| 用途 | 採用 | 不要用 |
|---|---|---|
| 執行環境 | .NET 10 | — |
| UI | Avalonia 11.3.x | WPF、MAUI、WinForms |
| MVVM | CommunityToolkit.Mvvm | ReactiveUI（個別地方需要節流或合併事件時，可局部用 System.Reactive） |
| DI / 設定 / Log | Microsoft.Extensions.Hosting（DI、Options、Logging） | 其他 DI 容器 |
| JSON | System.Text.Json（含 JsonSchemaExporter） | Newtonsoft.Json |
| 本機資料 | SQLite（Microsoft.Data.Sqlite） | — |
| HTTP | IHttpClientFactory + typed client | 自己 new HttpClient |
| 測試 | xUnit | — |

不要為了功能夾帶 Python、Node 或其他執行環境，一律用 C# 實作。

## 5. 分層與相依方向

```
Lumi.App ──► Lumi.Sprites.* ──► Lumi.Sprites.Abstractions
   │
   ├──► Lumi.Providers ─┐
   ├──► Lumi.Tools ─────┤
   ├──► Lumi.Speech ────┼──► Lumi.Core
   ├──► Lumi.Infrastructure ┘
   └──► Lumi.Platform
```

| 專案 | 職責 | 規則 |
|---|---|---|
| `Lumi.Core` | 資料模型、所有核心介面、對話迴圈、Prompt 組成 | **不得引用 Avalonia**，也不得引用任何供應商 SDK |
| `Lumi.Providers` | Anthropic 原生、OpenAI 相容 | 只透過 `IChatProvider` 對外；各家格式轉換留在這一層 |
| `Lumi.Tools` | 內建工具 | 每個工具都是 `ToolBase<TArgs>` |
| `Lumi.Speech` | 公司內部 Whisper 服務的客戶端 | 只透過 `ISpeechToText` 對外 |
| `Lumi.Infrastructure` | SQLite、設定檔、金鑰安全儲存 | 實作 Core 的儲存介面 |
| `Lumi.Platform` | 所有 P/Invoke（游標、點擊穿透、錄音） | **原生呼叫只能出現在這裡** |
| `Lumi.Sprites.Abstractions` | 角色介面、狀態、manifest、`SpriteLimits` | 帶版本號；改介面要升版並記錄 |
| `Lumi.Sprites.*` | 各個角色 | 只畫自己，不碰視窗、平台、對話 |
| `Lumi.App` | Avalonia UI、工作台、小精靈殼、設定頁、DI 組裝 | 唯一的組裝點（composition root） |

## 6. 核心介面（名稱固定）

| 介面 | 用途 |
|---|---|
| `IChatProvider` | 傳送對話、回傳 `IAsyncEnumerable<ChatEvent>` 串流 |
| `ITool` / `ToolBase<TArgs>` / `IToolRegistry` | 工具定義、執行、註冊 |
| `IToolApprovalService` | 需要確認的工具，向使用者請求同意 |
| `IPromptLayer` / `IPromptComposer` | 分層組成 prompt |
| `IConversationStore` | 對話與訊息存取 |
| `IModelCatalog` | 連線與模型設定（可多筆） |
| `ISecretStore` | 金鑰與 Token（Windows DPAPI / macOS Keychain） |
| `ISpeechToText` / `IAudioRecorder` | 語音轉文字 / 錄音 |
| `ISpriteFactory` / `ISpriteView` | 小精靈角色 |
| `IOverlayPlatform` | 游標輪詢、點擊穿透（POC 已驗證） |

所有實作都用 DI 註冊，不要用靜態單例或 service locator。

## 7. 對話與串流

- 對話迴圈輸出 `IAsyncEnumerable<ChatEvent>`。事件種類：`TextDelta`、`ThinkingDelta`、`ToolCallStarted`、`ToolCallCompleted`、`ApprovalRequired`、`Usage`、`Completed`、`Failed`。
- UI 只消費事件，不直接呼叫供應商。**小精靈的動畫狀態也是從這些事件推導出來的。**
- 所有非同步方法都要接受 `CancellationToken`，並且真的往下傳。
- SSE 解析：以「完整事件」為單位解析 JSON，UTF-8 解碼要有狀態。不要逐 byte 轉字串，否則 emoji 和中文會被切壞。
- token 用量以供應商回傳的 `usage` 為準。只有送出前的上下文預估才用估算值。
- 工具呼叫迴圈要有最大次數限制，預設 10 次，超過就停止並回報。

## 8. Tool 規則

- 一律繼承 `ToolBase<TArgs>`，參數用強型別 record；JSON Schema 由 `JsonSchemaExporter` 自動產生，不要手寫。
- `Descriptor` 必須寫清楚的 `Description`：模型要靠它判斷何時呼叫。
- 必須宣告**權限等級**：`ReadOnly`、`Write`、`Network`、`Destructive`。`Write` 以上預設要使用者確認。
  - 權限等級是必填項目，不給預設值；沒有宣告就無法編譯。移植過來的工具也要逐一補上。
  - `Write`、`Network` 可以選「本次對話都允許」（只存在記憶體）；`Destructive` 每次都問，不提供記住選項。
  - 確認 UI 與拒絕、停止時的行為見 `docs/ui-spec.md` 第 5 節。
- 必須宣告**可用模式**：工作台、小精靈，或兩者都可用。
- 檔案類工具只能存取 `ToolContext.Scope`（使用者加入的檔案與資料夾）。路徑要正規化並檢查，防止用 `..` 跳出範圍。
- 錯誤用 `ToolResult.Error(...)` 回傳給模型，不要讓例外直接往外丟。
- 每個工具都要有逾時設定。
- 註冊方式：`services.AddLumiTool<MyTool>()`。
- 外部工具（MCP、移植過來的 WPF 工具）一律透過 Adapter 實作 `ITool`，不要另開一條呼叫路徑。

## 9. Prompt 組成

由 `IPromptComposer` 依序組合各個 `IPromptLayer`：

| 順序 | 層 | 放置位置 |
|---|---|---|
| 1 | 基底（身分、工具規則、安全規則） | system |
| 2 | 人設（個性、積極度、自訂人設） | system |
| 3 | 模式格式（工作台 / 小精靈） | **最新一則使用者訊息的後面** |
| 4 | 角色語氣（只在小精靈模式） | 同上 |

- 衝突時的優先順序：**基底 > 模式格式 > 人設 > 角色語氣**。
- system 內容要保持穩定，**不要因為切換模式而改動 system**，否則 Anthropic 的 prompt 快取會失效。
- 小精靈模式的限制：最多兩句、約 60 字、不用 Markdown；需要長篇時呼叫 `show_details` 工具；`max_tokens` 上限較低；氣泡也有字數上限並會截斷。

## 10. 模型與金鑰

- 「連線」（供應商、Base URL、金鑰）和「模型設定」是兩層，各自可以新增多筆。
- 工作台與小精靈可以各自指定預設模型。
- **金鑰與 Token 只能存在 `ISecretStore`**，不能出現在設定檔、SQLite、log、例外訊息或 UI 截圖中。log 一律遮蔽。

## 11. 語音

- 一律透過 `ISpeechToText`。第一個實作是公司內部的 Whisper 服務，規格見 `outter_sys_spec/whisper.md`（內部文件，不納入版控，只存在本機）。
- 錄音輸出 16kHz、單聲道 WAV。
- `source_pgm` 固定為 `Lumi`，`session_id` 帶入對話 ID。
- **回應格式尚未確認**：在使用者提供範例之前，不要實作解析邏輯。

## 12. UI 規則

- MVVM：使用 CommunityToolkit.Mvvm（`[ObservableProperty]`、`[RelayCommand]`）。code-behind 只放純視覺的東西，例如動畫、視窗行為。
- 背景執行緒不要直接改 UI，要透過 `Dispatcher.UIThread`。
- **所有顯示對話內容的地方都要經過 `EmojiText`**（切成字形叢集、emoji 叢集指定 emoji 字型）。不要直接設定 `TextBlock.Text`，否則 ❤️‍🔥 這類組合 emoji 會被拆開。
- 介面文字使用繁體中文，集中放在資源檔。
- **字型一律打包進 app**（IBM Plex Sans、Noto Sans TC、JetBrains Mono），元件只引用 `LumiUiFont`／`LumiMonoFont` 資源，不直接寫字型名稱，也不依賴系統字型。
- **顏色一律引用主題 token**（Light／Dark 兩套，預設跟隨系統），不在元件裡寫死色碼。token 清單見 `docs/ui-spec.md` 第 1 節。
- 回應中（含等待工具確認）停用模式切換與模型切換；小精靈回應中仍可「開啟工作台查看」，但工作台此時不能送出訊息（`docs/ui-spec.md` 2.2）。
- 小精靈的「殼」與「角色」嚴格分離，詳見 `docs/sprite-dev-guide.md`。

## 13. 繪製與效能（POC 實測得到的教訓）

- **`RadialGradientBrush` 不要讓 `GradientOrigin` 和 `Center` 不同。** Skia 會改用雙點錐形漸層，畫出一圈硬邊圓盤。要偏心的亮面時，改用同心漸層，把圓心移到亮點、半徑放大，再裁切。
- 柔邊效果：用多個色標，邊緣平滑收到 0。只有 2～3 個色標的線性漸層會出現「斷層圈」。
- 常駐畫面**不要用 `BlurEffect`**，用漸層模擬，比較省 GPU。
- 動畫統一由 `FrameDriver` 驅動（高幀率跟隨螢幕刷新，低幀率改用計時器）。元件不要自己開計時器。
- 小精靈閒置時要降低幀率，休眠時再降一級（POC：互動 60、待機 30、休眠 12）。

## 14. 平台

- P/Invoke 只放在 `Lumi.Platform`，並標上 `[SupportedOSPlatform]`。
- macOS 透過 `objc_msgSend` 呼叫 AppKit；Windows 透過 user32。
- 新增原生功能時，要同時提供兩個平台的實作，或明確提供退化行為。

## 15. 測試

- `Lumi.Core`、`Lumi.Tools`、`Lumi.Providers` 要有單元測試。
- Provider 測試使用錄製好的回應範本（fixture），**測試中不得呼叫真實 API**。
- 工具的範圍檢查（路徑跳脫）一定要有測試。

## 16. 禁止事項

- 在 `Lumi.Core` 引用 Avalonia。
- 把金鑰或 Token 寫進任何檔案或 log。
- 繞過 `IToolRegistry` 直接呼叫工具。
- 角色程式碼直接操作視窗、平台 API 或對話。
- 為了小精靈切換模式而修改 system prompt。
- 沒有規格就自行假設外部系統的回應格式。
- 未經討論就更換第 4 節的技術棧。
