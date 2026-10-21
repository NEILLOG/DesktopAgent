# 08 素材型角色

- 狀態：進行中（Q1～Q5 於 2026-10-22 照建議確認；**8A～8D 程式與文件完成**，待 D4 手動驗收）
- 對應里程碑：M8
- 前置：M7 程式完成（殼、`ISpriteFactory`／`ISpriteView`、設定 ›「小精靈」都已就緒）。素材包（使用者用 Claude Code 做的像素柯基）是第一個驗證對象；原本 `plan/corgi-pixel.zip` 有兩處特效被切掉，已於 2026-10-22 修好（`plan/corgi-pixel_1.zip`，見「現況」）

## 目標

不用寫程式：把圖片加一份 `manifest.json` 放進「角色」資料夾，重新載入就能在設定 ›「小精靈」選用，殼完全不用改。
素材型角色不執行任何程式碼；載入的檔案一律當作不可信的輸入處理（路徑、大小、尺寸、語氣文字都要驗證）。

## 範圍

做：
- manifest 的載入與驗證：欄位、`SpriteLimits` 自動縮放並警告、動畫引用、檔案與圖片大小上限、語氣的檢查（M7 留下來的）
- sprite sheet 播放器：狀態動畫、`loop`／`next`／`hold`、戳（`poke.press`／`poke.release`）與拖曳（`drag`）的動畫、依動畫的 fps 回報 `PreferredFps`
- 繪製：切格子畫進角色框；`rendering: pixelated`（最近鄰）與整數倍貼齊（`pixelGrid`）；`hitShape: alpha`（依目前這一格的透明度判斷命中）
- 「角色」資料夾自動載入、重新載入、載入失敗的原因顯示
- 設定頁的角色清單加上預覽圖、程式型／素材型標示、「重新載入」「開啟角色資料夾」、載入失敗清單
- `Lumi.Sprites.Abstractions` 升到 1.1（新增選填欄位，舊角色不受影響）；`docs/sprite-dev-guide.md` 第 7 節改寫
- 用柯基素材做端對端驗證，並放進 `samples/sprites/` 當範例與測試資料

不做：
- Lottie（Avalonia 目前只有實驗性套件，之後再評估）
- PNG 連續圖（每格一個檔案）：第一版只支援 sprite sheet（Q4）
- 程式碼型角色外掛（第三方 dll）
- 角色隨程式內建附帶：M11 打包時再決定（Q2）
- 眼睛跟著游標之類的「看向」表現：frames 型角色收到 `Look`／`Near` 就忽略

## 現況（開工前盤點）

- `docs/sprite-dev-guide.md` 第 7 節已經定了 manifest 的大部分欄位（`schemaVersion`、`id`、`box`、`effectMargin`、`hitShape`、`bubbleAnchor`、`voice`、`animations.*`、`fallback`），**但還沒有程式**：`SpriteManifest` 是程式型角色用的，沒有動畫資訊。
- M7 已經有可以重用的部分：`SpriteLimits.Fit`（縮放與警告）、`SpriteStateResolver`（退化）、`SpriteSettings`（角色 id、尺寸）、`ISpriteCatalog`（目前是啟動時固定的清單）、設定頁的預覽區與狀態預覽（用同一個 `ISpriteFactory`，素材型角色上線後自動可用）、`SpriteVoiceProvider`（讀 manifest 的 `voice`）。
- M7 把「語氣超過 200 字或像指令的檢查」留到這裡（那時才有外部來源的 manifest）。
- 殼對角色的要求（`ISpriteView`）：`Visual` 大小是 `Outer`、座標原點是 `Visual` 左上角、`HitTest` 不超出角色框、`Advance(dt)` 由殼統一驅動、`PreferredFps`。素材型角色照同一份契約，不需要殼知道它是素材。
- **柯基素材包的盤點**（`plan/corgi-pixel.zip`，我已經讀過、沒有執行任何東西）：
  - 13 個動畫（10 個狀態＋`poke.press`／`poke.release`／`drag`），格子一律 280×280 px、透明背景、無半透明像素、每個動畫 7～10 色；每個圖的大小剛好是「列數×欄數×280」，張數與 `frames`／`columns` 一致。
  - **一個邏輯像素是 10×10 px（整張格子是 28×28 個邏輯像素）**，所有邊界都落在 10 的倍數上。框 112 DIP 在中檔每個邏輯像素是 4 DIP，所以 1 倍、2 倍螢幕上都是整數；小檔（89.6）、大檔（145.6）不是。
  - 角色腳底都貼在格子底邊（y=279），水平置中，播放不會抖。`drag` 腳底上移 10px（被提起）。
  - 已有的 manifest 欄位全部合規（框 112×112、長寬比 1:1、外擴 0、最長邊 112 ≤ 128、`voice` 在 200 字內）；另外用了規格還沒有的 `"rendering": "pixelated"`。
  - **（已修好，2026-10-22）兩處素材原本要修**：`thinking.png` 第 5～10 格的特效方塊貼到右邊緣；`sleep.png` 第 3、4、7、8 格的「Z」貼到上緣。殼只會畫格子內的東西，貼邊的部分會被切掉。`id` 也已由 `lumi.corgi-pixel` 改成 `neillog.corgi-pixel`（`lumi.` 前綴保留給內建角色）。新包重新量過：所有動畫所有格子離左、上、右邊緣最少 10px、邊界都在 10px 網格上、沒有半透明像素、全包只用 11 個顏色。
  - 這個專案要求：除了腳底（設計上貼底邊），所有內容離格子左、上、右邊緣至少 1 個邏輯像素（10px）。載入時檢查貼邊只能警告（作者可能故意），但驗收時柯基要全部通過。

## M8 拆成四個部分

| 部分 | 名稱 | 產出 | 依賴 |
|---|---|---|---|
| **8A** | 規格擴充與載入驗證 | Abstractions 1.1（`Rendering`、`PixelGrid`、`HitShape.Alpha`、`SpriteVoiceValidator`）；新專案 `Lumi.Sprites.Assets` 的 manifest 解析、驗證、資料夾掃描、載入失敗報告；`samples/sprites/corgi-pixel` | M7 |
| **8B** | 播放器與繪製 | `SpriteAnimator`（純邏輯：狀態動畫、`next`／`hold`、戳與拖曳的覆蓋）、`SpriteSheetControl`、`AssetSpriteFactory`／`AssetSpriteView`、整數倍貼齊、alpha 命中 | 8A |
| **8C** | 目錄與設定頁整合 | 可重新載入的 `ISpriteCatalog`、DI 與資料夾位置、設定頁的預覽圖／種類／重新載入／開啟資料夾／失敗清單、角色消失時殼的退回 | 8B |
| 8D | 文件與柯基驗收 | `docs/sprite-dev-guide.md` 第 7 節改寫、`agent.md`、`ui-spec`；用真的柯基包做端對端測試與手動驗收 | 8C |

每段結束都要能編譯、測試全過；8A 與 8B 之間只靠 manifest 的資料型別，可以交錯做。

---

## 開工前要確認的事

以下是影響結構或行為的決定，我各自給了建議；**沒有意見的話就照建議做**。

| # | 問題 | 建議 | 影響 |
|---|---|---|---|
| Q1 | 角色資料夾放哪裡、怎麼安裝？ | **使用者資料夾下的 `sprites/`**（Windows `%APPDATA%\Lumi\sprites`、macOS `~/Library/Application Support/Lumi/sprites`，和 `settings.json` 同一層）。一個角色一個子資料夾（裡面是 `manifest.json` 與圖片）。安裝＝把資料夾複製進去，然後在設定 ›「小精靈」按「重新載入」。**啟動時掃描一次，不做檔案監看**（監看在不同平台行為不一，也沒有必要）。設定頁提供「開啟角色資料夾」鈕（資料夾不存在就先建立）。 | 8C |
| Q2 | 柯基要不要隨程式附帶？ | **M8 不內建**：放在 repo 的 `samples/sprites/corgi-pixel/`，當作範例、當作測試資料、也當作做新角色的起點；使用者要用就複製到角色資料夾。內建角色（隨安裝程式附帶）留到 M11 打包時再決定，到時只需要把這個資料夾加進打包內容與預設掃描路徑。 | 8D、M11 |
| Q3 | 語氣超過 200 字、或像指令時，要拒載整個角色還是只丟掉語氣？ | **只忽略語氣，角色照常載入**，並在設定頁的角色資訊與載入警告裡說明「語氣被忽略：原因」。理由：語氣是 prompt 最低優先的點綴，不是角色能不能用的條件；整個拒載會讓作者為了一句話看不到美術。檢查規則見設計（它是安全網，不是保證——語氣層本來就在最低優先，而且會用固定的前綴包住）。 | 8A |
| Q4 | 「PNG 連續圖」（每格一個檔案）要不要第一版就支援？ | **第一版只做 sprite sheet**：規格（`docs/sprite-dev-guide.md`）和柯基都是 sheet；連續圖可以用任何工具轉成 sheet，而且每格一個檔案會讓檔案數量、載入錯誤與記憶體管理都複雜很多。manifest 的 `kind` 欄位保留（目前只有 `"frames"`），之後要加連續圖是新增選填欄位，不用升主版本。 | 8A、8B |
| Q5 | 像素角色縮放：只用最近鄰，還是再加「整數倍貼齊」？ | **加整數倍貼齊（選填 `pixelGrid`）**。像素圖在非整數倍時最近鄰縮放會讓有的邏輯像素 5px、有的 6px，看起來粗細不均。做法：作者在 manifest 宣告 `pixelGrid`（柯基是 28：整張格子的邏輯像素數），殼把每個邏輯像素畫成整數個**裝置像素**（向下取整），整張圖**底部置中**貼齊角色框（腳底基準線不變）。中檔在 1 倍、2 倍螢幕剛好是整數，小檔、大檔會縮到 84／140 DIP 左右（比要求的略小，不會超出框）。沒有宣告 `pixelGrid` 的角色維持一般縮放（`rendering: pixelated` 時最近鄰、否則平滑）。這個效果要實機看過才知道值不值得，所以做成選填，預設不影響其他角色。 | 8B |

其他我直接定的事（有意見再指出）：

- **每一格涵蓋的是 `Outer`（角色框加上特效外擴），不是角色框**：這是對 `docs/sprite-dev-guide.md` 第 7 節的釐清。原本的範例（框 100、外擴 0.3、格子 260px）沒有說清楚；光暈與特效要畫在框外那一圈，格子就必須包含它。`effectMargin` 是 0（像柯基）時兩者相同，所以柯基不受影響。格子的長寬比要和 `Outer` 一致（差超過 2% 警告），畫的時候等比例縮放、底部置中。
- **所有動畫的格子大小必須相同**（以第一個動畫為準，不同就警告並拒載那個動畫）：換動畫時畫面不會跳大小。
- **新增的欄位都是選填、都有預設**：`rendering`（`smooth`｜`pixelated`，預設 `smooth`）、`pixelGrid`（整數，預設沒有）、`hitShape` 多一個 `alpha`。舊的 manifest 不用改；`schemaVersion` 維持 1，Abstractions 升 1.1（新增成員、沒改既有的意義）。
- **命中範圍 `alpha`**：目前這一格在該點（容許周圍 1 個邏輯像素內）有不透明像素就算命中，而且仍然限制在角色框內。柯基是瘦高的輪廓、腳在左右下角，橢圓會漏掉腳，矩形又包含大片透明區、會擋住下方桌面，所以柯基用 `alpha`。命中判斷用的是從圖片解出來的 alpha 遮罩（每個動畫第一次用到時才解，之後快取）。
- **`id` 規則**：小寫英數、`.`、`_`、`-`，3～64 字元；`lumi.` 前綴保留給內建角色（素材型角色用了就拒載）；和已經註冊的 id 重複時，先到的留下，後面的拒載並說明。
- **載入失敗的角色不進清單**，但原因（哪個資料夾、哪個欄位、為什麼）會列在設定頁與 log；警告（會自動修正的，例如縮放、貼邊、語氣被忽略）角色照常載入。
- **解圖的記憶體**：每個動畫第一次用到時才解碼並快取，不預先全部解。柯基 12 張圖解開約 30 MB；每個角色的解碼後總量上限 96 MB，超過就拒載（避免巨大的圖拖垮程式）。
- **設定頁的預覽與清單用同一個 `ISpriteFactory`**，所以素材型角色不需要設定頁另外的程式；清單的縮圖用 `preview.png`（選填，沒有就只顯示名稱）。
- **選用的角色被刪掉或載入失敗**：殼退回第一個可用的角色（`SpriteSettings.SpriteId` 保留不動，角色回來就自動換回去）。M7 已經有這個退回邏輯，8C 補上「清單重新載入後」的觸發。

---

## 設計

### 專案與相依

```
Lumi.App ──► Lumi.Sprites.Assets ──► Lumi.Sprites.Abstractions
                   └──► Avalonia（Bitmap、DrawingContext）
```

- 新專案 `Lumi.Sprites.Assets`：manifest 解析與驗證、資料夾掃描、動畫播放邏輯、`AssetSpriteFactory`／`AssetSpriteView`。**只引用 Abstractions 與 Avalonia**，不引用 Core、Providers、Infrastructure、Platform、Tools。
- 資料夾位置由 App 傳進來（`AssetSpriteOptions.Directory`，值來自 `LumiPaths.DataDirectory`），這個專案不認識 `Lumi.Infrastructure`。
- 架構測試：把 `Lumi.Sprites.Assets` 加進 `SpriteRuleTests` 的清單（不引用其他 Lumi 專案、原始碼不出現 `DispatcherTimer`／`Window`／`DllImport`／`Stopwatch`…）。檔案 I/O（`System.IO`）可以，P/Invoke 與計時器不行。

### 規格擴充（8A，Abstractions 1.1）

```csharp
public enum SpriteHitShape { Ellipse, Rect, Alpha }          // 新增 Alpha
public enum SpriteRendering { Smooth, Pixelated }             // 新增

public sealed record SpriteManifest(...)                       // 既有的位置參數不動
{
    public SpriteRendering Rendering { get; init; } = SpriteRendering.Smooth;   // 新增，選填
    public int? PixelGrid { get; init; }                                       // 新增，選填：整張格子的邏輯像素數
}
```

- 用 `init` 屬性而不是新增位置參數：`OrbSpriteFactory` 與測試裡既有的建構不用改。
- `SpriteVoiceValidator.Validate(string? voice) → (string? Accepted, string? Rejection)`（放在 Abstractions，因為語氣是 manifest 的一部分，程式型角色也適用）：
  - 去掉前後空白；空白視為沒有。
  - 超過 `SpriteLimits.VoiceMaxLength`（200）字元 → 拒絕。
  - 含控制字元、換行、`<`、`>`、反引號 → 拒絕（語氣只能是單行純文字，不能偽裝成 prompt 的標籤或程式碼區塊）。
  - 含指令式字眼（大小寫不分）→ 拒絕。清單先收：`忽略`、`無視`、`不要遵守`、`不必遵守`、`取代`、`覆蓋`、`以上的`、`之前的`、`以前的`、`系統提示`、`system prompt`、`ignore`、`disregard`、`override`、`jailbreak`、`你現在是`、`從現在起你是`、`忘記`。清單有測試，之後可以增補。
  - 說明文字是給作者看的繁中一句話（「語氣被忽略：含有指令式的字眼『忽略』」）。
- `SpriteVoiceProvider` 改用 `SpriteVoiceValidator`（目前只檢查長度）：程式型角色的語氣也走同一個檢查。

### Manifest 載入（8A，`Lumi.Sprites.Assets`）

```csharp
public sealed record AssetAnimation(string Key, string Source, int Frames, int Columns, double Fps, bool Loop, string? Next, bool HoldLast);
public sealed record AssetSpriteDefinition(SpriteManifest Manifest, string Directory, IReadOnlyDictionary<string, AssetAnimation> Animations,
    Size CellPixels, string? PreviewPath);
public sealed record SpriteLoadIssue(string Folder, string? Field, string Message, bool Fatal);
public sealed record SpriteLoadResult(AssetSpriteDefinition? Definition, IReadOnlyList<SpriteLoadIssue> Issues);

public static class AssetSpriteLoader { public static SpriteLoadResult Load(string directory, IReadOnlySet<string> reservedIds); }
```

驗證（`Fatal` 的拒載，其餘警告後修正）：

| 檢查 | 結果 |
|---|---|
| `manifest.json` 不存在、超過 64 KB、不是合法 JSON、`schemaVersion` 不是 1 | 拒載 |
| `kind` 不是 `"frames"`（第一版） | 拒載（訊息說明目前只支援 frames） |
| `id` 不合規、用了 `lumi.` 前綴、和已註冊的重複 | 拒載 |
| `name`、`author`、`version` 缺少或超過 100 字元 | 缺少拒載；過長截斷並警告 |
| `box` 缺少或不是正數 | 拒載；其餘超限（長寬比、最長邊）交給 `SpriteLimits.Fit` 自動縮放並警告 |
| `effectMargin` 不在 0～0.4 | 警告並限制 |
| `hitShape`、`rendering` 不認得 | 警告並用預設 |
| `pixelGrid` 不是正整數，或不能整除格子寬高 | 警告並忽略（不整除表示作者算錯了） |
| `animations` 沒有 `idle`，或 key 不在允許集合（十個狀態、`poke.press`、`poke.release`、`drag`） | 沒有 `idle` 拒載；不認得的 key 警告並忽略 |
| 動畫的 `source`：絕對路徑、含 `..`、不是 `.png`、在資料夾外（解析後）、是符號連結或捷徑 | 拒載那個動畫；是 `idle` 就整個角色拒載 |
| 檔案不存在、超過 8 MB、PNG 標頭不合法、寬高超過 4096、解碼後總量超過 96 MB | 同上 |
| `frames` 不在 1～120、`columns` 不在 1～`frames`、`fps` 不在 1～60 | 同上 |
| 圖片大小不是「列數×欄數×格子」的整數倍 | 同上（格子大小＝圖寬 ÷ `columns`） |
| 格子和第一個動畫的大小不同 | 拒載那個動畫並說明 |
| 格子長寬比和 `Outer` 差超過 2% | 警告（會等比例縮放、底部置中） |
| `next` 指向不存在的動畫；`next` 與 `loop: true` 同時出現；`hold` 不是 `"last"` | 警告並忽略該欄位（`loop: true` 的動畫不會結束，`next` 沒有意義） |
| `fallback` 的 key 或值不是狀態名稱 | 警告並忽略該項 |
| 動畫內容貼著格子的左、上、右邊緣（底邊不算） | 警告（只在解碼時檢查，8B 解到才報；驗收時柯基要零警告） |
| `voice` 沒通過 `SpriteVoiceValidator` | 警告，語氣被忽略（Q3） |

- PNG 的標頭（寬高）在**解碼之前**讀 IHDR 檢查，不靠 Avalonia 解完才知道（避免壓縮炸彈）。
- 掃描資料夾：只看第一層的子資料夾；一個子資料夾出錯不影響其他的；以 id 排序，結果穩定。
- `Supported`（殼用來解析退化的集合）＝ manifest 裡有定義動畫的狀態；`Fallback`＝manifest 的 `fallback`。

### 播放（8B）

**`SpriteAnimator`（純邏輯，不碰 Avalonia）**：

- 兩層：**底層**是目前狀態的動畫；**覆蓋層**是互動動畫（戳、拖曳），有覆蓋時它優先。
- 吃 `SetState(state)`、`OnInteraction`、`Advance(TimeSpan)`，吐「現在要畫哪個動畫的第幾格」。
- 規則（每條都有測試）：

| 事件 | 結果 |
|---|---|
| `SetState(s)` | 從該狀態動畫的第 0 格重新開始（同一個狀態重複設定不重來） |
| 動畫播到最後一格 | `loop: true` 從頭；`loop: false` 有 `next` 就接那個動畫（接著循環它，不會再回原動畫），`hold: "last"` 停在最後一格，都沒有就停在最後一格 |
| `PokeBegin` | 覆蓋層換成 `poke.press`（播完停在最後一格，等放開）；角色沒有這個動畫就忽略 |
| `PokeEnd` | 覆蓋層換成 `poke.release`，播完（接 `next`，通常是 `idle`）覆蓋層清掉、回到底層；沒有 `poke.release` 就直接清掉；`PokeEnd` 前面沒有 `PokeBegin` 忽略 |
| `DragBegin` | 覆蓋層換成 `drag`（循環）；`DragEnd` 清掉覆蓋層回到底層 |
| 戳住時開始拖 | 殼不會讓這發生（追蹤器保證）；萬一收到，`drag` 蓋過 `poke.*` |
| `Near`、`Look` | 忽略 |
| `Advance(dt)` | 累計時間換算成格數；單次 dt 上限 0.25 秒（電腦睡眠醒來後的第一幀不會一次跳很多格） |

- `PreferredFps`＝目前播放中動畫的 fps（互動時是覆蓋層動畫的）；停在最後一格、不會再變的畫面回 1（`SetState` 與互動事件會立即更新畫面，不必等下一次 `Advance`）。所以柯基待機只要 8 fps、休眠 4 fps，比光球的 30／12 fps 省電。

**繪製（`SpriteSheetControl`）**：

- `Visual` 大小是 `Outer`。`Render` 用 `DrawImage(bitmap, 來源格子, 目的矩形)`。
- 一般縮放：格子等比例縮放，放進 `Outer`，底部置中；`rendering: pixelated` 時 `RenderOptions.BitmapInterpolationMode = None`，否則高品質。
- 整數倍貼齊（有 `pixelGrid` 時）：先算裝置像素大小 `Outer × RenderScaling`，每個邏輯像素的裝置像素數 `s = floor(裝置寬 ÷ pixelGrid)`（至少 1），目的矩形＝`pixelGrid × s` 個裝置像素（換回 DIP），底部置中。`RenderScaling` 在 `Render` 時從 `VisualRoot` 取（還沒掛上視窗時用 1）。
- 圖片用 Avalonia 的 `Bitmap` 解碼；每個動畫第一次用到時載入並快取。`AssetSpriteFactory` 擁有快取，同一個角色的多個 view（殼與設定頁預覽）共用。角色被取代或清單重新載入時，舊的快取交給 GC（`Bitmap` 的 `Dispose` 在工廠被丟掉時做）。

**命中與氣泡**：

- `HitTest(p)`：先確認在角色框內；`Ellipse`／`Rect` 同程式型角色；`Alpha` 把點換算到目前這一格的來源座標，查 alpha 遮罩（容許周圍 1 個邏輯像素；沒有 `pixelGrid` 時用格子寬的 1/28 當容許範圍）。
- `BubbleAnchor`＝manifest 的比例座標（相對角色框），換算成 `Visual` 座標，和光球同樣的做法。

### 目錄與整合（8C）

```csharp
public interface IReloadableSpriteCatalog : ISpriteCatalog
{
    event EventHandler? Changed;
    IReadOnlyList<SpriteLoadIssue> Issues { get; }
    Task ReloadAsync(CancellationToken ct = default);
}
```

- `SpriteLibrary : IReloadableSpriteCatalog`（在 `Lumi.Sprites.Assets`）：內建（程式型）工廠＋資料夾裡載入的素材型工廠；`ReloadAsync` 在背景執行緒掃描、解析，完成後在呼叫端的同步內容上換掉清單並發 `Changed`。`Find`／`All` 永遠回傳目前的快照（不可變清單，換的時候整份換）。
- 啟動時載入一次（不阻塞視窗顯示：先用內建角色，載入完成後通知）。
- App 註冊：`AddLumiAssetSprites(o => o.Directory = Path.Combine(LumiPaths.DataDirectory, "sprites"))`，取代 `SpriteCatalog` 成為 `ISpriteCatalog`（`AddSprite<T>()` 的 `TryAddSingleton` 讓它能被取代）。
- `SpriteWindow`：聽 `Changed`，已經有的「有變才重建」邏輯補一個條件（目前選用的角色不在清單裡就退回第一個，在回來時再換回去）；不用改殼的其他部分。
- 設定頁（`SpriteSettingsViewModel`）：`Sprites` 清單聽 `Changed` 重建；每項多 `Kind`（程式型／素材型）、`PreviewImage`（`preview.png`，≤ 1 MB、≤ 512×512，沒有或壞掉就沒有縮圖）；新增「重新載入」「開啟角色資料夾」兩個命令（開資料夾由 `IFolderOpener` 做，測試用假的）；底下的「載入失敗與警告」清單列出 `Issues`（資料夾、欄位、原因）。
- `SpriteVoiceProvider` 已經讀 `Manifest.Voice`；載入時語氣沒通過檢查就不會進 manifest，所以不用再改。

---

## 工作項目

### 8A 規格擴充與載入驗證

- [x] A1 Abstractions 1.1：`SpriteRendering`、`SpriteManifest.Rendering`／`PixelGrid`、`SpriteHitShape.Alpha`；版本常數與開發指南版本紀錄；既有測試不改就過
- [x] A2 `SpriteVoiceValidator` ＋ 測試（長度邊界、控制字元、換行、`<>`、反引號、指令字眼大小寫、空白、正常語氣放行）；`SpriteVoiceProvider` 改用它
- [x] A3 新專案 `Lumi.Sprites.Assets`：`AssetAnimation`、`AssetSpriteDefinition`、`SpriteLoadIssue`／`SpriteLoadResult`、manifest JSON 解析（容忍未知欄位、有大小上限）
- [x] A4 驗證表的每一列各一個測試（用測試裡動態產生的最小 PNG，不依賴使用者的美術）；PNG 標頭讀取與各種上限；路徑跳脫（`..`、絕對路徑、符號連結）要有測試
- [x] A5 資料夾掃描：一個壞的不影響其他、id 重複與保留前綴、結果穩定排序、資料夾不存在視為空
- [x] A6 `samples/sprites/corgi-pixel/`（使用者修好的版本）；測試：真的柯基包載入後零警告（素材已修好，所以不需要標已知問題）
- [x] A7 架構測試：`Lumi.Sprites.Assets` 加進角色專案的規則

### 8B 播放器與繪製

- [x] B1 `SpriteAnimator` ＋ 測試（上面規則表的每一列；`next` 鏈；`hold`；覆蓋層；時間步進上限；`PreferredFps` 隨動畫變、靜止回 1）
- [x] B2 解碼與快取：延遲解碼、alpha 遮罩（命中用）、邊緣貼邊檢查（解到才報警告）、解碼後總量上限
- [x] B3 `SpriteSheetControl`：切格繪製、`pixelated`、整數倍貼齊（底部置中）、`RenderScaling`；用 Skia 的 headless 測試畫出來檢查像素（格子對、貼齊的邏輯像素寬一致、腳底在框底）
- [x] B4 `AssetSpriteFactory`／`AssetSpriteView`：`ISpriteView` 契約（`Visual` 大小、座標、`HitTest` 三種形狀、`BubbleAnchor`、`PreferredFps`）；用 `SpriteLimits.Fit` 的結果建
- [x] B5 契約測試：和光球同一套（每個狀態都能設定、畫出來非空白、互動序列不丟例外、`HitTest` 不超出角色框）

### 8C 目錄與設定頁整合

- [x] C1 `SpriteLibrary`／`IReloadableSpriteCatalog`：背景載入、整份換清單、`Changed`、`Issues`；測試（載入、重新載入、增減角色、並發 `Find`）
- [x] C2 DI：`AddLumiAssetSprites`、資料夾位置、啟動載入；組裝測試
- [x] C3 `SpriteWindow` 聽清單變更：選用的角色消失就退回、回來就換回去，位置不跳動；測試
- [x] C4 設定頁：縮圖、種類、重新載入、開啟資料夾、失敗與警告清單；ViewModel 測試與 headless 渲染測試
- [x] C5 設定頁預覽與狀態預覽對素材型角色的測試（退化標示、`poke.*`／`drag` 不在十個狀態內，所以不列入狀態清單）

### 8D 文件與柯基驗收

- [x] D1 `docs/sprite-dev-guide.md` 第 7 節改寫：安裝位置、`rendering`、`pixelGrid`、`hitShape: alpha`、格子涵蓋 `Outer`、互動動畫的語意（`poke.press`／`poke.release`／`drag`）、各項上限與驗證訊息、語氣規則；第 9 節加「素材型角色怎麼驗證」；版本紀錄 1.1
- [x] D2 `agent.md`、`docs/ui-spec.md` 第 7 節（角色清單的新內容）、主計畫狀態
- [x] D3 端對端測試：把真的柯基包複製到暫存的角色資料夾，經 `SpriteLibrary` 載入、在 `SpriteWindow` 裡播放十個狀態與互動
- [ ] D4 手動驗收（見驗收標準）

---

## 驗收標準

自動（`dotnet test` 全過）：

- 既有所有測試不改斷言就過；光球與其他程式型角色不受 Abstractions 1.1 影響。
- 載入驗證表的每一列都有測試；壞的 manifest 不會讓程式當掉，也不影響其他角色。
- 路徑跳脫、符號連結、巨大的圖（宣告的寬高超過上限）、超過 `idle` 以外動畫的上限，都被擋下並說明原因。
- `SpriteVoiceValidator` 與 `SpriteVoiceProvider`：程式型與素材型角色都經過同一個檢查。
- `SpriteAnimator` 的規則表每一列都有測試。
- 繪製：像素角色整數倍貼齊時邏輯像素寬一致、腳底基準線在角色框底；`rendering: pixelated` 沒有平滑出來的中間色。
- 三種 `hitShape` 都不超出角色框；`alpha` 在透明處回 false、在不透明處與其周圍 1 個邏輯像素回 true。
- `ISpriteView` 契約測試（和光球同一套）全過。
- 真的柯基包：載入零警告、十個狀態與三個互動動畫都畫得出來。
- 架構測試：`Lumi.Sprites.Assets` 只引用 Abstractions；原始碼沒有計時器、視窗、P/Invoke。

手動（Mac 我能陪著驗；Windows 需要你測）：

1. 把柯基資料夾複製進角色資料夾，設定 ›「小精靈」按「重新載入」→ 清單出現「柯基（像素）」與縮圖，載入警告為空。
2. 選用柯基：小精靈立即換成柯基，位置不跳動；點十個狀態逐一預覽，動畫順暢、像素沒有糊；`thinking`、`sleep` 的特效沒有被切掉。
3. 在小精靈上：靠近、長按戳（按住時 `poke.press`、放開後 `poke.release`）、拖曳（`drag`）、回應中各狀態、完成後回待機、60 秒休眠都正確；點腳（左右下角）也能命中、點旁邊的透明處會穿透。
4. 三個尺寸（64／80／104）各看一次，像素粗細一致（中檔最準，小、大檔略小）；1 倍與 2 倍螢幕各看一次（Windows 的 125%、150% 也看）。
5. 語氣：聊幾句，確認回答有「汪」；到設定關掉語氣再聊，沒有了。
6. 壞掉的角色：故意做一個缺 `idle`、一個 `id` 用了 `lumi.`、一個 `source` 指向 `../`，確認設定頁的失敗清單說得清楚、其他角色不受影響。
7. 耗能：用 HUD 量柯基待機與休眠的 CPU（預期明顯低於光球）。
8. 重開程式：柯基仍是選用的角色；把資料夾移走再重開，小精靈退回光球、設定裡的選擇不丟；把資料夾放回去重新載入，自動換回柯基。

## 風險

- **像素粗細不均**（Q5）：整數倍貼齊靠 `RenderScaling` 在繪製時才知道，跨螢幕（不同縮放）拖動時要重算；視窗移到另一個螢幕、縮放變了之後要重新繪製。8B 的 headless 測試只能驗證計算，真正的觀感要實機看。
- **記憶體**：柯基解開約 30 MB（12 張 sheet 都用過之後）。如果之後的角色更大，96 MB 上限會擋掉；到時再考慮只留最近用到的動畫、或解碼成較小的格式。8D 的手動驗收順便量實際的記憶體數字。
- **作者的素材品質**：規格能擋的是格式與上限，擋不了美術（例如貼邊被切掉、各格對位不一致）。載入時只能警告；所以開發指南第 9 節的檢查清單與 8D 的柯基驗收是主要的防線。
- **語氣檢查是安全網，不是保證**：指令字眼清單擋不住所有寫法。真正的防線是語氣層本來就在 prompt 最低優先、單行純文字、不能含標籤字元，而且使用者可以在設定關閉。
- **載入在背景執行緒、清單在 UI 執行緒換**：整份換掉（不可變清單）可以避免讀到一半的狀態；`SpriteWindow` 與設定頁都只在 UI 執行緒上讀。

## 實作紀錄

### 8A 規格擴充與載入驗證（2026-10-22）

結果：全部 1,765 個測試通過（Sprites 293〔8A 新增 177〕、App 556、其餘不變）。新專案 `Lumi.Sprites.Assets`（只引用 Abstractions）、Abstractions 1.1、`samples/sprites/corgi-pixel`（柯基載入零警告）。偏離或補充原設計的地方：

- **`AssetAnimation` 多帶了幾個衍生資料**：`FullPath`（已驗證在角色資料夾內）、`SheetPixels`、`Rows`、`CellPixels`；`AssetSpriteDefinition` 帶 `FolderName` 與 `Warnings`（載入成功的角色的警告跟著角色走，設定頁選到那個角色時看得到）。
- **非 `idle` 動畫壞掉時只丟那個動畫**（記一項非 Fatal 的 issue，對應的狀態走退化）；`idle` 壞掉、沒有 `idle`、或解碼總量超過上限才讓整個角色失敗。`SpriteLoadIssue.Fatal` 因此是「這一項讓角色失敗」，不是「這一項是錯誤」。
- **`loop` 沒寫時預設 `false`**（規格範例每個動畫都有寫，沒寫就當作不循環）；**`kind` 沒寫時視為 `frames`**。
- **掃描器負責「素材型角色彼此 id 重複」**：只把內建角色的 id 當作保留傳給載入器，重複由掃描器依資料夾名稱排序先到先得，訊息會說出是哪個資料夾用了（載入器自己只知道「已被使用」）。
- **語氣檢查放在 Abstractions 的 `SpriteVoiceValidator`**，而且 `SpriteVoiceProvider` 也再檢查一次（程式型角色不經過載入器，同樣要擋）：載入器擋掉的語氣不會進 manifest，所以正常流程不會觸發第二次，但它是第二道防線。指令字眼清單目前 18 個，每個都有測試。
- **柯基範例的 manifest 在 repo 裡比使用者的原始包多兩個欄位**：`"hitShape": "alpha"`（原本是 `ellipse`）與 `"pixelGrid": 28`；其餘逐項相同（用 diff 對過）。使用者的原始 zip 不動。`samples/sprites/README.md` 說明用途與安裝位置。
- **測試基礎設施**：`TestPng`（自己寫的 PNG 編碼，不依賴影像程式庫，可以只產生標頭來測「宣告的大小超過上限」）與 `SpritePack`（暫存資料夾裡的合法最小角色，測試改欄位做出各種壞法）。建立符號連結失敗的環境（例如沒有權限的 Windows）會略過那幾個測試。
- **尚未做、留給 8B**：解碼後內容貼邊的警告（要解碼才知道）、alpha 遮罩、實際繪製。載入階段只讀 PNG 標頭，不解碼。

### 8B 播放器與繪製（2026-10-22）

結果：Sprites 測試 406 個全過（8B 新增 113 個，含真的把柯基用 Skia 畫出來檢查像素）。偏離或補充原設計的地方：

- **沒有 `next` 也沒有 `hold` 的非循環動畫，底層播完回 `idle`**（`idle` 本身停在最後一格）；計畫的規則表只寫了「都沒有就停在最後一格」，實作時改成這樣——否則 `done` 之類的動畫忘了寫 `next` 會永遠卡在最後一格。覆蓋層另外處理：`poke.release` 播完清掉覆蓋層、回到底層**目前**的狀態（不用 release 的 `next`，因為戳的時候狀態可能是思考中，回 idle 會錯）；`poke.press` 一律停在最後一格等放開。覆蓋期間底層時間凍結，結束後從原來的位置繼續。
- **`pixelGrid` 的定義釐清**：是格子**寬度方向**的邏輯像素數，邏輯像素必須是正方形，所以高度方向的個數由長寬比決定、必須剛好是整數。載入器的檢查改成「寬度整除、`高×grid` 整除寬」（正方形格子與原本相同）；已寫進 `SpriteManifest.PixelGrid` 的註解。
- **解碼用 `Avalonia.Media.Imaging.Bitmap`，alpha 遮罩從 `CopyPixels` 取出**（RGBA、BGRA 的 alpha 都在第 4 個位元組）；一張圖解碼一次、繪製與命中共用。貼邊檢查在解碼時做：離格子左、上、右邊緣不到 1 個邏輯像素（沒有 `pixelGrid` 時以格子寬 ÷ 28 為準）就警告並列出是第幾格，底邊不算；警告進 `AssetSpriteFactory.Issues`（和載入時的警告合併，設定頁用）。
- **`AssetSpriteFactory` 公開、其餘內部**：`Lumi.Sprites.Assets` 對外只有載入器、掃描器、型別、`SpriteAnimator`、`SpriteSheetLayout` 與工廠；`SpriteSheetStore`／`SpriteSheetControl`／view 是內部實作（測試用 `InternalsVisibleTo`）。
- **`PreferredFps` 靜止回 1 之後，殼的輪詢每 33 ms 會重新讀它**，所以狀態或互動一來幀率立即回升，不會等一秒。
- **測試用的小開關**：`SpriteSheetControl.RenderScalingOverride`（internal）。headless 的視窗固定是 1 倍，而繪製用的縮放比正式是從 `VisualRoot` 取，所以 2 倍螢幕的測試靠它指定。
- **柯基實測**（Skia，每個像素都量）：中檔 1 倍、2 倍剛好每個邏輯像素 4／8 裝置像素；小檔 3／6、大檔 5／10（整張圖分別是 84、140 DIP，比要求的 89.6、145.6 略小）；所有橫列上有顏色的連續段落長度都是該倍數的整數倍，沒有 5px、6px 交錯。命中：胸前、頭、左右腳（橢圓會漏掉）都命中，左上角與右下角的透明處、框外都不命中。待機 8 fps、休眠 4 fps、`error` 停住後 1 fps。
- **尚未做、留給 8C／8D**：目錄與重新載入、設定頁、殼的整合；跨螢幕（不同縮放）拖動時的重繪靠 Avalonia 的重新渲染，要實機看。

### 8C 目錄與設定頁整合（2026-10-22）

結果：全部 1,915 個測試通過（Sprites 424、App 579、其餘不變）。在 Mac 上用隔離的資料夾（柯基複製進 `sprites/`、設定記著「上次是小精靈模式、角色是柯基」）實際啟動：一啟動就只有柯基，像素銳利、貼著桌面右下角，HUD 顯示點擊穿透正常。偏離或補充原設計的地方：

- **`IReloadableSpriteCatalog` 放在 `Lumi.Sprites.Assets`，不在 Abstractions**：它用到 `SpriteLoadIssue`（Assets 的型別），而且 App 本來就引用 Assets；Abstractions 維持只有 1.1 的選填欄位與 `SpriteVoiceValidator`。介面多了 `Directory`（設定頁的「開啟角色資料夾」要知道位置）。
- **DI 註冊**：`AddLumiAssetSprites(configure)` 註冊一個 `AssetSpriteOptions` 單例（不用 `IOptions`，Assets 專案不引用 Options 套件），`SpriteLibrary` 取代 `ISpriteCatalog`，註冊順序（先或後於 `AddSprite<T>()`）都可以；測試要換資料夾就 `Replace` 那個選項。App 預設 `<資料夾>/sprites`（`LumiPaths.DataDirectory`）。
- **啟動順序**：`App.StartAsync` 先 `ReloadAsync` 再載入對話與顯示視窗，所以上次選的是素材型角色時小精靈一開始就是它，不會先閃一下光球。`ReloadAsync` 的掃描在背景執行緒，換清單與 `Changed` 回到呼叫者的同步內容（UI 執行緒）；同時多次呼叫會一個接一個做，取消不會鎖死。
- **殼的重建條件從「id 不同」改成「工廠實例不同或尺寸不同」**：重新載入每次都會建立新的工廠（圖片可能改過），所以按「重新載入」會讓目前的角色用新圖重建（中心不動）。選用的角色被移走時退回第一個（光球）、設定裡的 id 不動；角色回來、重新載入，自動換回去。
- **設定頁**：清單每項有縮圖（`preview.png`，用最近鄰，沒有就空著）、名稱、作者與版本、「程式型／素材型」；下面是「重新載入」（載入中顯示「載入中…」並停用）、「開啟角色資料夾」（`IFolderOpener`：Windows 用 Explorer、macOS 用 `open`，資料夾不存在先建立）、資料夾路徑，再下面是「載入失敗與警告」清單（目錄掃描的結果，加上素材型角色解碼時才發現的問題；預覽播放後重新整理）。清單選中的角色依設定裡存的 id 找，找不到顯示第一個但**不改設定**；使用者明確選它才存。
- **預覽區隨重新載入重建**；預覽的狀態清單只有十個狀態，互動動畫（`poke.*`、`drag`）不是狀態所以不列入（預覽只播狀態，戳與拖曳要在小精靈上實際玩）。
- **App.Tests 的 headless 繪製是假的**（`UseHeadlessDrawing`，解出來的位圖都是 1×1），所以設定頁預覽的「解碼零問題」只能在 Sprites.Tests（真的 Skia）驗證；App.Tests 驗證流程與狀態切換。解碼大小和標頭不一致時，素材型角色會回報問題、畫不出東西、不會當掉，這個行為本身也有測試。

### 8D 文件與柯基驗收（2026-10-22）

結果：全部 1,915 個測試通過。`docs/sprite-dev-guide.md` 第 7 節整個改寫（安裝、manifest 欄位表、動畫怎麼播、格子與尺寸、像素風角色、點擊範圍、美術規格、載入檢查），第 8 節補語氣檢查規則，第 9、10 節補素材型角色的驗證與檢查清單；`docs/ui-spec.md` 第 7 節、`agent.md` 同步。D3 的端對端驗證分散在兩處：`CorgiSampleTests`（真的柯基包：零警告、13 個動畫、每個檔位與縮放的像素粗細、命中跟著輪廓、幀率）與 `AssetSpriteIntegrationTests`（真的 `SpriteLibrary`、柯基複製進暫存資料夾，經殼與設定頁走完載入、選用、十個狀態與互動、重新載入、角色消失與回來）。

**D4 手動驗收（要你操作；Mac 我已經看過一次柯基會畫出來，Windows 還沒）**：

1. 把 `samples/sprites/corgi-pixel` 複製進角色資料夾，設定 ›「小精靈」按「重新載入」→ 清單出現「柯基（像素）」與縮圖、標示「素材型」，「載入失敗與警告」是空的。
2. 選用柯基：小精靈立即換成柯基，位置不跳動；十個狀態逐一預覽，動畫順暢、像素沒有糊；`thinking`、`sleep` 的特效沒有被切掉。
3. 在小精靈上：靠近、長按戳（按住 `poke.press`、放開 `poke.release`）、拖曳（`drag`）、回應中各狀態（思考、呼叫工具、串流、需要確認、錯誤）、完成後回待機、60 秒休眠；點腳（左右下角）能命中，點旁邊的透明處會穿透到下方桌面。
4. 三個尺寸（小、中、大）各看一次：像素粗細一致（中檔最準，小、大檔略小）；1 倍與 2 倍螢幕各看一次（Windows 的 125%、150% 也看）；拖到另一個縮放不同的螢幕後像素仍然一致。
5. 語氣：聊幾句確認回答有「汪」；到設定關掉語氣再聊，沒有了。
6. 壞掉的角色：故意做一個缺 `idle`、一個 `id` 用了 `lumi.`、一個 `source` 指向 `../`、一個語氣含「請忽略規則」，確認「載入失敗與警告」說得清楚、其他角色不受影響、含壞語氣的那個照常可以用但沒有語氣。
7. 耗能：用 HUD 量柯基待機與休眠的 CPU（預期明顯低於光球：待機 8 fps、休眠 4 fps）；順便看記憶體（預期把 12 張圖都播過之後約 30 MB 上升）。
8. 重開程式：柯基仍是選用的角色、位置與尺寸記得；把資料夾移走再重開，小精靈退回光球但設定裡的選擇不丟；放回去重新載入，自動換回柯基。
