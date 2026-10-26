# 工作台頂端列分層與用量圖示

> 狀態：**已確認**（2026-10-27）。實作 M1／M2 工作台介面時以這份為準。
> 取代 `docs/ui-spec.md` 的 2.1「頂端工具列」、2.3「底部狀態列」，以及第 3 節「每則助理訊息下方的 token 數」。
> 視覺稿：Claude Design 畫布「Lumi 用量圖示」，畫板 1–6。
> **實作時與本稿不同的地方（尺寸維持 32、膠囊文字不變、保留 token 進階開關、色彩用既有 token 並跟著主題）以 [21-chat-topbar-and-home.md](21-chat-topbar-and-home.md) 的「與已確認稿的差異」為準。**

---

## 0. 給實作者（含 AI）的摘要

1. 頂端分兩列，**一列一個層級**：第一列是整個 App，第二列只放「目前這則對話」。
2. 設定改成第一列右側的**齒輪圖示**，不再藏在「⋯」裡。
3. 「⋯」留在第二列最右側，只放**對話動作**（重新命名、匯出、刪除）。
4. 用量從模型選單裡移出，改成第二列右側一個獨立的**圓環圖示** `UsageRing`，點一下開 Flyout 看詳情。
5. 模型選單只負責選模型。
6. 底部狀態列與每則訊息下方的 token 數都移除。

---

## 1. 版面

```
┌────────────────────────────────────────────────────────────────────┐
│ ●●●  Lumi                                      [✦ 小精靈]  [⚙]     │  第一列：App（Chrome 底）
├────────────────────────────────────────────────────────────────────┤
│ [+] [🕘]            ( ● deepseek-flash ⌄ )              [◔] [⋯]    │  第二列：這則對話（Surface 底）
├────────────────────────────────────────────────────────────────────┤
│                           訊息流 …                                  │
```

Windows 11（自繪標題列）：

```
│ ◉ Lumi                              [✦ 小精靈]  [⚙] │  ─   □   ✕ │
```

- 小精靈與齒輪放在系統視窗按鈕**左邊**，中間一條 1px × 18px 的 Border 色分隔線，避免誤按關閉。
- Windows 標題列左側顯示小光球（琥珀 `#F2B544`，帶微光）＋「Lumi」。macOS 左側是紅綠燈，再接「Lumi」。

### 1.1 第一列：App 層級

| 項目 | 規格 |
|---|---|
| 高度 | macOS 44，Windows 40 |
| 底色 | `Chrome` |
| 左右內距 | 左 16，右 12（Windows 右側為系統按鈕，不加內距） |
| 小精靈按鈕 | 高 32、圓角 9、左右內距 14、圖示 16＋文字 14／500；底 `AccentSoft`、字 `AccentText`。行為同原規格（回應中停用，見 ui-spec 2.2） |
| 設定按鈕 | 32×32、圓角 9、無框、透明底；齒輪圖示 19px、線寬 1.8、色 `Text`；hover 底 `SurfaceAlt`；tooltip「設定」；`AutomationProperties.Name="設定"`。點擊開設定獨立視窗 |
| 間距 | 小精靈與齒輪間距 8 |

### 1.2 第二列：對話層級

| 項目 | 規格 |
|---|---|
| 高度 | 56 |
| 底色 | `Surface`，上下各 1px `Border` |
| 左右內距 | 16，元素間距 8 |
| 左 | 新對話、歷史對話：36×36、圓角 10、1px `BorderStrong` 框、`Surface` 底、圖示 18 |
| 中 | 模型膠囊：高 36、圓角 18、1px `BorderStrong`；內容 = 狀態點 8px＋模型顯示名稱（14／500）＋下拉箭頭 14。**不再顯示連線名稱**（選單內已依連線分組） |
| 右 | 用量圖示 `UsageRing`（見第 2 節），接著「⋯」對話選項 |
| 「⋯」對話選項 | 36×36、圓角 10、1px `BorderStrong`、`Surface` 底；`AutomationProperties.Name="對話選項"` |

「⋯」選單（MenuFlyout，寬 200，向下、右緣對齊）：

1. 重新命名對話
2. 匯出為 Markdown
3. 刪除這則對話（文字 `Danger`；點擊後二次確認）

回應中：「重新命名」可用；「匯出」「刪除」停用。

---

## 2. 用量圖示 `UsageRing`

### 2.1 外觀

- 按鈕 36×36、圓角 10、1px `BorderStrong` 框、`Surface` 底，內含 22×22 的圓環。
- 圓環幾何（以 22×22 畫布計）：圓心 (11,11)、半徑 8、線寬 3、圓頭線端；軌道為整圈，進度弧從 12 點鐘方向**順時針**畫。
- 進度 = `ContextUsed / ContextLimit`，夾在 0–1。

| 等級 | 條件 | 進度弧顏色 | 額外 |
|---|---|---|---|
| `Normal` | < 60% | `UsageNormal` | — |
| `Warning` | 60%–85% 未滿 | `UsageWarning` | — |
| `Danger` | ≥ 85% | `UsageDanger` | 按鈕底改 `WarningBg`、框改 `WarningBorder`；右上角 10px 紅點（`UsageDanger`，2px `Surface` 外框，位移 -4,-4） |
| `Empty` | 還沒有任何 usage | — | 只畫軌道，改為虛線（線寬 2、色 `#B4B9C2`／深色 `#5A606B`、dash 2.5 / 3.2） |

- 面板開啟中：按鈕框改 `UsageNormal`、底改 `SelectedSoft`。
- hover tooltip：`上下文 9% · 點一下看詳情`；`Empty` 時為 `尚無用量資料`。
- 無障礙名稱：`本對話用量：上下文 9%`。

參考 SVG（Normal、9%）：

```svg
<svg width="22" height="22" viewBox="0 0 22 22">
  <circle cx="11" cy="11" r="8" fill="none" stroke="#E1E3E8" stroke-width="3"/>
  <circle cx="11" cy="11" r="8" fill="none" stroke="#5B5BD6" stroke-width="3"
          stroke-linecap="round" stroke-dasharray="4.52 50.27"
          transform="rotate(-90 11 11)"/>
</svg>
```

### 2.2 新增色彩 token

加進 ui-spec 1.2 的 token 表：

| Token | 淺色 | 深色 | 用途 |
|---|---|---|---|
| `UsageTrack` | `#E1E3E8` | `#3B3E5E` | 小圓環軌道 |
| `UsageTrackLarge` | `#ECEDF1` | `#2E323A` | 面板大圓環軌道 |
| `UsageNormal` | `#5B5BD6` | `#8E8EF0` | 正常進度、選取框 |
| `UsageWarning` | `#D98A00` | `#F2B544` | 過半 |
| `UsageDanger` | `#C2410C` | `#F08A5D` | 快滿、紅點 |
| `SelectedSoft` | `#ECEBFB` | `#262844` | 按鈕開啟中底色、選取列底色 |

> `UsageNormal` 沿用目前實作的靛藍色。若之後統一回 `Info` 或琥珀，只改 token 值，不動元件。

### 2.3 Avalonia 實作建議

- 圓環用 `Avalonia.Controls.Shapes.Arc` 兩個疊起來（軌道 `SweepAngle=360`，進度 `StartAngle=-90`、`SweepAngle=360*ratio`），或自訂 `Control` 覆寫 `Render`，用 `StreamGeometry.ArcTo` 畫。大小兩種圓環共用同一個 `UsageRing` 控制項，以 `Diameter`、`Thickness` 屬性區分。
- `ratio` 變化時用 200ms ease-out 動畫補間 `SweepAngle`；`ratio == 0` 時不畫進度弧（避免圓頭線端畫出一個點）。
- 按鈕用 `Button` + `Button.Flyout`（`Flyout`，`Placement="BottomEdgeAlignedRight"`，`VerticalOffset=6`）。Esc、點外面關閉。
- 顏色以 `Level` 透過樣式選擇器切換（`:normal`／`:warning`／`:danger`／`:empty` pseudo-class），不在程式碼寫死色碼。

---

## 3. 用量面板（Flyout）

寬 320、內距 18、圓角 14、1px `Border`、`Surface` 底、陰影 `0 12 32 rgba(20,24,32,.16)`（深色 `.45`）。區塊間距 16。由上而下：

1. **標題列**：左「本對話用量」（14／600）；右為目前模型顯示名稱（12、`TextMuted`）。
2. **上下文區**（左右排列，間距 18）
   - 左：大圓環，直徑 104、半徑 44、線寬 10，顏色規則同 2.1。圓心疊兩行文字：百分比（22／600，tabular nums）、「上下文」（12、`TextMuted`）。
   - 右：三行
     - `12K / 128K`（16／500；「/ 128K」用 `TextMuted`、400）
     - `還可以再用約 116K`（13、`TextMuted`）
     - 狀態文字（13／500）：`Normal` →「空間充足」`Success`；`Warning` →「已用超過一半」`WarningText`；`Danger` →「快滿了，建議開新對話」`Danger`
3. **三個小方塊**（3 欄等寬、間距 8；每格 `SurfaceAlt` 底、圓角 10、內距 10）
   - **輸入**：左上一個 18px 迷你圓環（半徑 6.5、線寬 3、`Success`）表示快取命中率；數字（17／600）；下方「快取 N%」（12、`Success`）。供應商沒有回報快取時，迷你圓環改成輸入箭頭圖示，下方寫「tokens」。
   - **輸出**：向下箭頭圖示；數字；「tokens」。
   - **回合**：迴轉箭頭圖示；數字；「來回」。
4. **連線列**（上方 1px `Border` 分隔線、上內距 12）：8px 狀態點＋`已連線 · {連線名稱}`。狀態：已連線 `Success`、尚未確認 `UsageWarning`、連線失敗 `Danger`（文字改「連線失敗 · 重試」，「重試」可點）。
5. **說明**（12、`TextMuted`、行高 1.5）：`費用由供應商依 token 用量計算，從你的帳號扣款。`

### 3.1 快滿提醒（`Danger` 時才出現）

插在第 2 區與第 3 區之間：`WarningBg` 底、1px `WarningBorder`、圓角 10、內距 12。

- 文字（13、`WarningText`）：「對話快超過模型上限，再聊下去較早的內容可能會被截掉。」
- 按鈕：「開新對話」（`PrimaryButton`，高 32）、「繼續這則」（次要按鈕，只關閉面板）。
- 「開新對話」：回應中停用。

### 3.2 數字格式

- `< 1,000`：原數字（`31`）。
- `≥ 1,000`：除以 1,000，`< 10K` 留一位小數（`9.6K`），`≥ 10K` 取整數（`118K`）。百萬以 `M` 同規則。
- 百分比四捨五入到整數；有用量但 `< 1%` 顯示「<1%」，圓環仍畫出最小 2° 的弧。
- 「還可以再用約」= `ContextLimit - ContextUsed`，下限 0。

---

## 4. 資料來源

ViewModel（建議名稱 `ConversationUsageViewModel`，掛在目前對話上）：

| 屬性 | 來源與定義 |
|---|---|
| `ContextLimit` | 目前模型設定的「上下文上限」 |
| `ContextUsed` | **最近一次** API 請求的 input tokens（含快取讀取）＋ output tokens。代表下一次請求大約會送出的量 |
| `InputTokens` | 本對話所有請求的 input tokens **累計**（含快取） |
| `CachedInputTokens` | 本對話所有請求的快取讀取 tokens 累計 |
| `OutputTokens` | 本對話所有請求的 output tokens 累計 |
| `TurnCount` | 使用者送出的次數（一次送出可能觸發多次工具迴圈請求，仍算一回合） |
| `ConnectionState` | `Unknown`／`Connected`／`Failed` |
| `Ratio`、`Level`、各種顯示字串 | 由上面計算，UI 只綁定這些 |

各供應商的快取欄位：

- Anthropic：`usage.cache_read_input_tokens`（`input_tokens` 不含快取；總輸入 = `input_tokens` + `cache_creation_input_tokens` + `cache_read_input_tokens`）
- OpenAI 相容：`usage.prompt_tokens_details.cached_tokens`（已包含在 `prompt_tokens` 內）
- DeepSeek：`usage.prompt_cache_hit_tokens`（已包含在 `prompt_tokens` 內）
- 取不到時視為「不支援快取回報」，見 3. 的輸入方塊備援畫法。

更新時機：每次收到供應商 `usage` 就更新（串流結束、或 `message_delta` 帶 usage 時）。切換對話時整組換成該對話的值；新對話為 `Empty`。

> 待實作前確認：輸入／輸出採**累計**，與目前實作（只顯示最近一次）不同。

---

## 5. 模型選單（精簡後）

- 只保留：搜尋框、依連線分組的模型清單、「管理連線與模型…」。
- 移除原本上半部的「本對話」用量、進度條、連線狀態、費用說明。
- 寬 340、內距 10、圓角 14；每列為顯示名稱（14）＋模型 ID（12、`TextMuted`）；目前選取列 `SelectedSoft` 底並在左側打勾。

---

## 6. 回應中的行為（補充 ui-spec 2.2）

- 用量圖示與面板**都可以用**，只是唯讀；數字在每次收到 `usage` 時更新。
- 設定齒輪可用（開設定視窗不影響回應）；設定裡改到目前模型的值，下一回合才生效。
- 「⋯」選單依 1.2 停用部分項目。
- 「查看中」狀態（小精靈回應中開啟工作台）：同上。

---

## 7. 移除項目

- `ui-spec` 2.3 底部狀態列：整條移除，資訊改由用量面板提供。
- `ui-spec` 第 3 節：每則助理訊息下方只留「複製」「重新產生」，不再顯示 `↑ 輸入 · ↓ 輸出 tokens`。
- 模型選單的用量區塊（見第 5 節）。

---

## 8. 驗收

- [ ] 第一列只有 App 層級的東西（小精靈、設定）；第二列只有對話層級的東西。
- [ ] Windows 11 自繪標題列上，齒輪與關閉鈕之間有分隔線，且可以拖曳標題列空白處移動視窗。
- [ ] 用量圖示在 0%、59%、60%、84%、85%、100% 的顏色與紅點都正確；新對話顯示虛線圓環。
- [ ] 面板數字與供應商回傳的 `usage` 一致；Anthropic、OpenAI 相容、DeepSeek 三種的快取命中率都算對。
- [ ] 不回報快取的供應商（例如 Ollama），輸入方塊改為箭頭圖示，不顯示「快取 0%」。
- [ ] 淺色、深色模式下文字對比皆 ≥ 4.5:1。
- [ ] 1920×1080 在 100%、125% 縮放下，22px 小圓環清晰、不糊邊。
- [ ] 鍵盤可 Tab 到用量圖示並以 Enter／Space 開啟面板，Esc 關閉，焦點回到圖示。
