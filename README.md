# Lumi — 桌面 AI 小精靈

Lumi 是一個跨平台（Windows / macOS）的桌面 AI 助理。使用者自帶 API 金鑰（BYOK），支援 Anthropic 與 OpenAI 相容格式的模型。

它有兩種模式，共用同一則對話：

- **工作台**：完整的對話視窗。可以切換模型、加入檔案或資料夾當上下文、語音輸入，並在模型選單裡顯示本對話的 token 用量與回合數。
- **小精靈**：一個住在桌面上的透明角色（第一個角色是會呼吸的光球）。游標靠近時才浮出輸入列，用簡短的氣泡回答，並用動畫表現聆聽、思考、呼叫工具、完成等過程。

> 目前狀態：M0～M8 與 M12（UI 重新設計）程式完成，多數待人工驗收與 Windows 驗證（各里程碑見 `plan/00-master-plan.md`）。M9「歷史對話與完成度」：歷史對話的搜尋、分組、改名、刪除，視窗位置記憶，啟動失敗說明，log 程式完成並待人工驗收；Windows 國旗 emoji 的判定等待實機結果。M10「WPF 小工具的工具移植」：10A～10E 程式完成，待人工驗收。M14「長期記憶」：程式完成，待人工驗收（`plan/14-long-term-memory.md` 第 14F 節）。M15「工具清單」與 M16「MCP 伺服器」：程式完成，待人工驗收（`plan/manual-verification.md` 的 M15、M16）。M17「網路基礎與網頁讀取」：程式完成，待人工驗收（需要公司網路，`plan/manual-verification.md` 的 M17）。M18「瀏覽器操作」：程式完成，待人工驗收（需要公司的 Windows＋Edge，`plan/manual-verification.md` 的 M18）。M19「網站爬取」：程式完成，待人工驗收（`plan/manual-verification.md` 的 M19）。M20「設定頁補強、App 圖示、單一檔案發佈」：程式完成，待人工驗收（`plan/manual-verification.md` 的 M20；發佈方式見 `plan/20-settings-release.md`）。

## 特色

- **BYOK、多模型**：可以新增多組連線與模型，工作台和小精靈可以各自用不同的模型
- **工具呼叫**：內建工具採用統一的 `ITool` 介面，以 DI 注入；寫入類工具會先詢問使用者
- **檔案與資料夾上下文**：資料夾不會整包塞進 prompt，而是讓模型透過「限定範圍」的工具按需讀取
- **檔案整理與文件讀取**：在你加入的範圍內建立、追加、複製、移動檔案（覆寫每次都詢問）；可讀 Word、Excel、PowerPoint、PDF 的文字
- **筆記與提醒**：每天一個 Markdown 的筆記（相容 Obsidian）、提醒、週期排程與計時器（Lumi 執行中才會觸發）
- **長期記憶**：Lumi 記得你長期不變的事（身分、偏好、工作、專案），之後的對話都會帶上；可以在設定 › 記憶檢視、編輯、刪除，也能整個關閉，寫入記憶一律先詢問你。記憶存在這台裝置，並會隨每則訊息送到你選的模型供應商。
- **語音輸入**：工作台與小精靈都能錄音，透過 Whisper 服務轉成文字（以 `ISpeechToText` 注入，可替換）；服務端點與 Token 在設定 › 語音輸入，Token 只存在這台裝置的安全儲存
- **人設**：可設定個性、積極度與自訂人設；小精靈模式會自動用更短的方式回答
- **可擴充的小精靈角色**：殼與角色分離，支援程式繪製的角色，以及「美術素材 + JSON」的角色
- **全部程式繪製**：光球不使用任何圖片，閒置時自動降低幀率以節省電力
- **歷史對話**：抽屜可以搜尋標題與訊息內容、依日期分組、改名、刪除（直接刪除，不做垃圾桶；刪除會一併清掉附件）、載入更多
- **工具清單**：設定 › 工具列出 Lumi 所有能用的工具，依群組分類；可以逐一或整組啟用、關閉（新對話才生效），並把寫入、網路類工具設成「永遠允許」（破壞性工具與寫入記憶一律先問你）。
- **MCP 伺服器**：可以設定多個 MCP 伺服器（本機指令或遠端 HTTP），它們的工具和內建工具一樣出現在工具清單、一樣先確認；支援貼上 Claude Desktop／Claude Code 的設定 JSON 匯入。環境變數與標頭的值只存在這台裝置。
- **網路工具與 Proxy**：讀網頁（轉成 Markdown，PDF／Office 也能讀）、列出連結、讀 RSS、呼叫 API（送出資料的另外一個工具，每次都先問）、下載檔案到你加入的資料夾、網路診斷、**爬取網站**（同一個網站最多 20 頁，存成 Markdown 與目錄；內外網都可以，需要登入的網站自動改用瀏覽器）。Proxy 是選擇性的（跟隨系統／PAC 網址／手動／不使用，支援 AD 帳號驗證），設定套用到所有連線。Lumi 只會自動連到對話中出現過的網站，其他網站先問你。**Lumi 不提供網路搜尋**：請直接給 Lumi 網址。
- **瀏覽器操作**：Lumi 可以用電腦上已安裝的 **Edge（優先）或 Chrome** 開網頁、點擊、輸入、擷取內容、截圖、下載與上傳（不需要另外安裝 Node 或設定 MCP）。登入狀態保留在 Lumi 專屬的瀏覽器設定檔，不讀你平常瀏覽器的 Cookie；遇到登入或驗證碼，Lumi 不會替你輸入密碼，而是打開視窗請你自己處理。預設背景執行，可以在設定 › 網路打開「顯示瀏覽器視窗」看到每一步操作。需要電腦上有 Edge 或 Chrome（Windows 內建 Edge）。
- **視窗記憶與啟動說明**：工作台記得上次的位置、大小與最大化狀態（螢幕變動時退回置中）；啟動失敗時顯示說明視窗，記錄寫在使用者資料夾的 `logs/lumi-yyyy-MM-dd.log`

## 技術棧

.NET 10 · Avalonia 11.3 · CommunityToolkit.Mvvm · Microsoft.Extensions.Hosting · SQLite · System.Text.Json · Markdig（Markdown 解析）

## 專案結構

```
agent.md                    給 AI agent 與開發者的開發指引（必讀）
plan/                       工作計畫
  00-master-plan.md         主計畫：里程碑、決策紀錄、待確認事項
  01-skeleton-chat.md       M1 細部計畫：方案骨架、金鑰儲存、Anthropic、工作台
  02-models-settings-workbench.md  M2 細部計畫：模型管理、設定頁、工作台完整介面
  03-tools-loop.md          M3 細部計畫：Tool 框架、對話迴圈、權限確認、工具卡片
  04-files-context.md       M4 細部計畫：檔案與資料夾上下文、範圍限定的檔案工具
docs/
  ui-spec.md                UI 規格（字型、主題系統、RWD、元件行為；設計原則以 plan/ui-guide/ui-guide.html 為準）
  sprite-dev-guide.md       小精靈角色開發指南
src/
  Lumi.Core/                資料模型、核心介面、對話迴圈（不依賴 UI）
    Network/                系統網路狀態（只在失敗後補說明文字）
  Lumi.Providers/           Anthropic（官方 SDK）、OpenAI 相容（M2）
  Lumi.Infrastructure/      SQLite（對話與訊息）、設定檔
    Logging/                寫進使用者資料夾 logs/ 的記錄檔
    Startup/                啟動失敗的原因分類（資料庫被鎖、權限、磁碟、較新 schema、設定檔）
  Lumi.Platform/            原生 API、金鑰安全儲存（DPAPI、Keychain）、麥克風錄音（waveIn、AudioQueue）
  Lumi.Speech/              Whisper 代理客戶端（上傳、回應解析、錯誤對應、測試連線）
  Lumi.Tools/               內建工具（時間、計算、範圍內檔案讀寫、筆記、提醒與計時器、文件讀取）
  Lumi.Documents/          讀取 Word、Excel、PowerPoint、PDF 的文字（OpenXml、PdfPig；只透過 Core 的 IDocumentReader 對外）
  Lumi.Browser/            瀏覽器自動化（Playwright 只在這裡引用）：偵測 Edge／Chrome、啟動參數、瀏覽器程序與分頁管理
  Lumi.Web/                Proxy（含 PAC）、網路工具的 HTTP 用戶端、HTML 轉 Markdown、RSS、網路診斷（解析套件只在這裡引用）
  Lumi.Mcp/                MCP 伺服器的連線與工具 Adapter（官方 SDK 只在這裡引用；只透過 Core 的 IMcpToolSource 對外）
  Lumi.App/                 Avalonia 介面與 DI 組裝（唯一的 composition root）
tests/
  Lumi.Core.Tests/
  Lumi.Platform.Tests/      金鑰儲存的契約測試（記憶體、DPAPI、Keychain）
  Lumi.Providers.Tests/     Anthropic Provider 的 fixture 測試（不呼叫真實 API）
  Lumi.Speech.Tests/        Whisper 客戶端與解析器（假的 HttpMessageHandler，不呼叫真實服務）
  Lumi.Infrastructure.Tests/ SQLite 實作（暫存資料夾的真資料庫檔）
  Lumi.Tools.Tests/         工具與運算式解析器（含惡意輸入、路徑逃逸）
  Lumi.Documents.Tests/    四種文件格式的讀取（測試檔由程式產生，不提交二進位檔）
  Lumi.Browser.Tests/      偵測與啟動參數的單元測試；真瀏覽器測試需要 LUMI_BROWSER_TESTS=1（本機 HTTP 站，不連真實網站）
  Lumi.Web.Tests/          Proxy／PAC、HTTP 用戶端、內容轉換（本機 TcpListener 與假的 HttpMessageHandler，不連真實網站）
  Lumi.Mcp.Tests/          MCP 管理員、工具 Adapter、結果格式（用假連線與記憶體內的 SDK 伺服器）
  Lumi.App.Tests/           ViewModel、Markdown 解析與渲染（含 Avalonia.Headless）、EmojiText、字串資源
  Lumi.Architecture.Tests/  相依規則、色碼檢查、P/Invoke 與供應商 SDK 的歸屬
poc/
  Lumi.OrbPoc/              光球 POC（驗證小精靈視窗的可行性，不在 Lumi.sln 內）
  Lumi.EmojiPoc/            Emoji 驗證頁（Windows 國旗判定，不在 Lumi.sln 內）
  Lumi.BrowserPoc/          瀏覽器 POC（Playwright 驅動 Edge／Chrome 的關鍵假設，不在 Lumi.sln 內；公司 Windows＋Edge 的驗證步驟見 README）
```

之後會加入的專案（詳見 `agent.md`）：

```
Lumi.Sprites.Abstractions   小精靈角色介面
Lumi.Sprites.*              各個角色
```

## 執行工作台

需要 [.NET 10 SDK](https://dotnet.microsoft.com/download)。

```bash
dotnet build            # 零警告
dotnet test
dotnet run --project src/Lumi.App
```

## 試跑光球 POC

需要 [.NET 10 SDK](https://dotnet.microsoft.com/download)。

```bash
cd poc/Lumi.OrbPoc
dotnet run -c Release
```

操作方式：
- 游標靠近時睜眼，並浮出輸入列
- 點一下切換示範狀態
- 長按可以「戳」，拖曳可以移動
- 右鍵開啟選單（大小、眼睛、診斷 HUD 等）

完整的驗證清單見 `poc/Lumi.OrbPoc/README.md`。

## 開發流程

1. 先讀 `agent.md`
2. 在 `plan/00-master-plan.md` 找到下一個里程碑
3. 寫細部計畫 `plan/NN-名稱.md`，確認後才開工
4. 完成後更新計畫狀態與實作紀錄

## 已知限制

- **Windows 國旗 emoji 待驗證**：Segoe UI Emoji 沒有國旗字形，預期會顯示成兩個字母（TW、JP）。判定由 Windows 實機執行 `dotnet run --project poc/Lumi.EmojiPoc` 完成，結果填進 `poc/Lumi.EmojiPoc/README.md`；結果出來之前不附 emoji 字型。
- 離線判斷只在請求失敗後才補說明，不主動偵測網路。
- 歷史搜尋只比對標題與訊息文字（不分大小寫，英文以外的大小寫折疊不做），命中字只在清單中加粗，不捲動到內文。

## 第三方資料

- 模型價目表：[LiteLLM](https://github.com/BerriAI/litellm) 的 `model_prices_and_context_window.json`（MIT 授權），快照放在 `src/Lumi.Web/Pricing/`，用來自動帶入模型單價與上下文上限；快照日期見同資料夾的 `litellm-model-prices.date.txt`。

## 授權

尚未決定。
