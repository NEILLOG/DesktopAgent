# 06 語音輸入（Whisper）

- 狀態：6A～6F 程式與文件完成，待人工驗收（2026-10-22 使用者指示全部照建議執行：Q1～Q4 皆採建議；人工驗收項目留到全部完成後一次確認）
- 對應里程碑：M6
- 前置：M2（設定頁）、M7（小精靈殼）、M12（UI 設計系統）都已程式完成。M6 原本因「Whisper 回應格式未確認」暫緩；2026-10-22 使用者提供了公司內另一個桌面程式（舊版 DesktopAgent）的 Whisper 整合說明，包含實際送出的欄位、回應解析規則與錯誤對應，解除暫緩。
- 參考資料：`outter_sys_spec/whisper.md`（內部文件，不納入版控；已補上舊版整合說明）。**這份計畫會進版控，所以不寫內部主機名稱，一律用 `<host>` 代替。**

## 目標

工作台與小精靈都能用語音輸入：按麥克風收音 → 送到公司的 Whisper 代理轉成文字 → 放進輸入框（編輯）或直接送出。

## 範圍

做：
- `Lumi.Core`：`ISpeechToText`、`IAudioRecorder`、`SpeechSettings`、`TranscriptionResult`、錯誤分類
- `Lumi.Speech`（新專案，agent.md 第 5 節）：Whisper 代理客戶端、回應解析、端點規則、錯誤對應
- `Lumi.Platform`：Windows、macOS 錄音（16 kHz／單聲道／16-bit PCM），麥克風權限
- 設定 ›「語音」頁：端點、語言、逾時、Token、測試連線
- 工作台收音面板（ui-spec 6.3）、小精靈收音列（ui-spec 8.4）、`Listening`／`Transcribing` 狀態
- 文件同步：`docs/ui-spec.md` 6.2、6.3、7、8.2、8.4，`agent.md` 第 11 節，README，主計畫

不做：
- 說話者分離（`mode=diarization`、`min_speakers`／`max_speakers`）：語音輸入是單人口述，不需要（Q2）
- 即時（串流）轉寫、本機 Whisper（whisper.net，主計畫已列為「之後」）
- 把轉寫結果存成「語音訊息」型別：訊息只存文字，沒有輸入模式欄位；M9 再看是否需要（M9 評估結果：不做）
- 針對語音調整 prompt 或關閉思考（舊版有 `DisableThinkingForVoice`，Lumi 沒有對應設定）
- macOS 的 `Info.plist` 說明文字、`audio-input` entitlement、簽章與公證：M11（M6 先寫進風險與驗收說明）

## 舊版整合說明帶來的結論

| 項目 | 舊版（WPF，Windows only） | Lumi 的做法 |
|---|---|---|
| 呼叫方式 | `POST multipart/form-data` + `Authorization: Bearer`，代理端點的 `…/v1/Transcribe` 路徑 | 相同 |
| 實際送出的欄位 | `file`（`audio/wav`、`voice.wav`）、`mode=normal`、`language`（非空才送）、`return_format=json`、`source_pgm`、`file_name`；另帶 `Accept: application/json` | 相同，另外加 `session_id`＝對話 ID（agent.md 第 11 節）；`source_pgm` 固定 `Lumi` |
| 回應解析 | 寬鬆：`text/*` 直接用；JSON 找 `text` → `result`／`data`／`transcription`／`transcript`／`output` 包裝層（最多 3 層）→ `segments[].text`；`language`、`duration` 有就讀；中日文 segments 不補空白 | 相同；**測試用的 fixture 是依這段描述構造的，不是真實錄製**（見風險，驗收時抓真實回應補一份） |
| 錄音 | NAudio（Windows） | NAudio 不能跨平台，且 agent.md 規定原生呼叫只在 `Lumi.Platform`：自己用 P/Invoke 寫兩個平台（Q1） |
| Token 保存 | DPAPI 加密成 `speech-whisper.dat`，**綁定端點 Host**，換 Host 要重新輸入 | `ISecretStore`（鍵名 `speech/whisper/token`，`SecretKeyTests` 已有此範例）；Host 綁定記在 `SpeechSettings.TokenHost`，換 Host 就不送 Token，設定頁要求重新輸入 |
| 可用判斷 | 端點通過驗證且已存有 Token → 麥克風鈕啟用 | 相同，另加「有麥克風權限」不預先判斷（macOS 要實際收音才知道，見 6C） |
| 邊界 | 少於 0.5 秒忽略；上限 90 秒自動結束；結果為空提示「沒有聽到內容」 | 相同 |
| 錯誤對應 | 401／403／404／413／429／408／504／5xx／本機逾時／TLS 憑證／其他連線失敗 | 相同分類；訊息不含伺服器回應內文、不含 Token |
| TLS | 內部 CA 要裝到系統受信任根憑證，不關閉驗證 | 相同，並補 macOS：安裝到「鑰匙圈存取」並設為信任（Q3） |
| 取消 | 使用者取消靜默回閒置 | 相同；**但轉寫失敗時保留錄音**，提供「重試轉寫」「放棄」（ui-spec 8.4，舊版沒有） |
| 停止後 | 只有「停止並送出」 | 兩種：「停止並編輯」「停止並送出」（ui-spec 6.3） |

---

## 開工前要確認的事

以下是影響結構或行為的決定，各給了建議；**確認前不動工**（agent.md 第 3 節）。

| # | 問題 | 建議 | 影響 |
|---|---|---|---|
| Q1 | 錄音要怎麼做？舊版用 NAudio，只有 Windows。 | **自己寫兩個平台的 P/Invoke**：Windows 用 `winmm` 的 `waveIn`（可以直接要 16 kHz／單聲道／16-bit，不用重新取樣）、macOS 用 AudioToolbox 的 `AudioQueue`（同樣可直接指定 PCM 格式）。理由：agent.md 已把「錄音」列在 `Lumi.Platform` 的 P/Invoke 範圍、技術棧不加新的原生依賴，兩平台程式量都不大（各約 200 行），WAV 檔頭與音量計算是平台無關的純邏輯，可單元測試。替代做法：Windows 用 NAudio 的 `WaveInEvent`、macOS 自己寫——Windows 這邊省事，但多一個 NuGet 依賴與授權，而且兩個平台行為不一致；或用跨平台原生套件（PortAudio／OpenAL），要隨 app 打包各平台的 dylib／dll，M11 會更麻煩。 | 6C |
| Q2 | 主計畫 M6 寫「設定頁：API 網址、Token、模式、語言、講者數、逾時」。舊版只送 `mode=normal`，沒有講者數。要保留模式與講者數嗎？ | **不放**：語音輸入是單人口述，`diarization` 會回講者分段，輸入框也沒地方顯示。設定頁只有端點、語言、逾時、Token（和舊版一致），`mode` 固定 `normal`。要做會議逐字稿之類的用途再開新的里程碑。這是對主計畫 M6 項目的調整，確認後我改主計畫並記進決策紀錄。 | 6B |
| Q3 | 內網憑證（主計畫待確認事項）：Whisper 服務若是自簽或內部 CA？ | **不提供「略過憑證驗證」選項。** 憑證不受信任時，錯誤訊息指引安裝內部 CA（Windows：受信任的根憑證授權單位；macOS：「鑰匙圈存取」匯入並設為「永遠信任」）。理由：Token 會送過這條連線，略過驗證等於讓任何中間人拿到公司 Token；舊版也是這樣做、沒有出過問題。（替代做法：設定頁加「信任此憑證指紋」，使用者貼上指紋——可行但屬於新功能，需要時再加。） | 6A、6B |
| Q4 | 端點預設值：舊版預設是內網 URL。Lumi 的原始碼在版控裡，要把公司主機名稱寫進預設值嗎？ | **預設留空**，使用者在設定頁輸入一次（`src/Lumi.App/appsettings.json` 有進版控，也不能拿來預填）。理由：公司主機名稱不該進版控（`outter_sys_spec/` 已因此被 `.gitignore`）；欄位的說明文字與測試只用 `https://<host>/...`。代價：第一次使用要多貼一行網址。端點沒設定時，麥克風鈕停用並寫出原因「尚未設定語音服務」。 | 6A、6B |

其他我直接定的事（有意見再指出；數字都集中在 `SpeechOptions`／`SpeechSettings`，之後可調）：

- 錄音上限 90 秒、最短 0.5 秒、取樣 16 kHz／單聲道／16-bit；WAV 只存在記憶體（上限約 2.8 MB），不寫磁碟。
- 逾時預設 120 秒，實際限制 5～600 秒（讀取設定時 clamp）；語言預設 `zh`，留空則不送 `language`。
- 語言欄位用文字輸入加常用語言快選（zh、en、ja、留空＝由服務端決定），不做完整的語言清單。
- 測試連線送 0.5 秒靜音 WAV，只確認服務接受請求與 Token，**不檢查辨識內容**。
- 端點規則：必須 https（只有 localhost 可用 http）、不含帳密／查詢字串／錨點；規則寫在 `SpeechEndpointRules`（Core，純函式）。
- 錯誤訊息不含伺服器回應內文（比照 `OpenAIHttpErrors`）；Token 以 `SecretValue` 傳遞，只在組請求標頭時 `Reveal()`；Authorization 標頭設在每次請求上，不放 `DefaultRequestHeaders`（比照 `AddLumiProviders` 的做法，避免出現在 `HttpClientFactory` 的 log）。
- 逾時用每次請求自己的 `CancellationTokenSource.CreateLinkedTokenSource` + `CancelAfter`，才分得出「本機逾時」（顯示「超過 N 秒沒有回應，可調高逾時秒數」）與「使用者取消」（靜默）。`HttpClient.Timeout` 設無限（同 OpenAI 相容 Provider）。
- 回應解析器是純函式，不碰網路；多個解析結果都找不到文字時丟 `SpeechFailure`（`UnrecognizedResponse`），**不猜格式**（agent.md 第 3 節）。
- 同一時間只有一個錄音／轉寫（工作台與小精靈共用同一個 `VoiceInputController`，單例）；回應中（`IChatSession.IsResponding`）、工作台「查看中」時麥克風停用並以 Tooltip 說明原因。
- 轉寫完成但使用者已經換對話：結果仍放進當時的輸入目標，不自動送到別則對話（「停止並送出」若對話已改變，改成放進輸入框並提示）。

---

## 設計

### 專案與相依

```
Lumi.App ──► Lumi.Speech ──► Lumi.Core
   └──────► Lumi.Platform ──► Lumi.Core   （IAudioRecorder 的兩個平台實作）
```

- `Lumi.Speech` 只引用 Core 與 `Microsoft.Extensions.Http`（typed client），**不引用 Platform、Avalonia**。
- 架構測試新增：`Lumi.Speech` 不引用 Avalonia／Providers／Platform／Infrastructure；原始碼不出現 `DllImport`／`LibraryImport`（既有的 `PlatformInteropRuleTests` 把 `Lumi.Speech` 加進清單）；`Lumi.Speech` 的原始碼不 log Token（`SettingsSecretRuleTests` 的延伸：Speech 的 log 呼叫不含 `Reveal()`）。

### Core（6A）

```csharp
public interface ISpeechToText
{
    /// <summary>wav：16 kHz／單聲道／16-bit 的 WAV（含檔頭）。</summary>
    Task<TranscriptionResult> TranscribeAsync(Stream wav, string fileName, string? sessionId, CancellationToken ct);
}

public sealed record TranscriptionResult(string Text, string? Language, TimeSpan? Duration);

public sealed record SpeechSettings(string Endpoint, string Language, int TimeoutSeconds, string? TokenHint, string? TokenHost)
{
    public static readonly SpeechSettings Default = new("", "zh", 120, null, null);
    public SpeechSettings Normalize() { /* clamp 5–600、trim、不認得的值改回預設 */ }
}

public enum SpeechFailureKind
{
    NotConfigured, InvalidToken, NotFound, TooLarge, RateLimited, ServerTimeout, ServerError,
    LocalTimeout, CertificateUntrusted, Network, UnrecognizedResponse, Unknown,
}

/// <summary>Message 是可以直接顯示給使用者的繁體中文；不含伺服器內文、Token。</summary>
public sealed class SpeechFailure(SpeechFailureKind kind, string message, bool retryable) : Exception(message)
```

- `AppSettings` 加 `Speech`（init 屬性，同 `Persona`、`Sprite` 的做法，舊檔沒有時用預設，版本號維持 1）；`IModelCatalog` 加 `SetSpeechAsync(SpeechSettings, SecretValue? newToken)`、`HasSpeechTokenAsync`、`ClearSpeechTokenAsync`（比照連線金鑰）。儲存時若端點 Host 與 `TokenHost` 不同，**視為沒有 Token**：沒帶新 Token 就清掉舊的（`ISecretStore` 刪除、`TokenHint`／`TokenHost` 清空），設定頁提示重新輸入；帶了新 Token 就以新 Host 記錄。這樣舊 Token 不可能被送到新 Host。
- 錄音介面（6C 才用）：

```csharp
public interface IAudioRecorder
{
    bool IsRecording { get; }
    /// <summary>音量 0..1，約每 50 ms 一次（在背景執行緒發出，觀察者自己轉到 UI 執行緒）。</summary>
    event EventHandler<float>? LevelChanged;
    /// <summary>麥克風沒權限、沒有裝置、被其他程式獨佔等：丟 <see cref="AudioRecorderException"/>（Kind + 繁中訊息）。</summary>
    Task StartAsync(CancellationToken ct = default);
    /// <summary>停止並回傳 WAV（含檔頭，記憶體）。時間不足 0.5 秒時 <c>TooShort = true</c>。</summary>
    Task<RecordedAudio> StopAsync();
    void Cancel();   // 丟掉錄到的內容
}
public sealed record RecordedAudio(byte[] Wav, TimeSpan Duration, bool TooShort, bool LooksSilent);
```

  `LooksSilent`（整段 RMS 幾乎為零）專門處理 macOS：沒有麥克風權限時 `AudioQueue` 不會報錯，只會收到靜音，用它提示「沒有收到聲音，請確認麥克風權限」，不送出去浪費一次請求。

### `Lumi.Speech`（6A）

| 檔案 | 內容 |
|---|---|
| `WhisperSpeechToText` | `ISpeechToText` 實作：讀 `IModelCatalog.Current.Speech` 與 Token → 驗證端點 → 組 multipart → 送出 → 解析；Token 的 Host 與端點 Host 不同就丟 `NotConfigured` |
| `WhisperResponseParser` | 純函式，規則見上表；輸入是 `(contentType, body)`，輸出 `TranscriptionResult` 或 `null`（找不到） |
| `WhisperErrorMapper` | 狀態碼與例外 → `SpeechFailure`（401／403→`InvalidToken`、404、413、429、408／504→`ServerTimeout`、5xx、TLS：`HttpRequestException` 內的 `AuthenticationException`→`CertificateUntrusted`、其他 `HttpRequestException`→`Network`） |
| `SilentWav` | 產生測試連線用的 0.5 秒靜音 WAV（純函式，`WavFormat` 與錄音共用 Core 的檔頭寫入） |
| `SpeechServiceCollectionExtensions` | `AddLumiSpeech()`：`AddHttpClient<WhisperSpeechToText>(c => c.Timeout = Infinite)`、`ISpeechToText`、`ISpeechConnectionTester` |

測試連線：`ISpeechConnectionTester.TestAsync(SpeechSettings, SecretValue token, ct)` → 成功或 `SpeechFailure`，設定頁用（編輯中的值尚未儲存也能測，比照 `IConnectionTester`）。

測試（6A）：fake `HttpMessageHandler`；表單欄位逐項斷言（有送／沒送、`language` 空白不送、`session_id` 有對話才送）；Authorization 標頭只在 Host 吻合時才帶；每種狀態碼的對應；本機逾時 vs 使用者取消；解析器對各種形狀（純文字、頂層 `text`、包裝層 1～3 層、第 4 層找不到、`segments` 中文不補空白／英文補空白、空字串、非 JSON、`language`／`duration`）。

### 設定頁（6B）

- 設定視窗左側 Menu 加第五個分類「語音」（副標「Whisper 語音辨識」，圖示用既有的 `LumiIconMic`），放在最後（索引 4，在「小精靈」之後），避免移動既有分類的索引與測試。
- 頁面（`SpeechViewModel` + `SpeechPanel`），欄位用 `FormField`，不包卡片：
  - 服務端點（必填；說明「完整網址，必須是 https」；錯誤文字即時顯示端點規則的原因）
  - 語言（選填；說明「留空由服務端判斷」）、逾時秒數（5～600）
  - Token（遮罩、可切換顯示；已設定時顯示「已設定 · 結尾 xxxx」與更換／清除；Host 不符時顯示警告「端點主機已變更，請重新輸入 Token」）
  - 底部：測試連線（secondary，結果同連線頁：Success 淡底加打勾／Danger 淡底加三角圖示加文字）、儲存（primary，因欄位無效而停用時旁邊寫原因）
  - 說明 Caption：「Token 需具備 WHISPER_API 權限；只存在這台裝置」「憑證不受信任時，請把公司的內部 CA 安裝到系統」
- 版面、間距、token、按鈕層級照 M12 的規則；新的字串進 `Strings.resx`；`ui-spec` 第 7 節新增「語音」小節（6F 同步）。

### 錄音（6C）

- `WavFormat`（Core，純函式）：`WriteHeader`／`FromPcm`；16 kHz／1 ch／16-bit。單元測試：檔頭欄位、長度計算、空資料。
- `LevelMeter`（純函式）：PCM → RMS → 0..1（含對數壓縮，波形比較有感）。
- `RecordingLimits`：90 秒上限（到了自動停止並發出事件，控制器照「停止並編輯」處理）、0.5 秒下限。
- `WindowsWaveInRecorder`（`[SupportedOSPlatform("windows")]`）：`waveInOpen`（`CALLBACK_FUNCTION`，用 `[UnmanagedCallersOnly]`）、3～4 個 100 ms 的緩衝輪流重送、`waveInStop`／`Reset`／`Close` 的順序與緩衝釋放（用 `SafeHandle` 或 `try/finally`，不洩漏原生記憶體）。沒有預設裝置、被獨佔 → `AudioRecorderException`。
- `MacAudioQueueRecorder`（`[SupportedOSPlatform("macos")]`）：`AudioQueueNewInput`（指定 `AudioStreamBasicDescription`）、回呼用 `[UnmanagedCallersOnly]` 加 `GCHandle` 帶回託管物件、`AudioQueueAllocateBuffer`／`EnqueueBuffer`／`Start`／`Stop`／`Dispose`。權限：`AVCaptureDevice.authorizationStatusForMediaType(audio)`（沿用 `ObjC.cs` 的 `objc_msgSend`）；`NotDetermined` 時先 `requestAccess` 再開始；`Denied` 丟 `AudioRecorderException(PermissionDenied)`，訊息指引「系統設定 › 隱私權與安全性 › 麥克風」。
- 兩個實作都只是「把 PCM 交給純邏輯」：累積 → `WavFormat`、音量 → `LevelMeter`、時間限制 → `RecordingLimits`，所以平台程式碼越薄越好，**純邏輯用單元測試**，平台程式碼用真機手動驗收（C 清單）。
- `AddLumiPlatform` 依作業系統註冊 `IAudioRecorder`（單例；一次只錄一個）。
- 測試：`FakeAudioRecorder`（Core.Tests 或共用測試輔助）供控制器、畫面測試用；`Platform.Tests` 的 `ContractImplementationTests` 加錄音實作的型別檢查（兩個平台的實作都有實作介面），無法在 CI 真的收音。

### 語音輸入控制器與工作台（6D）

`VoiceInputController`（`Lumi.App/Services`，單例，工作台與小精靈共用）——狀態機，**不碰畫面**，用 `IUiDispatcher` 把事件帶回 UI 執行緒：

```
Idle ──Start──► Recording ──StopEdit/StopSend/達 90 秒──► Transcribing ──成功──► Idle（結果交給呼叫端）
                  │ Cancel                                   │ Cancel（取消請求，丟掉錄音）
                  ▼                                          ▼ 失敗
                 Idle                                      Failed（錄音保留）──Retry──► Transcribing
                                                              └─Discard──► Idle
```

- 公開：`State`、`Elapsed`、`Level`（給波形）、`Failure`、`CanStart`／`DisabledReason`（沒設定端點、沒有 Token、回應中、查看中、已在錄音 → 各自的原因文字）、`StartAsync`、`StopAsync(VoiceStopAction)`、`Cancel`、`RetryAsync`、`Discard`，與事件 `Transcribed(text, VoiceStopAction, target)`。
- `VoiceStopAction`：`Edit`（文字放進輸入框、標「語音轉寫」）、`Send`（轉寫完直接 `IChatSession.SendAsync`）。
- 沒聽到內容（結果空白）：提示「沒有聽到內容」，回 Idle；錄音太短：「錄音時間太短，已忽略」，不送出。
- 離開時清理：視窗關閉、對話切換、App 結束 → `Cancel`；`IAsyncDisposable`，不留麥克風開著。
- **工作台**（ui-spec 6.3）：麥克風鈕改成可用（拿掉「即將推出」Caption）；錄音時收音面板**取代輸入框**（`VoicePanel`：紅點、`mm:ss / 01:30`、波形、取消／停止並編輯／停止並送出）；轉寫中轉圈加錄音長度，可取消；失敗時顯示錯誤文字加「重試轉寫」「放棄」；成功的「編輯」結果放進 `InputText`，輸入框上方一個小「語音轉寫」標籤，使用者一改文字或清空就拿掉標籤。
- 波形控制項 `LevelWaveform`（`Lumi.App/Controls`）：固定數量的長條，每個新的 Level 往左推；顏色只用 token、不自己開計時器（跟隨 Level 事件重繪，或用 `FrameDriver`——小精靈那邊的限制同樣適用於「不自己開計時器」）；`AutomationProperties` 提供文字說明（「收音中 00:12」）。
- 無障礙：麥克風鈕與三個動作鈕都有 `ToolTip.Tip` 加 `AutomationProperties.Name`；錄音開始／結束／失敗用 live region 或狀態文字通知（不只靠顏色）；Esc＝取消錄音。
- 設計規則照 M12：只用 token、間距 4／8／12／16／24／32、不用 Unicode 符號、每個動作區最多一個 primary（收音面板的 primary 是「停止並送出」）；**所有新的 UI 細節以 `plan/ui-guide/ui-guide.html` 為準**，需要偏離時先問。

### 小精靈（6E）

- 輸入列加麥克風鈕、收音列取代輸入列（ui-spec 8.4）。輸入列尺寸被 `SpriteInputBarLayoutTests` 鎖定（300×48、輸入框 240×12 在 (17,18)、送出鈕 36×36 在 (257,6)），加麥克風鈕要重排——**排法在 6E 開工時依 ui-guide 與現有測試決定並更新該測試**，不在這份計畫預先定死。
- `SpriteStateMapper` 加來源：`VoiceInputController.State`：`Recording`→`Listening`、`Transcribing`→`Transcribing`；對話事件的狀態優先序不變（回應中時麥克風本來就停用，所以兩者不會同時發生）。角色早就支援這兩個狀態，**殼要負責退化**（`Transcribing → Listening → Idle`，既有的 `SpriteStateResolver`）。
- 停止並送出：氣泡先以小字引用轉寫結果，再進入思考（ui-spec 8.4）；停止並編輯：文字留在輸入列並標上「語音」。
- 失敗：氣泡顯示一句說明加「重試轉寫」「放棄這段錄音」；失敗氣泡不自動淡出（比照錯誤氣泡）。
- 小精靈隱藏（暫時隱藏）期間不能開始錄音；錄音中收到需要權限確認時，錄音不中斷（確認在氣泡內，輸入列已被收音列取代，確認區照常顯示）。

### 文件與收尾（6F）

- `docs/ui-spec.md`：6.2（麥克風不再是佔位）、6.3（拿掉「回應格式未確認」）、7（新增「語音」頁）、8.2／8.4、第 9 節 RWD 表的麥克風 Caption 列、停用佔位清單。
- `agent.md` 第 11 節：拿掉「回應格式尚未確認」，改寫成已定案的規則（欄位、解析寬鬆但找不到就失敗、Token 的 Host 綁定、錄音只在記憶體、不略過憑證驗證）；第 5 節 `Lumi.Speech` 保持原樣；第 6 節介面表補 `ISpeechConnectionTester`。
- `README.md`：語音輸入的說明與專案清單。
- 主計畫：M6 狀態、Q2 的調整、決策紀錄（Q1～Q4）、「待確認事項」兩項（回應格式、憑證）改成已確認並註明依據。
- `outter_sys_spec/whisper.md`：補上舊版整合說明（已做，本機檔，不進版控）。

---

## 拆成六個部分

| 部分 | 名稱 | 產出 | 依賴 |
|---|---|---|---|
| **6A** | Core 契約與 `Lumi.Speech` | `ISpeechToText`、`SpeechSettings`、`SpeechFailure`、端點規則、Whisper 客戶端、解析器、錯誤對應、測試連線、DI；**沒有畫面變化** | — |
| **6B** | 設定頁「語音」 | `AppSettings.Speech` 讀寫、Token 存取與 Host 綁定、`SpeechViewModel`／`SpeechPanel`、測試連線按鈕、字串資源 | 6A |
| **6C** | 錄音 | `IAudioRecorder`（Core）、`WavFormat`／`LevelMeter`／`RecordingLimits`（純邏輯＋測試）、Windows `waveIn`、macOS `AudioQueue`＋權限；**可以先用假的錄音器接 6D** | — |
| **6D** | 控制器與工作台收音面板 | `VoiceInputController`、`VoicePanel`、`LevelWaveform`、麥克風鈕啟用條件、編輯／送出、失敗重試 | 6A、6B、6C |
| **6E** | 小精靈 | 輸入列麥克風鈕、收音列、`Listening`／`Transcribing`、氣泡引用與失敗處理 | 6D、M7 |
| **6F** | 文件與手動驗收 | 文件同步、真實服務與兩平台的人工驗收、補一份真實回應的 fixture | 6E |

6A、6C 互不依賴，可以交錯做；6B 要等 6A；6D 起才有使用者看得到的功能。每段結束都要能編譯、測試全過。

---

## 風險

1. **回應格式只有文字描述，沒有真實樣本。** 解析器依舊版文件的規則寫，且找不到文字時失敗、不猜。6F 驗收時用真實服務抓一份回應存成 fixture（去掉任何識別資訊），並用它跑一次解析測試；若形狀跟描述不同，在這裡記錄偏離並修正。
2. **macOS 麥克風權限。** 從 `dotnet run`／IDE 執行時，權限是跟著「終端機或 IDE」走；打包成 `.app` 後要 `Info.plist` 的 `NSMicrophoneUsageDescription` 與簽章後的 `com.apple.security.device.audio-input` entitlement（M11）。沒有權限時 `AudioQueue` 只收到靜音，所以要有 `LooksSilent` 與 `AVCaptureDevice` 權限檢查，兩者都要在真機驗證。
3. **兩個平台的原生錄音程式只能在真機驗證。** CI 沒有麥克風；平台層刻意寫薄，邏輯放純函式。Windows 的實機驗證在使用者的 Windows 機器上做（同 M0／M1 的做法），開工前要確認有人可以測。
4. **原生回呼與執行緒。** `waveIn` 與 `AudioQueue` 的回呼都在原生執行緒：回呼內只做「複製 PCM 到託管緩衝、算音量」，不碰 UI、不丟例外；停止時要等回呼全部結束再釋放（避免回呼存取已釋放的記憶體）。
5. **Token 與隱私。** 轉寫會把錄音與對話 ID（`session_id`）送到公司的服務；設定頁文字要寫明；Token 只存 `ISecretStore`、不進 log、不進例外訊息，並有架構測試守住。
6. **服務端逾時與長錄音。** 90 秒的 16 kHz WAV 約 2.9 MB，沒有分段上傳；若公司服務對檔案大小有上限會回 413，錯誤訊息已涵蓋。
7. **內網憑證。** 依 Q3，不略過驗證；使用者沒裝 CA 時第一次測試連線就會看到明確的指引。

---

## 測試與驗收

自動測試（每段都要綠）：Core／Speech 的單元測試（上面列的）、`WavFormat`／`LevelMeter`／`RecordingLimits`、`VoiceInputController` 狀態機（用 `FakeAudioRecorder`、`FakeSpeechToText`、假的時鐘與 `IUiDispatcher`：取消、失敗重試、達 90 秒、換對話、回應中停用）、設定頁 ViewModel（端點驗證、Host 變更要求重新輸入、測試連線）、Headless 畫面測試（麥克風鈕在各種狀態的啟用與 Tooltip、收音面板的按鈕與焦點順序、Compact／Narrow 版面）、架構測試（上面列的）、既有的色彩／間距／符號／按鈕規則測試自動涵蓋新的 XAML。

手動驗收（6F，使用者在真機）：

- [ ] Windows：設定語音 → 測試連線成功；工作台錄音 → 停止並送出／停止並編輯；達 90 秒自動結束；少於 0.5 秒被忽略；拔掉麥克風的錯誤提示
- [ ] macOS：第一次錄音的權限詢問；拒絕後的指引；允許後可用
- [ ] 兩平台：小精靈錄音列、`Listening`／`Transcribing` 動畫、氣泡引用、失敗後重試與放棄
- [ ] 錯誤：錯誤 Token（401）、沒有權限（403）、錯誤路徑（404）、斷網、本機逾時（把逾時調成 5 秒＋慢速服務或手動限速）、憑證不受信任
- [ ] 換端點 Host 後，Token 被要求重新輸入且不會送到新 Host
- [ ] 真實回應存成 fixture，解析測試通過

## 實作紀錄

### 6A

完成（2026-10-22）：Core 契約（`Lumi.Core.Speech`：`ISpeechToText`、`SpeechSettings`、`SpeechFailure`、`SpeechEndpointRules`、`ISpeechConnectionTester`、`SpeechSecrets`、`WavFormat`）、`AppSettings.Speech` 與 `IModelCatalog` 的語音方法（`JsonModelCatalog` 實作，設定檔版本號維持 1）、新專案 `Lumi.Speech`（`WhisperSpeechToText`、`WhisperResponseParser`、`WhisperErrorMapper`、`WhisperConnectionTester`、`AddLumiSpeech()`，已在 `Lumi.App` 的組裝處註冊）、測試與架構規則。沒有畫面變化。

偏離原計畫或需要記下的地方：

- 靜音 WAV 用 Core 的 `WavFormat.Silence(TimeSpan)`，原計畫表格中的 `SilentWav` 不另外建立（避免兩份檔頭程式）。
- `SecretStoreException`（Token 讀不出來）對應到 `NotConfigured`，訊息是「語音 Token 無法讀取，請重新輸入」；原計畫的分類表沒有單獨的種類，沒有另外加。
- 解析器對 `segments` 的處理：只要 `segments` 是陣列（即使是空的，或每段都沒有 `text`），就視為「找到但為空」，不回 null。理由：空陣列代表服務沒有聽到內容，應由上層提示「沒有聽到內容」，而不是當成格式錯誤。這一點屬於對描述的解讀，真實回應到手後要確認。
- 解析器的包裝層順序：每一層先看 `text`，再往下一層包裝（最多 3 層），最後才看該層的 `segments`；`language` 先看找到文字的那一層，沒有再看頂層。
- 端點規則的 `?`、`#` 以字串檢查（而不是 `Uri` 的屬性），避免 `https://x/?` 這種空的查詢字串漏掉。
- 端點為空時，`SetSpeechAsync` 允許儲存（「尚未設定」）；端點不是有效網址、或沒有端點卻帶 Token，丟 `ArgumentException`（同 `SaveConnectionAsync` 的 Base URL 檢查）。
- Host 綁定：端點清空時 Host 視為不同，綁定的 Token 會被刪掉（依原計畫「Host 不同即視為沒有 Token」的字面規則）。
- `SpeechSettings` 的建構參數都有預設值，讓 settings.json 裡缺欄位時 System.Text.Json 能直接還原（逾時缺欄位時不會變成 0 再被 clamp 到 5）。
- 測試連線的靜音檔名是 `connection-test.wav`（原計畫沒有指定）。
- `WhisperConnectionTester` 使用與轉寫相同的逾時（`TimeoutSeconds`）與同一套錯誤對應；回應內容不解析，只要 2xx 就成功。
- `CompositionRootTests`（App.Tests）也加上 `AddLumiSpeech()`，與 `Program.cs` 保持一致。
- `Lumi.Speech.Tests` 的 `Local_timeout_...` 測試實際等待最短的 5 秒，這是為了測真正的逾時路徑（不為測試另外開計時選項）。
- 解析器 fixture 是依描述構造的，不是真實錄製（風險 1，6F 補真實回應）。

驗證：`dotnet build Lumi.sln` 0 警告 0 錯誤；`dotnet test Lumi.sln` 全部通過：Core 271（基準 229 + 新增 42）、Infrastructure 111（97 + 14）、Tools 329、Sprites 424、Platform 54 + 13 略過、Providers 136、Architecture 67（63 + 4）、App 1199、Speech 67（新專案）。

### 6B

完成（2026-10-22）：設定視窗第五個分類「語音」（索引 4，在「小精靈」之後；副標「Whisper 語音辨識」，圖示 `LumiIconMic`）。`SpeechViewModel`（Transient）、`SettingsViewModel` 加 `Speech`／`ShowSpeech`／`SettingsSection.Speech`、`HasUnsavedChanges` 納入語音草稿、頁面與字串（43 個，`Strings.resx` ＋ `Strings.cs`）、`ui-spec` 第 7 節「語音」小節與 Menu 表。沒有改 6A 的 Core 契約。

偏離原計畫或需要記下的地方：

- 頁面寫在 `SettingsWindow.axaml`，沒有獨立的 `SpeechPanel` 元件，和人格頁的做法一致（同一個 DataContext 結構）。
- 語言快選用四個 tertiary 按鈕（留空、zh、en、ja），不用 ComboBox；文字就是按鈕名稱，不另外加 Tooltip。
- 逾時是文字輸入，超出 5～600 或不是整數時是**錯誤**（欄位下方文字，儲存與測試停用），不是靜默 clamp。`SpeechSettings.Normalize()` 的 clamp 只當最後防線。
- 端點空白時的原因寫「請先輸入服務端點」（計畫原文），沒有用規則的「尚未輸入服務端點。」；其他端點錯誤直接用 `SpeechEndpointRules` 的原因。
- 「測試連線」在沒有任何 Token（既沒輸入也沒儲存）或端點主機已變更且沒輸入新 Token 時停用，並寫出原因（計畫沒有寫這個停用條件）。
- 測試連線的 Token 讀取：已儲存的 Token 由 `SpeechViewModel` 注入的 `ISecretStore` 以 `SpeechSecrets.WhisperToken` 讀出，直接交給測試器，不留在 ViewModel；讀不出來（`SecretStoreException`）顯示「語音 Token 無法讀取，請重新輸入。」。
- 「清除」Token 是立即動作（先確認，比照連線金鑰的清除），不經過「儲存」。
- Token 綁定的主機變了又沒有輸入新 Token 時，儲存會照 6A 的規則刪除舊 Token；設定頁在這之前已用警告文字提示，沒有額外的確認框。
- 測試連線沒有取消鈕（連線頁有）：逾時由 `WhisperConnectionTester` 的本機逾時控制，計畫沒有要求取消。
- 「還原」只丟掉未儲存的草稿與輸入中的 Token，不清除已儲存的 Token（清除有自己的按鈕）。
- 動作列：沒有頁首的主要動作，底部是「測試連線」（secondary）、「還原」（secondary）、「儲存」（primary），對齊人格頁。
- 測試連線的成功與失敗樣式沿用連線頁的 `Border.result` 樣式（在 `SettingsWindow.axaml` 的 Window.Styles 重複宣告一份，因為 `ConnectionsPanel` 的樣式只在它自己的範圍內）。
- `FakeModelCatalog.SetSpeechAsync`（App.Tests 的假目錄）從 6A 的 `NotSupportedException` 佔位改成記錄參數並寫回 `Current`，讓測試可以驗證儲存時 Token 是否為 null。
- 測試資料的端點一律用 `https://speech.example.com/v1`（example.com 是保留網域），沒有寫入內部主機名稱。

既有測試的調整（不是弱化，是分類數與新頁面的正常更新）：

- `SettingsWindowShellTests`：分類數 4 → 5，並加入「語音」項目。
- `SettingsAccessibilityTests`：`CategoryName` 對應索引 4；方向鍵測試改為小精靈之後是「語音」並停在最後一個；`ExpectedOrder` 加入兩個語音狀態。停用的「還原」「儲存」（沒有變更時）不在 Tab 鏈上，所以期望順序裡沒有它們。
- `SettingsRig.Pages` 加入 4 個語音畫面狀態（`speech`、`speech-token-stored`、`speech-test-ok`、`speech-test-fail`），所以可存取性、對比、按鈕覆蓋等掃描自動涵蓋新頁面；`AppSurfaces` 也會多開這 4 個視窗（測試時間略增）。

新增測試：`SpeechViewModelTests`（端點規則、逾時驗證、儲存參數與 Token 為 null、Token 清除與確認、更換／取消、主機綁定與警告、測試連線的成功／失敗／測試中／無 Token／讀取失敗、還原、外部變更）、`Headless/SpeechPageTests`（Menu 第五項與鍵盤選取、欄位與按鈕名稱、按鈕層級、錯誤與停用原因文字、Token 狀態與主機警告、測試結果樣式）。

驗證：`dotnet build Lumi.sln` 0 警告 0 錯誤；`dotnet test Lumi.sln` 全部通過：Core 271、Infrastructure 111、Tools 329、Sprites 424、Platform 54 + 13 略過、Providers 136、Architecture 67、Speech 67、App 1279（1199 + 新增 80）。

待人工驗收（6F 一併）：Mac 上實際開設定視窗看語音頁的版面（寬度 720 附近的快選按鈕換行、Token 狀態行與警告的排版）；用真的服務端點測試連線（401、斷網、憑證不受信任的文字）；換端點 Host 後 Token 不送到新主機。

### 6C

完成（2026-10-22，程式碼與單元測試；原生錄音只在 macOS 上編譯驗證，沒有實際收音）：Core（`Lumi.Core.Speech`）新增 `IAudioRecorder`、`RecordedAudio`、`AudioRecorderException`／`AudioRecorderFailureKind`、`RecordingLimits`、`LevelMeter`、`PcmRecorderCore`（平台無關的累積器）；Platform 新增 `Recording/WindowsWaveInRecorder`（winmm `waveIn`）、`Recording/MacAudioQueueRecorder`（AudioToolbox `AudioQueue`），P/Invoke 宣告在 `Interop/Windows/WinMm.cs`、`Interop/Mac/AudioToolbox.cs`、`Interop/Mac/AVFoundation.cs`，`ObjC.cs` 加 `objc_getClass`、`SendPtrArg`；`AddLumiPlatform` 依作業系統註冊 `IAudioRecorder` 單例。測試：`FakeAudioRecorder`、Core 的純邏輯測試、Platform 的型別契約與註冊測試、手動 smoke 測試。沒有畫面變化。

偏離原計畫或需要記下的地方：

- **macOS 權限請求的取捨**：沒有實作 `requestAccessForMediaType:completionHandler:`（要自己組 Objective-C block 的 ABI，無法在這台機器上安全地實際觸發授權視窗來驗證）。改成：`Denied`／`Restricted` 直接丟 `PermissionDenied`；`NotDetermined` 不先要求，直接開始收音，由系統在第一次開始錄音時彈出授權視窗（**這點要在 Mac 真機確認**：若 AudioQueue 不會觸發授權，第一次就只會收到靜音，由 `LooksSilent` 提示；那時就要補 block 版本的請求）。
- **緩衝大小 50 ms，不是原計畫的約 100 ms**：`LevelChanged` 的規格是約每 50 ms 一次，緩衝大小直接決定回呼頻率，所以兩個平台都用 50 ms、4 個緩衝（原計畫的 3～4 個）。
- **Windows 的權限與裝置檢查**：原計畫只寫「沒有預設裝置、被獨佔 → `AudioRecorderException`」。另外加了兩件事：(1) 開啟前讀登錄檔 `CapabilityAccessManager\ConsentStore\microphone`（以及 `\NonPackaged`）的 `Value`，為 `Deny` 時丟 `PermissionDenied`，因為 waveIn 在權限被關閉時不一定報錯；(2) `waveInGetNumDevs() == 0` 時丟 `NoDevice`。`MMSYSERR_NODRIVER` 對應 `NoDevice`（訊息也提到隱私權設定），`MMSYSERR_ALLOCATED` 對應 `DeviceBusy`，`WAVERR_BADFORMAT` 對應 `Unsupported`，其他是 `Unknown`（訊息含錯誤代碼，不含敏感資訊）。**這些對應只能在 Windows 真機確認**。
- **`IAudioRecorder` 本身不繼承 `IDisposable`**（依計畫的介面表）；兩個平台實作都是 `IDisposable`，Dispose 等同 Cancel。6D 的控制器要在離開時呼叫 `Cancel()`。
- **停止的執行緒與釋放順序**：`StopAsync` 以 `Task.Run` 執行（等緩衝歸還會阻塞）。Windows 的順序是 `waveInStop` → `waveInReset` → 等所有緩衝歸還（最多 3 秒）→ `waveInUnprepareHeader` → `waveInClose` → 釋放記憶體與 GCHandle；若超時，**不釋放**記憶體與 GCHandle（寧可漏掉，也不讓驅動程式寫入已釋放的記憶體），並保留 `IsRecording = false`，讓下一次開始時由系統回報裝置被占用。macOS 用 `AudioQueueStop(immediate)` 加 `AudioQueueDispose(immediate)`，返回後不會再有回呼。
- **回呼的例外**：原生回呼與事件處理器都包在 try/catch 裡（不丟例外，也不讓事件處理器的例外中斷重新排入緩衝）。
- **音量事件在原生執行緒發出**，觀察者要自己轉到 UI 執行緒（計畫已寫）。
- **`FakeAudioRecorder`** 放在 `tests/Lumi.Core.Tests/TestSupport/` 與 `tests/Lumi.App.Tests/TestSupport/`（兩份相同，比照 `FakeModelCatalog` 的做法）。App.Tests 的那份目前沒有使用者，6D 才會用到。
- **手動 smoke 測試**：`tests/Lumi.Platform.Tests/Recording/MicrophoneSmokeTests.cs`，只有 `LUMI_RUN_MIC_TESTS=1` 才執行。**這次沒有執行**（會真的開麥克風），所以原生錄音路徑沒有在執行期驗證過。
- Windows 的 `waveInProc` 與 macOS 的 `InputCallback` 都是 `[UnmanagedCallersOnly]`，`WaveInProc` 使用 stdcall（x86 才有差別）。

只能真機驗證的項目（6F 一併）：

- Windows：麥克風清單與錄音長度（約 1 秒）、停止後 WAV 檔頭正確、拔掉麥克風的錯誤訊息、權限關閉時的 `PermissionDenied` 對應、錄音中途按停止的釋放（重複錄音不會 DeviceBusy）。
- macOS：第一次錄音是否出現麥克風授權視窗（`NotDetermined` 的取捨）、拒絕後的 `PermissionDenied`、`LooksSilent` 的提示、`AudioQueue` 的回呼在停止後不再進來、重複開始與停止不洩漏。
- 兩平台：`MicrophoneSmokeTests` 在有麥克風的機器上以 `LUMI_RUN_MIC_TESTS=1` 執行。

驗證：`dotnet build Lumi.sln` 0 警告 0 錯誤；`dotnet test Lumi.sln` 全部通過：Core 294（271 + 新增 23）、Infrastructure 111、Tools 329、Sprites 424、Platform 58 + 14 略過（54 + 13 略過；新增 4 通過、1 略過的手動測試）、Providers 136、Architecture 67、Speech 67、App 1279。

### 6D

完成（2026-10-22，程式碼與自動測試；真機收音、真實服務與讀屏軟體尚未驗證，見下）：`IVoiceInputController`／`VoiceInputController`（`Lumi.App/Services`，單例，DI 註冊；純狀態機，事件經 `IUiDispatcher` 回到 UI 執行緒）、`LevelWaveform`（`Lumi.App/Controls`）、`VoicePanel`（`Lumi.App/Views`，收音面板的三種狀態）、工作台的麥克風鈕與輸入區改寫（`WorkbenchViewModel`、`ChatComposer`）、字串（`Strings.resx`／`Strings.cs`）、`docs/ui-spec.md` 6.2、6.3、9.4。沒有動小精靈（6E）。

公開型別與簽章（6E 依賴的部分）：

```csharp
public enum VoiceInputState { Idle, Recording, Transcribing, Failed }
public enum VoiceStopAction { Edit, Send }
public sealed record VoiceTranscribedArgs(string Text, VoiceStopAction Action, string? ConversationId, string? Language);

public interface IVoiceInputController : INotifyPropertyChanged
{
    VoiceInputState State { get; }
    TimeSpan Elapsed { get; }              // 錄音中即時算；其他狀態 0
    TimeSpan? RecordedDuration { get; }    // 轉寫中與失敗時顯示用；閒置時 null
    double Level { get; }                  // 0..1，節流 100 ms
    string? LastFailureMessage { get; }    // 失敗時的繁中訊息（不含 Token）
    string? Notice { get; }                // 一次性提示；下次開始收音或 ClearNotice 時清除
    bool CanStart { get; }
    string? DisabledReason { get; }        // 不能開始時的原因（Tooltip 用）
    event EventHandler<VoiceTranscribedArgs>? Transcribed;
    Task StartAsync(CancellationToken ct = default);
    Task StopAsync(VoiceStopAction action);
    void Cancel();
    Task RetryAsync();
    void Discard();
    void ClearNotice();
}
```

偏離原計畫或需要記下的地方：

- **介面名稱與型別**：計畫的 `Failure` 實作成 `LastFailureMessage`（字串），`Transcribed(text, action, target)` 實作成 `VoiceTranscribedArgs`（多一個 `Language`，來自 `TranscriptionResult.Language`）。另外加 `RecordedDuration`（轉寫中顯示「錄音長度」要用）與 `ClearNotice()`。`Notice` 是字串，不是事件。
- **停止後立刻進入轉寫中**：`StopAsync` 一開始就切到 Transcribing（錄音器交出 WAV 要一點時間），畫面不會停在「收音中」。太短或沒聲音的錄音再回到 Idle。
- **切換對話會取消**：收音中、轉寫中或失敗中的錄音，只要 `IChatSession.ConversationChanged`（工作台或小精靈切換、新對話）就取消，不詢問，失敗的錄音也一併丟掉。這是計畫「至少 Cancel 錄音後再切換」中選的做法（比提示或禁用切換簡單，也不會把結果送到別的對話）。
- **回應開始**（`RunStarted`）只取消「收音中」；轉寫中的結果照「停止並送出」的規則，對話已在回應中就改成放進輸入框。
- **停止並送出的例外規則**：同一則對話、沒有在回應、輸入框是空的，才走 `SendCommand`；如果輸入框已有草稿，**不送出草稿**，改成把語音文字附加在後面並提示「輸入框已有文字，語音結果已附加在後面，請確認後再送出」（避免送出使用者沒有看過的草稿）。對話已改變或回應中時，「停止並送出」與「停止並編輯」都一樣放進輸入框並提示。
- **無障礙的 live region**：Avalonia 11.3 沒有 `AutomationProperties.LiveSetting`，改用焦點搬移：開始收音時焦點到「停止並編輯」，轉寫時到「取消」，失敗時到「重試轉寫」，回閒置時回到輸入框；狀態文字同時是面板的無障礙名稱。讀屏軟體的實際朗讀要真機驗證。
- **焦點搬移延後一拍**：面板剛顯示時 `Focus()` 會回傳 false，所以 `ChatComposer` 用 `Dispatcher.UIThread.Post(…, DispatcherPriority.Loaded)` 搬移焦點。
- **下層工具列（收音時）**：「加入檔案」「加入資料夾」用 `ContextBarViewModel.IsLocked`（和回應中相同的鎖）停用，Tooltip「語音輸入中，暫時無法加入」；這同時鎖住移除 chip 與貼上截圖，範圍在收音時不改變。送出鈕與提示文字**隱藏**（不是停用：收音面板有自己的「停止並送出」，隱藏比停用更不容易誤按）。麥克風鈕保留、停用，Tooltip 寫「已在錄音或轉寫中」。輸入區高度：面板的最低高度與輸入框相同（Normal 64、short 起 48）。
- **停止並編輯的圖示**：計畫沒有指定，用方形（`LumiIconSquare`，與停止同形）；停止並送出用箭頭（與送出鈕相同）；取消用 X。
- **按鈕層級**：收音列每個動作區只有一個 primary（停止並送出）；失敗列是 primary（重試轉寫）加 tertiary（放棄）。
- **麥克風 Caption 與 `Composer_MicSoon`** 已拿掉（resx 與 `Strings.cs` 一起），`ResponsiveWorkbenchTests` 的 MicCaption 斷言一併移除。
- **停用原因文字**：沒有設定端點是「尚未設定語音服務，請到設定裡的語音頁」（計畫原文是「請到設定 › 語音」）。`NoSymbolIconRuleTests` 禁止 resx 裡出現 `›`（Unicode 符號），所以改寫；`ui-spec` 同步。
- **Esc 取消錄音**只在焦點位於輸入區內時生效（收音時焦點本來就在面板）。
- **計時與音量**：計時器用 `TimeProvider.CreateTimer`，週期 200 ms，只負責發出 `Elapsed` 的變更通知；`Elapsed` 的值由時鐘即時計算。音量節流 100 ms（錄音器約每 50 ms 一次）。原生執行緒來的事件（音量、達上限）都經 `IUiDispatcher.Post` 回到 UI 執行緒。
- **波形**：`LevelWaveform` 固定 32 根，新的 Level 往左推，只用 `LumiAccentBrush`，沒有自己的計時器。轉圈沿用工具卡片的 `ctl|LumiIcon.spin` 動畫（複製了一份樣式，沒有抽成共用）。
- **`WorkbenchViewModel` 的語音參數**是可選的（沒有傳入時用 `NoVoiceInputController`，麥克風停用，原因「尚未設定語音服務」）。既有測試不需要改建構方式。
- **測試替身**：`FakeAudioRecorder`（App.Tests 已有）、`FakeSpeechToText`、`FakeChatSession`（只給控制器用）、`FakeVoiceInput`（給工作台與畫面測試用，手動發出狀態與轉寫）。

既有測試的調整（不是弱化）：

- `ChatComposerTests.TheMicrophone_IsADisabledPlaceholder…`：改為「沒有控制器時停用，Tooltip 是原因，沒有 Caption」。原本斷言的「即將推出」已不存在。
- `ResponsiveWorkbenchTests`：移除 `MicCaption` 的可見性斷言（元件已拿掉）。
- `HeightAndAccessibilityTests` 的 Tab 順序（麥克風停用時不在 Tab 鏈上）不需要改：預設沒有設定語音，麥克風仍停用。

新增測試：

- `VoiceInputControllerTests`（39）：正常編輯／送出、取消（錄音中、轉寫中、失敗中）、太短、靜音、空白結果、失敗後重試與放棄、未預期的例外（訊息不含原始例外內容）、達 90 秒自動編輯、StartAsync 的 `AudioRecorderException`、回應中不可開始與原因文字、Token／端點／Host 不符的原因、設定變更後的可用判斷、RunStarted 與對話切換時取消、Elapsed 與 Level（含節流與夾限）、Dispose 釋放麥克風與取消轉寫。
- `WorkbenchVoiceTests`（19）：麥克風命令與 Tooltip、狀態旗標、時間與狀態文字、收音中送出與 Add 鈕停用、三個命令轉給控制器、提示的鏡像與清除、編輯（插入、附加草稿、空白草稿、標籤移除與保留、焦點請求）、送出（正常路徑經 `SendCommand`、對話不同、回應中、有草稿）。
- `VoiceComposerTests`（22，Headless）：麥克風鈕啟用與停用原因、收音面板取代輸入框、三個按鈕的層級／圖示／Tooltip／名稱／命令、紅點 token 與計時文字、波形接收音量（32 根、往左推）、Esc 取消（收音中有效、其他時候無效）、預設焦點、轉寫中的轉圈與取消、失敗的圖示＋文字＋重試（primary）＋放棄、面板高度與輸入框相同、Compact（800）與 Narrow（500）的收音按鈕收成圖示且不超出輸入區、Wide 顯示文字、提示與標籤、加入鈕收音中停用、收音結束後送出鈕回來、焦點搬移。

驗證：`dotnet build Lumi.sln` 0 警告 0 錯誤；`dotnet test Lumi.sln` 全部通過：Core 294、Infrastructure 111、Tools 329、Sprites 424、Platform 58 + 14 略過、Providers 136、Architecture 67、Speech 67、App 1359（1279 + 新增 80）。

只能真機或真服務驗證的項目（6F 一併）：

- Windows 與 macOS 的真實收音：收音中的波形與計時是否順暢；達 90 秒自動結束；少於 0.5 秒被忽略；按 Esc 後麥克風確實關閉（工作管理員／系統的麥克風指示燈）。
- macOS：第一次收音的權限視窗（6C 的 `NotDetermined` 取捨）；拒絕後是否出現「沒有收到聲音，請確認麥克風權限」而不是卡住。
- 讀屏軟體（VoiceOver、NVDA）對焦點搬移與面板名稱的朗讀；不能只靠自動測試確認無障礙。
- Compact（620–999）與 Narrow（< 620）的實際排版：三個收音按鈕收成圖示時的間距、失敗列在窄寬度的換行、高度 < 700 的輸入區高度。
- 真實服務的錯誤（401、404、逾時、斷網、憑證不受信任）顯示在收音面板的失敗列，「重試轉寫」與「放棄」的行為。
- 收音中切換對話（工作台的歷史抽屜、小精靈的新對話）確實取消錄音、麥克風指示燈熄滅。
- 關閉 App 時若正在收音，麥克風會釋放（`DisposeAsync` 經由 DI 容器）。

### 6E

完成（2026-10-22，程式碼與自動測試；真機收音、真實服務與讀屏軟體尚未驗證，見下）：小精靈輸入列的麥克風鈕與收音列（`Sprites/SpriteWindow.axaml`、`SpriteVoiceBar`、`SpriteVoiceBarViewModel`）、語音的氣泡（`SpriteBubbleViewModel`：引用、提示、失敗）、`SpriteStateMapper.OnVoiceState`、`SpriteConversationController` 的語音接線、`VoiceOrigin`。沒有動錄音器、Lumi.Speech、設定頁。`docs/ui-spec.md` 8 節（8.1、8.2、8.4 與開頭的版面段）、`docs/sprite-dev-guide.md` 第 2 節已同步。

新版面（`SpriteInputBarLayoutTests` 的鎖定值，座標是輸入列內部的 DIP，外框 1px 不計入）：

| 元件 | 尺寸 | 座標 | 說明 |
|---|---|---|---|
| 輸入列外框 | 300×48 | — | 不變（視窗與角色的版面沒動） |
| 麥克風鈕 | 36×36 | (7,6) | 左側 Padding 6、圓形 tertiary |
| 輸入框 | 198×12 | (51,18) | 原本 240×12 在 (17,18)；左右各留 8 |
| 送出鈕 | 36×36 | (257,6) | 不變 |
| 語音標籤（草稿時） | 寬度由文字決定（高 18 左右） | 輸入框左側 | 收起時寬度 0，輸入框位置不動；顯示時輸入框往右並縮窄 |

理由：外框與視窗都不變，`SpriteLayout`（點擊穿透、角色與輸入列的相對位置）完全不受影響；麥克風只能放在輸入框左側，所以輸入框縮窄（240 → 198）。語音標籤放在輸入框內部（不另外佔高度），因為輸入列上方是角色，沒有空間。

偏離原計畫或需要記下的地方：

- **收音列的按鈕尺寸**：三個圓形圖示鈕 36×36（同麥克風與送出），收音列內的寬度預算（估計，實際由版面決定，測試只驗證都在框內）：紅點 8＋間距＋計時（約 40）＋波形（最小 40，剩下的寬度）＋三鈕 116，內容區寬 286。沒有用計畫外的新尺寸。
- **失敗時的兩層顯示**：收音列顯示「語音轉寫失敗：<一句話>」（完整文字在 Tooltip）加圖示鈕「重試轉寫」（primary）「放棄這段錄音」（tertiary）；氣泡顯示同一句話加文字按鈕。計畫兩處都寫了，都做。
- **光球狀態：失敗用 `Error`，不用 `Attention`**。`docs/sprite-dev-guide.md` 的 `Error` 是「API 失敗、斷線」，`Attention` 是「工具需要使用者同意」（橘色彈跳）。語音失敗屬於前者，所以用 `Error`，停到錯誤被處理為止（點光球、重試、放棄）。
- **對話優先**：`SpriteStateMapper.OnVoiceState` 在小精靈這一輪回應進行中忽略語音狀態（只有 `OnRunStarted` 到 `OnRunFinished` 之間）。轉寫在回應開始前已經開始的，回應結束後的語音狀態才會生效。工作台發起的回應不影響小精靈的語音狀態（小精靈只追蹤自己的回應）。語音回到 `Idle` 時只收回語音自己設定的狀態（`_voiceDriven`），不會把對話的 `Done`／`Error` 蓋掉。
- **`VoiceOrigin`（6D 沒有的介面變更）**：工作台與小精靈共用一個控制器，而且兩邊都訂閱 `Transcribed`，所以結果必須知道是誰開始的。因此 `StartAsync(VoiceOrigin origin, ct)`、`VoiceTranscribedArgs` 多一個 `Origin`、介面新增 `Origin` 與 `DisabledReasonFor(origin)`（另一邊錄音或轉寫時回「另一個視窗正在語音輸入」，新字串 `Voice_OtherWindow`）。提示（Notice）也只顯示在開始錄音的那一邊。工作台的既有測試改成傳入 `VoiceOrigin.Workbench`（機械式替換，沒有刪或弱化斷言）。
- **隱藏與送出**：小精靈隱藏（暫時隱藏或切到工作台模式）時，小精靈開始的錄音會取消（不在背景錄音）；轉寫中的結果若在隱藏時回來，一律放進輸入框、不送出（和「對話已改變」相同的規則）。
- **送出的條件**與 6D 的工作台相同，另外加上「小精靈顯示中」。送出時先以小字引用（`SpriteBubbleViewModel.OnRunStarted(quote)`，截斷 48 個字形叢集），回答開始就清掉引用。
- **草稿分隔符號**：小精靈的輸入框是單行（`AcceptsReturn=False`），草稿之間用空白接，不用換行（工作台用換行）。
- **提示與其他氣泡**：提示（`SpriteBubbleKind.Notice`）約 4 秒淡出（新選項 `SpriteShellOptions.NoticeLinger`，游標停在上面時暫停），淡出時不通知角色（不是錯誤）。權限確認與錯誤氣泡正在顯示時，提示**不顯示**（不排隊；確認與錯誤要能按，提示只是補充）。語音失敗的氣泡在權限確認時不蓋過確認（失敗仍在收音列上）。
- **輸入框的語音標籤**：偵測文字改變用 `TextBox.TextProperty` 的屬性變更，不用 `TextChanged` 事件（在這個 Avalonia 版本，程式設定 `Text` 時 `TextChanged` 不會發出，測試抓到的）。使用者改動或清空文字後標籤拿掉。
- **焦點**：狀態改變時焦點跟著走（收音→「停止並編輯」，轉寫→「取消」，失敗→「重試轉寫」，閒置→輸入框），和工作台相同，延後一拍（`Loaded`）。Esc 在收音中取消（焦點在收音列上時也有效，`SpriteWindow.OnWindowKeyDown`）。
- **權限確認與錄音**：計畫寫「錄音中收到需要權限確認時錄音不中斷」。實作上這個情況不會發生：回應開始時收音會取消（6D 已定案的規則），所以權限確認只會出現在沒有收音的時候。收音列與確認氣泡互不干擾（確認氣泡不在輸入列上）。**待確認**：要不要改 6D 的規則（回應開始時不取消收音）；目前沒有改。
- **錯誤的氣泡與收音**：語音失敗的氣泡按「重試轉寫」→ 氣泡收起、狀態回到 `Transcribing`；「放棄」→ 回到閒置。切換對話時（`ConversationChanged`）收音與失敗的錄音一併丟掉（6D 規則），氣泡的語音錯誤也一併收起。
- **讀屏**：收音列的按鈕都有 `AutomationProperties.Name`，收音列的 Name 是「收音中 00:12」（`Voice_StatusRecording`）。實際朗讀要真機驗證。

新增字串（`Strings.resx`／`Strings.cs`）：`Voice_OtherWindow`、`Sprite_VoiceHidden`、`Sprite_VoiceLabel`（「語音」）、`Sprite_VoiceTranscribing`、`Sprite_VoiceFailedFormat`（「語音轉寫失敗：{0}」）。其他沿用 6D 的字串（`Voice_StopEdit`、`Voice_StopSend`、`Voice_Retry`、`Voice_DiscardTip`、`Voice_Cancel*`、`Voice_Notice*`）。

新增與修改的測試（App.Tests 1359 → 1432）：

- `Sprites/SpriteVoiceStateTests`：狀態對應、失敗與重試、語音的 Idle 不蓋過聊天的 Error／Done、對話進行中的優先序、退化（`SpriteStateResolver`）。
- `Sprites/SpriteVoiceBarViewModelTests`：麥克風的啟用與原因（含隱藏、另一個視窗）、收音列的文字、隱藏時取消、Esc、命令轉給控制器、草稿標籤。
- `Sprites/SpriteBubbleVoiceTests`：引用（截斷、說完就清掉）、提示（淡出、懸停暫停、不蓋過確認與錯誤）、語音失敗（不淡出、重試與放棄的事件、`HideVoiceFailure` 只收語音錯誤、等待確認時不蓋過）。
- `Sprites/SpriteVoiceConversationTests`：真的 `ChatSession`、`SpriteWindow` 與假的語音控制器：停止並送出（引用先於思考、送出後回答）、停止並編輯（標籤、不送出、追加草稿、改動後標籤拿掉）、草稿時送出被擋、對話不同、回應中、隱藏時、工作台的結果不進小精靈、提示氣泡（小精靈與工作台來源）、失敗氣泡與重試／放棄、狀態跟著語音（含收音列取代輸入列）；另有一組用**真的 `VoiceInputController`**（假的錄音器與假的 Whisper）：停止並送出端到端、失敗後重試放進輸入框。
- `Headless/SpriteVoiceBarTests`：麥克風的啟用、Tooltip（ShowOnDisabled）、無障礙名稱、隱藏與另一個視窗的停用原因；收音列三種狀態的按鈕名稱、Tooltip、36×36、都在輸入列框內；計時名稱；Esc（收音中有效、轉寫中無效）；焦點；語音標籤的位置與框內。
- `Sprites/SpriteInputBarLayoutTests`：鎖定值改成新版面（麥克風 (7,6)、輸入框 198×12 在 (51,18)、送出鈕不變），理由見上。
- `WorkbenchVoiceTests`、`VoiceInputControllerTests`：origin 標記、另一邊的停用原因、小精靈的結果與提示不進工作台。

驗證：`dotnet build Lumi.sln` 0 警告 0 錯誤；`dotnet test Lumi.sln`：Core 294、Infrastructure 111、Tools 329、Sprites 424、Platform 58（14 略過）、Providers 136、Architecture 67、Speech 67、App 1432（1359 + 73 新增）。既有的色彩、間距、符號、按鈕覆蓋與對比測試全部通過（`Sprites/` 仍是間距規則的例外）。

只能真機或真服務驗證的項目（併入 6F 的手動驗收）：

- 小精靈的收音列：波形與計時在真實音量下的表現；按 Esc 後麥克風指示燈熄滅；隱藏小精靈時收音停止。
- 三個圓形圖示鈕在角色的各種色調與淺深主題下的對比（光球背景上的輸入列是 Surface 底，應無問題，實機確認）。
- 「語音」標籤與縮窄後的輸入框在長文字時的表現（輸入框捲動、標籤不被擠掉）。
- 讀屏軟體（VoiceOver、NVDA）對焦點搬移與收音列名稱的朗讀。
- 真實服務的錯誤顯示在收音列與氣泡（401、404、逾時、斷網、憑證不受信任），以及「重試轉寫」「放棄」。

### 6F

- 文件同步：`docs/ui-spec.md`（6B～6E 已各自更新）、`agent.md` 第 11 節改寫為已定案規則、第 6 節介面表加 `ISpeechConnectionTester`、`README.md`、主計畫（M6 狀態、設定頁項目調整、決策紀錄、待確認事項兩項改為已確認）。
- **人工驗收清單尚未執行**（見上方「手動驗收」）；真實回應 fixture 待補。
