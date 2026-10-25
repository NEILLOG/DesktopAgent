# 15 工具清單

- 狀態：**草稿**（2026-10-25 細節與使用者討論定案，待確認後動工）
- 對應里程碑：M15
- 前置：M3（工具框架與權限確認）、M10（工具移植）、M12（設定頁新版面）、M14（記憶工具與記憶開關）皆已程式完成。
- 後續：M16（MCP）會在這一頁加上「MCP 伺服器」分段與依伺服器分組的 MCP 工具；本計畫**不預留** M16 的畫面。
- 設計依據：`plan/ui-guide/ui-guide.html`（權威）、`docs/ui-spec.md` 第 7 節（設定視窗）。
- 撰寫方式：依主計畫「細部計畫的撰寫方式」——讀者是低階快速模型，多個子代理平行實作。

## 目標

使用者在設定視窗看得到 Lumi 所有內建工具，依群組分類；可以逐一或整組啟用、關閉；`Write`／`Network` 工具可以設成「永遠允許」（不再詢問）。

## 範圍

做：

- **Phase 0**（協調者）：設定欄位、`ToolDescriptor` 新欄位、`ToolDefinition`（請求改送定義快照）、`IToolCatalog`、migration 9、`ToolApproval.AllowedAlways`。
- **15A** Infrastructure：設定檔讀寫 `Tools` 區段、SQLite 存取工具快照。
- **15B** Core：`ToolCatalog`、`ConversationService` 凍結工具清單與「永遠允許」。
- **15C** Tools：每個內建工具填上群組與給使用者看的一句說明。
- **15D** App：`ToolsViewModel`、`ToolsPanel`、工具卡片的「永遠允許」文字。
- **整合**（協調者）：設定視窗新分類、DI、文件。
- **人工驗收**（使用者）。

不做：

- MCP（M16）。
- 在工具確認卡片上加「永遠允許」按鈕：只在設定頁設定，避免使用者在對話中順手按下。
- 改變工具的權限等級（內建工具的等級固定；M16 只讓 MCP 工具可調）。
- 依模式或依對話個別設定工具。
- 搜尋、篩選工具（23 個，不需要）。

## 已定案的決定（2026-10-25 與使用者討論）

| # | 決定 | 理由 |
|---|---|---|
| D1 | **啟用／關閉從新對話才生效**。第一次送出時把工具定義的快照存在對話上（`conversations.tool_snapshot`，migration 9），之後這則對話一律送快照。 | Anthropic 的 thinking 簽章綁定工具清單，中途改變會 400（主計畫 2026-10-25）。同 `system_prompt` 的做法。 |
| D2 | 快照存**完整定義**（名稱、說明、input schema），不只是名稱。 | 程式更新後內建工具的說明可能改變；M16 的 MCP 伺服器也可能改定義或離線。存完整定義，送出的內容就逐字不變。執行時仍用「目前的」工具。 |
| D3 | 舊對話（快照是空的）：已經有助理訊息 → 快照＝**全部**內建工具（和過去送的一樣）；還沒有助理訊息 → 快照＝目前**已啟用**的工具。 | 舊對話過去送的是全部工具，改成「已啟用的」會改變清單而 400。 |
| D4 | 快照裡有、但目前找不到的工具，執行時回錯誤「這個工具目前無法使用。」；快照裡沒有的工具名稱，照舊回「沒有這個工具：{name}」。設定裡已關閉、但在快照裡的工具，**照常可以執行**。 | D1：設定改變不影響進行中的對話。 |
| D5 | 頁面也處理權限：`Write`／`Network` 可以設「永遠允許」；**`Destructive` 與 `memory_save` 不行**。「永遠允許」**立即生效**（它不改變送出的工具清單）。 | 使用者選擇。`memory_save`：寫入記憶＝永久性提示注入（M14 Q2）。 |
| D6 | 設定視窗新增一個分類「工具」，排在「記憶」之後。M15 只有一頁（沒有分段）；M16 再加「工具清單｜MCP 伺服器」分段。 | 使用者選擇（一個分類＋分段）；M15 不放空分段。 |
| D7 | `show_details` 不能關閉（勾選框停用並寫出原因）。 | 小精靈靠它把長內容放到工作台。 |
| D8 | 記憶工具可以開關；記憶在「記憶」頁關閉時，群組下方寫說明。 | 兩個開關的作用不同：這裡決定送不送定義，記憶頁決定能不能執行。 |
| D9 | 群組：一般、讀取檔案、寫入檔案、筆記與對話、提醒與計時、長期記憶、小精靈（`read_document` 併入「讀取檔案」）。 | 只有一個工具的群組太零碎。 |
| D10 | 開關與確認方式都是**改了立即儲存**，沒有「儲存」鈕（同記憶頁、小精靈頁）。改成「永遠允許」前要先確認。 | 安全相關的設定要讓使用者看清楚後果。 |
| D11 | 模型設定的「啟用工具」照舊：關閉時整份清單都不送。工具頁在目前預設模型關閉工具時顯示 Warning 說明。 | 兩者層次不同：模型能不能用工具 vs. 用哪些工具。 |

## 設計

### 資料

```csharp
// Lumi.Core/Models/ToolSettings.cs（Phase 0）
/// 工具清單頁的設定（M15）。Disabled／AlwaysAllow 存工具名稱，依序數排序、去重。
public sealed record ToolSettings
{
    public static readonly ToolSettings Default = new();
    public IReadOnlyList<string> Disabled { get; init; } = [];
    public IReadOnlyList<string> AlwaysAllow { get; init; } = [];
    public ToolSettings Normalize() => ...;   // 去空白項目、去重、StringComparer.Ordinal 排序
}
// AppSettings 新增：public ToolSettings Tools { get; init; } = ToolSettings.Default;
// IModelCatalog 新增：Task SetToolsAsync(ToolSettings tools, CancellationToken ct = default);
```

settings.json 新增區段（缺少或格式錯誤時用預設值）：

```json
"tools": { "disabled": ["overwrite_file"], "alwaysAllow": ["add_note"] }
```

### `ToolDescriptor` 新增的欄位（Phase 0，加在參數最後面，都有預設值）

```csharp
string Group = ToolGroups.General,   // 群組代號，見 ToolGroups
string? Summary = null,              // 給使用者看的一句話（設定頁），不送給模型
bool CanDisable = true,              // false＝設定頁不能關（show_details）
bool CanAlwaysAllow = true           // false＝不能設「永遠允許」（memory_save）；Destructive 一律不行，不必另外設
```

```csharp
// Lumi.Core/Tools/ToolGroups.cs（Phase 0）
public sealed record ToolGroupInfo(string Id, string Name, int Order);
public static class ToolGroups
{
    public const string General = "general", FilesRead = "files-read", FilesWrite = "files-write",
        Notes = "notes", Reminders = "reminders", Memory = "memory", Sprite = "sprite";
    public static IReadOnlyList<ToolGroupInfo> BuiltIn { get; } = [
        new(General, "一般", 0), new(FilesRead, "讀取檔案", 1), new(FilesWrite, "寫入檔案", 2),
        new(Notes, "筆記與對話", 3), new(Reminders, "提醒與計時", 4), new(Memory, "長期記憶", 5), new(Sprite, "小精靈", 6)];
    public static ToolGroupInfo? Find(string id);
}
```

### 送出的工具改成「定義」（Phase 0）

```csharp
// Lumi.Core/Tools/ToolDefinition.cs
/// 送給供應商的工具定義（名稱、說明、input schema）。對話第一次送出時存成快照，之後原樣送出（M15 D1、D2）。
public sealed record ToolDefinition(string Name, string Description, JsonNode InputSchema)
{
    public static ToolDefinition From(ITool tool);   // InputSchema 用 DeepClone()
}
public static class ToolSnapshotJson
{
    public static string Serialize(IReadOnlyList<ToolDefinition> tools);       // [{"name","description","input_schema"}]
    public static IReadOnlyList<ToolDefinition>? Deserialize(string? json);    // null／空字串／格式錯誤 → null
}
```

- `ChatRequest.Tools` 的型別從 `IReadOnlyList<ITool>?` 改成 `IReadOnlyList<ToolDefinition>?`。
- `AnthropicRequestConverter.ToTool` 與 `OpenAICompatibleChatProvider`（第 240 行附近）改讀 `ToolDefinition` 的三個欄位。
- `IConversationStore` 新增（預設實作同 `GetSystemPromptAsync`）：
  - `Task<string?> GetToolSnapshotAsync(string conversationId, CancellationToken ct = default)`
  - `Task SetToolSnapshotAsync(string conversationId, string snapshotJson, CancellationToken ct = default)`
- migration 9：`ALTER TABLE conversations ADD COLUMN tool_snapshot TEXT;`

### `IToolCatalog`（Phase 0 介面；15B 實作）

```csharp
// Lumi.Core/Tools/IToolCatalog.cs
/// 設定頁與對話迴圈共用的工具目錄：內建工具（M16 起加上 MCP 工具）＋使用者的啟用設定。
public interface IToolCatalog
{
    /// 新對話第一次送出時要凍結的工具：已啟用的（CanDisable=false 的一律算啟用），依名稱排序。
    Task<IReadOnlyList<ITool>> GetEnabledForNewConversationAsync(CancellationToken ct);
    /// 全部工具（含已關閉），依名稱排序。舊對話補快照用（D3）。
    IReadOnlyList<ITool> GetAll();
    /// 依名稱找目前能執行的工具；找不到回 null。
    ITool? Find(string name);
    /// 設定頁用：依群組排好的清單（群組依 Order，組內依 DisplayName 的序數排序）。
    IReadOnlyList<ToolCatalogGroup> ListGroups();
    /// 這個工具能不能設「永遠允許」：Permission 是 Write 或 Network，且 CanAlwaysAllow。
    static bool CanAlwaysAllow(ToolDescriptor d) => d.Permission is ToolPermission.Write or ToolPermission.Network && d.CanAlwaysAllow;
}
public sealed record ToolCatalogGroup(ToolGroupInfo Group, IReadOnlyList<ITool> Tools);
```

### 對話迴圈（15B）

```
RunTurnAsync 開頭（只在 model.ToolsEnabled 時）：
  snapshotJson = store.GetToolSnapshotAsync(id)
  definitions = ToolSnapshotJson.Deserialize(snapshotJson)
  if definitions is null:
      history 裡已有助理訊息 ? catalog.GetAll() : await catalog.GetEnabledForNewConversationAsync()
      → definitions = 上面的工具.Select(ToolDefinition.From)；store.SetToolSnapshotAsync(...)
  request.Tools = definitions（Count 為 0 時送 null）
執行每個 tool_use：
  名稱不在 definitions 裡 → 「沒有這個工具：{name}」（現有行為）
  catalog.Find(name) 是 null → 「這個工具目前無法使用。」，summary「無法使用」
  其餘照現有流程（記憶關閉、模式不符、權限）
權限：
  Permission != ReadOnly 時，先看 settings.Tools.AlwaysAllow 含這個名稱且 IToolCatalog.CanAlwaysAllow(descriptor)
    → approval = ToolApproval.AllowedAlways，不詢問
  否則照現有流程（ToolPermissionMemory、詢問）
```

### 畫面：設定 › 工具（15D）

位置：設定視窗左側 Menu 新增分類「工具」（副標「能用的工具與確認方式」），排在「記憶」之後。分類索引變成：0 AI 連線、1 人格與主動性、2 記憶、**3 工具**、4 外觀與操作、5 小精靈、6 語音。圖示用 Lucide `wrench`（整合包加進 `Icons.axaml`）。

```
工具                                                     ← eyebrow
Lumi 能用的工具                                          ← 頁面標題 28/700
選擇 Lumi 在對話中可以使用哪些工具，以及使用前要不要先問你。  ← 說明
                                                         （沒有頁首主要動作）
啟用與關閉從下一則新對話開始生效；進行中的對話維持原本的工具。 ← Caption
確認方式改了立即生效。                                     ← Caption
[⚠ 目前的預設模型「Opus 5.5」關閉了「啟用工具」…]          ← 只在需要時，Warning 淡底

[☑] 一般                       已啟用 2／2              ← 群組列：三態勾選框＋14/700＋Caption
    [☑] 查詢目前時間  唯讀                   不需確認
        查詢現在的日期與時間。
        get_current_time
    [☑] 計算  唯讀                           不需確認
    ...
────────────────────────────────────────────────────── ← 分隔線（1px Border，上下 16）
[▣] 寫入檔案                   已啟用 4／5
    [☑] 寫入檔案  寫入 · 只限工作台         [每次詢問 ▾]
    [☐] 覆寫檔案  破壞性 · 只限工作台       每次都會詢問
...
──────────────────────────────────────────────────────
[全部恢復預設]（secondary）   目前已是預設（停用時的 Caption）
```

- 群組列：`CheckBox`（`IsThreeState="True"`）。全部啟用＝勾、全部關閉＝空、部分＝不確定。點選「不確定」或「空」→ 全部啟用；點選「勾」→ 全部關閉。`CanDisable=false` 的工具不受影響。群組裡所有工具都是 `CanDisable=false` 時，群組勾選框停用。
- 工具列：左邊 `CheckBox`（`AutomationProperties.Name`＝工具的顯示名稱）；中間三行：顯示名稱（14/600）＋權限徽章＋模式文字、說明（`Summary`，Body、Muted）、工具名稱（Caption、`LumiMonoFont`）；右邊是確認方式。
- 權限徽章（沿用 `Border.badge`）：唯讀「唯讀」、Write「寫入」、Network「網路」、Destructive「破壞性」（Destructive 用 Danger 淡底，樣式仿 `MemoryPanel.axaml` 的 `Border.warn-badge`，改成 Danger 的三個 brush）。模式：`Workbench`「只限工作台」、`Sprite`「只限小精靈」、`Both` 不顯示。
- 確認方式（右欄）：
  - ReadOnly → Caption「不需確認」。
  - 不能永遠允許（Destructive 或 `CanAlwaysAllow=false`）→ Caption「每次都會詢問」；`memory_save` 的 Tooltip「寫進記憶的內容每則訊息都會送出，所以一律先問你。」，Destructive 的 Tooltip「這個工具會覆蓋或刪除內容，所以一律先問你。」
  - 其他 → `ComboBox`（寬 140）兩個選項「每次詢問」「永遠允許」。改成「永遠允許」前用 `IConfirmService` 確認；取消就把選項改回「每次詢問」。
- `CanDisable=false`：勾選框停用、保持勾選，說明下方加一行 Caption「小精靈靠它把完整內容放到工作台，不能關閉。」
- 記憶群組：`AppSettings.Memory.Enabled` 是 false 時，群組列下方加 Caption「長期記憶已在「記憶」頁關閉，這些工具目前不會執行。」
- Warning（目前工作台預設模型的 `ToolsEnabled` 是 false）：Warning 淡底＋三角圖示＋文字，樣式仿 `MemoryPanel.axaml` 的 `Border.warn-badge`，但做成整行。
- 底部：「全部恢復預設」（secondary）。按下先確認；設定已經是預設時停用，旁邊 Caption「目前已是預設」。
- 儲存失敗：頁面頂端（Caption 之下）顯示 `TextBlock.error`「儲存失敗：{原因}」，並把畫面上的狀態還原成設定檔的值。
- 窄視窗（`Window.narrow`）：右欄移到說明下方、靠左（仿 `MemoryPanel.axaml` 的 `StackPanel.memory-actions` 兩條 Style）。
- `ToolCatalog` 的 `ChangeToken`：不需要。這一頁只有使用者自己會改；開啟頁面時讀一次 `IModelCatalog.Current`。

## 實作分包

```
Phase 0（協調者） ──► Wave 1：15A、15B、15C、15D 同時進行 ──► 整合（協調者） ──► 人工驗收
```

四包修改的檔案完全不重疊。**所有包的共同規則**：

- 在自己的 git worktree 工作；不改 `plan/`、不 commit；只碰「可以新增／修改的檔案」清單裡的檔案。
- 程式碼註解與介面文字用繁體中文，風格仿照同一個資料夾裡的既有檔案。
- 遇到計畫沒寫到、或和程式現況不符的地方，**停下來在回報裡說明，不要自己猜**。
- 完成檢查一律包含：`dotnet build Lumi.sln` 零警告。
- 回報：改了哪些檔案、新增的測試數量與結果、和計畫不同的地方與原因、沒做完或卡住的項目。

### Phase 0：共用契約（協調者）

1. `Lumi.Core/Models/ToolSettings.cs`、`AppSettings.Tools`、`IModelCatalog.SetToolsAsync`（照「設計 › 資料」）。`JsonModelCatalog.SetToolsAsync` 先寫成 `MutateAsync(s => s with { Tools = tools.Normalize() }, ct)`（設定檔的讀寫由 15A 補）。三個 `FakeModelCatalog`（`tests/Lumi.Core.Tests/TestSupport/`、`tests/Lumi.App.Tests/TestSupport/`、`tests/Lumi.Tools.Tests/Memory/TestSupport/`）加上同樣的方法；用 `grep -rn ": IModelCatalog"` 確認沒有其他實作。
2. `ToolDescriptor` 加四個欄位；`Lumi.Core/Tools/ToolGroups.cs`；`ToolDefinition`、`ToolSnapshotJson`（含單元測試：往返、null、壞 JSON）。
3. `ChatRequest.Tools` 改型別；兩個 provider 改讀 `ToolDefinition`；`ConversationService` 第 244 行暫時改成 `tools.Select(ToolDefinition.From).ToList()`；修正 `tests/Lumi.Providers.Tests/` 七個檔案裡的 `new ChatRequest(...)`（`[tool]` → `[ToolDefinition.From(tool)]`）。
4. `IConversationStore` 的兩個快照方法（預設實作）；`SqliteSchema` 加 migration 9 的 SQL。
5. `ToolApproval` 列舉最後加 `AllowedAlways`（數值 4，現有數值不變）。
6. `Lumi.Core/Tools/IToolCatalog.cs`（介面與 `ToolCatalogGroup`）。
7. `memory_save` 設 `CanAlwaysAllow: false`、`show_details` 設 `CanDisable: false`、`Group: ToolGroups.Sprite`（15C 不再改這兩個欄位）。
8. `dotnet build` 零警告、`dotnet test` 全過後，分派 Wave 1。

### 包 15A：Infrastructure — 設定檔與快照存取（Wave 1）

- **目標**：`settings.json` 能讀寫 `tools` 區段；SQLite 能存取對話的工具快照。
- **前置**：Phase 0 合併之後。
- **先讀這些檔案**：
  - `src/Lumi.Infrastructure/Settings/JsonModelCatalog.cs`（仿照 `Memory` 區段的三處：`ReadMemory`、寫檔時的 `SerializeToElement(settings.Memory...)`、載入時的 `Memory = ReadMemory(file.Memory)`）
  - `src/Lumi.Infrastructure/Persistence/SqliteConversationStore.cs`（仿照 `GetSystemPromptAsync`／`SetSystemPromptAsync`）
  - `tests/Lumi.Infrastructure.Tests/SqliteConversationStoreTests.cs`（仿照 system prompt 的測試）
- **可以新增／修改的檔案**：
  - `src/Lumi.Infrastructure/Settings/JsonModelCatalog.cs`
  - `src/Lumi.Infrastructure/Persistence/SqliteConversationStore.cs`
  - `tests/Lumi.Infrastructure.Tests/` 底下：`JsonModelCatalogToolsTests.cs`（新）、`SqliteToolSnapshotTests.cs`（新）
- **步驟**：
  1. 設定檔的檔案結構加 `Tools` 欄位（JSON 名稱 `tools`），讀取用新的 `ReadTools(JsonElement?)`：缺少或格式錯誤回 `ToolSettings.Default`，讀到的值經過 `Normalize()`。
  2. 寫檔時寫入 `settings.Tools.Normalize()`。
  3. `SetToolsAsync` 保留 Phase 0 的寫法（確認會寫檔）。
  4. `SqliteConversationStore` 實作 `GetToolSnapshotAsync`（`SELECT tool_snapshot FROM conversations WHERE id = $id;`）與 `SetToolSnapshotAsync`（`UPDATE conversations SET tool_snapshot = $snapshot WHERE id = $id;`）。
- **測試**：
  - `JsonModelCatalogToolsTests`：`Missing_tools_section_uses_default`、`Malformed_tools_section_uses_default`、`SetToolsAsync_round_trips_through_file`（寫入後用新的 `JsonModelCatalog` 讀回相同的值）、`Saved_lists_are_sorted_and_distinct`。
  - `SqliteToolSnapshotTests`：`New_conversation_has_no_snapshot`（回 null）、`Snapshot_round_trips`、`Upgrade_from_version_8_keeps_conversations`（仿照既有 migration 測試的寫法；升級後舊對話的 `tool_snapshot` 是 null）。
- **完成檢查**：`dotnet test tests/Lumi.Infrastructure.Tests` 全過。
- **不要做**：不改 `SqliteSchema.cs`（Phase 0 已加 SQL）；不改 `IModelCatalog`。

### 包 15B：Core — `ToolCatalog` 與對話迴圈（Wave 1）

- **目標**：新對話凍結「已啟用」的工具；舊對話不變；「永遠允許」的工具不再詢問。
- **前置**：Phase 0 合併之後。
- **先讀這些檔案**：
  - `src/Lumi.Core/Chat/ConversationService.cs`（第 183～400 行：`RunTurnAsync`；仿照 `GetSystemPromptAsync` 那段「第一次送出時記下來」的寫法）
  - `src/Lumi.Core/Tools/ToolRegistry.cs`、`src/Lumi.Core/Tools/ToolTypes.cs`、`src/Lumi.Core/Tools/IToolCatalog.cs`、`src/Lumi.Core/Tools/ToolDefinition.cs`
  - `tests/Lumi.Core.Tests/TestSupport/InMemoryConversationStore.cs`、`tests/Lumi.Core.Tests/TestSupport/FakeModelCatalog.cs`
  - 既有的 `ConversationService` 工具測試（`tests/Lumi.Core.Tests/` 裡檔名含 `ConversationService` 或 `ToolLoop` 的檔案），仿照它們的寫法
- **可以新增／修改的檔案**：
  - `src/Lumi.Core/Tools/ToolCatalog.cs`（新）
  - `src/Lumi.Core/Chat/ConversationService.cs`
  - `tests/Lumi.Core.Tests/TestSupport/InMemoryConversationStore.cs`、`tests/Lumi.App.Tests/TestSupport/InMemoryConversationStore.cs`（加上快照的兩個方法，仿照 system prompt）
  - `tests/Lumi.Core.Tests/ToolCatalogTests.cs`（新）、`tests/Lumi.Core.Tests/ToolSnapshotFreezeTests.cs`（新）、`tests/Lumi.Core.Tests/AlwaysAllowTests.cs`（新）
- **步驟**：
  1. `public sealed class ToolCatalog(IToolRegistry registry, IModelCatalog settings) : IToolCatalog`：
     - `GetAll()`＝`registry.GetAll()`。
     - `GetEnabledForNewConversationAsync`＝`GetAll()` 去掉 `settings.Current.Tools.Disabled` 裡的名稱，但 `CanDisable=false` 的一律保留；回 `Task.FromResult`。
     - `Find(name)`＝`registry.Find(name)`（`IToolRegistry` 沒有 `Find` 的話，改用 `GetAll().FirstOrDefault(...)`，不要改介面）。
     - `ListGroups()`：依 `ToolGroups.Find(descriptor.Group)` 分組；找不到群組的工具放進「一般」；群組依 `Order`，組內依 `DisplayName`（`StringComparer.Ordinal`）；沒有工具的群組不回傳。
  2. `ConversationService` 建構式加一個可選參數 `IToolCatalog? toolCatalog = null`（放在最後面）；null 時用 `new ToolCatalog(registry, catalog)`。欄位名稱 `_toolCatalog`。
  3. 依「設計 › 對話迴圈」改 `RunTurnAsync`：
     - 第 195 行的 `availableTools` 改成：`model.Model.ToolsEnabled` 為 false → 空清單；否則依快照決定 `IReadOnlyList<ToolDefinition>`。快照只在**每一輪開頭讀一次**（for 迴圈外）。
     - 「已經有助理訊息」＝`_store.GetMessagesAsync` 的結果裡有任何 `ChatRole.Assistant`。
     - 執行時用名稱比對快照，用 `_toolCatalog.Find` 取得工具；找不到時建立 `ToolResultBlock(use.Id, "這個工具目前無法使用。", true, use.Name, null, "無法使用", 0, ToolApproval.NotRequired)`，並和「沒有這個工具」一樣送出 `ToolCallStarted`／`ToolCallCompleted`。
     - 權限段落：在 `_memory.IsAllowed` 之前加「永遠允許」的判斷（讀 `_catalog.Current.Tools.AlwaysAllow`，用 `IToolCatalog.CanAlwaysAllow(descriptor)` 再檢查一次），成立時 `approval = ToolApproval.AllowedAlways`。
  4. 檔案開頭的類別註解補一句：工具清單在一則對話內固定（M15 D1）。
- **測試**：
  - `ToolCatalogTests`：`Disabled_tools_are_excluded_from_new_conversations`、`Tools_that_cannot_be_disabled_stay_enabled`、`GetAll_ignores_settings`、`Groups_follow_order_and_skip_empty`、`Unknown_group_falls_back_to_general`、`CanAlwaysAllow_rules`（ReadOnly false、Write true、Network true、Destructive false、Write 但 `CanAlwaysAllow=false` 也是 false）。
  - `ToolSnapshotFreezeTests`：`First_send_stores_enabled_tools`、`Later_sends_reuse_snapshot_even_after_settings_change`（改設定後第二則訊息送出的定義逐字相同）、`Legacy_conversation_with_assistant_message_snapshots_all_tools`、`Disabled_tool_in_snapshot_still_executes`、`Tool_missing_from_catalog_returns_unavailable_error`、`Tool_not_in_snapshot_returns_unknown_tool`、`Model_without_tools_sends_no_tools_and_stores_no_snapshot`。
  - `AlwaysAllowTests`：`Always_allowed_write_tool_runs_without_asking`（approval 是 `AllowedAlways`、approval service 沒被呼叫）、`Destructive_tool_in_always_allow_still_asks`、`Memory_save_in_always_allow_still_asks`、`Always_allow_applies_immediately_in_existing_conversation`。
- **完成檢查**：`dotnet test tests/Lumi.Core.Tests` 與 `dotnet test tests/Lumi.Providers.Tests` 全過。
- **不要做**：不改 `IToolCatalog`、`ToolDescriptor`、`IConversationStore`（Phase 0 的契約）；不碰 DI 註冊檔。

### 包 15C：Tools — 群組與一句說明（Wave 1）

- **目標**：每個內建工具都有 `Group` 與 `Summary`。
- **前置**：Phase 0 合併之後。
- **先讀這些檔案**：`src/Lumi.Core/Tools/ToolTypes.cs`、`src/Lumi.Core/Tools/ToolGroups.cs`、任一個工具檔（例如 `src/Lumi.Tools/Files/ReadFileTool.cs`）的 `Descriptor`。
- **可以新增／修改的檔案**：`src/Lumi.Tools/` 底下 23 個工具檔（只改 `Descriptor` 的建構，用具名參數 `Group:`、`Summary:` 加在最後）；`tests/Lumi.Tools.Tests/ToolGroupTests.cs`（新）。
- **步驟**：照下表設定，`Summary` 原文照抄。`show_details` 的 `Group` 已由 Phase 0 設定，只補 `Summary`。

  | 工具 | Group | Summary |
  |---|---|---|
  | `get_current_time` | `General` | 查詢現在的日期與時間。 |
  | `calculate` | `General` | 計算數學算式。 |
  | `list_dir` | `FilesRead` | 列出你加入的資料夾裡有哪些檔案。 |
  | `read_file` | `FilesRead` | 讀取你加入的文字檔內容。 |
  | `search` | `FilesRead` | 在你加入的檔案裡搜尋文字。 |
  | `read_document` | `FilesRead` | 讀取 Word、Excel、PowerPoint、PDF 的文字。 |
  | `write_file` | `FilesWrite` | 新增檔案，或在檔案結尾加上內容。 |
  | `overwrite_file` | `FilesWrite` | 用新內容覆蓋整個既有檔案。 |
  | `copy_file` | `FilesWrite` | 複製檔案。 |
  | `move_file` | `FilesWrite` | 移動檔案或改名。 |
  | `create_folder` | `FilesWrite` | 建立資料夾。 |
  | `add_note` | `Notes` | 把一件事記到今天的筆記。 |
  | `read_notes` | `Notes` | 查看之前記的筆記。 |
  | `conversation_search` | `Notes` | 搜尋過去的對話內容。 |
  | `create_reminder` | `Reminders` | 在指定時間提醒你。 |
  | `start_timer` | `Reminders` | 開始倒數計時。 |
  | `list_reminders` | `Reminders` | 列出目前的提醒與計時器。 |
  | `cancel_reminder` | `Reminders` | 取消提醒或計時器。 |
  | `set_reminder_enabled` | `Reminders` | 暫停或恢復週期提醒。 |
  | `memory_save` | `Memory` | 記住關於你的長期事實。 |
  | `memory_update` | `Memory` | 修改一則記憶。 |
  | `memory_delete` | `Memory` | 刪除一則記憶。 |
  | `show_details` | （已設定） | 把完整內容放到工作台，讓你之後點開查看。 |

- **測試**：`ToolGroupTests`：`Every_builtin_tool_has_a_known_group`（`ToolGroups.Find` 不是 null）、`Every_builtin_tool_has_a_summary`（非空白、以「。」結尾、≤ 40 字）、`Only_show_details_cannot_be_disabled`、`Only_memory_save_opts_out_of_always_allow`。工具清單用 `services.AddLumiTools()` 建出來（仿照 `tests/Lumi.Tools.Tests/` 既有的組裝測試）。
- **完成檢查**：`dotnet test tests/Lumi.Tools.Tests` 全過。
- **不要做**：不改任何工具的 `Description`、權限、模式、逾時。

### 包 15D：App — 工具清單頁（Wave 1；本 Wave 唯一可以改 `Strings` 的包）

- **目標**：`ToolsPanel` 與 `ToolsViewModel` 可以獨立顯示與操作（掛進設定視窗由整合包做）；工具卡片顯示「永遠允許」。
- **前置**：Phase 0 合併之後。15B 還沒合併也能做：測試用 Phase 0 的 `IToolCatalog` 介面自己寫一個假實作。
- **先讀這些檔案**：
  - `src/Lumi.App/ViewModels/MemoryViewModel.cs`、`src/Lumi.App/Views/MemoryPanel.axaml`（整體寫法、樣式 class、`Window.narrow` 規則、`Border.badge`／`Border.warn-badge`）
  - `src/Lumi.App/ViewModels/IConfirmService.cs`、`tests/Lumi.App.Tests/TestSupport/FakeConfirmService.cs`
  - `src/Lumi.App/ViewModels/ToolCardViewModel.cs`（第 175～185 行的 approval 文字）
  - `src/Lumi.App/Resources/Strings.resx`、`Strings.cs`（新增字串兩邊一起改）
  - `tests/Lumi.App.Tests/MemoryViewModelTests.cs`、`tests/Lumi.App.Tests/Headless/MemoryPanelTests.cs`
  - `plan/ui-guide/ui-guide.html`（按鈕層級、文字層級、分隔線）
- **可以新增／修改的檔案**：
  - `src/Lumi.App/ViewModels/ToolsViewModel.cs`（新；含 `ToolGroupViewModel`、`ToolItemViewModel`）
  - `src/Lumi.App/Views/ToolsPanel.axaml`、`ToolsPanel.axaml.cs`（新）
  - `src/Lumi.App/ViewModels/ToolCardViewModel.cs`（只加 `AllowedAlways` 的對應）
  - `src/Lumi.App/Resources/Strings.resx`、`Strings.cs`
  - `tests/Lumi.App.Tests/TestSupport/FakeToolCatalog.cs`（新）、`tests/Lumi.App.Tests/ToolsViewModelTests.cs`（新）、`tests/Lumi.App.Tests/Headless/ToolsPanelTests.cs`（新）
- **步驟**：
  1. `ToolsViewModel(IToolCatalog tools, IModelCatalog settings, IConfirmService confirm)`：建構時用 `tools.ListGroups()` 與 `settings.Current` 建出群組與項目。
  2. `ToolItemViewModel`：`Name`、`DisplayName`、`Summary`、`PermissionText`、`IsDestructive`、`ModeText`（null＝不顯示）、`IsEnabled`（雙向）、`CanToggle`（＝`CanDisable`）、`ConfirmKind`（列舉 `NotNeeded`／`AlwaysAsk`／`Choosable`）、`AlwaysAllow`（雙向，只在 `Choosable` 時有意義）、`AlwaysAskTooltip`。
  3. `ToolGroupViewModel`：`Name`、`Items`、`IsAllEnabled`（`bool?`，三態，雙向）、`CountText`（「已啟用 {0}／{1}」）、`CanToggle`、`MemoryOffNote`（只有記憶群組、記憶關閉時有值）。
  4. 任何一個開關或確認方式改變 → 組出新的 `ToolSettings` → `await settings.SetToolsAsync(...)`。同一時間只會有一個儲存在進行（用 `SemaphoreSlim(1,1)`）。失敗時把 `SaveError` 設成「儲存失敗：{例外訊息}」，並從 `settings.Current` 重新載入狀態。
  5. 把 `AlwaysAllow` 改成 true 時：先 `await confirm.ConfirmDestructiveAsync(訊息, "永遠允許")`；false → 改回 false、不儲存。
  6. `ResetToDefaultsCommand`：先確認；確認後 `SetToolsAsync(ToolSettings.Default)`。`CanResetToDefaults`＝目前不是預設；不能按時 `ResetBlockedReason`＝「目前已是預設」。
  7. `ModelToolsOffWarning`：`settings.Current` 的工作台預設模型存在且 `ToolsEnabled` 是 false 時，回「目前的預設模型「{0}」關閉了「啟用工具」，對話中不會送出任何工具。可以到「AI 連線」的「模型設定」打開。」（`{0}`＝模型的 `DisplayName`）；否則 null。
  8. `ToolsPanel.axaml` 依「設計 › 畫面」排版。頁首（`SettingsPageHeader`）由整合包放在 `SettingsWindow`，這個面板從 Caption 開始。間距只能用 4、8、12、16、24、32；顏色只能用既有的 `Lumi*Brush` token。
  9. `ToolCardViewModel`：`ToolApproval.AllowedAlways => Strings.Tool_ApprovedAlways`。
  10. 新增字串（key 與文字照抄）：

      | Key | 文字 |
      |---|---|
      | `Tools_MenuTitle` | 工具 |
      | `Tools_MenuSubtitle` | 能用的工具與確認方式 |
      | `Tools_HeaderEyebrow` | 工具 |
      | `Tools_HeaderTitle` | Lumi 能用的工具 |
      | `Tools_HeaderDescription` | 選擇 Lumi 在對話中可以使用哪些工具，以及使用前要不要先問你。 |
      | `Tools_EffectNote` | 啟用與關閉從下一則新對話開始生效；進行中的對話維持原本的工具。 |
      | `Tools_ConfirmNote` | 確認方式改了立即生效。 |
      | `Tools_ModelToolsOff` | 目前的預設模型「{0}」關閉了「啟用工具」，對話中不會送出任何工具。可以到「AI 連線」的「模型設定」打開。 |
      | `Tools_GroupCount` | 已啟用 {0}／{1} |
      | `Tools_MemoryOff` | 長期記憶已在「記憶」頁關閉，這些工具目前不會執行。 |
      | `Tools_PermissionReadOnly` | 唯讀 |
      | `Tools_PermissionWrite` | 寫入 |
      | `Tools_PermissionNetwork` | 網路 |
      | `Tools_PermissionDestructive` | 破壞性 |
      | `Tools_ModeWorkbenchOnly` | 只限工作台 |
      | `Tools_ModeSpriteOnly` | 只限小精靈 |
      | `Tools_ConfirmNotNeeded` | 不需確認 |
      | `Tools_ConfirmAlwaysAsk` | 每次都會詢問 |
      | `Tools_ConfirmAsk` | 每次詢問 |
      | `Tools_ConfirmAlways` | 永遠允許 |
      | `Tools_ConfirmLabel` | 確認方式 |
      | `Tools_AlwaysAskMemoryTooltip` | 寫進記憶的內容每則訊息都會送出，所以一律先問你。 |
      | `Tools_AlwaysAskDestructiveTooltip` | 這個工具會覆蓋或刪除內容，所以一律先問你。 |
      | `Tools_CannotDisable` | 小精靈靠它把完整內容放到工作台，不能關閉。 |
      | `Tools_AlwaysAllowConfirm` | 之後 Lumi 使用「{0}」時不會再先問你，所有對話都是如此。確定要永遠允許嗎？ |
      | `Tools_ResetDefaults` | 全部恢復預設 |
      | `Tools_ResetConfirm` | 所有工具都會重新啟用，確認方式都改回「每次詢問」。 |
      | `Tools_ResetConfirmButton` | 恢復預設 |
      | `Tools_ResetBlocked` | 目前已是預設 |
      | `Tools_SaveFailed` | 儲存失敗：{0} |
      | `Tool_ApprovedAlways` | 已允許（永遠允許） |

- **測試**：
  - `ToolsViewModelTests`：`Groups_and_items_come_from_catalog`、`Toggling_item_saves_disabled_list`、`Group_checkbox_reflects_partial_state`（`null`）、`Group_toggle_from_partial_enables_all`、`Group_toggle_skips_tools_that_cannot_be_disabled`、`Always_allow_requires_confirmation`（取消時不儲存、值改回 false）、`Always_allow_only_offered_for_choosable_tools`、`Reset_restores_defaults_after_confirmation`、`Reset_disabled_when_already_default`、`Save_failure_shows_error_and_reloads`、`Model_tools_off_warning_shows_model_name`、`Memory_off_note_only_on_memory_group`。
  - `Headless/ToolsPanelTests`：`Renders_all_groups_at_default_width`（1040）、`Renders_without_clipping_at_minimum_width`（720）、`Narrow_moves_confirm_column_below`（強制加 `narrow` class，仿照 `MemoryPanelTests`）、`Locked_tool_checkbox_is_disabled`、`Destructive_badge_uses_danger_style`。
- **完成檢查**：`dotnet test tests/Lumi.App.Tests` 全過（含既有的按鈕層級、色碼、間距掃描）。
- **不要做**：不改 `SettingsWindow.axaml`、`SettingsViewModel.cs`、`AppServiceCollectionExtensions.cs`、`Icons.axaml`（整合包做）；不在 View 裡寫色碼。

### 整合（協調者）

1. 合併 15A～15D，解決衝突，跑完整測試。
2. DI：`IToolCatalog` → `ToolCatalog`（`AddLumiCore`）；確認 `ConversationService` 拿到同一個實例（`CompositionRootTests` 加一條，直接檢查欄位，同 M14）；`ToolsViewModel`。
3. 設定視窗：`SettingsSection.Tools`、Menu 項目（`wrench` 圖示加進 `Icons.axaml`）、`SettingsViewModel` 的分類索引改成 0～6、`ShowTools`、頁首（eyebrow／標題／說明，沒有主要動作）；更新依賴分類索引的既有測試。
4. 文件：`docs/ui-spec.md` 第 7 節（Menu 表格加一列、「工具」一段）、第 5 節（永遠允許）；`agent.md` 第 8 節（工具清單在對話內固定、永遠允許的規則與例外）；主計畫（M15 狀態、決策紀錄）；本檔實作紀錄。

## 驗收標準

- [ ] `dotnet build` 零警告、`dotnet test` 全過；架構測試通過。
- [ ] 新對話送出的工具＝已啟用的工具；對話中途改設定，下一則訊息送出的工具定義逐字不變。
- [ ] 舊對話（M15 之前開的）繼續聊不會 400。
- [ ] 「永遠允許」的工具不再出現確認；`Destructive` 與 `memory_save` 沒有這個選項。
- [ ] 設定頁在 1040 與 720 寬都不破版；五套主題 × 淺深可讀（沿用 M12 的對比測試）。

## 人工驗收清單（使用者；需要真實金鑰；Mac 與 Windows 各一次）

- [ ] 關閉「寫入檔案」群組 → 開**新對話**請 Lumi 建立檔案 → 它說做不到（沒有工具卡片）。
- [ ] 在一則已經用過工具、出現過思考的對話中，到設定關閉幾個工具 → 回到同一則對話繼續問 → 沒有 400，原本的工具照常可用。
- [ ] 把 `add_note` 設成「永遠允許」（會先確認）→ 請 Lumi 記筆記 → 沒有確認，工具卡片寫「已允許（永遠允許）」。
- [ ] `overwrite_file`、`memory_save` 沒有「永遠允許」選項；滑鼠停在「每次都會詢問」看得到原因。
- [ ] `show_details` 的勾選框不能取消，看得到原因。
- [ ] 記憶頁關閉長期記憶 → 工具頁的記憶群組出現說明。
- [ ] 把預設模型的「啟用工具」關掉 → 工具頁出現 Warning。
- [ ] 「全部恢復預設」先確認，恢復後按鈕停用。
- [ ] 設定視窗縮到最小寬度，版面完整。

## 實作紀錄

### Phase 0（2026-10-25，協調者完成，已 commit）

與計畫不同或補充的地方（子代理以這裡為準）：

- 有**四個**實作 `IModelCatalog` 的測試替身，不是三個：多了 `tests/Lumi.Speech.Tests/TestSupport/FakeSpeechCatalog.cs`。四個都已補上 `SetToolsAsync`；`FakeModelCatalog`（App.Tests）另有 `ToolSaves` 清單。`JsonModelCatalog.SetToolsAsync` 已是 `MutateAsync(s => s with { Tools = tools.Normalize() })`，但設定檔的讀寫（`tools` 區段）**尚未做**，是 15A 的工作。
- `memory_save` 與 `show_details` 兩個工具的 `Group`、`Summary`、`CanAlwaysAllow`／`CanDisable` 已在 Phase 0 設好；**15C 不要再改這兩個檔案**，其餘 21 個工具才由 15C 處理。
- `ChatRequest.Tools` 已改為 `IReadOnlyList<ToolDefinition>?`；`ConversationService` 暫時用 `tools.Select(ToolDefinition.From).ToList()`（15B 會改成快照）。既有測試裡的 `t.Descriptor.Name` 已改成 `t.Name`。
- migration 9 已加；既有升級測試（降版用的 SQL）已補 `DROP COLUMN tool_snapshot`，並已新增 `UpgradeFromVersion8_AddsToolSnapshot_AndKeepsConversations`；**15A 不必再寫這個測試**，只需寫 `GetToolSnapshotAsync`／`SetToolSnapshotAsync` 的測試。
- `ToolSnapshotJson`、`ToolSettings` 的單元測試已在 `tests/Lumi.Core.Tests/Tools/`。
- Phase 0 結束時：build 零警告、全部測試通過（Core 676、Tools 969、Providers 147、Infrastructure 365、App 1734）。
