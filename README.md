# Lumi — 桌面 AI 小精靈

Lumi 是一個跨平台（Windows / macOS）的桌面 AI 助理。使用者自帶 API 金鑰（BYOK），支援 Anthropic 與 OpenAI 相容格式的模型。

它有兩種模式，共用同一則對話：

- **工作台**：完整的對話視窗。可以切換模型、加入檔案或資料夾當上下文、語音輸入，並顯示 token 用量與回合數。
- **小精靈**：一個住在桌面上的透明角色（第一個角色是會呼吸的光球）。游標靠近時才浮出輸入列，用簡短的氣泡回答，並用動畫表現聆聽、思考、呼叫工具、完成等過程。

> 目前狀態：**M1 進行中**。光球 POC 已在 macOS 驗證；M1 的四段（骨架與外觀、金鑰安全儲存、Anthropic Provider、工作台與 SQLite）已完成並在 Mac 驗收，可以在工作台和 Claude 對話；待 Windows 驗證。

## 特色

- **BYOK、多模型**：可以新增多組連線與模型，工作台和小精靈可以各自用不同的模型
- **工具呼叫**：內建工具採用統一的 `ITool` 介面，以 DI 注入；寫入類工具會先詢問使用者
- **檔案與資料夾上下文**：資料夾不會整包塞進 prompt，而是讓模型透過「限定範圍」的工具按需讀取
- **語音輸入**：透過 Whisper 服務轉成文字（以 `ISpeechToText` 注入，可替換）
- **人設**：可設定個性、積極度與自訂人設；小精靈模式會自動用更短的方式回答
- **可擴充的小精靈角色**：殼與角色分離，支援程式繪製的角色，以及「美術素材 + JSON」的角色
- **全部程式繪製**：光球不使用任何圖片，閒置時自動降低幀率以節省電力

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
docs/
  ui-spec.md                工作台 UI 規格（字型、色系、元件行為）
  sprite-dev-guide.md       小精靈角色開發指南
src/
  Lumi.Core/                資料模型、核心介面、對話迴圈（不依賴 UI）
  Lumi.Providers/           Anthropic（官方 SDK）、OpenAI 相容（M2）
  Lumi.Infrastructure/      SQLite（對話與訊息）
  Lumi.Platform/            原生 API、金鑰安全儲存（DPAPI、Keychain）
  Lumi.App/                 Avalonia 介面與 DI 組裝（唯一的 composition root）
tests/
  Lumi.Core.Tests/
  Lumi.Platform.Tests/      金鑰儲存的契約測試（記憶體、DPAPI、Keychain）
  Lumi.Providers.Tests/     Anthropic Provider 的 fixture 測試（不呼叫真實 API）
  Lumi.Infrastructure.Tests/ SQLite 實作（暫存資料夾的真資料庫檔）
  Lumi.App.Tests/           ViewModel、Markdown 解析與渲染（含 Avalonia.Headless）、EmojiText、字串資源
  Lumi.Architecture.Tests/  相依規則、色碼檢查、P/Invoke 與供應商 SDK 的歸屬
poc/
  Lumi.OrbPoc/              光球 POC（驗證小精靈視窗的可行性，不在 Lumi.sln 內）
```

之後會加入的專案（詳見 `agent.md`）：

```
Lumi.Tools                  內建工具
Lumi.Speech                 語音轉文字
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

## 授權

尚未決定。
