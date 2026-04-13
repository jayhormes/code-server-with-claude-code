# Code Server With Claude Code

一份 Claude Code Skill，用來在 Mac 上架設 code-server 並透過 Cloudflare Tunnel 暴露給任意瀏覽器使用 Claude Code。

## 這個 Skill 做什麼

讓你在「不能安裝軟體」的電腦（例如公司筆電）上，用瀏覽器開一個分頁就能跑 Claude Code。實際運算仍在你家裡的 Mac 上執行。

```
公司瀏覽器 → https://code.你的域名.com → Cloudflare Tunnel → Mac
                                                             ├─ code-server（瀏覽器版 VS Code）
                                                             ├─ Claude Code 擴充
                                                             └─ 終端機 → claude CLI
```

把 [SKILL.md](./SKILL.md) 交給任何支援 Claude Agent Skill 規格的 agent（或手動丟給 Claude Code），它就會依照指引幫你：

- 安裝 `code-server`、`cloudflared`、Node.js、`claude` CLI
- 建立 Cloudflare Tunnel（分兩條路線，見下）
- 設定 macOS LaunchDaemon 讓服務開機自啟
- 在 code-server 內裝好 Claude Code extension 並完成首次 OAuth
- 驗證 tunnel 連線、回報完成狀態

## 兩條建置路線

Skill 會先問你是否擁有自己的域名，再選擇路線：

| 路線 | 適用 | 網址 | 需要域名 | 開機自啟 |
|------|------|------|---------|---------|
| **Quick Tunnel** | 先驗證、臨時使用 | 隨機 `*.trycloudflare.com` | 否 | 否 |
| **Named Tunnel** | 長期正式使用 | 固定 `code.你的域名.com` | 是 | 是 |

兩條路線互不衝突——可先用 Quick Tunnel 驗證，日後買了域名再升級。

## Claude Code 專屬支援

除了基礎建設，Skill 還涵蓋 Claude Code 在 code-server 環境下的特殊處理：

- `claude` CLI 裝在 Mac 端（而非 client 筆電）與 OAuth 首次登入
- Extension 與 CLI 的 auth 共用機制
- MCP server 在無 GUI 環境下的限制與設定檔位置
- 瀏覽器端常見問題：WebSocket 斷線、快捷鍵衝突、剪貼簿權限、extension 快取、CLI lockfile
- CLI 與 extension 的更新流程

## 安裝為 Claude Code Skill

把 `SKILL.md` 放到 Claude Code 的 skills 目錄即可自動載入：

```bash
mkdir -p ~/.claude/skills/code-server-setup
cp SKILL.md ~/.claude/skills/code-server-setup/SKILL.md
```

之後在 Claude Code 對話中說「幫我在 Mac 上設定 code-server 讓我從瀏覽器遠端用 Claude Code」，skill 會自動觸發。

## 手動給其他 agent 使用

若 agent 不支援 Skill 自動載入機制，直接把 `SKILL.md` 作為 prompt 上下文貼給它，告訴它「照這份文件執行」即可。

## 安全建議

- 把 code-server 預設隨機密碼改成自訂強密碼
- 正式使用時加一層 Cloudflare Zero Trust（Email OTP 或 Google SSO）
- 不要讓密碼以明文長期留在 `config.yaml`
