# 02 模型管理、設定頁與工作台完整介面

- 狀態：2A、2B、2C 程式完成，待手動驗收（C9，含 B5）；2D、2E 程式完成，待手動驗收（D8、E7）；
- 對應里程碑：M2
- 前置：M1 在 Mac 完成；Windows 驗證可以和 M2 並行，不擋開工

## 目標

不同供應商的多個模型可以在工作台切換使用；連線與模型在獨立的設定視窗管理；工作台補齊 ui-spec 第 2、3 節的互動。金鑰不出現在任何檔案或 log。

## M2 拆成五個部分

| 部分 | 名稱 | 產出 | 依賴 |
|---|---|---|---|
| **2A** | 模型目錄與設定儲存 | `IModelCatalog`、設定檔、從 M1 的開發設定搬家、依連線分派 Provider | M1 |
| **2B** | OpenAI 相容 Provider、測試連線 | OpenAI 相容串流、兩種格式的「列出模型」與「測試連線」 | 2A |
| 2C | 設定視窗 | 連線、模型設定兩個分頁；一般（主題）；缺金鑰改開設定視窗 | 2A、2B |
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

- [x] **B1 串流 Provider**：請求轉換、SSE 解析、事件對應、錯誤對應。
- [x] **B2 列出模型／測試連線**：`IConnectionTester`，兩種格式各一個實作。
- [x] **B3 fixture 與測試**：純文字、中文與 emoji 切塊、`reasoning_content`、沒有 usage、`length`、`content_filter`、HTTP 錯誤、連不上、取消、無金鑰時不送 `Authorization`。
- [x] **B4 架構與分派**：`RoutingChatProvider` 接上；架構測試不變（OpenAI 相容不引用 SDK）。
- [ ] **B5 手動驗收**：Ollama（本機）、公司 Gateway 各對話一次。（待你執行；2C 完成前要手動編輯 `settings.json` 加連線，見下方實作紀錄）

### 驗收標準

- [x] fixture 測試在 1／3／7 byte 切塊下結果一致。
- [x] 同一則對話中途從 Anthropic 換到 OpenAI 相容再換回來，都能正常回答。（測試涵蓋：歷史來回後，thinking 區塊與 signature 仍原樣送回 Anthropic，OpenAI 只收到文字）
- [x] 測試連線能分辨：成功、金鑰錯誤（401）、Base URL 錯（404）、服務沒開（連線失敗）。（測試涵蓋；實機待驗）

---

## 2C 設定視窗

### 範圍

**做**
- 獨立的設定視窗（可與工作台並排），左側分類 M2 只有「連線與模型」「一般」
- 連線分頁：清單、表單、金鑰、測試連線、刪除
- 模型設定分頁：工作台／小精靈預設模型、依連線分組的清單、表單、從連線列出可用模型
- 一般：主題（跟隨系統／淺色／深色），立即生效
- 缺金鑰時改為開啟設定視窗並定位到該連線，M1 的 `ApiKeyDialog` 整個刪除
- 工作台頂端先加一顆「設定」按鈕（2D 會重排整條工具列，這裡只放進入點）

**不做**
- 語音、人格與主動性、工具權限、小精靈分類（各自的里程碑再加，不放空頁面）
- 模型切換膠囊、歷史對話（2D）

### 設計

#### 視窗與導覽

- `SettingsWindow`：單一執行個體，已開啟時只把它帶到前面並切到要去的位置（`ISettingsWindowService.Show(section, connectionId?)`）。非強制回應視窗（不鎖住工作台），所以可以一邊改設定一邊看工作台。
- 左側分類清單：連線與模型、一般。「連線與模型」內用兩個分頁：**連線**、**模型設定**（ui-spec 第 7 節）。
- 視窗大小 880×620，可縮放；版面用 1A 的 token，文字放資源檔。
- 關閉視窗時如果表單有未儲存的變更，詢問「放棄變更？」。

#### 連線分頁

- 左側清單：每筆顯示名稱、格式、底下有幾個模型設定、狀態點（最近一次「測試連線」的結果，沒測過是灰色，只存在記憶體）。下方「新增連線」。
- 表單：名稱、格式（Anthropic 原生／OpenAI 相容）、Base URL、API 金鑰。
  - 新增時依格式帶入範本 Base URL（Anthropic：`https://api.anthropic.com`；OpenAI 相容：`https://api.openai.com/v1`，Ollama 改成 `http://localhost:11434/v1`），使用者自己改。
  - **金鑰欄位**：已設定時顯示「已設定 · 結尾 xxxx」＋「更換」「清除」按鈕；按「更換」或還沒有金鑰時才出現輸入框（密碼欄位，旁邊一顆「顯示」只切換正在輸入的內容）。**已存的金鑰永遠不讀回來顯示**。M1 搬家建立的連線沒有末 4 碼提示，這時只顯示「已設定」（`IModelCatalog.HasApiKeyAsync`，只查有沒有，不讀內容，Mac 不會跳授權視窗）。OpenAI 相容的金鑰可以留空（Ollama）。
  - 驗證：名稱不可為空；Base URL 必須是 http 或 https 的完整網址；**用 `http://` 連到非本機位址時顯示警告**（金鑰會以明文在網路上傳送），不擋。
- 「測試連線」：用表單上目前的設定與**還沒儲存的金鑰**（沒輸入就用已儲存的）呼叫 `IConnectionTester`；顯示「成功 · 可用模型 N 個」或 2B 定義好的失敗原因。測試期間按鈕變成轉圈、可取消。成功時把模型清單留在記憶體，給模型設定分頁用。
- 「儲存」：`IModelCatalog.SaveConnectionAsync`（有輸入新金鑰才寫進 `ISecretStore`）；「清除金鑰」：`ClearApiKeyAsync`，需要確認。
- 「刪除」：確認視窗寫明「底下 N 個模型設定也會一併刪除，金鑰也會一併刪除」。刪到沒有任何連線是允許的（工作台顯示「未設定模型」）。
- 切換清單選取時，表單有未儲存的變更 → 詢問「放棄變更？」。

#### 模型設定分頁

- 上方兩個下拉：**工作台預設模型**、**小精靈預設模型**（M7 才用，先存）。改了立即生效（`SetWorkbenchModelAsync`／`SetSpriteModelAsync`）。
- 左側清單依連線分組，預設的項目標上「工作台預設」「小精靈預設」。下方「新增模型設定」（需要至少一個連線，沒有時按鈕停用並提示先加連線）。
- 表單：使用的連線（下拉）、模型 ID、顯示名稱、最大輸出 tokens、上下文上限、顯示思考過程。
  - **模型 ID 是可編輯的下拉**：選了連線之後，背景用 `IConnectionTester` 以已儲存的金鑰列出可用模型填進下拉；取不到（404、服務沒開、沒有金鑰）時下拉是空的，顯示一行說明，使用者可以直接手動輸入。旁邊「重新整理」。
  - 顯示名稱留空時，選了／輸入模型 ID 後自動帶入模型 ID；使用者改過就不再覆蓋。
  - 新增時的預設值依連線格式：Anthropic 最大輸出 64000／上下文 1,000,000／顯示思考過程開；OpenAI 相容 4096／128,000／關。使用者自己依模型改（M2 不內建模型規格表）。
  - 驗證：模型 ID、顯示名稱不可為空；兩個數字必須大於 0。
- 刪除：確認後刪除；如果它是預設模型，預設自動改成剩下的第一個（`JsonModelCatalog` 已處理），畫面上提示「預設模型已改為 xxx」。

#### 一般分頁

- 主題：三選一，選了立即呼叫 `SetThemeAsync`。
- `ThemeService`（App）訂閱 `IModelCatalog.Changed`，把 `ThemePreference` 對應到 Avalonia 的 `RequestedThemeVariant`（System → `Default`、Light、Dark）；啟動時也套用一次，避免先閃一下系統色。對應函式是純函式，可單元測試。

#### 缺金鑰流程

- `IApiKeyPrompt` 介面保留（`WorkbenchViewModel` 不改），實作換成 `SettingsApiKeyPrompt`：開啟設定視窗、定位到工作台目前模型的連線、直接聚焦金鑰輸入框，**視窗關閉時**回報「該連線現在有金鑰了嗎」（有 → 工作台自動重送，沒有 → 留下錯誤列）。
- 這種開法用非強制回應視窗，所以使用者可以順便按「測試連線」確認金鑰。
- 刪除 `ApiKeyDialog`、`ApiKeyDialogViewModel` 與相關字串。

#### 同時操作的邊界

- 回應進行中也可以改設定：進行中的回應已經解析好 `ResolvedModel`，不受影響；之後的請求才用新設定。
- 設定檔版本太新（唯讀）時，所有儲存動作失敗，設定視窗頂端顯示 `LoadProblem`，表單的「儲存」「刪除」停用。

#### ViewModel 與測試

- `SettingsViewModel`（分類切換）、`ConnectionsViewModel`＋`ConnectionEditorViewModel`、`ModelsViewModel`＋`ModelEditorViewModel`、`GeneralViewModel`。命令與狀態都在 ViewModel，code-behind 只管視窗行為（聚焦、關閉詢問）。
- 對話框抽象：`IConfirmService`（`ConfirmAsync(訊息, 確認文字)`），ViewModel 測試用假的。
- 測試直接用真的 `JsonModelCatalog`（暫存資料夾）＋記憶體 `ISecretStore`＋假的 `IConnectionTester`，不用另外維護一份假目錄：
  - 新增／編輯／刪除連線、金鑰只在有輸入時寫入、清除金鑰、末 4 碼顯示、刪除時模型與金鑰一併消失
  - 驗證（名稱、網址、http 警告）、未儲存變更的詢問
  - 測試連線：成功、失敗原因、使用未儲存的金鑰、可取消、結果留給模型分頁
  - 模型分頁：新增／編輯／刪除、預設模型標示、預設值依格式、顯示名稱自動帶入、清單取不到時可手動輸入
  - 一般：主題選擇會寫入並觸發 `ThemeService` 的對應
  - 缺金鑰流程：關閉時有金鑰 → 回 true；沒有 → 回 false
  - 金鑰不出現在任何 ViewModel 屬性、log、設定檔（沿用 2A 的測試與架構測試）

### 工作項目

- [x] **C1 目錄補強**：`IModelCatalog.HasApiKeyAsync`（只查有沒有）＋ `JsonModelCatalog` 實作與測試；2A 的假目錄補上。
- [x] **C2 主題**：`ThemeService`（對應函式、訂閱變更、啟動套用）＋測試。
- [x] **C3 共用元件**：`IConfirmService` 與視窗實作、`ISettingsWindowService`、設定視窗骨架（分類導覽、`LoadProblem` 提示、關閉詢問）。
- [x] **C4 連線分頁**：清單、表單、金鑰欄位、驗證、測試連線、儲存／清除／刪除、未儲存詢問；ViewModel 測試。
- [x] **C5 模型設定分頁**：預設模型下拉、分組清單、表單、可用模型列出、手動輸入、預設值與自動帶入；ViewModel 測試。
- [x] **C6 一般分頁**：主題選擇。
- [x] **C7 缺金鑰流程**：`SettingsApiKeyPrompt`、刪除 `ApiKeyDialog*`；工作台頂端加「設定」按鈕。
- [x] **C8 字串資源**：所有新文字進 `Strings.resx`／`Strings.cs`（`StringsTests` 會檢查同步）；色碼檢查測試守住不寫死顏色。
- [ ] **C9 手動驗收**（需要你操作，同時完成 B5）：見驗收標準。（**待你執行**）
- [x] **C10 文件**：`agent.md`（如有新規則）、本檔實作紀錄、主計畫狀態。

### 驗收標準

- [x] `dotnet build` 零警告、`dotnet test` 全過。
- [ ] 設定視窗可以和工作台並排使用；重複按「設定」只會帶到前面，不會開第二個。
- [ ] 新增一個 OpenAI 相容連線（公司 Gateway 或 Ollama）→ 測試連線成功並列出模型 → 新增模型設定 → 設為工作台預設 → 回工作台送出訊息收到回答（B5）。
- [ ] 故意把 Base URL 改錯、金鑰改錯、關掉服務，測試連線各自顯示對應的原因。
- [ ] 已存的金鑰任何地方都不會顯示；`settings.json` 與資料庫裡找不到金鑰（用 Python 逐位元組比對，不用 `grep`）。
- [ ] 刪除連線時提示會一併刪除模型設定；刪除後鑰匙圈裡該連線的金鑰也消失。
- [ ] 主題切換立即生效；重開程式後維持所選。
- [ ] 清掉金鑰後送出訊息：開啟設定視窗並定位到該連線，存好金鑰關閉視窗後自動重送。
- [ ] Windows 11 上同樣可以完成以上操作（和 M1 的 Windows 驗證一起做）。

---

## 2D 工作台完整介面

### 範圍

**做**
- 頂端工具列：歷史對話、新對話、Lumi 標誌、模型切換膠囊、小精靈模式（停用）、設定
- 模型切換膠囊：依連線分組的選單、搜尋、「管理連線與模型…」
- 歷史對話：最簡的左側抽屜，列出對話、點了切換
- 回應中的限制：膠囊與小精靈按鈕停用並提示；切換對話或開新對話前先停止回應
- 助理訊息：思考過程（標題帶秒數）、複製、重新產生、本輪 token
- 錯誤列：重試、換模型重試
- 狀態列右側：連線狀態點與文字

**不做**
- 對話列表的搜尋、刪除、改名、依日期分組（M9）
- 小精靈模式本身（M7；按鈕先停用）
- Markdown 渲染（2E）
- 上下文列、加入檔案、語音（M4、M6）

### 設計

#### 頂端工具列（ui-spec 2.1）

`[歷史對話] [新對話] ●Lumi ─── [● 模型名稱 · 連線名稱 ▾] ─── [小精靈模式] [設定]`

- 歷史對話：切換左側抽屜（`SplitView` 內嵌，開啟時把訊息區往右推，不蓋住它）。
- 新對話：目前這則對話還沒有任何訊息時什麼都不做（已經是空白的了）；否則建立新對話。**回應中也可以按**：先停止回應（見下方），再開新對話。
- 模型切換膠囊：顯示連線狀態點、模型顯示名稱、連線名稱、下拉箭頭；沒有模型時顯示「未設定模型」，點了直接開設定視窗的模型分頁。
- 小精靈模式：M7 前一律停用（灰色，提示「之後的版本提供」）。

#### 模型切換選單

- 彈出式選單（`Flyout`），內容：搜尋框、依連線分組的模型清單（連線名稱當標題列、不可選取；目前使用的模型打勾）、最下方「管理連線與模型…」（開設定視窗的模型分頁）。
- 搜尋同時比對顯示名稱、模型 ID、連線名稱；沒有符合的連線標題不顯示。
- 選了模型 ＝ `SetWorkbenchModelAsync`（M2 的 Q3：切換就是改工作台預設模型，下次開啟也是它）。寫入失敗（例如設定檔唯讀）時在選單內顯示原因。
- 回應中整個膠囊停用，旁邊顯示「回應中，暫時無法切換」（ui-spec 2.2）。
- 同一個選單內容（`ModelPicker` 控制項）也給錯誤列的「換模型重試」使用，所以做成可重複使用的元件。

#### 連線狀態

- `ConnectionStatusCache` 加 `MarkResult(連線 Id, 成功與否)`：保留已有的可用模型清單，只改狀態。
- 工作台每次回應結束時更新目前連線的狀態：收到 `Completed` → 已連線；`Network`、`ServerError`、`Overloaded`、`StreamError`、`InvalidApiKey`、`PermissionDenied`、`ModelNotFound` → 連線失敗；其他（`RateLimited`、`Refused`、`BadRequest`、缺金鑰…）不改——這些代表連線本身是通的，或根本沒送出去。
- 膠囊上的狀態點與狀態列右側（「已連線」「連線失敗」「尚未確認」）用同一份狀態。設定頁「測試連線」的結果也會更新它。

#### 歷史對話抽屜

- 清單依最近更新時間排列，每列：標題、時間（今天顯示 `HH:mm`，其他顯示 `M/d`）；目前對話標示。沒有任何訊息的對話不列出。
- 點選列 → 切換對話。回應中點選：先停止目前的回應（已收到的文字存成「已停止」），再切換（ui-spec 2.2：抽屜仍可瀏覽，切換前先停止）。
- `IConversationStore.ListAsync(limit)`：只回有訊息的對話，依 `updated_at` 由新到舊；M2 先取最近 100 則，M9 做搜尋與分頁時再改。
- 清單在這些時機更新：啟動、新對話、一則回應結束或停止（更新時間變了）、設定視窗不影響它。

#### 回應中的切換與停止

- `WorkbenchViewModel.StopAndWaitAsync()`：取消目前回應，並**等它的收尾全部做完**（存「已停止」的訊息、移除空的助理泡泡）才回來；切換對話與開新對話都先呼叫它。
- 收尾完成的判斷用一個 `TaskCompletionSource`，放在回應流程最外層的 `finally`，保證即使流程中途出錯也會完成。
- **修正一個 M1 就有的潛在問題**：回應失敗後跳出金鑰視窗期間，使用者可能切到別的對話；視窗關閉後的自動重送不能打到別的對話上。重送前檢查「現在的對話還是當初失敗的那一則」，不是就不重送。

#### 助理訊息

- **思考過程**：預設收合，標題「思考過程 · N 秒」。秒數只在當下串流時量得到（第一個 `ThinkingDelta` 到第一個 `TextDelta` 或完成）；重開程式後只顯示「思考過程」（資料庫沒存時間，不為這個改 schema）。計時用 `TimeProvider`，測試可控制。
- **訊息底下的動作列**（串流完成後才出現，不是每則都顯示全部）：
  - 複製：複製這則的文字（不含思考過程）到剪貼簿，按鈕短暫顯示「已複製」。剪貼簿透過 `IClipboardService`，ViewModel 不碰 Avalonia。
  - 重新產生：只在**最後一則助理訊息**出現，回應中停用。
  - `↑ 輸入 N · ↓ 輸出 N tokens`：用這則訊息存的 usage；輸入含快取；沒有 usage（OpenAI 相容伺服器沒回）就不顯示。
- **重新產生的語意**：移除最後那則助理訊息（含「已停止」的），用同一段歷史再要一次回應，新回應取代舊的（不保留兩個版本）。
  - 這需要「從尾端移除」：`IConversationStore.RemoveLastMessageAsync`，只能移除最後一則。**`agent.md` 的「只能往後加」規則放寬為「只能往後加，或從尾端移除」**：從尾端移除不會讓前面任何一則的內容改變，不影響 thinking 簽章的一致性（1C 已確認）。
  - 流程：`ConversationService.RegenerateAsync(conversationId)`：最後一則是助理訊息才有效，否則丟 `InvalidOperationException`。

#### 錯誤列（ui-spec 第 3 節）

- 現有：原因 + 「對話內容已保留」+「重試」。
- 新增「換模型重試」：按下彈出同一份模型選單，選了模型 → 切換工作台預設模型 → 立刻重試。回應中不會出現（錯誤列只在失敗後、非回應中顯示）。

#### 狀態列

- 左：`輸入 N · 輸出 N · 回合 N · 上下文 [進度條] 14K / 1M`（M1 已有）。
- 右（新增）：連線狀態點 + 文字。

#### ViewModel 結構

- `WorkbenchViewModel`（既有）：加 `History`、`ModelSwitcher`、`RetrySwitcher` 三個子 ViewModel（由它自己建立，依賴都已經在它手上）；加 `NewConversationCommand` 的新行為、`OpenConversationAsync`、`StopAndWaitAsync`、`RegenerateCommand`、`CopyCommand`、連線狀態屬性。建構式多收 `ConnectionStatusCache`、`IClipboardService`、`TimeProvider`。
- `HistoryViewModel`、`ModelSwitcherViewModel`（含搜尋與分組）、`MessageViewModel`（加 token 文字、思考秒數、是否為最後一則、已複製）。

#### 測試

- Infrastructure：`ListAsync`（排序、略過空對話、上限）、`RemoveLastMessageAsync`（只移除最後一則、之後再加訊息的 `seq` 接得上、對話更新時間不倒退）。
- Core：`RegenerateAsync`（移除最後的助理訊息並重送、最後是使用者訊息時丟例外、失敗時舊訊息已被移除的處理、請求內容不含被移除的訊息）。
- App（真的 `JsonModelCatalog` ＋ 假的 Provider／store）：
  - 模型選單：分組與搜尋、選了會寫入工作台預設、回應中停用、寫入失敗的提示
  - 切換對話：回應中切換會先存「已停止」再載入新對話、舊回應的收尾不會污染新對話的訊息列表
  - 新對話：空白對話不重複建立；回應中可按
  - 金鑰視窗期間切換對話不會重送到別的對話
  - 動作列：複製（假剪貼簿）、重新產生只在最後一則、token 文字格式、無 usage 不顯示
  - 思考秒數（假 `TimeProvider`）：串流中量得到、重開後沒有
  - 連線狀態：各種 `ChatFailureKind` 對應成功／失敗／不變
  - 換模型重試：選了模型後自動重送
- 色碼檢查與字串同步測試（既有）守住新畫面。

### 工作項目

- [x] **D1 儲存**：`IConversationStore.ListAsync`、`RemoveLastMessageAsync`（介面、SQLite、兩份假 store）＋測試；`agent.md` 第 7 節放寬規則。
- [x] **D2 重新產生**：`ConversationService.RegenerateAsync` ＋測試。
- [x] **D3 連線狀態**：`ConnectionStatusCache.MarkResult`、失敗分類函式 ＋測試。
- [x] **D4 訊息 ViewModel**：token 文字、思考秒數（`TimeProvider`）、最後一則旗標、已複製、`IClipboardService` ＋測試。
- [x] **D5 工作台 ViewModel**：`StopAndWaitAsync`、切換對話、新對話的新行為、金鑰視窗後的重送檢查、`HistoryViewModel`、`ModelSwitcherViewModel`、換模型重試、連線狀態屬性 ＋測試。
- [x] **D6 畫面**：頂端工具列、膠囊與 `ModelPicker`、歷史抽屜、動作列、思考過程標題、錯誤列新按鈕、狀態列右側；`ClipboardService` 實作。
- [x] **D7 字串資源**。
- [ ] **D8 手動驗收**（需要你操作）：見驗收標準。（**待你執行**）
- [x] **D9 文件**：`agent.md`、`docs/ui-spec.md`（如有出入）、本檔實作紀錄、主計畫狀態。

### 驗收標準

- [x] `dotnet build` 零警告、`dotnet test` 全過。
- [ ] 模型切換膠囊：切到另一個模型後下一則回應確實用它；重開程式維持；跨連線（Anthropic ↔ OpenAI 相容）都能切。
- [ ] 回應中：膠囊與小精靈按鈕停用並顯示提示；按停止後恢復。
- [ ] 歷史對話：能列出多則對話並切換；回應中切換會先停止（已收到的文字留成「已停止」）；舊回應不會跑到新對話裡。
- [ ] 新對話：空白對話下重複按不會產生一堆空對話。
- [ ] 重新產生：只出現在最後一則助理訊息；按了之後舊回答被取代；重開程式後只剩新的。
- [ ] 複製：貼到別處是這則的純文字（不含思考過程）。
- [ ] 思考過程標題帶秒數；token 數字與狀態列一致。
- [ ] 故意讓請求失敗：錯誤列的「重試」與「換模型重試」都可用；連線狀態點會變成失敗。
- [ ] Windows 11 上同樣可以完成以上操作（和 M1 的 Windows 驗證一起做）。

---

## 2E Markdown 渲染

### 範圍

**做**
- 助理訊息的文字用 Markdown 渲染：標題、段落、粗體／斜體／刪除線／行內程式碼、連結、有序與無序清單（可巢狀）、引用、分隔線、表格、程式碼區塊
- 程式碼區塊：語言標籤、「複製」按鈕、等寬字型、深底（兩套色系都維持深底）
- 串流中邊收邊渲染，不能每個字都重建整棵畫面
- 所有文字經過 `EmojiText`（`agent.md` 第 12 節），顏色只用主題 token

**不做**
- 語法上色（程式碼區塊先是單色；要做是另一件事）
- 圖片（顯示成「[圖片：說明]」連結文字）、HTML（原樣顯示成文字，不渲染）
- 數學公式、Mermaid 等擴充
- 使用者自己的訊息（維持純文字）、思考過程（維持純文字）
- 滑鼠選取部分文字（Avalonia 的 `SelectableTextBlock` 與我們的 emoji 分段、連結點擊判斷互相影響，風險大於效益；整則複製已有按鈕。列為後續項目）

### 設計

#### 做法：Markdig 解析 ＋ 自己的渲染器

| | 用現成的 Avalonia Markdown 套件 | **Markdig 解析 ＋ 自己渲染（採用）** |
|---|---|---|
| emoji 分段（`EmojiText`） | 套件自己建 `TextBlock`，插不進去 | 渲染器自己建文字，每段都走 `EmojiText` |
| 主題 token、字型資源 | 要靠覆寫套件的樣式，容易漏 | 直接用 `LumiXxxBrush`、`LumiMonoFont` |
| 串流增量更新 | 通常整份重建 | 自己控制：只重建有變的區塊 |
| 程式量 | 少 | 多（約一個渲染器 ＋ 一個控制項） |

- 套件：`Markdig`（BSD-2、純 C#、沒有其他相依），只在 `Lumi.App` 引用（UI 專用；`Lumi.Core` 不需要知道 Markdown）。解析選項：pipe tables、刪除線、自動連結；**停用 HTML**（`DisableHtml`）：模型輸出的 HTML 一律當文字顯示，不渲染。
- 兩層分開，方便測試：
  1. `MarkdownParser`：文字 → 自己定義的區塊模型（`MdBlock`／`MdInline` 的不可變記錄），不依賴 Avalonia，直接單元測試。
  2. `MarkdownRenderer`：區塊模型 → Avalonia 控制項；`MarkdownView` 控制項包住它，處理串流更新。

#### 區塊模型

- 區塊：`Paragraph`、`Heading(level)`、`CodeBlock(language, code)`、`Quote`、`ListBlock(ordered, start, items)`、`Table(header, rows, alignments)`、`Rule`。
- 行內：文字、粗體、斜體、刪除線、行內程式碼、連結（網址＋子內容）、換行。
- 每個最上層區塊記下它在原文裡的文字（`Source`），串流時用來比對「這個區塊有沒有變」。
- 軟換行（段落中的單一換行）**顯示成換行**，不是 CommonMark 預設的空白：聊天訊息常靠單一換行排版，M1 的純文字也是這樣顯示的。

#### 串流增量渲染

- `MarkdownView.Markdown`（字串）與 `IsStreaming` 兩個屬性；繫結到訊息的 `Text`。
- 文字一變就排一個 `Background` 優先權的更新（同一個畫面週期內連續的多個 `TextDelta` 自然合併成一次，不需要計時器）；`IsStreaming` 變 false（回應完成）時立刻渲染最後一次。
- 每次更新：整份重新解析（Markdig 很快，幾十 KB 是毫秒級），然後和上一次的最上層區塊**逐個比對 `Source`**：相同的沿用原本的控制項，從第一個不同的區塊開始往後重建。串流時通常只有最後一個區塊在變，所以重建的量是常數。
- 串流到一半的語法：沒關閉的程式碼圍欄＝程式碼區塊一路到結尾（正好符合串流時的直覺）；沒關閉的 `**` 暫時顯示原樣，關閉後變粗體。

#### 渲染細節

- **段落與標題**：`EmojiTextBlock`；行內樣式用 `Run`（粗體、斜體、刪除線設屬性；行內程式碼與連結用樣式類別 `mdcode`、`mdlink`，顏色與字型在樣式裡引用 token）。`EmojiText` 重構出 `AddText(inlines, text, style)`：切字形叢集、emoji 叢集換字型，段落、粗體內的文字都走它。
- **連結**：只允許 `http`、`https`、`mailto`；其他（`javascript:`、`file:`…）顯示成純文字不可點。Avalonia 沒有內建可點的超連結 Inline，所以：建立段落時記下每個連結的文字範圍，在段落上攔截滑鼠點擊，用 `TextLayout` 的命中測試換算成文字位置，落在連結範圍內就用系統預設瀏覽器開啟；滑鼠移到連結上改成手形游標。點擊判斷用 Headless 測試驗證。
- **清單**：每項一列（符號欄＋內容欄）；無序符號依層級輪替 `•`、`◦`；有序用起始編號；項目裡可以有多個區塊（段落、巢狀清單、程式碼）。
- **引用**：左邊一條 `BorderStrong` 色線，文字用 `TextMuted`。
- **分隔線**：1px 的 `Border` 色線。
- **程式碼區塊**：`CodeBackground`／`CodeText`；上方一列語言標籤（沒有語言時不顯示標籤）與「複製」鈕（點了顯示「已複製」1.5 秒）；內容用 `LumiMonoFont`、不換行、水平捲動；仍經過 `EmojiText`。
- **表格**：水平可捲動的 `Grid`，欄寬自動（單欄最寬 360，超過換行），標題列粗體，格線用 `Border` token，欄位對齊依 Markdown 的 `:---:` 設定。
- **間距**：區塊之間 10；標題上方多留一點；清單項目之間 4。
- 圖片：`[圖片：說明]`，點了當連結處理（網址受上面的協定限制）。

#### 整合

- 訊息列表裡助理訊息的 `ctl:EmojiTextBlock Text="{Binding Text}"` 換成 `MarkdownView`；使用者訊息、思考過程維持 `EmojiTextBlock`。
- 「複製」整則訊息仍然複製原始 Markdown 文字（保留格式，貼到別處能再渲染）。

#### 測試

- 解析（純邏輯）：各種區塊與行內、巢狀清單、表格對齊、空程式碼區塊、沒關閉的圍欄與強調、HTML 當文字、軟換行、圖片、連結協定白名單、`Source` 切片正確。
- Headless（`Avalonia.Headless.XUnit`，不開真的視窗）：
  - 渲染出預期的控制項結構（標題字級、清單層級、程式碼區塊有語言標籤與複製鈕、表格欄數）
  - **增量更新**：串流追加文字時，前面的區塊控制項是同一個物件（沒有被重建）、只有最後一個變
  - 複製按鈕把程式碼放進剪貼簿
  - 連結：點在連結範圍內會開啟（用假的啟動器）、點在外面不會、非白名單協定不可點、連結前面有換行與 emoji 時範圍仍然正確
  - emoji：粗體內與程式碼區塊內的 ❤️‍🔥 不被拆開（走 `EmojiText`）
  - 色碼檢查（既有的架構測試）守住新的樣式檔

### 工作項目

- [x] **E1 套件與解析**：加 `Markdig`；`MdBlock`／`MdInline` 模型；`MarkdownParser`；測試。
- [x] **E2 EmojiText 重構**：抽出 `AddText(inlines, text, style)`，`Apply` 改用它；既有測試不變、補新測試。
- [x] **E3 渲染器**：`MarkdownRenderer`（區塊、行內、清單、引用、分隔線、表格、程式碼區塊）＋ `Themes/Markdown.axaml` 樣式（只用 token）。
- [x] **E4 `MarkdownView` 控制項**：串流增量更新（排程合併、區塊比對）、連結點擊與游標、程式碼複製。
- [x] **E5 整合**：訊息範本換成 `MarkdownView`；字串資源。
- [x] **E6 Headless 測試專案設定**與渲染／增量／連結／emoji 測試。
- [ ] **E7 手動驗收**（需要你操作）：見驗收標準。（**待你執行**）
- [x] **E8 文件**：README、本檔實作紀錄、主計畫（M2 狀態）。

### 驗收標準

- [x] `dotnet build` 零警告、`dotnet test` 全過。
- [ ] 請模型輸出一份含標題、清單、表格、程式碼（含語言）、引用、粗斜體、連結的長回答，畫面排版正確，淺色與深色都清楚可讀。
- [ ] 串流過程順暢：長回答邊收邊渲染，沒有明顯卡頓或閃爍；程式碼圍欄還沒關閉時就已經顯示成程式碼區塊。
- [ ] 程式碼區塊的「複製」貼出來只有程式碼本身。
- [ ] 點連結用預設瀏覽器開啟；`javascript:` 之類的連結不可點。
- [ ] 回答裡的 emoji（含 ❤️‍🔥 這類組合）在粗體、清單、表格、程式碼區塊裡都完整。
- [x] 重開程式後，載入的舊對話一樣用 Markdown 渲染。（Mac 實機已確認：舊對話裡的程式碼區塊、標題、清單都正確渲染）
- [ ] Windows 11 上同樣可以完成以上操作（和 M1 的 Windows 驗證一起做）。

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

### 2B（2026-10-18）

- **`IConnectionTester` 放 Core**（計畫只說「`IConnectionTester`」沒指定位置）：設定頁（App）只認得 Core 的介面；實作在 Providers，依連線格式分派（`RoutingConnectionTester`）。結果型別 `ConnectionTestResult` 帶模型 ID 清單或 `ChatFailure`，失敗原因沿用 `ChatFailureKind`，所以「金鑰錯、網址錯、服務沒開」分得出來。
- **OpenAI 相容的最大輸出用 `max_tokens`**：這是各家相容伺服器最普遍支援的欄位；OpenAI 自己新的推理模型要的是 `max_completion_tokens`，目前不處理，遇到再加。
- **串流結束的判斷**：收到 `[DONE]` 或 `finish_reason` 才算正常結束；連線關了但兩者都沒有 → `StreamError`（避免把被截斷的回應當成完整回答存檔）。有 `[DONE]` 但沒有 `finish_reason` 仍視為完成（有些相容伺服器這樣）。
- **`reasoning_content` 只顯示、不存**：沒有 signature，不能送回；所以換回 Anthropic 時歷史裡不會有這段。
- **usage**：OpenAI 的 `prompt_tokens` 含快取；轉成我們的 `TokenUsage` 時 `InputTokens` 扣掉 `cached_tokens`、快取另記，和 Anthropic 的算法一致。伺服器沒回 usage 時訊息的 `Usage` 是 null。
- **錯誤內文不放進 `ChatFailure`**：伺服器回的錯誤訊息可能含敏感資訊，只用狀態碼分類，log 只記狀態碼。
- **逾時**：串流用的 `HttpClient` 逾時設成無限（本機模型載入、長回答可能很久，停止靠 `CancellationToken`）；測試連線 30 秒。
- **Anthropic 的 `Models.List`** 用 SDK；測試連線要求一定要有金鑰（沒有就直接回「尚未輸入」，不送請求）。OpenAI 相容沒有金鑰仍會送請求（Ollama）。
- **B5 手動驗收還沒做**：這台電腦沒有 Ollama，公司 Gateway 要你實測。2C（設定視窗）還沒有，要先驗可以手動編輯 `~/Library/Application Support/Lumi/settings.json`：在 `connections` 加一筆 `{"id":"company","name":"公司 Gateway","format":"openAICompatible","baseUrl":"https://…/v1"}`、在 `models` 加一筆 `{"id":"m-company","connectionId":"company","modelId":"你的模型","displayName":"…","maxOutputTokens":4096,"contextWindow":128000,"showThinking":false}`，並把 `workbenchModelId` 改成 `m-company`；金鑰則要先有 2C 的設定頁才能存（目前缺金鑰對話框會把金鑰存到工作台目前模型的連線，所以這樣設定後送出會跳對話框，輸入金鑰即可）。

### 2C（2026-10-19）

- **模型清單的「分組」**：計畫寫「依連線分組」，Avalonia 的 `ListBox` 沒有分組標頭，所以做成依連線名稱排序、每一列顯示「連線名稱 · 模型 ID」，沒有真的分組標頭。
- **畫面確認**：Mac 上實際開過設定視窗（連線、模型設定、缺金鑰流程）。模型分頁用你的金鑰真的列出 13 個可用模型，順便驗證了 Anthropic 的測試連線。沒有輔助使用權限所以無法自動點擊，是用暫時的環境變數在啟動時開視窗截圖，截完已移除（不在 commit 裡）。截圖時修掉兩個視覺問題：選取列（琥珀底）文字對比不夠、分頁標題字太大。
- **新增 `IModelCatalog.HasApiKeyAsync`**：只查有沒有金鑰、不讀內容；M1 搬家的連線沒有末 4 碼，設定頁就只顯示「已設定」。
- **缺金鑰流程多擋一種無限重送**：金鑰「無效」時，使用者開了設定視窗什麼都沒改就關掉，不能算「存好了」，否則會重送→失敗→再開視窗，一直循環。所以 `SettingsApiKeyPrompt` 對 Invalid／Unreadable 要求視窗開著期間設定真的被儲存過（`Changed` 事件）；Missing 則只看現在有沒有金鑰。
- **`ThemeService`** 用委派套用主題，所以不需要真的有 Avalonia 應用程式就能測試；啟動時先套用一次。
- **連線狀態點**只存在記憶體（`ConnectionStatusCache`）：連線一改、一刪、清金鑰就清掉，因為舊的結果不準了。
- **設定視窗每次開啟都重新建立**（關掉就丟掉），狀態一律從目錄重讀，不用處理「視窗關著但 ViewModel 還活著」的過期問題。
- **刪除 M1 的 `ApiKeyDialog`**、`ApiKeyDialogViewModel`、`ApiKeyPromptService` 與相關字串。
- **還沒做**：Windows 全部；`ConfirmDialog`（確認視窗）、設定視窗的鍵盤操作與無障礙沒有另外測試。

### 2D（2026-10-19）

- **沒有做 `OpenSetup`**（模型選單的「沒有模型時點膠囊直接開設定」）：膠囊的彈出選單裡本來就有「管理連線與模型…」，多一個入口是重複的，刪掉了。沒有模型時膠囊顯示「未設定模型，點此開啟設定」，點開選單裡有說明與「管理連線與模型…」。
- **`ModelSwitcherViewModel` 加了 `Selected` 事件**：選了模型後由畫面據此關掉彈出選單（兩個選單：頂端膠囊、錯誤列的「換模型重試」，各一個執行個體）。
- **金鑰視窗期間的重送檢查**照計畫做了，而且把「回應收尾」和「事後處理（金鑰視窗、重送）」拆成兩個階段：`StopAndWaitAsync` 只等第一階段，所以金鑰視窗開著時切換對話不會被卡住。這個情境有測試（`KeyPromptStaysOpen_UserSwitchesConversation_NoRetryHitsTheOtherConversation`）。
- **重新產生失敗時畫面與資料庫對齊**：重新產生會先移除舊答案，如果之後失敗，畫面會從資料庫重讀（舊答案可能已經被移除了）；模型都沒設定時，在移除之前就擋下來，舊答案原封不動。
- **回合數改成重新計算**（`RefreshMessageState`：非串流中的助理訊息數），不再 `Turns++`：重新產生、停止、失敗、切換對話都走同一條路，不會算錯。
- **切換對話不清空輸入框的草稿**（草稿屬於視窗，不屬於對話）。
- **歷史清單的時間**用 `TimeProvider.LocalTimeZone`，所以測試可以固定時區；今天顯示 `HH:mm`，其他 `M/d`。
- **畫面確認**：Mac 上實際看過歷史抽屜、模型選單（含打勾、搜尋框、「管理連線與模型…」）、動作列（複製、重新產生只在最後一則、token）、狀態列右側的連線狀態。一樣是用暫時的環境變數在啟動時打開（沒有輔助使用權限無法點擊），已移除。
- **一個不穩的測試修掉了**：`ChangingDefaultDropdowns_AppliesImmediately` 偶爾失敗，原因是測試等「目錄的值」變了就繼續，但畫面的更新（在 `Changed` 事件裡、稍後才跑）和測試的執行緒同時動到清單。改成等畫面狀態本身。
- **還沒做**：複製目前沒有鍵盤快速鍵；歷史抽屜開關狀態不會記住（重開程式是收合的）；Windows 全部。

### 2E（2026-10-19）

- **新增 `Avalonia.Headless.XUnit`**（`Lumi.App.Tests`）：不開真的視窗就能建立控制項、排版、做命中測試、送滑鼠事件。載入和正式程式相同的 token／字型／Markdown 樣式。注意它用的是「無繪圖」的字型替身（每個字固定寬度），所以能驗證結構、偏移量與事件，**不能驗證真實字型的外觀**——外觀靠 Mac 實機截圖確認。
- **連結點擊，Headless 測試抓到兩個真問題**：
  1. **沒有背景的 `TextBlock` 不會被滑鼠命中測試**：點擊會被下面的 `Panel` 接走，連結永遠點不到（真實視窗一樣）。有連結的段落要設透明背景才收得到事件。
  2. **`TextLayout.HitTestPoint` 的 `IsInside` 在多行時不可靠**（第二行起回傳 false）。改成自己判斷：用 `TextPosition` 找到字元，再用 `HitTestTextPosition` 確認點落在這個字元的上下左右範圍內（行尾字元用該行的寬度當右邊界）。這同時避免點在一行文字右邊的空白處被誤判成點到行尾的連結。
- **文字位置的換算**（連結範圍）：確認 `LineBreak` 在 `TextLayout` 裡占 1 個字元、組合 emoji（❤️‍🔥）占它的 UTF-16 長度（5）；測試用「連結前有換行與 emoji」的段落驗證範圍和排版一致。
- **不渲染的東西維持計畫**：HTML 當文字、圖片顯示成「[圖片：說明]」連結文字、沒有語法上色、沒有滑鼠選取部分文字。
- **軟換行顯示成換行**（不是 CommonMark 預設的空白）：聊天訊息常靠單一換行排版。若之後覺得不對，改解析器一處即可。
- **串流增量渲染的實際效果**：測試確認追加文字時，前面的區塊控制項是同一個物件、只重建有變的區塊。沒有量化效能（沒有做長回答的 benchmark）；整份重新解析的成本在 Markdig 的量級是毫秒，之後若實測卡頓再優化。
- **字型資源抽成 `Themes/Typography.axaml`**（原本在 `App.axaml`）：App 與 Headless 測試共用同一份，不重複定義。
- **架構測試**：Markdig 只有 `Lumi.App` 引用。
- **畫面確認**：Mac 上實際看過舊對話裡的標題、帶粗體與行內程式碼的清單、程式碼區塊都正確渲染。**表格、引用、連結點擊、串流中的渲染沒有實機看過**（需要模型輸出含這些的回答，見 E7）。
- **還沒做**：Windows 全部；程式碼區塊「複製」按鈕的實際剪貼簿行為（測試用假的服務驗證）。

