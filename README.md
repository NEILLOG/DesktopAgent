# Lumi — 桌面 AI 小精靈

Lumi 是一個跨平台（Windows / macOS）的桌面 AI 助理。使用者自帶 API 金鑰（BYOK），支援 Anthropic 與 OpenAI 相容格式的模型。

它有兩種模式，共用同一則對話：

- **工作台**：完整的對話視窗。可以切換模型、加入檔案或資料夾當上下文、語音輸入，並顯示 token 用量與回合數。
- **小精靈**：一個住在桌面上的透明角色（第一個角色是會呼吸的光球）。游標靠近時才浮出輸入列，用簡短的氣泡回答，並用動畫表現聆聽、思考、呼叫工具、完成等過程。

> 目前狀態：**規劃與 POC 階段**。光球 POC 已在 macOS 驗證，正式程式尚未開始。

## 特色

- **BYOK、多模型**：可以新增多組連線與模型，工作台和小精靈可以各自用不同的模型
- **工具呼叫**：內建工具採用統一的 `ITool` 介面，以 DI 注入；寫入類工具會先詢問使用者
- **檔案與資料夾上下文**：資料夾不會整包塞進 prompt，而是讓模型透過「限定範圍」的工具按需讀取
- **語音輸入**：透過 Whisper 服務轉成文字（以 `ISpeechToText` 注入，可替換）
- **人設**：可設定個性、積極度與自訂人設；小精靈模式會自動用更短的方式回答
- **可擴充的小精靈角色**：殼與角色分離，支援程式繪製的角色，以及「美術素材 + JSON」的角色
- **全部程式繪製**：光球不使用任何圖片，閒置時自動降低幀率以節省電力

## 技術棧

.NET 10 · Avalonia 11.3 · CommunityToolkit.Mvvm · Microsoft.Extensions.Hosting · SQLite · System.Text.Json

## 專案結構

```
agent.md                    給 AI agent 與開發者的開發指引（必讀）
plan/                       工作計畫
  00-master-plan.md         主計畫：里程碑、決策紀錄、待確認事項
docs/
  sprite-dev-guide.md       小精靈角色開發指南
poc/
  Lumi.OrbPoc/              光球 POC（驗證小精靈視窗的可行性）
```

正式程式預計的分層（尚未建立），詳見 `agent.md`：

```
Lumi.Core                   資料模型、核心介面、對話迴圈（不依賴 UI）
Lumi.Providers              Anthropic、OpenAI 相容
Lumi.Tools                  內建工具
Lumi.Speech                 語音轉文字
Lumi.Infrastructure         SQLite、設定、金鑰安全儲存
Lumi.Platform               原生 API（游標、點擊穿透、錄音）
Lumi.Sprites.Abstractions   小精靈角色介面
Lumi.Sprites.*              各個角色
Lumi.App                    Avalonia 介面
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
