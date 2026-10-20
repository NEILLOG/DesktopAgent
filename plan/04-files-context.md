# 04 檔案與資料夾上下文

- 狀態：Q1～Q4 照建議確認（2026-10-20）；**4A～4C 程式完成**，待手動驗收（B8／C5，需要你的金鑰）
- 對應里程碑：M4
- 前置：M3 程式完成（3A～3C）；M3 的手動驗收可以和 M4 並行，不擋開工

## 目標

使用者在輸入框加入的檔案與資料夾，在同一則對話中都能當作上下文：小檔案的內容直接帶給模型、圖片以圖片格式送出、資料夾只送檔案樹，內容由「範圍限定」的工具按需讀取。加入一個大型資料夾不會撐爆 token，工具也讀不到範圍外的任何檔案。

## M4 拆成三個部分

| 部分 | 名稱 | 產出 | 依賴 |
|---|---|---|---|
| **4A** | 範圍與檔案工具 | `ToolContext.Scope`、路徑安全（含符號連結、`..`）、`list_dir`／`read_file`／`search`、對話的上下文項目存檔（SQLite schema 3） | M3 |
| **4B** | 上下文送出與圖片 | 送出時把新加入的項目嵌進使用者訊息（檔案文字、檔案樹、圖片）、圖片存檔、兩家 Provider 的圖片格式、token 預估 | 4A |
| 4C | 輸入區 UI | 上下文列與 chip、加入檔案／資料夾（選擇器、拖放、貼上截圖）、預估 token 顯示、使用者氣泡上的 chip | 4A、4B |

4A 完成時，用假的 Provider 加上手動塞進去的範圍就能讓工具運作（有測試）；4B 讓資料真的送給模型；4C 讓使用者看得到、加得了。每段結束都要能編譯、測試全過。

主計畫 M4 項目的對應：

| 主計畫項目 | 部分 |
|---|---|
| `ToolContext.Scope` 與路徑跳脫防護（要有測試） | 4A |
| 資料夾：只送檔案樹，內容由範圍限定的 `list_dir`、`read_file`、`search` 讀取 | 4A（工具）、4B（檔案樹） |
| 單一檔案：文字直接放入；圖片用圖片格式送出 | 4B |
| 送出前預估上下文 token | 4B（估算）、4C（顯示） |
| 輸入框的「加入檔案」「加入資料夾」、chip、可移除 | 4C |

---

## 開工前要確認的事

以下是影響資料格式、行為語意或安全的決定。我已經各自給了建議；**沒有意見的話就照建議做**。

| # | 問題 | 建議 | 影響 |
|---|---|---|---|
| Q1 | 上下文什麼時候、怎麼送給模型？ui-spec 說「上下文屬於整則對話，不會因為送出訊息而消失」，但 agent.md 第 7 節要求歷史只能往後加（改動前面的訊息會讓快取失效，thinking 簽章也可能出問題）。 | **加入時拍快照、帶進下一則使用者訊息**：加入的項目在「下一次送出」時，把當下的內容（檔案文字、資料夾的檔案樹、圖片）嵌進那則使用者訊息並存檔，之後不再重複送。chip 一直顯示在輸入框上方、代表「目前在範圍內」。**移除 chip＝從範圍拿掉**（工具立刻讀不到），但已經送出的內容留在歷史裡，模型不會「忘記」；UI 在移除時說明這一點。再加入同一個檔案＝再拍一次新快照。檔案之後被改動，快照不更新，要新內容時模型用 `read_file`。（替代做法：每次請求都重組進 system prompt，永遠最新，但每次重送整份、快取全失效、檔案多時很貴。） | 4A、4B |
| Q2 | 圖片存哪裡？ | **存成檔案**（使用者資料夾下 `attachments/`，檔名是內容雜湊），訊息裡的 `ImageBlock` 只存檔名、媒體類型與原始名稱，送出時才讀檔轉 base64。不把 base64 放進 `content_json`：歷史重讀時整則對話的內容都會載入，幾張截圖就會拖慢每次載入。對話被刪除時，沒有任何訊息引用的圖片檔一併清掉。 | 4B |
| Q3 | 範圍內的機密檔案要不要擋？（`.env`、私鑰、`secrets.json` 之類；模型一旦讀了就會送到雲端） | **預設擋**：`list_dir`／`read_file`／`search` 對「明顯是機密」的檔名（`.env*`、`*.pem`、`*.key`、`*.pfx`、`id_rsa*`、`*.p12`、`secrets.json`、`.git/` 內部、`.ssh/`）不回傳內容，告訴模型「這個檔案被視為機密，不會讀取」。**使用者自己加入的單一檔案不擋**（那是明確的選擇）。名單寫在一處、有測試，之後可以改成設定。 | 4A |
| Q4 | 示範用的 `save_note`／`delete_note` 怎麼處理？（M3 說 M4 有真正的檔案工具後移除；但 M4 只有唯讀工具，移除後就沒有東西可以驗收 `Write`／`Destructive` 的確認流程） | **保留，但 `EnableDemoTools` 預設改成關**（要驗收時在設定檔打開）。真正的寫入類檔案工具是 M10（WPF 工具移植），到時候再移除示範工具。M4 不做寫入類工具：寫入的範圍與確認要一起設計，不在這個里程碑。 | 4A |

其他我直接定的數字（寫在這裡方便你有意見時指出來，都集中在一個選項類別，之後可調）：

- **內嵌檔案的上限**：單一文字檔 ≤ 100 KB（約 3 萬 token）才嵌進訊息；更大的檔案**只放進範圍**（chip 標示「太大，由模型按需讀取」），模型用 `read_file` 分段讀。整則訊息嵌入的檔案文字合計 ≤ 200 KB，超過的後面幾個同樣只放進範圍。
- **圖片**：PNG／JPEG／GIF／WebP，≤ 5 MB（Anthropic 單張上限），其他格式或更大的拒絕並說明。沒有「模型是否支援圖片」的旗標：照送，伺服器回 400 且請求帶了圖片時，提示「這個模型可能不支援圖片」（和 M3 的工具提示同樣做法）。
- **資料夾檔案樹**：最多 300 個項目、深度 4 層，預設略過 `.git`、`node_modules`、`bin`、`obj`、`.DS_Store`、`__pycache__` 等（固定名單，不解析 `.gitignore`），超過時寫「… 還有 N 個項目」。
- **`read_file`**：一次最多回傳 `MaxResultChars`（2 萬字元，M3 已有），可用 `offset`／`limit`（行）分段；回傳附行號與「還有 N 行」。UTF-8（含 BOM）與 UTF-16 偵測，其他編碼或二進位檔回錯誤。
- **`search`**：在範圍內搜尋檔名（glob）或內容（正規表示式，用 `NonBacktracking` 避免災難性回溯）；最多掃 5,000 個檔案、單檔 ≤ 1 MB、回 100 筆符合。
- **token 預估**：啟發式（ASCII 約 4 字元 1 token、非 ASCII 約 1 字 1 token、圖片固定估 1,500），顯示為「≈ N tokens」，只是送出前的參考；實際用量仍以供應商回傳的 `usage` 為準（agent.md 第 7 節）。

---

## 設計總覽（跨三個部分共用）

### 範圍（Core 介面，`Lumi.Tools` 實作）

```csharp
// 對話目前的範圍：使用者加入的檔案與資料夾。工具只能經由它存取檔案系統。
public interface IFileScope
{
    IReadOnlyList<ScopeRoot> Roots { get; }               // 檔案或資料夾，已是絕對路徑
    // 把工具收到的路徑（相對於某個根、或絕對路徑）解析成範圍內的真實路徑；範圍外、符號連結跳出去、
    // 機密檔案一律回失敗（附給模型看的原因），不丟例外。
    ScopeResolution Resolve(string path);
}

public sealed record ToolContext(string ConversationId, ChatMode Mode, TimeProvider Time, IFileScope Scope);
```

- 路徑一律先 `Path.GetFullPath` 正規化，再解析符號連結（`ResolveLinkTarget`／逐段檢查），最後確認「真實路徑」在某個根之下；比較用平台對應的大小寫規則（Windows、macOS 預設不分大小寫），而且以目錄分隔字元結尾比對，避免 `/a/b` 誤判 `/a/bc` 在範圍內。
- 拒絕：`..` 跳出、絕對路徑在範圍外、UNC、裝置名稱（Windows `CON`、`NUL`…）、符號連結指向範圍外、機密檔名（Q3）。
- 檔案類根（使用者只加了單一檔案）：範圍只有那一個檔案，不含同資料夾的其他檔案。

### 對話的上下文項目（存檔）

```csharp
public enum ContextKind { File, Folder, Image }
public sealed record ContextItem(string Id, ContextKind Kind, string Path, string DisplayName, DateTimeOffset AddedAt, string? EmbeddedInMessageId);
```

- `IConversationContextStore`：`GetAsync(conversationId)`、`AddAsync`、`RemoveAsync`、`MarkEmbeddedAsync(itemIds, messageId)`。SQLite 新資料表 `conversation_context`（`ON DELETE CASCADE`）；**schema 升到 3**（新資料表加上新的內容區塊型別，舊版程式要明確拒絕開啟）。
- `EmbeddedInMessageId` 為 null＝還沒送出（下一則訊息要帶）。移除項目＝刪掉那一列。
- 路徑存絕對路徑；檔案之後被搬走或刪除時，chip 顯示「找不到」，範圍內的工具呼叫回錯誤（歷史裡的快照不受影響）。

### 訊息裡的內容區塊（4B）

```csharp
// 檔案或資料夾的快照：UI 在使用者氣泡上方畫成 chip；送給模型時轉成文字
public sealed record ContextBlock(ContextKind Kind, string DisplayName, string Path, string? Content, bool Embedded, int? ItemCount) : ContentBlock;
public sealed record ImageBlock(string FileName, string MediaType, string DisplayName) : ContentBlock;
```

- 使用者訊息的內容順序：`ContextBlock`／`ImageBlock`…，最後是使用者打的 `TextBlock`。
- 轉成文字的格式（兩家共用）：`<file path="…">\n行號 內容…\n</file>`、`<folder path="…">\n檔案樹…\n</folder>`；沒嵌入內容的檔案寫 `<file path="…" note="太大，請用 read_file 讀取" />`。
- 圖片：Anthropic 的 `image` 區塊（base64 來源）、OpenAI 相容的 `image_url`（data URI）；別家不認得的區塊照 `ProviderBlock` 規則略過。

### 工具（`Lumi.Tools`，都是 `ReadOnly`、兩種模式都可用）

| 工具 | 說明 |
|---|---|
| `list_dir` | 列出範圍內某個資料夾的項目（名稱、種類、大小）；參數 `path`、`depth`（預設 1、最多 3）；項目超過上限時截斷並說明 |
| `read_file` | 讀範圍內的文字檔；`path`、`offset`、`limit`（行）；輸出附行號 |
| `search` | 在範圍內搜尋：`pattern` 比對內容（正規表示式），可選 `glob` 限制檔名；回傳「檔案:行號: 內容」 |

- 工具的 `Preview().Target` 顯示路徑（卡片標題「讀取檔案 · plan/00-overview.md · 182 行」）；`Summary` 由工具提供。
- 沒有任何範圍（使用者沒加入資料夾或檔案）時，工具回錯誤「目前沒有加入任何檔案或資料夾」，而且**不送出這三個工具的定義**（`ConversationService` 依範圍是否為空過濾，省 token 也避免模型亂試）。

---

## 4A 範圍與檔案工具

### 範圍

**做**：`IFileScope`／`ToolContext.Scope`、`ContextItem` 與 `IConversationContextStore`（Core 介面＋SQLite 實作＋記憶體測試替身）、路徑安全、機密檔名名單、三個檔案工具、`ConversationService` 依對話的範圍建立 `ToolContext.Scope`（沒有範圍就不送檔案工具）、`EnableDemoTools` 預設改關、SQLite migration 3、架構測試（檔案系統存取只在 `Lumi.Tools` 的檔案工具與範圍實作裡）。

**不做**：把內容送給模型（4B）、任何畫面（4C）、寫入類檔案工具（M10）。

### 工作項目

- [x] **A1 Core**：`IFileScope`、`ScopeRoot`、`ScopeResolution`、`ContextItem`、`IConversationContextStore`、`ToolContext.Scope`（改既有建構處與測試）；沒有範圍時的空實作。
- [x] **A2 路徑安全**：`FileScope` 實作（正規化、符號連結、大小寫、目錄邊界、機密名單）；**大量測試**：`..`、絕對路徑、前綴誤判、符號連結逃逸（資料夾內與單一檔案）、UNC／裝置名稱（Windows 限定的測試標記）、大小寫差異、尾端分隔字元、空字串與非法字元、機密檔名。
- [x] **A3 `read_file`**：編碼偵測、二進位檔判斷、行號、`offset`／`limit`、截斷說明；測試。
- [x] **A4 `list_dir`**：深度、項目上限、略過名單、排序（資料夾在前）；測試。
- [x] **A5 `search`**：glob、正規表示式（`NonBacktracking`、逾時）、檔案與結果上限、略過二進位與大檔；測試（含惡意正規表示式）。
- [x] **A6 存檔**：`conversation_context` 資料表、schema 升到 3（含升級與舊版拒絕的測試）、`SqliteConversationContextStore`、記憶體替身；測試。
- [x] **A7 接進對話迴圈**：`ConversationService` 讀出對話的範圍、建立 `ToolContext.Scope`、範圍為空時不送檔案工具（讓工具標示 `RequiresScope`，由服務過濾）；假 Provider 腳本測試「讀範圍內檔案成功、讀範圍外被拒絕並由模型看到原因」。
- [x] **A8 組裝與文件**：`AddLumiTools` 註冊三個工具、DI、`EnableDemoTools` 預設改關、架構測試、`agent.md` 第 8 節（範圍與機密名單）、本檔實作紀錄、主計畫狀態。

### 驗收標準

- [x] `dotnet build` 零警告、`dotnet test` 全過。
- [x] 路徑安全測試涵蓋上面列的所有逃逸手法；**任何一種都讀不到範圍外的檔案**。
- [x] 模型（假 Provider）要求讀範圍外的檔案時，收到錯誤結果「不在可存取的範圍內」並能繼續對話；要求讀機密檔名時收到「被視為機密」。
- [x] 沒有範圍時，請求不帶這三個工具的定義。
- [x] 舊資料庫（schema 2）打開後自動升到 3，既有對話與訊息完好。

---

## 4B 上下文送出與圖片

### 範圍

**做**：送出時把「還沒送出」的上下文項目嵌進使用者訊息（檔案文字、檔案樹、圖片）並標記已送出；`ContextBlock`／`ImageBlock`；圖片存檔（`attachments/`）與清理；兩家 Provider 的轉換與 fixture 測試；token 預估；上限與錯誤（太大、不是文字、圖片格式不支援）。

**不做**：輸入區的畫面（4C）。

### 設計要點

- `ConversationService.SendAsync(conversationId, text, …)` 讀出該對話 `EmbeddedInMessageId == null` 的項目，交給 `IContextAttachmentBuilder`（Core 介面、`Lumi.Tools` 實作，因為要讀檔案）產生區塊，和使用者文字一起存成**同一則**使用者訊息，再標記項目已送出；兩步之間失敗時，項目維持「未送出」，下次送出會再帶（不會重複：訊息沒存成功就不存在）。
- 建構區塊時讀不到的項目（檔案被刪了）：不中斷送出，產生一個 `ContextBlock`（`Embedded = false`、說明找不到）並照常標記，使用者與模型都看得到。
- 圖片：讀進記憶體驗證格式與大小、存進 `attachments/<sha256>.<ext>`（已存在就略過）；`ImageBlock` 只存檔名與媒體類型。清理：刪除對話時，沒有任何訊息引用的檔案才刪（以資料庫查詢判斷）。
- Provider 轉換時才讀檔轉 base64（Provider 透過注入的 `IAttachmentStore` 讀；Providers 專案不碰檔案路徑規則）。
- 預估 token：`ITokenEstimator`（Core 介面、啟發式實作）；輸入是「歷史＋待送出的項目」，由 4C 顯示。

### 工作項目

- [x] **B1 內容區塊與存檔格式**：`ContextBlock`、`ImageBlock`、JSON 多型測試（舊資料仍可讀）。
- [x] **B2 `IContextAttachmentBuilder`**：檔案（嵌入／太大只放範圍／找不到）、資料夾（檔案樹、上限、略過名單）、圖片（驗證、存檔）；總量上限；測試（含 Unicode 檔名、空檔案、二進位檔、超大檔、權限不足）。
- [x] **B3 送出路徑**：`ConversationService` 嵌入與標記；失敗時不遺失；測試（沒有項目時行為不變、多個項目、嵌入後再送一次不會重複）。
- [x] **B4 Anthropic**：`ContextBlock` 轉文字、`ImageBlock` 轉 image 區塊；fixture 測試。
- [x] **B5 OpenAI 相容**：同上，圖片走 `image_url`；帶圖片時的 400 提示；fixture 測試。
- [x] **B6 `ITokenEstimator`**：啟發式；測試（中英混合、emoji、空字串、大檔）。
- [x] **B7 圖片清理**：刪除對話後的孤兒檔案；測試。
- [ ] **B8 手動驗收**（需要你的金鑰）：兩家各加入一個小檔案問內容、加入圖片問畫面、加入資料夾請模型找某個檔案並讀取。
- [x] **B9 文件**：`agent.md`（快照語意、圖片存放）、本檔實作紀錄、主計畫狀態。

### 驗收標準

- [x] 加入一個含上萬個檔案的資料夾：訊息裡只有不超過上限的檔案樹，token 預估不爆。
- [x] 同一個項目不會被送兩次；移除 chip 之後工具立刻讀不到，歷史裡的快照還在。
- [x] 兩家格式的圖片請求符合各家規格（fixture 驗證請求內容）。
- [x] 讀不到的檔案、太大的檔案、不支援的圖片格式都有清楚的說明，而且不會讓送出失敗。

---

## 4C 輸入區 UI

### 範圍

**做**：上下文列（chip、預估 token）、加入檔案／資料夾（選擇器、拖放、貼上截圖）、移除、使用者氣泡上方的 chip、回應中的限制。

**不做**：小精靈的上下文操作（M7）、對 chip 內容的預覽視窗、設定上下文大小上限的畫面（上限在 `FileToolOptions`）。

### 設計

- **`ContextBarViewModel`**（注入 `WorkbenchViewModel`）：管理目前對話的項目清單（chip）、加入、移除、預估 token；對話切換、新對話、送出完成後重新載入。加入與移除在**回應中停用**（範圍不能在工具執行到一半時改變）。
- **chip 的狀態**：待送出（下一則訊息會帶上）、已帶入（已經送出過，仍在範圍內）、有問題（找不到、不是文字檔、太大由模型按需讀取、不支援的圖片格式、圖片太大）。狀態由 `IContextInspector`（Core 介面、`Lumi.Tools` 實作）檢查檔案系統得到，**畫面不碰檔案系統**。
- **加入時判斷種類**：資料夾 → 資料夾；檔案開頭是 PNG／JPEG／GIF／WebP → 圖片；其他 → 檔案（二進位檔也能加，但 chip 會標「不是文字檔」，不會放進內容）。選擇器、拖放、貼上都走同一條路（`AddPathsAsync`），重複加入同一項不會多一個 chip（存檔層已處理）。
- **貼上截圖**：剪貼簿的圖片存進附件資料夾（`IAttachmentStore.SaveAsync`，新增 `GetPath` 取得完整路徑），再當成圖片項目加入，名稱「貼上的截圖 14:32」。
- **預估 token**：對話歷史（`BuildRequestMessages` 之後）＋待送出項目（`IContextAttachmentBuilder.EstimateTokensAsync`）＋輸入框草稿，顯示成「≈ 18.2K tokens」；超過模型的上下文上限時改警告色。只是參考，說明寫在提示文字（實際用量看狀態列的供應商回傳值）。
- **移除已帶入的項目**：列上顯示一次性的說明「已從範圍移除；模型之前看過的內容仍在這則對話裡」。
- **使用者氣泡**：上方顯示送出當時帶的 chip——送出的瞬間用待送出的 chip、重讀歷史時由 `ContextBlock`／`ImageBlock` 重建（所以重開程式後還在）。
- **畫面**：上下文列只在有項目時出現（輸入框上方）；輸入框左下「加入檔案」「加入資料夾」；整個輸入區接受拖放（拖進來時外框變色）；圖片 chip 顯示縮圖（在畫面層解碼，失敗就只顯示名稱）；資料夾 chip 琥珀淡底並顯示項目數。只用 token 上色。

### 工作項目

- [x] **C1 Core／Tools**：`IContextInspector`（`ClassifyAsync`、`InspectAsync`）與實作、`IAttachmentStore.GetPath`、資料夾項目計數；測試。
- [x] **C2 `ContextBarViewModel`**：chip、加入／移除、預估、狀態、回應中停用、一次性說明；ViewModel 測試（用假的選擇器、剪貼簿、檢查器）。
- [x] **C3 接進工作台**：`WorkbenchViewModel` 與對話切換／送出的整合、使用者氣泡的 chip（送出時與重讀時）、DI；測試。
- [x] **C4 畫面**：上下文列、chip 樣式與縮圖、輸入框的按鈕、拖放、貼上；`IFilePickerService`、`IClipboardImageService` 的實作；Headless 測試（chip 顯示、按鈕、拖放事件）。
- [ ] **C5 手動驗收**（需要你的金鑰）：這同時是 4B 的 B8——兩家各加入小檔案、圖片、資料夾，問內容、請模型找檔案並讀取、移除後再問。
- [x] **C6 文件**：`docs/ui-spec.md`（與實作有出入處）、本檔實作紀錄、主計畫（M4 狀態）、README。

### 驗收標準

- [x] `dotnet build` 零警告、`dotnet test` 全過。
- [x] 加入檔案、資料夾、圖片（選擇器、拖放、貼上）都能成為 chip；重複加入不會多一個；移除後範圍立刻縮小。
- [x] 回應中不能加入或移除；回應結束後恢復。
- [x] 送出後 chip 變成「已帶入」，使用者氣泡上方有 chip；重開程式後氣泡上的 chip 仍在。
- [x] 預估 token 隨項目與歷史變動，超過上限時有警告。
- [x] 找不到、不是文字檔、太大、不支援的圖片格式都在 chip 上看得到原因，而且不會讓加入或送出失敗。

---

## 實作紀錄

### 4A（2026-10-20）

- **Q1～Q4 照建議採用**（快照語意、圖片存檔案、預設擋機密檔案、示範筆記工具保留但預設關）。
- **新增 `Lumi.Core.Context`**（`ContextItem`、`ContextKind`、`IConversationContextStore`、空實作）與 `Lumi.Core.Tools.FileScope.cs`（`IFileScope`、`ScopeRoot`、`ScopeResolution`、`IFileScopeFactory`、空範圍）。`ToolContext` 多了必填的 `Scope`；`ToolDescriptor` 多了選填的 `RequiresScope`。
- **`IFileScope` 多了 `ToDisplayPath`**（計畫沒寫）：工具回傳給模型與畫面的路徑要相對於範圍的根，有多個根時以名稱開頭，所以範圍自己負責換算。
- **相對路徑的規則**（計畫只寫了「相對於某個根」，這裡定下來）：只有一個根時相對於它；有多個根時要以根的名稱開頭（重複的名稱加「 (2)」）；單一檔案的範圍只接受那個檔名。`list_dir` 在多個根又沒給路徑時列出各根的名稱。
- **符號連結**：逐段展開（目標路徑本身經過符號連結也會再展開，上限 40 層，防止迴圈）；使用者加入的資料夾本身是符號連結（macOS 的 `/var`、`/tmp`）也正常——範圍用的是它指向的真實位置，測試裡的暫存資料夾剛好就是這種情況。壞掉的連結不丟例外。`list_dir` 與 `search` 遇到指向範圍外的連結只標示、不跟隨；範圍內的連結迴圈用「已走過的真實路徑」避開。
- **平台規則可以在任何平台測試**：`FileScopeRules(CaseInsensitive, WindowsPathRules, NormalizeUnicode)` 預設依作業系統，測試可以指定另一個平台的規則；`CheckWindowsRules` 是純字串判斷。macOS 的 NFC／NFD 檔名差異也在比對時正規化。
- **機密名單**（`ScopeRules`）：`.env`（`.env.example`／`.sample`／`.template`／`.dist` 不算）、`.env.*`、`*.pem`、`*.key`、`*.pfx`、`*.p12`、`id_rsa*`／`id_ed25519*`／`id_ecdsa*`／`id_dsa*`、`secrets.json`、`.git/` 與 `.ssh/` 內部。只檢查「根底下」的部分，所以使用者加入的資料夾本身叫什麼都可以。`list_dir` 會列出機密檔案的名稱並標「機密，不會讀取」；`search` 略過並在結尾報告略過了幾個。
- **`read_file`**：嚴格 UTF-8（不合法就回錯誤，不默默換成亂碼）、UTF-8／UTF-16 BOM 自動處理、前 8 KB 有 NUL 就當二進位檔；以字元預算截斷並告訴模型用哪個 `offset` 繼續；總行數會一路數到檔尾（檔案上限 10 MB，所以便宜）；用 `FileShare.ReadWrite | Delete` 開檔，別的程式正在寫入的檔案（log）也讀得到。
- **`search`**：正規表示式一律用 `NonBacktracking`（時間與輸入長度成正比，有測試用 `(a+)+$` 對 5,000 個 `a` 驗證），不支援反向參照與 lookaround 時回清楚的錯誤；glob 沒有斜線時只比對檔名；結果與掃描檔案數都有上限，結尾說明被截斷或略過了什麼。
- **`EnableDemoTools` 預設改成 false**（`ToolsOptions` 與設定檔皆是）；`ToolsOptions.Files` 是 `FileToolOptions`（所有上限集中一處）。
- **SQLite**：migration 3 建立 `conversation_context`（`UNIQUE(conversation_id, kind, path)`、`ON DELETE CASCADE`）；`SqliteConversationStore` 同時實作 `IConversationContextStore`（共用連線字串與初始化）。`AddAsync` 對已存在的項目：已送出的重設成待送出（再拍一次快照）、待送出的不動。
- **`ConversationService`** 每一輪迴圈重新讀取對話的上下文項目、建立範圍、依 `RequiresScope` 過濾工具（工具執行中使用者加入資料夾，下一輪就會帶上）。完整建構式多兩個選填參數（`IConversationContextStore`、`IFileScopeFactory`），既有呼叫不用改；Core 的 `TryAdd` 預設是空實作，Infrastructure 與 Tools 註冊真正的實作。
- **架構測試**：碰檔案系統的工具（示範筆記工具除外）原始碼必須出現 `context.Scope`；有對照組確認偵測方式找得到。
- **抓到一個不穩的測試**：`SettingsHost.WaitUntil` 輪詢條件時，回應事件在背景執行緒修改畫面清單（正式程式有 UI 同步內容，測試沒有）會偶爾丟「Collection was modified」。多一個 `await` 改變了時序才浮現（3C 的工具卡片測試連續失敗）；輪詢改成遇到這個例外就當作「還沒好」。
- **測試數**：Core 121、Tools 270（新增範圍 79 個、檔案工具 91 個）、Infrastructure 54、Architecture 47、Providers 103、App 257、Platform 51（13 略過）。

### 4B（2026-10-20）

- **`ContextBlock` 多了 `Note` 欄位**（計畫的簽名沒有）：放不進內容時說明原因，同一個型別也用來表示「圖片讀不到／格式不支援」。順序是 `ContextBlock(Kind, DisplayName, Path, Content, Embedded, Note, ItemCount)`。
- **轉成文字的格式集中在 `ContextBlockFormatter`（Core）**：檔案是 `<file path="絕對路徑">` ＋ 行號（和 `read_file` 同樣的 `PadLeft(6)` + Tab）；資料夾是 `<folder path=… entries=…>` ＋ 檔案樹；沒放內容的是自我結尾的標籤加 `note`。屬性值會跳脫 `"`、`<`、`&`，路徑或說明裡的字元跳不出標籤。用絕對路徑是因為不管有幾個範圍，絕對路徑一定能被 `read_file` 解析。
- **`IContextAttachmentBuilder` 的介面多了 `EstimateTokensAsync`**：做法是「不存圖片的試跑」，再交給 `ITokenEstimator`；4C 要顯示預估時直接用。
- **送出路徑**（`ConversationService.SendAsync`）：先讀待送出的項目、建構區塊、存使用者訊息，**訊息存好之後**才標記項目已送出（標記用 `CancellationToken.None`）。建構器丟例外時什麼都不存、項目維持待送出；訊息存好但標記失敗時（極少見）下次會重複帶一次，這個取捨比「標記了但訊息沒存」安全。`RetryAsync`／`RegenerateAsync` 不嵌入任何東西。
- **`FolderTree`**：不跟隨符號連結（只標示「（符號連結）」，因為目標可能在範圍外）、機密資料夾不展開、機密檔案標示；放滿之後只數剩下幾個（上限 5 萬，超過寫「超過 50,000 個」），不會為了數數走遍整個大型資料夾。
- **圖片格式**：`ImageFormats.Detect` 依魔術數字判斷（PNG、JPEG、GIF87a／89a、WebP），副檔名騙不了；BMP、SVG 等回「不支援的圖片格式」。
- **附件存放**（`FileAttachmentStore`）：檔名是 SHA-256 加副檔名（相同內容一份）；先寫暫存檔再改名；**讀取時檔名必須完全符合格式**（64 位小寫十六進位＋已知副檔名），資料庫被改成 `../lumi.db` 也讀不到別的檔案（有測試）；清理只動自己建立的檔案（符合格式的附件與殘留的 `.tmp-*`），資料夾裡別的東西不碰。
- **孤兒清理**：`AttachmentCleaner`（Core）＋`AttachmentCleanupService`（App，啟動時背景跑一次）。只在啟動時做，因為「圖片先存、訊息後存」的瞬間檔案看起來像孤兒。引用從資料庫掃描含圖片區塊的訊息得到（`SqliteConversationStore` 同時實作 `IAttachmentReferences`）。目前還沒有「刪除對話」的功能（M9），到時候同樣的清理在刪除後呼叫。
- **Provider**：Anthropic 的圖片是 `image` 區塊（base64 來源，媒體類型對應 SDK 的 `MediaType` 列舉；注意成員名稱是 `ImageWebP`）；OpenAI 相容的使用者訊息沒有圖片時維持字串內容，有圖片才改成 `[text, image_url…]` 陣列（data URI）。兩家在圖片檔讀不到時都改成「[圖片「名稱」已經遺失，沒有送出]」。OpenAI 相容的使用者訊息多個文字區塊現在以空行隔開（原本直接相連）。400 的提示：請求帶了工具、圖片或兩者時分別說明「不支援工具」「不支援圖片」「不支援工具或圖片」。
- **整合測試**（`ContextEndToEndTests`）：真的 SQLite、附件資料夾、檔案範圍、快照建構器與檔案工具，只有供應商是腳本：一個 1 萬個檔案的資料夾只產生不超過上限的檔案樹；資料夾＋檔案＋圖片送出後快照存檔、第二次不重複、重新加入拿到新快照而歷史不變；工具在範圍內讀得到、範圍外被拒絕；移除資料夾後下一輪不送檔案工具但歷史快照還在；檔案送出前被刪掉不會讓送出失敗。
- **測試數**：Core 167、Tools 297、Infrastructure 83、Providers 120、App 263、Architecture 47、Platform 51（13 略過）。
- **還沒做**：B8 手動驗收（需要你的金鑰）；4C 的畫面。

### 4C（2026-10-21）

- **新增 `IContextInspector`（Core 介面、`Lumi.Tools` 實作）**：畫面要知道「這個路徑是檔案、資料夾還是圖片」以及 chip 該標什麼問題，而 ViewModel 不能碰檔案系統。它和快照建構器用同一組上限（測試確認：chip 標「太大」的檔案，建構器就真的不嵌入），所以 chip 上看到的就是送出時會發生的事。資料夾項目數數到 1 萬就停（顯示「10,000+ 個項目」）。
- **`IAttachmentStore.GetPath`**（預設實作回 null）：貼上的截圖要當成圖片項目加入，項目需要一個路徑；存進附件資料夾後用它取得完整路徑，之後走和一般圖片完全相同的流程（快照建構器讀檔、相同內容雜湊相同所以不會存兩份）。
- **`ContextBarViewModel`**（注入 `WorkbenchViewModel`，選填參數，預設是什麼都不做的實例，所以既有的測試建構方式不用改）：
  - 選擇器、拖放、貼上都走 `AddPathsAsync`；找不到的路徑列在 `Error`，其餘照常加入；重複加入不會多一個 chip（存檔層處理）。
  - 回應中 `IsLocked`：命令停用，`AddPathsAsync` 與 `PasteImageAsync` 也直接不做事（拖放不會繞過鎖）。
  - 重讀時用版本號丟掉過時的結果（切換對話途中舊的載入才完成）。
  - 預估 token = 歷史（`BuildRequestMessages` 之後）＋待送出項目（`EstimateTokensAsync`）＋草稿；草稿變動只重算草稿那一項（不重讀檔案）。已帶入的項目不再算待送出（它們已經在歷史裡）。
- **使用者氣泡的 chip**：送出的瞬間用待送出的 chip 先顯示；重讀歷史時由 `ContextBlock`／`ImageBlock` 重建（資料夾的項目數、沒放入內容的說明都在區塊裡）。氣泡上的 chip 沒有縮圖（區塊只記附件檔名，要縮圖得讓 ViewModel 認識附件路徑，不值得）。
- **貼上**：Ctrl／Cmd+V 在輸入框的 tunnel 階段攔截：剪貼簿有圖片就加進上下文，沒有就呼叫 `TextBox.Paste()` 照常貼文字。剪貼簿給的是解碼後的點陣圖，統一存成 PNG。
- **拖放**：整個輸入區接受；用 Avalonia 11.3 的新 API（`DragEventArgs.DataTransfer`，舊的 `Data` 已標為過時）；拖進來時外框變成琥珀色，回應中不接受。
- **縮圖**：`PathToBitmapConverter` 只解碼成 40 px 寬；讀不到或解碼失敗回 null，chip 就不顯示圖（綁定看 `Source` 是否為 null，不留空白）。
- **畫面確認**：Mac 上淺色與深色都看過（用暫時的環境變數塞示範資料，已移除）：資料夾 chip 琥珀淡底、一般檔案 chip、有問題的 chip 警告色、「已帶入」、移除說明、預估 token 與警告符號、使用者氣泡上方的 chip。**拖放與貼上截圖沒有在真實視窗實際操作過**（沒有輔助使用權限無法點擊或拖曳）：ViewModel 與渲染結構有測試，但 Avalonia 的拖放事件與剪貼簿圖片在 macOS／Windows 上的實際行為，要靠你的手動驗收（C5）。
- **測試數**：Core 167、Tools 323、Infrastructure 84、Providers 120、App 316（新增上下文列 33、工作台整合 11、Headless 9）、Architecture 47、Platform 51（13 略過）。
- **還沒做**：C5／B8 手動驗收；Windows 全部。
