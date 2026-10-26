# M20 設定頁補強、App 圖示與單檔發佈

分支 `m20-settings-release`。使用者 2026-10-26 確認：macOS 做 `.app`（單檔包進 zip）、要 GitHub Actions 自動發佈、Windows 只做 x64、mac 只做 Apple Silicon。另外直接拿掉設定視窗左上角的「Lumi 設定／個人偏好與 AI 連線」標題區。

## 工作拆分

| 包 | 內容 | Wave | 負責 |
|---|---|---|---|
| Phase 0 | 字串、介面簽章、ViewModel 骨架、`SettingsCategory` 列舉 | — | 協調者（已完成） |
| 20A | 提醒的真正刪除（store 與 service） | 1 | 子代理 |
| 20C | 資料與隱私：`DataLocationProvider`、`DataPrivacyViewModel`、`DataPrivacyPanel` | 1 | 子代理 |
| 20D | 設定首頁 `HomePanel`、工具頁分類標題樣式 | 1 | 子代理 |
| 20E | 光球圖示產生器與圖示檔、托盤圖示、macOS Dock 圖示 | 1 | 子代理 |
| 20B | 提醒排程頁：`RemindersViewModel`、`RemindersPanel` | 2（20A 合併後） | 子代理 |
| 20F | 發佈腳本、macOS `.app` 打包、GitHub Actions | 2（20E 合併後） | 子代理 |
| 整合 | `SettingsWindow.axaml`、選單、測試修正、單檔發佈屬性、Playwright 單檔處理、文件 | — | 協調者 |

通則（所有包）：在自己的 worktree 工作，不 commit、不改 `plan/`、不碰清單以外的檔案；遇到計畫沒寫到或和程式現況不符就停下來回報，不要猜。UI 只用既有 token 與樣式 class（`muted`、`divider`、`badge`、`primary`／`secondary`／`tertiary` 按鈕、`LumiAccentSoftBrush` 等），不自己挑顏色。所有使用者看得到的文字已在 Phase 0 加進 `Strings.resx`／`Strings.cs`，**直接用 `Strings.Xxx`，不要新增字串**；真的缺字串就停下來回報。建置零警告（`TreatWarningsAsErrors`）。

回報格式：改了哪些檔案、新增測試數量與結果、和計畫不同的地方與原因、沒做完或卡住的項目。

---

### 包 20A：提醒的真正刪除（Wave 1）

- **目標**：`IScheduledItemStore.DeleteAsync`、`DeleteFinishedAsync` 與 `ReminderService.DeleteAsync`、`DeleteFinishedAsync` 真的能刪（目前是 `throw new NotImplementedException()` 的骨架）。
- **先讀**：`src/Lumi.Core/Reminders/IScheduledItemStore.cs`、`IReminderService.cs`、`InMemoryScheduledItemStore.cs`、`ReminderService.cs`（仿照 `CancelAsync` 的鎖與 `SignalChanged()`）、`src/Lumi.Infrastructure/Persistence/SqliteScheduledItemStore.cs`（仿照 `PurgeFinishedAsync`）、`tests/Lumi.Core.Tests/Reminders/ScheduledItemStoreTests.cs`、`ReminderServiceFireTests.cs`、`tests/Lumi.Core.Tests/TestSupport/ReminderTestSupport.cs`、`tests/Lumi.Infrastructure.Tests/SqliteScheduledItemStoreTests.cs`。
- **可修改**：上列四個 `src` 檔，以及 `tests/Lumi.Core.Tests/Reminders/ScheduledItemStoreTests.cs`、`tests/Lumi.Core.Tests/Reminders/ReminderServiceDeleteTests.cs`（新增）、`tests/Lumi.Infrastructure.Tests/SqliteScheduledItemStoreTests.cs`。
- **步驟**：
  1. `InMemoryScheduledItemStore.DeleteAsync(id)`：在 `lock (_lock)` 內 `_items.Remove(id)`，回傳是否移除。`DeleteFinishedAsync`：同一把鎖內移除所有 `IsFinished(status)` 的項目，回傳筆數。
  2. `SqliteScheduledItemStore.DeleteAsync(id)`：`DELETE FROM scheduled_items WHERE id = $id;`，回傳 `ExecuteNonQueryAsync` > 0。`DeleteFinishedAsync`：`DELETE ... WHERE status IN (Completed, Cancelled, Missed)`（寫法同 `PurgeFinishedAsync`，沒有時間條件），回傳筆數。
  3. `ReminderService.DeleteAsync(string id, CancellationToken ct)`：`await _gate.WaitAsync(ct)`、`store.DeleteAsync(id, ct)`、finally 放鎖；刪成功才呼叫 `SignalChanged()`（鎖外，仿 `CancelAsync`）；回傳是否刪除。`DeleteFinishedAsync(ct)`：同樣，筆數 > 0 才 `SignalChanged()`。
  4. 骨架裡參數名是 `reminderId` 的話改成 `id`，與介面一致。
- **測試**（每個斷言一個 `[Fact]`）：
  - store（InMemory 與 Sqlite 各一組，仿既有測試的建立方式）：刪除存在的 id 回 true 且 `GetAsync` 回 null；刪除不存在的回 false；`DeleteFinishedAsync` 只刪 Completed／Cancelled／Missed、保留 Pending，回傳筆數正確；刪掉待觸發項目後 `GetNextFireAtAsync` 不再是它的時間。
  - service（`ReminderServiceDeleteTests`，用 `ManualClock`、`RecordingNotificationSink`、`InMemoryScheduledItemStore`）：刪除待觸發項目後再 `ProcessDueAsync` 不會通知；刪除成功後 `ChangeToken.IsCancellationRequested`（先取 token 再刪）；刪除不存在的 id 回 false 且 token 沒被取消；`DeleteFinishedAsync` 只清已結束的，筆數 > 0 才取消 token。
- **完成檢查**：`dotnet build Lumi.sln` 零警告；`dotnet test tests/Lumi.Core.Tests --filter Reminder`、`dotnet test tests/Lumi.Infrastructure.Tests --filter ScheduledItem` 全過。
- **不要做**：不改介面簽章、不動 `ReminderHostedService`、不加 migration。

---

### 包 20C：資料與隱私（Wave 1）

- **目標**：`DataLocationProvider` 回傳實際位置；`DataPrivacyViewModel` 與 `DataPrivacyPanel.axaml` 完成（尚未掛進 `SettingsWindow`，由協調者整合）。
- **先讀**：`src/Lumi.App/ViewModels/DataLocation.cs`、`DataPrivacyViewModel.cs`（骨架，保留 `NavigateRequested` 與 `GoToCommand`、建構子簽章）、`src/Lumi.App/Services/DataLocationProvider.cs`（骨架）、`src/Lumi.Infrastructure/LumiPaths.cs`、`Persistence/PersistenceOptions.cs`、`Attachments/AttachmentOptions.cs`、`Settings/ModelCatalogOptions.cs`、`src/Lumi.Platform/SecretStoreOptions.cs`、`src/Lumi.Browser/BrowserOptions.cs`、`src/Lumi.Core/Notes/INoteFolderProvider.cs`、`src/Lumi.Sprites.Assets/SpriteLibrary.cs`（`AssetSpriteOptions`）、`src/Lumi.App/AppServiceCollectionExtensions.cs` 與 `Program.cs`（看這些 Options 怎麼註冊）、`src/Lumi.App/Views/MemoryPanel.axaml`（面板版型、樣式寫法）、`ToolsPanel.axaml`、`src/Lumi.App/ViewModels/BrowserSettingsViewModel.cs`（`IFolderOpener`、`IClipboardService` 的用法與訊息處理）。
- **可修改／新增**：`src/Lumi.App/Services/DataLocationProvider.cs`、`src/Lumi.App/ViewModels/DataPrivacyViewModel.cs`、`src/Lumi.App/Views/DataPrivacyPanel.axaml`、`DataPrivacyPanel.axaml.cs`、`tests/Lumi.App.Tests/DataLocationProviderTests.cs`、`tests/Lumi.App.Tests/DataPrivacyViewModelTests.cs`、`tests/Lumi.App.Tests/Headless/DataPrivacyPanelTests.cs`。
- **步驟**：
  1. `DataLocationProvider`：建構子注入 `IOptions<PersistenceOptions>`、`IOptions<AttachmentOptions>`、`IOptions<ModelCatalogOptions>`、`IOptions<SecretStoreOptions>`、`IOptions<BrowserOptions>`、`IOptions<AssetSpriteOptions>`、`INoteFolderProvider`（若某個 Options 沒有註冊成 `IOptions`，改成讀它實際被設定的來源：瀏覽器是 `Path.Combine(LumiPaths.DataDirectory, "browser")`、角色素材是 `Path.Combine(LumiPaths.DataDirectory, "sprites")`，並在回報裡說明）。`GetLocations()` 依序回傳九筆，Id 與資料如下（名稱與說明用 `Strings`）：
     | Id | Name／Description | Path | IsFile |
     |---|---|---|---|
     | `data` | `Privacy_LocData`／`Privacy_LocDataDesc` | `LumiPaths.DataDirectory` | false |
     | `database` | `Privacy_LocDatabase`／`Privacy_LocDatabaseDesc` | `PersistenceOptions.ResolveDatabasePath()` | true |
     | `attachments` | `Privacy_LocAttachments`／`...Desc` | `AttachmentOptions.ResolveDirectory()` | false |
     | `settings` | `Privacy_LocSettings`／`...Desc` | `ModelCatalogOptions.ResolveSettingsPath()` | true |
     | `secrets` | Windows：`Privacy_LocSecretsWindows`／`Privacy_LocSecretsWindowsDesc`，路徑 `SecretStoreOptions.ResolveDpapiFilePath()`，IsFile true；其他系統：`Privacy_LocSecretsMac`／`string.Format(Privacy_LocSecretsMacDesc, SecretStoreOptions.KeychainService)`，路徑 null，IsFile false | | |
     | `logs` | `Privacy_LocLogs`／`...Desc` | `LumiPaths.LogDirectory` | false |
     | `browser` | `Privacy_LocBrowser`／`...Desc` | 瀏覽器設定檔根資料夾 | false |
     | `sprites` | `Privacy_LocSprites`／`...Desc` | 角色素材資料夾 | false |
     | `notes` | `Privacy_LocNotes`／`...Desc` | `INoteFolderProvider.Folder` | false |
     判斷 Windows 用 `OperatingSystem.IsWindows()`。
  2. `DataPrivacyViewModel`（保留既有建構子簽章與 `NavigateRequested`、`GoToCommand`）：
     - `IReadOnlyList<DataLocationItemViewModel> Locations`（建構時由 `locations.GetLocations()` 建立）。`DataLocationItemViewModel`（同檔，`ObservableObject`）：`Id`、`Name`、`Description`、`PathText`（Path 是 null 時為空字串）、`HasPath`、`[ObservableProperty] string? SizeText`、`bool NotCreated`（有 Path 但檔案或資料夾不存在）、`OpenCommand`、`CopyCommand`（沒有 Path 時兩者 `CanExecute` 為 false）。
     - `OpenButtonText`：`OperatingSystem.IsMacOS()` ? `Strings.Privacy_OpenMac` : `Strings.Privacy_OpenWindows`。
     - `[ObservableProperty] string? Message`、`bool MessageIsError`、`HasMessage`。
     - Open：`folders.OpenAsync(IsFile ? Path.GetDirectoryName(path)! : path)`；失敗設 `Message = Strings.Privacy_OpenFailed`、`MessageIsError = true`。Copy：`clipboard.SetTextAsync(path)`，成功設 `Message = Strings.Privacy_Copied`。
     - `public async Task RefreshSizesAsync()`：對每個有 Path 且存在的項目 `await Task.Run(() => ComputeSize(path))`，結果以 `FormatSize` 格式化後設 `SizeText = string.Format(Strings.Privacy_SizeFormat, text)`；不存在的設 `NotCreated = true`。`ComputeSize` 檔案取 `FileInfo.Length`；資料夾遞迴 `EnumerateFiles`（`EnumerationOptions { RecurseSubdirectories = true, IgnoreInaccessible = true }`）加總。`static string FormatSize(long bytes)`：< 1024 → `"{n} B"`；< 1 MB → `"{x:0.#} KB"`；< 1 GB → `"{x:0.#} MB"`；否則 `"{x:0.#} GB"`（`CultureInfo.InvariantCulture`）。
  3. `DataPrivacyPanel.axaml`（`UserControl`，`x:DataType="vm:DataPrivacyViewModel"`，版型仿 `ToolsPanel`：`ScrollViewer` → `ctl:FractionWidthLimiter MaxPixels="640"` → `StackPanel`）。由上到下：
     - 區段標題（`TextBlock Classes="section-title"`）`Privacy_LocationsTitle`，下面一行 `muted` 的 `Privacy_LocationsNote`。
     - 訊息 `TextBlock`（`Message`，`Classes="muted"`，`MessageIsError` 時加 `Classes.error`），`IsVisible="{Binding HasMessage}"`。
     - `ItemsControl` 綁 `Locations`：每列一個 `Border`（下邊 1px `LumiBorderBrush`、Padding `0,12`，樣式寫在 UserControl.Styles，仿 `MemoryPanel` 的 `Border.memory-row`）：左欄 `StackPanel`：名稱（14／600）、說明（`muted`、換行）、路徑（`SelectableTextBlock`，等寬字型 `LumiMonoFont`，11、`LumiTextMutedBrush`、換行；`IsVisible="{Binding HasPath}"`）、大小與「尚未建立」（`muted`，`SizeText` 有值才顯示；`NotCreated` 時顯示 `Strings.Privacy_NotCreated`）；右欄兩個按鈕（`Classes="secondary"` 開啟、`Classes="tertiary"` 複製，`MinHeight="{StaticResource LumiControlHeight}"`、`Padding="12,8"`，開啟鈕文字用 `$parent[UserControl].((vm:DataPrivacyViewModel)DataContext).OpenButtonText`）。窄視窗（`Window.narrow`）按鈕移到說明下方，寫法仿 `ToolsPanel` 的 `StackPanel.tool-confirm`。
     - 區段標題 `Privacy_SendTitle`，下面五行 `Privacy_SendModel`、`Privacy_SendBrowser`、`Privacy_SendSpeech`、`Privacy_SendFiles`、`Privacy_SendKeys`（每行一個 `TextBlock`，`TextWrapping="Wrap"`，行間距 8，前面不加項目符號）。
     - 區段標題 `Privacy_ControlTitle`，下面六列（寫死在 XAML，每列名稱 14／600、說明 `muted`、右側 `Classes="secondary"` 按鈕 `Privacy_GoSettings`，`Command="{Binding GoToCommand}"`，`CommandParameter` 依序：`Memory`、`Tools`、`Network`、`Reminders`、`Speech`；第一列「刪除對話」沒有按鈕）：`Privacy_CtlHistory`／`Desc`、`Privacy_CtlMemory`、`Privacy_CtlTools`、`Privacy_CtlBrowser`、`Privacy_CtlReminders`、`Privacy_CtlSpeech`。
     - 每個按鈕與可互動控制項都要有 `AutomationProperties.Name`；各區段與列給 `x:Name`（`PrivacyLocationList`、`PrivacySendSection`、`PrivacyControlSection`）。
  4. `DataPrivacyPanel.axaml.cs`：`InitializeComponent()`，`AttachedToVisualTree` 時 `_ = (DataContext as DataPrivacyViewModel)?.RefreshSizesAsync();`（仿 `MemoryPanel.axaml.cs` 的寫法與註解風格）。
- **測試**：
  - `DataLocationProviderTests`：用臨時資料夾組 Options，斷言九筆的 Id 順序、各 Path 等於 Options 的解析結果、`database`／`settings` 的 `IsFile` 為 true、`notes` 取自 `INoteFolderProvider`；非 Windows 時 `secrets.Path` 為 null 且說明含 `io.github.neillog.lumi`。
  - `DataPrivacyViewModelTests`（用 `FakeFolderOpener`、`FakeClipboardService`、`FakeDataLocationProvider`，見 `tests/Lumi.App.Tests/TestSupport`）：檔案項目的開啟傳入的是父資料夾、資料夾項目傳入自己；複製把路徑放進剪貼簿並顯示 `Privacy_Copied`；開啟失敗顯示 `Privacy_OpenFailed` 且 `MessageIsError`；`RefreshSizesAsync` 對臨時資料夾裡兩個檔案算出正確加總文字、不存在的路徑 `NotCreated`；`FormatSize` 邊界（0、1023、1024、1536、5 MB）；`GoToCommand` 以 `"Memory"` 觸發 `NavigateRequested(SettingsSection.Memory)`、亂字串不觸發。
  - `Headless/DataPrivacyPanelTests`（仿 `tests/Lumi.App.Tests/Headless/MemoryPanelTests.cs` 的做法）：放進一個視窗後，`PrivacyLocationList` 有與 Locations 同數量的列、六個「怎麼控制」區段存在。
- **完成檢查**：`dotnet build Lumi.sln` 零警告；`dotnet test tests/Lumi.App.Tests --filter "DataLocationProviderTests|DataPrivacyViewModelTests|DataPrivacyPanelTests"` 全過。
- **不要做**：不改 `SettingsWindow.axaml`、`SettingsViewModel.cs`、`Strings.*`、DI 檔；不新增任何破壞性功能（沒有「刪除全部」）。

---

### 包 20D：設定首頁與工具頁分類標題（Wave 1）

- **目標**：`HomePanel`（常用設定連結）完成；工具頁的分類標題明顯比工具名稱突出。
- **先讀**：`src/Lumi.App/Views/SettingsWindow.axaml`（第 109–160 行 `ListBox.menu` 的 hover／選取樣式寫法；左側選單各項的圖示）、`src/Lumi.App/ViewModels/SettingsViewModel.cs`（`GoToCommand`，參數是 `SettingsSection` 名稱）、`src/Lumi.App/Views/ToolsPanel.axaml`、`MemoryPanel.axaml`。
- **可修改／新增**：`src/Lumi.App/Views/HomePanel.axaml`、`HomePanel.axaml.cs`、`src/Lumi.App/Views/ToolsPanel.axaml`、`tests/Lumi.App.Tests/Headless/HomePanelTests.cs`。
- **步驟**：
  1. `HomePanel.axaml`（`UserControl`，`x:DataType="vm:SettingsViewModel"`；`ScrollViewer` → `ctl:FractionWidthLimiter MaxPixels="640"` → `ItemsControl x:Name="HomeLinkList"`，不用資料模板，直接寫 11 個 `Button`，`AutomationProperties.Name` 為標題）。每個 `Button Classes="home-link"`：`Command="{Binding GoToCommand}"`、`CommandParameter="<SettingsSection 名稱>"`、內容 `Grid ColumnDefinitions="Auto,*,Auto"`：圖示 `ctl:LumiIcon Size="18"`（右邊距 12）、`StackPanel`（標題 `TextBlock` 14／600 `LumiTextBrush`；說明 `TextBlock Classes="muted"`，`TextWrapping="Wrap"`）、右側 `LumiIconChevronRight` Size 16。樣式（寫在 `UserControl.Styles`）：`Button.home-link` 的 `Background=Transparent`、`HorizontalAlignment=Stretch`、`HorizontalContentAlignment=Stretch`、`Padding="12,10"`、`CornerRadius="{StaticResource LumiRadiusInput}"`、`BorderThickness=0`；`Button.home-link:pointerover /template/ ContentPresenter#PART_ContentPresenter` 與 `:pressed` 的 `Background` 設 `LumiAccentSoftBrush`（寫法仿 `SettingsWindow.axaml` 第 120–125 行）。連結列間距 4。
  2. 11 個連結（順序、參數、圖示、標題、說明）：
     | 參數 | 圖示 | 標題 | 說明 |
     |---|---|---|---|
     | `Connections` | `LumiIconDiamond` | `Settings_CategoryConnections` | `Settings_CategoryConnectionsSub` |
     | `Models` | `LumiIconSettings` | `Home_LinkModels` | `Home_LinkModelsSub` |
     | `Persona` | `LumiIconUser` | `Settings_CategoryPersona` | `Settings_CategoryPersonaSub` |
     | `Memory` | `LumiIconBrain` | `Settings_CategoryMemory` | `Settings_CategoryMemorySub` |
     | `Reminders` | `LumiIconBell` | `Settings_CategoryReminders` | `Settings_CategoryRemindersSub` |
     | `Tools` | `LumiIconWrench` | `Tools_MenuTitle` | `Tools_MenuSubtitle` |
     | `General` | `LumiIconSun` | `Settings_CategoryGeneral` | `Settings_CategoryGeneralSub` |
     | `Sprite` | `LumiIconSparkle` | `Settings_CategorySprite` | `Settings_CategorySpriteSub` |
     | `Speech` | `LumiIconMic` | `Settings_CategorySpeech` | `Settings_CategorySpeechSub` |
     | `Network` | `LumiIconGlobe` | `Settings_CategoryNetwork` | `Settings_CategoryNetworkSub` |
     | `Privacy` | `LumiIconShield` | `Settings_CategoryPrivacy` | `Settings_CategoryPrivacySub` |
  3. `HomePanel.axaml.cs`：只有 `InitializeComponent()`（仿 `MemoryPanel.axaml.cs`）。
  4. `ToolsPanel.axaml`：(a) 把 `TextBlock.group-title` 改成 `FontSize="16"`、`FontWeight=Bold`、`Foreground="{DynamicResource LumiAccentTextBrush}"`；(b) 群組標題那一列（現在的 `Grid ColumnDefinitions="Auto,*,Auto"`）外面包一個 `Border Classes="group-header"`：`Background="{DynamicResource LumiAccentSoftBrush}"`、`CornerRadius="{StaticResource LumiRadiusInput}"`、`Padding="12,8"`；(c) 群組之間的上方留白：群組 `StackPanel` 加 `Margin="0,8,0,0"`；(d) 該群組的工具清單（`ItemsControl ItemsSource="{Binding Items}"`）加 `Margin="12,0,0,0"` 縮排一階。工具名稱樣式（`item-title`）不改。
- **測試**（`HomePanelTests`，仿 `tests/Lumi.App.Tests/Headless/MemoryPanelTests.cs`，用 `SettingsHost`）：面板有 11 個 `home-link` 按鈕；點「記憶」連結（`Memory`）後 `host.Settings.ShowMemory` 為 true；點「模型設定」後 `ShowConnectionsAndModels` 為 true 且 `ShowModelsTab` 為 true；每個按鈕的 `AutomationProperties.Name` 非空。另外既有的 `ToolsPanel` 相關測試全部仍通過。
- **完成檢查**：`dotnet build Lumi.sln` 零警告；`dotnet test tests/Lumi.App.Tests --filter "HomePanelTests|Tools"` 全過。
- **不要做**：不改 `SettingsWindow.axaml`、`SettingsViewModel.cs`、`Strings.*`；首頁不顯示即時狀態。

---

### 包 20E：光球圖示（Wave 1）

- **目標**：產出 app 圖示檔（`.icns`、`.ico`、PNG），托盤圖示改用同一組圖，macOS 執行期設定 Dock 圖示。
- **先讀**：`src/Lumi.App/Sprites/TrayIconFactory.cs`（現有光球配色：白 FFFFFF 0%→奶油 FFE9C7 30%→琥珀 F5B76A 70%→焦橙 C9783A 100%，圓心偏左上 35%／30%）、`src/Lumi.App/Services/TrayService.cs`、`src/Lumi.App/App.axaml.cs`、`src/Lumi.Platform/Interop/Mac/ObjC.cs`（`objc_msgSend` 各簽章）與 `Overlay/MacOverlay.cs`（怎麼取得 `NSApplication`、呼叫 selector）、`tests/Lumi.App.Tests/Sprites/TrayIconTests.cs`、`poc/Lumi.OrbPoc/OrbControl.cs`（光球的原始畫法，參考）。
- **可修改／新增**：`tools/Lumi.IconGen/*`（新）、`src/Lumi.App/Assets/Icons/*`（新，產出物）、`src/Lumi.App/Sprites/TrayIconFactory.cs`、`src/Lumi.Platform/Shell/MacDockIcon.cs`（新）、`src/Lumi.App/App.axaml.cs`、`tests/Lumi.App.Tests/Sprites/TrayIconTests.cs`、`tests/Lumi.App.Tests/AppIconAssetTests.cs`（新）、`tests/Lumi.Platform.Tests/MacDockIconTests.cs`（新）。
- **步驟**：
  1. 新增 `tools/Lumi.IconGen/Lumi.IconGen.csproj`（`OutputType` Exe，繼承 `Directory.Build.props`；`PackageReference Include="Avalonia.Skia"`，直接使用 `SkiaSharp`；**不加進 `Lumi.sln`**，像 `poc/` 一樣獨立）與 `Program.cs`。用法 `dotnet run --project tools/Lumi.IconGen -- <輸出資料夾>`。
  2. 繪製（SkiaSharp，`SKSurface` + `SKCanvas`，全部以 1024 為基準座標，其他尺寸等比縮放重畫，不要縮圖）：
     - **光球 `DrawOrb(canvas, cx, cy, r)`**：先畫外暈：`SKPaint` 圓、半徑 `r*1.35`、`SKShader.CreateRadialGradient` 中心 `(cx,cy)` 半徑 `r*1.35`，顏色 `#F5B76A` alpha 0x90 → `#F5B76A` alpha 0，位置 `[0.55, 1]`。再畫球體：圓半徑 `r`，`SKShader.CreateRadialGradient` 中心 `(cx - 0.15*r, cy - 0.2*r)`、半徑 `1.15*r`，顏色與位置 `[#FFFFFF, #FFE9C7, #F5B76A, #C9783A]`／`[0, 0.3, 0.7, 1]`。最後畫高光：圓半徑 `0.22*r`、圓心 `(cx - 0.3*r, cy - 0.38*r)`，`SKShader.CreateRadialGradient` 白色 alpha 0xC0 → 白色 alpha 0。
     - **`lumi-1024.png`（macOS 用）**：1024×1024 透明畫布；圓角方塊 `SKRect(100,100,924,924)`、圓角半徑 185，`SKShader.CreateLinearGradient` 由上 `(0,100)` 到下 `(0,924)`：`#34463F` → `#1C2622`；方塊內畫光球，圓心 `(512,512)`、`r = 250`。方塊先裁切（`canvas.ClipRoundRect`）再畫光球外暈，避免暈光溢出方塊。
     - **`lumi-256.png`（視窗圖示）**：256×256 透明底，不含方塊，只有光球：`DrawOrb(cx=128, cy=128, r=88)`。
     - **`lumi-tray-64.png`（托盤）**：64×64 透明底，光球 `r=27`、圓心 `(32,32)`；**這個尺寸不要畫外暈**（`DrawOrb` 加 `withGlow` 參數），避免 16px 時糊成一團。
     - **`Lumi.ico`**：Windows，內含 16、24、32、48、64、128、256 七個尺寸的光球（無方塊）；各尺寸分別用 `DrawOrb(r = 0.34*size)`，`size <= 32` 時 `withGlow=false`，其餘 `true`。寫 ICO 格式自己組：ICONDIR（reserved 0、type 1、count 7）＋每個 ICONDIRENTRY（寬高 256 寫 0、planes 1、bitCount 32、bytesInRes、imageOffset）＋ PNG 資料（PNG 壓縮的 ICO，Vista 以後支援）。
     - **`Lumi.icns`**：僅 macOS：把 `lumi-1024.png`（方塊版）依序重畫成 `icon_16x16.png`、`icon_16x16@2x.png`、`icon_32x32.png`、`icon_32x32@2x.png`、`icon_128x128.png`、`icon_128x128@2x.png`、`icon_256x256.png`、`icon_256x256@2x.png`、`icon_512x512.png`、`icon_512x512@2x.png`（方塊版、各尺寸重畫）放進暫存 `Lumi.iconset` 資料夾，執行 `iconutil -c icns <iconset> -o <輸出>/Lumi.icns`；不是 macOS（找不到 `iconutil`）就略過並印出提示。
  3. 執行工具，輸出到 `src/Lumi.App/Assets/Icons/`，**這些檔案要在 worktree 中產生出來**（`lumi-1024.png`、`lumi-256.png`、`lumi-tray-64.png`、`Lumi.ico`、`Lumi.icns`）；用 `Read` 工具看 PNG 確認是光球、沒有破圖、方塊有圓角、深色底上光球明亮。
  4. `TrayIconFactory.CreateOrbIcon(int size = 64)`：改成讀 `avares://Lumi.App/Assets/Icons/lumi-tray-64.png`（`AssetLoader.Open` → `new Bitmap` → `new WindowIcon(bitmap)`）；保留方法簽章（`size` 參數保留但不使用，XML 註解說明「圖檔固定 64px，系統會自行縮放」）。更新類別註解（不再是「程式畫的簡單光球」）。
  5. `src/Lumi.Platform/Shell/MacDockIcon.cs`：`[SupportedOSPlatform("macos")] public static class MacDockIcon`，`public static bool TryApply(byte[] png)`：呼叫 ObjC（`NSData dataWithBytes:length:` → `NSImage alloc/initWithData:` → `[NSApplication sharedApplication] setApplicationIconImage:`）；任何失敗回 false，不丟例外。需要的 `objc_msgSend` 簽章如 `ObjC.cs` 沒有就在 `ObjC.cs` 加（只加，不改既有）；`ObjC.cs` 是 internal，`MacDockIcon` 同組件。**必須在 UI 執行緒呼叫。**
  6. `App.axaml.cs`：在 `OnFrameworkInitializationCompleted` 裡（桌面主視窗建立之後）加：`if (OperatingSystem.IsMacOS())` 讀 `avares://Lumi.App/Assets/Icons/lumi-1024.png` 的位元組，呼叫 `MacDockIcon.TryApply`，失敗不處理（最多 `logger` 記一筆 Warning，若該處沒有 logger 就不記）。
- **測試**：
  - `AppIconAssetTests`（`[AvaloniaFact]`，用 `AssetLoader.Open` 讀 `avares://Lumi.App/Assets/Icons/...`）：`lumi-1024.png` 解出來是 1024×1024、`lumi-256.png` 是 256×256、`lumi-tray-64.png` 是 64×64，三者角落像素（0,0）alpha 為 0（透明底）；`lumi-1024.png` 中心像素是亮色（R > 200）。另外在測試裡直接用 `File.ReadAllBytes` 讀 `Lumi.ico`（路徑由 `[CallerFilePath]` 往上推到 `src/Lumi.App/Assets/Icons/Lumi.ico`）：前 6 位元組是 `00 00 01 00 07 00`，七個尺寸的寬度依序是 16,24,32,48,64,128,0（256 寫 0）。
  - `TrayIconTests` 既有測試保持通過。
  - `MacDockIconTests`（`[SkippableFact]`，非 macOS 略過）：傳入亂位元組 `TryApply` 回 false 且不丟例外（不要呼叫會真的改 Dock 的成功路徑）。
- **完成檢查**：`dotnet build Lumi.sln` 零警告；`dotnet test tests/Lumi.App.Tests --filter "AppIconAssetTests|TrayIconTests"`、`dotnet test tests/Lumi.Platform.Tests --filter MacDockIcon` 全過。
- **不要做**：不改 `Lumi.App.csproj`（協調者處理 `ApplicationIcon`；`AvaloniaResource` 已加好）、不改 `Lumi.sln`、不加視窗的 `Icon` 屬性。

---

### 包 20B：提醒排程頁（Wave 2，20A 合併後）

- **目標**：`RemindersViewModel` 與 `RemindersPanel.axaml` 完成：列出所有提醒、可刪除、可清除已結束；不提供新增（尚未掛進 `SettingsWindow`，由協調者整合）。
- **先讀**：`src/Lumi.App/ViewModels/RemindersViewModel.cs`（骨架，**保留建構子簽章** `(ReminderService reminders, IConfirmService confirm, IUiDispatcher dispatcher, TimeProvider time)` 與 `IDisposable`）、`src/Lumi.App/ViewModels/MemoryViewModel.cs`（仿照 `Subscribe`／`OnStoreChanged`／`Dispose` 的變更訊號寫法、`SetMessage`、刪除確認的寫法、`ClearAllAsync`）、`src/Lumi.App/Views/MemoryPanel.axaml` 與 `.axaml.cs`（面板版型、列的樣式、窄視窗規則、訊息列）、`src/Lumi.Core/Reminders/ReminderService.cs`（`ChangeToken`、`ListAsync`、`DeleteAsync`、`DeleteFinishedAsync`）、`ScheduledItem.cs`、`ScheduledTypes.cs`、`Trigger.cs`、`ScheduleCalculator.cs`（`Describe`、`ResolveTimeZone`）、`tests/Lumi.App.Tests/TestSupport/SettingsHost.cs`（已有 `Reminders`、`ReminderService`、`ReminderStore`）、`tests/Lumi.App.Tests/Headless/MemoryPanelTests.cs`、`tests/Lumi.App.Tests/TestSupport/FakeTimeProvider.cs`、`SyncUiDispatcher.cs`、`FakeConfirmService.cs`。
- **可修改／新增**：`src/Lumi.App/ViewModels/RemindersViewModel.cs`、`src/Lumi.App/Views/RemindersPanel.axaml`、`RemindersPanel.axaml.cs`、`tests/Lumi.App.Tests/RemindersViewModelTests.cs`、`tests/Lumi.App.Tests/Headless/RemindersPanelTests.cs`。
- **步驟**：
  1. `RemindersViewModel`（`ObservableObject`，`IDisposable`）：
     - `ObservableCollection<ReminderRowViewModel> Pending`、`Finished`；`bool HasPending`、`HasFinished`、`ShowPendingEmpty`（`!HasPending`）、`ShowFinishedEmpty`（`!HasFinished`）；`[ObservableProperty] string? Message`、`bool MessageIsError`、`HasMessage`。
     - 建構時 `Subscribe()`（註冊 `reminders.ChangeToken`，被取消時 `dispatcher.Post(OnChanged)`，`OnChanged` 重新 `Subscribe()` 後 `_ = ReloadAsync()`；`Dispose` 後不再處理）並 `_ = ReloadAsync()`。先取得權杖、之後才讀資料（同 `MemoryViewModel`）。
     - `public async Task ReloadAsync()`：`reminders.ListAsync(new ReminderQuery(null, null, 500), CancellationToken.None)`；狀態是 `Pending` 的進 `Pending`（維持 store 的順序），其餘進 `Finished`（反過來排，最近的在最上面）；例外時 `Message = Strings.Reminders_LoadFailed`、`MessageIsError = true`。
     - `ReminderRowViewModel`（同檔，sealed）：`Id`、`Title`、`Message`（可空白）、`KindText`（`Reminders_KindReminder`／`KindTimer`／`KindSchedule`）、`StatusText`（待觸發且 `Enabled == false` → `Reminders_StatusPaused`；Completed／Missed／Cancelled → 對應 `Reminders_StatusXxx`；其餘 null）、`ScheduleText`、`CreatedByText`（`Reminders_CreatedByAgent`／`CreatedByUser`）、`DeleteCommand`（`IAsyncRelayCommand`）。`ScheduleText`：待觸發的週期項目是 `ScheduleCalculator.Describe(recurrence, item.TimeZoneId)`，有 `NextFireAt` 時再接 `" · "` 與 `string.Format(Strings.Reminders_NextFireFormat, 時間)`；待觸發的一次性項目只有「下次」那段；已結束的項目是 `string.Format(Strings.Reminders_LastFiredFormat, 時間)`（沒有 `LastFiredAt` 就是 null）。時間格式固定 `"yyyy/MM/dd HH:mm"`，轉成 `ScheduleCalculator.ResolveTimeZone(item.TimeZoneId)` 的當地時間（`TimeZoneInfo.ConvertTime`）。
     - `DeleteAsync(row)`：`confirm.ConfirmDestructiveAsync($"{Strings.Reminders_DeleteQuestion}\n{row.Title}", Strings.Reminders_Delete)`，取消就結束；`reminders.DeleteAsync(row.Id, CancellationToken.None)`；成功 `Message = Strings.Reminders_Deleted`；回 false（已經不存在）也當成成功並重新載入；例外 `Message = Strings.Reminders_DeleteFailed`、`MessageIsError = true`。
     - `[RelayCommand(CanExecute = nameof(HasFinished))] ClearFinishedAsync()`：確認 `string.Format(Strings.Reminders_ClearFinishedQuestion, Finished.Count)`、按鈕 `Strings.Reminders_ClearFinished`；`reminders.DeleteFinishedAsync`；`Message = string.Format(Strings.Reminders_Cleared, 筆數)`。`HasFinished` 變動時 `ClearFinishedCommand.NotifyCanExecuteChanged()`。
  2. `RemindersPanel.axaml`（`UserControl`，`x:DataType="vm:RemindersViewModel"`，版型仿 `MemoryPanel`：`ScrollViewer` → `ctl:FractionWidthLimiter MaxPixels="640"` → `StackPanel`）：
     - 頂端說明 `muted`：`Strings.Reminders_RunningNote`；訊息列（`Message`，`muted`，`MessageIsError` 時 `Classes.error`），`IsVisible="{Binding HasMessage}"`。
     - 區段標題（`TextBlock Classes="section-title"`）`Reminders_PendingTitle`；沒有項目時顯示 `Reminders_EmptyPending`（`muted`，綁 `ShowPendingEmpty`）；`ItemsControl x:Name="PendingList"` 綁 `Pending`。
     - 區段標題 `Reminders_FinishedTitle`，右側「清除已結束」按鈕（`Classes="secondary"`，`Command="{Binding ClearFinishedCommand}"`）；下一行 `muted` 的 `Reminders_FinishedNote`；沒有項目時 `Reminders_EmptyFinished`；`ItemsControl x:Name="FinishedList"` 綁 `Finished`。
     - 一列（兩個清單共用同一個 `DataTemplate`，`x:DataType="vm:ReminderRowViewModel"`）：下邊 1px `LumiBorderBrush`、Padding `0,12`；左欄：第一行 `WrapPanel`：標題（14／600）、種類徽章（`Border Classes="badge"` 內 `KindText`）、狀態徽章（`StatusText` 非空才顯示，同樣 `badge`）；第二行訊息（`muted`，換行，空白不顯示）；第三行 `ScheduleText`（`muted`）與 `CreatedByText`（`muted`），以 `WrapPanel` 並排、間距 12；右欄「刪除」按鈕：`Button Classes="tertiary"`，內容 `StackPanel Horizontal` 的 `LumiIconTrash`（Size 16）加文字 `Strings.Reminders_Delete`，`Command="{Binding DeleteCommand}"`，`AutomationProperties.Name` 用 `Reminders_Delete`。窄視窗（`Window.narrow`）時刪除鈕移到內容下方、靠左（仿 `MemoryPanel` 的 `StackPanel.memory-actions` 規則）。
     - `RemindersPanel.axaml.cs`：只有 `InitializeComponent()`。
- **測試**（`RemindersViewModelTests`，用 `SettingsHost.Reminders`／`ReminderService`，`SyncUiDispatcher`、`FakeConfirmService`；時間用 `ManualTimeProvider` 或固定時間，建項目用 `ReminderService.CreateReminderAsync(new NewReminder(...))`，仿 `ReminderServiceCreateTests` 的建法）：
  - 建一個一次性提醒與一個每天的週期提醒後 `ReloadAsync`：`Pending` 有 2 筆、`Finished` 空、`ShowFinishedEmpty` 為 true；週期那筆的 `ScheduleText` 含 `ScheduleCalculator.Describe` 的結果。
  - 取消其中一筆（`CancelAsync`）後（變更訊號觸發自動重新載入）：該筆出現在 `Finished`、`StatusText` 是 `Reminders_StatusCancelled`。
  - 刪除：`Confirm` 回 true → 項目從清單消失且 `ReminderService.ListAsync` 也查不到、`Message` 是 `Reminders_Deleted`；`Confirm` 回 false → 項目還在、沒有訊息。
  - 刪掉待觸發項目後，`ReminderService.ProcessDueAsync` 不再為它通知（不需要，20A 已測；這裡只測 ViewModel，**不要重複**）。
  - 清除已結束：有 2 筆已結束時 `ClearFinishedCommand.CanExecute` 為 true；確認後 `Finished` 空、`Message` 含「2」；沒有已結束時 `CanExecute` 為 false。
  - 外部（工具）新增一筆提醒（直接呼叫 `ReminderService.CreateReminderAsync`）後 `Pending` 自動多一筆（驗證變更訊號）；`Dispose` 之後再新增，`Pending` 不再變動。
  - `Headless/RemindersPanelTests`（仿 `MemoryPanelTests`）：面板內 `PendingList` 與 `FinishedList` 存在；有 1 筆待觸發時 `PendingList` 有 1 個項目、每個刪除鈕 `AutomationProperties.Name` 非空；沒有項目時顯示空狀態文字。
- **完成檢查**：`dotnet build Lumi.sln` 零警告；`dotnet test tests/Lumi.App.Tests --filter "RemindersViewModelTests|RemindersPanelTests"` 全過。
- **不要做**：不新增「新增提醒」功能、不加啟停開關（只顯示「已暫停」狀態）、不改 `SettingsWindow.axaml`、`SettingsViewModel.cs`、`Strings.*`、DI 檔。
