# 01 方案骨架與基本對話

- 狀態：1A、1B、1C 已完成（Windows 待驗）；**1D 完成（Mac 手動驗收通過）**；M1 只剩 Windows 驗證
- 對應里程碑：M1

## 目標

可以在工作台用 Anthropic 模型進行串流對話；能中途停止；重開程式後對話還在。

## M1 拆成四個部分

M1 範圍大，拆成四段，每段結束都要能編譯、能執行、測試全過，才進下一段。
金鑰安全儲存原本在 M2，提前到 M1（1B），M1 就不需要「開發用環境變數」這種過渡做法。

| 部分 | 名稱 | 產出 | 依賴 |
|---|---|---|---|
| **1A** | 方案骨架與外觀基礎 | 方案與專案、Generic Host + Avalonia、字型、Light／Dark 主題 token、空的工作台視窗 | M0 |
| **1B** | 金鑰安全儲存 | `ISecretStore`、Windows DPAPI 實作、macOS Keychain 實作、測試用記憶體實作、契約測試 | 1A |
| 1C | Core 模型與 Anthropic Provider | `ChatEvent`、訊息與對話模型、`IChatProvider`、Anthropic 串流與 usage、fixture 測試 | 1B |
| 1D | 工作台最小版與 SQLite | `IConversationStore` + SQLite、訊息列表（經過 `EmojiText`）、輸入框、送出與停止、缺金鑰時的最小輸入對話框、重開後還原 | 1C |

主計畫 M1 的項目對應：

| 主計畫項目 | 部分 |
|---|---|
| 建立方案與各專案（相依方向依 `agent.md` 第 5 節） | 1A |
| Generic Host 組裝 DI；Avalonia 從 Host 取得服務 | 1A |
| 外觀基礎：字型、`LumiUiFont`／`LumiMonoFont`、Light／Dark 主題 token | 1A |
| （從 M2 提前）`ISecretStore`：Windows DPAPI、macOS Keychain | 1B |
| Core：`ChatEvent`、訊息與對話模型、`IChatProvider` | 1C |
| Anthropic Provider：串流（SSE）、usage | 1C |
| 工作台最小版：訊息列表、輸入框、送出與停止 | 1D |
| 訊息顯示經過 `EmojiText` | 1A 搬入元件、1D 使用 |
| SQLite：對話與訊息的基本存取 | 1D |

---

## 開工前要確認的事

| # | 問題 | 決定／建議 | 狀態 | 影響 |
|---|---|---|---|---|
| Q1 | M1 要建立哪些專案？ | 只建需要的。1A：`Lumi.Core`、`Lumi.Providers`、`Lumi.Infrastructure`、`Lumi.App` + 測試；1B 再建 `Lumi.Platform`。 | ✅ 已確認 | 1A |
| Q2 | M1 的 API 金鑰從哪裡來？ | `ISecretStore` 提前到 M1（1B）。設定頁仍在 M2，所以 1D 在缺金鑰時跳一個最小的輸入對話框，存進 `ISecretStore`；M2 由設定頁取代。不使用環境變數。 | ✅ 已確認提前；對話框待確認 | 1B、1D |
| Q3 | M1 用哪個模型、Base URL？ | 寫在 `appsettings.json` 的 `Lumi:Dev:Anthropic`，只放非機密值；M2 由 `IModelCatalog` 取代。建議預設 `claude-opus-5-5`、`https://api.anthropic.com`、`MaxTokens` 64000、`ContextWindow` 1000000（細節見 1C「設定」）。 | ✅ 已確認 | 1C |
| Q4 | Anthropic 用官方 SDK 還是自己寫？ | **建議改用官方 SDK**（NuGet `Anthropic`），只在 `Lumi.Providers` 內使用。它可以接我們的 `HttpClient`，所以仍走 `IHttpClientFactory`，測試也能用假 handler 餵 fixture。比較見 1C「Q4 兩個選項」。 | ✅ 已確認 | 1C、M2 |
| Q5 | 「Core 不引用 Avalonia」怎麼守？ | 架構測試。 | ✅ 已確認 | 1A |
| Q6 | Noto Sans TC 可變字重在 Avalonia 能不能正確出 400／500／700？ | 1A 第一項就做驗證（A3）。結果：不行，改用靜態字重檔（見實作紀錄）。 | ✅ 已驗證（Mac） | 1A |
| Q7 | 金鑰實作放哪個專案？ | 放 `Lumi.Platform`（見 1B 設計「放哪一層」）。`agent.md` 第 5 節已同步修改。 | ✅ 已確認 | 1B |
| Q9 | 要不要開「拒絕時自動改用其他模型」（server-side fallbacks）？ | 建議開：`claude-opus-5-5` 的安全分類器偶爾會誤擋正常請求（回 `stop_reason: refusal`）；開了之後由 API 在同一次呼叫內改用其他模型回答。要加 beta 標頭，回應可能多出 `fallback` 區塊，要原樣保存才能再送回去。可在設定裡關掉。 | ✅ 已確認 | 1C |
| Q10 | `ChatEvent` 八種事件，1C 要全部定義嗎？ | 建議 1C 只定義會發出的五種（`TextDelta`、`ThinkingDelta`、`Usage`、`Completed`、`Failed`）；工具相關的三種（`ToolCallStarted`、`ToolCallCompleted`、`ApprovalRequired`）到 M3 有工具規格時再加，避免先猜欄位。 | ✅ 已確認 | 1C、M3 |
| Q11 | fixture 從哪裡來？ | 先依官方文件的格式手寫（測試不能呼叫真實 API）。C11（錄製工具）先不做，之後有需要再補。 | ✅ 已確認 | 1C |
| Q12 | 缺金鑰對話框什麼時候跳？（Q2 留下的部分） | 送出時才檢查，不在啟動時跳：收到 `MissingApiKey`、`InvalidApiKey`（401）、`SecretUnreadable` 就跳出對話框；按「儲存」後存進 `ISecretStore`，**自動重送剛才那則訊息**；按「取消」則留下錯誤列，可以之後按「重試」。 | ✅ 已確認 | 1D |
| Q13 | M1 要不要「新對話」按鈕？ | 要，只做一顆按鈕（頂端列左側），開一則空白對話。歷史對話抽屜、切換舊對話是 M2。 | ✅ 已確認 | 1D |
| Q14 | 思考過程在 M1 怎麼顯示？ | 先用最簡單的方式接上（收合的「思考過程」，展開是純文字），正式的樣式到 M2 做 ui-spec 第 3 節時再處理。 | ✅ 已確認 | 1D |
| Q15 | API 失敗時要不要存檔？ | 使用者的訊息一定存；失敗的助理回應**不存**，錯誤列只在畫面上，提供「重試」。重開程式後錯誤列消失，但使用者的訊息還在，可以直接再送一次。 | ✅ 已確認 | 1D |
| Q16 | 資料庫檔案放哪、叫什麼？ | Mac：`~/Library/Application Support/Lumi/lumi.db`；Windows：`%APPDATA%\Lumi\lumi.db`（和 1B 的 `secrets.json` 同一個資料夾）。 | ✅ 已確認 | 1D |
| Q8 | Keychain 的 service 名稱 | `io.github.neillog.lumi`（沒有自有網域，依 GitHub 帳號的慣例格式），同時作為 M11 的 macOS bundle id。定案後不再更改，改了舊的鑰匙圈項目會找不到。 | ✅ 已確認 | 1B、M11 |

---

## 1A 方案骨架與外觀基礎

### 範圍

**做**
- 方案、專案、共用建置設定、套件版本集中管理
- Generic Host（DI、Options、Logging）與 Avalonia 的組裝
- 打包字型、`LumiUiFont`／`LumiMonoFont` 資源
- Light／Dark 主題 token，預設跟隨系統，系統切換即時生效
- 介面文字資源檔的基本架構
- 把 POC 的 `EmojiText` 搬進 `Lumi.App`
- 空的工作台主視窗：只有版面骨架（頂端列、訊息區、輸入區、狀態列四塊），用來驗收字型與主題
- 測試專案與架構測試

**不做**
- 金鑰儲存（1B）、任何對話、Provider、SQLite（1C、1D）
- 設定頁、主題切換 UI（M2；1A 只做「跟隨系統」）
- 小精靈視窗（M7）
- 打包、簽章（M11）

### 設計

#### 方案結構

```
Lumi.sln
Directory.Build.props        共用：net10.0、Nullable、ImplicitUsings、TreatWarningsAsErrors、LangVersion
Directory.Packages.props     集中管理套件版本（Central Package Management）
global.json                  固定 .NET 10 SDK 版本
src/
  Lumi.Core/
  Lumi.Providers/            1A 只建專案，內容 1B 才寫
  Lumi.Infrastructure/       1A 只建專案，內容 1C 才寫
  Lumi.App/
tests/
  Lumi.Core.Tests/
  Lumi.Architecture.Tests/   相依規則
poc/                         維持原樣，不納入 Lumi.sln
```

相依：`App → Core / Providers / Infrastructure`；`Providers → Core`；`Infrastructure → Core`。Core 沒有任何專案參考。

#### 套件（版本實作時取最新穩定版，寫進 `Directory.Packages.props`）

| 專案 | 套件 |
|---|---|
| Lumi.Core | `Microsoft.Extensions.DependencyInjection.Abstractions`、`Microsoft.Extensions.Logging.Abstractions`、`Microsoft.Extensions.Options` |
| Lumi.App | `Avalonia` 11.3.x、`Avalonia.Desktop`、`Avalonia.Themes.Fluent`、`CommunityToolkit.Mvvm`、`Microsoft.Extensions.Hosting` |
| 測試 | `xunit`、`xunit.runner.visualstudio`、`Microsoft.NET.Test.Sdk` |

#### Host 與 Avalonia 的組裝

`Program.Main`（`[STAThread]`，同步）：

1. `Host.CreateApplicationBuilder(args)`：讀 `appsettings.json`、設定 Logging、呼叫各層的 `AddLumiCore()`、`AddLumiApp()` 擴充方法註冊服務。
2. `host.Start()`。
3. `AppBuilder.Configure(() => new App(host.Services))` → `UsePlatformDetect()` → `ConfigureFonts(...)` → `StartWithClassicDesktopLifetime(args)`。
4. Avalonia 結束後 `host.StopAsync()`、`Dispose()`。

規則：
- `App` 由建構子拿到 `IServiceProvider`，**只在 `OnFrameworkInitializationCompleted` 解析主視窗**，這是唯一的 composition root；其他地方一律建構子注入，不傳 `IServiceProvider`、不用靜態存取。
- ViewModel 與 Window 都註冊在 DI（Window 用 transient，ViewModel 依用途）。
- Logging：M1 先用 Console + Debug provider；檔案 log 之後再說。

#### 字型

- 檔案放 `src/Lumi.App/Assets/Fonts/`（`AvaloniaResource`），授權檔放 `Assets/Fonts/LICENSES/`（三份 OFL）。
- `ConfigureFonts` 註冊 `EmbeddedFontCollection`，key `fonts:Lumi`。
- `App.axaml` 定義：
  - `LumiUiFont` = `fonts:Lumi#IBM Plex Sans, fonts:Lumi#Noto Sans TC`
  - `LumiMonoFont` = `fonts:Lumi#JetBrains Mono, fonts:Lumi#Noto Sans TC`
- 全域 Style 讓 `TextBlock`、`TextBox`、`Window` 預設用 `LumiUiFont`，字級內文 15。
- 字型來源：官方 GitHub release（IBM/plex、notofonts/noto-cjk 或 Google Fonts、JetBrains/JetBrainsMono），檔名與版本記在 `LICENSES/README.md`。

#### 主題 token

- `Themes/Tokens.axaml`：`ResourceDictionary.ThemeDictionaries` 下 `Light`、`Dark` 兩套，內容照 `docs/ui-spec.md` 1.2 的表，每個 token 同時提供 `Color` 與 `SolidColorBrush`（命名 `LumiXxxColor`／`LumiXxxBrush`）。
- `RequestedThemeVariant="Default"` 跟隨系統。
- Fluent 主題的強調色改用 Accent token，避免 Fluent 預設藍色混進來。
- 規則：元件 XAML 只能用 `{DynamicResource LumiXxxBrush}`，不能寫色碼。加一個簡單檢查（測試或建置步驟），掃 `Lumi.App/**/*.axaml` 中 `Tokens.axaml` 以外出現的 `#RRGGBB`。

#### 介面文字

- `Resources/Strings.resx`（繁中為預設語系），用 `x:Static` 引用。
- 1A 只放主視窗會用到的幾個字串，建立用法範例。

#### EmojiText

- 從 `poc/Lumi.OrbPoc/EmojiText.cs` 搬到 `Lumi.App/Controls/`，namespace 改為 `Lumi.App.Controls`。
- 包成一個可綁定的控制項（例如 `EmojiTextBlock`，有 `Text` 依賴屬性，變更時呼叫 `Apply`），讓 1D 的訊息列表可以直接用 binding，不用 code-behind。
- `IsEmoji` 是純函式，可加單元測試（放在 App 的測試專案或之後再加；見 A9）。

### 工作項目

- [x] **A1 方案與共用設定**：`global.json`、`Directory.Build.props`、`Directory.Packages.props`、`Lumi.sln`；`.gitignore` 確認涵蓋 `bin/`、`obj/`。
- [x] **A2 建立專案**：`Lumi.Core`、`Lumi.Providers`、`Lumi.Infrastructure`（類別庫）、`Lumi.App`（Avalonia 桌面）；設定專案參考。`app.manifest` 沿用 POC 的設定（DPI 感知）。
- [x] **A3 字型驗證（先做）**（Mac 完成、**Windows 11 待驗**）：在 Lumi.App 放三套字型，畫一個測試畫面顯示中英混排在 400／500／700 三種字重；Mac 確認後，Windows 11 再確認一次。可變字重不行就改靜態檔，並更新 ui-spec 1.1 與決策紀錄。
- [x] **A4 字型資源**：`EmbeddedFontCollection`、`LumiUiFont`／`LumiMonoFont`、全域預設字型與字級；授權檔。
- [x] **A5 主題 token**：`Tokens.axaml` 兩套色系、Fluent 強調色覆寫、跟隨系統；色碼檢查。
- [x] **A6 Host 組裝**：`Program`、`App(IServiceProvider)`、`AddLumiCore()`／`AddLumiApp()` 擴充方法、`appsettings.json`、Logging；程式關閉時 Host 正常停止。
- [x] **A7 字串資源**：`Strings.resx` 與引用方式。
- [x] **A8 工作台主視窗骨架**：`WorkbenchWindow` + `WorkbenchViewModel`（CommunityToolkit.Mvvm），四塊版面用 token 上色；放一段含中英文、程式碼字型、❤️‍🔥 的示範文字作為驗收用。
- [x] **A9 EmojiText 搬入**：`EmojiTextBlock` 控制項；示範文字改用它顯示。
- [x] **A10 測試專案**：`Lumi.Core.Tests`（先放一個 smoke test）、`Lumi.Architecture.Tests`（Core 不參考 Avalonia、Core 沒有專案參考）。
- [x] **A11 文件**：README 的「專案結構」與「怎麼執行」改成正式方案；主計畫 M1 狀態改 🟨；本檔實作紀錄。

### 驗收標準

- [x] `dotnet build` 零警告（`TreatWarningsAsErrors`），`dotnet test` 全過。
- [ ] Mac 與 Windows 11 都能 `dotnet run` 開出工作台骨架視窗。
- [x] 系統切換淺色／深色時，視窗即時換色，不需重啟。（Mac 已驗；Windows 待驗）
- [ ] 乾淨的 Windows 11（沒有另外安裝字型）上，中文是 Noto Sans TC，不是細明體或正黑體；程式碼文字是 JetBrains Mono。1920×1080 在 100%、125% 縮放下各看一次。
- [x] 示範文字中的 ❤️‍🔥 沒有被拆開。
- [x] 架構測試證明 `Lumi.Core` 不引用 Avalonia。
- [x] `Lumi.App` 的 axaml 除了 `Tokens.axaml` 沒有寫死色碼。

---

## 1B 金鑰安全儲存

### 範圍

**做**
- Core：`ISecretStore` 介面、`SecretKey`、`SecretValue`（防止誤印的包裝型別）
- Windows：DPAPI（目前使用者範圍）加密後存檔
- macOS：Keychain（登入鑰匙圈的 generic password 項目）
- 測試用的記憶體實作；一套契約測試，三個實作都跑
- DI 註冊：依作業系統選實作
- log 遮蔽的基本規則

**不做**
- 設定頁的金鑰輸入、測試連線（M2）
- 缺金鑰時的輸入對話框（1D）
- Linux 實作（不在支援範圍；Linux 只跑記憶體實作的測試）
- 金鑰匯出／跨機器搬移（不提供：金鑰綁在這台電腦的這個使用者）

### 設計

#### 介面（`Lumi.Core/Secrets/`）

```csharp
public interface ISecretStore
{
    Task<SecretValue?> GetAsync(SecretKey key, CancellationToken ct = default);
    Task SetAsync(SecretKey key, SecretValue value, CancellationToken ct = default);
    Task<bool> DeleteAsync(SecretKey key, CancellationToken ct = default);   // 不存在回 false
    Task<bool> ExistsAsync(SecretKey key, CancellationToken ct = default);
}

// 名稱格式：小寫英數、- 與 /，例如 connection/{connectionId}/api-key、speech/whisper/token
public readonly record struct SecretKey(string Name) { /* 建構時驗證格式，不合法丟 ArgumentException */ }

// 包住明文，ToString() 一律回傳 "***"，避免被 log、例外訊息、除錯顯示帶出去
public sealed class SecretValue
{
    public string Reveal();          // 只在真正要用的地方（組 HTTP 標頭）呼叫
    public override string ToString() => "***";
}
```

- 介面是非同步的：Keychain 呼叫可能跳出系統授權視窗而卡住，不能在 UI 執行緒上同步等。
- `SecretKey` 只放「這是哪個連線的金鑰」，不放金鑰內容；`connectionId` 在 M1 固定為 `dev-anthropic`，M2 起改用 `IModelCatalog` 的連線 ID。
- 讀取失敗分兩種：「沒有這個金鑰」回 `null`；「有但讀不出來」（解密失敗、鑰匙圈被鎖、使用者拒絕授權）丟 `SecretStoreException`，訊息不含金鑰內容，UI 顯示「金鑰無法讀取，請重新輸入」。

#### 放哪一層（Q7）

- `ISecretStore` 介面在 `Lumi.Core`。
- 兩個實作放 `Lumi.Platform`：Keychain 需要 P/Invoke，而 `agent.md` 第 14 節規定 P/Invoke 只能在 Platform。DPAPI 雖然是受控 API（`ProtectedData`），也是 Windows 專屬，一起放 Platform，兩個平台的實作放在同一處比較好維護。
- 依 `agent.md` 第 6 節，`IAudioRecorder`、`IOverlayPlatform` 本來就是 Core 介面、由 Platform 實作，所以 `Platform ──► Core` 原本就隱含存在，這次只是把它寫明。
- 記憶體實作 `InMemorySecretStore` 不另開專案，直接放在測試專案裡。

```
src/Lumi.Platform/
  Secrets/
    WindowsDpapiSecretStore.cs     [SupportedOSPlatform("windows")]
    MacKeychainSecretStore.cs      [SupportedOSPlatform("macos")]
  Interop/Mac/
    Security.cs                    SecItem* 的 P/Invoke
    CoreFoundation.cs              CFString、CFData、CFDictionary 建立與釋放
  PlatformServiceCollectionExtensions.cs   AddLumiPlatform()：依 OS 註冊對應實作
```

#### Windows：DPAPI

- `ProtectedData.Protect(明文 UTF-8, entropy, DataProtectionScope.CurrentUser)`。套件：`System.Security.Cryptography.ProtectedData`。
- `entropy`：固定的應用程式專屬位元組（不是祕密，只是把 Lumi 的資料和其他程式的 DPAPI 資料區隔開）。
- 存放：`%APPDATA%\Lumi\secrets.json`，內容只有加密後的 base64：

  ```json
  { "version": 1, "items": { "connection/dev-anthropic/api-key": "AQAAANCMnd8B..." } }
  ```

- 寫入：先寫暫存檔再取代（`File.Replace`／`File.Move(overwrite: true)`），避免寫一半當機把整個檔案弄壞。同一程序內用 `SemaphoreSlim` 序列化寫入。
- 讀取：每次從檔案讀，不另外快取（檔案小、讀取便宜，也避免明文長時間留在記憶體）。
- 已知限制：Windows 密碼被管理員重設（不是使用者自己改）時，DPAPI 金鑰會失效，讀取會解密失敗 → 走 `SecretStoreException`，請使用者重新輸入。

#### macOS：Keychain

- 項目類型：`kSecClassGenericPassword`；`kSecAttrService` = `io.github.neillog.lumi`（Q8）；`kSecAttrAccount` = `SecretKey.Name`；`kSecValueData` = 明文 UTF-8；`kSecAttrLabel` = `Lumi：{Name}`（在「鑰匙圈存取」App 裡看得懂）。
- API：`SecItemAdd`、`SecItemCopyMatching`、`SecItemUpdate`、`SecItemDelete`（Security.framework）。Set 的流程：先 Update，回 `errSecItemNotFound` 再 Add。
- `kSecClass`、`kSecAttrService` 等常數是 framework 匯出的符號，用 `NativeLibrary.GetExport` 取位址再讀出 `CFStringRef`，不要自己寫字串。
- 所有 CF 物件用完都要 `CFRelease`；包成 `SafeHandle`（`SafeCFHandle`）避免漏掉。
- 錯誤碼對應：`errSecItemNotFound`(-25300) → `null`／`false`；`errSecUserCanceled`(-128)、`errSecAuthFailed`(-25293)、`errSecInteractionNotAllowed`(-25308) → `SecretStoreException`；其他 → `SecretStoreException`，附 `OSStatus` 數字（不附內容）。
- **使用登入鑰匙圈（legacy file-based keychain），不使用 data protection keychain**：後者需要簽章與 `keychain-access-groups` entitlement，未簽章的開發版會直接失敗（-34018）。M11 簽章後再評估要不要切換。
- 已知開發期的麻煩：登入鑰匙圈的存取權限綁在程式的簽章上。`dotnet run` 每次重新建置，簽章都會變，系統會再跳一次「Lumi 想要使用鑰匙圈中的機密資訊」。開發期接受這件事；M11 正式簽章後就只會問一次。

#### DI

```csharp
// Lumi.Platform
public static IServiceCollection AddLumiPlatform(this IServiceCollection services)
{
    if (OperatingSystem.IsWindows())     services.AddSingleton<ISecretStore, WindowsDpapiSecretStore>();
    else if (OperatingSystem.IsMacOS())  services.AddSingleton<ISecretStore, MacKeychainSecretStore>();
    else throw new PlatformNotSupportedException("Lumi 只支援 Windows 與 macOS");
    return services;
}
```

- 選項（`SecretStoreOptions`）：DPAPI 檔案路徑、Keychain service 名稱（預設 `io.github.neillog.lumi`；測試用 `io.github.neillog.lumi.tests`）。測試時可以換成測試專用的值，不會碰到正式資料。

#### 防止外洩

- `SecretValue.ToString()` 回傳 `***`；`SecretKey` 可以印（只是名稱）。
- 1C 的 HTTP client 要關閉標頭記錄，或把 `x-api-key`、`Authorization` 標頭遮蔽（`RedactLoggedHeaders`）。這條寫進 1C 的工作項目。
- 不實作「顯示目前金鑰」的功能；M2 的設定頁只顯示「已設定 · 結尾 xxxx」，結尾四碼在 Set 時算好，存在設定（不是祕密）或每次讀出後只取後四碼。→ M2 再決定。

### 工作項目

- [x] **B1 Core 介面**：`ISecretStore`、`SecretKey`（含格式驗證）、`SecretValue`、`SecretStoreException`；單元測試（名稱驗證、`ToString` 遮蔽）。
- [x] **B2 契約測試**：抽象測試類別 `SecretStoreContractTests`，涵蓋：讀不存在的回 null、寫後讀、覆寫、刪除、刪除不存在的回 false、中文與 emoji 內容、長字串（4 KB）、多個 key 互不影響。
- [x] **B3 記憶體實作**：放測試專案，跑契約測試。
- [x] **B4 建立 `Lumi.Platform`**：專案、`AddLumiPlatform()`、`SecretStoreOptions`；架構測試加上「P/Invoke（`DllImport`／`LibraryImport`）只能出現在 `Lumi.Platform`」。
- [x] **B5 Windows DPAPI 實作**：原子寫入、寫入序列化、檔案損毀或解密失敗時丟 `SecretStoreException`。
- [x] **B6 macOS Keychain 實作**：CoreFoundation／Security 的 P/Invoke（用 `LibraryImport`）、`SafeCFHandle`、錯誤碼對應。
- [x] **B7 平台測試**（Mac 完成、**Windows 待跑**）：`Lumi.Platform.Tests`，用測試專用的檔案路徑與 service 名稱跑契約測試；在非對應平台上自動略過。Mac、Windows 各跑一次。
- [x] **B8 手動驗證**（Mac 完成、**Windows 待驗**）：Mac 在「鑰匙圈存取」看得到項目、名稱正確；Windows 的 `secrets.json` 裡沒有明文；兩邊重開程式後都讀得到。
- [x] **B9 文件**：`agent.md` 第 5 節已依 Q7 更新；主計畫已把 `ISecretStore` 從 M2 移到 M1，決策紀錄已加。

### 驗收標準

- [x] 契約測試在記憶體、DPAPI（Windows）、Keychain（Mac）三個實作都通過。（記憶體、Keychain 已過；DPAPI 契約測試已寫好，待在 Windows 跑）
- [x] 在檔案系統、log 中搜尋測試用的金鑰字串，找不到明文。（加密檔以測試驗證；Keychain 項目以 `security` 確認；Windows 待驗）
- [x] `Lumi.Core` 只有介面，沒有任何平台相關程式碼；P/Invoke 只在 `Lumi.Platform`（架構測試保證）。
- [x] 讀取失敗（例如 Mac 上在授權視窗按「拒絕」）不會讓程式崩潰，回報的錯誤訊息不含金鑰。（錯誤碼對應與訊息不含內容已由測試與程式碼保證；「按拒絕」的實機操作未測）

---

## 1C Core 模型與 Anthropic Provider

### 範圍

**做**
- Core：訊息與對話模型、內容區塊、`ChatRequest`、`ChatEvent`、`TokenUsage`、`IChatProvider`
- `Lumi.Providers`：Anthropic Provider（串流、thinking、usage、錯誤對應），從 `ISecretStore` 取金鑰
- HTTP log 遮蔽 `x-api-key`、`Authorization`
- `Lumi.Providers.Tests`：fixture 測試（不呼叫真實 API）
- 架構測試：Core 不引用供應商 SDK

**不做**
- 工具呼叫（M3）、Prompt 分層（M5；1C 不送 system prompt）
- 對話迴圈與存檔（1D）、任何 UI（1D）
- OpenAI 相容（M2）、多組連線與模型（M2 的 `IModelCatalog`）
- 自動重試串流中斷的回應（只有「還沒開始串流」的連線錯誤會由 SDK 重試）

### Q4 兩個選項

| | A. 官方 SDK（建議） | B. 自己寫 HTTP + SSE |
|---|---|---|
| 做法 | `Lumi.Providers` 引用 NuGet `Anthropic`（目前 12.58.0），把 SDK 的串流事件轉成 `ChatEvent` | typed client 自己組 JSON、自己解析 SSE |
| `IHttpClientFactory` | 可以：SDK 的 client 有 `HttpClient` 屬性，傳入 factory 建立的 client | 可以 |
| 要自己維護 | 只有事件轉換與錯誤對應 | 請求與回應的格式、SSE 解析、重試、`anthropic-version`、新欄位 |
| API 改版（新區塊類型、新 beta） | 升 SDK 版本 | 自己追 |
| 對 M2 的影響 | OpenAI 相容仍要另寫，屆時再決定用 OpenAI SDK 或自寫 SSE 解析 | SSE 解析可共用 |
| 風險 | 多一個相依（只在 Providers）；SDK 型別不能外流到 Core | 格式細節（thinking signature、fallback 區塊、usage 合併）容易漏 |

`agent.md` 第 4 節沒有禁止供應商 SDK，只規定 Core 不能引用；第 5 節本來就寫「各家格式轉換留在 Providers」。選 A 不需要改 `agent.md`，但要在主計畫決策紀錄加一筆。
選 A 時，原計畫的「SSE 解析器（放 Providers 共用）」從 1C 移到 M2，看那時的需要再決定。下面的設計以 A 為準；選 B 時只有「Anthropic Provider」一節要換掉，其他不變。

### 設計

#### Core 模型（`Lumi.Core/Chat/`）

```csharp
public enum ChatRole { User, Assistant }

// 內容區塊。為了能把對話原樣送回 API，回應裡的每個區塊都要保留下來。
public abstract record ContentBlock;
public sealed record TextBlock(string Text) : ContentBlock;
public sealed record ThinkingBlock(string Thinking, string Signature) : ContentBlock;   // Signature 是供應商的不透明資料，原樣送回
public sealed record RedactedThinkingBlock(string Data) : ContentBlock;
// 不認得或供應商專屬的區塊（例如 fallback 區塊）：存原始 JSON，只有同一個供應商會送回去，其他供應商略過
public sealed record ProviderBlock(string Provider, string Type, string RawJson) : ContentBlock;

public enum MessageStatus { Complete, Stopped, Failed }

public sealed record ChatMessage(
    string Id, ChatRole Role, IReadOnlyList<ContentBlock> Content,
    DateTimeOffset CreatedAt, MessageStatus Status = MessageStatus.Complete,
    string? ModelId = null, TokenUsage? Usage = null);

public sealed record Conversation(string Id, string Title, DateTimeOffset CreatedAt, DateTimeOffset UpdatedAt);

public sealed record ChatRequest(string ModelId, IReadOnlyList<ChatMessage> Messages, int MaxTokens);

public sealed record TokenUsage(int InputTokens, int OutputTokens, int CacheCreationInputTokens, int CacheReadInputTokens);
```

- ID 用 `Guid.CreateVersion7()` 的字串（依時間排序，1D 存 SQLite 時方便）。
- `ContentBlock` 的 JSON 序列化（`[JsonPolymorphic]`）放 Core，1D 存檔時用；1C 先寫好並測來回轉換。
- **對話歷史只能往後加**：已送出的訊息與區塊不能修改或重排，thinking 區塊要原樣送回。新模型會檢查 thinking 區塊的 signature 與前面的對話是否一致，改過歷史會回 400。從尾端移除（例如 M2 的「重新產生」）沒有問題。這條規則寫進 `agent.md` 第 7 節。

#### `ChatEvent` 與 `IChatProvider`

```csharp
public abstract record ChatEvent;
public sealed record TextDelta(string Text) : ChatEvent;
public sealed record ThinkingDelta(string Text) : ChatEvent;
public sealed record Usage(TokenUsage Tokens) : ChatEvent;            // 數字有更新就發，最後一次為準
public sealed record Completed(ChatMessage Message, StopReason Reason) : ChatEvent;
public sealed record Failed(ChatFailure Failure) : ChatEvent;
// ToolCallStarted、ToolCallCompleted、ApprovalRequired：M3（見 Q10）

public enum StopReason { EndTurn, MaxTokens, StopSequence, Other }

public interface IChatProvider
{
    IAsyncEnumerable<ChatEvent> StreamAsync(ChatRequest request, CancellationToken ct = default);
}
```

事件順序的約定（寫進介面註解，fixture 測試逐條驗證）：
- 一次呼叫**剛好以一個 `Completed` 或一個 `Failed` 結束**，之後不再有事件。
- `Completed.Message` 是完整的助理訊息，包含 thinking 區塊與 signature，1D 直接存檔。
- **取消**：丟 `OperationCanceledException`，不發 `Failed`。已收到的文字由呼叫端（1D 的對話流程）自己累積，存成 `Stopped`。未完成的 thinking 沒有 signature、不能送回，不保留。
- 例外不往外丟（取消除外），一律轉成 `Failed`。

`ChatFailure`：

```csharp
public sealed record ChatFailure(ChatFailureKind Kind, string Message, bool Retryable, TimeSpan? RetryAfter = null, string? RequestId = null);

public enum ChatFailureKind
{
    MissingApiKey,      // ISecretStore 沒有金鑰（1D 跳出輸入對話框）
    SecretUnreadable,   // SecretStoreException
    InvalidApiKey,      // 401
    PermissionDenied,   // 403
    ModelNotFound,      // 404
    BadRequest,         // 400、413
    RateLimited,        // 429，帶 retry-after
    Overloaded,         // 529
    ServerError,        // 其他 5xx
    Network,            // 連不上、DNS、TLS、逾時
    StreamError,        // 串流中途收到 error 事件或連線中斷
    Refused,            // stop_reason: refusal（含 fallback 也拒絕）
    Unknown,
}
```

`Message` 是給使用者看的繁中說明（放資源檔的文字由 1D 對應，Provider 只給 Kind 與供應商原始訊息），**不得包含金鑰**。`refusal` 時已串出的部分內容要丟掉，不存檔。

#### Anthropic Provider（`Lumi.Providers/Anthropic/`）

- `AnthropicChatProvider : IChatProvider`，typed client：`services.AddHttpClient<AnthropicChatProvider>()`，`IChatProvider` 轉註冊到它。每次呼叫用 factory 給的 `HttpClient` 建 SDK client，`ApiKey` 從 `ISecretStore` 讀（`connection/dev-anthropic/api-key`），用完不保留。
- 請求：`model`、`max_tokens`、`messages`、`stream`；`thinking: {type: "adaptive", display: "summarized"}`（`claude-opus-5-5` 一律會思考，不設 display 的話 thinking 內容是空字串，UI 就沒東西可顯示）；`output_config.effort` 有設才送（不設時這個模型預設 `medium`）；Q9 開啟時加 `fallbacks: "default"` 與 beta `server-side-fallback-2026-07-01`。
- 內容區塊轉換：`TextBlock`、`ThinkingBlock`（含 signature）、`RedactedThinkingBlock` 對應 SDK 型別；`ProviderBlock(Provider: "anthropic")` 原樣放回；其他供應商的 `ProviderBlock` 略過。
- 事件對應：

| API 事件 | 處理 |
|---|---|
| `message_start` | 記下 message id、model、初始 usage → 發 `Usage` |
| `content_block_start` | 依類型開一個累積中的區塊（text／thinking／redacted_thinking／其他存原始 JSON） |
| `content_block_delta` `text_delta` | 累積 → 發 `TextDelta` |
| `content_block_delta` `thinking_delta` | 累積 → 發 `ThinkingDelta` |
| `content_block_delta` `signature_delta` | 累積到該 thinking 區塊的 signature，不發事件 |
| `content_block_stop` | 區塊完成 |
| `message_delta` | 記下 `stop_reason`；usage 合併（每個欄位以最後收到的值為準）→ 發 `Usage` |
| `message_stop` | `stop_reason` 是 `refusal` → `Failed(Refused)`；否則 `Completed` |
| `ping` | 略過 |
| `error` | `Failed`（`overloaded_error` → Overloaded，其餘 → StreamError） |
| 沒看過的事件 | 略過並記 Debug log（API 會新增事件類型，不能因此失敗） |

- 錯誤對應：SDK 的例外類別（`Anthropic.Exceptions`）→ `ChatFailureKind`，依上表；連線類例外 → `Network`；串流中斷 → `StreamError`。SDK 的重試次數設 2（只重試還沒開始串流的 429／5xx／連線錯誤）。
- 逾時：長回應可能串很多分鐘，SDK 與 `HttpClient` 的逾時都要放寬（實作時確認 SDK 的 `Timeout` 是整個請求還是到第一個 byte）；停止一律靠 `CancellationToken`。
- log：`AddHttpClient` 加 `RedactLoggedHeaders(["x-api-key", "authorization"])`；Provider 自己的 log 不記請求與回應內容，只記 model、stop reason、usage、request id。

#### 設定（Q3）

`appsettings.json`：

```json
"Lumi": {
  "Dev": {
    "Anthropic": {
      "BaseUrl": "https://api.anthropic.com",
      "Model": "claude-opus-5-5",
      "MaxTokens": 64000,
      "ContextWindow": 1000000,
      "Effort": null,
      "UseServerFallbacks": true
    }
  }
}
```

- 對應 `AnthropicDevOptions`（Providers），`ValidateOnStart`：BaseUrl 必須是 https 的絕對網址、MaxTokens 1～128000。
- `ContextWindow` 1C 不用，1D 的狀態列用。
- 金鑰不在這裡：只放 `ISecretStore`。

#### 專案與套件

- `Lumi.Providers`：加 `Anthropic`、`Microsoft.Extensions.Http`。
- 新增 `tests/Lumi.Providers.Tests`。
- `Lumi.App`：呼叫 `AddLumiProviders()`、綁定設定；1C 不會實際呼叫 Provider，UI 在 1D。
- 架構測試：只有 `Lumi.Providers` 能引用 `Anthropic` 組件。

#### 測試

- 假的 `HttpMessageHandler` 回傳 fixture，並**故意把回應切成很小的片段**（1、3、7 byte）送出，確保中文與 emoji 被切在 UTF-8 中間也不會壞。
- Fixture（`tests/Lumi.Providers.Tests/Fixtures/anthropic/*.sse`、`*.json`）：
  - 純文字；中文 + ❤️‍🔥 等組合 emoji；thinking + signature + 文字
  - 中途夾 `ping`；沒看過的事件類型
  - `stop_reason: max_tokens`；`refusal`（開頭就拒絕、串到一半才拒絕）；含 `fallback` 區塊的回應
  - 串流中的 `error` 事件（`overloaded_error`）；串到一半連線中斷
  - HTTP 400／401／404／429（含 retry-after）／529 的 JSON 錯誤
- 其他測試：
  - 請求內容：model、max_tokens、stream、thinking 設定、fallbacks 開關、送回的 thinking 區塊與 signature 原樣保留
  - `x-api-key` 只出現在送出的標頭，不在 log、例外、`ChatFailure` 裡
  - 缺金鑰 → `Failed(MissingApiKey)`，而且沒有發出任何 HTTP 請求
  - 取消：串到一半取消 → `OperationCanceledException`，連線被關閉
  - 事件順序約定（剛好一個結尾事件、結尾後沒有事件）
  - `ContentBlock` JSON 來回轉換
- 選用（Q11）：`tools/RecordAnthropicFixture`，本機手動執行、用你的金鑰錄一次真實回應，不在 CI 跑。

### 工作項目

- [x] **C1 Core 模型**：`ChatMessage`、`ContentBlock` 各類型、`Conversation`、`ChatRequest`、`TokenUsage`；`ContentBlock` 的 JSON 序列化與測試。
- [x] **C2 事件與介面**：`ChatEvent` 五種、`StopReason`、`ChatFailure`、`IChatProvider`（註解寫明事件順序與取消的約定）。
- [x] **C3 設定**：`AnthropicDevOptions`、`appsettings.json`、驗證。
- [x] **C4 Provider 骨架**：套件、`AddLumiProviders()`、typed client、header 遮蔽、從 `ISecretStore` 取金鑰、缺金鑰處理。
- [x] **C5 請求轉換**：Core 訊息 → SDK 請求（含 thinking 區塊原樣送回、`ProviderBlock` 篩選、thinking／effort／fallbacks 設定）。
- [x] **C6 串流轉換**：SDK 事件 → `ChatEvent`、區塊累積、usage 合併、stop reason、refusal。
- [x] **C7 錯誤對應**：HTTP 錯誤、串流 error、連線中斷 → `ChatFailure`。
- [x] **C8 Fixture 與測試**：上面列的 fixture 與測試；`Lumi.Providers.Tests` 加進方案。
- [x] **C9 架構測試**：Core 不引用 `Anthropic`；只有 Providers 引用。
- [x] **C10 文件**：`agent.md` 第 7 節加「對話歷史只能往後加、thinking 區塊原樣送回」；主計畫決策紀錄（Q4、Q9）；本檔實作紀錄。
- [ ] ~~C11（選用，Q11）手動錄製工具~~：先不做，只用手寫 fixture；之後要核對格式再補。

### 驗收標準

- [x] `dotnet build` 零警告，`dotnet test` 全過，測試中沒有任何真實的網路呼叫。
- [x] 所有 fixture 在 1／3／7 byte 切塊下結果一致，中文與 emoji 完整。
- [x] 每次呼叫剛好以一個 `Completed` 或 `Failed` 結束；取消時丟 `OperationCanceledException`。
- [x] `Completed.Message` 送回 Provider 時，thinking 區塊與 signature 和收到時完全相同。
- [x] 金鑰不出現在 log、例外訊息、`ChatFailure` 中（測試保證）。
- [x] 架構測試證明 Core 不引用 `Anthropic` SDK（只有 `Lumi.Providers` 引用）。
- [x] ~~（有 C11 時）用真實金鑰錄一次~~：C11 依 Q11 先不做。

## 1D 工作台最小版與 SQLite

### 範圍

**做**
- Core：`IConversationStore` 介面、對話流程 `ConversationService`（組請求、呼叫 Provider、存檔、停止）
- `Lumi.Infrastructure`：SQLite 實作、schema 版本管理、資料庫路徑
- 工作台：訊息列表、思考過程（最簡版）、輸入框、送出／停止、錯誤列與重試、狀態列、「新對話」按鈕
- 缺金鑰時的最小輸入對話框
- 啟動時載入最近一則對話
- 測試：SQLite 實作、對話流程、ViewModel

**不做**
- Markdown 渲染、程式碼區塊、複製與重新產生（M2）
- 歷史對話抽屜、切換舊對話、刪除對話（M2）
- 模型切換、設定頁（M2；M1 用 `appsettings.json` 的模型）
- 檔案與資料夾上下文、語音（M4、M6）
- 小精靈（M7）

### 設計

#### `IConversationStore`（`Lumi.Core/Chat/`）

```csharp
public interface IConversationStore
{
    Task<Conversation> CreateAsync(string title, CancellationToken ct = default);
    Task<Conversation?> GetLatestAsync(CancellationToken ct = default);            // 依 UpdatedAt
    Task<IReadOnlyList<ChatMessage>> GetMessagesAsync(string conversationId, CancellationToken ct = default);
    Task AppendMessageAsync(string conversationId, ChatMessage message, CancellationToken ct = default);
    Task RenameAsync(string conversationId, string title, CancellationToken ct = default);
}
```

- **只有新增，沒有修改或刪除訊息**：配合「對話歷史只能往後加」（`agent.md` 第 7 節）。M2 的「重新產生」需要從尾端移除時再加方法。
- `AppendMessageAsync` 同時更新該對話的 `UpdatedAt`。

#### SQLite（`Lumi.Infrastructure/Persistence/`）

- 套件：`Microsoft.Data.Sqlite`。每次操作開一條連線（SQLite 連線很便宜），啟動時設 `PRAGMA journal_mode=WAL`、`foreign_keys=ON`。
- 資料表：

```sql
CREATE TABLE schema_version (version INTEGER NOT NULL);

CREATE TABLE conversations (
  id          TEXT PRIMARY KEY,
  title       TEXT NOT NULL,
  created_at  TEXT NOT NULL,          -- ISO 8601 UTC
  updated_at  TEXT NOT NULL
);

CREATE TABLE messages (
  id              TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  seq             INTEGER NOT NULL,   -- 對話內的順序，從 1 開始；排序以它為準，不靠 id
  role            TEXT NOT NULL,      -- user / assistant
  status          TEXT NOT NULL,      -- complete / stopped / failed
  model_id        TEXT,
  content_json    TEXT NOT NULL,      -- ContentBlock 陣列，用 1C 的 JSON 序列化，thinking signature 原樣保存
  input_tokens    INTEGER, output_tokens INTEGER,
  cache_creation_input_tokens INTEGER, cache_read_input_tokens INTEGER,
  created_at      TEXT NOT NULL,
  UNIQUE (conversation_id, seq)
);
CREATE INDEX ix_conversations_updated ON conversations(updated_at);
```

- schema 版本：程式內一串依序執行的 migration（M1 只有第 1 版），啟動時檢查 `schema_version`，在同一個 transaction 裡補到最新。資料庫版本比程式新時拒絕開啟並回報（避免舊版程式寫壞新版資料）。
- 資料庫路徑（Q16）：`LumiPaths.DataDirectory` 依作業系統決定，放 `Lumi.Infrastructure`；測試用暫存資料夾。
- 金鑰不會進資料庫：`ContentBlock` 裡沒有金鑰，架構上也沒有路徑把 `SecretValue` 寫進來。

#### 對話流程 `ConversationService`（`Lumi.Core/Chat/`）

```csharp
public sealed class ConversationService
{
    // 送出一則使用者訊息：先存使用者訊息，再串流助理回應
    IAsyncEnumerable<ChatEvent> SendAsync(string conversationId, string userText, CancellationToken ct);
    // 不新增使用者訊息，直接用目前的歷史再要一次回應（錯誤列的「重試」、存好金鑰後的自動重送）
    IAsyncEnumerable<ChatEvent> RetryAsync(string conversationId, CancellationToken ct);
}
```

流程：
1. `SendAsync`：把使用者訊息存檔（`Complete`）。第一則訊息時，用前 30 個字當對話標題。
2. 讀出整段歷史，**組成請求**：
   - 略過沒有內容的訊息（例如還沒收到任何文字就停止的回應）。
   - 連續兩則以上同角色的訊息合併成一則（例如上一輪失敗沒有助理回應，使用者又送了一則）。API 要求 user／assistant 交替；合併只發生在送出的請求裡，資料庫保持原樣。
   - 只送 `Complete` 和 `Stopped` 的訊息。
3. 呼叫 `IChatProvider.StreamAsync`，事件原樣轉給呼叫端，同時累積文字。
4. 收到 `Completed` → 存助理訊息（`Complete`，含 thinking 與 usage）。
5. 收到 `Failed` → 不存（Q15）。
6. 取消（`OperationCanceledException`）→ 如果已經收到文字，存一則 `Stopped` 的助理訊息（只有文字，未完成的思考內容不保留，見 1C），然後把例外往外丟，讓 UI 知道是停止。存檔用 `CancellationToken.None`，不能因為取消而存不進去。

- 模型與 `MaxTokens` 從 `AnthropicDevOptions` 來；Core 不認得 Providers 的型別，所以由 App 注入一個 `ChatDefaults`（`ModelId`、`MaxTokens`、`ContextWindow`）給 Core。M2 由 `IModelCatalog` 取代。
- 同一則對話同時只能有一個回應進行中，第二次呼叫丟 `InvalidOperationException`（UI 本來就會停用送出鈕，這是防呆）。

#### 工作台 UI（`Lumi.App`）

版面沿用 1A 的四塊，內容換成真的：

| 區塊 | M1 內容 |
|---|---|
| 頂端列 | 左：「新對話」按鈕（Q13）、Lumi 標誌；中：目前模型名稱（只顯示，不能切換） |
| 訊息區 | 使用者訊息靠右、`UserBubble` 底色；助理訊息靠左、不加底，左側小光球（純色圓點）。文字一律用 `EmojiTextBlock`。思考過程用最簡單的收合區塊接上（Q14），正式樣式是 M2。停止的訊息尾端顯示「你已停止這次回應」。失敗時最後一則下面顯示錯誤列（原因 + 「重試」）。 |
| 輸入區 | 多行輸入框：Enter 送出、Shift+Enter 換行、最多 8 行後捲動；右下送出（琥珀圓形）／停止切換。回應中輸入框仍可打字，送出鈕換成停止，旁邊提示「回應結束後才能送出」（ui-spec 2.2）。 |
| 狀態列 | `輸入 N · 輸出 N · 回合 N · 上下文 [進度條] 14K / 1M`（ui-spec 2.3）；數字取最後一次回應的 usage，上下文 = 該次的 input + output token。連線狀態點 M1 先不做。 |

- ViewModel：`WorkbenchViewModel`（頂端、輸入、狀態列、指令）、`MessageViewModel`（單則訊息，串流中會持續更新 `Text`、`Thinking`）。用 CommunityToolkit.Mvvm 的 `[ObservableProperty]`、`[RelayCommand]`；送出與停止是同一個 `CancellationTokenSource`。
- 執行緒：`ConversationService` 在背景執行，事件透過 `IUiDispatcher`（App 內的小介面，實作是 `Dispatcher.UIThread.Post`）回到 UI 執行緒更新，ViewModel 不直接碰 Avalonia 的 Dispatcher，所以 ViewModel 測試不需要 Avalonia。
- 串流文字更新頻率：每個 `TextDelta` 直接 append。若實測卡頓，再改成 50 ms 合併一次（記在實作紀錄）。
- 自動捲動：在最底部時跟著捲；使用者往上捲之後就不自動捲，回到底部再恢復。
- 啟動：載入 `GetLatestAsync()` 的對話與訊息；沒有對話就建一則空的。
- 所有新文字放 `Strings.resx`，顏色只用 token（1A 的色碼檢查測試會守住）。

#### 缺金鑰對話框（Q12）

- `ApiKeyDialog`：說明文字、密碼欄位（`PasswordChar`）、「儲存」「取消」。不顯示既有金鑰。
- 觸發：`Failed` 的 Kind 是 `MissingApiKey`、`InvalidApiKey`、`SecretUnreadable`。後兩者的說明文字不同（「金鑰無效，請重新輸入」／「金鑰無法讀取，請重新輸入」）。
- 儲存：`ISecretStore.SetAsync(connection/dev-anthropic/api-key)`，成功後自動 `RetryAsync`。取消：保留錯誤列。
- 輸入的字串去掉前後空白；空字串不能儲存。金鑰不寫進 log、ViewModel 不保留（存完就清空欄位）。
- M2 由設定頁取代，所以這個對話框的程式碼盡量獨立，之後整個刪掉。

#### 專案與套件

- `Lumi.Infrastructure`：加 `Microsoft.Data.Sqlite`、`AddLumiInfrastructure()`。
- `Lumi.Core`：`AddLumiCore()` 註冊 `ConversationService`。
- 新增 `tests/Lumi.Infrastructure.Tests`。
- 架構測試：只有 `Lumi.Infrastructure` 引用 `Microsoft.Data.Sqlite`。

#### 測試

- `Lumi.Infrastructure.Tests`（暫存資料夾的真 SQLite 檔）：建立／讀回對話、訊息順序、`content_json` 來回（含 thinking signature、中文、emoji）、`GetLatestAsync` 依更新時間、重開連線後資料還在、migration 從空資料庫建到最新、資料庫版本比程式新時拒絕開啟。
- `Lumi.Core.Tests`（假 Provider + 記憶體 store）：送出時先存使用者訊息、`Completed` 存助理訊息、`Failed` 不存、取消時存 `Stopped`（有文字才存）、空訊息略過、連續同角色合併、`RetryAsync` 不新增使用者訊息、第一則訊息設標題、同時兩次呼叫被擋。
- `Lumi.App.Tests`（假 `ConversationService` 依賴、同步的 `IUiDispatcher`）：送出後輸入框清空、回應中送出停用且停止可用、停止後恢復、收到缺金鑰 → 要求開對話框、狀態列數字、Enter／Shift+Enter 的判斷。
- 手動驗收（需要你的金鑰，會產生少量 API 費用）：實際對話數輪、中途停止、重開程式後對話還在、缺金鑰與錯誤金鑰的對話框。

### 工作項目

- [x] **D1 Store 介面**：`IConversationStore`、`ChatDefaults`。
- [x] **D2 SQLite**：`Lumi.Infrastructure` 套件、`LumiPaths`、schema 與 migration、`SqliteConversationStore`、`AddLumiInfrastructure()`。
- [x] **D3 Infrastructure 測試**：`Lumi.Infrastructure.Tests` 加進方案。
- [x] **D4 對話流程**：`ConversationService`（送出、重試、組請求的略過與合併、存檔、取消）與測試。
- [x] **D5 ViewModel**：`WorkbenchViewModel`、`MessageViewModel`、`IUiDispatcher`；測試。
- [x] **D6 工作台畫面**：訊息列表、思考過程（最簡版）、錯誤列、輸入框鍵盤行為、送出／停止、狀態列、新對話、自動捲動；新字串進資源檔。
- [x] **D7 缺金鑰對話框**：`ApiKeyDialog` 與觸發、儲存後自動重送。
- [x] **D8 啟動流程**：載入最近一則對話；DI 組裝。
- [x] **D9 架構測試**：`Microsoft.Data.Sqlite` 只在 Infrastructure。
- [x] **D10 手動驗收**（Mac 已通過；Windows 待驗）：見下方驗收標準。
- [x] **D11 文件**：README、主計畫 M1 狀態、本檔實作紀錄。

### 驗收標準

主計畫 M1 的完成標準：能連續對話、中途停止、重開程式後對話還在；Core 不引用 Avalonia；乾淨的 Windows 11 上沒有出現細明體或正黑體。具體：

- [x] `dotnet build` 零警告，`dotnet test` 全過。
- [x] 實際和 `claude-opus-5-5` 連續對話至少 3 輪，文字邊收邊顯示，中文與 emoji 正常。
- [x] 回應中按停止：立刻停下，已收到的文字留著並標示「你已停止這次回應」，之後可以繼續對話。
- [x] 關掉再打開，最近一則對話完整還原（含已停止的訊息與思考過程）。
- [x] 沒有金鑰時送出會跳對話框，存好後自動送出。
- [ ] 錯誤金鑰會再跳一次並說明是金鑰無效（ViewModel 測試涵蓋，實機未測）。
- [x] 狀態列的 token 數字與 API 回傳的 usage 一致。
- [x] 資料庫裡搜尋不到金鑰字串。
- [ ] Windows 11 上同樣可以完成以上操作（M1 整體的 Windows 驗證一起做）。

## 實作紀錄

### 1A（2026-10-18）

- **字型改用靜態檔**（偏離計畫）：A3 驗證發現 Avalonia 11.3 不套用可變字型的 `wght` 軸，Noto Sans TC 與 JetBrains Mono 在 500／700 仍是 400。
  改成各放 Regular／Medium／Bold 靜態檔（用 fonttools 從可變字型切出，每個 Noto 檔約 7 MB）；IBM Plex Sans 本來就用官方靜態檔。
  已更新 `docs/ui-spec.md` 1.1、`Assets/Fonts/LICENSES/README.md` 與主計畫決策紀錄。
- **多了 `Lumi.App.Tests`**：計畫只列兩個測試專案，但 A9 要求 `EmojiText.IsEmoji` 的單元測試，放在新的 App 測試專案。
- **`tests/Directory.Build.props`**：測試專案共用 `Using Xunit`。
- **Fluent 強調色**：`ColorPaletteResources` 無法取 ThemeDictionaries 的值，所以 `Tokens.axaml` 另有一個不分淺深色的 `LumiAccentFixedColor`，供 `App.axaml` 引用，仍符合「`Tokens.axaml` 以外沒有色碼」。
- **NoWarn AVLN3001**：Window 由 DI 以建構子注入 ViewModel，不走執行階段 XAML 載入器，Avalonia 的這則警告不適用。
- **`EmojiTextBlock`** 直接繼承 `TextBlock` 並在 `Text` 變更時呼叫 `EmojiText.Apply`（不另開新屬性），可直接取代 `TextBlock` 使用 `Text` 繫結。
- **`Strings.cs`**：以手寫的 `ResourceManager` 包裝取代 resx 設計工具產生的程式碼（CLI 建置不會自動產生）；新增字串時 resx 與 `Strings.cs` 要一起改。
- **Mac 實測**：字重、❤️‍🔥 不被拆開、深色／淺色跟隨系統，且 App 執行中切換系統外觀會即時換色。
- **尚未驗證**：Windows 11 全項（字型沒有退回細明體／正黑體、100%／125% 縮放、主題即時切換）。

### 1B（2026-10-18）

- **`SecretKey` 格式**：計畫寫「小寫英數、- 與 /」，實作進一步規定每段只能有單一的 `-`、不能以 `-` 或 `/` 開頭結尾、不能連續分隔符號、最長 128。
- **`EncryptedFileSecretStore` 抽出來**（計畫沒有）：Windows 的檔案處理（原子寫入、序列化、損毀處理）與加密分開，加密由 `ISecretProtector` 提供。
  `WindowsDpapiSecretStore` 只是繼承它並提供 DPAPI 的 protector。這樣檔案處理的邏輯在 Mac 上就能用假加密跑完整測試，Windows 只需要驗 DPAPI 本身。
- **契約測試與記憶體實作放在 `Lumi.Platform.Tests`**，不放 `Lumi.Core.Tests`：三個實作共用同一個抽象類別，不必讓測試專案互相參考。`Lumi.Core.Tests` 放 `SecretKey`／`SecretValue` 的單元測試。
  套件 `Xunit.SkippableFact`：不支援的平台是「略過」而不是假裝通過。
- **Keychain 呼叫一律 `Task.Run`**：授權視窗可能卡住，不能擋住呼叫端的執行緒。`ExistsAsync` 只要求屬性、不要求資料，確認存在不需要授權讀取內容。
- **App 已呼叫 `AddLumiPlatform()`**，但 1B 沒有任何地方使用 `ISecretStore`，1D 才會用。
- **架構測試**加上「P/Invoke 只在 Platform」（以反射找 `PinvokeImpl` 方法，並有對照組證明偵測有效；另掃原始碼的 `DllImport`／`LibraryImport`）。
- **B8 做法**：用一個放在專案外的小型主控台程式寫入項目，用 `security find-generic-password` 確認 service／account／label，再用另一個程序讀回，最後刪除。Mac 驗證通過，沒有殘留項目。
- **尚未驗證**：Windows 的 DPAPI 契約測試（13 項目前在 Mac 被略過）、`secrets.json` 的實際內容；Mac 授權視窗按「拒絕」的實機行為。

### 1C（2026-10-18）

- **確認 Q4 後改用官方 SDK**：NuGet `Anthropic` 12.58.0，只在 `Lumi.Providers` 引用；走 `client.Beta.Messages`（不是 `client.Messages`），因為 server-side fallback（Q9）的 `Fallbacks` 欄位只在 beta 端點存在。
- **官方 SDK 的命名比文件描述的更不一致**，以下是實測（用故意寫錯型別讓編譯器報出真正型別名稱的方式逐一確認，不是用反射探針）：
  - `content_block_start` 事件的 `ContentBlock` 屬性型別就叫 `ContentBlock`（在 `Beta.Messages` 命名空間下，不是 `BetaContentBlock`），但它的 `TryPick*` 方法卻是 `TryPickBetaText`／`TryPickBetaThinking`／`TryPickBetaRedactedThinking`（Beta 前綴只出現在方法名稱，不在型別名稱）。`content_block_delta` 的 `Delta` 聯集則相反：型別和方法都沒有 Beta 前綴（`TryPickText`／`TryPickThinking`／`TryPickSignature`）。
  - `message_delta` 的 `stop_reason` 是 `ApiEnum<string, BetaStopReason>`，呼叫 `.ToString()` 印出的是型別資訊，不是 wire 值；要用它的隱含轉換（`(string)` 或直接指派給 `string`）才拿得到 `"end_turn"`／`"refusal"` 這些實際字串。第一版用了 `.ToString()`，被 fixture 測試抓到（`StopReason` 永遠是 `Other`）。
  - `Fallbacks` 沒有 `"default"` 純量形式的建構方式（至少目前這版 SDK 找不到），改用文件給的陣列形式：`Fallbacks = [new(Model.ClaudeOpus4_8)]`，候補模型固定是 `claude-opus-4-8`（官方文件：cyber 類別的拒絕會建議導到它）。
  - `AnthropicRateLimitException`、`AnthropicApiException` 目前沒有公開的 `RetryAfter`／`RequestId` 屬性（編譯器直接否認這兩個成員存在），`ChatFailure` 的這兩個欄位先留 null，不影響 `ChatFailureKind` 的分類。
- **串流中途的 `error` 事件，SDK 一律包成 `AnthropicSseException`**，把 `overloaded_error` 這些細節壓平成同一種例外。伺服器原始的錯誤 JSON 會原封不動放在 `Exception.Message`（實測得到，不是文件保證的行為），所以 `AnthropicErrorMapper` 從訊息裡撈 `error.type` 字串來還原 `Overloaded` 分類；撈不到就當一般的 `StreamError`。訊息格式是 `"...: '{...}'"`（結尾還有一個單引號），所以是從第一個 `{` 切到最後一個 `}`，不能切到字串尾端。
- **`StreamAsync` 的 enumerator 延後到迴圈內、在 try 裡才建立**（而不是在迴圈外先 `GetAsyncEnumerator()`）：`CreateStreaming(...).GetAsyncEnumerator(...)` 會不會立刻送出 HTTP 請求、因而立刻丟例外，是 SDK 的實作細節，不能假設一定是延遲求值；建在迴圈外會讓「請求一開始就失敗」繞過 try/catch，直接讓例外往外丟，違反「例外一律轉成 `Failed`」的驗收標準。
- **fixture 用純文字 `.sse`／`.json` 檔案**（照計畫），但用假的 `HttpMessageHandler` + 會把每次讀取限制在 1／3／7 byte 的 `Stream` 來餵，而不是直接餵位元組給一個手寫的 SSE 解析器——因為改用官方 SDK 後，SSE 解析本身是 SDK 的責任（`System.Net.ServerSentEvents`），我們的測試驗證的是「就算底層的網路讀取被切得很碎，最後組出來的中文與組合 emoji 還是完整的」這件事有沒有在整條路徑上成立。
- **`fallback-block.sse` 的區塊格式**：Anthropic 文件沒有給完整 schema，但這個 skill 裡的 jq 範例明確示範了 `content[] | select(.type=="fallback")` 配上 `.from.model`／`.to.model`，所以照這個格式寫 fixture（agent.md 第 3 節：沒有規格不要自己發明格式——這裡的「規格」來源是官方文件引用的範例，不是憑空猜的）。測試只驗證不崩潰、存成 `ProviderBlock`、後面的文字正常，不驗證 `RawJson` 的確切序列化形狀（那是 SDK 內部的序列化細節，沒有把握精確斷言）。
- **`InternalsVisibleTo`**：`Lumi.Providers` 加了 `AssemblyInfo.cs` 讓 `Lumi.Providers.Tests` 能測內部的 `AnthropicRequestConverter`／`AnthropicStreamAccumulator`／`AnthropicErrorMapper`，這三個類別本來就不該是公開 API。
- **尚未做**：C11（錄製工具）依 Q11 先不做；Windows 全部待驗（1A、1B、1C 共通）。

### 1D（2026-10-18）

- **沒有做 `IUiDispatcher`**（計畫有）：命令在 UI 執行緒上執行，`await foreach` 會回到 UI 的同步內容，事件本來就在 UI 執行緒處理，不需要另外轉送。ViewModel 測試在沒有同步內容的 xUnit 下直接跑，一樣不需要 Avalonia。
- **錯誤列的文字直接用 `ChatFailure.Message`**（Providers 裡寫好的繁中），沒有在 App 依 `ChatFailureKind` 對應資源檔。M2 做完整錯誤列（換模型重試）時再搬進資源檔。金鑰相關的三種錯誤走對話框，文字在資源檔。
- **思考過程**（Q14）：Avalonia 預設的 `Expander`，標題只有「思考過程」，沒有「· N 秒」。
- **視窗標題**：第一則訊息送出時跟著改成對話標題（和資料庫一致）。第一版漏了，寫測試時發現。
- **啟動時**沒有任何對話會建一則空白的「新對話」，所以第一次開啟就會建立資料庫檔案。
- **串流文字**每個 `TextDelta` 直接接到字串後面，`EmojiTextBlock` 每次重算整段；長回應是 O(n²)，M1 先這樣，實測卡頓再改成合併更新。
- **Enter／Shift+Enter** 是 code-behind 的視覺行為（tunnel 攔截），沒有單元測試，靠手動驗收。
- `Lumi.App.Tests` 複製了一份 `Lumi.Core.Tests` 的假 store 與假 Provider（測試專案不互相參考，同 1B 的做法）。
- `AnthropicDevOptions.ApiKeySecret` 改成公開常數，金鑰對話框與 Provider 共用同一個名稱。
- 連線狀態點（狀態列右側）照計畫 M1 不做。
- **D10 手動驗收（2026-10-18，Mac）**：使用者實測通過——缺金鑰對話框存好後自動送出、連續對話 3 輪以上（中文與 emoji 正常）、中途停止、重開後還原、狀態列 token 數字。
  另外確認：金鑰在登入鑰匙圈（service `io.github.neillog.lumi`、account `connection/dev-anthropic/api-key`）；資料庫有 5 則使用者訊息、4 則完成與 1 則已停止的助理訊息。
  資料庫搜尋 `sk-ant`：第一次用 `grep -c` 回報 1 筆，但之後用原始位元組逐一比對、`grep -a -o`、`strings` 三種方式都是 0 筆，也不在任何訊息或標題裡，第一次的結果無法重現。程式上也沒有任何路徑會把金鑰寫進資料庫。（M2 時發現Claude Code 執行指令的環境裡，`grep` 是轉給 `ugrep` 的 shell 函式（不是你終端機的設定），很可能是它對二進位檔的計數方式造成的，見 `plan/02` 的 2A 實作紀錄。）
- **尚未驗證**：Windows 全部（「錯誤金鑰會再跳一次對話框」這項 Mac 也沒有實測，邏輯由 ViewModel 測試涵蓋）。

