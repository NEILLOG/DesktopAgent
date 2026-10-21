# Lumi 對話視窗 RWD 指導原則

> 適用技術：C#、Avalonia、XAML、MVVM  
> 適用範圍：Lumi 完整對話視窗，不包含無邊框「小精靈模式」本體  
> 設計目標：在視窗寬度降低時，保留模型識別、訊息閱讀、輸入及模式切換等核心功能，不以等比例縮放整個 UI。

---

## 1. 核心原則

1. **重新排列，不做整頁縮放**：縮窄時改變元件排列、文字顯示與間距，不使用 `RenderTransform` 縮小整個畫面。
2. **輸入框永遠完整可見**：主版面固定為 `Auto / * / Auto` 三列，只有中間對話區伸縮及捲動。
3. **模型資訊不可消失**：Lumi 採 BYOK，模型會影響費用，因此窄版仍應顯示模型名稱。
4. **小精靈入口位置固定**：按鈕永遠位於系統視窗控制按鈕左側，只依寬度縮短文字。
5. **漸進式精簡**：先隱藏次要文字，再縮短標籤，最後才改為純 Icon。
6. **程式碼不強制折行**：Code Block 保留等寬字體與水平捲動，避免破壞縮排。
7. **寬度與高度分開處理**：窄視窗處理橫向空間，低視窗則減少歡迎區、訊息間距與輸入框預設行數。

---

## 2. RWD 斷點

### Wide：寬度 >= 1000 px

完整桌面模式。

- 顯示「小精靈模式」完整標籤。
- 顯示「新對話」與「歷史對話」的 Icon 和文字。
- 模型選擇器顯示供應商與完整模型名稱。
- 對話區左右 Padding 建議 `64–96 px`。
- AI 訊息最大寬度建議 `760–840 px`。
- 使用者訊息最大寬度建議為內容區的 `70–75%`。
- 輸入框工具可顯示 Icon 與文字。

### Compact：寬度 620–999 px

主要窄版模式，適合一般桌面視窗縮至約一半。

- 「小精靈模式」縮成「小精靈」。
- 新對話與歷史對話改為純 Icon，使用 Tooltip 說明。
- 模型選擇器保留 `供應商 · 模型名稱`，過長時使用省略號。
- 對話區左右 Padding 改為 `20–24 px`。
- AI 訊息最大寬度為內容區的 `92%`。
- 使用者訊息最大寬度為內容區的 `80–85%`。
- 輸入框內的檔案、資料夾、麥克風改為純 Icon。
- 隱藏 `Enter 傳送 · Shift + Enter 換行` 提示。
- 回覆操作列可只顯示 Icon。

### Narrow：寬度 < 620 px

極窄桌面模式。

- 小精靈按鈕只顯示 `✦`，保留 Tooltip。
- 模型選擇器只顯示模型名稱，例如 `gpt-5-mini`。
- 對話區左右 Padding 改為 `12–16 px`。
- 隱藏歡迎區副標題；高度不足時連歡迎區一起隱藏。
- 訊息時間與非必要名稱可以隱藏。
- AI Avatar 可縮至 `24 px`。
- 所有附件操作改為 Icon。
- 建議最小視窗寬度設為 `500 px` 左右；低於此值時提示使用小精靈模式，不建議繼續壓縮完整對話 UI。

---

## 3. 各區域行為

### 3.1 自訂標題列

排列方式：

```text
Wide    Lumi                    [✦ 小精靈模式] | [-] [□] [×]
Compact Lumi                         [✦ 小精靈] | [-] [□] [×]
Narrow  Lumi                                [✦] | [-] [□] [×]
```

- 小精靈按鈕與系統視窗按鈕間保留 `8–12 px`，可加入淡色分隔線。
- 小精靈按鈕最小點擊範圍建議 `36 x 32 px`。
- 不要將小精靈按鈕放進內容工具列，避免視窗層級功能隨內容排列而漂移。
- 標題列可拖曳區域不可覆蓋互動按鈕。

### 3.2 內容工具列

```text
Wide    [+ 新對話] [歷史對話] [OpenAI · gpt-5-mini v] [...]
Compact [+] [歷史 Icon]        [OpenAI · gpt-5-mini v] [...]
Narrow  [+] [歷史 Icon]              [gpt-5-mini v] [...]
```

- Icon-only 按鈕必須提供 `ToolTip.Tip` 與可存取名稱。
- 模型文字使用 `TextTrimming="CharacterEllipsis"`，不要讓模型選擇器把其他按鈕推出畫面。
- 模型選擇器應設定合理 `MinWidth` 與 `MaxWidth`，並讓中間文字欄位承擔伸縮。
- 下拉箭頭建議使用固定尺寸 `PathIcon`，不要使用文字字元 `⌄`，避免不同字型造成垂直跑版。

### 3.3 對話區

- 對話列表放在獨立 `ScrollViewer` 中。
- 新訊息抵達時，僅在使用者接近底部時自動捲到底；使用者正在閱讀舊訊息時不要強制跳動。
- 訊息 Bubble 以百分比概念限制寬度，但 Avalonia XAML 沒有直接百分比 `MaxWidth` 時，可在父容器尺寸改變時由 Converter 或自訂 Panel 計算。
- 窄版不要把左右兩種訊息都改為滿版，仍保留使用者靠右、AI 靠左的辨識性。

### 3.4 Code Block

- 外層寬度跟隨 Bubble。
- 內層使用水平 `ScrollViewer`。
- `TextWrapping="NoWrap"`。
- 建議最小字級 `12–13 px`，不要為塞入更多程式碼而繼續縮字。
- 複製按鈕固定於 Code Block 標題列右側，不隨程式碼水平捲動。

### 3.5 輸入框

窄版採上下兩層：

```text
輸入文字區
--------------------------------
[加入檔案] [加入資料夾] [麥克風]                 [送出]
```

- 主視窗使用 `Grid RowDefinitions="Auto,*,Auto"`。
- 輸入框必須位於最後一列，不可放入對話 `ScrollViewer`。
- 文字輸入區建議 `MinHeight="64"`、`MaxHeight="160"`。
- 送出按鈕固定在工具列最右側。
- 錄音中不可只靠顏色表示，應同步改變 Icon、Tooltip 或顯示短文字。
- 附件名稱應在輸入框上方以可移除 Chip 呈現；窄版允許換行，但不可推走送出按鈕。

---

## 4. Avalonia 建議施作方式

### 4.1 優先採用 Container Query 或斷點 ViewModel

Avalonia 官方文件將響應式方案區分為 Container Query、`OnFormFactor`、可回流 Panel 與斷點 ViewModel。Lumi 是桌面視窗，而且需要在使用者即時拖曳視窗時切換，因此：

- **若專案版本支援 Container Query**：元件級 RWD 優先採用 Container Query。
- **若使用 Avalonia 11 或需一次切換多個屬性**：採用斷點 ViewModel 最穩定。
- **不要用 `OnFormFactor` 處理視窗縮放**：`OnFormFactor` 適合啟動時依裝置類型切換，並不負責桌面視窗即時 Resize。

官方參考：

- https://docs.avaloniaui.net/docs/layout/responsive-layouts
- https://docs.avaloniaui.net/docs/how-to/responsive-layout-how-to

### 4.2 ViewModel 斷點設計

```csharp
public enum WindowSizeClass
{
    Wide,
    Compact,
    Narrow
}

public partial class ChatViewModel : ObservableObject
{
    [ObservableProperty]
    private WindowSizeClass sizeClass = WindowSizeClass.Wide;

    public void UpdateWindowSize(double width)
    {
        SizeClass = width switch
        {
            >= 1000 => WindowSizeClass.Wide,
            >= 620  => WindowSizeClass.Compact,
            _       => WindowSizeClass.Narrow
        };
    }
}
```

在 View 的 `Bounds` 變更時更新狀態。若 Resize 造成大量版面更新，可加入約 `50–100 ms` debounce，但一般三段式切換只在跨越斷點時更新即可。

```csharp
protected override void OnPropertyChanged(AvaloniaPropertyChangedEventArgs change)
{
    base.OnPropertyChanged(change);

    if (change.Property == BoundsProperty && DataContext is ChatViewModel vm)
    {
        vm.UpdateWindowSize(Bounds.Width);
    }
}
```

實務上可封裝為 Behavior 或由 Window code-behind 專門轉送尺寸，避免 ViewModel 直接依賴 `Window`。

### 4.3 使用 Classes 切換樣式

在視窗取得 `SizeClass` 後，替根節點設定 `wide`、`compact`、`narrow` Class，再由 Style 集中管理，不要在大量控制項各自寫 Converter。

概念範例：

```xml
<Style Selector=".compact .full-label">
    <Setter Property="IsVisible" Value="False" />
</Style>

<Style Selector=".compact .compact-label">
    <Setter Property="IsVisible" Value="True" />
</Style>

<Style Selector=".narrow .secondary-copy">
    <Setter Property="IsVisible" Value="False" />
</Style>
```

若現有架構已使用 `DataTrigger`，也可以直接綁定 `SizeClass`。不過當規則數量增加時，根節點 Classes 通常較容易維護。

### 4.4 Container Query 範例

以下語法適用於支援 Container Query 的 Avalonia 版本，請以專案實際版本文件為準：

```xml
<Border Container.Name="chat" Container.Sizing="Width">
    <Border.Styles>
        <ContainerQuery Name="chat" Query="max-width:999">
            <Style Selector="Button.toolbar-text TextBlock">
                <Setter Property="IsVisible" Value="False" />
            </Style>
        </ContainerQuery>

        <ContainerQuery Name="chat" Query="max-width:619">
            <Style Selector="TextBlock.welcome-subtitle">
                <Setter Property="IsVisible" Value="False" />
            </Style>
        </ContainerQuery>
    </Border.Styles>

    <!-- Chat content -->
</Border>
```

Container Query 比直接觀察 Window 更適合可重用元件，例如模型選擇器、輸入 Composer 或未來嵌入其他頁面的聊天面板。

### 4.5 主版面配置

```xml
<Grid RowDefinitions="Auto,*,Auto">
    <local:ChatToolbar Grid.Row="0" />

    <ScrollViewer Grid.Row="1"
                  VerticalScrollBarVisibility="Auto"
                  HorizontalScrollBarVisibility="Disabled">
        <ItemsControl ItemsSource="{Binding Messages}" />
    </ScrollViewer>

    <local:ChatComposer Grid.Row="2" />
</Grid>
```

關鍵是中間列使用 `*`；上方工具列與下方輸入框使用 `Auto`。不要替整個聊天頁指定固定高度，也不要讓輸入框參與訊息區的捲動。

### 4.6 模型選擇器避免箭頭跑版

```xml
<Grid ColumnDefinitions="Auto,*,Auto" ColumnSpacing="8">
    <Ellipse Grid.Column="0" Width="7" Height="7" Fill="#55A981" />

    <TextBlock Grid.Column="1"
               Text="{Binding CurrentModelDisplayName}"
               TextTrimming="CharacterEllipsis"
               VerticalAlignment="Center" />

    <PathIcon Grid.Column="2"
              Width="12"
              Height="12"
              VerticalAlignment="Center"
              Data="{StaticResource ChevronDownPath}" />
</Grid>
```

- 箭頭使用 `PathIcon`，不用 Unicode 字元。
- 文字放在 `*` 欄位並啟用 Ellipsis。
- Icon 放固定 `Auto` 欄位，確保不被文字擠壓。

### 4.7 小精靈模式切換

- 正常模式與小精靈模式建議使用兩個明確 View 或兩個 Window，而不是在同一棵大型 Visual Tree 上切換所有控制項。
- 小精靈 Window 使用無邊框、透明背景、Topmost 是否啟用由使用者設定決定。
- 切換前保存完整視窗的 `Position`、`Width`、`Height` 與 `WindowState`。
- 回到完整模式時恢復原位置，並限制在目前螢幕 WorkingArea 內，避免外接螢幕拔除後視窗出現在畫面外。
- 跨 Windows、macOS、Linux 測試透明視窗、拖曳、陰影與 DPI，這些行為常有平台差異。

---

## 5. 高度不足時的規則

除了寬度，也建議加入高度條件：

### 高度 < 700 px

- 縮小歡迎區上下 Margin。
- 輸入框預設由三行改成兩行。
- 訊息垂直間距由約 `22 px` 降至 `14–16 px`。

### 高度 < 600 px

- 隱藏歡迎區。
- 保留工具列、對話 ScrollViewer 及輸入框。
- 輸入框高度不得低於可輸入兩行文字及一列工具按鈕的高度。

---

## 6. 建議元件拆分

```text
ChatWindow
├─ CustomTitleBar
│  ├─ Brand
│  ├─ CompanionModeButton
│  └─ WindowControls
├─ ChatToolbar
│  ├─ NewChatButton
│  ├─ HistoryButton
│  ├─ ModelSelector
│  └─ MoreButton
├─ ConversationView
│  ├─ WelcomePanel
│  ├─ UserMessageBubble
│  ├─ AssistantMessageBubble
│  └─ CodeBlock
└─ ChatComposer
   ├─ AttachmentChips
   ├─ MessageTextBox
   ├─ FileButton
   ├─ FolderButton
   ├─ MicrophoneButton
   └─ SendButton
```

每個元件只處理自身的 Compact/Narrow 差異，避免單一 XAML 檔案累積大量條件。

---

## 7. 測試矩陣

至少測試下列視窗尺寸：

- `1200 x 800`：完整寬版。
- `900 x 700`：Compact 上緣。
- `700 x 650`：典型半寬視窗。
- `620 x 650`：Compact/Narrow 邊界。
- `520 x 650`：建議最小寬度附近。
- `800 x 560`：高度不足情境。

每個尺寸都要檢查：

- 標題列按鈕沒有重疊，且仍有可拖曳區域。
- 模型名稱可辨識，箭頭沒有位移。
- 長模型名稱會省略，不會推走其他按鈕。
- 長中文、英文與沒有空白的 URL 不會撐破 Bubble。
- Code Block 可水平捲動。
- 輸入框完整可見。
- 附件 Chip 換行後仍可送出訊息。
- 125%、150%、200% DPI 下沒有裁切。
- Windows、macOS、Linux 的字型替代不會造成控制項高度異常。
- 鍵盤 Tab 順序、Enter/Shift+Enter、Tooltip 與螢幕閱讀器名稱正確。

---

## 8. 驗收標準

1. 從 `1200 px` 連續縮到 `500 px` 時，不產生水平捲動條。
2. 所有寬度下，輸入框及送出按鈕皆完整可見。
3. Compact 與 Narrow 模式仍能辨識目前模型。
4. 小精靈模式按鈕位置固定，僅切換顯示文字。
5. 所有純 Icon 按鈕都有 Tooltip 與可存取名稱。
6. Code Block 不破壞縮排，且能水平捲動。
7. Resize 過程不頻繁重建整個對話列表，不造成明顯閃爍。
8. 邊界值 `619/620` 與 `999/1000 px` 不出現反覆切換或版面跳動。

---

## 9. 建議的第一階段實作順序

1. 將主畫面改為 `Auto / * / Auto`，先解決輸入框裁切。
2. 建立 `WindowSizeClass` 與三個斷點。
3. 將標題列、工具列、對話區、Composer 拆成獨立元件。
4. 先完成 Compact 模式，因為這是最常見的半寬視窗。
5. 再完成 Narrow 模式及最低寬度限制。
6. 將所有 Unicode 箭頭替換為 `PathIcon`。
7. 補齊 Tooltip、AutomationProperties 與鍵盤操作。
8. 最後處理小精靈 Window 的動畫與跨平台差異。
 