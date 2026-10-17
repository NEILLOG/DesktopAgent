# 01 方案骨架與基本對話

- 狀態：1A、1B 已確認，可以開工；1C、1D 只列大綱，輪到時再展開並確認
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
| 1C | Core 模型與 Anthropic Provider | `ChatEvent`、訊息與對話模型、`IChatProvider`、SSE 解析、Anthropic 串流與 usage、fixture 測試 | 1B |
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
| Q3 | M1 用哪個模型、Base URL？ | 寫在 `appsettings.json` 的 `Lumi:Dev:Anthropic`（`BaseUrl`、`Model`、`MaxTokens`、`ContextWindow`），只放非機密值；M2 由 `IModelCatalog` 取代。 | 待確認 | 1C |
| Q4 | Anthropic 用官方 SDK 還是自己寫？ | 自己寫：`IHttpClientFactory` typed client + 自寫 SSE 解析。 | 待確認 | 1C |
| Q5 | 「Core 不引用 Avalonia」怎麼守？ | 架構測試。 | ✅ 已確認 | 1A |
| Q6 | Noto Sans TC 可變字重在 Avalonia 能不能正確出 400／500／700？ | 1A 第一項就做驗證（A3）。 | 待驗證 | 1A |
| Q7 | 金鑰實作放哪個專案？ | 放 `Lumi.Platform`（見 1B 設計「放哪一層」）。`agent.md` 第 5 節已同步修改。 | ✅ 已確認 | 1B |
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

- [ ] **B1 Core 介面**：`ISecretStore`、`SecretKey`（含格式驗證）、`SecretValue`、`SecretStoreException`；單元測試（名稱驗證、`ToString` 遮蔽）。
- [ ] **B2 契約測試**：抽象測試類別 `SecretStoreContractTests`，涵蓋：讀不存在的回 null、寫後讀、覆寫、刪除、刪除不存在的回 false、中文與 emoji 內容、長字串（4 KB）、多個 key 互不影響。
- [ ] **B3 記憶體實作**：放測試專案，跑契約測試。
- [ ] **B4 建立 `Lumi.Platform`**：專案、`AddLumiPlatform()`、`SecretStoreOptions`；架構測試加上「P/Invoke（`DllImport`／`LibraryImport`）只能出現在 `Lumi.Platform`」。
- [ ] **B5 Windows DPAPI 實作**：原子寫入、寫入序列化、檔案損毀或解密失敗時丟 `SecretStoreException`。
- [ ] **B6 macOS Keychain 實作**：CoreFoundation／Security 的 P/Invoke（用 `LibraryImport`）、`SafeCFHandle`、錯誤碼對應。
- [ ] **B7 平台測試**：`Lumi.Platform.Tests`，用測試專用的檔案路徑與 service 名稱跑契約測試；在非對應平台上自動略過。Mac、Windows 各跑一次。
- [ ] **B8 手動驗證**：Mac 在「鑰匙圈存取」看得到項目、名稱正確；Windows 的 `secrets.json` 裡沒有明文；兩邊重開程式後都讀得到。
- [x] **B9 文件**：`agent.md` 第 5 節已依 Q7 更新；主計畫已把 `ISecretStore` 從 M2 移到 M1，決策紀錄已加。

### 驗收標準

- [ ] 契約測試在記憶體、DPAPI（Windows）、Keychain（Mac）三個實作都通過。
- [ ] 在檔案系統、log 中搜尋測試用的金鑰字串，找不到明文。
- [ ] `Lumi.Core` 只有介面，沒有任何平台相關程式碼；P/Invoke 只在 `Lumi.Platform`（架構測試保證）。
- [ ] 讀取失敗（例如 Mac 上在授權視窗按「拒絕」）不會讓程式崩潰，回報的錯誤訊息不含金鑰。

---

## 1C Core 模型與 Anthropic Provider（大綱，輪到時展開）

- Core 模型：`ChatMessage`（角色、內容區塊）、`Conversation`、`ChatRequest`（模型、system、訊息、`MaxTokens`）、`ChatEvent` 的八種事件（`agent.md` 第 7 節），M1 實際會發出 `TextDelta`、`ThinkingDelta`、`Usage`、`Completed`、`Failed`
- `IChatProvider`：`IAsyncEnumerable<ChatEvent> StreamAsync(ChatRequest, CancellationToken)`
- SSE 解析器：以完整事件為單位、UTF-8 有狀態解碼（放 Providers 共用，OpenAI 相容在 M2 也會用）
- Anthropic Provider：`POST {BaseUrl}/v1/messages`、`stream: true`、`x-api-key`（從 `ISecretStore` 取）、`anthropic-version`；處理 `message_start`、`content_block_*`、`message_delta`、`message_stop`、`ping`、`error`；usage 合併 `message_start` 與 `message_delta` 的數字
- HTTP log 遮蔽 `x-api-key`、`Authorization` 標頭
- 錯誤：HTTP 4xx／5xx、串流中的 `error` 事件、取消，都轉成 `Failed` 或正常結束；缺金鑰時回 `Failed`（原因：未設定金鑰）
- 測試：錄製好的 SSE fixture（含中文、emoji 跨 chunk 切斷、thinking、錯誤事件、中途取消）

## 1D 工作台最小版與 SQLite（大綱，輪到時展開）

- `IConversationStore` + SQLite 實作：`conversations`、`messages` 兩張表、schema 版本表；資料庫放使用者資料夾（Mac：`~/Library/Application Support/Lumi`，Windows：`%APPDATA%\Lumi`）
- 最小對話流程（Core）：組請求 → 呼叫 Provider → 轉發事件 → 完成後存檔；停止時保留已收到的文字並標記「已停止」
- 工作台：訊息列表（`EmojiTextBlock`，M1 先純文字，Markdown 是 M2）、輸入框（Enter 送出、Shift+Enter 換行）、送出／停止切換
- 缺金鑰時：最小的「輸入 Anthropic API 金鑰」對話框（密碼欄位、儲存、取消），存進 `ISecretStore`；M2 由設定頁取代
- 啟動時載入最近一則對話
- 驗收：主計畫 M1 的完成標準

---

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
