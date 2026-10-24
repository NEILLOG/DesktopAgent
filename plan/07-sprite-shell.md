# 07 小精靈殼與光球角色

- 狀態：進行中（Q1～Q3 已於 2026-10-21 照建議確認；7A～7E 程式完成，待手動驗收 C8、7D、E6）
- 對應里程碑：M7
- 前置：M3、M5 程式完成（M5 的 C4 手動驗收可以並行，但 7D 的「兩句內」要靠它的結果）。M0 的 Windows 驗證已於 2026-10-21 由使用者回報通過（風險一節的第一項因此解除，7C 照 POC 的單一視窗做）

## 目標

小精靈模式正式上線：透明無框的桌面光球，游標靠近才浮出輸入列，用氣泡回答，並用動畫表現思考、呼叫工具、需要確認等狀態；和工作台共用同一則對話，兩邊可以來回切換。
殼與角色嚴格分離：照著 `docs/sprite-dev-guide.md` 新增一個角色，不需要修改殼。

## 範圍

做：
- `Lumi.Sprites.Abstractions`（介面、狀態、互動、manifest、`SpriteLimits`，版本 1.0）與 `Lumi.Sprites.Orb`（光球）
- 殼：視窗、點擊穿透、靠近偵測、戳與拖曳判定、幀率控制、位置記憶、尺寸檔位、自動休眠、右鍵選單、托盤圖示
- 對話整合：共用對話會話、對話事件轉成角色狀態、氣泡、精簡輸入列、氣泡內的權限確認、兩個視窗的模式切換與「查看中」
- 設定 ›「小精靈」：角色選擇與狀態預覽、尺寸、角色語氣開關
- 把 M5 的開發用「小精靈模式」切換換成真正的模式切換

不做：
- 語音（麥克風、收音列、`Listening`／`Transcribing` 的觸發）：M6 暫緩。角色**仍然支援**這兩個狀態（`docs/sprite-dev-guide.md` 的狀態表），只是殼目前不會觸發；輸入列先不放麥克風鈕（M6 再加）
- 素材型角色、manifest 的 JSON 載入、角色資料夾：M8（M7 只定義 `SpriteManifest` 這個型別）
- 視窗位置與大小記憶（工作台的）、歷史對話：M9（M9 程式已完成待驗收，見 plan/09-history-polish.md）
- macOS 的 accessory 模式（不顯示 Dock 圖示）、安裝程式、開機自動啟動：M11

## 現況（開工前盤點）

- 工作台工具列的「小精靈模式」按鈕是停用的（`WorkbenchWindow.axaml`）；M5 加了開發用切換（`Lumi:Developer:ShowSpriteModeToggle`），7E 要移除它。
- `ChatMode.Sprite`、`ModeFormatPromptLayer`、`show_details`、`SpriteModelId`、`ISpriteVoiceProvider`（目前永遠回 null）都已就緒。
- **對話的「執行」目前綁在 `WorkbenchViewModel` 裡**：目前對話 Id、取消權杖、`IsResponding`、整段事件迴圈（約 150 行，同時負責更新訊息列表）。小精靈要送訊息、要看到同一份事件、要能在另一個視窗回答權限確認，所以這段一定要先拆出來（Q2）。
- 權限確認：`ToolApprovalService` 是單例，工具卡片上的按鈕呼叫 `Resolve(callId, decision)`，對話迴圈在另一端等。**回答之後只有按的那張卡片會更新**——ui-spec 2.2 要求的「一邊回答，另一邊同步更新」還沒有。
- App 以 `WorkbenchWindow` 當 `MainWindow`：關掉它整個程式就結束。小精靈模式下工作台是隱藏的，生命週期要改成明確結束（Q3）。
- `Lumi.Platform` 目前只有金鑰儲存，沒有任何視窗用的原生程式。POC 的 `Platform/`（`WindowsOverlay`、`MacOverlay`）要搬進來；POC 的版本直接拿 Avalonia 的 `Window`，而 `IOverlayPlatform` 在 agent.md 是 Core 的介面（Core 不能引用 Avalonia），所以要改成只吃原生視窗控制代碼（見設計）。
- `ui-spec` 第 8 節（小精靈的對話介面）標著「草稿，待確認」，是主計畫「待確認事項」的最後一項（Q1）。
- POC 的 `OrbControl` 已經有 `SetState`、`Advance(dt)`、`BeginPoke`／`EndPoke`、`HitOrb`、`LookTarget`，幾乎就是 `ISpriteView`，主要工作是整理與搬家，不是重寫。

## M7 拆成五個部分

| 部分 | 名稱 | 產出 | 依賴 |
|---|---|---|---|
| **7A** | 共用對話會話 | `ChatSession`：目前對話、模式、回應中、停止、事件扇出；`WorkbenchViewModel` 改成它的觀察者；權限確認的「已回答」事件。**沒有任何畫面上的變化**，既有測試全過 | M3、M4、M5 |
| **7B** | 抽象層與光球角色 | `Lumi.Sprites.Abstractions`、`Lumi.Sprites.Orb`、狀態退化解析、`SpriteLimits` 縮放、`AddSprite<T>()` | — |
| **7C** | 殼：視窗與平台 | `IOverlayPlatform`（Core）＋ Windows／macOS 實作（`Lumi.Platform`）、`SpriteWindow`、`FrameDriver`、靠近偵測、戳與拖曳判定、點擊穿透、位置與尺寸、自動休眠、托盤、App 生命週期 | 7B |
| **7D** | 對話整合 | 對話事件→角色狀態、氣泡、精簡輸入列、氣泡內權限確認、錯誤與重試、模式切換與「查看中」、右鍵選單、角色語氣 | 7A、7C |
| 7E | 設定頁與收尾 | 設定 ›「小精靈」、`SpriteSettings`、狀態預覽、移除開發切換、換角色不改殼的驗證、指南更新、手動驗收 | 7D |

7A、7B 互不依賴，可以交錯做；7C 結束時殼可以單獨跑（用假的狀態驅動，像 POC）；每段結束都要能編譯、測試全過。

主計畫 M7 項目的對應：

| 主計畫項目 | 部分 |
|---|---|
| `Lumi.Sprites.Abstractions`：介面、狀態、manifest、`SpriteLimits` | 7B |
| 小精靈殼：從 POC 的 MainWindow 拆出；靠近才浮現輸入列、氣泡、右鍵選單、位置記憶 | 7C（視窗、輸入列外殼、選單外殼、位置）、7D（氣泡與輸入列的行為、選單的動作） |
| 光球角色：從 POC 的 OrbControl 搬入 | 7B |
| 對話事件轉成角色狀態 | 7D |
| 與工作台共用對話；「在工作台開啟完整內容」 | 7A、7D |
| 角色選擇與狀態預覽（設定頁） | 7E |

---

## 開工前要確認的事

以下是影響結構或行為的決定，我各自給了建議。**Q1～Q3 已於 2026-10-21 全部照建議確認。**

| # | 問題 | 建議 | 影響 |
|---|---|---|---|
| Q1 | `docs/ui-spec.md` 第 8 節（氣泡、輸入列、氣泡內權限確認、右鍵選單）與 2.2（回應中、查看中）是草稿，要照它做嗎？ | **照著做**，語音相關（8.4、麥克風鈕）留給 M6。實作時我把兩個模式的關係定成下面這樣，請特別看這一段：**兩種模式互斥**——工作台模式時小精靈隱藏；按工作台的「小精靈模式」＝隱藏工作台、顯示小精靈；小精靈模式下「開啟工作台」可以讓兩個視窗同時開著，**在工作台送出才算切換到工作台模式（小精靈隱藏）**；小精靈回應中開啟的工作台是唯讀的「查看中」，回應結束自動解除。**（2026-10-24 修改：兩個視窗不再同時開著。「開啟工作台」＝隱藏小精靈並切到工作台模式；回應中先以「查看中」開啟，回應結束才切換；查看中關閉工作台則回到小精靈。）**確認後我把 ui-spec 第 8 節的「草稿」拿掉。 | 7D、7E |
| Q2 | 共用對話的做法：現在整段回應流程都在 `WorkbenchViewModel` 裡。小精靈怎麼拿到同一份事件？ | **抽出 `ChatSession`（放 Core）**：它擁有目前對話 Id、模式、回應中旗標、取消，並把每個 `ChatEvent` 發給所有觀察者；`WorkbenchViewModel` 和小精靈各自是觀察者（工作台負責更新訊息列表，小精靈負責氣泡與角色狀態）。這是一次不改行為的重構，放在 7A 最先做，用既有的 85 個左右的工作台測試當安全網。（替代做法：小精靈直接呼叫 `WorkbenchViewModel` 送訊息、訂閱它的事件。改動小，但小精靈變成依賴另一個視窗的 ViewModel，工作台視窗不存在或被簡化時就會出問題。） | 7A、7D |
| Q3 | 小精靈隱藏（「暫時隱藏 30 分鐘」）、或只剩小精靈時，沒有任何視窗可以按。要不要加托盤圖示（Windows 通知區域、macOS 選單列）？關閉工作台視窗又該怎麼辦？ | **加最小的托盤圖示**：「顯示小精靈」「開啟工作台」「結束 Lumi」三項。**App 改成明確結束**（`ShutdownMode.OnExplicitShutdown`）：工作台模式下按關閉＝結束程式（和現在一樣）；小精靈模式下開著的工作台按關閉＝只關掉工作台。圖示先用程式畫的簡單光球（M11 再換正式圖示）。沒有托盤的話，隱藏後只能重開程式。 | 7C、7D |

其他我直接定的事（有意見再指出；數字都集中在 `SpriteShellOptions`，之後可調）：

- **沿用 POC 驗證過的數字**：靠近半徑 150（以直徑 104 為基準，隨尺寸等比）、離開後 0.6 秒才收起、長按 0.45 秒算戳、位移超過 4 DIP 算拖曳、待機 60 秒進入休眠、幀率 60／30／12。
- **氣泡**：寬度上限 280；完成後停留 8 秒淡出，滑鼠停在上面時暫停；字數上限 120 個字形叢集（比提示詞的「約 60 字」寬一倍，是第三層硬性上限），超過就在叢集邊界截斷並加「…」，同時顯示「在工作台開啟完整內容 →」。截斷只影響氣泡，存檔與工作台都是完整的。
- **`Done` 停 3 秒回 `Idle`**；`Error` 停到氣泡被關掉或使用者再互動；`Attention` 停到確認被回答。
- **啟動時回到上次的模式**（存在設定裡）；第一次啟動是工作台模式。
- **位置記憶**：記光球中心的螢幕座標，存在 `settings.json`（拖曳結束時存）；啟動時如果那個點已經不在任何螢幕內（拔掉外接螢幕），改放主螢幕右下角。
- **設定檔**：`AppSettings` 加 `Sprite`（init 屬性，和 `Persona` 同樣做法），版本號維持 1。
- **角色語氣**：`ISpriteVoiceProvider` 改成讀目前角色 manifest 的 `voice`，設定可以關閉；光球沒有語氣（M7 不會有實際效果，M8 的素材型角色才會用到）。載入時拒絕超過 200 字或像指令的語氣的檢查放在 M8（那時才有外部來源的 manifest），M7 只定義 `SpriteLimits.VoiceMaxLength`。
- **診斷 HUD**（POC 的 F1）保留成開發選項（`Lumi:Developer:ShowSpriteHud`），補 M0 還沒量的耗能數據。
- **氣泡的文字一律經過 `EmojiTextBlock`**（agent.md 第 12 節），不用 Markdown。

---

## 設計

### 專案與相依

```
Lumi.App ──► Lumi.Sprites.Orb ──► Lumi.Sprites.Abstractions
   │                                        ▲
   └───────────────────────────────────────┘
```

- `Lumi.Sprites.Abstractions`：只引用 Avalonia 的基礎型別（`Control`、`Point`），**不引用 Core、Providers、Platform**。版本常數 `SpriteAbstractionsVersion = 1.0`。
- `Lumi.Sprites.Orb`：只引用 Abstractions 與 Avalonia。
- 殼（視窗、`FrameDriver`、互動判定、ViewModel）都在 `Lumi.App`，放在 `Sprites/` 資料夾；它是唯一同時認識「對話」和「角色」的地方。
- 新增架構測試：`Lumi.Sprites.*` 不引用 Core／Providers／Infrastructure／Platform／Tools；原始碼不出現 `DispatcherTimer`、`Window`、`DllImport`、`LibraryImport`、`Stopwatch`（角色不碰視窗、平台、自己開計時器）。

### 抽象層（7B）

```csharp
public enum SpriteState { Idle, Listening, Transcribing, Thinking, Tool, Streaming, Done, Attention, Error, Sleep }

public abstract record SpriteInteraction
{
    public sealed record Near(bool IsNear) : SpriteInteraction;
    public sealed record Look(Vector Direction) : SpriteInteraction;     // -1..1
    public sealed record PokeBegin(Point At) : SpriteInteraction;
    public sealed record PokeEnd : SpriteInteraction;
    public sealed record DragBegin : SpriteInteraction;
    public sealed record DragEnd : SpriteInteraction;
}

public sealed record SpriteManifest(
    int SchemaVersion, string Id, string Name, string Author, string Version,
    Size Box, double EffectMargin, SpriteHitShape HitShape, Point BubbleAnchor,
    string? Voice, IReadOnlySet<SpriteState> Supported, IReadOnlyDictionary<SpriteState, SpriteState> Fallback);

public interface ISpriteFactory { SpriteManifest Manifest { get; } ISpriteView Create(SpriteOptions options); }
public interface ISpriteView { /* docs/sprite-dev-guide.md 第 4 節，原樣 */ }

public static class SpriteLimits { /* 長寬比 0.5～2、最長邊 1.6 倍、外擴 0.4、語氣 200 字、檔位 64/80/104 */ }
```

- `SpriteStateResolver.Resolve(state, manifest)`：依指南的退化欄（`Transcribing → Listening → Idle` 這種可串接的規則）往下找，manifest 的 `Fallback` 優先，最後一定落到 `Idle`；有循環時也要停（測試）。**殼負責解析**，角色的 `SetState` 只會收到它自己支援的狀態。
- `SpriteLimits.Fit(manifest, tier)`：把角色框縮放到上限內，回傳實際的框與縮放比，超出時回報警告文字（殼寫進 log）；角色不能覆寫。
- `AddSprite<T>()`：`services.AddSingleton<ISpriteFactory, T>()`；`ISpriteCatalog` 列出所有已註冊的角色（M8 會把素材型的也加進來）。
- 指南第 10 節「版本紀錄」把 1.0 的「規劃中」改成實際日期。

### 光球角色（7B）

- 從 `poc/Lumi.OrbPoc/OrbControl.cs` 搬進 `Lumi.Sprites.Orb`，包成 `OrbSpriteFactory`／`OrbSpriteView`。保留 POC 的繪製（同心漸層、不用 `BlurEffect`、多色標邊緣）。
- 要改的只有邊界：`State` 改用 `SpriteState`；`OrbControl` 上殼才該管的東西（`Dragging`、`Near`、`LookTarget` 的公開設定）改成經由 `OnInteraction`；`Advance(double)` 改收 `TimeSpan`；補 `PreferredFps`（互動中 60、待機 30、休眠 12，由狀態與最近互動推導）、`HitTest`、`BubbleAnchor`（頭頂）。
- 尺寸：以直徑為基準等比縮放，`SpriteOptions` 帶尺寸檔位。`EyeMode`（POC 的右鍵選項）不進正式版，用預設的「偶爾」。
- 測試（headless）：每個狀態都能設定、不丟例外、畫得出非空白；`HitTest` 在圓內為真、框外為假；`PreferredFps` 隨狀態變；戳與拖曳的互動序列不丟例外。

### 平台（7C）

`IOverlayPlatform` 放在 Core，**不吃 Avalonia 型別**：

```csharp
public readonly record struct OverlayPoint(double X, double Y);

public interface IOverlayPlatform
{
    bool IsSupported { get; }                        // false＝退化：沒有游標輪詢與點擊穿透
    void Attach(nint nativeWindowHandle);            // 視窗 Opened 之後
    bool TryGetCursorInWindow(double renderScaling, double clientHeightDip, out OverlayPoint dip);
    void SetClickThrough(bool on);
}
```

- `WindowsOverlay`、`MacOverlay` 搬進 `Lumi.Platform`（P/Invoke 只能在這裡，標 `[SupportedOSPlatform]`），`NullOverlay` 當退化。POC 的 `WS_EX_LAYERED` 開關（`--no-layered`）保留成 `Lumi:Developer:OverlayLayered`，M0 的 Windows 驗證結果出來前先預設照 POC（加 `LAYERED`）。
- macOS：沿用 POC 的 `setHasShadow:false`、`CanJoinAllSpaces`、`FullScreenAuxiliary`。
- 平台測試：`NullOverlay` 的行為；兩個真實實作只能手動驗證（既有的 `Lumi.Platform.Tests` 在不同平台跳過）。

### 殼的視窗（7C）

- `SpriteWindow`：透明、無框、置頂、不在工作列、不可調整大小。視窗大小由「角色框＋特效外擴＋氣泡＋輸入列」自動算（指南第 5 節）；氣泡與輸入列的空間先固定預留，不隨內容改視窗大小（避免拖曳與動畫時視窗一直變形）。
- **點擊穿透**（POC 的做法）：每 33 ms 輪詢游標；游標在「可互動區」才關閉穿透，其餘一律穿透。可互動區＝角色的 `HitTest`、**有按鈕的**氣泡（權限確認、連結、重試）、已浮現的輸入列、開著的右鍵選單、按住中。沒有按鈕的氣泡維持穿透，它的「滑鼠停留」靠輪詢的游標座標判斷。
- `FrameDriver`：從 POC 搬進 `Lumi.App/Sprites`，統一驅動角色的 `Advance`；殼取「角色的 `PreferredFps`」與殼自己的需求（氣泡動畫中不降）較大者。元件不自己開計時器（agent.md 第 13 節）。
- **互動判定抽成純邏輯 `SpriteInteractionTracker`**（吃座標與時間，吐 `SpriteInteraction` 與「戳／拖曳」事件，時間用 `TimeProvider`）：靠近、離開延遲、長按戳、拖曳門檻、戳住後不會變拖曳、短按後立刻拖。POC 手動驗證過的規則變成有單元測試的程式。
- 拖曳：POC 的 5b 要選「手動設定 Position」或「系統 `BeginMoveDrag`」，Mac 驗證結果記在 POC README，7C 開工時照那個結果選；拖曳結束夾回螢幕內、存位置。
- 自動休眠：待機 60 秒且沒人靠近 → `Sleep`；游標靠近喚醒。休眠時 12 fps。
- 尺寸檔位小 64／中 80／大 104，預設中；切換時視窗重算、角色等比縮放、位置以光球中心為準不跳動。
- 多螢幕與 DPI：位置夾到「光球中心所在的螢幕」的可見範圍內；跨螢幕後 `RenderScaling` 變了要重新計算。
- **托盤圖示**（Q3）：Avalonia `TrayIcon`＋`NativeMenu`。圖示用程式畫的光球（`RenderTargetBitmap` 輸出 PNG 資源），先不做深淺色版本。
- **App 生命週期**（Q3）：`ShutdownMode.OnExplicitShutdown`；「結束 Lumi」（托盤、右鍵選單）是唯一的結束路徑，另外工作台模式下關閉工作台視窗也呼叫它。

### 共用對話會話（7A）

```csharp
// Lumi.Core/Chat
public interface IChatSession
{
    string? ConversationId { get; }
    ChatMode ActiveMode { get; }
    bool IsResponding { get; }

    event EventHandler<ConversationChangedArgs>? ConversationChanged;   // 切換或新對話
    event EventHandler<RunStartedArgs>? RunStarted;                     // 帶模式與觸發來源
    event EventHandler<ChatEvent>? EventReceived;                       // 每個 ChatEvent，依序
    event EventHandler<RunFinishedArgs>? RunFinished;                   // 失敗／被停止／未預期錯誤；收尾做完才發
    event EventHandler<ChatMode>? ModeChanged;

    Task SendAsync(string text, CancellationToken ct = default);
    Task RetryAsync(CancellationToken ct = default);
    Task RegenerateAsync(CancellationToken ct = default);
    void Stop();
    Task StopAndWaitAsync();
    Task OpenConversationAsync(string conversationId);                  // 回應中先停止
    Task NewConversationAsync();
    bool TrySetMode(ChatMode mode);                                     // 回應中回 false（ui-spec 2.2）
}
```

- 搬進 `ChatSession` 的：`_conversationId`、`_responseCts`、`_responseDone`、`IsResponding`、`await foreach` 迴圈與取消／未預期錯誤的收尾（`StreamPhaseAsync` 去掉更新畫面的部分）。
- **留在 `WorkbenchViewModel`（變成觀察者）的**：訊息列表的組成（氣泡、工具卡片、群組）、`LastUsage`、連線狀態標記、金鑰提示、錯誤列、重新產生時移除最後一輪的畫面動作。它訂閱 `EventReceived`，用和現在完全相同的規則更新畫面。
- 事件在呼叫者的同步內容上發出（正式程式是 UI 執行緒），測試用同步內容。觀察者丟例外不能中斷回應：記 log、繼續。
- 「查看中」不屬於會話：它只是「作用中的回應是小精靈發起的」＋「工作台視窗開著」，由 7D 的 `AppModeCoordinator` 推導。
- `IToolApprovalResponder` 加 `event Resolved(callId, decision)`；`ToolApprovalService` 不論是誰回答都發出。`ToolCardViewModel` 訂閱它，所以在氣泡回答時工作台的卡片同步變成「已允許」「你已拒絕」，反過來也一樣（這也順便修掉單一視窗時「從別處解決」的洞）。
- 重構的安全網：既有 `WorkbenchViewModelTests`、`WorkbenchConversationsTests`、`WorkbenchToolCardsTests`、`WorkbenchContextTests`、`WorkbenchDetailsAndModeTests` 全數不改斷言（只改建構方式）就要過；另外加 `ChatSessionTests`（事件順序、停止、重複送出被拒、模式在回應中不能改、觀察者例外、多個觀察者收到同一份事件）。

### 對話事件→角色狀態（7D）

純邏輯 `SpriteStateMapper`（吃 `ChatEvent`／會話事件，吐 `SpriteState`，計時用 `TimeProvider`），**角色不知道對話**：

| 來源 | 狀態 |
|---|---|
| `RunStarted`（送出後、第一個事件前） | `Thinking` |
| `ThinkingDelta` | `Thinking` |
| `TextDelta` | `Streaming` |
| `ToolCallStarted` | `Tool` |
| `ApprovalRequired` | `Attention`（被回答後回 `Tool`） |
| `ToolCallCompleted` | `Thinking`（等模型根據結果回應） |
| `Completed`（`StopReason.ToolUse`） | 不變（這一輪還沒結束） |
| `Completed`（其他）且是這一輪最後一個 | `Done`，3 秒後 `Idle` |
| `Failed`／未預期錯誤 | `Error` |
| 使用者停止 | `Idle` |
| 待機 60 秒無人靠近 | `Sleep`（游標靠近喚醒） |

`Listening`／`Transcribing`：M6 才會有來源。

### 氣泡（7D）

`SpriteBubbleViewModel`（純邏輯，計時用 `TimeProvider`），依 ui-spec 8.1：

- 狀態：隱藏／狀態一行（思考或工具；沿用工具卡片標題「動作 · 對象」）／串流中／完成／等待確認／錯誤。
- 同一時間只顯示最新一則，不顯示使用者的問題，也不顯示上一輪。
- 串流文字進到氣泡，超過字數上限就截斷（見上）。有完整內容的條件：呼叫了 `show_details`、回答被截斷、或 `StopReason.MaxTokens`，這時底下出現「在工作台開啟完整內容 →」，由使用者自己點。
- 完成後 8 秒淡出，游標停在氣泡上暫停計時；等待確認與錯誤不自動淡出。
- **氣泡內的權限確認**（8.3）：預設精簡（一句說明、對象用等寬字、「允許」＝允許一次、「拒絕」、「更多選項」）；展開後「允許一次／這則對話都允許／拒絕／在工作台檢視」；`Destructive` 標「無法復原」、列出所有對象、只有「允許刪除」「拒絕」。允許後氣泡回到「已允許 · 寫入檔案中…」。「在工作台檢視」開啟查看中的工作台並捲到那張卡片。
- **錯誤**：一句說明＋「重試」「在工作台查看」；缺金鑰、金鑰無效、金鑰讀不出來這幾種直接帶「開啟設定」（沿用 `IApiKeyPrompt`，存好金鑰後自動重試，和工作台同樣）。
- 顏色用共用主題 token，另外加細框線與陰影，確保在任何桌面背景上都看得清楚。

### 精簡輸入列（7D）

- 游標靠近才浮現：「問小精靈…」、送出。Enter 送出、Shift+Enter 換行**不要**（單行），中文輸入法選字的 Enter 不能當送出（POC 清單第 6 項，Mac 已驗證的做法搬過來）。
- 回應中可以打草稿，送出鈕換成停止。
- 送出前會話的模式必須是 `Sprite`；在小精靈送出＝這一輪用 `ChatMode.Sprite`、工作台預設模型（2026-10-24 起不再有 `SpriteModelId`）、較低的 `max_tokens`（M5 已實作）。

### 模式與視窗（7D）：`AppModeCoordinator`

單一個負責「哪個視窗顯示」的服務（`Lumi.App/Services`），把 Q1 的規則集中在這裡：

| 事件 | 結果 |
|---|---|
| 工作台按「小精靈模式」（沒在回應） | `ActiveMode = Sprite`；隱藏工作台、顯示小精靈 |
| 小精靈右鍵／氣泡連結「開啟工作台」 | 隱藏小精靈、顯示工作台、切到工作台模式（2026-10-24 前是兩個視窗同時開著）。回應中＝「查看中」（唯讀，頂端橫幅「小精靈回應中，結束後可在這裡繼續」，輸入、模型切換、模式切換都停用），小精靈也先隱藏，回應結束後才切到工作台模式；查看中關閉工作台＝回到小精靈、維持小精靈模式 |
| 在工作台送出訊息 | `ActiveMode = Workbench`；隱藏小精靈（正常情況下已經是工作台模式，這是保險） |
| 回應結束 | 工作台自動解除「查看中」 |
| 權限確認（任一邊回答） | 另一邊同步更新（`Resolved` 事件） |
| 小精靈被「暫時隱藏」時需要確認 | 自動顯示小精靈（不然對話迴圈會一直等） |
| 回應中 | 不能切換模式、不能換模型、「新對話」先停止目前回應 |

- 工作台的「查看中」只是工作台 ViewModel 的一個唯讀旗標（`IsViewOnly`），由協調者設定。
- 協調者的狀態（`ActiveMode`、小精靈顯示與否、工作台顯示與否）可以不開任何真的視窗就測（視窗用介面 `IWindowHost` 包起來，測試用假的）。

### 右鍵選單（7D）

開啟工作台、新對話、切換角色…（打開設定 ›「小精靈」）、設定…、暫時隱藏 30 分鐘、結束 Lumi。回應中：「開啟工作台」可用（查看中），「新對話」停用。暫時隱藏用 `TimeProvider` 計時，時間到自動顯示；托盤的「顯示小精靈」可以提前結束隱藏。

### 設定（7E）

```csharp
public enum SpriteSize { Small, Medium, Large }   // 64 / 80 / 104
public sealed record SpritePlacement(double CenterX, double CenterY);
public sealed record SpriteSettings(string SpriteId, SpriteSize Size, bool VoiceEnabled, SpritePlacement? Placement, ChatMode LastMode);
```

- `AppSettings.Sprite`、`IModelCatalog.SetSpriteAsync`（和 `SetPersonaAsync` 同樣做法）；舊設定檔沒有這個欄位時用預設（`lumi.orb`、中、語氣開、沒有位置、工作台）。
- 設定視窗左側加「小精靈」：角色清單（目前只有光球，介面已經是清單）、尺寸三選一（立即生效）、角色語氣開關、**狀態預覽**（十個狀態各一個按鈕，在設定頁內的預覽區用同一個角色即時播放；沒有支援的狀態標示它會退化成哪個）。設定視窗現在的三項分類（連線與模型、人格與主動性、一般）變成四項。
- `SpriteVoiceProvider : ISpriteVoiceProvider`：目前角色的 `Manifest.Voice`（設定關閉時回 null），取代 Core 預設的 `NoSpriteVoiceProvider`。

### 診斷（7C）

POC 的 `PerfMonitor` 與 HUD 搬進 `Lumi.App/Sprites/Diagnostics`，由 `Lumi:Developer:ShowSpriteHud` 打開（F1 切換）。補上 M0 還沒量的待機、休眠 CPU 數字，記在驗收結果。

---

## 工作項目

### 7A 共用對話會話

- [x] A1 `IChatSession`、`ChatSession`、事件型別（Core）；DI 註冊
- [x] A2 把回應迴圈從 `WorkbenchViewModel` 搬進 `ChatSession`（停止、取消收尾、未預期錯誤、`StopAndWaitAsync`）
- [x] A3 `WorkbenchViewModel` 改成觀察者：訊息列表、用量、連線狀態、錯誤列、重新產生；既有測試不改斷言
- [x] A4 對話切換與新對話走會話（回應中先停止）；模式放進會話（`TrySetMode`），取代 `IsSpriteMode` 的本地欄位（開發切換暫時還在，改成讀寫會話的模式）
- [x] A5 `IToolApprovalResponder.Resolved` 事件、`ToolCardViewModel` 訂閱（別處回答時同步成「已允許」「你已拒絕」）
- [x] A6 `ChatSessionTests`、卡片同步測試

### 7B 抽象層與光球角色

- [x] B1 `Lumi.Sprites.Abstractions` 專案：狀態、互動、manifest、`ISpriteFactory`／`ISpriteView`、`SpriteOptions`、`SpriteLimits`、版本常數、`AddSprite<T>()`、`ISpriteCatalog`
- [x] B2 `SpriteStateResolver`（退化串接、循環保護）與 `SpriteLimits.Fit`
- [x] B3 `Lumi.Sprites.Orb`：搬 `OrbControl`、包成 factory／view、改邊界（見設計）
- [x] B4 架構測試（角色專案的相依與禁用項目）
- [x] B5 測試：解析器、縮放、光球（headless）；`docs/sprite-dev-guide.md` 的介面範例同步成實際簽章

### 7C 殼：視窗與平台

- [x] C1 `IOverlayPlatform`（Core）、`NullOverlay`；`WindowsOverlay`、`MacOverlay` 搬進 `Lumi.Platform`
- [x] C2 `FrameDriver`、`SpriteShellOptions`
- [x] C3 `SpriteInteractionTracker`（純邏輯）＋ 單元測試（靠近、戳、拖曳、戳後不變拖曳）
- [x] C4 `SpriteWindow`：透明置頂、視窗尺寸計算、點擊穿透與可互動區、靠近偵測、浮現與收起動畫
- [x] C5 位置記憶與夾回螢幕（含多螢幕、DPI）、尺寸檔位、自動休眠與喚醒
- [x] C6 托盤圖示、`ShutdownMode.OnExplicitShutdown`、程式畫的圖示
- [x] C7 診斷 HUD；暫時用假狀態驅動殼（像 POC 的示範流程，開發選項）
- [ ] C8 手動驗收（Mac；Windows 見風險）

### 7D 對話整合

- [x] D1 `SpriteStateMapper` ＋ 測試（上面表格的每一列；一輪有多個 `Completed`）
- [x] D2 `SpriteBubbleViewModel` ＋ 測試（字數截斷在字形叢集邊界、完整內容條件、淡出與暫停、確認與錯誤不淡出）
- [x] D3 氣泡畫面：狀態一行、串流、完成、確認（精簡與展開、`Destructive`）、錯誤；`EmojiTextBlock`
- [x] D4 精簡輸入列：浮現、送出、停止、草稿、輸入法 Enter
- [x] D5 `AppModeCoordinator` ＋ `IWindowHost` ＋ 測試（上面表格每一列，包含隱藏中需要確認）
- [x] D6 工作台：「小精靈模式」按鈕啟用、「查看中」唯讀旗標與橫幅、「在工作台檢視」捲到指定卡片
- [x] D7 右鍵選單與暫時隱藏
- [x] D8 錯誤與缺金鑰流程（重試、開啟設定後自動重試）
- [x] D9 `ISpriteVoiceProvider` 接上目前角色（設定開關 7E 才有，這裡先讀 manifest）

### 7E 設定頁與收尾

- [x] E1 `SpriteSettings`、`IModelCatalog.SetSpriteAsync`、舊設定檔相容、測試
- [x] E2 設定 ›「小精靈」：角色清單、尺寸、語氣開關、狀態預覽（含退化說明）
- [x] E3 移除 M5 的開發用切換（`DeveloperOptions.ShowSpriteModeToggle` 與工作台那個 CheckBox）；M5 手動驗收改走真正的小精靈
- [x] E4 「換角色不改殼」自動測試：測試專案裡一個最小的假角色（只畫一個色塊、只支援 `Idle`），用 `AddSprite<T>()` 註冊並在設定選用，殼照常運作、不支援的狀態走退化
- [x] E5 更新 `docs/sprite-dev-guide.md`（介面實際簽章、`Lumi.Sprites.Orb` 當參考實作、版本紀錄）、`docs/ui-spec.md`（第 8 節拿掉草稿、補上查看中橫幅與托盤）、agent.md（`IOverlayPlatform` 的位置與參數、`ChatSession`）、主計畫狀態
- [ ] E6 手動驗收

---

## 驗收標準

自動（`dotnet test` 全過）：

- 既有所有測試不改斷言就過（7A 的重構安全網）。
- `SpriteStateResolver`：每個狀態的退化鏈都落在 `Idle`；manifest 覆寫優先；循環不會無限迴圈。
- `SpriteLimits.Fit`：長寬比與最長邊超限時被縮回，並回報警告。
- `SpriteInteractionTracker`、`SpriteStateMapper`、`SpriteBubbleViewModel`、`AppModeCoordinator`：上面各表的每一列都有測試。
- 架構測試：`Lumi.Sprites.*` 只引用 Abstractions；Abstractions 不引用 Core；角色原始碼沒有視窗、平台、計時器；P/Invoke 仍然只在 `Lumi.Platform`。
- 假角色（E4）在不改殼的情況下可用。

手動（Mac 我能陪著驗；Windows 需要你測，見風險）：

1. **POC 清單重跑**（`poc/Lumi.OrbPoc/README.md` 的 1a～9）：這次是在正式的殼上，Win、Mac 兩欄都填。重點是 2（點擊穿透）、4a／4b（待機與休眠 CPU，用 F1 的 HUD）、6（中文輸入法 Enter）、7（鍵盤焦點）。
2. 在小精靈問 15 題（沿用 M5 的清單）：每則回答在氣泡內都是兩句內；需要長內容的會出現「在工作台開啟完整內容 →」，點了工作台開啟並看得到展開的內容卡。
3. 角色狀態：送出後 `Thinking` → 呼叫工具時 `Tool` → 回覆時 `Streaming` → `Done` → 3 秒後 `Idle`；60 秒不動進入 `Sleep`，游標靠近喚醒。
4. 權限確認：寫入類工具（開啟示範工具）在氣泡裡確認，光球 `Attention`、氣泡有警示色外框；在氣泡回答後，同時開著的工作台卡片同步更新；`Destructive` 只有「允許刪除」「拒絕」；在工作台回答也同步到氣泡。
5. 模式切換：工作台按「小精靈模式」→ 工作台隱藏、小精靈出現；回應中這顆鈕停用；小精靈回應中開啟工作台是唯讀的查看中，結束後解除；在工作台送出訊息 → 小精靈隱藏。
6. 兩個視窗是同一則對話：小精靈問的問題在工作台看得到，反過來也是。
7. 托盤：「暫時隱藏 30 分鐘」後可以從托盤提前顯示；「結束 Lumi」可以結束；工作台模式下關閉工作台視窗會結束程式；小精靈模式下關閉工作台只關那個視窗。
8. 位置與尺寸：拖到新位置、重開程式，回到同一個位置；拔掉外接螢幕後重開，光球落在主螢幕右下角；64／80／104 三檔切換位置不跳動。
9. 錯誤：拔網路或用錯誤的金鑰，氣泡顯示一句說明、「重試」可用；缺金鑰時可以直接開啟設定，存好後自動重試。

## 風險

- **Windows 沒驗證過**（M0 遺留）。POC 的 `WindowsOverlay` 只在 Mac 上寫過、沒跑過。最大的未知是 `WS_EX_LAYERED`＋`WS_EX_TRANSPARENT` 與 Avalonia 透明渲染的相容性。7C 的程式在 Mac 開發，Windows 的行為要你實機驗；POC README 已經寫了備案：兩種旗標組合都不行的話，把視窗縮到只包住光球，輸入列與氣泡改成另開的小視窗。**建議在 7C 開工前先在 Windows 跑一次 POC**（`dotnet run -c Release`，照 README 清單填 Win 欄），結果會決定 `SpriteWindow` 是單一視窗還是拆成多個，越早知道越好。
- **macOS 全螢幕 App、accessory 模式**：POC 已設定 `CanJoinAllSpaces`＋`FullScreenAuxiliary`，但「全螢幕 App 上方是否可見」還沒定論；不顯示 Dock 圖示留到 M11。7C 驗收時把結果記下來。
- **7A 的重構會動到工作台最核心的一段**。緩解方式：先搬、不改行為，既有測試不改斷言；真的卡住時退回 Q2 的替代做法（小精靈借用 `WorkbenchViewModel`）。
- **氣泡與輸入列在同一個透明視窗內**，視窗大小固定預留空間，所以輸入列浮出、氣泡變長都不能讓視窗變形；這是 POC 驗證過的版面，但 POC 的氣泡沒有按鈕，加上權限確認後要重新量一次點擊穿透的可互動區。

## 實作紀錄

### 7A 共用對話會話（2026-10-21）

結果：既有 343 個 App 測試**不改任何斷言**就全過；新增 `ChatSessionTests`（20 個，Core）與 `WorkbenchSharedSessionTests`（12 個，App），並在組裝測試確認 `IChatSession` 是單例、工作台初始化走它。偏離原設計的地方：

- **`SendAsync`／`RetryAsync`／`RegenerateAsync` 回傳 `ChatRunResult`**（失敗、是否未預期、是否被停止），不是 `Task`。工作台發起的一輪要在跑完後處理金鑰提示與重送，需要結局；同樣的資料也在 `RunFinished` 事件裡。
- **`OpenConversationAsync` 收 `Conversation`，不是 id**：`IConversationStore` 沒有「依 id 取一則」，歷史清單本來就有完整物件。`NewConversationAsync` 與 `InitializeAsync` 多收「新對話」標題（它是 App 的本地化字串，Core 不認識）；三個都回傳 `Conversation`。
- **`RunFinishedArgs.Defer(Task)`**：工作台在「使用過工具的一輪被停止」時要從資料庫重讀訊息，這是非同步的，但事件處理常式不能 await。觀察者用 `Defer` 登記收尾，會話等全部做完才算結束，所以 `StopAndWaitAsync` 的保證和重構前相同（收尾做完才回來）。
- **`RunStartedArgs` 帶 `Kind`（送出／重試／重新產生）與 `UserText`**：重新產生時觀察者要移除上一輪的畫面；別處（小精靈）送出時，工作台要補上使用者的訊息。
- **權限確認的同步由工作台 ViewModel 轉發，不是每張卡片各自訂閱**：卡片訂閱單例服務會讓整個訊息列表（重讀、切換對話都會重建卡片）無法被回收。`WorkbenchViewModel` 只訂閱一次 `IToolApprovalResponder.Resolved`，依 call id 對到「這一輪」的卡片，呼叫新增的 `ToolCardViewModel.ApplyResolved`。`Resolved` 在介面上是預設實作為空的事件，不需要同步的假實作（4 個測試用的）不用改。
- **工作台對「別處發起的一輪」只更新畫面**：訊息列表、用量、連線狀態、錯誤列都照常；失敗後的金鑰提示與自動重送只在工作台自己發起時做（小精靈的版本在 7D）。
- 其他：`WorkbenchViewModel` 建構子最後加了選用的 `IChatSession? session = null`（和 `approvals`、`contextBar` 同樣做法；沒給就自己組一個，所以既有測試不用改建構方式）。`IsSpriteMode`（開發用切換）改成讀寫會話的模式，`ModeChanged` 會通知它。

### 7B 抽象層與光球角色（2026-10-21）

結果：新增 `Lumi.Sprites.Abstractions`、`Lumi.Sprites.Orb`、`Lumi.Sprites.Tests`（78 個測試，含用 Skia 真的把 10 個狀態 × 3 個檔位畫出來檢查非空白）與 `SpriteRuleTests`（架構測試 10 個）。全部 1,272 個測試通過。`Lumi.App` 還沒有引用這兩個專案（7C 才接上）。偏離或補充原設計的地方：

- **`Visual` 的大小是 `Outer`（角色框加外擴），不是角色框**：POC 的光球控制項本來就比光球大（2.3 倍）來放光暈。Avalonia 的重繪範圍以控制項的邊界為準，畫在控制項外面的東西會留下殘影，所以不讓角色畫出 `Visual`。特效外擴因此定義成「每邊，角色框對應邊長 × `effectMargin`」，`SpriteOptions` 帶 `Diameter`、`Box`、`Outer` 與 `BoxRect`（角色框在 `Outer` 裡的位置）。所有座標（`HitTest`、`BubbleAnchor`、`PokeBegin`）的原點都是 `Visual` 左上角；已寫進指南第 4、5 節。
- **光球的角色框是直徑的 1.2 倍**（中檔 96），不是 80：POC 的點擊範圍是 1.15 倍半徑，比光球本體大一圈；指南規定 `HitTest` 不能超出角色框，所以框要留這一圈。外擴 0.4 剛好容納光暈（半徑 2.1 倍光球半徑）。
- **`SpriteLimits.Fit` 回傳 `SpriteFit(Options, Scale, Warnings)`**，已經算好 `SpriteOptions`，殼直接拿去 `Create`。長寬比超出用「縮窄／縮矮」（不放大）處理，最長邊超出才整個等比例縮；外擴超出限制在 0.4。
- **`SpriteStateResolver` 永遠可以退到 `Idle`**：就算 manifest 忘了把 `Idle` 列進 `Supported` 也一樣（指南規定 `Idle` 必須支援，這是殼這一側的保險）；預設退化表放在 `SpriteStateResolver.DefaultFallback`，manifest 的 `Fallback` 逐項覆寫。
- **`HitTest` 不依賴排版**：`OrbControl.Center` 在還沒排版（`Bounds` 是空的）時用宣告的 `Width`／`Height`。殼可能在 `Visual` 排版前就問。
- **`OrbControl` 改成 internal**，對外只有 `OrbSpriteFactory`；`EyeMode`（POC 的右鍵選項）移除，固定「偶爾」。`PreferredFps`：互動中（按住、回彈、拖曳）60、待機 30（有人靠近 60，眼睛跟著游標）、休眠 12、串流／需要確認／錯誤 60、其餘 30。
- 測試專案需要 `Avalonia.Skia`（加進 `Directory.Packages.props`），因為要真的繪製才能檢查點陣圖；`Lumi.App.Tests` 仍用 `UseHeadlessDrawing`，不受影響。
- 架構測試：角色專案不引用任何 Lumi 專案（Orb 只引用 Abstractions），Core 與各函式庫不引用角色專案，角色原始碼（非註解行）不出現 `DispatcherTimer`、`DllImport`、`LibraryImport`、`Stopwatch`、`Window`、`Application`、`Dispatcher`；另外檢查 `docs/sprite-dev-guide.md` 的版本紀錄有目前的 Abstractions 版本。

### 7C 殼：視窗與平台（2026-10-21）

結果：全部 1,415 個測試通過（App 426、Tools 329、Core 229、Providers 136、Infrastructure 93、Sprites 78、Platform 67〔13 個在非對應平台跳過，Mac 上實際執行 54〕、Architecture 57）。在 Mac 上用 Release 版實際啟動過：視窗透明置頂、光球正常繪製、HUD 顯示 `macOS (AppKit via objc_msgSend)`、游標座標與「穿透中」狀態會更新。開啟方式：環境變數（或 appsettings）`Lumi__Developer__ShowSpriteShell=true`，加 `Lumi__Developer__ShowSpriteHud=true` 顯示 HUD（F1 切換）。Windows 與 Mac 的 C8 手動驗收要使用者操作。

偏離或補充原設計的地方：

- **`IOverlayPlatform` 多了 `Name` 與 `ClickThroughActive`**（HUD 要顯示）；`IsSupported` 在 `Attach` 成功後才是 true，`Attach` 失敗不丟例外。放在 `Lumi.Core/Platform`；`NullOverlay` 在 Core。實作與 `OverlayOptions`（`Layered`，`Lumi:Developer:OverlayLayered`）在 `Lumi.Platform`，P/Invoke 改用 `LibraryImport`（和既有的 Keychain 一致）。每個視窗各用一個實例（Transient）。
- **macOS 的控制代碼判斷**：`Attach` 只拿到 `nint`，沒有 POC 用的 `HandleDescriptor`，所以改問 `respondsToSelector:(setIgnoresMouseEvents:)`——能收就是 NSWindow，不能就從 NSView 取 `-window`（對 NSWindow 呼叫 `-window` 會丟 unrecognized selector）。
- **拖曳用「手動設定 Position」**：POC 的 5b（手動 vs 系統 `BeginMoveDrag`）沒有記錄結果；POC 預設是手動，使用者在 Windows 與 Mac 都用它驗證過，所以沿用，不提供系統拖曳。
- **`SpriteSettings` 與 `IModelCatalog.SetSpriteAsync` 提前到 7C 做**（原本在 7E 的 E1），因為位置記憶與尺寸檔位要存。位置存成螢幕像素整數（`SpritePlacement(int CenterX, int CenterY)`，虛擬桌面座標），不是 DIP：不同縮放的螢幕上像素座標才穩定。7E 只剩設定頁。
- **殼的純邏輯都抽出來並有單元測試**：`SpriteInteractionTracker`（靠近、離開延遲、長按戳、拖曳門檻）、`SpriteLayout`／`SpritePlacementRules`（版面、夾回螢幕、拔掉螢幕後退回主螢幕右下角）、`SpriteSleepPolicy`（自動休眠，疊在要求的狀態上，不是對話要求的）。長按用輪詢（33 ms）的時間判定，不另開計時器。
- **`SpriteWindow` 的對外介面**（7D 用）：`SetState`（要求的狀態；殼負責解析退化與疊上休眠）、`ShowBubble(text)`、`InputSubmitted`、`SpriteClicked`、`SetSizeAsync`。氣泡文字經 `EmojiTextBlock`。氣泡與輸入列在 7C 只是外殼（文字與單行輸入）；串流、確認、錯誤是 7D。
- **視窗配色用主題 token**；陰影是 `Tokens.axaml` 裡的固定資源 `LumiSpriteShadow`（架構測試只允許 Tokens.axaml 寫色碼）。
- **托盤與生命週期**：`TrayService`（顯示小精靈、開啟工作台、結束 Lumi）；`ShutdownMode.OnExplicitShutdown`，工作台關閉時結束程式。7C 階段托盤只在開發開關 `ShowSpriteShell` 打開時才建立，避免正式版出現一個還沒接上對話的小精靈；7D 之後托盤常駐，並依模式區分「關工作台＝結束」與「只關視窗」。
- **7A 補一個競態修正**：`ToolApprovalService.Resolve` 改成先發 `Resolved` 事件、再放行等待中的對話迴圈。原本的順序（先放行再通知）在測試（沒有同步內容、事件在執行緒池上）偶爾讓晚到的通知把已完成的卡片蓋回「執行中」；正式程式因為都在 UI 執行緒所以不會發生，但新順序更安全。
- **POC 已無法單獨建置**：根目錄的 `Directory.Packages.props`（中央套件管理）要求 `PackageReference` 不帶版本，POC 的 csproj 還帶版本。POC 的功能已全部搬進正式專案，不修；要跑舊 POC 用 `bin/Release` 裡既有的建置。

**耗能實測（Mac，Release，單核 %，工作台視窗同時開著；用 `ps` 的 CPU 時間差，20 秒）**：只有工作台 2.6%；殼待機（30 fps，沒人靠近）12.7%（殼約 10%）；殼休眠（>60 秒，12 fps）6.2%（殼約 3.6%）；POC 同機待機 12.0%。POC 清單 4a 的目標是待機 < 3%、4b 休眠 < 1%，**目前都沒達標，而且 POC 本身也一樣**（所以不是移植造成的退步）。可能的方向，留待確認是否值得做：休眠時降低輪詢頻率、快取每個畫面都重建的柔邊漸層筆刷（`Soft()` 一幀建好幾個 17 色標的筆刷）、待機幀率降到 20。Windows 的數字要使用者用 HUD 量。

### 7D 對話整合（2026-10-22）

結果：全部 1,523 個測試通過（App 534、Tools 329、Core 229、Providers 136、Infrastructure 93、Sprites 78、Platform 67、Architecture 57）。在 Mac 上用設定檔記著「上次是小精靈模式」啟動：只出現光球與 HUD，沒有工作台視窗。**還沒有用真的模型跑過**（需要使用者的金鑰），手動驗收見下方。偏離或補充原設計的地方：

- **「查看中」是推導出來的，不是協調者設的旗標**：`WorkbenchViewModel.IsViewOnly` ＝ 回應中、而且作用中的模式是小精靈（沒有協調者時永遠是 false，舊的開發用切換因此不受影響）。回應結束它自然消失，不需要誰去解除。工作台頂端的橫幅與輸入框停用都綁它；模型膠囊與模式按鈕本來就綁「回應中」。
- **新增 `SpriteConversationController`**（`Lumi.App/Sprites`）：把會話事件餵給 `SpriteStateMapper` 與 `SpriteBubbleViewModel`、處理輸入列的送出與停止、氣泡的重試與「開啟設定」。只處理小精靈模式的回應；工作台模式發起的回應（小精靈是隱藏的）不碰氣泡。角色仍然只從殼收到狀態。
- **缺金鑰的流程**：氣泡的錯誤帶「開啟設定」，按下去呼叫 `IApiKeyPrompt`，存好後自動重試（和工作台相同）；沒有金鑰問題的失敗（例如沒有可用的模型）只開設定視窗。小精靈發起的失敗，工作台只在錯誤列顯示，不會自己跳金鑰視窗（由發起的那一邊處理，7A 已經這樣做）。
- **啟動流程改了**：`App` 不再把工作台設成 `MainWindow`（小精靈模式下它不該出現）；啟動時先載入目前的對話（`WorkbenchViewModel.InitializeAsync` 現在只會執行一次，視窗第一次開啟時不會再載入），接上控制器與托盤，再由 `AppModeCoordinator.Start()` 顯示上次的模式。`WindowHost` 延遲取得真的視窗，所以協調者、工作台、小精靈之間沒有建構時的循環相依。
- **工作台視窗的關閉**：`WorkbenchWindow` 在 `Closing` 問協調者；工作台模式＝結束程式，小精靈模式＝取消關閉並隱藏（視窗還在，之後可以再開）。系統關機或程式結束時的關閉不攔。
- **托盤改成常駐**（7C 只在開發開關打開時才有）；7C 的開發開關 `ShowSpriteShell` 與假的示範驅動 `SpriteDemoDriver` 都移除了，現在是真的模式切換。`ShowSpriteHud` 保留，同時打開右鍵選單裡的開發項目（狀態預覽、尺寸、HUD、點擊穿透）。
- **右鍵選單**：開啟工作台、新對話（回應中停用）、設定…、暫時隱藏 30 分鐘、結束 Lumi。**「切換角色…」留到 7E**（設定頁有角色清單才有意義）。
- **氣泡版面**：預留高度從 180 加到 260，放得下展開的權限確認；氣泡有按鈕（確認、錯誤、完整內容連結）時才吃滑鼠，點擊穿透的可互動區包含氣泡；沒有按鈕的氣泡維持穿透。視窗隱藏時（暫時隱藏、工作台模式）暫停輪詢與繪圖。
- **新增測試輔助 `ManualTimeProvider`**：連 `Task.Delay(…, TimeProvider)` 的計時器也跟著假時間走（`FakeTimeProvider` 只有時鐘，計時器仍是真的），測「暫時隱藏 30 分鐘」用。
- **開發用的模式切換（工作台的 `ShowSpriteModeToggle`）還在**，留到 7E 的 E3 移除並把那些測試改成直接用會話；有協調者時在工作台送出本來就會切回工作台模式，所以它在正式流程裡已經沒有意義。

**7D 的手動驗收（要你的金鑰）**：照計畫驗收標準的第 2～7 項：小精靈問 15 題（沿用 M5 清單，每則在氣泡內兩句內、需要長內容時出現「在工作台開啟完整內容 →」並點開看得到展開的內容卡）；狀態 Thinking → Tool → Streaming → Done → 3 秒 Idle、60 秒 Sleep；寫入類工具在氣泡確認（Attention、警示色外框），在氣泡與工作台回答都同步，`Destructive` 只有「允許刪除」「拒絕」；模式切換與查看中；兩個視窗是同一則對話；托盤與關閉行為；錯誤與缺金鑰。

### 7E 設定頁與收尾（2026-10-22）

結果：全部測試通過（App 553，其餘不變；全部約 1,540 個）。偏離或補充原設計的地方：

- **設定視窗的分類順序是「連線與模型、人格與主動性、一般、小精靈」**（小精靈放在最後，索引 3），沒有插在「一般」前面：這樣既有的索引（人格 1、一般 2）和測試都不用動。`SettingsSection.Sprite` 導向它；右鍵選單的「切換角色…」開啟它（7D 先留下的缺口在這裡補上）。
- **小精靈分頁選了立即生效，沒有「儲存」鈕**（`SpriteSettingsViewModel`，和「一般」同樣做法）：角色、尺寸、語氣開關選了就寫進設定；存不起來（例如設定檔版本太新）時顯示錯誤並回到實際生效的值；設定在別處變了畫面也跟著變。所以它沒有未儲存的變更，不影響關視窗前的詢問。
- **小精靈視窗聽設定變更**：`SpriteWindow` 訂閱 `IModelCatalog.Changed`，角色 id 或尺寸和已經建好的不同才重建，位置以角色中心為準不跳動；設定的其他變更（主題、人設、語氣…）不會動到角色。`SetSizeAsync` 因此也只是寫設定再呼叫同一個「有變才重建」。
- **預覽區用殼的規則播放**：`SpritePreviewHost`（`FrameDriver`、`SpriteLimits.Fit`、`SpriteStateResolver`）放在 `Lumi.App/Sprites`，設定視窗的 code-behind 依 `PreviewFactory`／`PreviewState` 更新；只在「小精靈」分頁可見時才驅動畫面。狀態清單十列，沒有專屬表現的狀態標示退化成哪個（同一個解析器，所以預覽與實際行為一致）。預覽固定用「中」檔尺寸。
- **E3 移除開發用切換**：`ShowSpriteModeToggle`、工作台的 CheckBox、`IsSpriteMode`、`CanChangeMode` 與兩條相關字串都拿掉，`DeveloperOptions` 只剩 `ShowSpriteHud`。相關測試改成直接用會話的 `TrySetMode`（原本驗證小精靈 prompt 的測試內容不變）。
- **E4「換角色不改殼」**：`BlockSpriteFactory`（測試專案，只畫色塊、只支援 `Idle`）用 `AddSprite<T>()` 註冊並在設定選用；測試涵蓋殼的視窗與版面、狀態退化（角色只收到 `Idle`）、氣泡與輸入列、尺寸檔位、設定即時換角色（中心不動）、設定頁的預覽與退化說明。整個過程沒有改到殼的任何程式。
- **文件**：`docs/sprite-dev-guide.md` 新增第 9 節「驗證你的角色」（版本紀錄順延為第 11 節）、`docs/ui-spec.md` 第 7 節補上「小精靈」分頁、`agent.md` 補上 `IOverlayPlatform`（放 Core、只吃控制代碼）、`IChatSession`、`IAppModeCoordinator` 與殼的純邏輯說明。

**E6 手動驗收（要你操作）**：
1. 設定 ›「小精靈」：換尺寸三檔立即生效且光球位置不跳動；語氣開關；點十個狀態，預覽區即時播放；關掉設定視窗預覽就停。
2. 右鍵 ›「切換角色…」開啟設定的小精靈分頁。
3. 重開程式回到上次的模式與位置；拔掉外接螢幕後重開光球落在主螢幕右下角。
4. 7C 的 C8（POC 清單 1a～9 在正式的殼上重跑，Win、Mac 兩欄）與 7D 的對話驗收（見上面 7D 的清單）。
5. 耗能：用 HUD 量 Windows 的待機與休眠數字（Mac 的數字見 7C 的紀錄）。
