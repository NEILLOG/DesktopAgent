# 04 檔案與資料夾上下文

- 狀態：草稿，待確認「開工前要確認的事」後從 4A 開工
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

- [ ] **A1 Core**：`IFileScope`、`ScopeRoot`、`ScopeResolution`、`ContextItem`、`IConversationContextStore`、`ToolContext.Scope`（改既有建構處與測試）；沒有範圍時的空實作。
- [ ] **A2 路徑安全**：`FileScope` 實作（正規化、符號連結、大小寫、目錄邊界、機密名單）；**大量測試**：`..`、絕對路徑、前綴誤判、符號連結逃逸（資料夾內與單一檔案）、UNC／裝置名稱（Windows 限定的測試標記）、大小寫差異、尾端分隔字元、空字串與非法字元、機密檔名。
- [ ] **A3 `read_file`**：編碼偵測、二進位檔判斷、行號、`offset`／`limit`、截斷說明；測試。
- [ ] **A4 `list_dir`**：深度、項目上限、略過名單、排序（資料夾在前）；測試。
- [ ] **A5 `search`**：glob、正規表示式（`NonBacktracking`、逾時）、檔案與結果上限、略過二進位與大檔；測試（含惡意正規表示式）。
- [ ] **A6 存檔**：`conversation_context` 資料表、schema 升到 3（含升級與舊版拒絕的測試）、`SqliteConversationContextStore`、記憶體替身；測試。
- [ ] **A7 接進對話迴圈**：`ConversationService` 讀出對話的範圍、建立 `ToolContext.Scope`、範圍為空時不送檔案工具（讓工具標示 `RequiresScope`，由服務過濾）；假 Provider 腳本測試「讀範圍內檔案成功、讀範圍外被拒絕並由模型看到原因」。
- [ ] **A8 組裝與文件**：`AddLumiTools` 註冊三個工具、DI、`EnableDemoTools` 預設改關、架構測試、`agent.md` 第 8 節（範圍與機密名單）、本檔實作紀錄、主計畫狀態。

### 驗收標準

- [ ] `dotnet build` 零警告、`dotnet test` 全過。
- [ ] 路徑安全測試涵蓋上面列的所有逃逸手法；**任何一種都讀不到範圍外的檔案**。
- [ ] 模型（假 Provider）要求讀範圍外的檔案時，收到錯誤結果「不在可存取的範圍內」並能繼續對話；要求讀機密檔名時收到「被視為機密」。
- [ ] 沒有範圍時，請求不帶這三個工具的定義。
- [ ] 舊資料庫（schema 2）打開後自動升到 3，既有對話與訊息完好。

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

- [ ] **B1 內容區塊與存檔格式**：`ContextBlock`、`ImageBlock`、JSON 多型測試（舊資料仍可讀）。
- [ ] **B2 `IContextAttachmentBuilder`**：檔案（嵌入／太大只放範圍／找不到）、資料夾（檔案樹、上限、略過名單）、圖片（驗證、存檔）；總量上限；測試（含 Unicode 檔名、空檔案、二進位檔、超大檔、權限不足）。
- [ ] **B3 送出路徑**：`ConversationService` 嵌入與標記；失敗時不遺失；測試（沒有項目時行為不變、多個項目、嵌入後再送一次不會重複）。
- [ ] **B4 Anthropic**：`ContextBlock` 轉文字、`ImageBlock` 轉 image 區塊；fixture 測試。
- [ ] **B5 OpenAI 相容**：同上，圖片走 `image_url`；帶圖片時的 400 提示；fixture 測試。
- [ ] **B6 `ITokenEstimator`**：啟發式；測試（中英混合、emoji、空字串、大檔）。
- [ ] **B7 圖片清理**：刪除對話後的孤兒檔案；測試。
- [ ] **B8 手動驗收**（需要你的金鑰）：兩家各加入一個小檔案問內容、加入圖片問畫面、加入資料夾請模型找某個檔案並讀取。
- [ ] **B9 文件**：`agent.md`（快照語意、圖片存放）、本檔實作紀錄、主計畫狀態。

### 驗收標準

- [ ] 加入一個含上萬個檔案的資料夾：訊息裡只有不超過上限的檔案樹，token 預估不爆。
- [ ] 同一個項目不會被送兩次；移除 chip 之後工具立刻讀不到，歷史裡的快照還在。
- [ ] 兩家格式的圖片請求符合各家規格（fixture 驗證請求內容）。
- [ ] 讀不到的檔案、太大的檔案、不支援的圖片格式都有清楚的說明，而且不會讓送出失敗。

---

## 4C 輸入區 UI（大綱，4A、4B 完成後展開）

- **上下文列**（ui-spec 6.1）：標題「此對話的上下文」、chip（檔案、資料夾用琥珀淡底並顯示檔案數、圖片縮圖）、每個可移除、右側「≈ N tokens」；狀態：待送出／已送出／找不到／太大（由模型按需讀取）。
- **加入方式**：「加入檔案」「加入資料夾」（Avalonia `StorageProvider`，用 `IFilePickerService` 抽象讓 ViewModel 可測）、拖放、貼上截圖（剪貼簿圖片）。
- **使用者氣泡**：上方顯示送出當時帶的 chip（來自訊息的 `ContextBlock`／`ImageBlock`，重開程式後仍在）。
- **移除時的說明**：已送出的項目被移除，提示「模型之前看過的內容仍在這則對話裡」。
- **回應中**：加入／移除停用或維持可用？（待展開時決定；預設回應中不能改範圍，避免工具執行到一半範圍變了）。
- 測試：ViewModel（加入、移除、預估、重複加入、找不到）、Headless（chip 顯示、拖放事件）、手動驗收。

---

## 實作紀錄

（偏離原計畫的地方與原因，實作時填寫）
