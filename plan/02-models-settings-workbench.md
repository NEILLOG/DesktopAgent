# 02 模型管理、設定頁與工作台完整介面

- 狀態：2A 程式完成（待你實機確認搬家）；2B 待開工；2C～2E 只寫到會影響架構的程度，輪到時再展開
- 對應里程碑：M2
- 前置：M1 在 Mac 完成；Windows 驗證可以和 M2 並行，不擋開工

## 目標

不同供應商的多個模型可以在工作台切換使用；連線與模型在獨立的設定視窗管理；工作台補齊 ui-spec 第 2、3 節的互動。金鑰不出現在任何檔案或 log。

## M2 拆成五個部分

| 部分 | 名稱 | 產出 | 依賴 |
|---|---|---|---|
| **2A** | 模型目錄與設定儲存 | `IModelCatalog`、設定檔、從 M1 的開發設定搬家、依連線分派 Provider | M1 |
| **2B** | OpenAI 相容 Provider、測試連線 | OpenAI 相容串流、兩種格式的「列出模型」與「測試連線」 | 2A |
| 2C | 設定視窗 | 連線、模型設定兩個分頁；一般（主題） | 2A、2B |
| 2D | 工作台完整介面 | 模型切換膠囊、工具列、對話列表（最簡版）、思考過程、訊息動作、錯誤列、狀態列連線狀態 | 2A |
| 2E | Markdown 渲染 | Markdown 轉 Avalonia 元件，文字經過 `EmojiText`，程式碼區塊 | 1D |

2C、2D、2E 彼此不依賴，2A、2B 完成後可以任意順序做。

主計畫 M2 項目對應：

| 主計畫項目 | 部分 |
|---|---|
| 連線與模型兩層設定、各自多筆；測試連線；列出可選模型 | 2A、2B、2C |
| OpenAI 相容 Provider（OpenAI、Ollama、公司 Gateway） | 2B |
| 頂端模型切換選單；工作台與小精靈各自的預設模型 | 2A（資料）、2D（UI） |
| 工具列：設定、歷史對話、新對話、切換到小精靈；回應中停用切換 | 2D |
| 設定頁做成獨立視窗 | 2C |
| 思考過程區塊、錯誤列與重試；主題選擇 | 2D、2C |
| 底部狀態列 | 2D（M1 已有，補連線狀態） |
| Markdown 渲染（經過 emoji 處理） | 2E |

**不在 M2**（避免範圍膨脹，各自歸屬）：
- 對話列表的搜尋、刪除、重新命名、依日期分組 → M9。M2 的「歷史對話」只做能切換對話的最簡列表。
- 小精靈本身 → M7。M2 只存「小精靈預設模型」，工具列的「小精靈模式」按鈕先停用。
- 設定視窗的語音、人格、工具權限、小精靈分類 → 各自的里程碑再加，M2 不放空頁面。

---

## 開工前要確認的事

| # | 問題 | 建議 | 狀態 | 影響 |
|---|---|---|---|---|
| Q1 | 連線、模型設定、主題這些設定存哪？ | 存 JSON 檔 `settings.json`，和資料庫放同一個資料夾。寫入沿用 1B 的做法（先寫暫存檔再取代）。好讀、好備份、壞了容易手動修；資料量很小，不需要 SQLite 的交易。金鑰一樣只在 `ISecretStore`。 | ✅ 已確認 | 2A |
| Q2 | OpenAI 相容 Provider 用官方 OpenAI SDK 還是自己寫？ | **自己寫**：`HttpClient` + .NET 內建的 `System.Net.ServerSentEvents` 解析，只走 Chat Completions（`/v1/chat/completions`）。理由：「OpenAI 相容」的伺服器（Ollama、公司 Gateway）各自只實作一部分，官方 SDK 以 OpenAI 自己的 API 為準（新功能偏向 Responses API），遇到相容伺服器的差異反而難處理；自己寫可以對缺欄位寬容。和 M1 Anthropic 用官方 SDK 不同，原因是 Anthropic 只有一個標準實作。 | ✅ 已確認 | 2B |
| Q3 | 頂端的模型切換膠囊，切了之後算什麼？ | **改工作台的預設模型**（記住，下次開啟也是這個），不另外做「每則對話各自的模型」。每則助理訊息本來就記錄了是哪個模型回的。概念只有一個：「工作台現在用哪個模型」＝設定頁的「工作台預設模型」。 | ✅ 已確認 | 2A、2D |
| Q4 | 公司 Gateway 除了 Base URL 和金鑰，還需要什麼？ | 不需要：是標準的 OpenAI 相容格式（`Authorization: Bearer`），之前接 Copilot、Roo Code 都能直接用。2B 照標準格式做即可。 | ✅ 已確認 | 2B |

---

## 2A 模型目錄與設定儲存

### 範圍

**做**
- Core：`ConnectionSettings`、`ModelSettings`、`ProviderFormat`、`IModelCatalog`
- Infrastructure：`settings.json` 的讀寫（`JsonModelCatalog`）、一般設定（主題）
- 第一次啟動時從 M1 的 `Lumi:Dev:Anthropic` 搬家，既有金鑰繼續可用
- 依連線格式分派 Provider（`RoutingChatProvider`）；Anthropic Provider 改讀連線設定
- 工作台與小精靈的預設模型
- 刪除連線時一併刪除金鑰與底下的模型設定

**不做**
- 任何 UI（2C、2D）
- OpenAI 相容（2B；2A 只把分派的位置留好）

### 設計

#### 資料模型（`Lumi.Core/Models/`）

```csharp
public enum ProviderFormat { Anthropic, OpenAICompatible }

// 連線：一組供應商、Base URL、金鑰（金鑰在 ISecretStore，這裡只有提示）
public sealed record ConnectionSettings(
    string Id, string Name, ProviderFormat Format, string BaseUrl,
    string? ApiKeyHint);          // 「結尾 xxxx」，存金鑰時算好；沒有金鑰（例如 Ollama）時是 null

// 模型設定：掛在某個連線底下
public sealed record ModelSettings(
    string Id, string ConnectionId, string ModelId, string DisplayName,
    int MaxOutputTokens, int ContextWindow, bool ShowThinking);

public sealed record AppSettings(
    IReadOnlyList<ConnectionSettings> Connections,
    IReadOnlyList<ModelSettings> Models,
    string? WorkbenchModelId,     // ModelSettings.Id
    string? SpriteModelId,        // M7 才會用到，M2 先存
    ThemePreference Theme);       // System／Light／Dark
```

- 金鑰的名稱沿用 1B：`connection/{ConnectionId}/api-key`。
- `ApiKeyHint`：1B 說「M2 再決定」。決定存在設定檔（只有末 4 碼，不是祕密），設定頁顯示「已設定 · 結尾 xxxx」，不需要每次去讀鑰匙圈（Mac 可能跳授權視窗）。
- `ShowThinking`：關掉時不要求 `display: summarized`，也不顯示思考過程區塊。

#### `IModelCatalog`

```csharp
public interface IModelCatalog
{
    AppSettings Current { get; }
    event EventHandler<AppSettings>? Changed;      // UI 依此更新（模型選單、主題）

    Task SaveConnectionAsync(ConnectionSettings connection, SecretValue? newApiKey, CancellationToken ct = default);
    Task DeleteConnectionAsync(string connectionId, CancellationToken ct = default);   // 連同模型設定與金鑰
    Task SaveModelAsync(ModelSettings model, CancellationToken ct = default);
    Task DeleteModelAsync(string modelId, CancellationToken ct = default);
    Task SetWorkbenchModelAsync(string modelSettingsId, CancellationToken ct = default);
    Task SetSpriteModelAsync(string modelSettingsId, CancellationToken ct = default);
    Task SetThemeAsync(ThemePreference theme, CancellationToken ct = default);

    // 解析成一次請求要用的完整資訊；找不到（被刪了）時回 null
    ResolvedModel? Resolve(string modelSettingsId);
}

public sealed record ResolvedModel(ModelSettings Model, ConnectionSettings Connection);
```

- `Current` 是不可變的快照，每次儲存換一份新的，讀的人不需要上鎖。
- 刪掉目前的工作台預設模型時，預設改成剩下的第一個；一個都不剩就是 null（工作台送出時提示「請先到設定加入模型」）。

#### 設定檔（Q1）

- 位置：`LumiPaths.DataDirectory/settings.json`（Mac：`~/Library/Application Support/Lumi/`；Windows：`%APPDATA%\Lumi\`）。
- 格式帶 `"version": 1`，比程式新時拒絕讀取（同 1D 的資料庫規則），比程式舊時補到最新。
- 寫入：暫存檔 + 取代、程序內序列化（同 1B 的 `EncryptedFileSecretStore`，抽出共用的原子寫入工具）。
- 檔案損毀：改名為 `settings.json.broken-{時間}` 保留，用預設值啟動並在工作台提示一次。不直接覆蓋，避免使用者的設定消失。
- 架構測試加一條：設定檔的序列化型別裡沒有任何 `SecretValue`。

#### 從 M1 搬家

第一次啟動、`settings.json` 不存在時：
- 用 `appsettings.json` 的 `Lumi:Dev:Anthropic` 建一個連線，**Id 固定為 `dev-anthropic`**，名稱「Anthropic」。這樣 M1 存在鑰匙圈的金鑰（`connection/dev-anthropic/api-key`）不用搬就能繼續用。
- 建一個模型設定（`claude-opus-5-5`、上下文 1M、最大輸出 64000、顯示思考過程），設為工作台與小精靈的預設。
- 之後 `appsettings.json` 的 `Lumi:Dev` 不再讀取，從檔案移除；`AnthropicDevOptions` 刪掉，`ChatDefaults` 由 `IModelCatalog` 取代。

#### 依連線分派 Provider

- `ChatRequest` 改成帶 `ResolvedModel`（取代 M1 的 `ModelId` + `MaxTokens`），Provider 從裡面拿 Base URL、模型 ID、連線 Id（取金鑰用）、最大輸出、是否顯示思考。
- `RoutingChatProvider : IChatProvider` 依 `Connection.Format` 轉給 `AnthropicChatProvider` 或 2B 的 `OpenAICompatibleChatProvider`。UI 與 `ConversationService` 只認得 `IChatProvider`，不變。
- `ConversationService` 送出時用 `IModelCatalog.Current.WorkbenchModelId`（M7 的小精靈用 `SpriteModelId`）。
- **換模型時的歷史**：Anthropic 的 thinking 區塊、`ProviderBlock` 只送回 Anthropic 連線（1C 已是這樣）；送給 OpenAI 相容時只送文字。從 OpenAI 相容換回 Anthropic 時照常送出全部，API 會丟掉不認得的 thinking 區塊（不收費）。

### 工作項目

- [x] **A1 Core 模型與介面**：`ProviderFormat`、`ConnectionSettings`、`ModelSettings`、`AppSettings`、`ThemePreference`、`IModelCatalog`、`ResolvedModel`。
- [x] **A2 原子寫入工具**：從 1B 的 `EncryptedFileSecretStore` 抽出共用的「暫存檔 + 取代」寫入，兩邊共用。
- [x] **A3 `JsonModelCatalog`**：讀寫、版本、損毀處理、刪除連線時連同模型與金鑰、預設模型的維護、`Changed` 事件；測試。
- [x] **A4 M1 搬家**：第一次啟動建立 `dev-anthropic` 連線與模型；移除 `AnthropicDevOptions`、`ChatDefaults`、`appsettings.json` 的 `Lumi:Dev`；測試（既有金鑰名稱不變）。
- [x] **A5 分派**：`ChatRequest` 帶 `ResolvedModel`；`RoutingChatProvider`；Anthropic Provider 改讀連線；`ShowThinking` 關閉時不要求 summarized；更新 1C、1D 的測試。
- [x] **A6 工作台接上**：送出時用工作台預設模型；沒有任何模型時的提示；狀態列的上下文上限改讀模型設定。
- [x] **A7 架構測試**：設定檔型別不含 `SecretValue`。
- [x] **A8 文件**：`agent.md`、主計畫決策紀錄（Q1、Q3）、本檔實作紀錄。

### 驗收標準

- [x] `dotnet build` 零警告、`dotnet test` 全過。
- [ ] 從 M1 升級後第一次開啟：不用重新輸入金鑰就能繼續對話，舊對話還在。（搬家邏輯由組裝測試驗證；實機待你確認）
- [x] `settings.json` 裡搜尋不到金鑰；只有末 4 碼的提示。（測試保證）
- [x] 手動把 `settings.json` 改壞：程式能啟動、保留壞掉的檔案、提示一次。（測試涵蓋；畫面上的提示列待實機確認）

---

## 2B OpenAI 相容 Provider、測試連線

### 範圍

**做**
- `OpenAICompatibleChatProvider`：Chat Completions 串流、usage、錯誤對應
- 兩種格式的「列出模型」與「測試連線」（`IConnectionTester`）
- fixture 測試（同 1C：手寫、切成小塊餵、不呼叫真實 API）

**不做**
- 工具呼叫（M3）、圖片輸入（M4）
- OpenAI 的 Responses API

### 設計

- `POST {BaseUrl}/chat/completions`，`stream: true`、`stream_options: {include_usage: true}`。Base URL 由使用者填到 `/v1` 為止（OpenAI：`https://api.openai.com/v1`；Ollama：`http://localhost:11434/v1`）。
- 驗證：有金鑰時送 `Authorization: Bearer`；沒有金鑰（Ollama）時不送。公司 Gateway 也是這個格式（Q4）。
- 事件對應：`choices[0].delta.content` → `TextDelta`；`delta.reasoning_content`（Ollama、部分相容伺服器的推理內容）→ `ThinkingDelta`，但**不存成 `ThinkingBlock`**（沒有 signature，不能送回）；最後一個 chunk 的 `usage` → `Usage`；`finish_reason` → `StopReason`（`stop`、`length`、`content_filter` 對應 EndTurn、MaxTokens、Refused）；`data: [DONE]` → 結束。
- 寬容：伺服器沒回 `usage` 時，`Completed` 的 usage 是 null，狀態列顯示「—」；不認得的欄位略過。
- 錯誤：HTTP 狀態碼對應同 1C 的 `ChatFailureKind`；連不上（例如 Ollama 沒開）→ `Network`，訊息提示檢查 Base URL 或服務是否啟動。
- 送出的歷史：只送 `TextBlock`；其他區塊略過（見 2A）。
- 列出模型：Anthropic 用 SDK 的 Models API；OpenAI 相容用 `GET {BaseUrl}/models`。取不到時設定頁允許手動輸入模型 ID（ui-spec 第 7 節）。
- 測試連線 = 列出模型：成功顯示「可用模型 N 個」，失敗顯示原因（同 `ChatFailureKind` 的文字）。測試時用表單上**還沒儲存的**金鑰，不先寫進 `ISecretStore`。
- HTTP log 一樣不記錄 `Authorization` 標頭的值。

### 工作項目

- [ ] **B1 串流 Provider**：請求轉換、SSE 解析、事件對應、錯誤對應。
- [ ] **B2 列出模型／測試連線**：`IConnectionTester`，兩種格式各一個實作。
- [ ] **B3 fixture 與測試**：純文字、中文與 emoji 切塊、`reasoning_content`、沒有 usage、`length`、`content_filter`、HTTP 錯誤、連不上、取消、無金鑰時不送 `Authorization`。
- [ ] **B4 架構與分派**：`RoutingChatProvider` 接上；架構測試不變（OpenAI 相容不引用 SDK）。
- [ ] **B5 手動驗收**：Ollama（本機）、公司 Gateway 各對話一次。

### 驗收標準

- [ ] fixture 測試在 1／3／7 byte 切塊下結果一致。
- [ ] 同一則對話中途從 Anthropic 換到 OpenAI 相容再換回來，都能正常回答。
- [ ] 測試連線能分辨：成功、金鑰錯誤、Base URL 錯、服務沒開。

---

## 2C 設定視窗（大綱，輪到時展開）

- 獨立視窗，可以和工作台並排；從工作台的「設定」與模型選單的「管理連線與模型…」開啟，已開啟時只把它帶到前面
- 左側分類 M2 只有「連線與模型」「一般」
- 連線分頁：清單（名稱、狀態點、格式、模型數）、表單、測試連線、刪除確認（一併刪除模型設定）
- 金鑰欄位：已設定時顯示「已設定 · 結尾 xxxx」；輸入新金鑰才會取代；輸入時可以切換顯示正在輸入的內容，**已存的金鑰永遠不讀回來顯示**
- 模型設定分頁：工作台／小精靈預設下拉、依連線分組的清單、表單（模型從連線列出，取不到可手動輸入）
- 一般：主題（跟隨系統／淺色／深色），立即生效
- M1 的缺金鑰對話框：改成「開啟設定並定位到該連線」，對話框整個刪掉

## 2D 工作台完整介面（大綱，輪到時展開）

- 頂端：左「歷史對話」「新對話」Lumi 標誌；中「模型切換膠囊」（依連線分組、搜尋、最下面「管理連線與模型…」）；右「小精靈模式」（M7 前停用）、「設定」
- 回應中：膠囊與小精靈按鈕停用，旁邊顯示「回應中，暫時無法切換」（ui-spec 2.2）
- 歷史對話：最簡列表（依更新時間），點了切換；切換前先停止進行中的回應。搜尋、刪除、改名、日期分組是 M9
- 思考過程：預設收合，標題「思考過程 · N 秒」（秒數只在當下串流時量得到；重開後只顯示「思考過程」）
- 每則助理訊息：複製、重新產生、`↑ 輸入 · ↓ 輸出`。重新產生需要從尾端移除訊息，`IConversationStore` 加 `RemoveLastMessageAsync`（只能移除最後一則，仍符合「只能往後加」）
- 錯誤列：重試、換模型重試
- 狀態列：右側連線狀態點（最近一次請求成功／失敗）

## 2E Markdown 渲染（大綱，輪到時展開）

- 解析與轉換的做法（套件或自寫）輪到時再決定
- 硬性要求：所有文字經過 `EmojiText`；程式碼區塊用 `LumiMonoFont`、`CodeBackground`／`CodeText` token，有語言標籤與「複製」；串流中邊收邊渲染，不能每個字重建整棵畫面

---

## 實作紀錄

### 2A（2026-10-18）

- **`AtomicFile` 放在 `Lumi.Core/IO/`**：1B 的金鑰檔（Platform）和設定檔（Infrastructure）共用，兩者唯一的共同依賴是 Core。
- **`IUiDispatcher` 加回來**（1D 拿掉了）：`IModelCatalog.Changed` 可能從背景執行緒觸發，ViewModel 需要把更新丟回 UI 執行緒。命令本身仍在 UI 執行緒上。
- **`AnthropicDevOptions` 拆掉**：連線資訊改由 `IModelCatalog` 提供；只剩 Provider 本身的行為（effort、server-side fallback）放在 `AnthropicProviderOptions`，預設值就是正式行為，`appsettings.json` 的 `Lumi:Providers:Anthropic` 可選擇性覆寫。`ChatDefaults` 刪除。`appsettings.json` 只剩 Logging。
- **搬家建立的連線沒有 `ApiKeyHint`**：要算末 4 碼得讀鑰匙圈，Mac 開發期可能跳授權視窗，所以不讀；2C 的設定頁用 `ExistsAsync`（只查屬性、不讀內容）顯示「已設定」。
- **`ShowThinking` 關掉時完全不送 `thinking` 參數**：不只是不顯示，這樣不支援 adaptive thinking 的舊 Anthropic 模型也能用。
- **別家供應商的 `ProviderBlock`**：1C 是丟例外（要求呼叫端先篩選）；有了分派之後改成 Anthropic 轉換時直接略過，略過後沒內容的訊息整則不送。
- **`RoutingChatProvider`**：OpenAI 相容在 2B 之前回 `Failed`（訊息「還不支援」），不發 HTTP。
- **新增 `ChatFailureKind.ModelNotConfigured`**：模型全部刪光時，送出會保留使用者訊息並顯示錯誤列。
- **設定檔損毀／版本太新**：工作台頂端顯示一次性的提示列（「知道了」關掉）。版本太新時不寫檔（所有儲存都會失敗），避免舊版程式蓋掉新版的設定。
- **組裝測試**（`Lumi.App.Tests/CompositionRootTests`）：用和 `Program` 相同的方式組 DI（暫存路徑、測試用 Keychain service），確認每個服務都解析得到、第一次啟動會搬出 `dev-anthropic` 連線。
- **畫面上的確認沒做到**：開發時螢幕休眠，Avalonia 啟動時無法建立繪圖計時器（`RenderTimer`，錯誤碼 -6661），App 起不來；這是 macOS 的顯示狀態，不是程式問題。實機確認留給你。
- **補記 1D**：Claude Code 執行指令的環境裡，`grep` 是轉給 `ugrep` 的 shell 函式（不是你終端機的設定）。1D 驗收時「資料庫搜尋 `sk-ant` 第一次回報 1 筆、之後重現不出來」，很可能是 `ugrep` 對二進位檔的計數方式不同；改用 Python 逐位元組比對才是可靠的結果（0 筆）。
