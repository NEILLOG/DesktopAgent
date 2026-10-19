# 00 主計畫：Lumi 施工方向

這份是總覽：只列里程碑的目標、範圍和完成標準。
每個里程碑開工前，另外寫一份細部計畫 `plan/NN-名稱.md`（格式見文末），細部計畫寫好、確認之後才動工。

狀態圖示：⬜ 未開始　🟨 進行中　✅ 完成　⏸ 暫緩（等待外部資訊）

---

## 里程碑總覽

| # | 里程碑 | 狀態 | 依賴 |
|---|---|---|---|
| M0 | 光球 POC：驗證小精靈視窗 | 🟨 Mac 已驗證，Windows 待測 | — |
| M1 | 方案骨架與基本對話 | 🟨 Mac 完成，待 Windows 驗證 | M0 |
| M2 | 模型管理、設定頁與工作台完整介面 | 🟨 2A～2E 程式完成，待手動驗收（`plan/02-models-settings-workbench.md`） | M1 |
| M3 | Tool 框架與對話迴圈 | 🟨 3A～3C 程式完成，待手動驗收（`plan/03-tools-loop.md`） | M1 |
| M4 | 檔案與資料夾上下文 | 📝 細部計畫待確認（`plan/04-files-context.md`） | M3 |
| M5 | Prompt 分層與人設 | ⬜ | M1 |
| M6 | 語音輸入（Whisper） | ⏸ 等待回應格式 | M2 |
| M7 | 小精靈殼與光球角色 | ⬜ | M3、M5 |
| M8 | 素材型角色 | ⬜ | M7 |
| M9 | 歷史對話與完成度 | ⬜ | M2 |
| M10 | WPF 小工具的工具移植 | ⬜ 細節暫不展開 | M3 |
| M11 | 打包與發佈 | ⬜ | 全部 |

M2、M3、M5 都只依賴 M1，可以依需要調整順序。

---

## M0 光球 POC

**目標**：在寫正式程式之前，確認小精靈視窗最沒把握的幾件事。
**位置**：`poc/Lumi.OrbPoc/`，驗證清單在該資料夾的 `README.md`。

- ✅ 透明無框、置頂、程式繪製光球（漸層斷層與硬邊高光已修正）
- ✅ 眼睛、眨眼、wink、被戳、拖曳形變
- ✅ 大小選項：64 / 80 / 104，預設 80
- ✅ Emoji 顯示：用字形叢集切段指定字型（Mac 已驗證）
- ⬜ Windows 全項驗證，特別是點擊穿透（`WS_EX_LAYERED`）與國旗 emoji
- ⬜ 耗能數據：待機、休眠的 CPU 平均值

**完成標準**：README 驗證清單的 Win、Mac 兩欄都填完，失敗項目都有決定好的備案。

## M1 方案骨架與基本對話

**目標**：可以在工作台用 Anthropic 模型進行串流對話。

- 建立方案與各專案，相依方向依 `agent.md` 第 5 節
- 用 Generic Host 組裝 DI；Avalonia 從 Host 取得服務
- 外觀基礎：打包字型與 `LumiUiFont`／`LumiMonoFont` 資源、Light／Dark 主題 token（預設跟隨系統），規格見 `docs/ui-spec.md` 第 1 節
- `ISecretStore`：Windows DPAPI、macOS Keychain（從 M2 提前）
- Core：`ChatEvent`、訊息與對話模型、`IChatProvider`
- Anthropic Provider：串流（SSE）、usage
- 工作台最小版：訊息列表、輸入框、送出與停止
- 訊息顯示經過 `EmojiText`
- SQLite：對話與訊息的基本存取

**完成標準**：能連續對話、中途停止、重開程式後對話還在；Core 不引用 Avalonia；乾淨的 Windows 11 上沒有出現細明體或正黑體。

## M2 模型管理、設定頁與工作台完整介面

**目標**：完成對話一開始設計的工作台互動。

- 連線與模型設定，兩層各自可以新增多筆；測試連線；列出可選模型
- OpenAI 相容 Provider（涵蓋 OpenAI、Ollama、公司 Gateway）
- 頂端模型切換選單；工作台與小精靈可以各自指定預設模型
- 工具列：設定、歷史對話、新對話、切換到小精靈；回應中停用模式切換與模型切換
- 設定頁做成獨立視窗（可與工作台並排）
- 思考過程區塊（預設收合）、錯誤列與重試；設定 › 一般 的主題選擇（跟隨系統／淺色／深色）
- 底部狀態列：輸入與輸出 token、回合數、上下文用量
- Markdown 渲染（同樣經過 emoji 處理）

**完成標準**：不同供應商的多個模型可以切換使用；金鑰不出現在任何檔案或 log。

## M3 Tool 框架與對話迴圈

**目標**：模型可以呼叫自己寫的內建工具。

- `ITool`、`ToolBase<TArgs>`、`IToolRegistry`、`AddLumiTool<T>()`
- Schema 自動產生；權限等級；可用模式；逾時
- 對話迴圈：呼叫工具、回填結果、再次送出，並有最大次數限制
- `IToolApprovalService` 與工作台的確認 UI：確認區嵌在工具卡片內，各權限等級的選項見 `docs/ui-spec.md` 第 5 節
- 工具呼叫卡片：預設收合、標題為「動作 · 對象 · 結果摘要」、連續呼叫合併成群組卡（`docs/ui-spec.md` 第 4 節）
- 兩家 Provider 的 tool 格式轉換
- 先做 2～3 個示範工具（例如目前時間、計算）

**完成標準**：Anthropic 和 OpenAI 相容格式都能完成多輪工具呼叫；寫入類工具會先詢問。

## M4 檔案與資料夾上下文

**目標**：輸入框加入的檔案與資料夾，在同一則對話中都能當作上下文。

- 輸入框的「加入檔案」「加入資料夾」，以 chip 形式常駐顯示，可以移除
- 單一檔案：文字直接放入；圖片用圖片格式送出
- 資料夾：只送檔案樹，內容由範圍限定的 `list_dir`、`read_file`、`search` 工具讀取
- `ToolContext.Scope` 與路徑跳脫防護（要有測試）
- 送出前預估上下文 token

**完成標準**：加入一個大型資料夾不會撐爆 token；工具無法讀取範圍外的檔案。

## M5 Prompt 分層與人設

**目標**：回答風格可以設定，小精靈模式自動變短。

- `IPromptLayer`、`IPromptComposer`，順序與放置位置依 `agent.md` 第 9 節
- 人設設定頁：個性、積極度、自訂人設
- 模式格式層：工作台、小精靈
- `show_details` 工具（小精靈專用）

**完成標準**：切換模式不會讓 prompt 快取失效；小精靈的回答穩定維持在兩句內。

## M6 語音輸入（Whisper）　⏸

**目標**：工作台與小精靈都能用語音輸入。

- `IAudioRecorder`：Windows、macOS 實作，輸出 16kHz 單聲道 WAV
- `ISpeechToText`：公司內部 Whisper 服務的客戶端，規格見 `outter_sys_spec/whisper.md`（內部文件，不納入版控，只存在本機）
- 語音設定頁：API 網址、Token、模式、語言、講者數、逾時、測試按鈕
- 工作台的收音面板（波形、計時、取消／停止並編輯／停止並送出）
- 內網憑證處理方式（待確認）

**暫緩原因**：Whisper 回應格式尚未確認，確認後再寫細部計畫。

## M7 小精靈殼與光球角色

**目標**：小精靈模式正式上線。

- `Lumi.Sprites.Abstractions`：介面、狀態、manifest、`SpriteLimits`
- 小精靈殼：從 POC 的 MainWindow 拆出；靠近才浮現輸入列、氣泡、右鍵選單、位置記憶
- 光球角色：從 POC 的 OrbControl 搬入
- 對話事件轉成角色狀態
- 與工作台共用對話；「在工作台開啟完整內容」
- 角色選擇與狀態預覽（設定頁）

**完成標準**：照著 `docs/sprite-dev-guide.md` 新增一個角色，不需要修改殼。

## M8 素材型角色

**目標**：用美術素材加上 JSON 就能做新角色，不用寫程式。

- manifest 載入與驗證（依 `SpriteLimits` 自動縮放並警告）
- PNG 連續圖、sprite sheet 播放器
- 「角色」資料夾自動載入
- 之後再評估：Lottie（Avalonia 目前只有實驗性套件）

## M9 歷史對話與完成度

- 歷史對話列表、搜尋、刪除、重新命名
- Emoji 字型決策：若 Windows 國旗 emoji 有問題，就附上 Twemoji（COLR 版）並標示授權
- 錯誤處理與重試、離線提示
- 視窗位置與大小記憶

## M10 WPF 小工具的工具移植　（細節暫不展開）

把現有 WPF agent 小工具裡的工具，透過 Adapter 或改寫，接上 Lumi 的 `ITool`：
筆記、提醒、排程、文字檔、Excel、Word、PPT、PDF。
開工前先盤點現有介面，再決定採 Adapter 還是直接改寫。

## M11 打包與發佈

- macOS：.app bundle、`Info.plist`（麥克風權限說明）、簽章與公證
- Windows：安裝程式、開始選單、開機自動啟動選項
- 版本號與更新機制（之後再評估）

## 之後再評估（不在目前範圍）

- MCP 工具 Adapter
- 程式碼型角色外掛（第三方 dll）
- Lottie 角色
- 本機 Whisper（whisper.net）

---

## 決策紀錄

| 日期 | 決策 |
|---|---|
| 2026-10-16 | 只用 Avalonia，不用 WPF；.NET 10 |
| 2026-10-16 | 先做小規模 POC，只驗證小精靈視窗 |
| 2026-10-16 | MVVM 採用 CommunityToolkit.Mvvm |
| 2026-10-16 | 小精靈與工作台共用同一則對話 |
| 2026-10-16 | 先支援 Anthropic 原生與 OpenAI 相容格式 |
| 2026-10-16 | Prompt 分層；模式指示放在最新一則使用者訊息之後，以保留快取 |
| 2026-10-16 | 小精靈回答用三層控制：模式指示、`show_details`、硬性上限 |
| 2026-10-16 | 小精靈拆成「殼」與「角色」；角色分程式型與素材型 |
| 2026-10-16 | 角色尺寸由開發者在 `SpriteLimits` 定義上限 |
| 2026-10-16 | 現在只把 Abstractions 獨立並加版本號；dll 外掛之後再做 |
| 2026-10-16 | Emoji 以字形叢集切段、指定 emoji 字型顯示 |
| 2026-10-16 | 光球預設直徑 80，可選 64 / 80 / 104 |
| 2026-10-16 | WPF 工具移植列入主計畫，細節暫不展開 |
| 2026-10-16 | 工作台 UI 規格定案，記錄在 `docs/ui-spec.md` |
| 2026-10-16 | 回應中（含等待工具確認）不能切換模式與模型；但小精靈回應中可開啟工作台查看（不能送出），權限確認兩邊都能回答 |
| 2026-10-16 | 工具卡片預設收合，標題要明確寫出動作、對象與結果 |
| 2026-10-16 | 設定頁為獨立視窗 |
| 2026-10-16 | 權限確認嵌在工具卡片內；`Destructive` 每次詢問，`Write`／`Network` 可選本次對話都允許 |
| 2026-10-16 | 深淺兩套色系，預設跟隨系統；琥珀 `#F2B544` 為唯一強調色 |
| 2026-10-16 | 字型統一打包：IBM Plex Sans、Noto Sans TC、JetBrains Mono（兩平台相同） |
| 2026-10-17 | M1 拆成 1A～1D 四段，細部計畫見 `plan/01-skeleton-chat.md`；只建立當段需要的專案 |
| 2026-10-17 | `ISecretStore`（DPAPI、Keychain）從 M2 提前到 M1；實作放 `Lumi.Platform`，不放 Infrastructure |
| 2026-10-17 | macOS bundle id 與 Keychain service 名稱定為 `io.github.neillog.lumi`，定案後不更改 |
| 2026-10-17 | 「Core 不引用 Avalonia」與「P/Invoke 只在 Platform」以架構測試保證 |
| 2026-10-18 | 字型改用靜態字重檔（Noto Sans TC、JetBrains Mono 各 400／500／700）：Avalonia 11.3 不支援可變字型的字重軸 |
| 2026-10-18 | Anthropic 改用官方 SDK（NuGet `Anthropic`），不自己寫 HTTP／SSE 解析；只在 `Lumi.Providers` 引用，走 `client.Beta.Messages`（server-side fallback 只在 beta） |
| 2026-10-18 | 開啟 server-side fallback（拒絕時自動換 `claude-opus-4-8` 回答），可在設定關閉 |
| 2026-10-18 | `ChatEvent` 先只定義 `TextDelta`／`ThinkingDelta`／`Usage`／`Completed`／`Failed` 五種，工具相關三種留到 M3 |
| 2026-10-18 | 對話歷史只能往後加，thinking 區塊（含 signature）要原樣送回；寫進 `agent.md` 第 7 節 |
| 2026-10-18 | 連線、模型、主題等設定存 `settings.json`（使用者資料夾），金鑰仍只在 `ISecretStore` |
| 2026-10-18 | OpenAI 相容 Provider 自己寫（`HttpClient` + `System.Net.ServerSentEvents`，只走 Chat Completions），不用官方 OpenAI SDK |
| 2026-10-18 | 模型切換膠囊＝修改工作台預設模型，不另外記每則對話的模型 |

## 待確認事項

- [ ] 公司 Whisper 服務的回應 JSON 格式
- [ ] Whisper 服務的內網憑證是否為自簽，以及處理方式
- [ ] M0 的 Windows 驗證結果（特別是點擊穿透）
- [ ] 小精靈模式的對話介面（氣泡、權限確認、語音）：草稿見 `docs/ui-spec.md` 第 8 節

---

## 細部計畫的格式

檔名：`plan/NN-名稱.md`，NN 對應里程碑編號（例如 `plan/01-skeleton.md`）。

```markdown
# NN 名稱
- 狀態：草稿 / 已確認 / 進行中 / 完成
- 對應里程碑：Mx

## 目標
## 範圍（做什麼、不做什麼）
## 設計（介面、資料結構、流程）
## 工作項目（可勾選）
## 驗收標準
## 實作紀錄（偏離原計畫的地方與原因）
```
