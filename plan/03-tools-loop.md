# 03 Tool 框架與對話迴圈

- 狀態：草稿，待確認「開工前要確認的事」後從 3A 開工
- 對應里程碑：M3
- 前置：M2 程式完成；手動驗收與 Windows 驗證可以和 M3 並行，不擋開工

## 目標

模型可以呼叫我們自己寫的內建工具：Anthropic 和 OpenAI 相容兩種格式都能完成多輪工具呼叫；需要權限的工具會先在工具卡片內詢問使用者；工具呼叫以卡片顯示在訊息流裡。

## M3 拆成三個部分

| 部分 | 名稱 | 產出 | 依賴 |
|---|---|---|---|
| **3A** | 工具框架與對話迴圈 | Core 的 `ITool`／`ToolBase<TArgs>`／`IToolRegistry`／`IToolApprovalService`、新的內容區塊與事件、`Lumi.Tools`（示範工具）、ConversationService 的工具迴圈、存檔格式 | M2 |
| **3B** | Provider 的工具格式 | Anthropic 與 OpenAI 相容的「送出工具定義」「解析工具呼叫（串流）」「送回工具結果」；fixture 測試 | 3A |
| 3C | 工作台的工具 UI | 工具卡片（收合／展開／狀態）、群組卡、嵌在卡片內的權限確認、模型設定的「使用工具」開關 | 3A、3B |

3A 完成時用假的 Provider 就能跑完整個迴圈（有測試）；3B 讓真的模型能呼叫；3C 讓使用者看得到、答得了。每段結束都要能編譯、測試全過。

主計畫 M3 項目的對應：

| 主計畫項目 | 部分 |
|---|---|
| `ITool`、`ToolBase<TArgs>`、`IToolRegistry`、`AddLumiTool<T>()` | 3A |
| Schema 自動產生；權限等級；可用模式；逾時 | 3A |
| 對話迴圈：呼叫工具、回填結果、再次送出，並有最大次數限制 | 3A |
| `IToolApprovalService` 與確認 UI | 3A（服務與流程）、3C（畫面） |
| 工具呼叫卡片、群組卡 | 3C |
| 兩家 Provider 的 tool 格式轉換 | 3B |
| 先做 2～3 個示範工具 | 3A |

---

## 開工前要確認的事

以下都是影響資料格式或介面的決定。我已經各自給了建議；**沒有意見的話就照建議做**，有想改的告訴我。

| # | 問題 | 建議 | 影響 |
|---|---|---|---|
| Q1 | 工具結果在對話裡用什麼角色存？ | 新增 `ChatRole.Tool`（工具結果訊息）。Anthropic 轉換時把連續的工具結果合併成一則 user 訊息的多個 `tool_result` 區塊（API 要求同一輪的結果放在同一則）；OpenAI 相容轉成一則則 `role: tool`。UI 不把 `Tool` 訊息畫成使用者泡泡。資料庫 `role` 欄位多一個值 `tool`，不需要改 schema 版本。 | 3A、3B |
| Q2 | 示範工具要做哪些？（需要涵蓋「唯讀」與「會詢問」兩條路徑才能驗收） | 正式工具：`get_current_time`（ReadOnly）、`calculate`（ReadOnly，自己寫的安全運算式解析器，不用 `eval`）。**示範用**：`save_note`（Write，寫一則文字到資料夾下的 `notes/`）、`delete_note`（Destructive，只能刪 `notes/` 裡的檔案）——這兩個只為了驗收權限流程，用 `Lumi:Tools:EnableDemoTools` 開關控制（M3 預設開，M4 有真正的檔案工具後就移除）。真正的檔案、網路工具是 M4、M10。 | 3A |
| Q3 | 不支援工具的模型怎麼辦？（小型的 Ollama 模型常常不支援，送了工具定義會直接回 400） | `ModelSettings` 加 `ToolsEnabled`（預設開）；設定頁模型表單多一個「使用工具」核取方塊。關掉時送出的請求完全不帶工具定義。不做自動偵測（偵測不準、也很難測）。設定檔格式向前相容（舊檔缺這個欄位就是預設開），不升版本。 | 3A、3C |
| Q4 | 一次回應裡有多個工具呼叫，要平行還是依序執行？ | **依序**執行，每個都各自走權限確認。平行雖然快，但需要確認的工具會同時跳出多個確認區，使用者很難判斷；之後如果只有唯讀工具而且很慢，再優化。 | 3A |
| Q5 | 工具呼叫中間的「只有思考、沒有文字」的步驟要不要顯示？ | M3 不顯示（仍照樣存檔、送回模型，簽章不會壞）；畫面上只看到工具卡片與最後的文字。要不要在群組卡裡顯示每一步的思考，留給之後（需要的話再加）。 | 3C |

---

## 設計總覽（跨三個部分共用）

### 內容區塊與角色（Core）

```csharp
public enum ChatRole { User, Assistant, Tool }      // 新增 Tool

// 助理要呼叫工具：Id 由供應商產生，必須原樣送回；InputJson 是模型給的引數（原始 JSON 字串）
public sealed record ToolUseBlock(string Id, string Name, string InputJson) : ContentBlock;

// 工具結果。Content 是送給模型的文字；其餘欄位只給畫面用、不送給模型，
// 這樣重開程式後卡片的標題與狀態都還在，不依賴工具還註冊著。
public sealed record ToolResultBlock(
    string ToolUseId, string Content, bool IsError,
    string DisplayName, string? Target, string? Summary,
    long DurationMs, ToolApproval Approval) : ContentBlock;

public enum ToolApproval { NotRequired, AllowedOnce, AllowedForConversation, Denied }
```

- `ContentBlock` 的 JSON 多兩個型別（`tool_use`、`tool_result`），舊資料不受影響。
- 順序規則（兩家 API 都要求）：助理訊息裡的 `ToolUseBlock` 後面，**緊接著**一則 `Tool` 訊息，內含每個 `ToolUseBlock.Id` 的結果。組請求時（`BuildRequestMessages`）檢查：缺結果的呼叫（例如程式當掉）補一個「對話被中斷，沒有結果」的錯誤結果，避免 400。
- `StopReason` 加 `ToolUse`。

### 工具介面（Core，`Lumi.Core/Tools/`）

```csharp
public interface ITool
{
    ToolDescriptor Descriptor { get; }
    JsonNode InputSchema { get; }                                   // 由 TArgs 自動產生
    ToolPreview Preview(string argumentsJson);                      // 對象與影響說明（顯示用）
    Task<ToolResult> ExecuteAsync(string argumentsJson, ToolContext context, CancellationToken ct);
}

public abstract class ToolBase<TArgs> : ITool where TArgs : class
{
    public abstract ToolDescriptor Descriptor { get; }              // 沒有預設值：權限等級是必填
    public JsonNode InputSchema => /* JsonSchemaExporter，快取 */;
    protected abstract Task<ToolResult> ExecuteAsync(TArgs args, ToolContext context, CancellationToken ct);
    protected virtual ToolPreview Preview(TArgs args) => ToolPreview.None;
}

public sealed record ToolDescriptor(
    string Name,                       // ^[A-Za-z0-9_-]{1,64}$（兩家 API 的共同限制）
    string DisplayName,                // 繁中動作名稱，例如「讀取檔案」
    string Description,                // 給模型看，要寫清楚何時該呼叫
    ToolPermission Permission,         // ReadOnly／Write／Network／Destructive，必填
    ToolModes Modes,                   // Workbench／Sprite／Both，必填
    TimeSpan Timeout);                 // 每個工具都要有逾時

public sealed record ToolResult(bool IsError, string Content, string? Summary)
{
    public static ToolResult Ok(string content, string? summary = null) => ...;
    public static ToolResult Error(string message) => ...;          // 錯誤回給模型，不丟例外
}

public sealed record ToolPreview(string? Target, string? Detail);   // 「對象」與確認區顯示的影響說明（路徑、增減行數…）
```

- **Schema 自動產生**（`agent.md` 第 8 節）：`System.Text.Json` 的 `JsonSchemaExporter`；引數型別是強型別 record，用 `[Description]` 寫每個欄位的說明，由 `TransformSchemaNode` 轉成 schema 的 `description`；頂層一律 `type: object`、`additionalProperties: false`。測試固定幾個型別產生出來的 schema。
- 引數解析失敗（缺欄位、型別不對）→ `ToolResult.Error("參數不正確：…")` 回給模型，讓它自己修正；不丟例外。
- 工具執行拋例外 → 記 log（含例外），回給模型一個**不含內部細節**的錯誤（「工具執行失敗」）；逾時 → 錯誤「執行逾時（N 秒）」；使用者按停止 → 見下方「停止」。
- 結果太長（預設超過 20,000 字元）→ 截斷並附註「[內容過長，已截斷]」再送給模型，卡片仍可看到被截斷後的完整內容。
- `ToolContext`：`ConversationId`、`ChatMode`、`TimeProvider`。M4 才加 `Scope`（檔案範圍）。
- **註冊**：`services.AddLumiTool<MyTool>()`；`IToolRegistry`（`GetTools(mode)`、`Find(name)`）啟動時檢查名稱格式與重複，有問題就在啟動時丟例外（`ValidateOnStart` 的精神：寧可啟動失敗，不要執行到一半才發現）。
- 架構測試：`Lumi.Core` 只有介面與基底類別；內建工具都在 `Lumi.Tools`；每個 `ITool` 實作都繼承 `ToolBase<>`。

### 權限與確認（Core）

```csharp
public enum ToolPermission { ReadOnly, Write, Network, Destructive }

public interface IToolApprovalService
{
    // 對話迴圈在這裡暫停，直到使用者選擇（或被取消）
    Task<ApprovalDecision> RequestAsync(ToolApprovalRequest request, CancellationToken ct);
}
public enum ApprovalDecision { AllowOnce, AllowForConversation, Deny }
public sealed record ToolApprovalRequest(string CallId, string ToolName, string DisplayName, ToolPermission Permission, ToolPreview Preview, string ArgumentsJson);
```

規則（`docs/ui-spec.md` 第 5 節、`agent.md` 第 8 節）：
- `ReadOnly`：不詢問。`Write`、`Network`：詢問，可選「本次對話都允許（此等級）」。`Destructive`：**每次詢問，沒有記住選項**。
- 「本次對話都允許」只存在記憶體，每則對話各自一份（`ToolPermissionMemory`，以對話 Id 為鍵）；重開程式就消失，不寫進設定檔或資料庫。
- 拒絕 → 工具不執行，結果是 `ToolResult.Error("使用者拒絕執行")`，`Approval = Denied`，由模型決定改做法或說明。
- 每個結果都記下 `ToolApproval`，卡片顯示「已允許」或「你已拒絕」。
- 實作 `IToolApprovalService` 的是 App（3C）：它把請求變成卡片內的確認區，等使用者按按鈕。小精靈模式的確認方式還沒決定（ui-spec 第 5 節），M7 處理；M3 只有工作台。

### 事件（Core）

`ChatEvent` 補上 Q10 當時留到 M3 的三種：

```csharp
ToolCallStarted(string CallId, string ToolName, string DisplayName, ToolPermission Permission, string? Target, string ArgumentsJson)
ApprovalRequired(ToolApprovalRequest Request)       // 在等待使用者之前發出，畫面據此顯示確認區
ToolCallCompleted(string CallId, ToolResultBlock Result)
```

事件順序約定（更新 `IChatProvider` 的註解，並加到 `ConversationService`）：
- 一次回應（一輪）可以包含**多次**供應商呼叫；每次呼叫結束都會有一個 `Completed`（該次的助理訊息）。整輪以 `Failed` 或迴圈自然結束（最後一個 `Completed` 之後沒有更多事件）為終點。取消仍丟 `OperationCanceledException`。
- 工具呼叫的事件夾在兩個 `Completed` 之間：`ToolCallStarted` →（需要確認時）`ApprovalRequired` → `ToolCallCompleted`，每個呼叫都配成一組。

### 對話迴圈（`ConversationService`）

```
重複最多 MaxToolIterations 次（預設 10）：
  1. 組請求（含 registry.GetTools(mode)，模型的 ToolsEnabled 關閉時不帶）
  2. 呼叫 Provider，轉發事件；收到 Completed → 存助理訊息
  3. 沒有 ToolUseBlock → 結束
  4. 依序處理每個 ToolUseBlock：
       找工具 → 解析引數 → 需要確認就發 ApprovalRequired 並等待 → 執行（含逾時）→ 發 ToolCallCompleted
  5. 把結果存成一則 Tool 訊息，回到 1
超過次數 → Failed(ToolLoopLimit)，已完成的步驟都留著；使用者可以按「重試」接著跑
```

- `RetryAsync` 現在接受「最後一則是 User 或 Tool」：工具迴圈中途失敗（網路斷了、達到次數上限）時，歷史的最後一則是 Tool，重試就是從那裡繼續。
- **停止**（取消）：
  - 供應商串流中：和現在一樣，已收到的文字存成「已停止」（沒有完成的工具呼叫不存）。
  - 等待確認中：取消確認，這個工具記為「已取消」，結束這一輪。
  - 工具執行中：取消傳給工具；**已經存了的助理訊息裡每個沒結果的 `tool_use` 都補一個「使用者已停止」的錯誤結果**，存檔不受取消影響（用 `CancellationToken.None`），否則下次送出會因為缺結果被 API 拒絕。
- **重新產生**：移除最後一輪的所有助理與工具訊息（從尾端一路移除到最後一則使用者訊息），再要一次。
- 未知的工具名稱（模型亂叫）→ 錯誤結果「沒有這個工具」，不中斷迴圈。
- 模式：`ChatMode`（`Workbench`／`Sprite`）由呼叫端傳入，只有宣告了該模式的工具會送給模型；M3 只有工作台。

### 存檔格式

- 訊息 `role` 欄位多一個值 `tool`；`content_json` 多兩個區塊型別。舊資料與舊版程式的相容：新版寫的 `tool` 訊息，舊版程式讀到會因為不認得的 `role`／區塊型別而讀取失敗——這是「資料庫比程式新」的情況，M1 的版本檢查只擋 schema 版本；所以這次**把 schema 版本升到 2**（migration 只記版本、沒有結構變更），讓舊版程式明確拒絕開啟，而不是讀到一半壞掉。
- 歷史清單、標題等不受影響。

### 示範工具（`Lumi.Tools`）

| 工具 | 權限 | 說明 |
|---|---|---|
| `get_current_time` | ReadOnly | 回傳目前時間（可指定 IANA 時區，預設本機）；摘要「2026-10-19 14:32（Asia/Taipei）」 |
| `calculate` | ReadOnly | 四則運算、括號、`^`、常用函式（`sqrt`、`round`…）；自己寫的遞迴下降解析器，不執行任何程式碼；除以零、語法錯誤回錯誤結果 |
| `save_note`（示範） | Write | 寫一則文字到使用者資料夾的 `notes/`；檔名只允許簡單名稱（不接受路徑）；預覽顯示檔名與字數 |
| `delete_note`（示範） | Destructive | 刪除 `notes/` 裡的一個檔案；預覽顯示完整檔名 |

- `Lumi.Tools` 引用 `Lumi.Core`；`AddLumiTools()` 註冊（`Lumi:Tools:EnableDemoTools` 控制兩個示範工具）。
- `Lumi.App` 引用 `Lumi.Tools`（組裝點）。

---

## 3A 工具框架與對話迴圈

### 範圍

**做**：上面「設計總覽」的所有 Core 內容、`Lumi.Tools` 專案與四個工具、`ConversationService` 的迴圈、`BuildRequestMessages` 的工具歷史修補、`ToolsEnabled` 欄位（資料模型與設定檔；畫面在 3C）、存檔格式、schema 升版。

**不做**：Provider 的工具格式（3B）、任何畫面（3C）、檔案／網路工具（M4、M10）。

### 工作項目

- [ ] **A1 資料模型**：`ChatRole.Tool`、`ToolUseBlock`、`ToolResultBlock`、`ToolApproval`、`StopReason.ToolUse`、三個新事件、`ChatFailureKind.ToolLoopLimit`；JSON 多型測試（舊資料仍可讀）。
- [ ] **A2 工具介面與基底**：`ITool`、`ToolBase<TArgs>`、`ToolDescriptor`、`ToolResult`、`ToolPreview`、`ToolContext`、`ToolPermission`、`ToolModes`；Schema 產生（含 `[Description]`）、引數解析、結果截斷；測試。
- [ ] **A3 註冊與查找**：`IToolRegistry`、`AddLumiTool<T>()`、啟動時驗證；測試。
- [ ] **A4 權限**：`IToolApprovalService`、`ApprovalDecision`、`ToolPermissionMemory`、各等級的規則；測試。
- [ ] **A5 對話迴圈**：`ConversationService` 的工具迴圈、逾時、取消時補結果、次數上限、`RetryAsync` 接受 `Tool` 結尾、`RegenerateAsync` 移除整輪；用假 Provider 與假工具測。
- [ ] **A6 歷史修補**：`BuildRequestMessages` 保持 `tool_use`／`tool_result` 成對、缺結果補錯誤結果、不會把工具訊息和使用者訊息錯誤合併；測試。
- [ ] **A7 `Lumi.Tools`**：專案、四個工具、運算式解析器（要有大量測試，包含惡意輸入）、`AddLumiTools()`；`Lumi.App` 組裝。
- [ ] **A8 存檔**：SQLite 的 `tool` 角色、schema 升到 2、舊資料庫自動升級的測試、新資料庫被舊版拒絕的測試。
- [ ] **A9 `ModelSettings.ToolsEnabled`**：資料模型、設定檔向前相容、迴圈依它決定是否帶工具；測試。
- [ ] **A10 架構測試與文件**：工具只在 `Lumi.Tools`／都繼承 `ToolBase<>`；`agent.md` 第 7、8 節補充（事件順序、停止時補結果、權限記憶只在記憶體）；本檔實作紀錄。

### 驗收標準

- [ ] `dotnet build` 零警告、`dotnet test` 全過。
- [ ] 用假 Provider（腳本化的回應）跑完：單次呼叫、連續多次呼叫、平行多個呼叫、未知工具、引數錯誤、工具丟例外、工具逾時、使用者拒絕、停止（等待確認中／執行中）、達到次數上限，每一種之後的歷史都能再送出（沒有缺結果的 `tool_use`）。
- [ ] `Destructive` 每次都詢問；`Write` 選了「本次對話都允許」後同一則對話不再詢問、另一則對話仍詢問。
- [ ] `calculate` 對 `1+2*3`、`(1+2)*3`、`2^10`、`sqrt(16)` 正確，對 `1/0`、`1+`、超長輸入、巢狀括號 10,000 層都回錯誤結果而不是當掉。
- [ ] 舊的資料庫（schema 1）打開後自動升到 2，既有對話完好。

---

## 3B Provider 的工具格式

### 範圍

**做**：兩家 Provider 的三件事——送出工具定義、解析（串流中的）工具呼叫、把工具歷史轉成各家格式。fixture 測試（手寫，同 1C、2B）。

**不做**：伺服器端工具（網頁搜尋、程式碼執行等 Anthropic 的 server tools）；`tool_choice` 強制呼叫（Claude Opus 5.5 以後會回 400，`agent.md` 第 7 節沒有用到）。

### 設計

**Anthropic（官方 SDK、beta 端點）**
- 工具定義：名稱、描述、`input_schema`（直接用 `ITool.InputSchema`）。
- 串流：`content_block_start` 的 `tool_use`（Id、Name）＋一連串 `input_json_delta` 的 `partial_json`；到 `content_block_stop` 才把累積的字串當 `InputJson`（不在串流中途解析，避免半截 JSON）。`stop_reason: tool_use` → `StopReason.ToolUse`。
- 歷史：助理訊息的區塊順序原樣送回（thinking → text → tool_use；有啟用 thinking 時，工具呼叫的那則助理訊息必須帶著原本的 thinking 區塊與簽章，1C 已確保原樣保存）；連續的 `Tool` 訊息合併成**一則** user 訊息的多個 `tool_result`（`is_error` 對應 `IsError`）。
- 不要求引數串流（`eager_input_streaming` 不開）：我們本來就要等完整才執行。

**OpenAI 相容（自己寫）**
- 工具定義：`tools: [{type: "function", function: {name, description, parameters}}]`。
- 串流：`delta.tool_calls[]` 依 `index` 累積 `id`、`function.name`、`function.arguments`（引數是分段的字串）；`finish_reason: tool_calls` → `StopReason.ToolUse`。寬容：有些伺服器把整個呼叫放在單一 chunk、`index` 缺漏，或 `arguments` 是空字串（代表沒有引數）。
- 歷史：助理訊息輸出 `tool_calls`（`content` 沒有文字時為 null）；每個 `ToolResultBlock` 一則 `{role: "tool", tool_call_id, content}`；錯誤結果也是一則 tool 訊息（內文就是錯誤文字，OpenAI 沒有 `is_error`）。
- 之前 2B 的「歷史只送文字」改為：文字、`tool_use`、`tool_result` 都送；thinking 與別家的區塊仍略過。
- 不支援工具的伺服器回 400：`ChatFailure` 的訊息提示「這個模型可能不支援工具，可以在模型設定關掉『使用工具』」（只在請求帶了工具、而且回 400 時加上這句）。

### 工作項目

- [ ] **P1 Anthropic 請求**：工具定義轉換、歷史轉換（含合併 tool_result、保留 thinking 順序）；測試。
- [ ] **P2 Anthropic 串流**：`tool_use` 區塊累積、`input_json_delta`、`stop_reason: tool_use`、多個工具呼叫、thinking 與工具並存；fixture 以 1／3／7 byte 切塊。
- [ ] **P3 OpenAI 請求**：工具定義、歷史轉換；測試。
- [ ] **P4 OpenAI 串流**：`tool_calls` 累積（分段引數、缺 index、單 chunk、空引數、多個呼叫、文字與工具並存）、`finish_reason: tool_calls`；fixture 切塊測試。
- [ ] **P5 錯誤提示**：帶工具時的 400 提示。
- [ ] **P6 跨格式**：同一則對話含工具呼叫的歷史，在 Anthropic ↔ OpenAI 相容之間切換後仍能送出（工具區塊兩家都認得，thinking 只送回 Anthropic）；測試。
- [ ] **P7 手動驗收**（需要你的金鑰，會有少量費用）：兩家各請模型呼叫 `get_current_time`、`calculate`，多輪對話正常。

### 驗收標準

- [ ] 所有 fixture 在 1／3／7 byte 切塊下結果一致，引數 JSON 完整（含中文）。
- [ ] 用腳本化的假 HTTP 回應，Anthropic 與 OpenAI 相容各自完成「模型呼叫工具 → 回填結果 → 模型回答」的兩次請求，第二次請求的內容符合各家格式。
- [ ] 真實模型（Anthropic 與你的 Gateway）能完成多輪工具呼叫。

---

## 3C 工作台的工具 UI（大綱，3A、3B 完成後展開）

- **工具卡片**（ui-spec 第 4 節）：預設收合；標題固定格式「狀態圖示 動作 · 對象 · 結果摘要 … 耗時 ›」；動作名稱來自工具的 `DisplayName`、對象來自 `Preview`、結果摘要來自 `ToolResult.Summary`（UI 不猜）。狀態：執行中（轉圈）、成功、失敗（紅）、等待允許（黃）、已拒絕（灰）；執行中與等待允許的卡片強制展開。展開後顯示引數與結果，結果超過 6 行只顯示前段、提供「顯示完整輸出」與「複製」。
- **群組卡**：同一輪連續 3 個以上的工具呼叫（跨步驟、中間沒有文字）合併成一張，標題如「3 個工具呼叫 · 讀取 2 個檔案、搜尋 1 次，全部成功」；任一失敗時標題改失敗色並寫出失敗項目。
- **嵌在卡片內的權限確認**（ui-spec 第 5 節）：各等級的按鈕；`Destructive` 標「無法復原」、只有「允許執行」「拒絕」；按停止取消等待。確認的實作 `ToolApprovalService`（App）把請求交給 ViewModel、等按鈕。
- **訊息列表**：`Tool` 訊息不畫成泡泡；重開程式後從 `ToolUseBlock` ＋ `ToolResultBlock` 重建卡片（標題所需資料都存在結果區塊裡）；只有思考沒有文字的中間步驟不顯示（Q5）。
- **模型設定頁**加「使用工具」核取方塊（`ToolsEnabled`）。
- **回應中**：既有限制不變（膠囊停用等）；等待確認時「回應中」狀態維持，停止鈕可用。
- 測試：ViewModel（卡片狀態轉換、群組規則、確認流程、重開後重建）、Headless（卡片收合／展開、確認按鈕）、手動驗收。

---

## 實作紀錄

（偏離原計畫的地方與原因，實作時填寫）
