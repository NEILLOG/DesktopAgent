# 12 UI 重新設計（設計系統、五套主題、對話視窗 RWD、設定視窗新版面）
- 狀態：進行中（UA 開工）
- 對應里程碑：M12（排在 M9 之前）

## 依據（優先順序由高到低）

1. **`plan/ui-guide/ui-guide.html`（Lumi Design System）**：統一的設計原則。之後所有 UI 設計都以它為準；和下面兩份衝突時，以它為準。
2. `plan/ui-guide/RWD原則.md`：對話視窗的斷點與各區塊行為。
3. `plan/ui-guide/對話視窗範例.png`、`設定視窗範例.png`、`設計原則.png`：版面示意。圖上的橘紅色就是 terracotta 主題。

ui-guide.html 的三項核心原則：一般使用者優先，技術資訊需要時才出現；BYOK 一定要清楚顯示供應商、模型與費用歸屬；溫和但不幼稚，專業但不做成管理控制台。

## 目標

1. 建立設計系統：五套可切換主題（預設 sage），每套都有淺色和深色；字級、間距、圓角 token；元件樣式（輸入框、按鈕、卡片、Menu）。
2. 工作台改成設計稿的版面：自訂標題列、內容工具列、歡迎區、訊息（頭像＋時間）、兩層輸入框。
3. 工作台支援 Wide／Compact／Narrow 三段 RWD 與高度規則，從 1200 縮到 500 都不破版。
4. 設定視窗照設計系統重做（Menu、頁首、清單＋詳細、按鈕層級）。

## 範圍

**做**
- 主題系統：五套色票 × 淺／深、固定語意色、切換主題、設定存主題 ID。
- 設計 token：字級、間距、圓角；TextBox、Button、卡片、Menu 的共用樣式。
- 圖示資源 `Themes/Icons.axaml`，取代所有 Unicode 箭頭與符號。
- 斷點機制 `WindowSizeClass`（寬度三段＋高度兩段），用根節點 Classes 切樣式。
- 工作台拆成 `CustomTitleBar`、`ChatToolbar`、`ConversationView`、`ChatComposer` 四個 UserControl。
- 底部狀態列移除，內容移到模型選單頂部與模型膠囊的 Tooltip；「設定」移到「…」選單。
- BYOK 顯示：模型膠囊顯示確切的 model ID 與供應商，並加上費用說明。
- 設定視窗新版面；「回到對話」。
- 其餘 View（工具卡片、權限確認、ModelPicker、ConfirmDialog、小精靈氣泡）換 token 與圖示，不改結構。
- 沒有後端的功能先放**停用佔位**，旁邊用可見的說明寫出原因：👍👎、麥克風、設定 Menu 的「設定首頁」「資料與隱私」。

**不做**
- 設定的兩層「連線／模型設定」結構與分類內容（決定保留）。資料模型**只加 `ThemeId`**。
- 光球顏色（固定色，不隨主題變，`OrbControl.cs` 不動）。
- 小精靈視窗的版面與互動（RWD 指南明確排除；氣泡只換 token）。
- 👍👎、麥克風、設定首頁、資料與隱私的實際功能。
- POC（`poc/`）不動。

## 設計

### 已確認的決策（2026-10-22）

| 項目 | 決定 |
|---|---|
| 設計原則 | `ui-guide.html` 是統一原則，優先於 PNG 設計稿與 RWD原則.md |
| 主題 | 五套：`sage`（預設）、`teal`、`mist-blue`、`indigo`、`terracotta`。只換 Accent／Hover／Soft／Text／Muted／Background／Border；設定存主題 ID，不存個別顏色。撤銷先前「改用橘紅品牌色」的決定 |
| 深色 | 保留。五套各配一組深色（共 10 組）；明暗仍可選跟隨系統／淺色／深色 |
| 光球 | 維持自己的固定色，不隨主題變（撤銷先前「光球跟著改色」的決定） |
| 文字對比 | 維持 ≥ 4.5:1。Accent、背景、邊框、Soft 照原則色碼；只有文字用色在同色相下調深到合格 |
| 設定架構 | 保留兩層與現有分類，名稱改成白話（見「設定視窗」） |
| 無後端元素 | 停用佔位，旁邊加可見的 Caption 說明「即將推出」 |
| 底部狀態列 | 移除；token、回合、上下文用量移到模型選單頂部與膠囊 Tooltip |
| 設定按鈕 | 移到工具列「…」選單（設定、管理連線與模型…） |
| 設定視窗 | 維持獨立視窗；「回到對話」＝啟用工作台並關閉設定視窗 |
| 響應式機制 | 只用根節點 Classes（`wide`／`compact`／`narrow`、`short`／`veryshort`）。不用 Container Query，避免兩套機制並存 |
| 設定檔版本 | `JsonModelCatalog.CurrentVersion` 維持 1；`ThemeId` 為選填欄位，缺欄位或未知 ID 一律讀成 `sage`，不升版本、不寫遷移 |
| 設定頁主要按鈕 | 「AI 連線」頁頂部主要動作隨分段切換：連線頁是「＋新增連線」，模型設定頁是「＋新增模型設定」 |

### 主題系統

**色票**（淺色照 ui-guide.html；深色在 UA 推導後截圖確認）

| 主題 | Accent | Hover | Soft | Text | Muted | Background | Border |
|---|---|---|---|---|---|---|---|
| sage（預設） | `#6D927A` | `#5B8068` | `#EAF2EC` | `#29312D` | `#78817B` | `#F6F7F4` | `#E3E8E4` |
| teal | `#238B8E` | `#19777A` | `#E6F4F3` | `#253033` | `#718083` | `#F5F7F7` | `#E2E8E8` |
| mist-blue | `#5578C8` | `#4568B7` | `#EAF0FC` | `#252B36` | `#747D8C` | `#F5F7FA` | `#E2E6ED` |
| indigo | `#6269D8` | `#5259C6` | `#EEEDFD` | `#292A35` | `#777887` | `#F6F7FA` | `#E5E5EC` |
| terracotta | `#EF8354` | `#DC7045` | `#FFF0E8` | `#27272B` | `#85858E` | `#F6F5F2` | `#E8E6E1` |

固定語意色（不隨主題變）：Success `#55A981`、Warning `#D99A3E`、Danger `#BD5335`。

**對比調整**（依 WCAG 公式計算）
- 原則的 Muted 在五套都不到 4.5:1：對 Background 分別是 3.74／3.82／3.87／4.07／3.35。
- 選取項目的文字（Hover 疊在 Soft 上）在 sage 是 3.89、terracotta 是 2.93，也不到 4.5:1。
- terracotta 的 Accent 疊白字只有 2.61，所以送出鈕的圖示改用深色（Text）。
- 做法：每套主題另外定義「文字用色」`LumiTextMuted`、`LumiTextOnSoft`（選取文字、Soft 底上的 eyebrow），在同一個色相下調深，直到對 Background、Surface、Soft 都 ≥ 4.5:1。原則的 Muted 色碼保留給非文字用途，例如圖示與佔位線。
- 語意色：Success、Warning 疊在白底只有 2.85、2.43，所以文字一律用「Soft 底＋較深的同色文字」（例如原則示範的 `#397760`）。原色只用在圓點與圖示。Danger 疊白底是 4.72，可以直接當文字用。

**實作**
- 每套主題一份 `Themes/Palettes/{id}.axaml`，內含 Light／Dark 兩個 `ThemeDictionaries`。語意色、Code 色留在 `Tokens.axaml`。
- Token 名稱沿用 `Lumi*Brush`，對照原則的名稱如下：`LumiAccent`（Accent）、`LumiAccentHover`（新增）、`LumiAccentSoft`（Soft）、`LumiText`（TextPrimary）、`LumiTextMuted`（TextSecondary）、`LumiBackground`（AppBackground）、`LumiBorder`（Border）、`LumiTextOnSoft`（新增）。其他 token（Surface、Chrome、UserBubble…）由各主題推導。
- 切換主題：`src/Lumi.App/Services/ThemeService.cs` 換掉 `App.axaml` 裡 `MergedDictionaries` 的那一份色票，明暗仍由 `RequestedThemeVariant` 控制。元件一律用 `DynamicResource`，所以會即時生效。
- 設定：`AppSettings`（`src/Lumi.Core/Models/ConnectionSettings.cs:48`）加 `string ThemeId`，`IModelCatalog`／`JsonModelCatalog` 加 `SetThemeIdAsync`。舊設定檔沒有這個欄位，或 ID 不認得時，一律讀成 `sage`。
- Fluent 強調色（`App.axaml`）跟著目前主題的 Accent。
- 切換主題只換顏色，版面與尺寸不變（原則明文規定）。

### 設計 token

| 類別 | 值 |
|---|---|
| 字級 | 頁面標題 28／700、區段標題 20／700、項目標題 14／700、內文 14／400、Caption 11／400 |
| 間距 | 只用 4、8、12、16、24、32（`LumiSpace4`…`LumiSpace32`） |
| 圓角 | 輸入框 10、卡片 14、頁面區域 16、選取的 Menu 項目 10 |
| 陰影 | 只用在浮起（小精靈氣泡、Flyout）與可點擊卡片的 Hover，不拿來分隔內容 |

- 字型仍是打包的 IBM Plex Sans＋Noto Sans TC（ui-guide.html 裡的 Segoe UI 只是網頁示範）。`Typography.axaml` 的內文從 15 改 14、Caption 從 13 改 11，新增標題字級。
- 現有 XAML 裡零散的 6、9、10、14、18 等間距值，改寫時一併收斂到上面的尺度。
- 頁面 Padding：設定頁 32；工作台對話區 Wide 64、Compact 24、Narrow 16；訊息間距 24，`short` 時 16。

### 共用元件樣式（`Themes/Controls.axaml`）

- **TextBox／ComboBox**：高 40、圓角 10、邊框 1。Focus 時用 Accent 邊框加 3px Soft 光暈；TextBox 沒有 BoxShadow 屬性，要覆寫樣式模板。停用時要另外用 Caption 寫出原因。
- **欄位**：標籤放在輸入框上方，右側標「必填／選填」。錯誤時用 Danger 邊框加 Danger 錯誤文字，一定要有文字，不能只有顏色。說明文字用 Caption 放在輸入框下方。
- **按鈕**：
  - `primary`：Text 色底、白字。每個動作區最多一個。
  - `secondary`：外框。
  - `tertiary`：無框。
  - `danger`：紅框紅字，只用在破壞性動作。
  - 對話送出鈕是圖示按鈕，用 Accent 底，不算 Primary 文字按鈕。
- **卡片**：圓角 14、Padding 16，只用來代表一個物件（連線、模型設定、角色、工具卡片、資訊卡）。巢狀最多一層，只有可點擊的卡片才有 Hover。選取狀態是 Soft 底加邊框，並保留文字狀態。**不要把整份表單或每個欄位包成卡片。**
- **分隔線**：1px Border 色，上下間距 16。用來區分群組，不是每個欄位下面都畫一條。
- **狀態徽章**：Soft 底加圓點加文字（例如「連線正常」）。不能只用顏色或圖示表達狀態。

### 斷點

```text
寬度：Wide >= 1000、Compact 620–999、Narrow < 620（hysteresis ±8 px，避免 619/620、999/1000 抖動）
高度：Short < 700、VeryShort < 600
最小視窗：500 × 420
設定頁清單＋詳細：< 900 改上下排
```

- `Lumi.App/Views/Responsive/WindowSizeClass.cs`：列舉加純函式 `Classify(double width, WindowSizeClass previous)`，邊界可以單元測試。
- `ResponsiveClasses` 附加行為：掛在 Window 根節點，監聽 `Bounds`，跨越斷點時才更新根節點 Classes（`wide`／`compact`／`narrow`、`short`／`veryshort`）。不經 ViewModel，因為這是純視覺（`agent.md` 12 節：code-behind 只放視覺的東西）。

### BYOK 顯示

- 模型膠囊顯示**確切的 model ID**（例如 `gpt-5-mini`）。現在顯示的是使用者自訂的 `DisplayName`（`src/Lumi.App/ViewModels/ModelSwitcherViewModel.cs:83`），違反「不改寫真實模型名稱」。顯示名稱降為模型選單與設定清單裡的次要文字。
- 供應商在每個寬度都看得到：Wide／Compact 顯示「供應商 · model ID」；Narrow 用供應商字首圖示加 model ID，完整名稱放在 Tooltip。`ModelSwitcherViewModel` 要提供 `ProviderName`、`ModelId`、`CapsuleText`。
- 模型選單頂部與設定的模型設定頁各加一行 Caption：「實際費用由供應商依模型與 token 用量計算，費用由你的帳號支付。」
- 連線狀態不能只靠圓點：膠囊 Tooltip 和模型選單裡要寫出文字（已連線／連線失敗）。

### 工作台結構

```text
WorkbenchWindow（Grid Auto/Auto/*/Auto）
├─ CustomTitleBar         Lumi 標誌 | 拖曳區 | ✦ 小精靈模式（SpriteModeButton） | 系統按鈕
├─ ChatToolbar            + 新對話 | 歷史對話 | 模型膠囊（CapsuleButton） | …（MenuFlyout）
│  └─ ViewOnlyBanner      查看中橫幅（保留在工具列下方）
├─ SplitView（歷史抽屜；Narrow 改 Overlay）
│  └─ ConversationView    MessageScroller：WelcomePanel、提示、訊息、錯誤列
└─ ChatComposer           InputArea：ContextBar、InputBox、AddFileButton、AddFolderButton、麥克風佔位、SendButton／停止
```

- **標題列**：`ExtendClientAreaToDecorationsHint=True`。
  - Windows：系統按鈕在右，小精靈按鈕緊鄰其左，中間間隔 8～12 並加分隔線。
  - macOS：紅綠燈在左，品牌往右讓出約 80 px，小精靈按鈕放最右。
  - 拖曳區用 `BeginMoveDrag`，不覆蓋任何按鈕。
- **工具列**：新對話、歷史對話是 `secondary`，Compact 起改成純圖示；模型膠囊見「BYOK 顯示」；「…」是 `tertiary` 圖示按鈕。
- **回應中**：小精靈按鈕與模型膠囊停用，原本的「回應中，暫時無法切換」改成停用時的 Tooltip，不佔版面。
- **歡迎區**：對話沒有訊息時才顯示（Soft 底圖示方塊、區段標題 20、Caption 副標）。`narrow` 隱藏副標，`veryshort` 整個隱藏。
- **訊息**：
  - 使用者訊息靠右，上方是「你」和時間，用 UserBubble 底。
  - 助理訊息靠左，有頭像（Soft 底加 Accent 色「L」）、名稱、時間和淡底氣泡。助理氣泡是對話內容，不算卡片；裡面的程式碼區塊、工具卡片算第一層。
  - 動作列：複製、重新回答、👍👎（停用，Tooltip 說明即將推出）、本輪 token，都是 `tertiary`。
  - 寬度：Wide 時助理 ≤ 820、使用者 ≤ 72%；Compact 時助理 92%、使用者 80%；Narrow 同 Compact，並隱藏時間、頭像縮到 24。百分比用一個小的 `FractionOfWidthConverter` 綁 `MessageScroller.Viewport.Width`。
- **Composer**：
  - 上排是輸入框（MinHeight 64，`short` 時兩行高，MaxHeight 160）。
  - 下排是檔案、資料夾、麥克風（停用）等按鈕（`tertiary`），然後是提示文字和送出鈕。
  - Compact 起按鈕只剩圖示，並隱藏提示文字。
  - Chip 換行時不能推走送出鈕。
- **模型選單**：`ModelPicker.axaml` 頂部加「本對話」區塊：輸入／輸出／回合、上下文進度條、連線狀態文字、費用說明。
- **x:Name 全部保留**（測試依賴）：`SendButton`、`InputBox`、`InputArea`、`ContextBar`、`ContextChips`、`AddFileButton`、`AddFolderButton`、`SpriteModeButton`、`ViewOnlyBanner`、`CapsuleButton`、`MessageScroller`、`RetryOtherModelButton` 等。拆到 UserControl 之後，測試改用共用 helper 找控制項：`GetVisualDescendants().OfType<T>().Single(x => x.Name == ...)`。

### 設定視窗

```text
SettingsWindow（自訂標題列，同工作台）
├─ Menu（寬 240）
│  ├─ 標題區：L 標誌＋「Lumi 設定」＋Caption「個人偏好與 AI 連線」
│  ├─ ← 回到對話（tertiary）
│  ├─ AI 連線｜人格與主動性｜外觀與操作｜小精靈
│  ├─ 設定首頁（停用）｜資料與隱私（停用）＋Caption「即將推出」
│  └─ 底部 Caption：版本、「金鑰只儲存在這台裝置」
└─ 內容（Padding 32）：eyebrow＋頁面標題 28＋說明＋該頁唯一的主要動作
   ├─ AI 連線：分段切換「連線／模型設定」（取代 TabControl）
   │  └─ 清單 260（列：圖示、名稱、副標「model · 預設」、狀態文字；虛線「＋新增」）＋ 詳細
   │     詳細：標頭（名稱、供應商、狀態徽章）→ 欄位 → 「進階」區段（供應商格式、API 位置，預設收合）
   │     → 分隔線 → 底部 刪除（danger）｜測試連線（secondary）｜儲存（primary）
   ├─ 人格與主動性、外觀與操作：頁首＋區段標題＋分隔線，欄位不包卡片
   └─ 小精靈：角色清單用可選取卡片；其餘欄位不包卡片
```

- 分類名稱改成白話：「連線與模型」改「AI 連線」，「一般」改「外觀與操作」（放主題 ID 與明暗）。「人格與主動性」「小精靈」維持原名。
- 「外觀與操作」的主題選擇用五張可選取的小卡片，每張顯示主題名稱、說明（原則的副標，例如「柔和、生活化、低刺激」）和色塊。明暗選擇用下拉選單。
- Menu 選取項目用 Soft 底、`LumiTextOnSoft` 文字、圓角 10，主項目 14／600。
- 供應商圖示不用官方商標，改用字首方塊（Soft 底），依 `Format` 決定字首。
- 視窗寬度 < 900 時，清單＋詳細改上下排（清單在上）。
- 「回到對話」：`ISettingsWindowService` 加 `ReturnToWorkbench()`，關閉設定並啟用工作台。
- 標頭的主要動作（例如「＋新增連線」）和詳細區底部的「儲存」分屬不同動作區，各有一個 Primary，符合原則。

## 工作項目

### UA 設計系統基礎
- [x] `Themes/Palettes/` 五套主題 × 淺／深色票（腳本依原則色碼推導，見實作紀錄）；`Tokens.axaml` 只留語意色與 Code 色
- [x] `ThemeService` 切換主題色票；`AppSettings.ThemeId`、`SetThemeIdAsync`；舊設定檔（缺欄位）與未知 ID 讀成 sage，版本號維持 1（附測試）
- [x] 對比測試：五套主題 × 淺／深，所有文字組合 ≥ 4.5:1（`PaletteContrastTests`）
- [x] `Typography.axaml` 新字級；`Metrics.axaml` 的 `LumiSpace*`／`LumiRadius*` token
- [x] `Themes/Controls.axaml`：TextBox、ComboBox（含 Focus 光暈、error、disabled）、四種按鈕、卡片、分隔線、徽章（UA-8）
- [x] `Themes/Icons.axaml`（Lucide 0.468.0 的 22 個圖示）＋`Controls/LumiIcon`；授權檔 `Assets/LICENSES/Lucide-LICENSE.txt`（UA-8）
- [x] `WindowSizeClass`＋`ResponsiveClasses`（只用根節點 Classes），附邊界值單元測試（`ResponsiveRulesTests`）
- [x] `CustomTitleBar`（Windows／macOS 兩種排列）已套在工作台；`TitleBarLayout` 純函式（UA-9）。小精靈按鈕的 Compact／Narrow 縮短留給 UC

### UB 工作台拆元件（Wide）
- [ ] 拆出四個 UserControl，保留全部 x:Name；調整測試 helper，測試全綠
- [ ] 工具列：新對話、歷史、模型膠囊（PathIcon 箭頭、供應商＋model ID）、「…」選單（設定、管理連線與模型）
- [ ] 歡迎區；`MessageViewModel.TimeText`；助理頭像與淡底氣泡；動作列（👍👎 停用）
- [ ] Composer 兩層；麥克風停用佔位；送出／停止改成圖示
- [ ] 移除狀態列；`ModelPicker` 頂部加「本對話」區塊與費用說明；膠囊 Tooltip
- [ ] 回應中改用停用時的 Tooltip，取代原本的提示文字
- [ ] `Markdown.axaml` 程式碼區塊標題列樣式（語言、複製圖示）

### UC RWD
- [ ] Compact：小精靈短標籤、工具列純圖示、Padding 24、訊息百分比寬度、Composer 純圖示、隱藏提示文字
- [ ] Narrow：小精靈只剩 ✦、膠囊改成供應商字首加 model ID、Padding 16、隱藏時間、頭像 24、歷史抽屜 Overlay
- [ ] 高度規則：`short`、`veryshort`
- [ ] `MinWidth=500`
- [ ] 所有純圖示按鈕補上 `ToolTip.Tip` 與 `AutomationProperties.Name`

### UD 設定視窗
- [ ] 自訂標題列；Menu（標題區、回到對話、白話分類名稱、兩個停用佔位）
- [ ] 頁首元件（eyebrow、頁面標題、說明、主要動作）
- [ ] AI 連線：清單＋詳細、狀態徽章、「進階」收合、按鈕層級；模型設定頁同樣做法加上費用說明；用分段切換取代 TabControl；頁首主要動作隨分段切換（新增連線／新增模型設定）
- [ ] 外觀與操作：五張主題卡片＋明暗選擇
- [ ] 人格與主動性、小精靈：頁首＋區段＋分隔線；角色清單用可選取卡片
- [ ] 寬度 < 900 時上下排

### UE 收尾
- [ ] `ToolCardView`、`ToolRunView`、`ConfirmDialog`、`SpriteBubbleView` 換 token、圖示與按鈕層級
- [ ] 鍵盤 Tab 順序、Enter／Shift+Enter
- [ ] `docs/ui-spec.md`：改寫 1.2 色票（改成主題系統）、2.1 工具列、2.3 狀態列、3 訊息流、6 輸入區、7 設定視窗，新增 RWD 一節，並註明以 `ui-guide.html` 為準
- [ ] `agent.md` 第 12 節：寫明設計原則以 `ui-guide.html` 為準；元件只能引用 token，不能寫死色碼與零散間距；圖示一律用 `Icons.axaml`；斷點規則
- [ ] 更新 `00-master-plan.md` 的狀態

## 驗收標準

1. `dotnet build`、`dotnet test` 全綠，包含 Headless 測試，以及新增的斷點、對比、主題設定測試。
2. 用測試矩陣 1200×800、900×700、700×650、620×650、520×650、800×560 截圖：sage 的淺色與深色各一次，另外四套主題在 1200×800 各截一張。每張都要符合 `RWD原則.md` 第 7 節的每一項檢查。
3. 從 1200 拖到 500 時，不出現水平捲軸；輸入框與送出鈕完整可見；在 619/620、999/1000 不抖動；拖曳過程中對話列表不重建、不閃爍。
4. BYOK：任何寬度都看得到供應商和確切的 model ID；長 model ID 會省略，不會推走其他按鈕；模型選單與設定頁有費用說明。
5. 小精靈按鈕位置固定，只切換文字；所有純圖示按鈕都有 Tooltip 與可存取名稱。
6. 五套主題 × 淺／深的所有文字組合 ≥ 4.5:1；切換主題只換顏色，版面與尺寸不變，而且立即生效。
7. 照 ui-guide.html 逐條檢查：
   - 每個動作區最多一個 Primary 按鈕。
   - 錯誤一定有文字說明，停用一定有說明。
   - 卡片巢狀不超過一層，沒有把表單欄位包成卡片。
   - 間距只用 4／8／12／16／24／32。
   - 沒有模糊、玻璃效果或漸層滿版。
8. macOS 標題列的紅綠燈不與品牌或按鈕重疊，而且可以拖曳。Windows 由使用者回報（同 M1 的做法）。
9. 原有功能不退化：回應中停用切換、查看中、權限確認、上下文 chip、錯誤列與重試、歷史抽屜、設定各頁存檔、光球顏色不變。

## 待人工處理

（同一任務修 3 次仍過不了的項目記在這裡：任務編號、卡住的原因、已嘗試的做法。）

## 人工驗收

（需要人眼確認的項目：字型、顏色、縮放、動畫、平台差異。subagent 完成任務後把待確認項目補在這裡，不中斷流程。）

- [ ] UA：切換五套主題色票（設定 › 一般），確認 Fluent 強調色（核取方塊、選取、進度條）跟著變
- [ ] UA：五套主題的淺色與深色截圖，確認色票觀感、Caption 11px 的中文可讀性
- [ ] UA：選取列（歷史抽屜、設定 Menu）的 Soft 底與文字在各主題上看得清楚
- [ ] UA-8：深色模式的 TextBox／ComboBox 是主題的深灰 Surface，不是純黑
- [ ] UA-8：Focus 的 3px Soft 光暈在五套主題的淺色與深色都看得出來（深色 Soft 偏暗，例如 sage `#2B312D`）
- [ ] UA-8：ComboBox 的 chevron 線寬與位置（右側 12px）、點擊展開正常；下拉清單選取與 Hover 看得清楚
- [ ] UA-8：可編輯 ComboBox（設定頁模型 ID）沒有內層雙框或雙光暈
- [ ] UA-8：密碼欄、多行輸入框（工作台 InputBox、PersonaCustomBox）高度與文字對齊正常
- [ ] UA-8：**小精靈視窗輸入框**因全域 TextBox 樣式變成 40 高，確認外觀沒有跑版（計畫規定小精靈版面不動，若不要這個變化要加 class 排除）
- [ ] UA-9：macOS 實機，紅綠燈不與品牌重疊（讓出 80px 是否剛好）、標題列高 40 時紅綠燈垂直置中
- [ ] UA-9：macOS 拖曳標題列空白處可移動視窗，雙擊可縮放
- [ ] UA-9：Windows（使用者回報）系統按鈕疊在右側，小精靈按鈕與分隔線在其左；保留寬度 146（常數，Avalonia 無 API）不重疊也不留太大縫隙；不同 DPI／Windows 版本都要看
- [ ] UA-9：Windows 拖曳與雙擊最大化沒有和系統標題列處理衝突
- [ ] UA-9：小精靈按鈕 Soft 底、圖示與文字對比、Hover 邊框、停用外觀；回應中滑過停用按鈕出現「回應中，暫時無法切換」
- [ ] UA-9：標題列與工具列都是 Chrome 色，銜接自然（工具列下緣有 1px 邊線，標題列沒有）
- [ ] UA-9：Lumi 標誌方塊在各主題（尤其 terracotta）上的「L」清楚
- [ ] UA-8：LumiIcon 在 16px 的線條粗細（1.75 縮放後約 1.17px）看起來順眼

## 實作紀錄

### UA（2026-10-22）
- **色票產生**：五套 × 淺／深由腳本從原則色碼推導（腳本放在 scratchpad，不在專案內）。淺色的背景、邊框、文字、Accent、Hover、Soft 直接用原則色碼；Muted、AccentText 在同色相下調深到 4.5:1；OnAccent 取白或文字色中對比較高的一個（只用在送出鈕的圖示，門檻 3:1）。深色由同一色相推導。
- **偏離計畫 1：色碼檔位置**。`ThemeTokenRuleTests` 原本只允許 `Tokens.axaml` 含色碼；五套色票各一份檔案，所以規則改為「色碼只能在 `Themes/` 資料夾」。這是架構規則的調整，請確認。
- **偏離計畫 2：選取列**。`ListBoxItem:selected` 改成 Soft 底＋`LumiTextOnSoft` 文字（原本是 Accent 底＋OnAccent 文字，sage 上只有 3.85:1）。
- **語意色沒有重新上色**。`LumiSuccess／Danger／Warning` 的值保留原樣（文字用色已達 4.5:1）。原則的 `#55A981`、`#D99A3E`、`#BD5335` 要在 UB、UD 改寫圖示與狀態點時再套用，並為文字另外提供較深的 token。
- **Fluent 強調色**改成 `DynamicResource LumiAccentColor`，切換主題時跟著變（需實機確認）。
- **未完成**：Icons、CustomTitleBar、TextBox 光暈、ComboBox 樣式。
- 測試：全部專案通過（App 633、Core 229、Infrastructure 97、Architecture 58）。

### UA-8（Icons、輸入框與下拉選單）
- 圖示來自 Lucide（網路可用），線寬 1.75 縮放；`LumiIcon` 的 Foreground 與 TextBlock 共用繼承屬性，放在 Button 裡會跟著按鈕文字色。
- TextBox／ComboBox 重寫模板，Focus 光暈是模板內多一層透明外框（Margin -3），被會裁切的父層切到時看不到。
- 全域 TextBox 樣式也套到 SpriteWindow 的輸入框（見人工驗收）。
- 測試：App 662、其餘不變，全綠。

### UA-9（CustomTitleBar）
- Windows 系統按鈕寬度沒有 API，右側保留寬度用常數 138＋8 間距；`button.IsEffectivelyEnabled` 才反映命令 CanExecute。
- 工具列暫時保留原樣（膠囊旁「回應中」文字、設定按鈕），UB 重做。
- 測試：App 675，全綠。

- 2026-10-22 補完三項（設定檔版本、只用 Classes、頁首主要動作隨分段切換），狀態改為進行中。
